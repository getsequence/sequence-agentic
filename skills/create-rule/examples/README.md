# Rule examples

Copy-paste request bodies for `POST https://api.getsequence.io/platform/v1/rules`.

Each file contains a complete `createRule` request body (or a pair of bodies for two-rule patterns). Replace the placeholder IDs with real account IDs from `GET /accounts`.

---

## How to inject account IDs

1. Call `GET /accounts?pageSize=100` (paginate until `hasNextPage` is false).
2. Find the accounts you need by `name` or `type`.
3. Replace each placeholder in the JSON with the real `id` (a UUID).

### Account types

| Type | What it is |
|------|-----------|
| `INCOME_SOURCE` | Where incoming money lands (paycheck, revenue) |
| `POD` | Goal-based savings bucket |
| `CHECKING` | Checking account |
| `SAVINGS` | Savings account |
| `EXTERNAL` | External connected account (bank, credit card, liability) |

---

## Examples

### [pattern-a-percentage-save.json](./pattern-a-percentage-save.json)
**Save 20% of every paycheck**
- Trigger: `ON_FUNDS_TRANSFERRED` on income
- Action: `PERCENTAGE` — 20% of incoming amount → savings pod
- IDs to inject: `INCOME_SOURCE_ID`, `SAVINGS_POD_ID`

---

### [pattern-a-scheduled-fixed.json](./pattern-a-scheduled-fixed.json)
**Transfer a fixed amount monthly**
- Trigger: `SCHEDULED` / `MONTHLY` on the 1st
- Action: `FIXED` — $500 → savings pod
- IDs to inject: `CHECKING_ACCOUNT_ID`, `SAVINGS_POD_ID`
- Also update `startDate` to the next upcoming 1st of the month (UTC)

---

### [pattern-b-accumulate-payout.json](./pattern-b-accumulate-payout.json)
**Stage funds in a pod, then pay out on a date**
- Two rules: rule 1 funds a Tax pod on income; rule 2 sweeps the pod on a fixed date
- Pattern: accumulate → pay
- IDs to inject: `INCOME_SOURCE_ID`, `TAX_POD_ID`, `TAX_PAYMENT_ACCOUNT_ID`
- Also update `startDate` on rule 2 to your next payment date

---

### [pattern-c-multi-liability.json](./pattern-c-multi-liability.json)
**Pay multiple credit cards automatically**
- Two rules: rule 1 moves 30% of income to a Liabilities pod; rule 2 pays each card's last statement balance when money arrives
- IDs to inject: `INCOME_SOURCE_ID`, `LIABILITIES_POD_ID`, `CREDIT_CARD_1_ID`, `CREDIT_CARD_2_ID`, `STUDENT_LOAN_ID`
- Add or remove actions in rule 2 for each liability you want to pay

---

### [pattern-d-income-split.json](./pattern-d-income-split.json)
**Profit First income split (50/30/20)**
- Single rule, three simultaneous `PERCENTAGE` actions — all calculate from the full incoming amount at once
- IDs to inject: `INCOME_SOURCE_ID`, `OWNERS_PAY_POD_ID`, `TAX_POD_ID`, `PROFIT_POD_ID`
- Adjust percentages to match your allocation (they don't need to sum to 100%)

---

## Important notes

- Rules are always created in **DISABLED** status. After creation, activate via: `https://app.getsequence.io/map/rules/<RULE_ID>?action=activateRule`
- **One `ON_FUNDS_TRANSFERRED` rule per source account.** If one already exists on that source, you must edit it (`PATCH /rules/<id>`) rather than create a second.
- Amounts are always **integer cents** (`50000` = $500.00).
- `groupIndex` groups actions within a step that share a spending cap. Use the same index for actions that should share a combined limit; use different indices otherwise.
