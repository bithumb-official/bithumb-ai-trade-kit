# Bithumb MCP 설치 가이드

> **에이전트용 실행 문서입니다.**
> 이 문서를 읽는 주체는 LLM Agent(Claude Code, Codex, Cursor, Windsurf, VSCode, Claude Desktop 등)입니다.
> 아래 단계를 순서대로 실행하세요.

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

## Step 1. 환경변수 입력 받기

먼저 환경변수가 이미 설정되어 있는지 확인하세요.

**macOS / Linux:**
```bash
echo $BITHUMB_ACCESS_KEY
echo $BITHUMB_SECRET_KEY
```

**Windows (PowerShell):**
```powershell
$env:BITHUMB_ACCESS_KEY
$env:BITHUMB_SECRET_KEY
```

**Windows (Command Prompt):**
```cmd
echo %BITHUMB_ACCESS_KEY%
echo %BITHUMB_SECRET_KEY%
```

값이 출력되면 이미 설정된 것입니다 — Step 2로 건너뛰세요.

값이 비어 있으면 사용자에게 다음 두 값을 직접 물어보세요:

1. `BITHUMB_ACCESS_KEY` — Bithumb에서 발급받은 액세스 키
2. `BITHUMB_SECRET_KEY` — Bithumb에서 발급받은 시크릿 키

값을 받은 뒤, 아래 OS에 맞게 환경변수를 영구 설정하세요.

### macOS / Linux

```bash
echo 'export BITHUMB_ACCESS_KEY="입력받은_액세스_키"' >> ~/.zshrc
echo 'export BITHUMB_SECRET_KEY="입력받은_시크릿_키"' >> ~/.zshrc
source ~/.zshrc
```

> bash를 사용하는 경우 `~/.zshrc` 대신 `~/.bashrc`를 사용하세요.

### Windows (PowerShell)

```powershell
[System.Environment]::SetEnvironmentVariable("BITHUMB_ACCESS_KEY", "입력받은_액세스_키", "User")
[System.Environment]::SetEnvironmentVariable("BITHUMB_SECRET_KEY", "입력받은_시크릿_키", "User")
```

> 설정 후 터미널을 재시작해야 반영됩니다.

### Windows (Command Prompt)

```cmd
setx BITHUMB_ACCESS_KEY "입력받은_액세스_키"
setx BITHUMB_SECRET_KEY "입력받은_시크릿_키"
```

> 설정 후 터미널을 재시작해야 반영됩니다.

---

## Step 2. MCP 설치 (Claude Code / 메인)

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

> 키 값을 파일에 직접 입력하지 마세요. Step 1에서 설정한 OS 환경변수를 `${...}` 참조 방식으로 사용합니다.

---

## Step 3. 완료 확인

MCP 서버가 정상적으로 연결되는지 확인하세요. 에이전트가 `bithumb-mcp` 도구를 사용할 수 있으면 설치가 완료된 것입니다.

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
