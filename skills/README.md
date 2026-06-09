English | [한국어](./README.ko.md)

# Bithumb Skills

A collection of Bithumb Skills that help AI agents execute tasks like market data queries and orders in a defined order. Register them with any Skills-compatible agent — Claude Code, Cursor, Codex, Amp, Cline, Windsurf, and more.

## Skills

| Skill | Domain                                                                                                                | Auth Required |
|---|-----------------------------------------------------------------------------------------------------------------------|:---:|
| [`bithumb-market`](bithumb-market/SKILL.md) | Access public market data, including prices, candles, orderbook, trades, notices, and fees                            | No |
| [`bithumb-account`](bithumb-account/SKILL.md) | View balances, trading availability, wallet deposit/withdrawal availability, blockchain sync, and API key information | Yes |
| [`bithumb-trade`](bithumb-trade/SKILL.md) | Place, cancel, and query spot orders, including limit, market, batch, and TWAP orders                                 | Yes |
| [`bithumb-deposit`](bithumb-deposit/SKILL.md) | Manage deposit addresses, deposit history, and KRW deposit requests                                                   | Yes |
| [`bithumb-withdraw`](bithumb-withdraw/SKILL.md) | Manage withdrawal requests, pending crypto-withdrawal cancellation, history, fees, and allowed addresses              | Yes |
| [`bithumb-system`](bithumb-system/SKILL.md) | Diagnose connectivity, credentials, audit logs, and supported capabilities                                            | No |

## Security

Skills can place real orders and execute withdrawals using your API credentials. Keep the following in mind:

- Never enter your API Key or Secret Key in an AI chat window, and never share them with others.
- Grant only the permissions your use case requires. For market data only, use read-only permissions; for trading, limit to trading permissions only.
- Store your API Key and Secret Key locally only. If you suspect exposure, revoke them immediately and generate new ones.
- Do not use Skills on shared or untrusted machines.
- For actions like orders and withdrawals, configure your agent to require explicit user confirmation before executing.

## Prerequisites

- Node.js 18 or later
- An AI agent that supports skills (Claude Code, Cursor, Codex, Amp, Cline, Windsurf, etc.)
- Bithumb CLI — skills call the `bithumb` command internally:
  ```bash
  npm install -g @bithumb-official/bithumb-cli
  ```
- Bithumb API Key (not required for `bithumb-market` and `bithumb-system`) — see [CLI documentation](https://apidocs.bithumb.com/docs/cli) for authentication setup.

## Installation

Supported agents are auto-detected. Install all 6 skills at once:

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit --all
```

To choose which skills to install and configure options interactively:

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit
```

The installer walks you through:

1. Selecting which skills to install
2. Confirming target agent(s) — auto-detected
3. Selecting scope: Project or User (global)
4. Choosing install method: **Symlink** (recommended) or Copy

After installation, verify with:

```bash
npx skills@latest list
```

## How It Works

Once installed, ask your agent in plain language:

> "What's the current KRW-BTC price?"
> "Check my balance."
> "Buy 100,000 KRW worth of BTC at market price."

For a market buy request, the agent selects `bithumb-trade` and executes the full workflow automatically:

1. `bithumb account order-chance --market KRW-BTC` — check balance and limits
2. `bithumb market ticker KRW-BTC` — check current price
3. Ask for your confirmation (market, side, amount)
4. `bithumb trade place --market KRW-BTC --side bid --order-type price --price 100000` — execute
5. `bithumb trade get --order-id <id>` — verify order status

Unlike MCP (which invokes individual tools), Skills define the entire workflow — from pre-checks and user confirmation to post-verification.

## Troubleshooting

**Skills not detected by the agent**

Restart the agent and run `npx skills@latest list` to confirm installation.

**`bithumb` command not found**

Check that the CLI is installed: `bithumb --version`

**Authentication errors**

Make sure `BITHUMB_ACCESS_KEY` and `BITHUMB_SECRET_KEY` are set. Run `bithumb system diagnose` to check authentication status.

## Related

- **npm (CLI)**: [@bithumb-official/bithumb-cli](https://www.npmjs.com/package/@bithumb-official/bithumb-cli)
- **npm (MCP)**: [@bithumb-official/bithumb-mcp](https://www.npmjs.com/package/@bithumb-official/bithumb-mcp)
- **Documentation**: [Bithumb AI Trade Kit docs](https://apidocs.bithumb.com/docs/skills)

## License

MIT
