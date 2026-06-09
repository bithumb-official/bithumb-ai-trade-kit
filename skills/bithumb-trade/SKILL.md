---
name: bithumb-trade
description: "Place, cancel, and query Bithumb spot orders: single orders (limit, market buy/sell), batch orders (up to 20), batch cancel (up to 30), TWAP orders, and order history. Requires API credentials. 빗썸 현물 주문 생성·취소·조회, 배치 주문(최대 20건), 배치 취소(최대 30건), TWAP 주문을 처리합니다. 매수·매도·지정가·시장가·주문 취소·주문 조회·다건 주문·TWAP 관련 요청에 사용하세요."
---

# Bithumb CEX Trading CLI

Place, cancel, and monitor single or batch spot orders; run TWAP (Time-Weighted Average Price) orders for large positions. **Requires API credentials. All orders are real funds — Bithumb has no demo/simulated mode.**

## Tool Routing

Writes are **CLI-first**: prefer the documented `bithumb ...` CLI; fall back to MCP only when the CLI is unavailable/fails or the user asks for MCP. The CLI carries safety mechanisms MCP lacks (per-profile read-only gating, account isolation, terminal reproducibility). Read-only queries: either path is fine.

## Prerequisites

- Install: `npm install -g @bithumb-official/bithumb-cli`
- Credentials: `BITHUMB_ACCESS_KEY` / `BITHUMB_SECRET_KEY` env vars, or `~/.bithumb/config.toml`.

## Skill Routing

Market data → `bithumb-market`; assets/wallet/API keys → `bithumb-account`; deposits → `bithumb-deposit`; withdrawals → `bithumb-withdraw`; audit logs/diagnostics → `bithumb-system`; orders → this skill.

## Credential Check (before any authenticated command)

