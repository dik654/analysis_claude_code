# Claude Code MCP (Model Context Protocol) 시스템 심층 분석

## 1. MCP 구성 파일 분석

### 1.1 구성 파일 파싱 로직

`chunks.88.mjs` 및 `chunks.102.mjs` 분석을 바탕으로, MCP 구성 시스템의 핵심 구현:

#### 구성 파일 구조 정의
```javascript
// chunks.88.mjs:115-152
Av1 = n.object({
    type: n.literal("stdio").optional(),
    command: n.string().min(1, "Command cannot be empty"),
    args: n.array(n.string()).default([]),
    env: n.record(n.string()).optional()
})

lf4 = n.object({
    type: n.literal("sse"),
    url: n.string().url("Must be a valid URL"),
    headers: n.record(n.string()).optional()
})

Bv1 = n.union([Av1, lf4, if4, nf4, af4])

Ug = n.object({
    mcpServers: n.record(n.string(), Bv1)
})
```

#### 구성 로딩 로직
```javascript
// chunks.102.mjs:60-79 - 동적 구성 로딩
if (X) try {
    let YA, bA = Z8(X);
    if (bA) {
        let e1 = Ug.safeParse(bA);
        if (!e1.success) {
            let k1 = e1.error.errors.map((Q1) => `${Q1.path.join(".")}: ${Q1.message}`).join(", ");
            throw new Error(`Invalid MCP configuration: ${k1}`)
        }
        YA = e1.data.mcpServers
    } else {
        let e1 = pq5(X);
        YA = wo1(e1).mcpServers
    }
    L = UU(YA, (e1) => ({
        ...e1,
        scope: "dynamic"
    }))
} catch (YA) {
    console.error(`Error: ${YA instanceof Error?YA.message:String(YA)}`), process.exit(1)
}
```

### 1.2 다층 구성 우선순위 처리

구성의 스코프 시스템은 다음 열거형을 기반으로 합니다:
```javascript
// chunks.88.mjs:112
ef1 = n.enum(["local", "user", "project", "dynamic"])
```

#### 구성 파일 검색 순서
1. **Project-scoped (.mcp.json)**: `vC()` 함수가 프로젝트 레벨 구성 처리
2. **Local config**: `m9()` 함수가 로컬 구성 처리
3. **User/Global config**: `ZA()` 함수가 사용자 레벨 구성 처리

```javascript
// chunks.88.mjs:542-567 - 프로젝트 레벨 .mcp.json 처리
vC = L0(() => {
    let A = Iv1(dA(), ".mcp.json");
    if (!x1().existsSync(A)) return {};
    try {
        let B = x1().readFileSync(A, {
            encoding: "utf-8"
        }),
        Q = Qv4(B);
        return E1("tengu_mcpjson_found", {
            numServers: Object.keys(Q).length
        }), Q
    } catch {}
    return {}
})
```

### 1.3 구성 오류 처리 및 검증

구성 검증은 Zod schema를 사용한 엄격한 타입 검사를 적용합니다:

```javascript
// 전송 계층 타입 검증
WJ8 = n.enum(["stdio", "sse", "sse-ide", "http"])

// 구성 파싱 오류 처리
function Qv4(A) {
    let B = Z8(A), Q = {};
    if (B && typeof B === "object") {
        let I = Ug.safeParse(B);
        if (I.success) {
            let G = I.data;
            for (let [Z, D] of Object.entries(G.mcpServers)) Q[Z] = D
        } else M6(`Error parsing .mcp.json: ${I.error.message}`)
    }
    return Q
}
```

## 2. MCP 서버 라이프사이클 관리

### 2.1 서버 시작 및 연결 로직

`chunks.102.mjs`의 `AK1()` 함수 호출 분석을 기반으로, MCP 서버의 시작 프로세스:

```javascript
// chunks.102.mjs:105-109 - 서버 시작
let [s, {
    clients: d = [],
    tools: F1 = [],
    commands: X1 = []
}] = await Promise.all([J2A(), i || R ? await AK1(L) : {
    clients: [],
    tools: [],
    commands: []
}]);
```

#### 서버 상태 관리
```javascript
// chunks.102.mjs:154-159 - MCP 상태 초기화
mcp: {
    clients: [],
    tools: [],
    commands: [],
    resources: {}
}
```

### 2.2 연결 실패 처리 및 재시도 로직

MCP 서버의 오류 처리 및 재시도 메커니즘은 다음 구조로 관리됩니다:

```javascript
// chunks.102.mjs:278-290 - MCP serve 명령 오류 처리
.action(async ({
    debug: I,
    verbose: G
}) => {
    let Z = $T();
    if (E1("tengu_mcp_start", {}), !uq5(Z))
        console.error(`Error: Directory ${Z} does not exist`), process.exit(1);
    try {
        await qT(Z, "default", !1, !1), await hy2(Z, I ?? !1, G ?? !1)
    } catch (D) {
        console.error("Error: Failed to start MCP server:", D), process.exit(1)
    }
})
```

