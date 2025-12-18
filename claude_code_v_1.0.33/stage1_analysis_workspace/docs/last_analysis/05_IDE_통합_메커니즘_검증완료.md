# Claude Code IDE 연결 및 상호작용 메커니즘 심층 분석 보고서

## 요약

본 보고서는 Claude Code의 난독화된 소스 코드를 심층 분석하여 IDE와의 연결 구축, 정보 획득, 실시간 동기화 및 양방향 통신 메커니즘을 체계적으로 밝혀냅니다. Claude Code는 MCP(Model Context Protocol) 프로토콜을 통해 IDE와 깊이 통합되어 Language Server Protocol(LSP) 진단 정보 획득, 코드 실행, 파일 상태 모니터링 등의 기능을 구현합니다.

## 1. IDE 연결 구축 메커니즘

### 1.1 연결 프로토콜 아키텍처

Claude Code는 두 가지 주요 IDE 연결 프로토콜을 지원합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                Claude Code IDE 통합 아키텍처                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ IDE 탐지기   │  │ MCP 클라이언트│  │ 진단정보 관리자      │  │
│  │  (TF1)     │  │  (ue)       │  │      (PK)           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ 연결 관리자  │  │ Tool 필터    │  │ 상태 동기화기        │  │
│  │   (DV)      │  │  (l65)      │  │      (we0)          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                  MCP 전송 계층                               │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ SSE-IDE     │  │ WS-IDE      │                          │
│  │ (sse-ide)   │  │ (ws-ide)    │                          │
│  └─────────────┘  └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    IDE 확장                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ VS Code     │  │ Cursor      │  │ Windsurf            │  │
│  │ Extension   │  │ Extension   │  │ Extension           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 연결 프로토콜 유형

#### 1.2.1 SSE-IDE 연결

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 23402-23405

```javascript
// SSE-IDE 연결 설정
if4 = n.object({
  type: n.literal("sse-ide"),
  url: n.string().url("Must be a valid URL"),
  ideName: n.string()
})
```

**기능 설명**:
- Server-Sent Events (SSE)를 사용한 단방향 통신
- 진단 정보를 실시간으로 푸시해야 하는 시나리오에 적합
- 연결 구축 후 `ide_connected` 알림 자동 전송

#### 1.2.2 WS-IDE 연결

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 23408-23412

```javascript
// WS-IDE 연결 설정
nf4 = n.object({
  type: n.literal("ws-ide"),
  url: n.string().url("Must be a valid URL"),
  ideName: n.string(),
  authToken: n.string().optional()
})
```

**기능 설명**:
- WebSocket을 사용한 양방향 통신
- 인증 토큰 지원
- 양방향 상호작용이 필요한 시나리오에 더 적합

### 1.3 연결 구축 절차

#### 1.3.1 IDE 탐지 메커니즘

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 33585-33588

```javascript
function TF1(A) {
  let Q = A.find((I) => I.type === "connected" && I.name === "ide")?.config;
  return Q?.type === "sse-ide" || Q?.type === "ws-ide" ? Q.ideName : null
}
```

**기능 설명**:
- 현재 연결된 IDE 유형 탐지
- IDE 이름 반환 (예: "vscode", "cursor", "windsurf")
- 후속 특정 IDE 기능 적응에 사용

#### 1.3.2 연결 핸드셰이크 및 인증

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35512-35520

```javascript
// SSE-IDE 연결 구축
else if (B.type === "sse-ide") Q = new FF1(new URL(B.url));
// WS-IDE 연결 구축
else if (B.type === "ws-ide") {
  let X = new tB1.default(B.url, ["mcp"], B.authToken ? {
    headers: {
      "X-Claude-Code-Ide-Authorization": B.authToken
    }
  } : void 0);
  Q = new Do1(X)
}
```

**기능 설명**:
- SSE 연결은 HTTP 연결 직접 구축
- WebSocket 연결은 MCP 서브프로토콜 지원
- `X-Claude-Code-Ide-Authorization` 헤더를 사용한 인증

#### 1.3.3 연결 상태 관리

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35577-35591

```javascript
// 연결 실패 처리
} else if (B.type === "sse-ide" || B.type === "ws-ide") E1("tengu_mcp_ide_server_connection_failed", {});

// 연결 성공 처리
if (B.type === "sse-ide" || B.type === "ws-ide") {
  E1("tengu_mcp_ide_server_connection_succeeded", {
    serverVersion: Y
  });
  try {
    we0(I)  // IDE 연결 알림 전송
  } catch (X) {
    m7(A, `Failed to send ide_connected notification: ${X}`)
  }
}
```

