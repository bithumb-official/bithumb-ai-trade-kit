---
name: bithumb-trade
description: "Place, cancel, and query Bithumb spot orders: single orders (limit, market buy/sell), batch orders (up to 20), batch cancel (up to 30), TWAP orders, and order history. Requires API credentials. 빗썸 현물 주문 생성·취소·조회, 배치 주문(최대 20건), 배치 취소(최대 30건), TWAP 주문을 처리합니다. 매수·매도·지정가·시장가·주문 취소·주문 조회·다건 주문·TWAP 관련 요청에 사용하세요."
---

# Bithumb CEX Trading CLI

## Tool Routing

This skill's write operations are **CLI-first**: when both the Bithumb CLI and the MCP tools are available, use the documented `bithumb ...` CLI command as the default path. Fall back to the MCP tool only when the CLI is unavailable or fails, or when the user explicitly asks for MCP. Read-only queries have no such preference — either path is fine.

Why CLI-first for writes: the CLI carries safety mechanisms the MCP path lacks — per-profile read-only gating, account isolation, and terminal reproducibility (see the pre-flight read-only gate). These matter only for writes.

Spot order management on Bithumb exchange. Place, cancel, and monitor single or batch orders; execute TWAP (Time-Weighted Average Price) orders for large positions. **Requires API credentials.**

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

### Step A — Verify connection

```bash
bithumb system diagnose
```

- If connection or auth fails: **stop all operations**, guide the user to set `BITHUMB_ACCESS_KEY` and `BITHUMB_SECRET_KEY` environment variables or check `~/.bithumb/config.toml`.
- If OK: proceed to Step B.

### Step B — Verify trading permissions

```bash
bithumb account order-chance --market KRW-BTC
```

- Confirms available balance, fee rates, and order limits for the market
- If authentication fails: guide user to check API key permissions

### Handling Authentication Errors

If any command returns an authentication error:
1. **Stop immediately** — do not retry
2. Inform the user: "인증 실패. API 키가 유효하지 않거나 만료되었을 수 있습니다."
3. Guide the user to update credentials
4. After update, run `bithumb system diagnose` to verify
5. Only then retry the original operation

## Skill Routing

- For market data (prices, charts, depth) → use `bithumb-market`
- For account assets, wallet status, API keys → use `bithumb-account`
- For placing/cancelling orders → use `bithumb-trade` (this skill)
- For deposits → use `bithumb-deposit`
- For withdrawals → use `bithumb-withdraw`
- For audit logs, diagnostics → use `bithumb-system`

## Command Index