### 2.3 서버 프로세스 모니터링

서버의 실시간 상태 모니터링은 다음 원격 측정 이벤트를 통해 구현됩니다:

```javascript
// MCP 관련 원격 측정 이벤트
E1("tengu_mcp_start", {})
E1("tengu_mcp_add", { type: W, scope: Y, source: "command", transport: W })
E1("tengu_mcp_delete", { name: I, scope: J })
E1("tengu_mcp_list", {})
E1("tengu_mcpjson_found", { numServers: Object.keys(Q).length })
```

## 3. MCP 프로토콜 구현 분석

### 3.1 전송 계층 지원

MCP는 여러 전송 프로토콜을 지원합니다:

```javascript
// chunks.88.mjs:114
WJ8 = n.enum(["stdio", "sse", "sse-ide", "http"])
```

#### STDIO 전송 구현
```javascript
// chunks.88.mjs:116-121
Av1 = n.object({
    type: n.literal("stdio").optional(),
    command: n.string().min(1, "Command cannot be empty"),
    args: n.array(n.string()).default([]),
    env: n.record(n.string()).optional()
})
```

#### SSE 전송 구현
```javascript
// chunks.88.mjs:123-127
lf4 = n.object({
    type: n.literal("sse"),
    url: n.string().url("Must be a valid URL"),
    headers: n.record(n.string()).optional()
})
```

#### HTTP 전송 구현
```javascript
// chunks.88.mjs:142-146
af4 = n.object({
    type: n.literal("http"),
    url: n.string().url("Must be a valid URL"),
    headers: n.record(n.string()).optional()
})
```

### 3.2 메시지 직렬화 및 역직렬화

AWS SDK 구현을 기반으로 한 JSON-RPC 통신:

```javascript
// chunks.35.mjs:803-815 - JSON 직렬화
ry1 = e0((A) => {
    if (A == null) return {};
    if (Array.isArray(A)) return A.filter((B) => B != null).map(ry1);
    if (typeof A === "object") {
        let B = {};
        for (let Q of Object.keys(A)) {
            if (A[Q] == null) continue;
            B[Q] = ry1(A[Q])
        }
        return B
    }
    return A
}, "_json");
```

### 3.3 프로토콜 핸드셰이크 및 능력 협상

MCP Tool 및 리소스의 등록은 다음 인터페이스를 통해 이루어집니다:

```javascript
// Tool 등록 메커니즘
async function SE2(A, B) {
    return {
        name: A.name,
        description: await A.prompt({
            getToolPermissionContext: B.getToolPermissionContext,
            tools: B.tools
        }),
        input_schema: "inputJSONSchema" in A && A.inputJSONSchema ? A.inputJSONSchema : Nm(A.inputSchema)
    }
}
```

## 4. MCP Tool 통합 분석

### 4.1 외부 Tool 발견 및 등록

MCP Tool의 발견 및 등록 프로세스:

```javascript
// Tool 권한 검사
async checkPermissions(A, B) {
    return $S(gI, A, B.getToolPermissionContext())
}
```

### 4.2 MCP Tool과 내장 Tool 통합

Tool 호출의 통합 인터페이스:

```javascript
// chunks.94.mjs:26-31 - Tool 경로 가져오기
getPath(A) {
    return A.file_path
},

// Tool 실행 결과 매핑
mapToolResultToToolResultBlockParam({
    filePath: A,
    edits: B,
    userModified: Q
}, I) {
    let G = Q ? ".  The user modified your proposed changes before accepting them." : "";
    return {
        tool_use_id: I,
        type: "tool_result",
        content: `Applied ${B.length} edit${B.length===1?"":"s"} to ${A}${G}`
    }
}
```

### 4.3 Tool 권한 및 보안 제어

Tool 권한 검증 메커니즘:

```javascript
// Tool 권한 컨텍스트
toolPermissionContext: _,

// 권한 검사
r_2({
    allowedToolsCli: J,
    disallowedToolsCli: F,
    permissionMode: T,
    addDirs: E
})
```

### 4.4 Tool 호출 라우팅 및 실행

Tool 실행의 제너레이터 패턴:

```javascript
// chunks.94.mjs:215-257 - Tool 실행
async * call({
    file_path: A,
    old_string: B,
    new_string: Q,
    replace_all: I = !1
}, {
    readFileState: G,
    userModified: Z
}) {
    // Tool 실행 로직
    yield {
        type: "result",
        data: {
            filePath: A,
            oldString: B,
            newString: Q,
            originalFile: W,
            structuredPatch: J,
            userModified: Z ?? !1,
            replaceAll: I
        }
    }
}
```

## 5. MCP 명령줄 인터페이스 분석

### 5.1 MCP 서버 관리 명령

`chunks.102.mjs:278-485` 분석을 기반으로 한 MCP CLI 명령 구조:

