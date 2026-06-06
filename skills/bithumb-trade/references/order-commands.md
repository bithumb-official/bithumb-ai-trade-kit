# Single Order Commands

## trade place — Place Order

```bash
bithumb trade place --market KRW-BTC --side bid --order-type limit --volume 0.01 --price 135000000
```

| Param | Required | Description |
|---|---|---|
| `--market` | Yes | Market code (e.g., `KRW-BTC`) |
| `--side` | Yes | `bid` (buy) or `ask` (sell) |
| `--order-type` | Yes | `limit`, `price` (market buy), or `market` (market sell) — **canonical flag** |
| `--ord-type` | — | Deprecated alias of `--order-type`. Still accepted; emits a deprecation notice. |
| `--price` | Cond. | Required for `limit` and `price` orders |
| `--volume` | Cond. | Required for `limit` and `market` orders |
| `--client-order-id` | No | Client-assigned ID for idempotency (1–36 chars; letters, digits, hyphens, underscores only) |

> **Naming axes (one concept, three contexts)**:
> - CLI flag: `--order-type` (canonical) / `--ord-type` (deprecated alias)
> - Batch JSON field: `order_type` (canonical) / `ord_type` (auto-normalized with notice)
> - Bithumb API response field: `ord_type` (preserved as-is — do not modify response payloads)

### Order Type Details

| order-type | side | price meaning | volume meaning |
|---|---|---|---|
| `limit` | bid/ask | Unit price per coin | Coin quantity |
| `price` | bid only | **Total KRW to spend** | Not needed |
| `market` | ask only | Not needed | **Coin quantity to sell** |

### Price Tick (호가단위)

The minimum price increment for KRW market orders. Fixed policy by price range.

| Price Range | Price Tick |
|---|---|
| Under 1 KRW | 0.0001 |
| 1 ~ under 10 KRW | 0.001 |
| 10 ~ under 100 KRW | 0.01 |
| 100 ~ under 1,000 KRW | 1 |
| 1,000 ~ under 5,000 KRW | 1 |
| 5,000 ~ under 10,000 KRW | 5 |
| 10,000 ~ under 50,000 KRW | 10 |
| 50,000 ~ under 100,000 KRW | 50 |
| 100,000 ~ under 500,000 KRW | 100 |
| 500,000 ~ under 1,000,000 KRW | 500 |
| 1,000,000 KRW and above | 1,000 |

> This table applies to order **price** only — unrelated to volume precision. Also distinct from candle `convertingPriceUnit` (non-KRW closing price converted to KRW).

#### 호가단위 정규화 (모든 `limit` 가격에 적용 — 출처 무관)

`limit` 주문을 넣기 전, 가격이 호가단위에 정렬돼 있는지 **가격의 출처와 무관하게** 확인한다. 다음 두 경우에 똑같이 적용된다.

- **계산·퍼센트로 도출한** 가격 (예: "현재가 -5%" → 27,512), 그리고
- **사용자가 직접 명시한** 가격 (예: "123,456원에 매수").

빗썸은 호가단위에 맞지 않는 주문을 거부하므로, 사용자가 직접 말한 값이라도 비정렬이면 그대로 넣을 수 없다. 절차:

1. 가격이 속한 구간의 호가단위를 찾는다.
2. 가장 가까운 유효 호가로 **반올림(round)** 한다. (예: 27,512원 → 호가단위 10원 → 27,510원; 123,456원 → 호가단위 100원 → 123,500원)
3. 사용자 확인 단계에서 **보정 전(원본)과 보정 후(정규화값)를 나란히** 표시한다. 의도 방향(싸게/비싸게)이 반올림으로 어긋날 수 있으므로 사용자가 직접 판단할 수 있게 한다.

> 가격이 **이미 호가단위에 정렬된** 경우에만 정규화를 건너뛰고 그대로 주문한다. 판단 기준은 "사용자가 가격을 명시했는가"가 아니라 "최종 가격이 호가단위에 정렬됐는가"다. 가격 보정은 가이드(에이전트)의 책임이며 CLI/서버 코어는 가격을 자동 보정하지 않는다.

### Examples

