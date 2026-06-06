# Price & Market Data Commands

## market ticker — Current Price Ticker

```bash
bithumb market ticker KRW-BTC
bithumb market ticker KRW-BTC,KRW-ETH,KRW-XRP
```

| Param | Required | Description |
|---|---|---|
| `<markets>` | Yes | Comma-separated market codes (e.g., `KRW-BTC,KRW-ETH`) |

Returns per market: `market`, `trade_price` (last), `high_price` (24h high), `low_price` (24h low), `acc_trade_volume_24h` (24h volume), `signed_change_rate` (24h change %), `trade_timestamp`

```bash
# Single ticker
bithumb market ticker KRW-BTC
# → KRW-BTC | trade_price: 138,500,000 | change: +2.3% | high: 140,000,000 | low: 135,000,000

# Multiple tickers
bithumb market ticker KRW-BTC,KRW-ETH

# JSON output
bithumb market ticker KRW-BTC --json
```

---

## market orderbook — Order Book Depth

```bash
bithumb market orderbook KRW-BTC
bithumb market orderbook KRW-BTC,KRW-ETH
```

| Param | Required | Description |
|---|---|---|
| `<markets>` | Yes | Comma-separated market codes |

Returns: `orderbook_units` array with `ask_price`, `ask_size`, `bid_price`, `bid_size` for each level. Also includes `total_ask_size`, `total_bid_size`.

```bash
bithumb market orderbook KRW-BTC
# Asks: 138,600,000 / 0.5 BTC · 138,550,000 / 1.2 BTC ...
# Bids: 138,500,000 / 0.8 BTC · 138,450,000 / 0.3 BTC ...
```

---

## market trades — Recent Public Trades

```bash
bithumb market trades KRW-BTC
bithumb market trades KRW-BTC --count 50
bithumb market trades KRW-BTC --days-ago 1
```

| Param | Required | Default | Description |
|---|---|---|---|
| `<market>` | Yes | - | Market code |
| `--count` | No | 1 | Number of trades (1–500) |
| `--days-ago` | No | - | Filter trades from N days ago (1–7). Without this, query targets current time. |
| `--cursor` | No | - | Pagination cursor (sequential_id from previous response) |
| `--to` | No | - | Time-of-day base (KST). Format `HHmmss` or `HH:mm:ss`. Returns trades before this time. **Date is not accepted** — for past days use `--days-ago`. |

Returns: `trade_price`, `trade_volume`, `ask_bid` (ASK/BID), `trade_date_utc`, `trade_time_utc`, `sequential_id`

```bash
bithumb market trades KRW-BTC --count 20
```

---

## market candles-minutes — Minute Candles (OHLCV)

```bash
bithumb market candles-minutes KRW-BTC --unit 60
bithumb market candles-minutes KRW-BTC --unit 5 --count 100
bithumb market candles-minutes KRW-BTC --unit 240 --count 50 --to "2025-04-01T00:00:00"
```

| Param | Required | Default | Description |
|---|---|---|---|
| `<market>` | Yes | - | Market code (e.g., `KRW-BTC`) |
| `--unit` | No | 1 | Candle interval: `1`, `3`, `5`, `10`, `15`, `30`, `60`, `240` |
| `--count` | No | 1 | Number of candles (1–200) |
| `--to` | No | - | Return candles before this timestamp (ISO 8601) |

Returns per candle: `candle_date_time_utc`, `opening_price`, `high_price`, `low_price`, `trade_price` (close), `candle_acc_trade_volume` (volume), `candle_acc_trade_price` (turnover)

```bash
# 1-hour candles, last 24
bithumb market candles-minutes KRW-BTC --unit 60 --count 24

# 4-hour candles for ETH
bithumb market candles-minutes KRW-ETH --unit 240 --count 50

# 5-minute candles before specific time
bithumb market candles-minutes KRW-BTC --unit 5 --count 100 --to "2025-04-28T12:00:00"
```

---

## market candles-days — Daily Candles

```bash
bithumb market candles-days KRW-BTC
bithumb market candles-days KRW-BTC --count 30
bithumb market candles-days KRW-BTC --count 90 --to "2025-01-01T00:00:00"
```

| Param | Required | Default | Description |
|---|---|---|---|
| `<market>` | Yes | - | Market code |
| `--count` | No | 1 | Number of candles (1–200) |
| `--to` | No | - | Return candles before this timestamp |
| `--converting-price-unit` | No | - | Convert the close price to KRW for non-KRW markets (e.g., `BTC-ETH`); result populates `converted_trade_price`, visible only with `--json` (default table output omits it). Bithumb supports `KRW` only; KRW markets return `null`. |

Returns same OHLCV fields as minute candles, plus `prev_closing_price` and `change_price`.

```bash
# Last 30 daily candles
bithumb market candles-days KRW-BTC --count 30

# Daily candles before a date
bithumb market candles-days KRW-BTC --count 60 --to "2025-03-01T00:00:00"

# BTC 마켓 페어(BTC-ETH 등)의 종가를 KRW로 환산해 KRW 마켓과 비교 (converted_trade_price 필드 사용)
# KRW 마켓은 이미 원화 기준이므로 사용하지 않음; 사용해도 converted_trade_price는 null
bithumb market candles-days BTC-ETH --converting-price-unit KRW --json
```

---

## market candles-weeks — Weekly Candles

```bash
bithumb market candles-weeks KRW-BTC
bithumb market candles-weeks KRW-BTC --count 52
```

| Param | Required | Default | Description |
|---|---|---|---|
| `<market>` | Yes | - | Market code |
| `--count` | No | 1 | Number of candles (1–200) |
| `--to` | No | - | Return candles before this timestamp |

Returns same OHLCV fields as daily candles.

---

## market candles-months — Monthly Candles

```bash
bithumb market candles-months KRW-BTC
bithumb market candles-months KRW-BTC --count 12
```

| Param | Required | Default | Description |
|---|---|---|---|
| `<market>` | Yes | - | Market code |
| `--count` | No | 1 | Number of candles (1–200) |
| `--to` | No | - | Return candles before this timestamp |

Returns same OHLCV fields as daily candles.

```bash
# Last 12 months
bithumb market candles-months KRW-BTC --count 12
```

---

## Notes

- **Market code format**: Always `{quote}-{base}` — e.g., `KRW-BTC` (KRW 마켓), `BTC-ETH` (BTC 마켓).
- **Candle time**: `candle_date_time_utc` is UTC, `candle_date_time_kst` is KST (UTC+9)
- **Pagination**: Use `--to` parameter to paginate backward through time. Set `--to` to the oldest candle's timestamp from the previous response to get the next batch.
- **Candle `--to` format (KST naive, no timezone offset)**: candle `--to` values must be `YYYY-MM-DDTHH:MM:SS` in KST. **Do NOT append a timezone offset** — `+09:00`, `Z`, `+0900`, `-05:00` etc. cause HTTP 400 Invalid parameter. Example: `2026-01-01T00:00:00` ✓ / `2026-01-01T00:00:00+09:00` ✗.
- **Trades `--to` format** (different from candles): time-of-day only. `HHmmss` or `HH:mm:ss` (00:00:00–23:59:59). Date is not accepted — use `--days-ago` (1–7) for past days. Example: `230000` ✓ / `23:00:00` ✓ / `2026-01-01T00:00:00` ✗.
- **Max 200 candles per request**: For longer historical ranges, paginate with `--to`.
- **Volume**: `candle_acc_trade_volume` is in base currency, `candle_acc_trade_price` is in quote currency (turnover).
- **`--json`**: Add to any command for raw JSON output.
