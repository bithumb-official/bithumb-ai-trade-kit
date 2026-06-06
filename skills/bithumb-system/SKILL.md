---
name: bithumb-system
description: "Inspect Bithumb local audit logs (operation/trade history) and run connection and credential diagnostics. The diagnose command needs no auth to test credentials; audit reads local logs only. 빗썸 로컬 감사 로그 조회, 연결·인증 진단, 활성 모듈 상태 확인을 처리합니다. 감사 로그·거래 로그·시스템 진단·연결 상태 관련 요청에 사용하세요."
---

# Bithumb System CLI

Local audit logs and connection/auth diagnostics for the Bithumb CLI. All commands are **read-only**. `system diagnose` is useful even before credentials are valid (it helps diagnose auth problems).

## Prerequisites

1. **CLI**: `npm install -g @bithumb-official/bithumb-cli`
2. Credentials (for `audit` content and authenticated diagnostics):
   ```bash
   export BITHUMB_ACCESS_KEY=your_key
   export BITHUMB_SECRET_KEY=your_secret
   ```
3. `bithumb system diagnose` itself works without valid credentials — it reports what is missing.

## Skill Routing

- For market data (prices, charts, depth) → use `bithumb-market`
- For account assets, wallet status, API keys → use `bithumb-account`
- For placing/cancelling orders → use `bithumb-trade`
- For deposits → use `bithumb-deposit`
- For withdrawals → use `bithumb-withdraw`
- For audit logs and diagnostics → use `bithumb-system` (this skill)

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb system diagnose` | READ | Connection, auth, config, and module diagnostics |
| 2 | `bithumb system audit` | READ | Local audit logs (operation history) |

---

## Operation Flow

### Step 1 — Identify system action

| User intent | Command |
|---|---|
| Check connection / auth / config health | `bithumb system diagnose` |
| Review past operations (local log) | `bithumb system audit` |

> "사용 가능한 기능 알려줘" / "what can you do?" 같은 요청은 도구 호출이 아니라 스킬 라우팅 목록(market / account / trade / deposit / withdraw / system)으로 답한다. 특정 환경에서 어떤 모듈이 켜져 있는지 확인이 필요하면 `bithumb system diagnose`의 module status를 본다.

### Step 2 — Run immediately

All commands in this skill are **read-only** — no confirmation needed.

### Step 3 — No writes, no verification needed

All commands are read-only.

---

## CLI Reference

### system diagnose — System Diagnostics

```bash
bithumb system diagnose [--json]
```

Checks: API reachability, authentication status, TOML config validity, module status.

```bash
bithumb system diagnose
# → API: reachable | Auth: valid | Config: OK | Modules: all active
```

> **Shared credential check**: other authenticated skills (account, trade, deposit, withdraw) reference `bithumb system diagnose` as their pre-flight connection/auth check. If it fails, stop and guide the user to set `BITHUMB_ACCESS_KEY` / `BITHUMB_SECRET_KEY`.

---

### system audit — Audit Logs

```bash
bithumb system audit [--tool <name>] [--level <INFO|WARN|ERROR|DEBUG>] \
  [--limit <n>] [--since <timestamp>] [--json]
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--tool` | No | - | Filter by tool/command name |
| `--level` | No | - | Filter: `INFO`, `WARN`, `ERROR`, `DEBUG` |
| `--limit` | No | 20 | Max entries to return (>=1) |
| `--since` | No | - | ISO 8601 timestamp; entries at or after this time |

```bash
# Recent audit logs
bithumb system audit --limit 10

# Errors only
bithumb system audit --level ERROR

# Specific command logs
bithumb system audit --tool trade_place_order --limit 20
```

---

## Cross-Skill Workflows

### System health check
> User: "빗썸 연결 상태 확인해줘"

```
1. bithumb-system   bithumb system diagnose        → full diagnostics
2. bithumb-account  bithumb account wallet-status   → blockchain status
```

---

## Edge Cases

- **Audit logs are local**: `system audit` reads local log files, not Bithumb server-side history
- **Diagnose without auth**: `system diagnose` runs even when credentials are missing/invalid — it reports exactly what is missing
- **Empty audit**: returns "No audit entries found" when no operations have been logged yet

## Global Notes

- All commands are read-only
- `bithumb system diagnose` is the shared connection/auth check referenced by other authenticated skills
- `--json` returns raw Bithumb API / tool response
