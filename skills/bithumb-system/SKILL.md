---
name: bithumb-system
description: "Inspect Bithumb local audit logs (operation/trade history) and run connection and credential diagnostics. The diagnose command needs no auth to test credentials; audit reads local logs only. 빗썸 로컬 감사 로그 조회, 연결·인증 진단, 활성 모듈 상태 확인을 처리합니다. 감사 로그·거래 로그·시스템 진단·연결 상태 관련 요청에 사용하세요."
---

# Bithumb System CLI

Local audit logs and connection/auth diagnostics for the Bithumb CLI. All commands are **read-only**. `system diagnose` works even before credentials are valid — it helps diagnose auth problems.

## Skill Routing

- market data → `bithumb-market` | account/wallet/keys → `bithumb-account` | orders → `bithumb-trade` | deposits → `bithumb-deposit` | withdrawals → `bithumb-withdraw` | audit logs & diagnostics → `bithumb-system` (this skill)

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb system diagnose` | READ | Connection, auth, config, and module diagnostics |
| 2 | `bithumb system audit` | READ | Local audit logs (operation history) |

## Routing — Identify system action

| User intent | Command |
|---|---|
| Check connection / auth / config health | `bithumb system diagnose` |
| Review past operations (local log) | `bithumb system audit` |

> A request like `"사용 가능한 기능 알려줘"` / "what can you do?" is answered with the skill-routing list (market / account / trade / deposit / withdraw / system), **not** a tool call. To check which modules are enabled in a given environment, read the module status from `bithumb system diagnose`.

All commands here are read-only — run immediately, no confirmation or post-write verification needed.

## CLI Reference

### system diagnose — System Diagnostics

```bash
bithumb system diagnose
# → API: reachable | Auth: valid | Config: OK | Modules: all active
```

Checks API reachability, authentication status, TOML config validity, and module status. Other authenticated skills (account, trade, deposit, withdraw) reference this as their pre-flight connection/auth check. If it fails, stop and guide the user to set `BITHUMB_ACCESS_KEY` / `BITHUMB_SECRET_KEY`.

### system audit — Audit Logs

```bash
bithumb system audit [--tool <name>] [--level <INFO|WARN|ERROR|DEBUG>] [--limit <n>] [--since <timestamp>]
# Example: bithumb system audit --level ERROR --limit 10
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--tool` | No | - | Filter by tool/command name |
| `--level` | No | - | Filter: `INFO`, `WARN`, `ERROR`, `DEBUG` |
| `--limit` | No | 20 | Max entries to return (>=1) |
| `--since` | No | - | ISO 8601 timestamp; entries at or after this time |

## Cross-Skill Workflow — System health check

> User: "빗썸 연결 상태 확인해줘"

```
1. bithumb-system   bithumb system diagnose        → full diagnostics
2. bithumb-account  bithumb account wallet-status   → blockchain status
```

## Edge Cases

- **Audit logs are local**: `system audit` reads local log files, not Bithumb server-side history.
- **Diagnose without auth**: `system diagnose` runs even when credentials are missing/invalid — it reports exactly what is missing.
- **Empty audit**: returns "No audit entries found" when no operations have been logged yet.

## Global Notes

- Prerequisites: install the CLI (`npm install -g @bithumb-official/bithumb-cli`); set `BITHUMB_ACCESS_KEY` / `BITHUMB_SECRET_KEY` for `audit` content and authenticated diagnostics (`diagnose` still runs without them).
- All commands are read-only. `bithumb system diagnose` is the shared connection/auth check referenced by other authenticated skills.
- Append `--json` to any command for the raw Bithumb API / tool response.
