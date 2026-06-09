---
name: bithumb-deposit
description: "Manage Bithumb deposits: deposit address generation/lookup, deposit history (crypto and KRW), and KRW deposit requests. Requires API credentials. Do NOT use for deposit availability status (whether deposits are currently enabled) — use bithumb-account. 빗썸 입금 주소 조회·생성, 코인·원화 입금 내역 확인, 원화 입금 요청을 처리합니다. 입금 주소·입금 내역·원화 입금 관련 요청에 사용하세요. 입금 가능 여부 상태는 bithumb-account를 사용하세요."
---

# Bithumb Deposit CLI

Deposit address management, deposit history, and KRW (원화) deposit requests on Bithumb exchange. **Requires API credentials.**

## Tool Routing

Writes are **CLI-first**: when both the Bithumb CLI and MCP tools exist, use the `bithumb ...` CLI for writes (it carries per-profile read-only enforcement, account isolation, and reproducibility that MCP lacks); fall back to MCP only if the CLI is unavailable/fails or the user asks. Reads: either path is fine.

## Prerequisites

Install the CLI (`npm install -g @bithumb-official/bithumb-cli`) and set credentials (`BITHUMB_ACCESS_KEY`, `BITHUMB_SECRET_KEY`).

## Skill Routing

Market data → `bithumb-market`; account assets / wallet status / deposit availability → `bithumb-account`; orders → `bithumb-trade`; withdrawals → `bithumb-withdraw`; audit logs / diagnostics → `bithumb-system`. Deposits (this skill).

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

See [deposit-commands.md](references/deposit-commands.md) for per-command parameters, examples, and input-format details.

---

## Operation Flow

> ### 🛑 Mandatory Pre-flight Checklist — `deposit krw` / `deposit generate-address`
>
> **Before** any write operation in this skill, run **all** of the following in order. Do **not** skip steps even if the user says "just do it". Stop and ask for explicit user confirmation between step 2 and step 3.
>
> | # | Command | Purpose |
> |---|---|---|
> | 0 | `bithumb config show --json` | **Read-only gate (run first, both writes).** Read the effective profile's `read_only`. If `true`, **stop here** — do not run steps 1–3. List write-capable profiles and ask the user to re-run with `--profile <name>` (**never bypass read-only** — do not auto-select a profile, switch profiles, or disable `read_only` yourself; see gate). Once-per-session: skip if already checked. See [read-only-gate.md](references/read-only-gate.md). |
> | 1 | For `generate-address`: discover the valid `net_type` first (`bithumb deposit addresses` or `bithumb withdraw chance --currency <c>`). For `krw`: confirm the exact KRW amount | Avoid wrong-network address generation / wrong amount |
> | 2 | **🤚 Stop. Ask user to confirm**: for `generate-address` — currency + net_type; for `krw` — amount, and that Kakao 2FA will be triggered | Explicit approval |
> | 3 | `generate-address`: `bithumb deposit generate-address --currency <c> --net-type <n>` / `krw`: `bithumb deposit krw --amount <a> --two-factor-type kakao` | Execute |
> | 4 | `generate-address`: `bithumb deposit address --currency <c> --net-type <n>` (confirm new address) / `krw`: `bithumb deposit list-krw` (check status) | Verify after write |

### Step 1 — Identify deposit action (route to a command)

| User intent | Command |
|---|---|
| **First time / multi-network coin (USDT, USDC, XRP, etc.)** — discover available net_types | `bithumb deposit addresses` *or* `bithumb withdraw chance --currency <c>` to list valid `net_type` values **before** any address call |
| View all deposit addresses | `bithumb deposit addresses` |
| Get address for specific coin + network | `bithumb deposit address --currency BTC --net-type BTC` |
| Generate new deposit address (WRITE) | run the Pre-flight Checklist above |
| Look up one deposit by txid / deposit ID (single identifier) | `bithumb deposit get --currency BTC --txid "<txid>"` |
| Look up / search deposits by multiple or no identifiers | `bithumb deposit list` (coin) / `bithumb deposit list-krw` (KRW) |
| Request KRW deposit (WRITE) | run the Pre-flight Checklist above |

**Identifier rule (txid first)**: deposits use **txid as the primary identifier**. A bare numeric ID is ambiguous (KRW txid and deposit_id are both numeric), so query `--txids` first → if empty, retry the same value with `--deposit-ids` (or ask) before concluding "not found". Special-char/space/comma txids must use single `get`, not `list`. Details in [deposit-commands.md](references/deposit-commands.md).

**Read commands** (`addresses`, `address`, `get`, `list`, `list-krw`) run immediately — no gate.

---

## Edge Cases (safety-critical)

> **🌐 Multi-network coins (USDT, USDC, XRP, etc.)**: The same `currency` may have multiple `net_type` values. **Before any deposit address call**, discover the real values with `bithumb deposit addresses` or `bithumb withdraw chance --currency <c>`. Using a wrong `net_type` results in **permanent loss of funds**. Never guess — always discover first.

- **Secondary address (memo/tag)**: XRP, EOS, ATOM, and similar coins require a `secondary_address`. Always display it when present. Step 4 verify: confirm `secondary_address` is shown for these currencies — if missing, do NOT proceed with the deposit.
- **XRP net_type**: `--net-type XRP` may not be valid in all environments. Run `bithumb deposit addresses` first and use the actual registered `net_type` — do not assume `XRP`.
- **KRW deposit 2FA**: `deposit krw` requires Kakao authentication; the user must complete 2FA on their phone.
- **Block confirmations**: Crypto deposits require blockchain confirmations. Check `bithumb account wallet-status` (in `bithumb-account`) for block sync status before deposits.
- **Deposit states**: KRW — `PROCESSING` / `ACCEPTED` / `CANCELLED`. Coin states differ (UPPERCASE, e.g. `DEPOSIT_PROCESSING`); see [deposit-commands.md](references/deposit-commands.md).

## Global Notes

- When you query deposits (`deposit list` / `list-krw` / `get`) and present the result to the user, always include each entry's `deposit_id` — they need it to look up the deposit later.
- All commands require valid API credentials.
- Bithumb has no demo mode — all deposit operations are **real** (real funds).
- `--json` returns raw Bithumb API response (any command).
