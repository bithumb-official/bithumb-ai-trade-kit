---
name: bithumb-market
description: "Read-only public market data for Bithumb: prices, ticker, order book, candles (minute/day/week/month), recent trades, market list, virtual asset warnings, notices, and fees. No API credentials required. Do NOT use for wallet/blockchain status or deposit-withdrawal availability — use bithumb-account. 빗썸 시세·호가·캔들·체결 내역·마켓 목록·유의 종목·경보제·공지사항·수수료 등 공개 시장 데이터를 조회합니다. 인증 불필요. 빗썸 시세·차트·호가·체결 내역·유의 종목·경보제 관련 요청에 사용하세요. 입출금 현황·블록 갱신시각·블록 동기화·입출금 가능 여부는 bithumb-account를 사용하세요."
---

# Bithumb CEX Market Data CLI

> **Compliance notice**: This skill provides raw market data only. No strategy, recommendation, or optimization logic is embedded. All outputs are objective numerical values; interpretation and trading decisions remain solely with the user.

Public market data for Bithumb: prices, order books, candles, recent trades, market list, 유의 종목 지정, 경보제(주의 종목), notices, and fees. All commands are **read-only** and require **no API credentials**.

**Skill routing**
- Market data → `bithumb-market` (this skill)
- Account balance / wallet status → `bithumb-account`
- Place / cancel orders → `bithumb-trade`
- Deposits → `bithumb-deposit`
- Withdrawals → `bithumb-withdraw`
- Audit logs / diagnostics → `bithumb-system`

**Risk-flag intent routing (유의 종목 vs 경보제)**

빗썸은 두 개의 독립적인 거래 위험 표시 제도를 운영한다. 사용자 의도에 따라 한쪽만 호출한다.

| 사용자 의도 | 호출 명령 | 제도 |
|---|---|---|
| 단순 마켓 목록 조회 | `bithumb market markets` (isDetails 생략 — API default: false) | — |
| "거래유의 / 유의 종목" 명시적 요청 | `bithumb market markets --is-details` (`market_warning` 필드) | 유의 종목 지정 — 거래소 심사 기반 상시·정성적 |
| "주의 종목 / 경보 / 이상거래 탐지" | `bithumb market warnings` | 경보제 — 시장 데이터 기반 자동·정량적 탐지 |
| 모호한 표현 (예: "유의 상태", "위험한 코인") | 두 제도가 다름을 알리고 사용자에게 어느 쪽인지 확인 후 한쪽만 호출 | — |

두 API를 자동으로 둘 다 호출하지 않는다. 사용자가 명시적으로 "둘 다 / 전부"라고 요청한 경우에만 예외.

## Install

- **CLI**: `npm install -g @bithumb-official/bithumb-cli`

No API credentials required for market data.

```bash
bithumb market ticker KRW-BTC   # verify
```

## Command Index

| # | Command | Description |
|---|---|---|
| 1 | `bithumb market markets` | List all tradeable pairs (KRW and BTC markets) |
| 2 | `bithumb market ticker <markets>` | Current price ticker: last price, 24h high/low/vol/change |
| 3 | `bithumb market orderbook <markets>` | Order book bids/asks for one or more markets |
| 4 | `bithumb market trades <market>` | Recent public trade history |
| 5 | `bithumb market candles-minutes <market>` | Minute candles (1/3/5/10/15/30/60/240 min) |
| 6 | `bithumb market candles-days <market>` | Daily candles |
| 7 | `bithumb market candles-weeks <market>` | Weekly candles |
| 8 | `bithumb market candles-months <market>` | Monthly candles |
| 9 | `bithumb market warnings` | 경보제 — 시장 데이터 기반 이상거래 자동 탐지 (주의 종목) |
| 10 | `bithumb market notices` | Exchange notice/announcement list |
| 11 | `bithumb market fee-inout <currency>` | Deposit/withdrawal fee info by currency |

---

## Operation Flow

### Step 1 — Identify data type and load reference

| User intent | Reference to load |
|---|---|
| Price, candles, order book, recent trades | `{baseDir}/references/price-data-commands.md` |
| Market list, warnings, notices, fee info | `{baseDir}/references/instrument-commands.md` |

### Step 2 — Run commands immediately

All market data commands are read-only — no confirmation needed.

### Step 3 — No writes, no verification needed

All commands in this skill are read-only.

---

## Quickstart

