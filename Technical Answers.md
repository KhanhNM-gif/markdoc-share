# Technical Answers

---

## Q1. Tech Definitions & Implementation Plan

### Terminology

| Term                  | What it means                                                                                                   |
| --------------------- | --------------------------------------------------------------------------------------------------------------- |
| **creditLimit**       | The maximum amount a user is allowed to spend. TK sets and controls this via API.                               |
| **availableCredit**   | `creditLimit - amountSpent` — how much the user can still spend. Updated automatically by US Bank in real time. |
| **systemFreeBalance** | The portion of the system wallet not yet allocated to any user. Tracked by TK internally.                       |

---

### 1) Top-up

When a user tops up their HolaPay wallet, we **increase their `creditLimit`** on the card. No actual money moves between accounts.

```
User requests top-up +$X
        ↓
TK Guard: X <= systemFreeBalance ?
        ↓ pass
PATCH /accounts/{userAccountUID}/credit-limit
Body: { "creditLimit": currentLimit + X }
        ↓
TK DB: systemFreeBalance -= X
```

Result: the user's `availableCredit` increases by exactly $X.

---

### 2) Withdraw

When a user withdraws from their card back to the TK wallet, we **decrease their `creditLimit`**. This is entirely internal — no real money leaves US Bank.

```
User requests withdraw -$X
        ↓
TK Check: X <= user.availableCredit ?
        ↓ pass
PATCH /accounts/{userAccountUID}/credit-limit
Body: { "creditLimit": currentLimit - X }
        ↓
Webhook: corporate_cards.account.limit.updated
        ↓ confirmed
TK DB: user.tk_wallet_balance += X
       systemFreeBalance      += X
```

Result: `creditLimit` drops by $X, user's internal TK wallet increases by $X, and `systemFreeBalance` is freed up again.

> **Important**: Only credit the TK wallet after receiving the webhook confirmation from US Bank — never before.

---

### 3) Balance Management

Money never actually moves between user accounts. Everything sits in the system wallet (managing account). What TK does is adjust each user's `creditLimit` — essentially controlling how much of the shared pool they're allowed to use.

```
System Wallet: $1,000
  ├── User A: creditLimit=$400, availableCredit=$360 (spent $40)
  ├── User B: creditLimit=$300, availableCredit=$300
  └── systemFreeBalance: $300  (not allocated to anyone)
```

#### TK manages `creditLimit` only — not `availableCredit`

|                   | Managed by                                                     | Can TK change it?                  |
| ----------------- | -------------------------------------------------------------- | ---------------------------------- |
| `creditLimit`     | TK                                                             |  Yes — via `PATCH /credit-limit` |
| `availableCredit` | US Bank calculates automatically (`creditLimit - amountSpent`) |  No                              |
| `amountSpent`     | US Bank tracks on every transaction                            |  No                              |

We focus on `creditLimit` as the single control point — it's clean, predictable, and doesn't depend on real-time state from US Bank. `availableCredit` is just a derived value we read when we need to show it to the user.

- Fetch real-time available credit: `GET /accounts/{userAccountUID}/realtime-credit-details`
- Fetch total system balance: `GET /accounts/{managingAccountUID}`

---

### 4) Monthly Billing Cycle Reset

**Default US Bank behavior**: At the start of each billing cycle, US Bank automatically resets `availableCredit` back to `creditLimit`.

Example: a user with `creditLimit=$100` who spent $99 (only $1 left) will have their `availableCredit` reset back to $100 at the start of the next cycle.

#### TK needs to implement this — Jubielee will share a reference script

Jubielee built a custom script for their own app to preserve the actual remaining balance across resets. They've offered to share it so we can adapt it for TK.

**Implementation logic:**

```
End of billing cycle (before US Bank resets):
  1. GET /accounts/{userAccountUID}/realtime-credit-details
     → fetch each user's current availableCredit
  2. Save to TK DB: saved_balance = availableCredit

Start of new cycle (after US Bank has reset):
  3. PATCH /accounts/{userAccountUID}/credit-limit
     Body: { "creditLimit": saved_balance }
     → overwrite with saved value to preserve the real balance
```

> **Action item**: Receive the script from Jubielee → adapt it for TK → test on UAT before going to production.

