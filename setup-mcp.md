# Bithumb MCP 설치 가이드

> **에이전트용 실행 문서입니다.**
> 이 문서를 읽는 주체는 LLM Agent(Claude Code, Codex, Cursor, Windsurf, VSCode, Claude Desktop 등)입니다.
> 아래 단계를 순서대로 실행하세요.

> **[필수 규칙] Agent는 절대로 API 키 값을 묻지 않는다.**
> - `BITHUMB_ACCESS_KEY`, `BITHUMB_SECRET_KEY`의 실제 값을 채팅으로 요구하거나, 입력받거나, 대신 저장하지 않는다.
> - 환경변수 설정은 Agent의 작업이 아니다. 사용자가 직접 한다. Agent는 명령어를 "보여주기만" 하고, 사용자가 직접 실행하도록 안내만 한다.
> - 설정 파일과 명령어의 `${...}`, `your_access_key`, `your_secret_key`는 **그대로 둔다**. 실제 값으로 치환하려고 시도하지 않는다.
> - 키가 없거나 인증이 필요한 단계에 도달하면, 키를 묻지 말고 "사용자가 직접 환경변수를 설정한 뒤 알려달라"고 안내하고 멈춘다.

---

## Step 0. Node.js 확인

```bash
node -v
```

Node.js 18 이상이 출력되면 Step 1로 건너뛰세요.

출력이 없거나 18 미만이면 아래 방법으로 설치하세요.

### 권장: nvm (macOS / Linux)

nvm이 있는지 먼저 확인하세요.

```bash
nvm --version
```

nvm이 있으면 바로 설치합니다.

```bash
nvm install --lts
nvm use --lts
```

nvm이 없으면 먼저 nvm을 설치하세요. 설치 후 터미널을 재시작한 뒤 위 명령어를 실행하세요.

> nvm 설치: https://github.com/nvm-sh/nvm#installing-and-updating

nvm으로 설치하면 `npm install -g` 시 권한 오류(EACCES)가 발생하지 않습니다.

### 대안: 패키지 매니저

**macOS (Homebrew):**
```bash
brew install node
```

**Windows:**
```powershell
winget install OpenJS.NodeJS.LTS
```

> Homebrew나 winget으로 설치한 경우 `npm install -g` 실행 시 권한 오류가 발생할 수 있습니다. 오류가 나면 nvm으로 재설치하세요.

---

## Step 1. MCP 설치 (Claude Code / 메인)

사용자가 작업 중인 **프로젝트 루트**에 `.mcp.json` 파일을 생성하고 아래 내용을 작성하세요.

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

> 키 값을 파일에 직접 입력하지 마세요. OS 환경변수를 `${...}` 참조 방식으로 사용합니다 (아래 부록 참고).

---

## Step 2. 완료 확인

> **[중요] `.mcp.json`은 세션 시작 시점에만 로드됩니다.**
> 방금 `.mcp.json`을 생성·수정했다면, **현재 세션에서는 MCP 서버가 아직 연결되지 않은 상태**입니다.
> Agent는 지금 세션에서 `bithumb_*` 도구를 호출하려 시도하지 말고, 아래를 안내하고 멈춥니다.
> - 사용자가 클라이언트(Claude Code / Cursor / Claude Desktop 등)를 **재시작하거나 새 세션을 시작**해야 MCP 서버가 로드됩니다.
> - 키 인증이 필요하다면, 먼저 사용자가 환경변수를 설정(아래 부록)한 뒤 재시작해야 합니다.

재시작 후 새 세션에서, 에이전트가 `bithumb_market_ticker` 도구를 호출해 BTC 시세를 조회해보세요.

> "KRW-BTC 시세 알려줘"

인증 없이 실행되는 공개 데이터입니다. 결과가 반환되면 MCP 서버가 정상 연결된 것입니다.

---

## 부록. 다른 툴에서의 MCP 설치

### Cursor

`~/.cursor/mcp.json` 파일에 아래 내용을 추가하세요.

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

### Windsurf

`~/.codeium/windsurf/mcp_config.json` 파일에 아래 내용을 추가하세요.

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

### VSCode

프로젝트 루트의 `.mcp.json` 파일에 아래 내용을 작성하세요.

```json
{
  "mcpServers": {
    "bithumb": {
      "type": "stdio",
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

### Claude Desktop

아래 경로의 설정 파일에 `mcpServers` 항목을 추가하세요.

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

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

> 설정 후 Claude Desktop을 재시작해야 반영됩니다.

### Codex

`~/.codex/config.toml` 파일에 아래 내용을 추가하세요.

```toml
[mcp_servers.bithumb]
command = "npx"
args = ["-y", "@bithumb-official/bithumb-mcp", "--modules", "all"]
env_vars = ["BITHUMB_ACCESS_KEY", "BITHUMB_SECRET_KEY"]
```

---

## 부록. 환경변수 설정

`BITHUMB_ACCESS_KEY`와 `BITHUMB_SECRET_KEY`는 사용자가 직접 관리합니다.

> **[Agent 금지] Agent는 키 값을 묻지 않으며, 아래 명령어를 대신 실행하지도 않는다.**
> 아래 명령어는 **사용자가 직접** 자신의 터미널에 붙여넣어 실행하는 것이다.
> Agent는 명령어 블록을 그대로 보여주고, "직접 실행한 뒤 터미널을 재시작하라"고 안내만 한다.
> `your_access_key` / `your_secret_key`는 사용자가 자기 값으로 바꿔 넣는다 — Agent는 이 자리를 채우지 않는다.

API 키는 [Bithumb API 관리 페이지](https://www.bithumb.com/react/api-support/management-api)에서 발급받을 수 있습니다.

환경변수가 설정되지 않은 경우, 사용자에게 아래 명령어를 **제시**하세요(에이전트가 실행하지 말 것).

### macOS / Linux

```bash
echo 'export BITHUMB_ACCESS_KEY="your_access_key"' >> ~/.zshrc
echo 'export BITHUMB_SECRET_KEY="your_secret_key"' >> ~/.zshrc
source ~/.zshrc
```

> bash를 사용하는 경우 `~/.zshrc` 대신 `~/.bashrc`를 사용하세요.

### Windows (PowerShell)

```powershell
[System.Environment]::SetEnvironmentVariable("BITHUMB_ACCESS_KEY", "your_access_key", "User")
[System.Environment]::SetEnvironmentVariable("BITHUMB_SECRET_KEY", "your_secret_key", "User")
```

> 설정 후 터미널을 재시작해야 반영됩니다.

### Windows (Command Prompt)

```cmd
setx BITHUMB_ACCESS_KEY "your_access_key"
setx BITHUMB_SECRET_KEY "your_secret_key"
```

> 설정 후 터미널을 재시작해야 반영됩니다.