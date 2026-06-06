---
name: bithumb-withdraw
description: "Manage Bithumb withdrawals: crypto withdrawal, KRW withdrawal, withdrawal cancellation, withdrawal history, withdrawal fee lookup, and allowed-address management. Requires API credentials. Do NOT use for withdrawal chance / available withdrawal amount — use bithumb-account. 빗썸 코인·원화 출금 실행·취소, 출금 내역 조회, 출금 수수료 확인, 허용 주소 관리를 처리합니다. 출금·코인 전송·원화 출금·출금 취소·출금 내역·허용 주소 관련 요청에 사용하세요. 출금 가능 금액은 bithumb-account를 사용하세요."
---

# Bithumb Withdrawal CLI

## Tool Routing

This skill's write operations are **CLI-first**: when both the Bithumb CLI and the MCP tools are available, use the documented `bithumb ...` CLI command as the default path. Fall back to the MCP tool only when the CLI is unavailable or fails, or when the user explicitly asks for MCP. Read-only queries have no such preference — either path is fine.

Why CLI-first for writes: the CLI carries safety mechanisms the MCP path lacks — per-profile read-only gating, account isolation, and terminal reproducibility (see the pre-flight read-only gate). These matter only for writes.

Crypto and KRW (원화) withdrawal operations on Bithumb exchange. Includes withdrawal execution, cancellation, history, fee checks, and allowed address management. **Requires API credentials.**

> **CRITICAL**: Crypto withdrawals (`withdraw coin`) are **irreversible**. Always double-check the address, network, and amount before executing.

## Prerequisites

1. **CLI**: `npm install -g @bithumb-official/bithumb-cli`
2. Configure credentials:
   ```bash
   export BITHUMB_ACCESS_KEY=your_key
   export BITHUMB_SECRET_KEY=your_secret
   ```
3. Withdrawal addresses must be pre-registered on Bithumb (for `withdraw coin`)

## Skill Routing

- For market data (prices, charts, depth) → use `bithumb-market`
- For account assets, wallet status → use `bithumb-account`
- For placing/cancelling orders → use `bithumb-trade`
- For deposits → use `bithumb-deposit`
- For withdrawals → use `bithumb-withdraw` (this skill)
- For audit logs, diagnostics → use `bithumb-system`

## Command Index

| # | Command | Type | Description |
|---|---|---|---|
| 1 | `bithumb withdraw chance` | READ | Withdrawal availability: balance, fees, limits |
| 2 | `bithumb withdraw addresses` | READ | List allowed withdrawal addresses |
| 3 | `bithumb withdraw get` | READ | Get specific withdrawal details |
| 4 | `bithumb withdraw list` | READ | Coin withdrawal history |
| 5 | `bithumb withdraw list-krw` | READ | KRW withdrawal history |
| 6 | `bithumb withdraw coin` | WRITE | **Withdraw crypto (IRREVERSIBLE)** |
| 7 | `bithumb withdraw krw` | WRITE | **Withdraw KRW (2FA required)** |
| 8 | `bithumb withdraw cancel` | WRITE | Cancel pending crypto withdrawal |

---

## Operation Flow

