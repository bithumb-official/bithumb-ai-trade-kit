# Instrument & Info Commands

## market markets — List Tradeable Markets

```bash
bithumb market markets
bithumb market markets --is-details
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--is-details` | No | false | Include `market_warning` field (유의 종목 지정 여부) |

Returns per market: `market` (e.g., `KRW-BTC`), `korean_name`, `english_name`, `market_warning` (if `--is-details`).

`market_warning` 필드는 빗썸의 **유의 종목 지정**(거래소 심사 기반 상시·정성적 지정) 정보다. 값은 두 가지뿐:

| 값 | 의미 |
|---|---|
| `NONE` | 해당 사항 없음 |
| `CAUTION` | 거래 유의 (유의 종목으로 지정됨) |

이 필드는 단계 구분·사유·해제 시각이 없다. 시장 데이터 기반 자동 탐지인 **경보제**(`bithumb market warnings`)와는 별개의 제도이며, 두 API를 자동으로 둘 다 호출하지 않는다.

```bash
# List all markets
bithumb market markets

# With detailed info (market warnings, new listing flags)
bithumb market markets --is-details
```

### Market Code Convention

Bithumb market codes follow `{quote}-{base}` format:

| Quote | Example | Description |
|---|---|---|
| `KRW` | `KRW-BTC` | Korean Won market (원화 마켓) |
| `BTC` | `BTC-ETH` | Bitcoin market (BTC 마켓) |

---

## market warnings — 경보제 (Market Alert)

```bash
bithumb market warnings
```

No parameters required.

Returns: list of markets flagged by **경보제** (`/v1/market/virtual_asset_warning`) — 시장 데이터 기반 자동·정량적 이상거래 탐지 시스템. 빗썸 공식 spec에서는 "주의 종목"으로 부른다.

각 항목 필드:

| 필드 | 값 | 설명 |
|---|---|---|
| `market` | 예: `KRW-BTC` | 마켓 코드 |
| `warning_type` | `PRICE_SUDDEN_FLUCTUATION` / `PRICE_DIFFERENCE_HIGH` / `SPECIFIC_ACCOUNT_HIGH_TRANSACTION` / `TRADING_VOLUME_SUDDEN_FLUCTUATION` / `DEPOSIT_AMOUNT_SUDDEN_FLUCTUATION` | 경보 유형 (가격 급등락 / 글로벌 시세 차이 / 소수계정 거래 집중 / 거래량 급등 / 입금량 급등) |
| `warning_step` | `CAUTION` / `WARNING` / `DANGER` | 경보 단계 (주의 / 경고 / 위험) |
| `end_date` | `yyyy-MM-dd HH:mm:ss` (KST) | 자동 해제 시각 |

```bash
bithumb market warnings
# → list of markets currently under market alert
```

> **Use case**: 사용자가 "경보 / 주의 종목 / 이상거래 탐지"를 명시적으로 요청한 경우에만 호출한다. **유의 종목 지정**(`markets --is-details`의 `market_warning`)은 별개의 제도이며, 자동으로 둘 다 호출하지 않는다.

---

## market notices — Exchange Notices

```bash
bithumb market notices
bithumb market notices --count 10
```

| Param | Required | Default | Description |
|---|---|---|---|
| `--count` | No | 5 | Number of notices (min 1, max 20) |

Returns: list of recent exchange notices/announcements with title, date, and content.

```bash
# Latest 5 notices
bithumb market notices

# Latest 10 notices
bithumb market notices --count 10
```

> **Use case**: Check for maintenance schedules, new listings, delisting notices, fee changes, and system updates.

---

## market fee-inout — Deposit/Withdrawal Fees

```bash
bithumb market fee-inout BTC
bithumb market fee-inout ALL
```

| Param | Required | Description |
|---|---|---|
| `<currency>` | Yes | Currency symbol (e.g., `BTC`, `ETH`) or `ALL` for all currencies |

Returns per currency: deposit fee, withdrawal fee, minimum withdrawal amount, network info.

```bash
# BTC fees
bithumb market fee-inout BTC

# All currencies
bithumb market fee-inout ALL
```

> **Tip**: Use `ALL` to get a comprehensive fee table for all supported currencies. Useful when comparing withdrawal costs across different assets.

---

## Notes

- **Spot only**: Bithumb does not support derivatives (futures, options, swaps). All markets are spot trading pairs.
- **Market discovery workflow**: Run `bithumb market markets` first to get valid market codes, then use `market ticker` / `market orderbook` / `market candles-*` for price data.
- **유의 종목 vs 경보제 라우팅**: `markets --is-details`의 `market_warning`은 **유의 종목 지정**(거래소 심사 기반 상시·정성적), `warnings`는 **경보제**(시장 데이터 기반 자동·정량적 탐지)로 서로 다른 제도다. 두 API를 자동으로 둘 다 호출하지 않는다. 사용자 의도가 모호하면 어느 쪽인지 확인한 뒤 한쪽만 호출한다.
- **Fee check before withdrawal**: Always check `bithumb market fee-inout <currency>` before initiating a withdrawal to confirm the fee and minimum amount.
- **`--json`**: Add to any command for raw JSON output.

## Quickstart

```bash
# List all tradeable markets
bithumb market markets

# Check investment caution designation (유의 종목 지정 — market_warning: NONE/CAUTION)
bithumb market markets --is-details

# Check market alert system (경보제 — 주의 종목; step/type/end_date included)
bithumb market warnings

# Latest notices
bithumb market notices --count 5

# BTC deposit/withdrawal fees
bithumb market fee-inout BTC

# All currencies' fees
bithumb market fee-inout ALL
```