**기능 설명**:
- 연결 성공/실패 원격 측정 이벤트 전송
- 연결 성공 후 `ide_connected` 알림 전송
- 오류 처리 및 로그 기록

### 1.4 연결 실패 처리 및 재연결 메커니즘

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 48114-48136

```javascript
// 연결 종료 처리
F.client.transport.onclose = () => {
  if (N) N();
  if (F.config.type === "sse" || F.config.type === "sse-ide") {
    p2(F.name, "SSE transport closed, attempting to reconnect");
    // pending 상태로 재표시
    G((T) => T.map((L) => L.name !== F.name ? L : {
      name: L.name,
      type: "pending",
      config: L.config
    }));
    // 기존 연결 종료
    let R = F.client.transport;
    if (R && typeof R.close === "function") R.close().catch((T) => {
      p2(F.name, `Error closing old transport: ${T}`)
    });
    // 재연결 지연
    setTimeout(() => {
      // 재연결 로직
    }, 1000);
  }
};
```

**기능 설명**:
- 연결 끊김 자동 감지
- SSE 및 SSE-IDE 연결은 자동 재연결 지원
- 재연결 전 기존 연결 리소스 정리
- 지연 재연결을 사용하여 빈번한 연결 방지

## 2. IDE 정보 획득 시스템

### 2.1 진단 정보 관리자 (PK 클래스)

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 49-59

```javascript
class PK {
  static instance;
  baseline = new Map;           // 기준 진단 정보
  initialized = !1;            // 초기화 상태
  mcpClient;                   // MCP 클라이언트
  lastProcessedTimestamps = new Map;  // 마지막 처리 타임스탬프
  lastDiagnosticsByUri = new Map;     // 마지막 진단 정보
  rightFileDiagnosticsState = new Map; // 오른쪽 파일 진단 상태

  static getInstance() {
    if (!PK.instance) PK.instance = new PK;
    return PK.instance
  }
}
```

**기능 설명**:
- 싱글톤 패턴으로 진단 정보 관리
- 파일 진단 기준선 상태 유지
- 진단 정보 변경 추적
- 좌우 분할 화면의 진단 상태 관리 지원

### 2.2 LSP 프로토콜 통합

#### 2.2.1 getDiagnostics Tool 구현

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 109-119

```javascript
async beforeFileEdited(A) {
  if (!this.initialized || !this.mcpClient || this.mcpClient.type !== "connected") return;
  let B = Date.now();
  try {
    let Q = await gw("getDiagnostics", {
        uri: `file://${A}`
      }, this.mcpClient, !1),
      I = this.parseDiagnosticResult(Q)[0];
    if (I) {
      if (A !== this.normalizeFileUri(I.uri)) {
        b1(new Error(`Diagnostics file path mismatch: expected ${A}, got ${I.uri})`));
        return
      }
      this.baseline.set(A, I.diagnostics), this.lastProcessedTimestamps.set(A, B)
    } else this.baseline.set(A, []), this.lastProcessedTimestamps.set(A, B)
  } catch (Q) {}
}
```

**기능 설명**:
- 파일 편집 전 진단 정보 획득
- 후속 비교를 위한 진단 기준선 설정
- 파일 경로 불일치 오류 상황 처리
- 파일 수준 진단 정보 캐싱 지원

#### 2.2.2 실시간 진단 정보 획득

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 122-155

```javascript
async getNewDiagnostics() {
  if (!this.initialized || !this.mcpClient || this.mcpClient.type !== "connected") return [];
  let A = [];
  try {
    let G = await gw("getDiagnostics", {}, this.mcpClient, !1);
    A = this.parseDiagnosticResult(G)
  } catch (G) {
    return []
  }

  // 로컬 파일과 오른쪽 파일의 진단 정보 분리
  let B = A.filter((G) => this.baseline.has(this.normalizeFileUri(G.uri))).filter((G) => G.uri.startsWith("file://")),
      Q = new Map;
  A.filter((G) => this.baseline.has(this.normalizeFileUri(G.uri))).filter((G) => G.uri.startsWith("_claude_fs_right:")).forEach((G) => {
    Q.set(this.normalizeFileUri(G.uri), G)
  });

  // 진단 정보 변경 비교
  let I = [];
  for (let G of B) {
    let Z = this.normalizeFileUri(G.uri),
        D = this.baseline.get(Z) || [],
        Y = Q.get(Z),
        W = G;
    if (Y) {
      let F = this.rightFileDiagnosticsState.get(Z);
      if (!F || !this.areDiagnosticArraysEqual(F, Y.diagnostics)) W = Y;
      this.rightFileDiagnosticsState.set(Z, Y.diagnostics)
    }
    let J = W.diagnostics.filter((F) => !D.some((X) => this.areDiagnosticsEqual(F, X)));
    if (J.length > 0) I.push({
      uri: G.uri,
      diagnostics: J
    });
    this.baseline.set(Z, W.diagnostics)
  }
  return I
}
```

**기능 설명**:
- 모든 파일의 진단 정보 획득
- 로컬 파일과 오른쪽 파일의 분리 처리 지원
- 증분 진단 정보 감지 구현
- 새로 추가된 진단 정보 반환

### 2.3 진단 정보 파싱 및 비교

#### 2.3.1 진단 결과 파싱

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 156-162

```javascript
parseDiagnosticResult(A) {
  if (Array.isArray(A)) {
    let B = A.find((Q) => Q.type === "text");
    if (B && "text" in B) return JSON.parse(B.text)
  }
  return []
}
```

**기능 설명**:
- MCP Tool이 반환한 진단 정보 파싱
- 다양한 응답 형식 지원
- 텍스트 유형의 진단 데이터 추출

#### 2.3.2 진단 정보 비교 알고리즘

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 163-169

```javascript
areDiagnosticsEqual(A, B) {
  return A.message === B.message &&
         A.severity === B.severity &&
         A.source === B.source &&
         A.code === B.code &&
         A.range.start.line === B.range.start.line &&
         A.range.start.character === B.range.start.character &&
         A.range.end.line === B.range.end.line &&
         A.range.end.character === B.range.end.character
}