1. Run `bithumb system diagnose`. If it fails, **STOP — do not retry.** Inform the user (in Korean): `"인증 실패. API 키가 유효하지 않거나 만료되었을 수 있습니다."` and guide them to set `BITHUMB_ACCESS_KEY`/`BITHUMB_SECRET_KEY` or check `~/.bithumb/config.toml`. Re-run `bithumb system diagnose` after they update, then retry the original operation.
2. If OK, verify trading permissions for the target market with `bithumb account order-chance --market <m>` (also confirms balance, fee rates, order limits).

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb trade place` | WRITE | Place spot order (limit / price / market) |
| 2 | `bithumb trade cancel` | WRITE | Cancel order by order-id or client-order-id |
| 3 | `bithumb trade get` | READ | Get a SINGLE order by one order-id / client-order-id |
| 4 | `bithumb trade list` | READ | Query MULTIPLE orders or list by filters |
| 5 | `bithumb trade batch-place` | WRITE | Place multiple orders at once (max 20) |
| 6 | `bithumb trade batch-cancel` | WRITE | Cancel multiple orders at once (max 30) |
| 7 | `bithumb twap place` | WRITE | Place TWAP order (time-weighted execution) |
| 8 | `bithumb twap list` | READ | Get TWAP order history |
| 9 | `bithumb twap cancel` | WRITE | Cancel active TWAP order |

> **Querying multiple orders:** When the user asks for two or more order numbers at once, use `trade list --order-ids id1,id2` or `--client-order-ids id1,id2` (MCP `trade_get_orders` with the `order_ids`/`client_order_ids` array) — a single call (up to 100 ids; for more, split into batches of 100). Do NOT call `trade get` repeatedly per id. The server defaults to `state=wait`; to include done/cancelled add `--states wait,done,cancel`. `watch` cannot be mixed with `wait`/`done`/`cancel` (query separately with `--state watch`); `--state`/`--states` cannot be used together.

For full parameter details load the reference files in Step 1 below.

---

## Operation Flow

### Step 0 — Credential Check

See [Credential Check](#credential-check-before-any-authenticated-command) above before any authenticated command.

> ### 🛑 Mandatory Pre-flight Checklist — `trade place` / `batch-place` / `twap place`
>
> **Before** any write order, run all of the following in order. Do **not** skip even if the user says "just buy it". Stop and ask for explicit user confirmation between step 3 and step 4.
>
> | # | Command | Purpose |
> |---|---|---|
> | 0 | `bithumb config show --json` | **Read-only gate (run first).** Read the effective profile's `read_only`. If `true`, **stop here** — do not run steps 1–2. List write-capable profiles and ask the user to re-run with `--profile <name>` (**never bypass read-only** — do not auto-select a profile, switch profiles, or disable `read_only` yourself). Once-per-session: skip if already checked. See [read-only-gate.md](references/read-only-gate.md). |
> | 1 | `bithumb account order-chance --market <m>` | Confirm available balance, fee rate, min/max order size |
> | 2 | `bithumb market ticker <m>` | Confirm current price (vs. user's intended price) |
> | 3 | (for batch) read the JSON file aloud back to the user; (for TWAP) compute slice count = `duration/frequency` and check against minimum order size from step 1 |
> | 3b | **For every `limit` order, check the price against the price tick — regardless of whether it was calculated, derived by percentage, or stated directly by the user** (e.g. "buy at 123,456"). If not tick-aligned, round to the nearest valid tick using the [order-commands.md price tick table](references/order-commands.md), and display both original and adjusted values in step 4. Skip only when already tick-aligned. |
> | 4 | **🤚 Stop. Ask user to confirm**: market, side, order_type, price, volume, total notional |
> | 5 | `bithumb trade place --order-type ...` (or `batch-place` / `twap place`) | Execute |
> | 6 | `bithumb trade get --order-id <id>` (or `trade list`, `twap list`) | Verify state |
>
> Mirrored in [bithumb-account Cross-Skill Workflows](../bithumb-account/SKILL.md#cross-skill-workflows).

### Step 1 — Identify order type and load reference

| User intent | Reference to load |
|---|---|
| Single order: place, cancel, query | `{baseDir}/references/order-commands.md` |
| Batch orders: place or cancel multiple | `{baseDir}/references/batch-commands.md` |
| TWAP orders: time-weighted execution | `{baseDir}/references/twap-commands.md` |

### Step 2 — Confirm write parameters

**Reads** (`trade get`/`list`, `twap list`): run immediately, no confirmation.

**Writes** (place, cancel, batch, TWAP): first apply the [read-only gate](references/read-only-gate.md) (skip if already checked this session), then confirm key details once:
- **Place**: `--market`, `--side`, `--order-type` (canonical; `--ord-type` is a deprecated alias), `--volume`/`--price`
- **Cancel**: `--order-id` or `--client-order-id`
- **Batch place / cancel**: all orders / order IDs in the batch
- **TWAP place**: `--market`, `--side`, `--duration`, `--frequency`, total amount

### Step 3 — Verify after writes

- After `trade place`: `bithumb trade get --order-id <id>` to confirm status
- After `trade cancel`: `bithumb trade list` to confirm cancellation
- After `batch-place`: `bithumb trade list` to confirm all orders
- After `twap place`: `bithumb twap list` to confirm TWAP is active

### Step 4 — Handling `order_not_found`

When `trade cancel` or `batch-cancel` returns `order_not_found`, **always confirm state with a direct single lookup `trade get --order-id <id>`, not a `trade list` query** — `trade list --state done/cancel --limit N` can miss the order by exceeding the limit or omitting the market filter.

On a failed single cancel, run `bithumb trade get --order-id <id>`:
- `state: done` → tell the user (in Korean): `"이미 체결된 주문입니다. 취소 불가합니다."`
- `state: cancel` → tell the user (in Korean): `"이미 취소된 주문입니다."`
- lookup itself fails (`order_not_found`) → apply **Step 5 (ID routing fallback)**.

On partial batch-cancel failures, run `trade get --order-id <id>` per failed id, or look them up together with `bithumb trade list --order-ids id1,id2,... --states wait,done,cancel`.

### Step 5 — `order_id` ↔ `client_order_id` routing fallback

A Bithumb `client_order_id` has a **format constraint (1–36 chars; letters, digits, hyphens, underscores only)**, but within that constraint a user can make one identical to a Bithumb `order_id` pattern (`C` prefix + digits, e.g. `C0101000000002157190`) — `order_id` uses the same character set. **You cannot tell the two identifiers apart from the input string alone, so do not guess.**

For single-order `trade get` or `trade cancel`:

1. If the user explicitly names a `client-order-id` → run with `--client-order-id` immediately. No fallback.
2. Otherwise (user just says "order ID" / "order number", or gives an id alone):
   - **(1st)** run with `--order-id <value>` first.
   - **(2nd)** if no result (`order_not_found` / 404), retry the same value with `--client-order-id <value>`.
   - **(both fail)** tell the user (in Korean): `"이 ID가 현재 계정의 주문이 아니거나, 잘못된 ID이거나, client-order-id 형식이지만 등록되지 않은 값일 수 있습니다."`
3. If the 2nd attempt matches, add a one-line note above the result (in Korean): `"client-order-id로 매칭됨"`, so the user knows their input was actually a client_order_id.

> **The CLI core does not auto-fallback.** The fallback is this guide's (the agent's) responsibility, done as two explicit calls. A direct CLI user who passes `--order-id` runs as-is and gets the 404 verbatim.

---

## Edge Cases

- **Order types**: `limit` requires `--price` + `--volume`; `price` = market buy, `--price` is the **total KRW to spend** (no volume); `market` = market sell, `--volume` is the **coin quantity** (no price).
- **Side**: `bid` = buy, `ask` = sell.
- **Market code**: `KRW-BTC` (quote-base), not `BTC-KRW`.
- **Batch limits**: max 20 per `batch-place`, max 30 per `batch-cancel`.
- **TWAP**: duration 300–43200 s (5 min–12 h); frequency one of `15`, `20`, `30`, `60`, `120` s.
- **Client order ID**: optional `--client-order-id`. Constraint (server-enforced): **1–36 chars; letters, digits, hyphens (`-`), underscores (`_`) only**. Over-length or other characters → `invalid_parameter`; surface this *before* calling.
- **Order states**: `wait` (open), `watch` (reserved), `done` (filled), `cancel` (cancelled).
- **Spot-only**: no leverage, margin, futures, or options.

## Global Notes

- When you query orders (`trade list` / `trade get`) and present the result to the user, always include each order's `order_id` — they need it to look up or cancel the order later.
- All write commands require valid API credentials. Bithumb has no demo mode — every order is real funds, so always verify details before placing.
- Add `--json` to any command for the raw Bithumb API response.
- Review past operations with `bithumb system audit` (in `bithumb-system`).
