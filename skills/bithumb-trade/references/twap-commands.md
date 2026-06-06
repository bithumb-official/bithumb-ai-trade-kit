# TWAP Order Commands

TWAP (Time-Weighted Average Price) splits a large order into smaller slices executed at regular intervals over a specified duration. This reduces market impact for large positions.

## twap place — Place TWAP Order

```bash
bithumb twap place --market KRW-BTC --side bid --price 10000000 --duration 3600 --frequency 30
```

| Param | Required | Description |
|---|---|---|
| `--market` | Yes | Market code (e.g., `KRW-BTC`) |
| `--side` | Yes | `bid` (buy) or `ask` (sell) |
| `--price` | Cond. | Total KRW amount (required for `bid` / buy) |
| `--volume` | Cond. | Total coin quantity (required for `ask` / sell) |
| `--duration` | Yes | Execution duration in seconds, passed as string (min 300, max 43200) |
| `--frequency` | Yes | Order interval: `15`, `20`, `30`, `60`, or `120` seconds (passed as string) |

### Duration Guide

| Duration | Human Readable |
|---|---|
| `300` | 5 minutes (minimum) |
| `1800` | 30 minutes |
| `3600` | 1 hour |
| `7200` | 2 hours |
| `14400` | 4 hours |
| `28800` | 8 hours |
| `43200` | 12 hours (maximum) |

### Examples

```bash
# TWAP buy BTC with 10M KRW over 1 hour, every 30 seconds
bithumb twap place --market KRW-BTC --side bid --price 10000000 --duration 3600 --frequency 30
# → 120 slices, ~83,333 KRW per slice

# TWAP sell 1 ETH over 2 hours, every 60 seconds
bithumb twap place --market KRW-ETH --side ask --volume 1 --duration 7200 --frequency 60
# → 120 slices, ~0.00833 ETH per slice

# Quick TWAP: buy XRP over 5 minutes, every 15 seconds
bithumb twap place --market KRW-XRP --side bid --price 500000 --duration 300 --frequency 15
# → 20 slices, ~25,000 KRW per slice
```

### Slice Calculation

Number of slices = `duration / frequency`

| Duration | Frequency | Slices |
|---|---|---|
| 3600 (1h) | 30s | 120 |
| 3600 (1h) | 60s | 60 |
| 7200 (2h) | 60s | 120 |
| 43200 (12h) | 120s | 360 |

---

## twap list — Get TWAP Order History

```bash
bithumb twap list
bithumb twap list --state progress
bithumb twap list --market KRW-BTC --state done --limit 10
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--market` | No | - | Filter by market code |
| `--state` | No | `progress` | `progress` (active), `done` (completed), `cancel` (cancelled) |
| `--order-ids` | No | - | Filter by TWAP order IDs (comma-separated) |
| `--limit` | No | `100` | Max results (>=1) |
| `--order-by` | No | `desc` | `asc` or `desc` |
| `--next-key` | No | - | Pagination cursor |

```bash
# Active TWAP orders
bithumb twap list --state progress

# Completed TWAP orders for BTC
bithumb twap list --market KRW-BTC --state done

# All TWAP history
bithumb twap list --state done --limit 50
```

---

## twap cancel — Cancel TWAP Order

```bash
bithumb twap cancel --algo-order-id <twap-uuid>
```

| Param | Required | Description |
|---|---|---|
| `--algo-order-id` | Yes | TWAP order ID to cancel |

Returns: cancellation result. Already-executed slices are not reversed; only remaining slices are cancelled.

```bash
# Cancel active TWAP
bithumb twap cancel --algo-order-id twap-uuid-here

# Verify cancellation
bithumb twap list --state cancel
```

---

## TWAP Workflow

```bash
# 1. Check current price
bithumb market ticker KRW-BTC

# 2. Check available balance
bithumb account order-chance --market KRW-BTC

# 3. Place TWAP order (buy 5M KRW of BTC over 1 hour)
bithumb twap place --market KRW-BTC --side bid --price 5000000 --duration 3600 --frequency 30

# 4. Monitor progress
bithumb twap list --state progress

# 5. Cancel if needed (only remaining slices are cancelled)
bithumb twap cancel --algo-order-id <twap-uuid>
```

---

## Notes

- **Buy vs Sell**: For `bid` (buy), specify `--price` (total KRW). For `ask` (sell), specify `--volume` (total coins).
- **Partial cancel**: Cancelling a TWAP order only stops future slices. Already-executed slices are final.
- **Duration limits**: Minimum 300s (5 min), maximum 43200s (12 hours).
- **Frequency options**: Only `15`, `20`, `30`, `60`, `120` seconds are valid.
- **Market conditions**: TWAP executes at market price for each slice. In volatile markets, the average execution price may differ significantly from the price at order creation.
- **Minimum slice size**: Each slice must meet the market's minimum order size. If `total / slices` is below the minimum, the TWAP may fail or execute fewer slices.
- **`--json`**: Add to any command for raw JSON output.

## Failure Handling — 자동 재시도 금지 (NO automatic retry)

When `twap place` fails (validation error, minimum slice size violation, insufficient balance, etc.):

1. **Do NOT auto-retry**. Show the error to the user with the specific cause.
2. If the error is "slice below minimum order size":
   - Run `bithumb account order-chance --market <m>` to read the **current** minimum order size.
   - Compute `slice_amount = total / (duration / frequency)` and propose adjusted `duration` or `frequency` that keeps each slice ≥ minimum.
   - Present the adjusted plan to the user and **wait for explicit approval** before re-issuing `twap place`.
3. If the error is "insufficient balance": stop. Do NOT auto-retry with a smaller amount unless the user explicitly says so.
4. After successful `twap place`, always verify with `bithumb twap list --state progress`.
