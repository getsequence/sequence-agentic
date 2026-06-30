# Contractor Expense Reimbursement via Slack

> Receipt in → reimbursement out. A contractor drops a receipt photo into a
> Slack channel; the bot reads it, checks it against your expense policy, asks
> the right manager to approve, and on one tap reimburses the contractor
> straight to their account — with a confirmation posted back in the thread.

No expense forms, no spreadsheets, no manual bank transfers, no chasing
approvals over email. The same pattern generalizes to **any approve‑then‑pay
flow**.

## How it works

1. A contractor posts a **receipt image** to the expenses Slack channel.
2. A **multimodal LLM** reads the image and extracts `amount`, `vendor`,
   `category`, and `date`.
3. The receipt is **validated against your expense policy** (category allowed,
   amount under the per‑category limit). Out‑of‑policy receipts are rejected
   in‑thread with the specific reason — **no manager is pinged**.
4. The contractor is resolved from config (Slack user → their payout
   **`rule_id`**).
5. An **approval request** is posted to the manager in Slack with the parsed
   details and **Approve / Reject** buttons.
6. On **Approve**, the bot **dry‑runs** the payout rule (`simulation: true`),
   and if clean, **fires it live** with the receipt amount.
7. The bot replies in the original thread: **amount reimbursed + transfer ID**.

Money only ever moves by **triggering a pre‑created Sequence rule** (source =
company account, destination = contractor account, with a limit) — never a
one‑time ad‑hoc transfer. **No money moves without an explicit manager
approval.**

## Two ways to run it

| Variant | For | Folder |
| --- | --- | --- |
| **n8n** | You want an importable, hosted workflow | [`n8n/`](./n8n) |
| **Agent‑guided** | You want your own coding agent (Claude Code, etc.) to set it up and run it locally | [`agent-guided/`](./agent-guided) |

Both variants follow the same flow and the same setup; pick the runtime that
fits you. The agent‑guided variant ends by running the playbook periodically on
your machine, with a clear note about moving it to a hosted runtime if you want
it to keep running while your agent is offline.

## What you'll need

- A **Sequence** account with a company source account/pod and a connected
  contractor destination account — plus a **payout rule** per contractor and an
  **API key**. See the setup guide in either variant.
- A **Slack** workspace where you can install an app (bot token + the expenses
  channel).
- An **LLM** that can read images. The n8n variant calls
  [OpenRouter](https://openrouter.ai) so you can pick any vision‑capable model
  (Anthropic, OpenAI, Google, …) with one key; the agent‑guided variant uses
  your coding agent's own multimodal model, so no separate key is needed.

## Sequence API surface used

| Call | Purpose |
| --- | --- |
| `POST /rules/{id}/trigger` with `simulation: true` | Dry‑run the payout before any real money moves |
| `POST /rules/{id}/trigger` (live) with `executeAmount` | Fire the reimbursement for the receipt amount |
| `GET /rules/{ruleId}/executions/{id}` | Poll the execution for status + resulting `transferIds` |

Base URL: `https://api.getsequence.io/platform/v1` · Auth:
`Authorization: Bearer <api_key>` ·
[API reference](https://app.getsequence.io/api/platform/).

> **Note on the spec.** Earlier drafts referenced a `POST /rules/{id}/dryRun`
> endpoint. The live API does dry‑runs through the **same trigger endpoint**
> with `simulation: true`; this playbook uses that.

---

New to Sequence? [Sign up](https://app.getsequence.io) ·
[API docs](https://app.getsequence.io/api/platform/) ·
[Discord `#api`](https://discord.gg/kHP4AJpcyA).
