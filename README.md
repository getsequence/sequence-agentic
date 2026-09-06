# Sequence for AI agents

[Sequence](https://getsequence.io) is a financial automation platform for businesses. This repo
holds the public, agent-facing pieces: a **skill** that teaches an agent to use Sequence well, and
**playbooks** that wire it into real workflows.

Sequence runs a hosted MCP server, nothing to install or self-host:

```
https://app.getsequence.io/api/mcp
```

Authentication is OAuth. Sign in through your browser with your existing Sequence account.

## What your agent can do

Once connected, an agent can answer questions and take action across your business money:

- **Cash position** across bank accounts, pods (savings buckets), and connected outside accounts
- **History** — transfers, payments, card spending, and individual transactions
- **Automations** — list, inspect, and edit rules that move money on a trigger (split incoming pay,
  set aside taxes, top up a savings pod, or pay down a card on a schedule)
- **Money movement** — propose a one-off transfer, or run an existing rule now

## How money movement works

**Nothing moves without you.** Over OAuth, every money movement your agent initiates is a
*proposal*: you get an approval link, and the transfer runs only after you approve it in the
Sequence app. Unapproved requests expire after 24 hours.

Read access and automation authoring work as usual; the money itself stays behind that gate. Your
agent can also dry-run any movement first, so you see exactly what would happen before anything is
proposed.

For unattended execution, or to cap how much an agent can move, use
[API keys](https://app.getsequence.io/account/api-keys).

## Connect your agent

The [Connect your agent](https://app.getsequence.io/agents) page in the Sequence app has copy-paste
setup for every supported client. Common ones:

### Hermes

**Hermes Desktop** — open [Connect your agent](https://app.getsequence.io/agents), pick Hermes, and
click **Add to Hermes**.

**Hermes CLI**

```bash
hermes mcp add sequence --url https://app.getsequence.io/api/mcp --auth oauth
```

A browser opens for sign-in, then you pick which tools to enable. What's offered is already limited
to what your Sequence role allows, and money movement needs your approval either way. Already in a
chat? Run `/reload-mcp` to refresh your tool list.

### Claude Code

```bash
claude mcp add --transport http sequence https://app.getsequence.io/api/mcp
```

### Other MCP clients

Most clients need only the URL above with OAuth selected. Cursor, ChatGPT, and Grok each have their
own connector form; the in-app page lists the exact fields for each.

## The skill

[`skills/sequence-finance`](skills/sequence-finance) teaches an agent how to use Sequence properly:
when to reach for it, the vocabulary your business uses, and the approval contract above. Install it
alongside the connection — it works across coding agents:

```bash
npx skills add getsequence/sequence-agentic
```

That detects your agent and installs to it. To name the target and scope explicitly (useful if you
have more than one agent):

```bash
npx skills add getsequence/sequence-agentic -a hermes-agent -g
```

Some agents (including Hermes) don't pass a server's own operating instructions to the model. Without
the skill, an agent can connect successfully and still misunderstand what it did with your money. The
skill is small and worth installing.

## Playbooks

[`playbooks/`](playbooks) holds complete, opinionated workflows built on Sequence. Each includes
setup instructions plus an agent-guided and an n8n implementation:

- **[Profit First](playbooks/profit-first)** — every deposit is split into fixed-percentage buckets
  (Owner's Pay, Profit, Tax, Operating Expenses); you confirm each allocation from Slack.
- **[Contractor reimbursement](playbooks/contractor-reimbursement)** — a contractor drops a receipt
  photo in Slack; it's checked against your expense policy, the right manager approves, and the
  payout goes out on one tap.

## License

MIT. See [LICENSE](LICENSE).
