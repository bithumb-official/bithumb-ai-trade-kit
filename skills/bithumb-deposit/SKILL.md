---
name: bithumb-deposit
description: "Manage Bithumb deposits: deposit address generation/lookup, deposit history (crypto and KRW), and KRW deposit requests. Requires API credentials. Do NOT use for deposit availability status (whether deposits are currently enabled) — use bithumb-account. 빗썸 입금 주소 조회·생성, 코인·원화 입금 내역 확인, 원화 입금 요청을 처리합니다. 입금 주소·입금 내역·원화 입금 관련 요청에 사용하세요. 입금 가능 여부 상태는 bithumb-account를 사용하세요."
---

# Bithumb Deposit CLI

## Tool Routing

This skill's write operations are **CLI-first**: when both the Bithumb CLI and the MCP tools are available, use the documented `bithumb ...` CLI command as the default path. Fall back to the MCP tool only when the CLI is unavailable or fails, or when the user explicitly asks for MCP. Read-only queries have no such preference — either path is fine.

Why CLI-first for writes: the CLI carries safety mechanisms the MCP path lacks — per-profile read-only enforcement, account isolation, and terminal reproducibility. These matter only for writes.

Deposit address management, deposit history, and KRW (원화) deposit requests on Bithumb exchange. **Requires API credentials.**

## Prerequisites

1. **CLI**: `npm install -g @bithumb-official/bithumb-cli`
2. Configure credentials:
   ```bash
   export BITHUMB_ACCESS_KEY=your_key
   export BITHUMB_SECRET_KEY=your_secret
   ```

## Skill Routing

