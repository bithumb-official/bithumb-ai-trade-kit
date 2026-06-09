# Batch Order Commands

## trade batch-place — Place Multiple Orders

### CLI Usage

```bash
bithumb trade batch-place --file orders.json
```

| Param | Required | Description |
|---|---|---|
| `--file` | Yes | Path to JSON file containing array of order objects (max 20) |

> **Important**: Use `--file`, not `--orders`. The CLI reads orders from a JSON file. Each order object must use `order_type` (not `ord_type`).

**orders.json** example:

```json
[
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"134000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"133000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"132000000"}
]
```

### Order Object Fields

| Field | Required | Description |
|---|---|---|
| `market` | Yes | Market code (e.g., `KRW-BTC`) |
| `side` | Yes | `bid` (buy) or `ask` (sell) |
| `order_type` | Yes | `limit`, `price`, or `market` — **use `order_type`, not `ord_type`** |
| `price` | Cond. | Required for `limit` and `price` |
| `volume` | Cond. | Required for `limit` and `market` |
| `client_order_id` | No | Client-assigned ID for each order (1–36 chars; letters, digits, hyphens, underscores only) |

> **Naming consistency**: Use `order_type` in batch JSON. Bithumb's response field remains `ord_type` (preserved as-is — do not modify response payloads).

### Examples

```bash
# Place 3 limit buy orders at different prices (grid-style)
cat > /tmp/grid-orders.json << 'EOF'
[
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"134000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"133000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"132000000"}
]
EOF
bithumb trade batch-place --file /tmp/grid-orders.json

# Mixed market and limit orders
cat > /tmp/mixed-orders.json << 'EOF'
[
  {"market":"KRW-BTC","side":"bid","order_type":"price","price":"50000"},
  {"market":"KRW-ETH","side":"bid","order_type":"limit","volume":"0.1","price":"4500000"}
]
EOF
bithumb trade batch-place --file /tmp/mixed-orders.json

# With client order IDs for tracking
cat > /tmp/tracked-orders.json << 'EOF'
[
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"130000000","client_order_id":"grid-1"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"129000000","client_order_id":"grid-2"}
]
EOF
bithumb trade batch-place --file /tmp/tracked-orders.json
```

Returns: array of results for each order (success or error per order).

> **Partial success**: Some orders in the batch may succeed while others fail. Always check the response for each order's status.

---

## trade batch-cancel — Cancel Multiple Orders

```bash
bithumb trade batch-cancel --order-ids id-1,id-2,id-3
bithumb trade batch-cancel --client-order-ids my-order-1,my-order-2
```

| Param | Required | Description |
|---|---|---|
| `--order-ids` | Cond. | Comma-separated order IDs to cancel (max 30) |
| `--client-order-ids` | Cond. | Comma-separated client order IDs to cancel (max 30; each id 1–36 chars; letters, digits, hyphens, underscores only) |

Provide either `--order-ids` or `--client-order-ids` (not both).

### Examples

```bash
# Cancel by order IDs
bithumb trade batch-cancel --order-ids id-1,id-2,id-3

# Cancel by client IDs
bithumb trade batch-cancel --client-order-ids grid-1,grid-2,grid-3
```

Returns: array of cancellation results.

---

## Batch Workflow

### Place grid orders and manage them

```bash
# 1. Check available balance
bithumb account order-chance --market KRW-BTC

# 2. Place grid of limit buy orders
cat > /tmp/grid.json << 'EOF'
[
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"134000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"133000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"132000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"131000000"},
  {"market":"KRW-BTC","side":"bid","order_type":"limit","volume":"0.001","price":"130000000"}
]
EOF
bithumb trade batch-place --file /tmp/grid.json

# 3. Check which orders are still open
bithumb trade list --state wait

# 4. Cancel remaining open orders
bithumb trade batch-cancel --order-ids id-1,id-2
```

---

## Notes

- **Max batch sizes**: 20 orders for `trade batch-place`, 30 orders for `trade batch-cancel`
- **Atomic?**: No — each order in the batch is processed independently. Some may succeed while others fail.
- **Rate limits**: Batch operations count as a single API call, but each order within is validated separately.
- **Cross-market batches**: You can mix different markets in a single batch (e.g., KRW-BTC and KRW-ETH orders together).
- **Verification**: After batch operations, always run `bithumb trade list --state wait` to confirm the actual state.
- **`--json`**: Add to any command for raw JSON output.

## Partial Failure Handling — 자동 재시도 금지 (NO automatic retry)

`batch-place` may return a mix of success and error results per item. When **부분 실패** (partial failure) occurs:

1. **Do NOT auto-retry the failed items.** That can produce duplicate orders or burn balance silently.
2. Separate `success` vs `error` items and present a table on **stderr** showing each failed order's input + the API error reason.
3. Run `bithumb trade list --state wait` to show which orders actually made it.
4. Wait for **explicit user approval** before re-submitting any failed items. Helpful pattern: have the user save only the failed items to a new JSON file (e.g., `retry-batch.json`) and re-run `trade batch-place --file retry-batch.json`.
5. Common failure reasons agents must surface: insufficient balance per item, volume below minimum (per-market), price tick violation, market in maintenance.