> ### 🛑 Mandatory Pre-flight Checklist — `withdraw coin` / `withdraw krw` / `withdraw cancel`
>
> **Before** any write operation in this skill, run **all** of the following in order. Do **not** skip steps even if the user says "just send it". Stop and ask for explicit user confirmation between step 5 and step 7.
>
> Step 0 (the read-only gate) applies to **every** write here — including `withdraw cancel`. The heavier balance/address lookups (steps 1–8) are only for `withdraw coin` / `withdraw krw`; for `withdraw cancel`, run Step 0, then confirm the withdrawal ID and execute.
>
> | # | Command | Purpose |
> |---|---|---|
> | 0 | `bithumb config show --json` | **Read-only gate (run first, all writes incl. cancel).** Read the effective profile's `read_only`. If `true`, **stop here** — do not run steps 1–8. List write-capable profiles and ask the user to re-run with `--profile <name>` (**never bypass read-only** — do not auto-select a profile, switch profiles, or disable `read_only` yourself; see gate). Once-per-session: skip if already checked. See [read-only-gate.md](references/read-only-gate.md). |
> | 1 | `bithumb account assets` | Confirm sufficient balance |
> | 2 | `bithumb account wallet-status` | Confirm withdrawal is enabled (blockchain not under maintenance) |
> | 3 | `bithumb withdraw chance --currency <c> --net-type <n>` | Confirm fee, min amount, daily limit, supported `net_type` |
> | 4 | `bithumb withdraw addresses` | Confirm destination is in the allowed list (Bithumb requirement) |
> | 5 | `bithumb market fee-inout <currency>` | Cross-check withdrawal fee from market side |
> | 5b | **🛂 출금 타입 확인 후 수취인 정보 수집** | 빗썸 출금은 5종(내부 / CODE ID Connect / CODE 개인 / CODE 법인 / WHITELIST)이며 수취인 정보 필수 여부가 다르다. **CODE 개인/법인 출금일 때만** `--receiver-type`을 정하고 type별 필수 필드(아래 [Travel Rule 매트릭스](#-travel-rule--receiver-info-is-mandatory-for-vasp-code-withdrawals))를 모두 채운다. 내부 출금·ID Connect·WHITELIST·개인 지갑 직접 출금이면 `--receiver-type` 없이 생략 |
> | 6 | **🤚 Stop. Ask user to confirm**: currency, net_type, address (full string), amount, secondary_address (memo/tag) if applicable, **and (CODE 개인/법인 출금) receiver_type + all receiver fields** |
> | 7 | `bithumb withdraw coin --currency <c> --net-type <n> --amount <a> --address <addr> [--secondary-address <m>] [--exchange-name <e> --receiver-type <t> ...receiver fields]` | Execute |
> | 8 | `bithumb withdraw list --currency <c> --limit 1` | Verify status |
>
> Skipping any step (especially 4 / 5 / 6) is the most common cause of failed/lost withdrawals in past QA. The same checklist is mirrored in [bithumb-account Cross-Skill Workflows](../bithumb-account/SKILL.md#cross-skill-workflows).

### Step 1 — Identify withdrawal action

| User intent | Command |
|---|---|
| **First time / multi-network coin (USDT, USDC, XRP, etc.)** — discover net_types | `bithumb withdraw chance --currency <c>` (lists supported `net_type`) **before** calling `withdraw coin` |
| Check if withdrawal is possible & fees | `bithumb withdraw chance --currency BTC --net-type BTC` |
| View allowed withdrawal addresses | `bithumb withdraw addresses` |
| Look up one withdrawal by identifier (id/txid) — **식별자 1개면 단건** | `bithumb withdraw get --currency BTC` |
| Search withdrawal history / filter by state (식별자 없거나 여러 개) | `bithumb withdraw list` |
| View KRW withdrawal history | `bithumb withdraw list-krw` |
| Withdraw crypto | `bithumb withdraw coin --currency BTC --net-type BTC --amount 0.01 --address <addr>` |
| Withdraw KRW | `bithumb withdraw krw --amount 100000 --two-factor-type kakao` |
| Cancel pending withdrawal | `bithumb withdraw cancel --withdrawal-id <id>` |

### Step 2 — Read commands run immediately; write commands need careful confirmation

**Read commands** (`withdraw chance`, `withdraw addresses`, `withdraw get`, `withdraw list`, `withdraw list-krw`): run immediately.

**Write commands** — **ALL require explicit user confirmation**:

- `withdraw coin`: **IRREVERSIBLE** — verify currency, network, address, amount, and receiver info. Run `withdraw chance` first.
- `withdraw krw`: requires Kakao 2FA — confirm amount.
- `withdraw cancel`: confirm withdrawal ID.

### Step 3 — Verify after writes

- After `withdraw coin`: run `bithumb withdraw list --currency <c> --limit 1` to check status
- After `withdraw krw`: run `bithumb withdraw list-krw` to check status
- After `withdraw cancel`: run `bithumb withdraw get --currency <c> --withdrawal-id <id>` to confirm cancellation

---

## CLI Reference

### withdraw chance — Withdrawal Availability

```bash
bithumb withdraw chance --currency <symbol> --net-type <network> [--json]
```

| Param | Required | Description |
|---|---|---|
| `--currency` | Yes | Currency symbol (e.g., `BTC`) |
| `--net-type` | Yes | Network type (e.g., `BTC`, `ETH`, `TRX`) |

Returns: available balance, withdrawal fee, minimum amount, daily/per-transaction limits, and member level info.

> **Always call this first** before any withdrawal to verify limits and fees.

---

### withdraw addresses — Allowed Addresses

```bash
bithumb withdraw addresses [--json]
```

Returns: list of pre-registered withdrawal addresses.

> **Bithumb requirement**: Crypto can only be withdrawn to pre-registered addresses. If the desired address is not listed, the user must register it on the Bithumb website/app first.

---

### withdraw get — Specific Withdrawal

```bash
bithumb withdraw get --currency <symbol> [--withdrawal-id <id>] [--txid <txid>] [--json]
```

| Param | Required | Description |
|---|---|---|
| `--currency` | Yes | Currency symbol |
| `--withdrawal-id` | No | Withdrawal ID |
| `--txid` | No | Transaction ID |

> Only `--currency` is required. Optionally provide `--withdrawal-id` and/or `--txid` to narrow the lookup (the API combines them as an AND filter). With neither, the API returns the most recent withdrawal.

---

### withdraw list — Coin Withdrawal History

```bash
bithumb withdraw list [--currency <symbol>] [--state <PROCESSING|DONE|CANCELLED>] \
  [--withdrawal-ids <id1,id2>] [--txids <txid1,txid2>] \
  [--limit <n>] [--page <n>] [--order-by <asc|desc>] [--json]
```

> **🔁 식별자 폴백**: ID가 uuid(`--withdrawal-ids`)인지 txid인지 불명확하면 `--withdrawal-ids`로 먼저 조회 → **빈 배열이면 같은 값을 `--txids`로 재시도**(또는 사용자에게 "txids로 조회할까요?" 질문)한 뒤에 "없음"으로 단정한다. 두 필터 모두 정상 동작하므로, 한쪽이 비었다고 곧장 "데이터 없음"으로 끝내지 말 것.

---

### withdraw list-krw — KRW Withdrawal History

```bash
bithumb withdraw list-krw [--state <PROCESSING|DONE|CANCELLED>] \
  [--withdrawal-ids <id1,id2>] [--txids <txid1,txid2>] \
  [--limit <n>] [--page <n>] [--order-by <asc|desc>] [--json]
```

> **🔁 식별자 폴백 (KRW는 uuid·txid 모두 숫자)**: 원화 출금은 `--withdrawal-ids`(uuid)와 `--txids`가 **둘 다 숫자**라 값만으로 구분이 안 된다. 사용자가 맨 숫자 ID를 주면 `--withdrawal-ids`로 먼저 조회 → **빈 배열이면 같은 값을 `--txids`로 재시도**(또는 "txids로 조회할까요?" 질문)한 뒤에 "없음"으로 단정한다. 예: `--withdrawal-ids 43007582` → `[]` → `--txids 55955615` → 레코드(DONE).

---

### withdraw coin — Withdraw Crypto

> **IRREVERSIBLE**: Once confirmed on the blockchain, this cannot be undone. Wrong address or network = permanent loss of funds.

```bash
bithumb withdraw coin --currency <symbol> --net-type <network> \
  --amount <n> --address <addr> \
  [--secondary-address <addr>] [--exchange-name <name>] \
  [--receiver-type <personal|corporation>] \
  [--receiver-ko-name <name>] [--receiver-en-name <name>] \
  [--receiver-corp-ko-name <name>] [--receiver-corp-en-name <name>] \
  [--json]
```

| Param | Required | Description |
|---|---|---|
| `--currency` | Yes | Currency symbol (e.g., `BTC`) |
| `--net-type` | Yes | Network (e.g., `BTC`, `ETH`, `TRX`) |
| `--amount` | Yes | Withdrawal amount |
| `--address` | Yes | Registered withdrawal address |
| `--secondary-address` | No | Memo/tag (required for XRP, EOS, etc.) |
| `--exchange-name` | Cond. | Receiving exchange name (English). Required for CODE / ID Connect / WHITELIST withdrawals. **By itself it does NOT trigger receiver-info enforcement** — the trigger is `--receiver-type`. See matrix below |
| `--receiver-type` | Cond. | `personal` or `corporation`. **Only for CODE 개인/법인 withdrawals.** Its presence triggers receiver-info validation; omit it for 내부 출금 / ID Connect / WHITELIST |
| `--receiver-ko-name` | Cond. | Receiver Korean name. `personal`: 개인 국문명; `corporation`: 대표자 국문명. Required for both when `--receiver-type` is set |
| `--receiver-en-name` | Cond. | Receiver English name. `personal`: 개인 영문명; `corporation`: 대표자 영문명. Required for both when `--receiver-type` is set |
| `--receiver-corp-ko-name` | Cond. | Corporation Korean name (법인명 국문). Required for `corporation` |
| `--receiver-corp-en-name` | Cond. | Corporation English name (법인명 영문). Required for `corporation` |

#### 🛂 Travel Rule — receiver info is MANDATORY for VASP (CODE) withdrawals

특정금융정보법(트래블룰, 2022-03-25 시행)상 **트래블룰 솔루션(CODE) 연동 거래소로 출금**할 때는 수취인 정보가 필수다. 빗썸 출금은 5종으로 나뉘고, 수취인 정보 필수 여부가 타입마다 다르다:

| 출금 타입 | `--exchange-name` | `--receiver-type` 및 수취인 정보 |
|---|---|---|
| 내부 출금 (빗썸 회원 간) | ❌ | ❌ (트래블룰 비대상) |
| CODE ID Connect (개인) | ✅ | ❌ |
| CODE (개인) | ✅ | ✅ `personal` |
| CODE (법인) | ✅ | ✅ `corporation` |
| WHITELIST (사전 등록 주소) | ✅ | ❌ |

이 CLI는 **`--receiver-type`이 주어지면** 빗썸 API 호출 전에 코어에서 type별 필수 필드를 사전 검증한다 — 누락 시 빗썸에 보내기 전 `ValidationError`로 차단된다. `--receiver-type`이 없으면 검증을 건너뛴다(내부 출금·ID Connect·WHITELIST 모두 통과). `--exchange-name`만으로는 출금 타입을 결정론적으로 구분할 수 없으므로(ID Connect·WHITELIST도 `--exchange-name`은 필요하나 수취인 정보는 불필요) 트리거로 쓰지 않는다.

CODE 개인/법인 출금이면 **먼저 `--receiver-type`을 사용자에게 물어** 어떤 필드를 수집할지 결정한 뒤, 아래 매트릭스에 따라 필수 필드를 모두 채운다:

| `--receiver-type` | 추가로 반드시 채워야 할 필드 |
|---|---|
| `personal` | `--receiver-ko-name` (개인 국문명) + `--receiver-en-name` (개인 영문명) |
| `corporation` | `--receiver-corp-ko-name` + `--receiver-corp-en-name` (법인명 국문/영문) **및** `--receiver-ko-name` + `--receiver-en-name` (대표자명 국문/영문) |

> **주의**: 에이전트가 `personal` 출금에서 `--receiver-ko-name`만 채우고 `--receiver-en-name`(영문 성명)을 누락하면 코어가 `ValidationError`로 차단한다. `--receiver-type`을 지정한 출금에서는 위 매트릭스의 필드를 **하나도 빠짐없이** 수집하라. 어떤 필드가 빠졌는지는 코어 `ValidationError`의 suggestion에 명시된다. 반대로 내부 출금·ID Connect·WHITELIST에는 `--receiver-type`을 붙이지 말 것 — 붙이면 불필요한 수취인 정보 검증이 발동한다.

**Mandatory pre-checks before executing**:
1. `bithumb withdraw chance --currency <c> --net-type <n>` — verify balance, fees, limits
2. `bithumb withdraw addresses` — verify address is registered
3. `bithumb account wallet-status` — verify withdrawal is enabled
4. Confirm ALL details with user: currency, network, address, amount
5. Wait for explicit user approval

---

### withdraw krw — Withdraw KRW

> **WARNING**: Requires Kakao 2FA verification. Funds are sent to the registered bank account.

```bash
bithumb withdraw krw --amount <krw_amount> --two-factor-type kakao [--json]
```

| Param | Required | Description |
|---|---|---|
| `--amount` | Yes | Withdrawal amount in KRW |
| `--two-factor-type` | Yes | 2FA method (e.g., `kakao`) |

---

### withdraw cancel — Cancel Pending Withdrawal

```bash
bithumb withdraw cancel --withdrawal-id <id> [--json]
```

Only `PROCESSING` withdrawals can be cancelled. Once broadcasted to the blockchain, cancellation is impossible.

---

## Withdrawal Workflow

### Crypto withdrawal (full safety flow)

```bash
# 1. Check fees and limits
bithumb withdraw chance --currency BTC --net-type BTC

# 2. Check allowed addresses
bithumb withdraw addresses

# 3. Check wallet status (from bithumb-account)
bithumb account wallet-status

# 4. Check balance (from bithumb-account)
bithumb account assets

# 5. Confirm with user: currency=BTC, network=BTC, address=bc1q..., amount=0.01
#    → Wait for explicit "yes"

# 6. Execute withdrawal
bithumb withdraw coin --currency BTC --net-type BTC --amount 0.01 --address "bc1q..."

# 7. Verify
bithumb withdraw list --currency BTC --limit 1
```

---

## Edge Cases

- **Address not registered**: `withdraw coin` will fail if the address is not in `withdraw addresses`. User must register it on Bithumb website/app first.

> **🌐 Multi-network coins (USDT, USDC, XRP, etc.)**: The same `currency` may have multiple `net_type` values. **Before any withdrawal call**, run:
> ```bash
> bithumb withdraw chance --currency <c>
> ```
> to list supported `net_type` values. The `net_type` of the destination wallet must match. Sending USDT-ERC20 to a TRX address (or vice versa) results in **permanent loss of funds**. Never guess — always discover first.

- **Network mismatch**: Sending to wrong network = permanent loss. Always verify `--net-type` matches the receiving address format.
- **Memo/tag required**: XRP, EOS, ATOM, and similar coins require `--secondary-address`. Missing it = lost funds.
- **Minimum withdrawal**: Each currency has a minimum. Check `withdraw chance`.
- **Daily limits**: Withdrawal limits depend on KYC level. Check `withdraw chance`.
- **Processing time**: Crypto withdrawals may take minutes to hours depending on blockchain congestion.
- **Cancel window**: Only possible while status is `PROCESSING` (before blockchain broadcast).
- **KRW withdrawal**: Goes to the bank account registered on Bithumb. Cannot specify a different account.
- **식별자 모호성 (uuid vs txid)**: 원화 입·출금은 `withdrawal_id`(uuid)와 `txid`가 모두 숫자라 값만으로 구분되지 않는다. `list-krw`에서 한쪽 필터(`--withdrawal-ids`)가 빈 배열이면 같은 값을 반대쪽(`--txids`)으로 재조회하거나 사용자에게 확인한 뒤 "없음"으로 단정할 것. (코인 `list`도 uuid/txid 입력이 불명확하면 동일하게 폴백.)
- **`--json`**: add to any command for raw JSON output

## Global Notes

- All commands require valid API credentials
- Bithumb has no demo mode — all withdrawal operations are real and involve real funds
- Travel Rule compliance (특정금융정보법): **트래블룰 솔루션(CODE) 연동 거래소로 출금**할 때 수취인 정보가 필수다. `--receiver-type`이 주어지면 코어가 빗썸 호출 전 type별 필수 필드(개인: 국문+영문 성명, 법인: 법인명 국문/영문 + 대표자명 국문/영문)를 검증한다. 내부 출금·CODE ID Connect·WHITELIST는 `--receiver-type` 없이 통과한다. 자세한 5종 타입 매트릭스는 [withdraw coin](#-travel-rule--receiver-info-is-mandatory-for-vasp-code-withdrawals) 참고
- Always run the full safety flow (check → verify → confirm → execute → verify) for withdrawals
- `--json` returns raw Bithumb API response