### Single Orders (4 commands)

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb trade place` | WRITE | Place spot order (limit / price / market) |
| 2 | `bithumb trade cancel` | WRITE | Cancel order by order-id or client-order-id |
| 3 | `bithumb trade get` | READ | Get a SINGLE order by one order-id / client-order-id |
| 4 | `bithumb trade list` | READ | Query MULTIPLE orders (order-ids/client-order-ids) or list by filters |

> **Querying multiple orders:** When the user asks for two or more order numbers at once, use `trade list --order-ids id1,id2` or `trade list --client-order-ids id1,id2` (MCP `trade_get_orders` with the `order_ids` or `client_order_ids` array) — a single call (up to 100 ids per call; for more than 100, split into batches of 100). Do NOT call `trade get` (MCP `trade_get_order`) repeatedly per id. The server defaults to `state=wait`, so to look up done/cancelled orders by id add `--states wait,done,cancel` (MCP: `states: ["wait","done","cancel"]`). Note: `watch` (auto-orders) cannot be mixed with `wait`/`done`/`cancel` — query it separately with `--state watch`; and `--state`/`--states` cannot be used together.

### Batch Orders (2 commands)

| # | Command | Type | Description |
|---|---|---|---|
| 5 | `bithumb trade batch-place` | WRITE | Place multiple orders at once (max 20) |
| 6 | `bithumb trade batch-cancel` | WRITE | Cancel multiple orders at once (max 30) |

### TWAP Orders (3 commands)

| # | Command | Type | Description |
|---|---|---|---|
| 7 | `bithumb twap place` | WRITE | Place TWAP order (time-weighted execution) |
| 8 | `bithumb twap list` | READ | Get TWAP order history |
| 9 | `bithumb twap cancel` | WRITE | Cancel active TWAP order |

For full parameter details, see the reference files below.

---

## Operation Flow

### Step 0 — Credential Check

Before any authenticated command: see [Credential Check](#credential-check).

> ### 🛑 Mandatory Pre-flight Checklist — `trade place` / `batch-place` / `twap place`
>
> **Before** any write order, run all of the following in order. Do **not** skip even if the user says "just buy it". Stop and ask for explicit user confirmation between step 3 and step 4.
>
> | # | Command | Purpose |
> |---|---|---|
> | 0 | `bithumb config show --json` | **Read-only gate (run first).** Read the effective profile's `read_only`. If `true`, **stop here** — do not run steps 1–2. List write-capable profiles and ask the user to re-run with `--profile <name>` (**never bypass read-only** — do not auto-select a profile, switch profiles, or disable `read_only` yourself; see gate). Once-per-session: skip if already checked. See [read-only-gate.md](references/read-only-gate.md). |
> | 1 | `bithumb account order-chance --market <m>` | Confirm available balance, fee rate, min/max order size |
> | 2 | `bithumb market ticker <m>` | Confirm current price (vs. user's intended price) |
> | 3 | (for batch) read the JSON file aloud back to the user; (for TWAP) compute slice count = `duration/frequency` and check against minimum order size from step 1 |
> | 3b | **For every `limit` order, check the price against the price tick — regardless of whether it was calculated, derived by percentage, or stated directly by the user** (e.g. "buy at 123,456"). If it is not tick-aligned, round to the nearest valid tick using the [order-commands.md price tick table](references/order-commands.md), and display both the original and adjusted values in step 4. Skip only when the price is already tick-aligned. |
> | 4 | **🤚 Stop. Ask user to confirm**: market, side, order_type, price, volume, total notional |
> | 5 | `bithumb trade place --order-type ...` (or `batch-place` / `twap place`) | Execute |
> | 6 | `bithumb trade get --order-id <id>` (or `trade list`, `twap list`) | Verify state |
>
> The same checklist is mirrored in [bithumb-account Cross-Skill Workflows](../bithumb-account/SKILL.md#cross-skill-workflows).

### Step 1 — Identify order type and load reference

| User intent | Reference to load |
|---|---|
| Single order: place, cancel, query | `{baseDir}/references/order-commands.md` |
| Batch orders: place or cancel multiple | `{baseDir}/references/batch-commands.md` |
| TWAP orders: time-weighted execution | `{baseDir}/references/twap-commands.md` |

### Step 2 — Confirm write parameters

**Read commands** (`trade get`, `trade list`, `twap list`): run immediately, no confirmation needed.

**Write commands** (place, cancel, batch, TWAP): first apply the [read-only gate](references/read-only-gate.md) (skip if already checked this session), then confirm key details once before executing:
- **Place order**: confirm `--market`, `--side`, `--order-type` (canonical; `--ord-type` is a deprecated alias), `--volume`/`--price`
- **Cancel order**: confirm `--order-id` or `--client-order-id`
- **Batch place**: confirm all orders in the batch
- **Batch cancel**: confirm order IDs to cancel
- **TWAP place**: confirm `--market`, `--side`, `--duration`, `--frequency`, total amount

### Step 3 — Verify after writes

- After `trade place`: run `bithumb trade get --order-id <id>` to confirm order status
- After `trade cancel`: run `bithumb trade list` to confirm order is cancelled
- After `trade batch-place`: run `bithumb trade list` to confirm all orders
- After `twap place`: run `bithumb twap list` to confirm TWAP is active

### Step 4 — order_not_found 처리 절차

`trade cancel` 또는 `batch-cancel` 실행 후 `order_not_found` 에러가 반환되면 **`trade list` 리스트 조회 대신 반드시 `trade get --order-id <id>` 단건 직접 조회**로 상태를 확인한다.

> `trade list --state done/cancel --limit N`은 limit 범위 초과 또는 마켓 필터 누락으로 해당 주문을 놓칠 수 있다.

**단건 취소 실패 시:**
```bash
bithumb trade get --order-id <id>
```
- `state: done` → "이미 체결된 주문입니다. 취소 불가합니다." 안내
- `state: cancel` → "이미 취소된 주문입니다." 안내
- 조회 자체 실패(`order_not_found`) → **Step 5 (ID 라우팅 폴백)** 절차 적용

**다건 취소 중 일부 실패 시:**
- 실패한 각 order-id에 대해 `trade get --order-id <id>` 개별 조회 수행
- 또는 `bithumb trade list --order-ids id1,id2,... --states wait,done,cancel` 일괄 조회

### Step 5 — `order_id` ↔ `client_order_id` 라우팅 폴백

빗썸 `client_order_id`에는 **형식 제약(1–36자; 영문자·숫자·하이픈·언더스코어만)** 이 있으나, 그 제약 안에서 사용자가 빗썸 `order_id` 패턴(`C` 접두 + 숫자, 예 `C0101000000002157190`)과 동일하게 만들어 둘 수 있다(`order_id`도 같은 문자셋에 들어옴). **입력값의 문자열 형식만으로 두 식별자를 구분할 수 없으므로 추측하지 않는다.**

단건 주문 조회(`trade get`) 또는 취소(`trade cancel`)에서 다음 절차를 따른다:

1. 사용자가 명시적으로 `client-order-id`를 지목한 경우 → 즉시 `--client-order-id`로 실행. 폴백 없음.
2. 그 외 모든 경우(사용자가 단순히 "주문 ID", "주문번호", 또는 ID만 언급) →
   - **(1차)** `--order-id <값>`으로 먼저 실행.
   - **(2차)** 결과가 없으면(`order_not_found` / 404) 같은 값을 `--client-order-id <값>`으로 다시 실행.
   - **(둘 다 실패)** "이 ID가 현재 계정의 주문이 아니거나, 잘못된 ID이거나, client-order-id 형식이지만 등록되지 않은 값일 수 있습니다." 로 사용자에게 안내.
3. 2차에서 매칭되면 결과 위에 한 줄로 "client-order-id로 매칭됨"을 명시해 사용자가 본인 입력이 사실 client_order_id였음을 인지하도록 한다.

> **CLI 본체는 자동 폴백하지 않는다.** 폴백은 본 가이드(에이전트)의 책임이며 두 번의 명시적 호출로 수행한다. 직접 CLI 사용자가 `--order-id`를 명시한 호출은 그대로 실행되고 404가 그대로 반환된다.

---

## Quickstart

```bash
# Market buy BTC with 100,000 KRW (시장가 매수 — specify KRW amount)
bithumb trade place --market KRW-BTC --side bid --order-type price --price 100000

