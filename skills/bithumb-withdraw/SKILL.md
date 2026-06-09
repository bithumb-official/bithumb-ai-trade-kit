---
name: bithumb-withdraw
description: "Manage Bithumb withdrawals: crypto withdrawal, KRW withdrawal, withdrawal cancellation, withdrawal history, withdrawal fee lookup, and allowed-address management. Requires API credentials. Do NOT use for withdrawal chance / available withdrawal amount — use bithumb-account. 빗썸 코인·원화 출금 실행·취소, 출금 내역 조회, 출금 수수료 확인, 허용 주소 관리를 처리합니다. 출금·코인 전송·원화 출금·출금 취소·출금 내역·허용 주소 관련 요청에 사용하세요. 출금 가능 금액은 bithumb-account를 사용하세요."
---

# Bithumb Withdrawal CLI

Crypto and KRW (원화) withdrawals on Bithumb (빗썸): execution, cancellation, history, fee/availability checks, and allowed-address management.

> **CRITICAL**: Crypto withdrawals (`withdraw coin`) are **IRREVERSIBLE**. A wrong address or network = permanent loss of funds. Always double-check address, network, and amount before executing.

## Tool Routing

Writes are **CLI-first**: prefer the documented `bithumb ...` CLI over MCP tools (the CLI carries the per-profile read-only gate, account isolation, and terminal reproducibility that writes need). Fall back to MCP only if the CLI is unavailable/fails or the user asks for it; reads may use either path.

## Prerequisites

1. Install: `npm install -g @bithumb-official/bithumb-cli`
2. Credentials: `export BITHUMB_ACCESS_KEY=...` and `export BITHUMB_SECRET_KEY=...`. Withdrawal addresses must be pre-registered on Bithumb (for `withdraw coin`).

## Skill Routing

Market data → `bithumb-market`; assets / wallet status / available withdrawal amount → `bithumb-account`; orders → `bithumb-trade`; deposits → `bithumb-deposit`; audit/diagnostics → `bithumb-system`; withdrawals → this skill.