---

## Q2. managingAccountUID vs. AccountUID — Account Structure in TK

### The difference

|                | `managingAccountUID`                | `userAccountUID`                 |
| -------------- | ----------------------------------- | -------------------------------- |
| **What it is** | TK's system wallet (master account) | One account per end-user         |
| **How many**   | **One** for the entire system       | **One per user**                 |
| **Used for**   | Checking total system balance       | All user-related operations      |
| **Example**    | `GET /accounts/0142307324000033`    | `GET /accounts/{userAccountUID}` |

### How TK structures accounts internally

Each user has their own set of IDs. We store all three layers in TK's DB:

```
tk_user_id  (TK internal)
  └── accountUID    ← used for all balance and limit operations
        └── cardId  ← used for activation and card-level controls
```

### Pre-issue strategy (zero wait time for users)

Account creation isn't instant — there's processing time on US Bank's side. To avoid making users wait, we **create accounts in advance** and assign them when a user registers.

```
[Batch job — run in advance by admin]
  POST /accounts/setup × N
  → Save to DB: status = "UNASSIGNED"

[When user registers for a card]
  1. Pull one UNASSIGNED account from DB
  2. PATCH /accounts/{id}/owner          ← attach user's info
  3. PATCH /accounts/{id}/credit-limit   ← set initial limit
  4. POST  /accounts/{id}/activate       ← activate the card
  5. DB: status = "ASSIGNED", link tk_user_id
```

Current pre-issue count: **2,000–3,000 cards**. Can be scaled up as needed.

---

## Q3. Spend Limit Enforcement — Event-based Balance Tracking

### How spending is controlled

US Bank automatically declines any transaction that fails either of these checks:

```
① user.availableCredit >= spendAmount   ← user level
② systemBalance        >= spendAmount   ← system level
```

TK doesn't need to handle spend enforcement. What TK does need to handle is the **guard on top-up** — making sure we never allocate more than what's actually in the system wallet.

---

### Event-based tracking

TK keeps a `systemFreeBalance` value in the DB and updates it directly on every relevant event:

| Event                         | Change to `systemFreeBalance` |
| ----------------------------- | ----------------------------- |
| Admin funds system wallet +$X | `+= X`                        |
| User top-up +$X               | `-= X`                        |
| User withdraw -$X             | `+= X`                        |

**Example:**

```
Admin funds $1,000   → systemFreeBalance = 1,000
User A top-up +$100  → systemFreeBalance =   900
User B top-up +$200  → systemFreeBalance =   700
User A withdraw $30  → systemFreeBalance =   730
```

This is a single-row read — O(1), no need to sum across the entire users table.

---

### Full state machine (with concrete numbers)

```
Event                    | Change            | systemFreeBalance | user.creditLimit | user.availableCredit
-------------------------|-------------------|:-----------------:|:----------------:|:-------------------:
(Initial state)          |        —          |         0         |        0         |          0
Admin funds +$1,000      |      +1,000       |      1,000        |        —         |          —
User A top-up +$100      |       -100        |        900        |      100         |        100
User A spends $40        |         —         |        900        |      100         |         60
User A withdraws $30     |        +30        |        930        |       70         |         30
```

> Withdraw: we reduce `creditLimit` by $30 and credit $30 to the user's internal TK wallet. The actual money stays in US Bank.

---

### Periodic reconciliation (preventing drift)

```
Daily cron job:
  1. GET /accounts/{managingAccountUID}
     → fetch actual availableCredit from US Bank
  2. Compare against systemFreeBalance in TK DB
  3. If there's a discrepancy → overwrite + alert Ops
```

---

### Alert thresholds

```
systemFreeBalance < 10% of systemBalance  →  ⚠️  Warn Ops to add funds
systemFreeBalance < 5%  of systemBalance  →  🔴  Block new top-ups
systemFreeBalance = 0                     →  🔴  Emergency — notify Ops immediately
```

These are sensible defaults, but should be tuned based on TK's usage patterns:

- If top-up frequency is high, bump the warning threshold to 20% for more buffer
- For large user bases, absolute dollar thresholds may be easier to reason about than percentages
- The "block top-up" threshold should be configurable via the admin panel, not hardcoded
