# Setup — Profit First (n8n)

Time: ~30 min. You'll set up Sequence (buckets + allocation rules + API key) and
Slack, import the workflow, fill in your config, do a safe simulated run, then
schedule it hourly.

Work top to bottom.

---

## 1. Sequence: buckets, allocation rules, API key

**a. Create the bucket pods.** In the [dashboard](https://app.getsequence.io),
create a pod per bucket — **Owner's Pay, Profit, Tax, Operating Expenses** (or
your own set) — and identify the **income account** deposits land in.

**b. Create one allocation rule per preset.** Each preset (Balanced, Lean, …) is
a **rule that splits incoming money across your pods by the preset's
percentages**. The rule does the split math and rounding — the playbook only
fires it with the deposit amount.

> Each rule must:
> - take the **income account** as its source and the **bucket pods** as
>   destinations, splitting by **percentage** (e.g. Balanced = Owner's Pay 50% /
>   Tax 30% / Profit 20%);
> - be **MANUAL** trigger type (fired on demand by the API);
> - be **enabled by a human** — the API can't enable rules; a disabled rule
>   returns `RULE_DEACTIVATED`.

**Guided rule creation (preferred — coming soon).** A **deep link** will open the
Sequence AI chat pre‑loaded with the split intent (income source, pods,
percentages, name) so each rule is created in one click
([PRO‑4598](https://linear.app/getsequence/issue/PRO-4598)):

```
https://app.getsequence.io/chat?setup=<url-encoded rule context>   # TODO: replace once PRO-4598 ships
```

**Manual fallback (works today).** Rules → New rule → Manual trigger → action
that **splits by percentage** from the income account to each pod → name it
(e.g. `Allocate — Balanced 50/30/20`) → save → **enable**. Repeat for each
preset.

**c. Capture each rule's `rule_id`** and the **income account ID** (both in their
dashboard URLs). They go into the config in step 4.

**d. Generate an API key** (Settings → API Keys) scoped to:
- `READ_TRANSFERS` (the income account) — to detect new deposits,
- `TRIGGER_RULES` (the allocation rules) — to dry‑run and fire,
- `READ_RULES` (the allocation rules) — to poll executions.

Use a dedicated key (e.g. `n8n-profit-first`). Store it in an n8n credential.

---

## 2. Slack

1. Create a Slack app (bot) and install it; copy the **Bot User OAuth Token**
   (`xoxb-…`). Minimum scope: `chat:write` (plus `channels:read` if you target a
   channel by name). Invite the bot to your target channel.
2. In n8n, create a **Slack** credential (`slackApi`) from that token.

Approve/choose is handled by n8n's **Send and Wait** operation (its own resume
webhook), so no Slack "Interactivity" config is needed.

> **WhatsApp instead of Slack?** Swap the *Ask Allocation* and the three reply
> nodes for a **Twilio (WhatsApp)** node + a Wait‑for‑webhook resume. The rest of
> the flow is identical. Keep the same "nothing moves without a pick" behavior.

---

## 3. Import the workflow & connect credentials

1. n8n → **Workflows → Import from File →** [`workflow.json`](./workflow.json).
2. Attach credentials:
   | Credential (n8n type) | Used by | Value |
   | --- | --- | --- |
   | **Sequence API** (`httpHeaderAuth`) | *List New Income*, *Dry Run*, *Fire Live*, *Poll …* | Header `Authorization` = `Bearer <sequence-key>` |
   | **Slack** (`slackApi`) | *Ask Allocation* + all reply nodes | Bot token `xoxb-…` |

---

## 4. Fill in your config

Open **Load Config** and edit the `config` object (mirror in
[`config.example.json`](./config.example.json)):

- `incomeAccountId` — the account to watch for deposits.
- `slack.channelId` (and `userId`) — where prompts/confirmations go.
- `presets` — one entry per preset: its `ruleId` (from step 1c) and the
  `buckets` map of bucket → **% for display** (the rule does the real split).
- `resurfaceMinutes` — how long a skipped/unanswered deposit waits before being
  re‑surfaced (default 180). `lookbackHours` — how far back each poll looks
  (default 25, giving overlap so nothing is missed between hourly runs).

> **Keep the dropdown in sync.** If you rename or add presets here, update the
> options in the **Ask Allocation** node to match (it lists preset names +
> "Skip for now").

---

## 5. Safe test run (simulation)

1. Trigger a deposit into the income account (or wait for a real one), then click
   **Execute Workflow**.
2. After the first run, **pin** the *List New Income* output and confirm the
   deposit fields (`id`, `amountInCents`, `source.name`) look right, and that
   *Ask Allocation* submits a value your *Resolve Choice* node reads (it looks
   for `data.Allocation` — adjust if your n8n labels the field differently).
3. Pick a preset. **Dry Run (Simulation)** calls `POST /rules/{id}/trigger` with
   `simulation: true`; **Poll Sim Execution** reads it back. A simulation never
   moves money. Only a clean simulation proceeds to **Fire Live Allocation**.

---

## 6. Go live

1. **Save** the workflow and toggle it **Active** (saving + active is what makes
   dedupe persist — see below). It now runs **every hour**.
2. It's **human‑in‑the‑loop**: nothing allocates without your pick. A deposit
   you skip or don't answer stays **pending** and is re‑surfaced on a later run
   (after `resurfaceMinutes`) — never auto‑allocated.

---

## How it stays correct

- **Idempotent / no double‑processing.** Each deposit's status
  (`pending` / `allocated`) is tracked in the workflow's **static data**
  (`$getWorkflowStaticData`). A deposit is prompted once and allocated once. The
  workflow must be **saved and active** for static data to persist across runs.
- **One deposit per tick.** Each hourly run handles a single deposit (newest
  unseen, else the oldest pending past its interval); the hourly cadence drains
  the backlog. For high deposit volume, use the agent‑guided variant or split
  the allocation into a sub‑workflow.
- **Fire a rule, not transfers.** The rule owns the split + rounding; the
  playbook passes only `executeAmount`.
- **Dry‑run gate + idempotency keys.** Live fire only after a clean simulation;
  `dry-<depositId>` / `live-<depositId>` keys mean retries never double‑allocate.
- **Partial‑failure surfaced.** The confirmation flags `transfersFailed > 0` from
  the execution rather than hiding it. (If a preset maps to **multiple** rules,
  duplicate the dry‑run → fire branch per rule, or use the agent‑guided variant
  which loops over `ruleIds` and aggregates failures.)
- **Amounts are integer cents.** Minimum transfer is `$1.00`.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Same deposit prompted twice | Workflow not saved/active, so static data didn't persist. |
| `RULE_DEACTIVATED` | The allocation rule isn't enabled in the dashboard. |
| `ACCESS_DENIED` | Key missing `READ_TRANSFERS` / `TRIGGER_RULES` / `READ_RULES` for the resource. |
| Choice always reads as "skip" | *Resolve Choice* reads `data.Allocation`; pin *Ask Allocation*'s output and update the key if your n8n names the field differently. |
| No deposits ever found | Wrong `incomeAccountId`, or deposits aren't `MONEY_IN` + `COMPLETE` on that account yet. |