areDiagnosticArraysEqual(A, B) {
  if (A.length !== B.length) return !1;
  return A.every((Q) => B.some((I) => this.areDiagnosticsEqual(Q, I))) &&
         B.every((Q) => A.some((I) => this.areDiagnosticsEqual(I, Q)))
}
```

**기능 설명**:
- 진단 정보의 모든 필드를 정확하게 비교
- 진단 정보 배열의 깊은 비교 지원
- 진단 정보 변경 감지에 사용

## 3. MCP IDE 서버 Tool

### 3.1 IDE Tool 화이트리스트 메커니즘

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35471-35475

```javascript
c65 = ["mcp__ide__executeCode", "mcp__ide__getDiagnostics"]

function l65(A) {
  return !A.name.startsWith("mcp__ide__") || c65.includes(A.name)
}
```

**기능 설명**:
- IDE 관련 MCP Tool 화이트리스트 정의
- 특정 IDE Tool만 호출 허용
- 안전한 Tool 액세스 제어 제공

### 3.2 executeCode Tool

**소스 코드 위치**: MCP 프로토콜을 통해 호출, Tool 이름: `mcp__ide__executeCode`

**기능 설명**:
- IDE의 Jupyter 커널에서 Python 코드 실행
- 코드의 지속적인 실행 환경 지원
- 실행 결과 및 출력 반환

### 3.3 getDiagnostics Tool

**소스 코드 위치**: MCP 프로토콜을 통해 호출, Tool 이름: `mcp__ide__getDiagnostics`

**매개변수 형식**:
```javascript
// 특정 파일의 진단 정보 획득
{
  uri: "file:///path/to/file.js"
}

// 모든 파일의 진단 정보 획득
{}
```

**기능 설명**:
- LSP 진단 정보 획득
- 파일 수준 및 전역 수준 진단 지원
- 포맷팅된 진단 데이터 반환

## 4. 실시간 상태 동기화

### 4.1 IDE 상태 감지

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 23956, 33282, 52966

```javascript
// 터미널 유형 감지
return Boolean(A.WT_SESSION) || Boolean(A.TERMINUS_SUBLIME) || A.ConEmuTask === "{cmd::Cmder}" || Q === "Terminus-Sublime" || Q === "vscode" || B === "xterm-256color" || B === "alacritty" || B === "rxvt-unicode" || B === "rxvt-unicode-256color" || A.TERMINAL_EMULATOR === "JetBrains-JediTerm"

// 커서 터미널 감지
tR = mA.terminal === "cursor" || mA.terminal === "windsurf" || mA.terminal === "vscode"

