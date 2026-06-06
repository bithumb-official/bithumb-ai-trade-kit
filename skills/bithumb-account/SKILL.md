---
name: bithumb-account
description: "Read account state on Bithumb: balances (KRW/coin), order chance (available balance for trading), withdrawal chance (available withdrawal amount), wallet status, blockchain sync, deposit/withdrawal availability, block height/updated time, and API key info. Requires API credentials. 빗썸 계좌 잔고, 주문 가능 금액, 출금 가능 금액, 지갑 상태, 입출금 현황(가능 여부·블록 갱신시각·블록 동기화), API 키 정보를 조회합니다. 잔고·주문 가능 금액·출금 가능 금액·지갑 상태·입출금 가능 여부·블록 갱신시각·API 키 조회 관련 요청에 사용하세요."
---

# Bithumb Account CLI

Account balance, order availability, wallet status, and API key management on Bithumb exchange. **Requires API credentials.**

> Audit logs and system diagnostics live in the [`bithumb-system`](../bithumb-system/SKILL.md) skill. `bithumb system diagnose` is referenced below only as the shared credential/connection check.

## Prerequisites

1. **CLI**: `npm install -g @bithumb-official/bithumb-cli`
2. Configure credentials:
   ```bash
   export BITHUMB_ACCESS_KEY=your_key
   export BITHUMB_SECRET_KEY=your_secret
   ```
3. Verify connection:
   ```bash
   bithumb system diagnose
   ```

## Credential Check

**Run this check before any authenticated command.**

```bash
bithumb system diagnose
```

- If connection or auth fails: **stop all operations**, guide the user to set `BITHUMB_ACCESS_KEY` and `BITHUMB_SECRET_KEY`.
- If OK: proceed.

### Handling Authentication Errors

If any command returns an authentication error:
1. **Stop immediately** — do not retry
2. Inform the user: "인증 실패. API 키가 유효하지 않거나 만료되었을 수 있습니다."
3. Guide the user to check credentials
4. After update, run `bithumb system diagnose` to verify
5. Only then retry

## Skill Routing

- For market data (prices, charts, depth) → use `bithumb-market`
- For account assets, wallet status, API keys → use `bithumb-account` (this skill)
- For placing/cancelling orders → use `bithumb-trade`
- For deposits → use `bithumb-deposit`
- For withdrawals → use `bithumb-withdraw`
- For audit logs, diagnostics → use `bithumb-system`

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb account assets` | READ | All account assets (KRW + coins) |
| 2 | `bithumb account order-chance --market <m>` | READ | Order availability: balance, fees, limits for a market |
| 3 | `bithumb account wallet-status` | READ | Wallet status: block sync, deposit/withdrawal availability |
| 4 | `bithumb account api-keys` | READ | API key list and expiration dates |

> Audit logs and diagnostics (`bithumb system audit / diagnose`) are in [`bithumb-system`](../bithumb-system/SKILL.md).

---

## Operation Flow

### Step 1 — Identify account action

| User intent | Command |
|---|---|
| Check assets (all currencies) | `bithumb account assets` |
| Check trading availability for a market | `bithumb account order-chance --market KRW-BTC` |
| Check wallet/blockchain status | `bithumb account wallet-status` |
| Check API key info/expiration | `bithumb account api-keys` |

### Step 2 — Run immediately

All commands in this skill are **read-only** — no confirmation needed.

### Step 3 — No writes, no verification needed

All commands are read-only.

---

## CLI Reference

### account assets — All Assets

```bash
bithumb account assets [--json]
```

No parameters required. Returns all currencies with balance including: `currency`, `balance`, `locked`, `avg_buy_price`, `unit_currency`.

```bash
bithumb account assets
# → KRW: 1,500,000 | BTC: 0.05 (avg: 135,000,000) | ETH: 1.2 (avg: 4,800,000)
```

---

### account order-chance — Order Availability

```bash
bithumb account order-chance --market <market> [--json]
```

| Param | Required | Description |
|---|---|---|
| `--market` | Yes | Market code (e.g., `KRW-BTC`) |

Returns: `bid_fee` (buy fee rate), `ask_fee` (sell fee rate), available balances for bid/ask, order limits (min/max), and market info.

```bash
bithumb account order-chance --market KRW-BTC
# → bid_fee: 0.0025 | ask_fee: 0.0025 | available KRW: 1,500,000 | available BTC: 0.05
```

> **Pre-trade essential**: Always call this before placing an order to verify sufficient balance, fee rates, and market constraints.

---

### account wallet-status — Wallet Status

```bash
bithumb account wallet-status [--json]
```

No parameters required. Returns per currency: block sync status, deposit/withdrawal availability.

```bash
bithumb account wallet-status
# → BTC: block_state=normal, deposit=true, withdrawal=true
```

> **Check before deposit/withdrawal**: If `block_state` is not normal or deposit/withdrawal is disabled, the operation will fail.

---

### account api-keys — API Key Info

```bash
bithumb account api-keys [--json]
```

No parameters required. Returns: list of API keys with expiration dates.

> **Expiration monitoring**: Check periodically to avoid unexpected authentication failures.

---

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

> This checklist is also embedded directly in [bithumb-withdraw/SKILL.md](../bithumb-withdraw/SKILL.md) as the "🛑 Mandatory Pre-flight Checklist" section so it remains visible when the withdraw skill is the only one active.

> System health check (`bithumb system diagnose` + `wallet-status`) is documented in [`bithumb-system`](../bithumb-system/SKILL.md).

---

## Edge Cases

- **Empty balance**: `account assets` may return currencies with zero balance — filter as needed
- **Locked balance**: `locked` field shows funds reserved for open orders; not available for new orders
- **Wallet maintenance**: `account wallet-status` may show deposit/withdrawal disabled during blockchain maintenance
- **API key expiration**: Keys expire; check `account api-keys` if authentication suddenly fails

## Global Notes

- All commands require valid API credentials
- Bithumb has no demo/simulated trading mode — all data reflects real account state
- `avg_buy_price` in balance is the volume-weighted average purchase price
- `--json` returns raw Bithumb API response