```bash
# Limit buy 0.01 BTC at 135M KRW
bithumb trade place --market KRW-BTC --side bid --order-type limit --volume 0.01 --price 135000000

# Market buy BTC with 100,000 KRW
bithumb trade place --market KRW-BTC --side bid --order-type price --price 100000

# Market sell 0.5 ETH
bithumb trade place --market KRW-ETH --side ask --order-type market --volume 0.5

# Limit sell 100 XRP at 3,500 KRW with client ID
bithumb trade place --market KRW-XRP --side ask --order-type limit --volume 100 --price 3500 --client-order-id my-order-001
```

### Common Errors

- `price` order with `--side ask` → invalid (market buy only)
- `market` order with `--side bid` → invalid (market sell only)
- Volume below minimum → check `bithumb account order-chance --market KRW-BTC` for market limits
- Insufficient balance → check `bithumb account assets` first
- Confusing response field `ord_type` with input flag: response retains Bithumb's original `ord_type` field. For input always use `--order-type` (CLI) or `order_type` (JSON).

---

## trade cancel — Cancel Order

```bash
bithumb trade cancel --order-id <order-id>
bithumb trade cancel --client-order-id my-order-001
```

| Param | Required | Description |
|---|---|---|
| `--order-id` | Cond. | Order ID (provide one of order-id or client-order-id) |
| `--client-order-id` | Cond. | Client-assigned order ID (1–36 chars; letters, digits, hyphens, underscores only) |

Returns: cancelled order details.

> Only `wait` (open) or `watch` (reserved) orders can be cancelled.

#### Handling order_not_found Errors

When `order_not_found` is returned, **always use `trade get --order-id <id>` to look up the order directly** instead of querying via `trade list`.
`trade list --limit N` may miss the order due to exceeding the limit or a missing market filter.

```bash
bithumb trade get --order-id <id>
# state: done   → already filled — cannot cancel
# state: cancel → already cancelled
# order_not_found → belongs to a different account or does not exist
```

---

## trade get — Get Single Order

```bash
bithumb trade get --order-id <order-id>
bithumb trade get --client-order-id my-order-001
```

| Param | Required | Description |
|---|---|---|
| `--order-id` | Cond. | Order ID |
| `--client-order-id` | Cond. | Client-assigned order ID (1–36 chars; letters, digits, hyphens, underscores only) |

Returns: full order details including `state`, `side`, `ord_type`, `price`, `volume`, `remaining_volume`, `executed_volume`, `trades_count`, `created_at`.

---

## trade list — List Orders

```bash
bithumb trade list
bithumb trade list --state wait
bithumb trade list --market KRW-BTC --state done --limit 20
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--market` | No | - | Filter by market code |
| `--state` | No | - | Filter: `wait`, `watch`, `done`, `cancel` |
| `--states` | No | - | Filter by multiple states (comma-separated, e.g., `wait,watch`) |
| `--order-ids` | No | - | Filter by order IDs (comma-separated, max 100) |
| `--client-order-ids` | No | - | Filter by client order IDs (comma-separated, max 100) |
| `--page` | No | 1 | Page number (>=1) |
| `--limit` | No | `100` | Results per page (>=1) |
| `--order-by` | No | `desc` | Sort order: `asc` (oldest first) or `desc` (newest first) |

Returns: array of orders matching filters.

### Common Queries

```bash
# All open orders
bithumb trade list --state wait

# Completed BTC orders (last 20)
bithumb trade list --market KRW-BTC --state done --limit 20

# Cancelled orders
bithumb trade list --state cancel --limit 10
```

---

## Notes

- **Idempotency**: Use `--client-order-id` to prevent duplicate orders. If the same ID is sent twice, the second request returns the existing order instead of creating a new one.
- **Order state flow**: `wait` → `done` (filled) or `cancel` (cancelled). `watch` is a reserved intermediate state.
- **Partial fills**: Check `remaining_volume` vs `volume` to see if an order is partially filled.
- **Pre-order check**: Always run `bithumb account order-chance --market KRW-BTC` before placing an order to verify available balance, fee rates, and minimum order size.
- **`--json`**: Add to any command for raw JSON output.