// 지원되는 IDE 감지
return Aw1() === "darwin" && (mA.terminal === "iTerm.app" || mA.terminal === "Apple_Terminal") || mA.terminal === "vscode" || mA.terminal === "cursor" || mA.terminal === "windsurf" || mA.terminal === "ghostty"
```

**기능 설명**:
- 현재 실행 중인 IDE 환경 감지
- VS Code, Cursor, Windsurf 등 주류 IDE 지원
- 환경 변수를 통해 터미널 유형 식별

### 4.2 IDE 특정 기능 적응

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 33590-33599

```javascript
function ft(A) {
  switch (A) {
    case "vscode":
      return "VS Code";
    case "cursor":
      return "Cursor";
    case "windsurf":
      return "Windsurf";
    // ... 기타 IDE 매핑
  }
}
```

**기능 설명**:
- IDE 이름의 표준화된 매핑 제공
- 다양한 IDE의 특정 기능 지원
- 사용자 인터페이스 표시 및 기능 적응에 사용

### 4.3 키보드 단축키 통합

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 52994

```javascript
if (["iTerm.app", "vscode", "cursor", "windsurf", "ghostty"].includes(mA.terminal ?? "")) Q.shiftEnterKeyBindingInstalled = !0;
```

**기능 설명**:
- IDE의 Shift+Enter 단축키 지원 감지
- 단축키 바인딩 자동 활성화
- 더 나은 사용자 경험 제공

## 5. 양방향 통신 메커니즘

### 5.1 MCP 클라이언트 연결 팩토리

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35481-35520

```javascript
ue = L0(async (A, B) => {
  let Q;
  if (B.type === "stdio") {
    // STDIO 연결 구현
    let I = new Promise((V, X) => {
      let K = require("child_process").spawn(B.command, B.args || [], {
        stdio: ["pipe", "pipe", "pipe"],
        env: { ...process.env, ...B.env }
      });
      // ... 오류 처리 및 연결 구축
    });
    Q = new $N1(I, new YJ1(I))
  } else if (B.type === "http") {
    // HTTP 연결 구현
    let V = {
      authProvider: new MO(A, B),
      requestInit: {
        headers: {
          "user-agent": `Claude-Code/${v0}`,
          "Content-Type": "application/json",
          ...B.headers
        }
      }
    };
    Q = new FF1(new URL(B.url), V)
  } else if (B.type === "sse") {
    // SSE 연결 구현
    let V = {
      authProvider: new MO(A, B),
      requestInit: {
        headers: {
          "user-agent": `Claude-Code/${v0}`,
          "Content-Type": "application/json",
          ...B.headers
        }
      }
    };
    Q = new FF1(new URL(B.url), V)
  } else if (B.type === "sse-ide") {
    // SSE-IDE 연결 구현
    Q = new FF1(new URL(B.url));
  } else if (B.type === "ws-ide") {
    // WS-IDE 연결 구현
    let X = new tB1.default(B.url, ["mcp"], B.authToken ? {
      headers: {
        "X-Claude-Code-Ide-Authorization": B.authToken
      }
    } : void 0);
    Q = new Do1(X)
  }
  // ... 연결 구축 및 오류 처리
});
```

**기능 설명**:
- 통일된 MCP 클라이언트 연결 팩토리
- 다양한 전송 프로토콜 지원
- IDE 연결은 전용 구현 사용

### 5.2 IDE 연결 알림

**소스 코드 위치**: `we0(I)` 함수를 통해 전송

```javascript
// 연결 성공 후 알림 전송
try {
  we0(I)  // IDE 연결 알림 전송
} catch (X) {
  m7(A, `Failed to send ide_connected notification: ${X}`)
}
```

**기능 설명**:
- 연결 성공 후 IDE에 주도적으로 알림
- 양방향 통신 채널 구축
- 오류 처리 및 로그 기록

### 5.3 파일 작업 IDE 통합

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 32-38

```javascript
// Tool 호출 결과 처리
if ("content" in D && Array.isArray(D.content)) {
  let W = D.content,
      X = (await wJ("claude_code_unicode_sanitize") ? D$(W) : W).map((V) => No1(V, B)).flat();
  if (B !== "ide") await Zo1(X, Q, Z);  // 비IDE Tool은 특별 처리 필요
  return X
}
```

**기능 설명**:
- IDE Tool 호출 결과 특별 처리
- IDE Tool과 일반 Tool 구분
- 내용 포맷팅 및 표시 지원

## 6. 보안 및 권한 제어

### 6.1 IDE Tool 권한 제어

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35471-35475

```javascript
c65 = ["mcp__ide__executeCode", "mcp__ide__getDiagnostics"]

