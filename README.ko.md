[English](./README.md) | 한국어

# 빗썸 AI 트레이드 킷

빗썸 API로 시세 조회·계좌 조회·주문·입출금을 처리하는 AI 에이전트용 도구 모음입니다.

MCP 서버, 터미널 CLI, 그리고 AI 에이전트용 Skills를 제공하며, Claude, Cursor, VS Code, Windsurf, Codex 등의 환경에서 사용할 수 있습니다.

---

## 구성 요소

| 구성 요소 | 패키지 | 설명                                                       |
|---|---|----------------------------------------------------------|
| **MCP** | [`@bithumb-official/bithumb-mcp`](https://www.npmjs.com/package/@bithumb-official/bithumb-mcp) | Claude, Cursor, VS Code, Windsurf 등 MCP 호환 클라이언트를 빗썸과 연동 |
| **CLI** | [`@bithumb-official/bithumb-cli`](https://www.npmjs.com/package/@bithumb-official/bithumb-cli) | 터미널에서 시세 조회, 주문, 입출금 작업 수행                               |
| **Skills** | `skills/` | AI 에이전트의 작업 흐름을 구조화하는 가이드                                |


---

## MCP

전역 설치가 필요 없습니다. MCP 클라이언트 설정에 서버를 추가하면 `npx`를 통해 자동으로 실행됩니다.

**Claude Code** (프로젝트 루트의 `.mcp.json`, 또는 전역 설정은 `~/.claude.json`):

```json
{
  "mcpServers": {
    "bithumb": {
      "command": "npx",
      "args": ["-y", "@bithumb-official/bithumb-mcp", "--modules", "all"],
      "env": {
        "BITHUMB_ACCESS_KEY": "${BITHUMB_ACCESS_KEY}",
        "BITHUMB_SECRET_KEY": "${BITHUMB_SECRET_KEY}"
      }
    }
  }
}
```

더 자세한 가이드는 [MCP 문서](https://apidocs.bithumb.com/docs/mcp)를 참고하세요.

---

## CLI

설치:
```bash
npm install -g @bithumb-official/bithumb-cli
```

예제:
```bash
bithumb market ticker KRW-BTC
bithumb account assets
bithumb trade place --market KRW-BTC --side bid --order-type limit --price 50000000 --volume 0.001
bithumb system diagnose
```

더 자세한 가이드는 [CLI 문서](https://apidocs.bithumb.com/docs/cli)를 참고하세요.

---

## Skills

지원되는 에이전트가 자동으로 감지되며, 아래 명령어로 모든 Skills를 설치할 수 있습니다.

```bash
npx skills@latest add bithumb-official/bithumb-ai-trade-kit --all
```

더 자세한 가이드는 [Skills 문서](https://apidocs.bithumb.com/docs/skills)를 참고하세요.

---

## 인증

시세 및 시장 데이터 조회는 API Key 없이 사용할 수 있습니다. 계좌 조회, 주문, 입출금 기능을 사용하려면 빗썸 API Key가 필요합니다.

[빗썸 API 관리 페이지](https://www.bithumb.com/react/api-support/management-api)에서 필요한 권한으로 API Key를 설정한 후 환경 변수를 설정하세요:

```bash
export BITHUMB_ACCESS_KEY="your_access_key"
export BITHUMB_SECRET_KEY="your_secret_key"
```

CLI는 `~/.bithumb/config.toml`에서 여러 프로필을 관리할 수 있습니다:

```bash
bithumb config init       # 설정 파일 생성(interactive)
bithumb config use trading
```

---

## 사용 가이드

| 항목     | 링크                                                                        |
|--------|---------------------------------------------------------------------------|
| 통합     | [Bithumb AI Trade Kit 시작하기](https://apidocs.bithumb.com/docs/ai-트레이드-킷-시작하기) |
| MCP    | [MCP](https://apidocs.bithumb.com/docs/mcp)                             |
| CLI    | [CLI](https://apidocs.bithumb.com/docs/cli)                             |
| Skills | [Skills](https://apidocs.bithumb.com/docs/skills)                       |

---

## 참고

[OKX Agent Trade Kit](https://www.okx.com/agent-tradekit)을 참고하여 개발했습니다.

---

## 라이선스

MIT
