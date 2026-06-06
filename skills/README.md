# Bithumb Agent Skills

Pre-built skills for AI agents to operate Bithumb via the `bithumb` CLI. Each skill is a self-contained Markdown file with YAML frontmatter that tells the agent when to activate and how to execute tasks.

## Skills

| Skill | Description | Auth Required |
|-------|-------------|:-------------:|
| [`bithumb-market`](bithumb-market/SKILL.md) | Public market data: prices, order books, candles, trades, warnings, notices, fees | No |
| [`bithumb-trade`](bithumb-trade/SKILL.md) | Order management: single orders, batch orders, TWAP orders | Yes |
| [`bithumb-account`](bithumb-account/SKILL.md) | Account operations: balances, order chance, wallet status, API keys | Yes |
| [`bithumb-deposit`](bithumb-deposit/SKILL.md) | Deposit management: addresses, history, KRW deposit | Yes |
| [`bithumb-withdraw`](bithumb-withdraw/SKILL.md) | Withdrawal management: crypto/KRW withdrawal, cancellation, allowed addresses | Yes |
| [`bithumb-system`](bithumb-system/SKILL.md) | System: audit logs, diagnostics | Partial |

## Requirements

- `bithumb` CLI installed (`npm install -g @bithumb-official/bithumb-cli`)
- For authenticated skills: API credentials configured via environment variables:
  ```bash
  export BITHUMB_ACCESS_KEY=your_key
  export BITHUMB_SECRET_KEY=your_secret
  ```
  Or passed per-command via flags (`--access-key` / `--secret-key`). Flags take
  priority over environment variables, but may be visible in shell history — prefer
  environment variables for sensitive credentials.

## Skill Format

Each skill is a Markdown file with a YAML frontmatter header:

```yaml
---
name: skill-name
description: "Trigger description for the AI agent routing system."
---
```

The `description` field is used by the agent routing system to decide when to activate the skill. Skills with `references/` subdirectories use `{baseDir}` as a runtime-resolved path variable pointing to that skill's directory.

## CLI Command Coverage (42 commands)

| Category | Commands | Skill |
|----------|:--------:|-------|
| Market data | 11 | `bithumb-market` |
| Trade (single + batch) | 6 | `bithumb-trade` |
| Trade (TWAP) | 3 | `bithumb-trade` |
| Account | 4 | `bithumb-account` |
| System | 3 | `bithumb-system` |
| Deposit | 7 | `bithumb-deposit` |
| Withdrawal | 8 | `bithumb-withdraw` |

## Attribution

This project is inspired by and based in part on [OKX Agent Trade Kit](https://github.com/okx/agent-trade-kit), which is licensed under the MIT License. Copyright (c) 2026 OKX.

## License

MIT
