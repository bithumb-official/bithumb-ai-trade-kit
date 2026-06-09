---
name: bithumb-market
description: "Read-only public market data for Bithumb: prices, ticker, order book, candles (minute/day/week/month), recent trades, market list, virtual asset warnings, notices, and fees. No API credentials required. Do NOT use for wallet/blockchain status or deposit-withdrawal availability — use bithumb-account. 빗썸 시세·호가·캔들·체결 내역·마켓 목록·유의 종목·경보제·공지사항·수수료 등 공개 시장 데이터를 조회합니다. 인증 불필요. 빗썸 시세·차트·호가·체결 내역·유의 종목·경보제 관련 요청에 사용하세요. 입출금 현황·블록 갱신시각·블록 동기화·입출금 가능 여부는 bithumb-account를 사용하세요."
---

# Bithumb CEX Market Data CLI

> **Compliance notice**: This skill provides raw market data only. No strategy, recommendation, or optimization logic is embedded. All outputs are objective numerical values; interpretation and trading decisions remain solely with the user.

Public, **read-only** market data for Bithumb: prices, order books, candles, recent trades, market list, investment caution designation (유의 종목 지정), market alert system (경보제 / 주의 종목), notices, and fees. **No API credentials required.**

**Skill routing**: market data → this skill; balance/wallet → `bithumb-account`; orders → `bithumb-trade`; deposits → `bithumb-deposit`; withdrawals → `bithumb-withdraw`; audit/diagnostics → `bithumb-system`.

**Risk-flag intent routing — investment caution designation (유의 종목 지정) vs market alert system (경보제)**

Bithumb runs two independent trading-risk regimes. They use different data sources, and a market may appear in one, both, or neither — so calling the wrong API gives wrong risk info. Route by user intent and call only ONE side.

| User intent | Command | Regime |
|---|---|---|
| Plain market list | `bithumb market markets` (omit isDetails — API default: false) | — |
| Explicit "거래유의 / 유의 종목" request | `bithumb market markets --is-details` (`market_warning` field) | investment caution designation (유의 종목 지정) — exchange-review-based, standing, qualitative |
| "주의 종목 / 경보 / 이상거래 탐지" | `bithumb market warnings` | market alert system (경보제) — market-data-based, automatic, quantitative detection |
| Ambiguous (e.g., "유의 상태", "위험한 코인") | Explain the two regimes differ, confirm which the user means, then call only one | — |

Do NOT auto-call both APIs. Exception: only when the user explicitly asks for "둘 다 / 전부 (both / all)".

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
| 9 | `bithumb market warnings` | market alert system (경보제) — automatic anomaly detection (주의 종목) |
| 10 | `bithumb market notices` | Exchange notice/announcement list |
| 11 | `bithumb market fee-inout <currency>` | Deposit/withdrawal fee info by currency |

## Operation Flow

**Step 1 — Identify data type and load reference**

| User intent | Reference to load |
|---|---|
| Price, candles, order book, recent trades | `{baseDir}/references/price-data-commands.md` |
| Market list, warnings, notices, fee info | `{baseDir}/references/instrument-commands.md` |

**Step 2 — Run immediately.** All commands are read-only; no confirmation and no verification needed.

## Edge Cases

- **Market code format**: `{quote}-{base}`, e.g., `KRW-BTC` (not `BTC-KRW`).
- **Reversed-code auto-correction**: if the user inputs a reversed code like `BTC-KRW`, convert it to `KRW-BTC` and proceed, informing the user (e.g., "Converted BTC-KRW → KRW-BTC").
- **Multiple tickers**: pass comma-separated codes to `ticker`/`orderbook` (e.g., `KRW-BTC,KRW-ETH`).
- **Candle units**: minute candles support `1, 3, 5, 10, 15, 30, 60, 240` only.
- **Candle count**: min 1, max 200, default 1.
- **Candle `--to`**: KST naive datetime `YYYY-MM-DDTHH:MM:SS`; returns candles before this time. **Do NOT include a timezone offset (`+09:00`, `Z`, `+0900`, etc.) — Bithumb rejects offset-bearing values with HTTP 400 Invalid parameter.** `2026-01-01T00:00:00` ✓ / `...+09:00` ✗ / `...Z` ✗
- **Trades `--to`** (differs from candles): time-of-day only, `HHmmss` or `HH:mm:ss` (00:00:00–23:59:59). **Date is not accepted** (`2026-01-01T00:00:00` ✗). For past days use `--days-ago` (1–7). Omitted: without `--days-ago` → current time; with → that day's 00:00:00 base.
- **`--converting-price-unit`**: converts a non-KRW market's (e.g., `BTC-ETH`) close to KRW; Bithumb supports `KRW` only. Result (`converted_trade_price`) appears in `--json` only; KRW markets return `null`.
- **`fee-inout`**: use `ALL` to get all currencies at once.
- **investment caution designation (유의 종목 지정)**: `markets --is-details` → `market_warning` field (`NONE` / `CAUTION`). Exchange-review-based, standing/qualitative; no step, reason, or release time.
- **market alert system (경보제 / 주의 종목)**: `warnings` response. Market-data-based, automatic/quantitative; provides `warning_step` (3 levels: `CAUTION`/`WARNING`/`DANGER`), `warning_type` (5 reasons), and `end_date` auto-release time.
- **유의 종목 ≠ 경보제**: different data sources — a market may be flagged in one, both, or neither. Never auto-call both APIs; call only the side matching user intent, and confirm when ambiguous.
- **Notices count**: min 1, max 20, default 5.
- **Trades pagination**: use `--cursor` from the previous response; `--days-ago` filters 1–7 days back.

## Global Notes

- Install: `npm install -g @bithumb-official/bithumb-cli`. No API key required for any command in this skill; all are read-only.
- Bithumb supports two quote markets: KRW (원화) and BTC. Spot only — no derivatives.
- Candle data is sorted by time.
- `--json` returns the raw Bithumb API response (add to any command).