function l65(A) {
  return !A.name.startsWith("mcp__ide__") || c65.includes(A.name)
}
```

**기능 설명**:
- 엄격한 IDE Tool 화이트리스트 메커니즘
- 악의적인 Tool 호출 방지
- 안전한 IDE 통합 제공

### 6.2 신원 인증 및 권한 부여

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35514-35520

```javascript
// WebSocket IDE 연결 신원 인증
else if (B.type === "ws-ide") {
  let X = new tB1.default(B.url, ["mcp"], B.authToken ? {
    headers: {
      "X-Claude-Code-Ide-Authorization": B.authToken
    }
  } : void 0);
  Q = new Do1(X)
}
```

**기능 설명**:
- 토큰 기반 신원 인증 지원
- 사용자 정의 인증 헤더 사용
- 선택적 보안 메커니즘

## 7. 오류 처리 및 모니터링

### 7.1 연결 오류 처리

**소스 코드 위치**: `improved-claude-code-5.mjs`, 행번호: 35577-35578

```javascript
} else if (B.type === "sse-ide" || B.type === "ws-ide") E1("tengu_mcp_ide_server_connection_failed", {});
```

**기능 설명**:
- 연결 실패 원격 측정 이벤트 전송
- IDE 연결과 일반 연결 구분
- 모니터링 및 디버깅 정보 제공

### 7.2 진단 오류 처리

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 114-116

```javascript
if (A !== this.normalizeFileUri(I.uri)) {
  b1(new Error(`Diagnostics file path mismatch: expected ${A}, got ${I.uri})`));
  return
}
```

**기능 설명**:
- 파일 경로 불일치 오류 감지
- 상세한 오류 정보 제공
- 잘못된 진단 정보 연결 방지

## 8. 성능 최적화 메커니즘

### 8.1 진단 정보 캐싱

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 51-56

```javascript
class PK {
  baseline = new Map;                    // 진단 기준선 캐시
  lastProcessedTimestamps = new Map;     // 타임스탬프 캐시
  lastDiagnosticsByUri = new Map;        // URI 진단 캐시
  rightFileDiagnosticsState = new Map;   // 오른쪽 파일 상태 캐시
}
```

**기능 설명**:
- 다층 캐싱 메커니즘
- 중복 진단 정보 획득 감소
- 성능 및 응답 속도 향상

### 8.2 증분 업데이트 메커니즘

**소스 코드 위치**: `chunks.92.mjs`, 행번호: 147-152

```javascript
let J = W.diagnostics.filter((F) => !D.some((X) => this.areDiagnosticsEqual(F, X)));
if (J.length > 0) I.push({
  uri: G.uri,
  diagnostics: J
});
```

**기능 설명**:
- 새로 추가된 진단 정보만 반환
- 동일한 진단 중복 처리 방지
- 시스템 응답 효율성 향상

## 9. 요약

Claude Code는 MCP 프로토콜을 통해 IDE와의 깊은 통합을 구현했으며, 주요 특징은 다음과 같습니다:

### 9.1 기술적 특징

1. **프로토콜 통일**: MCP 프로토콜로 IDE 통신 통일
2. **전송 다양성**: SSE와 WebSocket 두 가지 전송 방식 지원
3. **기능 풍부**: LSP 진단, 코드 실행 등 기능 통합
4. **안전하고 신뢰성**: 엄격한 권한 제어 및 오류 처리

### 9.2 아키텍처 장점

1. **모듈화된 설계**: 각 컴포넌트의 책임이 명확함
2. **확장성 강화**: 새로운 IDE 유형 지원 용이
3. **성능 최적화**: 다층 캐싱 및 증분 업데이트
4. **사용자 경험**: 실시간 진단 및 단축키 통합

### 9.3 구현 세부사항

1. **연결 관리**: 자동 재연결 및 상태 모니터링
2. **데이터 동기화**: 실시간 진단 정보 동기화
3. **오류 처리**: 완벽한 오류 처리 메커니즘
4. **모니터링 로그**: 상세한 원격 측정 및 로그 기록

이러한 설계로 Claude Code는 현대 IDE와 원활하게 통합되어 사용자에게 매끄러운 개발 경험을 제공할 수 있습니다.
