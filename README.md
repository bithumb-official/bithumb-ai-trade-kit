English | [한국어](./README.ko.md)

# Bithumb AI Trade Kit

A toolkit for AI agents to handle market data, account queries, orders, deposits, and withdrawals via the Bithumb API.

Includes an MCP server, a terminal CLI, and Skills for AI agents — compatible with Claude, Cursor, VS Code, Windsurf, Codex, and more.

---

## What's in the Kit

| Component | Package | Description |
|---|---|---|
| **MCP** | [`@bithumb-official/bithumb-mcp`](https://www.npmjs.com/package/@bithumb-official/bithumb-mcp) | Connect Claude, Cursor, VS Code, Windsurf, and other MCP-compatible clients to Bithumb |
| **CLI** | [`@bithumb-official/bithumb-cli`](https://www.npmjs.com/package/@bithumb-official/bithumb-cli) | Run market queries, orders, deposits, and withdrawals from the terminal |
| **Skills** | `skills/` | Workflow guides that structure how AI agents execute tasks |

---

## MCP

No global install needed. Add the server to your MCP client config and it runs automatically via `npx`.

**Claude Code** (`.mcp.json` in project root, or `~/.claude.json` for global):

```json
{
  "mcpServers": {
    "bithumb": {
      "command": "npx",
      "args": ["-y", "@bithumb-official/bithumb-mcp", "--modules", "all"],
      "env": {
        "BITHUMB_ACCESS_KEY": "${BITHUMB_ACCESS_KEY}",
        "BITHUMB_SECRET_KEY": "${BITHUMB_SECRET_KEY}"
      }
    }
  }
}
```

For more details, see the [MCP guide](https://apidocs.bithumb.com/docs/mcp).

---

## CLI

Install:
```bash
npm install -g @bithumb-official/bithumb-cli
```

Examples:
```bash
bithumb market ticker KRW-BTC
bithumb account assets
bithumb trade place --market KRW-BTC --side bid --order-type limit --price 50000000 --volume 0.001
bithumb system diagnose
```

For more details, see the [CLI guide](https://apidocs.bithumb.com/docs/cli).

---

## Skills

Supported agents are auto-detected. Install all Skills with:

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit --all
```

For more details, see the [Skills guide](https://apidocs.bithumb.com/docs/skills).

---

## Authentication

Price and market data queries work without an API Key. Account queries, orders, deposits, and withdrawals require a Bithumb API Key.

Set up an API Key with the required permissions on the [Bithumb API management page](https://www.bithumb.com/react/api-support/management-api), then set environment variables:

```bash
export BITHUMB_ACCESS_KEY="your_access_key"
export BITHUMB_SECRET_KEY="your_secret_key"
```

The CLI also supports named profiles in `~/.bithumb/config.toml`:

```bash
bithumb config init       # create config interactively
bithumb config use trading
```

---

## User Guides

| Topic | Link |
|---|---|
| Getting started | [Bithumb AI Trade Kit](https://apidocs.bithumb.com/docs/ai-트레이드-킷-시작하기) |
| MCP | [MCP](https://apidocs.bithumb.com/docs/mcp) |
| CLI | [CLI](https://apidocs.bithumb.com/docs/cli) |
| Skills | [Skills](https://apidocs.bithumb.com/docs/skills) |

---

## Acknowledgements

This project was inspired by the [OKX Agent Trade Kit](https://www.okx.com/agent-tradekit).

---

## License

MIT
