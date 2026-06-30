# Setup — Contractor Expense Reimbursement (n8n)

Time: ~30–45 min. You'll set up Sequence, Slack, and an LLM key, import the
workflow, fill in your config, and do a safe simulated run before any real money
moves.

Work top to bottom — later steps depend on IDs you capture earlier.

---

## 1. Sequence: source account, per‑contractor payout rules, API key

**a. Create the company source account / pod.** In the
[Sequence dashboard](https://app.getsequence.io), create (or pick) the company
account that reimbursements are paid **from**, and connect each contractor's
destination account.

**b. Create one payout rule per contractor.** Each rule moves money **up to a
limit** from the company source to that contractor. The playbook fires this
rule with the receipt amount — it never creates ad‑hoc transfers.

> The rule must:
> - have source = company account, destination = the contractor's account;
> - carry a **max amount** (your per‑payout ceiling — the API rejects amounts
>   above it with `MAXIMUM_AMOUNT_EXCEEDED`);
> - be **MANUAL** trigger type (it's fired on demand by the API);
> - be **enabled by a human** in the dashboard — the API cannot enable a rule,
>   and a disabled rule returns `RULE_DEACTIVATED` when triggered.

**Guided rule creation (preferred — coming soon).** Sequence is adding a
**deep link** that opens the AI chat pre‑loaded with the rule's intent (source,
destination, limit, naming) so the rule is created in one click
([PRO‑4598](https://linear.app/getsequence/issue/PRO-4598)). When it ships, the
link looks like:

```
https://app.getsequence.io/chat?setup=<url-encoded rule context>   # TODO: replace once PRO-4598 ships
```

**Manual fallback (works today).** In the dashboard: **Rules → New rule →**
Manual trigger → action "Transfer" from the company account to the contractor's
account → set the **max amount** → name it (e.g. `Reimburse — Alex Rivera`) →
save → **enable** it.

**c. Capture each `rule_id`** (it's in the rule's URL / detail page). You'll put
these in the workflow's config in step 5.

**d. Generate an API key.** Settings → API Keys → new key. Give it the scopes
this playbook uses, each bound to the relevant rules:
- `TRIGGER_RULES` (the payout rules) — to dry‑run and fire,
- `READ_RULES` (the payout rules) — to poll executions.

Use a dedicated key for this automation (e.g. `n8n-reimbursement`). Store it
securely; you'll paste it into an n8n credential, never into the workflow JSON.

---

## 2. Slack app

Create the bot that watches the channel and posts approvals.

1. Go to <https://api.slack.com/apps> → **Create New App → From an app
   manifest**, pick your workspace, and paste in
   [`slack-app-manifest.yaml`](./slack-app-manifest.yaml).
2. **Install to Workspace**, then copy the **Bot User OAuth Token** (`xoxb-…`).
3. **Invite the bot** to your expenses channel: `/invite @expense-bot`.
4. You'll set the event Request URL in step 4 (after import, when n8n gives you
   the trigger's Production URL).

Approve/Reject is handled by n8n's **Send and Wait** Slack operation, which uses
n8n's own resume webhook — so you do **not** need Slack "Interactivity"
configured.

---

## 3. LLM key (OpenRouter)

The receipt reader calls [OpenRouter](https://openrouter.ai) so you can use any
vision‑capable model with one key.

1. Create an OpenRouter account and an **API key**.
2. Pick a model and set it in the **Load Config** node (`llm.model`). Defaults to
   `anthropic/claude-sonnet-4`; `openai/gpt-4o` or `google/gemini-2.5-pro` work
   too — any model that accepts image input.

---

## 4. Import the workflow & connect credentials

1. In n8n: **Workflows → Import from File →**
   [`workflow.json`](./workflow.json).
2. Create the four credentials the nodes reference and attach them:
   | Credential (n8n type) | Used by | Value |
   | --- | --- | --- |
   | **Slack** (`slackApi`, access token) | Slack Trigger + all Slack message nodes | Bot token `xoxb-…` |
   | **Slack Bot Token (xoxb)** (`httpHeaderAuth`) | *Get File Info*, *Download Receipt* | Header `Authorization` = `Bearer xoxb-…` |
   | **OpenRouter** (`httpHeaderAuth`) | *Read Receipt (OpenRouter)* | Header `Authorization` = `Bearer <openrouter-key>` |
   | **Sequence API** (`httpHeaderAuth`) | *Dry Run*, *Fire Live*, *Poll …* | Header `Authorization` = `Bearer <sequence-key>` |

   > Slack file downloads need the bot token as a raw `Authorization: Bearer`
   > header, which is why there's a separate header‑auth credential in addition
   > to the Slack credential.
3. Open the **Receipt Posted (Slack Trigger)** node, copy its **Production URL**,
   and paste it into your Slack app's **Event Subscriptions → Request URL**.
   Confirm Slack verifies it (✓). Set the channel filter on the trigger to your
   expenses channel.

---

## 5. Fill in your config

Open the **Load Config** node and edit the `config` object — it's the single
source of truth (a copy lives in
[`config.example.json`](./config.example.json) for reference):

- `slack.expenseChannelId`, `slack.approverUserId` — the channel and the manager
  who approves.
- `policy.categories` — allowed categories and per‑category **max in cents**
  (`7500` = $75.00). Out‑of‑policy receipts are rejected in‑thread; the manager
  is never pinged.
- `roster` — map each contractor's **Slack user ID** to their `name` and payout
  `ruleId` from step 1c.

Secrets are **not** in this node — they're the credentials from step 4.

---

## 6. Safe test run (simulation)

Before real money can move, prove the path end‑to‑end with a dry run:

1. With the workflow open, click **Execute Workflow** (or post a real receipt in
   the channel). After the first run, **pin** the *Receipt Posted* trigger
   output and confirm the Slack field names in **Load Config**
   (`file_id`/`channel`/`user`/`event_ts`) match what your event actually sends —
   adjust the `trigger` mapping there if needed.
2. Approve the request in Slack. The **Dry Run (Simulation)** node calls
   `POST /rules/{id}/trigger` with `simulation: true`; **Poll Sim Execution**
   reads it back. A simulation never moves money. Only if the simulation is
   clean (`status: EXECUTED`, no failed transfers, conditions met) does
   **Fire Live Reimbursement** run.
3. To keep it fully dry while testing, you can temporarily point the live node at
   simulation too, or test against a rule with a tiny limit.

---

## 7. Go live

1. Toggle the workflow **Active**. New receipts in the channel now trigger it.
2. The flow is **human‑in‑the‑loop**: nothing pays without a manager Approve, and
   the *Request Manager Approval* node's **Limit Wait Time** caps the wait at
   1 day (an unanswered request resolves as not‑approved rather than auto‑paying).
   Tighten that, or add a reminder/escalation branch, to taste.

---

## How money moves (and stays safe)

- **Fire a rule, never an ad‑hoc transfer.** Each contractor has a pre‑created,
  human‑enabled payout rule with a max amount. The playbook only passes
  `executeAmount` (the receipt total, in cents).
- **Dry‑run gate.** Live fire happens only after a clean simulation.
- **Idempotency.** Dry and live calls use distinct, receipt‑stable
  `Idempotency-Key`s (`dry-<fileId>` / `live-<fileId>`) so retries never
  double‑pay.
- **Amounts are integer cents.** `$75.00` → `7500`. Minimum transfer is `$1.00`.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `RULE_DEACTIVATED` on trigger | The payout rule isn't enabled in the dashboard. |
| `MAXIMUM_AMOUNT_EXCEEDED` | Receipt amount is above the rule's max — raise the limit or reject. |
| `ACCESS_DENIED` | API key missing `TRIGGER_RULES`/`READ_RULES` for that rule. |
| `IDEMPOTENCY_KEY_MISMATCH` | Same key reused with a different body — keys reset after 24h. |
| Trigger never fires | Bot not invited to the channel, or Event Request URL not verified. |
| Receipt fields empty | Model isn't vision‑capable, or the image failed to download (check the Slack header‑auth credential). |
| Always takes the "rejected" branch after approving | The *Approved?* node reads `$json.data.approved` from Send‑and‑Wait; if your n8n version names it differently, pin the *Request Manager Approval* output and update the condition. |
