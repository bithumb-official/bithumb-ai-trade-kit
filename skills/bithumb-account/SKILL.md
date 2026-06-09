---
name: bithumb-account
description: "Read account state on Bithumb: balances (KRW/coin), order chance (available balance for trading), withdrawal chance (available withdrawal amount), wallet status, blockchain sync, deposit/withdrawal availability, block height/updated time, and API key info. Requires API credentials. 빗썸 계좌 잔고, 주문 가능 금액, 출금 가능 금액, 지갑 상태, 입출금 현황(가능 여부·블록 갱신시각·블록 동기화), API 키 정보를 조회합니다. 잔고·주문 가능 금액·출금 가능 금액·지갑 상태·입출금 가능 여부·블록 갱신시각·API 키 조회 관련 요청에 사용하세요."
---

# Bithumb Account CLI

Read-only account state: balances, order chance, wallet status, API keys. **Requires API credentials.**

## Prerequisites

Install `@bithumb-official/bithumb-cli` and set `BITHUMB_ACCESS_KEY` / `BITHUMB_SECRET_KEY` env vars.

## Credential Check

Run `bithumb system diagnose` before any authenticated command. If it fails (connection or auth), **STOP — do not retry**. Inform the user (in Korean): "인증 실패. API 키가 유효하지 않거나 만료되었을 수 있습니다." and guide them to set/refresh `BITHUMB_ACCESS_KEY` and `BITHUMB_SECRET_KEY`; re-run `diagnose` before retrying.

## Skill Routing

Market data → `bithumb-market`; account/wallet/API keys → this skill; orders → `bithumb-trade`; deposits → `bithumb-deposit`; withdrawals → `bithumb-withdraw`; audit logs & diagnostics → `bithumb-system`.

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb account assets` | READ | All account assets (KRW + coins) |
| 2 | `bithumb account order-chance --market <m>` | READ | Order availability: balance, fees, limits for a market |
| 3 | `bithumb account wallet-status` | READ | Wallet status: block sync, deposit/withdrawal availability |
| 4 | `bithumb account api-keys` | READ | API key list and expiration dates |

> Audit logs and diagnostics (`bithumb system audit / diagnose`) are in [`bithumb-system`](../bithumb-system/SKILL.md).

---

## Routing — intent → command

| User intent | Command |
|---|---|
| Check assets (all currencies) | `bithumb account assets` |
| Check trading availability for a market | `bithumb account order-chance --market KRW-BTC` |
| Check wallet/blockchain status | `bithumb account wallet-status` |
| Check API key info/expiration | `bithumb account api-keys` |

All commands are **read-only** — run immediately, no confirmation or verification needed.

---

## CLI Reference

### account assets — All Assets

```bash
bithumb account assets
# → KRW: 1,500,000 | BTC: 0.05 (avg: 135,000,000) | ETH: 1.2 (avg: 4,800,000)
```

No parameters. Returns every currency held: `currency`, `balance`, `locked` (reserved for open orders), `avg_buy_price` (volume-weighted), `unit_currency`.

### account order-chance — Order Availability

```bash
bithumb account order-chance --market KRW-BTC
# → bid_fee: 0.0025 | ask_fee: 0.0025 | available KRW: 1,500,000 | available BTC: 0.05
```

`--market` (required): market code. Returns `bid_fee`/`ask_fee`, available bid/ask balances, order limits (min/max), market info. **Pre-trade essential** — call before placing an order to verify balance, fees, and constraints.

### account wallet-status — Wallet Status

```bash
bithumb account wallet-status
# → BTC: block_state=normal, deposit=true, withdrawal=true
```

No parameters. Per-currency block sync state and deposit/withdrawal availability. **Check before deposit/withdrawal** — if `block_state` is not normal or the operation is disabled (e.g. blockchain maintenance), it will fail.

### account api-keys — API Key Info

```bash
bithumb account api-keys
```

No parameters. Returns API keys with expiration dates. Check periodically (or when auth suddenly fails) to avoid unexpected authentication failures.

## Cross-Skill Workflows

### Pre-trade balance check
> User: "BTC 0.01개 살 수 있어?"

```
1. bithumb-account  bithumb account order-chance --market KRW-BTC  → check available KRW & fees
2. bithumb-market   bithumb market ticker KRW-BTC                  → current price
        ↓ user approves
3. bithumb-trade    bithumb trade place --market KRW-BTC --side bid ...
```

### Pre-withdrawal check (full Mandatory Pre-flight)
> User: "BTC 출금하려고 하는데 가능해?" / "BTC 출금하기 전에 다 체크해줘"

```
1. bithumb-account    bithumb account assets                           → check BTC balance
2. bithumb-account    bithumb account wallet-status                    → check BTC withdrawal enabled
3. bithumb-withdraw   bithumb withdraw chance --currency BTC --net-type BTC
                                                                              → fee, min, daily limit, net_type
4. bithumb-withdraw   bithumb withdraw addresses                       → confirm destination is allowed-list
5. bithumb-market     bithumb market fee-inout BTC                     → cross-check fee from market side
        ↓ user explicitly confirms (currency, net_type, full address, amount, memo if applicable)
6. bithumb-withdraw   bithumb withdraw coin --currency BTC --net-type BTC --amount ...
7. bithumb-withdraw   bithumb withdraw list --currency BTC --limit 1   → verify status
```

> This pre-flight is mirrored in [bithumb-withdraw/SKILL.md](../bithumb-withdraw/SKILL.md) as its "🛑 Mandatory Pre-flight Checklist" so it stays visible when only the withdraw skill is active.

---

## Global Notes

- All commands require valid API credentials and are read-only.
- Bithumb has no demo/simulated mode — all data reflects real account state.
- `--json` returns the raw Bithumb API response.
- `account assets` may include zero-balance currencies; `locked` funds are reserved for open orders and unavailable for new ones.
