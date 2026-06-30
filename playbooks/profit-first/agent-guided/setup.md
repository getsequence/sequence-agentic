# Agent‑guided setup — Profit First

This file is written **for your coding agent** (Claude Code, Cursor, etc.). Hand
it the whole file and say: *"Set this up for me and run it."* The agent does the
build; steps only a human can do are marked **[human]**.

---

## What you're building

A small local program that, **every hour**:

1. Asks Sequence for **new income** on the income account since the last run.
2. For each new deposit (deduped), pings you on Slack with the **per‑bucket
   dollar breakdown** for each preset and a **Skip** option.
3. Waits for your pick. **Nothing moves without an explicit choice.**
4. On a pick: **dry‑runs** the preset's allocation rule(s), then **fires them
   live** with the deposit amount, and confirms with the transfer IDs.

**Fire rules, not transfers — the rule does the split + rounding. Never
double‑process a deposit. A skipped/unanswered deposit stays pending and is
re‑surfaced, never auto‑allocated.**

---

## Prerequisites (do these first)

### Sequence — pods, allocation rules, API key
- **[human]** In the [dashboard](https://app.getsequence.io): create a **pod per
  bucket** (Owner's Pay, Profit, Tax, OpEx…) and note the **income account**.
- **[human]** Create **one allocation rule per preset** — each splits the income
  across the pods **by percentage** (Balanced = 50/30/20, Lean = 60/20/20).
  **MANUAL** trigger, **enabled** (the API can't enable rules; disabled →
  `RULE_DEACTIVATED`). A preset may use more than one rule if you prefer.
  - *Guided (preferred, coming soon):* deep link that opens the Sequence AI chat
    pre‑filled with the split intent
    ([PRO‑4598](https://linear.app/getsequence/issue/PRO-4598)) —
    `https://app.getsequence.io/chat?setup=<url-encoded context>` *(TODO: replace
    once PRO‑4598 ships)*.
  - *Manual fallback (today):* Rules → New rule → Manual → split‑by‑percentage
    from income account → pods → name → save → enable.
- **[human]** Capture the **income account ID** and each **`rule_id`** → into
  `config.json`.
- **[human]** Generate an **API key** scoped `READ_TRANSFERS` (income account) +
  `TRIGGER_RULES` + `READ_RULES` (allocation rules). Export as
  `SEQUENCE_API_KEY`.

### Chat — Slack (or WhatsApp)
- **[human]** Create a Slack app with `chat:write`, install it, `/invite` the bot
  to your channel, and export the token as `SLACK_BOT_TOKEN`. (Or use WhatsApp via
  Twilio — same flow, different send/receive calls.)

---

## Config

Copy [`config.example.json`](./config.example.json) → `config.json`: set
`incomeAccountId`, the chat target, and `presets` (each with `ruleIds` + display
`buckets`). Keep **operational state local** (`state.json`) — the per‑deposit
status map for dedupe + re‑surfacing. Do **not** put run‑state in a SaaS store.

---

## Build it (agent: implement this)

Any language/runtime. The contract:

### Detect new income (Sequence, `Authorization: Bearer $SEQUENCE_API_KEY`)
`GET {baseUrl}/accounts/{incomeAccountId}/transfers` with query
`accountRole=destination&direction=MONEY_IN&status=COMPLETE&from=<ISO ts>&pageSize=100`.
Use `from = now − lookbackHours` (overlap so nothing slips between runs). Response
is `{ data: { items: [...], pagination } }`, newest first; each item has `id`,
`amountInCents`, `source.name`, `createdAt`. Page via `pagination.hasNextPage`.

### Dedupe + pick (you, in code)
Keep `state.json` as `{ <transferId>: { status, amountInCents, vendor, promptedAt, preset } }`.
Each run:
- New deposits (not in state) → prompt.
- `pending` deposits whose `promptedAt` is older than `resurfaceMinutes` → re‑prompt.
- `allocated` deposits → never touch.
Unlike the n8n variant (one deposit/tick), you can handle **all** new deposits in a run.

### Prompt + wait (Slack, `Authorization: Bearer $SLACK_BOT_TOKEN`)
Post `chat.postMessage`: *"You received $X from Y — allocate as [preset A] or
[preset B]?"* with each preset's per‑bucket breakdown
(`bucketCents = round(amountInCents × pct / 100)`, shown as dollars — display
only). Offer the presets + **Skip**. Collect the choice (buttons + interactivity,
or poll a reaction/threaded reply — your call). **No pick → leave it `pending`.**

### Allocate (Sequence). All amounts integer cents.
For the chosen preset, for **each** `ruleId`:
1. **Dry run:** `POST /rules/{ruleId}/trigger`, header `Idempotency-Key: dry-<depositId>-<ruleId>`, body `{ "simulation": true, "executeAmount": <amountInCents> }` → `202 { data: { executionId } }`.
2. **Poll** `GET /rules/{ruleId}/executions/{executionId}` (exp. backoff) until terminal. Clean ⇔ `status === "EXECUTED" && conditionsNotMet === false && transfersFailed === 0`.
3. If clean, **fire live:** same POST, header `Idempotency-Key: live-<depositId>-<ruleId>` (distinct key — different body), body `{ "executeAmount": <amountInCents> }`.
4. **Poll** the live execution; collect `transferIds`, `transfersFailed`, `errorMessage`.

**Partial failure:** fire/report each rule independently — one rule failing must
not silently drop the others. Mark the deposit `allocated` once its rules have
been processed, and report per‑rule results in the confirmation.

Notes that matter:
- **Async + eventual consistency:** a fresh `executionId` may briefly `404`; retry with backoff.
- **Idempotency:** reuse the *same* key on network‑error retries (24h window); dry vs live keys differ because bodies differ.
- **Errors:** `RULE_DEACTIVATED`, `ACCESS_DENIED`, `MAXIMUM_AMOUNT_EXCEEDED`, `429` (honor `Retry-After`; 100 req/min/key).

### Confirm (Slack)
Reply: *"Allocated $X from Y as <preset> — transfers `<ids>`"*, and flag any
`transfersFailed > 0` per rule.

### Required behavior (don't skip)
- **Idempotent:** never prompt or allocate the same deposit twice (`state.json`).
- **Human‑in‑the‑loop:** no live fire without an explicit pick; unanswered →
  `pending` → re‑surfaced; never auto‑allocated.
- **Partial‑failure reported, not swallowed.**

---

## Test it (simulation first)

Run once against a real (or test) deposit and walk it to a pick. Confirm the
**dry‑run** path is clean before allowing live fire — to stay fully dry while
testing, send `simulation: true` on the live call too, or use a tiny‑limit rule.
A simulation never moves money.

---

## Run it periodically

Once it works, **run it hourly on this machine** — a loop sleeping ~`pollMinutes`,
or a cron entry. Each tick: fetch new income, re‑surface pending deposits, advance
anything just chosen.

> ⚠️ **Offline‑runtime warning.** This runs on **your machine via your coding
> agent** — it only works while that agent/process is up. If you close it, new
> deposits won't be detected and pending picks won't progress. If you want it
> running **24/7 independent of your machine**, talk to your agent about moving it
> to a hosted runtime — import the [`../n8n`](../n8n) workflow into a hosted n8n,
> or deploy this script to a cron host / small always‑on server. Keep `state.json`
> (or its equivalent) on durable storage there so dedupe and pending deposits
> survive restarts.