- For market data (prices, charts, depth) → use `bithumb-market`
- For account assets, wallet status → use `bithumb-account`
- For placing/cancelling orders → use `bithumb-trade`
- For deposits → use `bithumb-deposit` (this skill)
- For withdrawals → use `bithumb-withdraw`
- For audit logs, diagnostics → use `bithumb-system`

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb deposit addresses` | READ | Get all deposit addresses |
| 2 | `bithumb deposit address` | READ | Get deposit address for specific currency + network |
| 3 | `bithumb deposit generate-address` | WRITE | Generate new deposit address |
| 4 | `bithumb deposit get` | READ | Get a single deposit by deposit ID or txid |
| 5 | `bithumb deposit list` | READ | List coin deposit history |
| 6 | `bithumb deposit list-krw` | READ | List KRW deposit history |
| 7 | `bithumb deposit krw` | WRITE | Request KRW deposit (**2FA required**) |

---

## Operation Flow

### Step 1 — Identify deposit action

| User intent | Command |
|---|---|
| **First time / multi-network coin (USDT, USDC, XRP, etc.)** — discover available net_types | 1) `bithumb deposit addresses` *or* `bithumb withdraw chance --currency <c>` to list valid `net_type` values **before** any address call |
| View all deposit addresses | `bithumb deposit addresses` |
| Get address for specific coin + network | `bithumb deposit address --currency BTC --net-type BTC` |
| Generate new deposit address (then verify) | `bithumb deposit generate-address --currency BTC --net-type BTC` → verify with `bithumb deposit address --currency BTC --net-type BTC` |
| Look up one deposit by deposit ID — **식별자 1개면 단건** | `bithumb deposit get --currency BTC --deposit-id <id>` |
| Look up one deposit by txid — **식별자 1개면 단건**       | `bithumb deposit get --currency BTC --txid <txid>` |
| Look up deposit(s) by multiple txids                     | `bithumb deposit list --txids <txid1,txid2>` |
| Search deposit history / filter by state (식별자 없거나 여러 개) | `bithumb deposit list` |
| View KRW deposit history | `bithumb deposit list-krw` |
| Request KRW deposit | `bithumb deposit krw --amount 100000 --two-factor-type kakao` |

### Step 2 — Read commands run immediately; write commands need confirmation

**Read commands** (`deposit addresses`, `deposit address`, `deposit get`, `deposit list`, `deposit list-krw`): run immediately.

**Write commands**:
- `deposit generate-address`: confirm currency and network before generating
- `deposit krw`: **high risk** — confirm amount and 2FA method; requires Kakao 2FA

> **Read-only: never bypass.** deposit has no read-only pre-flight gate — the CLI
> blocks the write locally and throws before any network call (ADR 0010). If a
> write is blocked because the effective profile is read-only, relay that message
> to the user and stop. Do **not** work around it: never attach `--profile <other>`
> yourself, change `default_profile`, run `config set` to flip `read_only` to
> `false`, or retry the same write under a different profile. The user enabled
> read-only on purpose. You may **list** write-capable profiles, but selecting and
> running one is the user's call (see CONTEXT.md *Read-only bypass*).

### Step 3 — Verify after writes

- After `generate-address`: run `bithumb deposit address --currency <c> --net-type <n>` to confirm the new address
- After `krw`: run `bithumb deposit list-krw` to check deposit status

---

## CLI Reference

### deposit addresses — All Deposit Addresses

```bash
bithumb deposit addresses [--json]
```

No parameters required. Returns all registered deposit addresses across currencies.

---

### deposit address — Specific Deposit Address

```bash
bithumb deposit address --currency <symbol> --net-type <network> [--json]
```

| Param | Required | Description |
|---|---|---|
| `--currency` | Yes | Currency symbol (e.g., `BTC`, `ETH`, `USDT`) |
| `--net-type` | Yes | Network type (e.g., `BTC`, `ETH`, `TRX`) |

Returns: deposit address and secondary address (memo/tag) if applicable.

> **Network matters**: USDT exists on multiple networks (ETH, TRX, etc.). Always specify the correct `--net-type`.

```bash
bithumb deposit address --currency BTC --net-type BTC
bithumb deposit address --currency USDT --net-type ETH
```

---

### deposit generate-address — Generate New

```bash
bithumb deposit generate-address --currency <symbol> --net-type <network> [--json]
```

> **Confirm before generating**: Ask the user to confirm currency and network. Sending to the wrong network results in permanent loss.

---

### deposit get — Get Single Deposit by Deposit ID or Txid

```bash
bithumb deposit get --currency <symbol> [--deposit-id <id>] [--txid <txid>] [--json]
```

| Param | Required | Description |
|---|---|---|
| `--currency` | Yes | Currency symbol |
| `--deposit-id` | No | Deposit ID |
| `--txid` | No | Transaction ID |

> Only `--currency` is required. Optionally provide `--deposit-id` and/or `--txid` to narrow the lookup (the API combines them as an AND filter). With neither, the API returns the most recent deposit.

Examples:
```bash
bithumb deposit get --currency BTC --deposit-id abc-123
bithumb deposit get --currency BTC --txid 0xabcdef
```

---

### deposit list — Coin Deposit History

```bash
bithumb deposit list [--currency <symbol>] [--state <state>] \
  [--deposit-ids <id1,id2>] [--txids <txid1,txid2>] \
  [--limit <n>] [--page <n>] [--order-by <asc|desc>] [--json]
```

> **🔁 식별자 폴백**: ID가 uuid(`--deposit-ids`)인지 txid인지 불명확하면 `--deposit-ids`로 먼저 조회 → **빈 배열이면 같은 값을 `--txids`로 재시도**(또는 사용자에게 "txids로 조회할까요?" 질문)한 뒤에 "없음"으로 단정한다. 두 필터 모두 정상 동작한다.

| Param | Required | Default | Description |
|---|---|---|---|
| `--currency` | No | - | Filter by currency |
| `--state` | No | - | Coin deposit state (UPPERCASE, case-sensitive). Common: `DEPOSIT_PROCESSING`, `DEPOSIT_ACCEPTED`, `DEPOSIT_CANCELLED`. 15 states total (REQUESTED_* / DEPOSIT_* / REFUNDING_* / REFUNDED_*) — see Bithumb API docs for the full list. Note: these differ from KRW deposit states. |
| `--deposit-ids` | No | - | Filter by deposit IDs (comma-separated) |
| `--txids` | No | - | Filter by deposit TXIDs (comma-separated) |
| `--limit` | No | `100` | Max results (1-100) |
| `--page` | No | 1 | Page number (>=1) |
| `--order-by` | No | `desc` | Sort order |

---

### deposit list-krw — KRW Deposit History

```bash
bithumb deposit list-krw [--state <PROCESSING|ACCEPTED|CANCELLED>] \
  [--deposit-ids <id1,id2>] [--txids <txid1,txid2>] \
  [--limit <n>] [--page <n>] [--order-by <asc|desc>] [--json]
