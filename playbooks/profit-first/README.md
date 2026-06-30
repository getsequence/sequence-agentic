# Profit First

> Money in → split into your profit‑first buckets, you confirm each time.

This is the **Profit First** method (Mike Michalowicz): every dollar of income is
split into fixed‑percentage buckets — **Owner's Pay, Profit, Tax, Operating
Expenses** — instead of pooling in one account where it all feels spendable.

The playbook watches your income account. When money lands, it pings you on
Slack: *"You received $1,000 from Acme — allocate as **Balanced (50/30/20)** or
**Lean (60/20/20)**?"* with the exact dollar breakdown for each option. You pick
one and the money is split across your buckets instantly. **You stay in control
of every deposit — nothing moves until you choose.**

## How it works (hourly)

1. Every hour, check Sequence for **new incoming funds** on your income account.
2. For each new deposit, show the per‑bucket **dollar breakdown** under each
   configured preset.
3. Send a Slack message: *"You received $X from Y — allocate as [A] or [B]?"*
   (plus **Skip for now**).
4. **Wait for your choice. Nothing moves without an explicit pick.**
5. On choice: **dry‑run** the allocation rule (`simulation: true`), then **fire
   it live** with the deposit amount.
6. Reply with confirmation + transfer IDs.

The allocation is executed by **triggering a pre‑created Sequence rule** — and
**the rule itself does the split math and rounding**. The playbook never
computes or moves the final amounts; it just fires the rule with the deposit
amount.

## Two ways to run it

| Variant | For | Folder |
| --- | --- | --- |
| **n8n** | An importable, always‑on hosted workflow | [`n8n/`](./n8n) |
| **Agent‑guided** | Your own coding agent sets it up and runs it locally | [`agent-guided/`](./agent-guided) |

## Presets (shipped defaults, fully editable)

| Preset | Owner's Pay | Tax | Profit |
| --- | --- | --- | --- |
| **Balanced (50/30/20)** | 50% | 30% | 20% |
| **Lean (60/20/20)** | 60% | 20% | 20% |

Buckets and percentages are user‑configurable; add your own named presets (and
the matching allocation rule) any time.

## Required behavior

- **Auto‑detect & idempotent** — polls hourly and **never double‑processes** a
  deposit (no duplicate prompt, no duplicate allocation). Processed deposits are
  tracked in persistence (mechanism is the developer's choice — kept local, not
  in a SaaS store).
- **Human‑in‑the‑loop** — no allocation fires without an explicit choice. A
  deposit with no reply stays **pending** and is re‑surfaced; it is never
  auto‑allocated.
- **Fire rules, not transfers** — the rule handles the split + rounding.
- **Partial‑failure reporting** — if a preset fires multiple rules, individual
  failures are reported, not silently dropped.
- **Safe test run** — dry‑run via Sequence's `simulation` flag before real money
  moves.

## Sequence API surface used

| Call | Purpose |
| --- | --- |
| `GET /accounts/{id}/transfers?direction=MONEY_IN&status=COMPLETE&from=…` | Detect new incoming funds since the last run |
| `POST /rules/{id}/trigger` with `simulation: true` | Dry‑run the allocation |
| `POST /rules/{id}/trigger` (live) with `executeAmount` | Fire the split for the deposit amount |
| `GET /rules/{ruleId}/executions/{id}` | Poll for status + resulting `transferIds` |

Base URL: `https://api.getsequence.io/platform/v1` · Auth:
`Authorization: Bearer <api_key>` ·
[API reference](https://app.getsequence.io/api/platform/).

> Earlier drafts mentioned a `POST /rules/{id}/dryRun` endpoint; the live API
> does dry‑runs through the **same trigger endpoint** with `simulation: true`.

---

New to Sequence? [Sign up](https://app.getsequence.io) ·
[API docs](https://app.getsequence.io/api/platform/) ·
[Discord `#api`](https://discord.gg/kHP4AJpcyA).