> A user *question* about how much can be withdrawn ("출금 가능 금액") routes to `bithumb-account`. The `bithumb withdraw chance` *command* lives here and is run as a withdrawal pre-flight step (fees, limits, supported `net_type`) — see the checklist below. The two are not in conflict: account answers the question, this skill runs the pre-flight command.

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb withdraw chance` | READ | Withdrawal availability: balance, fees, limits |
| 2 | `bithumb withdraw addresses` | READ | List allowed withdrawal addresses |
| 3 | `bithumb withdraw get` | READ | Get specific withdrawal details |
| 4 | `bithumb withdraw list` | READ | Coin withdrawal history |
| 5 | `bithumb withdraw list-krw` | READ | KRW withdrawal history |
| 6 | `bithumb withdraw coin` | WRITE | **Withdraw crypto (IRREVERSIBLE)** |
| 7 | `bithumb withdraw krw` | WRITE | **Withdraw KRW (2FA required)** |
| 8 | `bithumb withdraw cancel` | WRITE | Cancel pending crypto withdrawal |

Full parameter tables, input formats, and the workflow example are in [references/withdraw-commands.md](references/withdraw-commands.md).

## Routing rules

| User intent | Command |
|---|---|
| Check if withdrawal is possible & fees / discover `net_type` | `bithumb withdraw chance --currency <c> [--net-type <n>]` |
| View allowed withdrawal addresses | `bithumb withdraw addresses` |
| Look up one withdrawal by identifier (id/txid) | `bithumb withdraw get --currency <c>` |
| Search/filter withdrawal history (no identifier or several) | `bithumb withdraw list` |
| View KRW withdrawal history | `bithumb withdraw list-krw` |
| Withdraw crypto | `bithumb withdraw coin ...` (pre-flight required) |
| Withdraw KRW | `bithumb withdraw krw ...` (pre-flight required) |
| Cancel pending withdrawal | `bithumb withdraw cancel --withdrawal-id <id>` (pre-flight required) |

**Identifier decision rule (txid-first)**: deposits/withdrawals use **txid as the primary identifier** — query `--txid`/`--txids` first; if empty, retry the same value as `--withdrawal-id(s)` (or ask the user) before concluding "not found". Details and the `--txids` JSON-array / special-char-txid handling are in [references/withdraw-commands.md](references/withdraw-commands.md).

Read commands (`chance`, `addresses`, `get`, `list`, `list-krw`) run immediately. All write commands require the pre-flight checklist and explicit user confirmation.

## 🛑 Mandatory Pre-flight Checklist — `withdraw coin` / `withdraw krw` / `withdraw cancel`

**Before** any write in this skill, run the steps below **in order**. Do **not** skip steps even if the user says "just send it". Stop and ask for explicit user confirmation between step 5 and step 7.

Step 0 (read-only gate) applies to **every** write here — including `withdraw cancel`. The heavier balance/address lookups (steps 1–5) are only for `withdraw coin` / `withdraw krw`; for `withdraw cancel`, run Step 0, confirm the withdrawal ID, then execute.

| # | Command | Purpose |
|---|---|---|
| 0 | `bithumb config show --json` | **Read-only gate (run first, all writes incl. cancel).** Read the effective profile's `read_only`. If `true`, **stop** — do not run later steps. List write-capable profiles and ask the user to re-run with `--profile <name>`. **Never bypass read-only** — do not auto-select/switch profiles or disable `read_only` yourself. See [references/read-only-gate.md](references/read-only-gate.md). |
| 1 | `bithumb account assets` | Confirm sufficient balance |
| 2 | `bithumb account wallet-status` | Confirm withdrawal is enabled (blockchain not under maintenance) |
| 3 | `bithumb withdraw chance --currency <c> --net-type <n>` | Confirm fee, min amount, daily limit, supported `net_type` |
| 4 | `bithumb withdraw addresses` | Confirm destination is in the allowed list (Bithumb requirement) |
| 5 | `bithumb market fee-inout <currency>` | Cross-check withdrawal fee from market side |
| 5b | **🛂 Classify withdrawal type and collect required params** | Open [references/withdrawal-type-params.md](references/withdrawal-type-params.md). Confirm whether this is Internal, CODE ID Connect, CODE 개인, CODE 법인, or WHITELIST; collect every required param for that type. |
| 6 | **🤚 Stop. Ask user to confirm**: withdrawal type, currency, net_type, amount, address (full string), secondary_address (memo/tag) if applicable, exchange_name if external, **and (CODE 개인/법인) receiver_type + all receiver fields** |
| 7 | `bithumb withdraw coin --currency <c> --net-type <n> --amount <a> --address <addr> [--secondary-address <m>] [--exchange-name <e>] [--receiver-type <t> ...receiver fields]` | Execute |
| 8 | `bithumb withdraw list --currency <c> --limit 1` | Verify status |

Skipping any step (especially 4 / 5 / 6) is the most common cause of failed/lost withdrawals in past QA. The same checklist is mirrored in [bithumb-account Cross-Skill Workflows](../bithumb-account/SKILL.md#cross-skill-workflows).

After a write, verify: `withdraw coin` → `withdraw list --currency <c> --limit 1`; `withdraw krw` → `withdraw list-krw`; `withdraw cancel` → `withdraw get --currency <c> --withdrawal-id <id>`.

## 🛂 Withdrawal Type Params

Before `withdraw coin`, classify the withdrawal type and check
[references/withdrawal-type-params.md](references/withdrawal-type-params.md).

Summary:

- Internal: no `--exchange-name`, no receiver fields.
- External: `--exchange-name` is required.
- CODE 개인/법인: `--receiver-type` and all receiver fields are required.
- `--secondary-address` is required when the asset/network needs a memo/tag.

## Global Notes

- When you query withdrawals (`withdraw list` / `list-krw` / `get`) and present the result to the user, always include each entry's `withdrawal_id` — they need it to look up or cancel the withdrawal later.
- All commands require valid API credentials. Bithumb has **no demo mode** — every withdrawal is real and involves real funds.
- `--json` on any command returns the raw Bithumb API response.
- Always run the 🛑 Mandatory Pre-flight Checklist above for every withdrawal — it is the single safety flow (no shorter sequence).