```

> **🔁 식별자 폴백 (KRW는 uuid·txid 모두 숫자)**: 원화 입금은 `--deposit-ids`(uuid)와 `--txids`가 **둘 다 숫자**라 값만으로 구분이 안 된다. 사용자가 맨 숫자 ID를 주면 `--deposit-ids`로 먼저 조회 → **빈 배열이면 같은 값을 `--txids`로 재시도**(또는 "txids로 조회할까요?" 질문)한 뒤에 "없음"으로 단정한다. 예: `--deposit-ids 405896908` → `[]` → `--txids 1657265` → 레코드.

---

### deposit krw — KRW Deposit Request

> **WARNING**: This operation requests a real KRW deposit. Requires Kakao 2FA verification.

```bash
bithumb deposit krw --amount <krw_amount> --two-factor-type kakao [--json]
```

| Param | Required | Description |
|---|---|---|
| `--amount` | Yes | Deposit amount in KRW |
| `--two-factor-type` | Yes | 2FA method (e.g., `kakao`) |

**Before calling**:
1. Confirm the exact amount with the user
2. Inform the user that Kakao 2FA will be triggered
3. Wait for explicit user approval

---

## Quickstart

```bash
# View all deposit addresses
bithumb deposit addresses

# Get BTC deposit address
bithumb deposit address --currency BTC --net-type BTC

# Get USDT (ERC-20) deposit address
bithumb deposit address --currency USDT --net-type ETH

# Recent coin deposits
bithumb deposit list --limit 10

# Recent KRW deposits
bithumb deposit list-krw --limit 10

# Look up a single BTC deposit by txid
bithumb deposit get --currency BTC --txid 0xabc123...

# Look up BTC deposit(s) by txid (multiple)
bithumb deposit list --txids 0xabc123...

# Look up a single BTC deposit by deposit ID
bithumb deposit get --currency BTC --deposit-id <deposit-id>
```

---

## Edge Cases

- **Network selection**: Always confirm `--net-type` with the user. Sending coins to the wrong network causes permanent loss.

> **🌐 Multi-network coins (USDT, USDC, XRP, etc.)**: The same `currency` may have multiple `net_type` values. **Before any deposit address call**, run one of:
> - `bithumb deposit addresses` — list ALL registered deposit addresses with their `currency` + `net_type`
> - `bithumb withdraw chance --currency <c>` — list networks supported for that currency
>
> Using a wrong `net_type` results in **permanent loss of funds**. Never guess — always discover first.

- **Secondary address (memo/tag)**: XRP, EOS, ATOM, and similar coins require a `secondary_address`. Always display it when present in the response. Step 3 verification: confirm `secondary_address` is shown for these currencies — if missing, do NOT proceed with deposit.
- **XRP net_type**: `--net-type XRP` may not be a valid value in all environments. Before querying an XRP deposit address, run `bithumb deposit addresses` first to confirm the actual `net_type` value registered for XRP. Do not assume `XRP` — use the value returned by the API.
- **KRW deposit 2FA**: `deposit krw` requires Kakao authentication. The user must complete 2FA on their phone.
- **Deposit states**: `PROCESSING` (pending confirmations), `ACCEPTED` (credited), `CANCELLED` (failed/reversed)
- **식별자 모호성 (uuid vs txid)**: 원화 입·출금은 `deposit_id`(uuid)와 `txid`가 모두 숫자라 값만으로 구분되지 않는다. `list-krw`에서 한쪽 필터(`--deposit-ids`)가 빈 배열이면 같은 값을 반대쪽(`--txids`)으로 재조회하거나 사용자에게 확인한 뒤 "없음"으로 단정할 것. (코인 `list`도 uuid/txid 입력이 불명확하면 동일하게 폴백.)
- **Block confirmations**: Crypto deposits require blockchain confirmations. Check `bithumb account wallet-status` for block sync status.
- **`--json`**: add to any command for raw JSON output

## Global Notes

- All commands require valid API credentials
- Bithumb has no demo mode — all deposit operations are real
- Always check `bithumb account wallet-status` (in `bithumb-account`) before deposits to verify the blockchain is synced
- `--json` returns raw Bithumb API response
