[English](./README.md) | 한국어

# 빗썸 Skills

AI 에이전트가 시세 조회, 주문 같은 작업을 정해진 순서대로 처리하도록 돕는 빗썸 Skill 모음입니다. Claude Code, Cursor, Codex, Amp, Cline, Windsurf 등 Skills를 지원하는 에이전트에 등록해 사용합니다.

## 스킬 목록

| 스킬 | 기능                                              | 인증 필요 |
|---|-------------------------------------------------|:-----:|
| [`bithumb-market`](bithumb-market/SKILL.md) | 현재가, 캔들, 호가창, 체결 내역, 공지사항, 수수료 등 공개 시장 데이터 조회   |  불필요  |
| [`bithumb-account`](bithumb-account/SKILL.md) | 잔고, 주문 가능 여부, 지갑 입출금 가능 여부, 블록 동기화, API 키 정보 조회 |  필요   |
| [`bithumb-trade`](bithumb-trade/SKILL.md) | 지정가·시장가·일괄·TWAP 등 주문 접수, 취소, 조회                 |  필요   |
| [`bithumb-deposit`](bithumb-deposit/SKILL.md) | 입금 주소 관리, 입금 내역 조회, 원화 입금 요청                    |  필요   |
| [`bithumb-withdraw`](bithumb-withdraw/SKILL.md) | 출금 요청, 대기 중 코인 출금 취소, 내역 조회, 수수료, 허용 주소 관리      |  필요   |
| [`bithumb-system`](bithumb-system/SKILL.md) | 연결 상태, 인증 정보, 실행 로그, 지원 기능 진단                   |  불필요  |

## 보안

Skills는 API Key로 실제 주문 접수와 출금 실행이 가능합니다. 다음 사항을 유의하세요.

- API Key와 Secret Key를 AI 채팅창에 입력하거나 다른 사람과 공유하지 마세요.
- API Key는 용도에 맞는 권한만 부여하세요. 시세 조회만 사용한다면 조회 권한만, 매매 용도라면 거래 권한만 설정하세요.
- API Key와 Secret Key는 로컬 환경에만 보관하세요. 노출이 의심되면 즉시 폐기하고 새로 발급하세요.
- 공용 PC나 보안이 취약한 환경에서는 사용하지 마세요.
- 주문, 출금 등의 작업은 에이전트가 실행 전 사용자 확인을 거치도록 설정하는 것을 권장합니다.

## 사전 요구사항

- Node.js 18 이상
- 스킬을 지원하는 AI 에이전트 (Claude Code, Cursor, Codex, Amp, Cline, Windsurf 등)
- Bithumb CLI: 스킬이 내부적으로 `bithumb` 명령을 실행합니다:
  ```bash
  npm install -g @bithumb-official/bithumb-cli
  ```
- 빗썸 Open API 키 (`bithumb-market`, `bithumb-system`은 불필요) — 인증 설정은 [CLI 문서](https://apidocs.bithumb.com/docs/cli)를 참고하세요.

## 설치

지원되는 에이전트는 자동으로 감지됩니다. 6개 스킬을 한 번에 모두 설치합니다:

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit --all
```

설치할 스킬과 옵션을 직접 선택하려면:

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit
```

설치 과정에서 다음을 안내합니다:

1. 설치할 스킬 선택
2. 대상 에이전트 확인(자동 감지)
3. 범위 선택: 프로젝트 또는 사용자(전역)
4. 설치 방식 선택: **Symlink**(권장) 또는 복사

설치 후 확인:

```bash
npx skills@latest list
```

## 동작 방식

설치 후 에이전트에게 자연어로 요청하세요:

> "KRW-BTC 현재가 알려줘"
> "내 잔고 확인해줘"
> "BTC 100,000원어치 시장가로 사줘"

시장가 매수 요청의 경우, 에이전트가 `bithumb-trade`를 선택하고 전체 워크플로를 자동으로 실행합니다:

1. `bithumb account order-chance --market KRW-BTC`: 잔고 및 주문 한도 확인
2. `bithumb market ticker KRW-BTC`: 현재가 확인
3. 사용자 확인 요청 (마켓, 매수/매도 방향, 금액)
4. `bithumb trade place --market KRW-BTC --side bid --order-type price --price 100000`: 주문 실행
5. `bithumb trade get --order-id <id>`: 주문 상태 확인

MCP가 개별 도구를 호출하는 방식이라면, Skills는 사전 확인부터 실행, 검증까지 전체 흐름을 하나로 처리합니다.

## 문제 해결

**스킬이 에이전트에서 감지되지 않을 때**

에이전트를 재시작하고 `npx skills@latest list`로 설치 여부를 확인하세요.

**`bithumb` 명령을 찾을 수 없을 때**

CLI가 설치되어 있는지 확인하세요: `bithumb --version`

**인증 오류가 날 때**

`BITHUMB_ACCESS_KEY`, `BITHUMB_SECRET_KEY` 환경 변수가 설정되어 있는지 확인하세요. `bithumb system diagnose`로 인증 상태를 점검할 수 있습니다.

## 관련 링크

- **npm (CLI)**: [@bithumb-official/bithumb-cli](https://www.npmjs.com/package/@bithumb-official/bithumb-cli)
- **npm (MCP)**: [@bithumb-official/bithumb-mcp](https://www.npmjs.com/package/@bithumb-official/bithumb-mcp)
- **사용 가이드**: [AI 트레이드 킷 시작하기](https://apidocs.bithumb.com/docs/skills)

## 라이선스

MIT
