# Agent‑guided setup — Contractor Expense Reimbursement

This file is written **for your coding agent** (Claude Code, Cursor, etc.).
Hand it the whole file and say: *"Set this up for me and run it."* The agent
does the build; the few steps only a human can do (creating accounts, approving
OAuth, enabling rules) are called out as **[human]**.

> **You — the agent — are the receipt reader.** You're multimodal, so you read
> the receipt image yourself. There is no separate vision service or LLM key in
> this variant.

---

## What you're building

A small local program that:

1. Polls a Slack channel for newly posted **receipt images**.
2. For each new one: downloads it, **reads the amount / vendor / category /
   date yourself**, and validates it against the expense **policy** in config.
3. Out‑of‑policy → replies in‑thread with the reason and stops (no approval
   requested).
4. In‑policy → resolves the contractor (Slack user → payout `rule_id`) and posts
   an **approval request** to the manager; waits for an explicit **Approve**.
5. On approve → **dry‑runs** the payout rule (`simulation: true`), and only if
   clean **fires it live** with the receipt amount, then confirms in‑thread with
   the transfer ID.

**Money only ever moves by triggering a pre‑created Sequence rule. Nothing pays
without a manager approval. Never double‑process a receipt.**

---

## Prerequisites (do these first)

### Sequence — accounts, payout rules, API key
- **[human]** In the [dashboard](https://app.getsequence.io): create/choose the
  company **source account**, connect each contractor's destination account.
- **[human]** Create **one payout rule per contractor**: source = company
  account, destination = contractor, **MANUAL** trigger, with a **max amount**
  (per‑payout ceiling). **Enable** each rule — the API can't enable rules, and a
  disabled rule returns `RULE_DEACTIVATED`.
  - *Guided (preferred, coming soon):* a deep link that opens the Sequence AI
    chat pre‑filled with the rule's intent
    ([PRO‑4598](https://linear.app/getsequence/issue/PRO-4598)) —
    `https://app.getsequence.io/chat?setup=<url-encoded context>` *(TODO: replace
    once PRO‑4598 ships)*.
  - *Manual fallback (today):* Rules → New rule → Manual → Transfer company →
    contractor → set max amount → name it → save → enable.
- **[human]** Capture each **`rule_id`** (in the rule URL) → into `config.json`
  `roster`.
- **[human]** Generate an **API key** (Settings → API Keys) scoped to
  `TRIGGER_RULES` + `READ_RULES` on those rules. Export it as
  `SEQUENCE_API_KEY`.

### Slack — bot
- **[human]** Create an app from the manifest in
  [`../n8n/slack-app-manifest.yaml`](../n8n/slack-app-manifest.yaml) (the bot
  scopes are identical; you can ignore the `event_subscriptions` block — this
  variant **polls** instead of receiving events). Install it, copy the bot token
  to `SLACK_BOT_TOKEN`, and `/invite` the bot to the expenses channel.

### LLM
- None. You read the receipts.

---

## Config

Copy [`config.example.json`](./config.example.json) → `config.json` and fill in:
`slack.expenseChannelId`, `slack.approverUserId`, the `policy` (allowed
categories + per‑category **max in cents** — `7500` = $75.00), and the `roster`
(Slack user ID → `{ name, ruleId }`). Secrets stay in env vars
(`SEQUENCE_API_KEY`, `SLACK_BOT_TOKEN`).

Keep **operational state local** (`state.json`): the set of already‑processed
file IDs (dedupe) and any pending approvals. Do **not** put run‑state in a SaaS
store.

---

## Build it (agent: implement this)

Use any language/runtime you like. The contract:

### Slack calls (REST, `Authorization: Bearer $SLACK_BOT_TOKEN`)
- **Find new receipts:** `GET conversations.history?channel=<id>` (or
  `files.list?channel=<id>`), newest first. Keep a cursor / processed‑set in
  `state.json`; only handle files you haven't seen, with an image mimetype.
- **Download the image:** `GET file.url_private` with the bearer header; read the
  bytes and look at them yourself.
- **Reply in thread:** `POST chat.postMessage` with `thread_ts` of the receipt
  message.
- **Approval:** `POST chat.postMessage` to the manager (DM or channel) with the
  parsed details; then poll `GET reactions.get` on that message and treat
  `slack.approveEmoji` from the **approver** as Approve, `rejectEmoji` as Reject.
  (A threaded `approve`/`reject` reply works too — your choice.)

### Policy check (you, in code/logic)
Reject if: amount unreadable, category not in `policy.categories`, or
`amount_cents > categories[category].maxAmountCents`. On reject, post the
specific reason in‑thread and **do not** ping the manager.

### Sequence calls (REST, `Authorization: Bearer $SEQUENCE_API_KEY`)
Base URL from `config.sequence.baseUrl`. All amounts are **integer cents**.

1. **Dry run:**
   `POST /rules/{ruleId}/trigger`
   headers: `Idempotency-Key: dry-<fileId>`, `Content-Type: application/json`
   body: `{ "simulation": true, "executeAmount": <amount_cents> }`
   → `202 { data: { executionId } }`.
2. **Poll the dry run:** `GET /rules/{ruleId}/executions/{executionId}` with
   exponential backoff until `status` is terminal. Clean ⇔
   `status === "EXECUTED" && conditionsNotMet === false && transfersFailed === 0`.
   If not clean → reply in‑thread with the error and stop.
3. **Fire live:**
   `POST /rules/{ruleId}/trigger`
   headers: `Idempotency-Key: live-<fileId>` (distinct from the dry key),
   `Content-Type: application/json`
   body: `{ "executeAmount": <amount_cents> }`
   → `202 { data: { executionId } }`.
4. **Poll the live run** the same way; read `transferIds` from the execution.
5. **Confirm in‑thread:** "Reimbursed $X to <name> — transfer `<id>`."

Notes that matter:
- **Async + eventual consistency:** a freshly returned `executionId` may briefly
  `404`; retry with backoff before treating it as missing.
- **Idempotency:** reuse the *same* key on network‑error retries (24h window);
  dry and live use *different* keys because their bodies differ.
- **Errors:** handle `RULE_DEACTIVATED` (enable the rule),
  `MAXIMUM_AMOUNT_EXCEEDED` (over the rule limit), `ACCESS_DENIED` (key scope),
  `429 RATE_LIMIT_EXCEEDED` (honor `Retry-After`). Rate limit is 100 req/min/key.

### Required behavior (don't skip)
- **Idempotent:** record each processed file ID in `state.json`; never prompt or
  pay twice for the same receipt.
- **Human‑in‑the‑loop:** no live fire without an explicit Approve. A receipt with
  no response stays **pending** in `state.json` and is re‑surfaced on the next
  run — never auto‑paid. Use `approval.reminderAfterMinutes` /
  `escalateAfterMinutes` to nudge, then escalate.
- **Partial failure is reported, not swallowed.**

---

## Test it (simulation first)

Run once and walk a real receipt through to the approval. Confirm the **dry
run** path works end‑to‑end *before* enabling live fire — to stay fully dry while
testing, send `simulation: true` on the live call too, or point at a rule with a
tiny limit. A simulation never moves money.

---

## Run it periodically

Once it works, **run it on a schedule on this machine** — e.g. a loop every
1–5 minutes, or a cron entry invoking the script. Each tick: fetch new receipts,
re‑surface pending approvals, advance anything that's now approved.

> ⚠️ **Offline‑runtime warning.** This runs on **your machine via your coding
> agent** — it only works while that agent/process is up. If you close it, the
> bot stops watching Slack and approvals won't progress. If you want it running
> **24/7 independent of your machine**, talk to your agent about moving it to a
> hosted runtime — import the [`../n8n`](../n8n) workflow into a hosted n8n, or
> deploy this script to a cron host / small always‑on server. Keep `state.json`
> (or its equivalent) on durable storage there so dedupe and pending approvals
> survive restarts.