# Limit buy 0.01 BTC at 135,000,000 KRW
bithumb trade place --market KRW-BTC --side bid --order-type limit \
  --volume 0.01 --price 135000000

# Market sell 0.01 BTC (시장가 매도 — specify BTC volume)
bithumb trade place --market KRW-BTC --side ask --order-type market --volume 0.01

# Limit sell 0.5 ETH at 5,000,000 KRW
bithumb trade place --market KRW-ETH --side ask --order-type limit \
  --volume 0.5 --price 5000000

# Cancel an order
bithumb trade cancel --order-id <order-id>

# Check open orders
bithumb trade list --state wait

# Check completed orders
bithumb trade list --state done --limit 10

# TWAP buy BTC over 1 hour, every 30 seconds
bithumb twap place --market KRW-BTC --side bid \
  --price 100000000 --duration 3600 --frequency 30
```

---

## Edge Cases

- **Order types**:
  - `limit`: requires both `--price` and `--volume`
  - `price`: market buy — `--price` is the **total KRW amount** to spend (no volume needed)
  - `market`: market sell — `--volume` is the **coin quantity** to sell (no price needed)
- **Side**: `bid` = buy, `ask` = sell
- **Market code format**: `KRW-BTC` (quote-base), not `BTC-KRW`
- **Minimum order**: varies by market. Check `bithumb account order-chance --market KRW-BTC` for limits.
- **Batch limits**: max 20 orders per `batch-place`, max 30 per `batch-cancel`
- **TWAP duration**: min 300 seconds (5 min), max 43200 seconds (12 hours)
- **TWAP frequency**: must be one of `15`, `20`, `30`, `60`, `120` seconds
- **Client order ID**: optional `--client-order-id` for idempotency. Constraint (server-enforced): **1–36 chars; letters, digits, hyphens (`-`), underscores (`_`) only**. Over-length or other characters are rejected with `invalid_parameter` — surface this to the user *before* calling rather than after.
- **Order states**: `wait` (open), `watch` (reserved), `done` (filled), `cancel` (cancelled)
- **No derivatives**: Bithumb is spot-only. No leverage, no margin, no futures/options.
- **`--json`**: add to any command for raw JSON output

## Global Notes

- All write commands require valid API credentials
- Bithumb has no demo/simulated trading mode — all orders are real
- Always verify order details before placing — orders are executed with real funds
- Use `bithumb system audit` (in `bithumb-system`) to review audit logs of past operations
- `--json` returns raw Bithumb API response