#### mcp serve - MCP 서버 시작
```javascript
Q.command("serve")
.description(`Start the ${m0} MCP server`)
.option("-d, --debug", "Enable debug mode")
.option("--verbose", "Override verbose mode setting")
.action(async ({ debug: I, verbose: G }) => {
    // 서버 시작 로직
})
```

#### mcp add - 서버 추가
```javascript
Q.command("add <name> <commandOrUrl> [args...]")
.description("Add a server")
.option("-s, --scope <scope>", "Configuration scope (local, user, or project)", "local")
.option("-t, --transport <transport>", "Transport type (stdio, sse, http)", "stdio")
.option("-e, --env <env...>", "Set environment variables")
.option("-H, --header <header...>", "Set HTTP headers")
```

#### mcp remove - 서버 제거
```javascript
Q.command("remove <name>")
.description("Remove an MCP server")
.option("-s, --scope <scope>", "Configuration scope")
```

#### mcp list - 서버 목록 표시
```javascript
Q.command("list")
.description("List configured MCP servers")
.action(async () => {
    let I = DV();
    if (Object.keys(I).length === 0)
        console.log("No MCP servers configured. Use `claude mcp add` to add a server.");
})
```

### 5.2 구성 가져오기 기능

Claude Desktop에서 구성 가져오기:

```javascript
Q.command("add-from-claude-desktop")
.description("Import MCP servers from Claude Desktop (Mac and WSL only)")
.action(async (I) => {
    let G = cd(I.scope),
        Z = Z7(),
        D = _y2();
    // 가져오기 로직
})
```

### 5.3 프로젝트 구성 재설정

프로젝트 레벨 MCP 구성 재설정:

```javascript
Q.command("reset-project-choices")
.description("Reset all approved and rejected project-scoped (.mcp.json) servers")
.action(async () => {
    let I = m9();
    B5({
        ...I,
        enabledMcpjsonServers: [],
        disabledMcpjsonServers: [],
        enableAllProjectMcpServers: !1
    });
})
```

## 6. 보안 메커니즘 분석

### 6.1 MCP 서버 권한 제어

구성 스코프는 다양한 레벨의 권한 제어를 제공합니다:

```javascript
// 스코프 검증
function cd(A) {
    // 스코프 검증 로직
}

// 프로젝트 레벨 구성의 보안 제어
enabledMcpjsonServers: [],
disabledMcpjsonServers: [],
enableAllProjectMcpServers: !1
```

### 6.2 구성 파일 권한 검증

프로젝트 레벨 .mcp.json 파일의 권한 확인 메커니즘:

```javascript
// 프로젝트 MCP 서버 선택 재설정
await E1("tengu_mcp_reset_mcpjson_choices", {});
```

### 6.3 환경 변수 및 헤더 보안

환경 변수 및 HTTP 헤더의 보안 처리:

```javascript
// chunks.88.mjs:80-89 - 환경 변수 파싱
function eZ0(A) {
    let B = {};
    if (A)
        for (let Q of A) {
            let [I, ...G] = Q.split("=");
            if (!I || G.length === 0)
                throw new Error(`Invalid environment variable format: ${Q}`);
            B[I] = G.join("=")
        }
    return B
}
```

## 7. 진단 및 디버깅 기능

### 7.1 MCP 디버그 모드

```javascript
// 디버그 옵션
.option("--mcp-debug", "[DEPRECATED. Use --debug instead] Enable MCP debug mode")
.option("-d, --debug", "Enable debug mode")
```

### 7.2 상세 로그 출력

```javascript
// 상세 출력 제어
.option("--verbose", "Override verbose mode setting from config")
```

### 7.3 오류 진단

원격 측정 시스템을 통한 오류 추적 및 진단:

```javascript
// MCP 관련 원격 측정 이벤트
E1("tengu_mcp_start", {})
E1("tengu_mcp_add", { scope, source, transport })
E1("tengu_mcp_delete", { name, scope })
```

## 8. 요약

Claude Code의 MCP 시스템은 다음과 같은 핵심 특성을 보여줍니다:

1. **다층 구성 관리**: local, user, project, dynamic 네 가지 스코프 지원
2. **다중 전송 프로토콜 지원**: STDIO, SSE, HTTP 등 다양한 전송 방식
3. **강타입 구성 검증**: Zod schema 기반의 엄격한 타입 검사
4. **완전한 CLI Tool 체인**: 서버 관리, 구성 가져오기, 디버깅 등의 기능
5. **보안 권한 제어**: 세분화된 권한 관리 및 구성 격리
6. **실시간 모니터링 및 진단**: 완전한 원격 측정 및 오류 처리 메커니즘

MCP 시스템의 설계는 Claude Code의 확장성과 보안성에 대한 중시를 충분히 반영하며, 제3자 Tool 통합을 위한 표준화되고 안전하게 제어 가능한 인터페이스를 제공합니다.