```bash
# Get BTC current price
bithumb market ticker KRW-BTC

# Get BTC and ETH prices at once
bithumb market ticker KRW-BTC,KRW-ETH

# Get BTC order book
bithumb market orderbook KRW-BTC

# Get BTC 1-hour candles (last 24)
bithumb market candles-minutes KRW-BTC --unit 60 --count 24

# Get BTC daily candles (last 30 days)
bithumb market candles-days KRW-BTC --count 30

# List all tradeable markets
bithumb market markets

# Check 유의 종목 지정 (market_warning: NONE/CAUTION)
bithumb market markets --is-details

# Check 경보제 (주의 종목 — 단계·사유·해제시각 포함)
bithumb market warnings

# Get latest notices
bithumb market notices --count 5

# Check BTC deposit/withdrawal fees
bithumb market fee-inout BTC

# Check all currencies' fees
bithumb market fee-inout ALL
```

---

## Edge Cases

- **Market code format**: `KRW-BTC` (quote-base), not `BTC-KRW`. Always `{quote}-{base}`.
- **Reversed market code auto-correction**: If the user inputs a reversed code like `BTC-KRW`, automatically convert it to `KRW-BTC` and proceed. Inform the user of the conversion (e.g., "Converted BTC-KRW → KRW-BTC").
- **Multiple tickers**: pass comma-separated market codes to `ticker` and `orderbook` (e.g., `KRW-BTC,KRW-ETH`)
- **Candle units**: minute candles support `1, 3, 5, 10, 15, 30, 60, 240` only
- **Candle count**: min 1, max 200, default 1 for all candle endpoints
- **Candle `--to` param**: KST naive datetime (`YYYY-MM-DDTHH:MM:SS`); returns candles before this time. **Do NOT include a timezone offset (`+09:00`, `Z`, `+0900`, etc.) — Bithumb API rejects offset-bearing values with HTTP 400 Invalid parameter.** Examples: `2026-01-01T00:00:00` ✓ / `2026-01-01T00:00:00+09:00` ✗ / `2026-01-01T00:00:00Z` ✗
- **Trades `--to` param** (different from candles): time-of-day only. Format `HHmmss` or `HH:mm:ss` (00:00:00–23:59:59). Examples: `230000` ✓ / `23:00:00` ✓. **Date is not accepted** — `2026-01-01T00:00:00` ✗. For past days use `--days-ago` (1–7). When omitted: without `--days-ago` → current time; with `--days-ago` → that day's 00:00:00 base.
- **`--converting-price-unit`**: 비KRW 마켓(예: `BTC-ETH`)의 종가를 원화로 환산할 때 사용. 빗썸은 `KRW`만 지원한다. 환산 결과(`converted_trade_price`)는 `--json` 출력에서만 보인다(기본 테이블 출력에는 없음). KRW 마켓(예: `KRW-BTC`)은 이미 원화 기준이라 `converted_trade_price: null`.
- **`fee-inout`**: use `ALL` for currency param to get all currencies at once
- **No derivatives**: Bithumb is spot-only. No swap, futures, or options data.
- **유의 종목 지정**: `bithumb market markets --is-details`의 `market_warning` 필드 (값: `NONE` / `CAUTION`). 거래소 심사 기반 상시·정성적 지정으로 단계·사유·해제 시각이 없다.
- **경보제 (주의 종목)**: `bithumb market warnings` 응답. 시장 데이터 기반 자동·정량적 탐지로 `warning_step` 3단계(`CAUTION`/`WARNING`/`DANGER`), `warning_type` 5가지 사유, `end_date` 자동 해제 시각을 제공한다.
- **유의 종목 ≠ 경보제**: 두 제도는 서로 다른 데이터 소스를 본다. 한쪽에만 잡히는 종목, 양쪽 모두에 잡히는 종목이 모두 존재한다. 두 API를 자동으로 둘 다 호출하지 않는다 — 사용자 의도에 맞게 한쪽만 호출하고, 표현이 모호하면 어느 쪽인지 확인한다.
- **Notices count**: min 1, max 20, default 5
- **Trades pagination**: use `--cursor` from previous response for next page; `--days-ago` filters 1-7 days back
- **`--json`**: add to any command for raw JSON output

## Global Notes

- No API key required for any command in this skill
- Bithumb supports two quote markets: KRW (원화) and BTC.
- Candle data is sorted by time
- `--json` returns raw Bithumb API response
