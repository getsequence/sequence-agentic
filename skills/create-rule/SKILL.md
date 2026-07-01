# Skill: Create a Sequence Rule

Use this skill whenever the user wants to automate money movement — even if they haven't mentioned "rules" or Sequence directly. Load this before calling the `createRule` API endpoint.

---

## Prerequisites

- A Sequence API key scoped to `CREATE_AND_EDIT_RULES` (and `READ_RULES` + `READ_ACCOUNTS`).
- At least one account already connected in Sequence. If none exist, use [connectExternalAccount](https://docs.getsequence.io) first.
- Base URL: `https://api.getsequence.io/platform/v1`

---

## Step-by-step workflow

Follow these steps **in order** every time. Never call `createRule` before completing all earlier steps.

### 1. Discover the goal

If the user's request is vague, ask these three questions **one at a time**:
1. What's causing you the most financial stress or taking up mental energy right now?
2. If you could automate one financial task to make your life easier, what would it be?
3. What's a financial goal you've been wanting to work toward but haven't started yet?

For clear requests, ask one clarifying question at a time. You need to know: **what money is moving, from where, to where, when, and whether any conditions apply** (balance threshold, deposit size, date).

### 2. Load accounts and rules

```
GET /accounts?pageSize=100     (paginate until hasNextPage is false)
GET /rules?pageSize=100        (paginate until hasNextPage is false)
```

You need both lists before presenting any plan.

### 3. Check the hard constraint

**Each account supports exactly ONE `ON_FUNDS_TRANSFERRED` rule.**

Before proposing anything:
- If a rule with `trigger.type: ON_FUNDS_TRANSFERRED` already exists on the intended source → you **must edit that rule** to add the new action(s). Never create a second one.
- For `SCHEDULED` rules: same source + same schedule → edit; different schedule → a new rule is fine.

### 4. Apply goal→pattern reasoning

Pick one of the four patterns below, then build the JSON:

---

## Patterns

### Pattern A — Direct transfer (single rule)

Use when money goes straight from source to destination with no staging.

**Triggers:** `ON_FUNDS_TRANSFERRED` or `SCHEDULED`
**Actions:** `FIXED`, `PERCENTAGE`, `TOP_UP`, `ROUND_DOWN`

Examples: "Put 20% of my income in savings", "Transfer $500 on the 1st", "Build an emergency fund", "Keep a $1,000 buffer and sweep the rest"

→ See [`examples/pattern-a-percentage-save.json`](./examples/pattern-a-percentage-save.json) and [`examples/pattern-a-scheduled-fixed.json`](./examples/pattern-a-scheduled-fixed.json)

---

### Pattern B — Accumulate then pay out (two rules)

Use when the user wants to stage funds in a pod before paying a liability or goal.

**Rule 1 (Funding):** `ON_FUNDS_TRANSFERRED` on income → `PERCENTAGE` or `FIXED` transfer to a holding pod
**Rule 2 (Payout):** `SCHEDULED` (specific date) or `ON_FUNDS_TRANSFERRED` on the pod (auto-pay when funded)

Examples: "Set aside 15% of income for taxes, then pay my quarterly bill", "Build up my credit card pod and auto-pay on the 25th"

→ See [`examples/pattern-b-accumulate-payout.json`](./examples/pattern-b-accumulate-payout.json)

---

### Pattern C — Multi-liability strategy (two rules)

Use when the user has multiple liabilities to pay from one funding pot.

**Rule 1:** `ON_FUNDS_TRANSFERRED` on income → `PERCENTAGE` or `FIXED` to a Liabilities pod
**Rule 2:** `ON_FUNDS_TRANSFERRED` on the Liabilities pod → multiple liability payment actions in a single step

Examples: "Pay all my credit cards automatically", "Distribute debt payments when money arrives"

→ See [`examples/pattern-c-multi-liability.json`](./examples/pattern-c-multi-liability.json)

---

### Pattern D — Income split (single rule, multiple actions)

Use when the user wants to split income across several buckets simultaneously.

**Trigger:** `ON_FUNDS_TRANSFERRED` on income
**Actions:** Multiple `PERCENTAGE` actions in one step — all calculated from the full incoming amount at once

Examples: "50/30/20 budget", "Profit First allocations", "Split paycheck into taxes, savings, and spending"

→ See [`examples/pattern-d-income-split.json`](./examples/pattern-d-income-split.json)

---

## Action types

| Type | Description | Required fields |
|------|-------------|-----------------|
| `FIXED` | Transfer a fixed dollar amount | `amountInCents` |
| `PERCENTAGE` | Transfer % of incoming amount or source balance | `percentageValue`, `percentageTarget` (`INCOMING_AMOUNT` or `SOURCE_ACCOUNT`) |
| `TOP_UP` | Fill destination to a target balance | `amountInCents` (target) |
| `ROUND_DOWN` | Keep a fixed buffer; sweep everything above it | `amountInCents` (buffer to keep) |
| `LAST_STATEMENT_BALANCE` | Pay last billing cycle amount | — |
| `NEXT_PAYMENT_MINIMUM` | Pay only the minimum due | — |
| `TOTAL_AMOUNT_DUE` | Pay the full current outstanding balance | — |
| `PERCENTAGE_LIABILITY_BALANCE` | Pay % of the outstanding balance | `percentageValue` |

**Action ordering:** Actions run top to bottom within a step. If one fails (e.g. insufficient funds), subsequent actions in that step are skipped. Put the most important actions first.

**Multiple `PERCENTAGE` actions in one step** all calculate from the full source amount simultaneously — they do not cascade.

---

## Trigger types

| Type | When it fires | Key fields |
|------|--------------|------------|
| `ON_FUNDS_TRANSFERRED` | When money arrives at `accountId` | `accountId` |
| `SCHEDULED` | On a calendar schedule | `scheduleType`, `startDate`, `accountId` |
| `MANUAL` | Only when explicitly triggered via API | `accountId` |

`scheduleType` values: `DAILY`, `WEEKLY`, `BI_WEEKLY` (1st and 15th), `MONTHLY`, `EVERY_OTHER_WEEK` (fortnightly), `ONE_TIME`

---

## API call

```
POST https://api.getsequence.io/platform/v1/rules
Authorization: Bearer sk_...
Content-Type: application/json
X-Called-Reason: <short description of what the user is trying to do>
```

Body shape:
```json
{
  "name": "Human-readable name",
  "trigger": { ... },
  "steps": [
    {
      "conditions": null,
      "actions": [ ... ]
    }
  ]
}
```

See the [example files](./examples/) for copy-paste-ready request bodies.

---

## After creation

**Rules are always created in DISABLED status.** After a successful `createRule` call:

1. Take the `id` from `response.data.id`
2. Show the user this activation link — once at the **top** and once at the **bottom** of your reply:
   ```
   https://app.getsequence.io/map/rules/<RULE_ID>?action=activateRule
   ```
3. Frame it as the natural next step, not an afterthought: *"Your rule is ready. Open this link to activate it."*

---

## Known API limitations

These are not available via API — instruct the user to add them manually in the rule drawer after activation:

- **Conditions** (balance threshold, day-of-month, transfer amount minimum/maximum)
- **Avalanche and Snowball debt payoff strategies**

If the user's goal implies one of these, note it in the plan: *"I'll create the rule now. You'll need to add the balance condition yourself in the app after activating."*
