# Kode-CLI MCP 통합 시스템 분석

## 목차
1. [개요](#개요)
2. [MCP 클라이언트 구현](#mcp-클라이언트-구현)
3. [MCP 서버 승인 시스템](#mcp-서버-승인-시스템)
4. [MCP 도구 시스템](#mcp-도구-시스템)
5. [URL Fetcher 도구](#url-fetcher-도구)
6. [Web Search 도구](#web-search-도구)
7. [HTML to Markdown 변환](#html-to-markdown-변환)
8. [캐싱 메커니즘](#캐싱-메커니즘)
9. [아키텍처 패턴 및 설계](#아키텍처-패턴-및-설계)

---

## 개요

Kode-CLI는 **Model Context Protocol (MCP)**을 통해 외부 서버와 통합하여 기능을 확장할 수 있는 시스템을 제공합니다. MCP는 AI 모델이 외부 도구 및 데이터 소스와 통신할 수 있는 표준 프로토콜입니다.

### 주요 컴포넌트
- **MCP Client**: 외부 MCP 서버와의 연결 및 통신 관리
- **MCP Server Approval**: 보안을 위한 서버 승인 시스템
- **MCP Tool**: 동적으로 로드되는 외부 도구
- **URL Fetcher Tool**: 웹 페이지 콘텐츠 가져오기 및 분석
- **Web Search Tool**: 웹 검색 기능 제공

---

## MCP 클라이언트 구현

### 1. 서버 설정 관리

MCP 서버는 3가지 범위(scope)로 관리됩니다:

```typescript
// 설정 범위 타입
const VALID_SCOPES = ['project', 'global', 'mcprc'] as const
type ConfigScope = (typeof VALID_SCOPES)[number]

// 우선순위: project > mcprc > global
export function listMCPServers(): Record<string, McpServerConfig> {
  const globalConfig = getGlobalConfig()
  const mcprcConfig = getMcprcConfig()
  const projectConfig = getCurrentProjectConfig()
  return {
    ...(globalConfig.mcpServers ?? {}),
    ...(mcprcConfig ?? {}),        // .mcprc가 global 우선
    ...(projectConfig.mcpServers ?? {}), // project가 최우선
  }
}
```

#### 설정 범위별 특징

1. **Global Scope** (`~/.kode.json`)
   - 모든 프로젝트에서 사용 가능
   - 사용자 레벨 공통 도구

2. **MCPRC Scope** (`.mcprc`)
   - 프로젝트별 MCP 서버 설정
   - 승인 시스템 적용 대상
   - Git으로 공유 가능

3. **Project Scope** (`.kode.json`)
   - 프로젝트별 설정
   - 최고 우선순위

### 2. 서버 연결 프로토콜

MCP는 두 가지 전송 방식을 지원합니다:

#### STDIO 전송 (로컬 프로세스)

```typescript
async function connectToServer(
  name: string,
  serverRef: McpServerConfig,
): Promise<Client> {
  const transport =
    serverRef.type === 'sse'
      ? new SSEClientTransport(new URL(serverRef.url))
      : new StdioClientTransport({
          command: serverRef.command,
          args: serverRef.args,
          env: {
            ...process.env,
            ...serverRef.env,
          } as Record<string, string>,
          stderr: 'pipe', // MCP 서버 에러 출력 방지
        })

  const client = new Client(
    {
      name: PRODUCT_COMMAND,
      version: '0.1.0',
    },
    {
      capabilities: {},
    },
  )

  // 타임아웃 설정 (5초)
  const CONNECTION_TIMEOUT_MS = 5000
  const connectPromise = client.connect(transport)
  const timeoutPromise = new Promise<never>((_, reject) => {
    const timeoutId = setTimeout(() => {
      reject(
        new Error(
          `Connection to MCP server "${name}" timed out after ${CONNECTION_TIMEOUT_MS}ms`,
        ),
      )
    }, CONNECTION_TIMEOUT_MS)

    connectPromise.then(
      () => clearTimeout(timeoutId),
      () => clearTimeout(timeoutId),
    )
  })

  await Promise.race([connectPromise, timeoutPromise])

  // stderr 에러 로깅 설정
  if (serverRef.type === 'stdio') {
    ;(transport as StdioClientTransport).stderr?.on('data', (data: Buffer) => {
      const errorText = data.toString().trim()
      if (errorText) {
        logMCPError(name, `Server stderr: ${errorText}`)
      }
    })
  }

  return client
}
```

**STDIO 방식 특징:**
- 로컬 프로세스로 MCP 서버 실행
- `command`, `args`, `env`로 서버 프로세스 시작
- stderr를 pipe로 연결하여 에러 로깅
- 예: Node.js 스크립트, Python 서버 등

#### SSE 전송 (원격 HTTP)

```typescript
serverRef.type === 'sse'
  ? new SSEClientTransport(new URL(serverRef.url))
```

**SSE 방식 특징:**
- Server-Sent Events를 통한 HTTP 통신
- 원격 MCP 서버 연결
- URL 기반 연결

### 3. 클라이언트 라이프사이클 관리

```typescript
type ConnectedClient = {
  client: Client
  name: string
  type: 'connected'
}
type FailedClient = {
  name: string
  type: 'failed'
}
export type WrappedClient = ConnectedClient | FailedClient

// Memoized 클라이언트 연결
export const getClients = memoize(async (): Promise<WrappedClient[]> => {
  // CI 환경에서는 MCP 비활성화 (hang 방지)
  if (process.env.CI && process.env.NODE_ENV !== 'test') {
    return []
  }

  const globalServers = getGlobalConfig().mcpServers ?? {}
  const mcprcServers = getMcprcConfig()
  const projectServers = getCurrentProjectConfig().mcpServers ?? {}

  // .mcprc 서버 중 승인된 것만 필터링
  const approvedMcprcServers = pickBy(
    mcprcServers,
    (_, name) => getMcprcServerStatus(name) === 'approved',
  )

  const allServers = {
    ...globalServers,
    ...approvedMcprcServers,
    ...projectServers,
  }

  // 병렬로 모든 서버 연결 시도
  return await Promise.all(
    Object.entries(allServers).map(async ([name, serverRef]) => {
      try {
        const client = await connectToServer(name, serverRef as McpServerConfig)
        return { name, client, type: 'connected' as const }
      } catch (error) {
        logMCPError(
          name,
          `Connection failed: ${error instanceof Error ? error.message : String(error)}`,
        )
        return { name, type: 'failed' as const }
      }
    }),
  )
})
```

**주요 특징:**
- **Memoization**: 한 번 연결한 클라이언트는 재사용
- **병렬 연결**: 모든 서버에 동시 연결 시도
- **에러 핸들링**: 연결 실패 시에도 계속 진행
- **타입 안전성**: 연결 성공/실패 타입 구분

### 4. MCP 도구 동적 로딩

```typescript
export const getMCPTools = memoize(async (): Promise<Tool[]> => {
  const toolsList = await requestAll<
    ListToolsResult,
    typeof ListToolsResultSchema
  >(
    {
      method: 'tools/list',
    },
    ListToolsResultSchema,
    'tools',
  )

  return toolsList.flatMap(({ client, result: { tools } }) =>
    tools.map(
      (tool): Tool => ({
        ...MCPTool,
        // MCP 도구 이름 규칙: mcp__[서버명]__[도구명]
        name: 'mcp__' + client.name + '__' + tool.name,
        async description() {
          return tool.description ?? ''
        },
        async prompt() {
          return tool.description ?? ''
        },
        inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
        async validateInput(input, context) {
          // MCP 도구는 자체 스키마 검증
          return { result: true }
        },
        async *call(args: Record<string, unknown>, context) {
          const data = await callMCPTool({ client, tool: tool.name, args })
          yield {
            type: 'result' as const,
            data,
            resultForAssistant: data,
          }
        },
        userFacingName() {
          return `${client.name}:${tool.name} (MCP)`
        },
      }),
    ),
  )
})
```

**MCP 도구 네이밍 규칙:**
- 내부 이름: `mcp__[server]__[tool]` (예: `mcp__filesystem__read_file`)
- 사용자 표시: `[server]:[tool] (MCP)` (예: `filesystem:read_file (MCP)`)

### 5. MCP 도구 실행

```typescript
async function callMCPTool({
  client: { client, name },
  tool,
  args,
}: {
  client: ConnectedClient
  tool: string
  args: Record<string, unknown>
}): Promise<ToolResultBlockParam['content']> {
  const result = await client.callTool(
    {
      name: tool,
      arguments: args,
    },
    CallToolResultSchema,
  )

  // 에러 응답 처리
  if ('isError' in result && result.isError) {
    const errorMessage = `Error calling tool ${tool}: ${result.error}`
    logMCPError(name, errorMessage)
    throw Error(errorMessage)
  }

  // toolResult 타입 응답
  if ('toolResult' in result) {
    return String(result.toolResult)
  }

  // content 배열 응답 (이미지 포함 가능)
  if ('content' in result && Array.isArray(result.content)) {
    return result.content.map(item => {
      if (item.type === 'image') {
        return {
          type: 'image',
          source: {
            type: 'base64',
            data: String(item.data),
            media_type: item.mimeType as ImageBlockParam.Source['media_type'],
          },
        }
      }
      return item
    })
  }

  throw Error(`Unexpected response format from tool ${tool}`)
}
```

**응답 타입:**
1. **String Result**: 단순 문자열 응답
2. **Content Array**: 텍스트 및 이미지 혼합 응답
3. **Error**: 도구 실행 실패

### 6. MCP 명령(Prompt) 지원

MCP는 도구뿐만 아니라 사전 정의된 프롬프트(명령)도 제공합니다:

```typescript
export const getMCPCommands = memoize(async (): Promise<Command[]> => {
  const results = await requestAll<
    ListPromptsResult,
    typeof ListPromptsResultSchema
  >(
    {
      method: 'prompts/list',
    },
    ListPromptsResultSchema,
    'prompts',
  )

  return results.flatMap(({ client, result }) =>
    result.prompts?.map(_ => {
      const argNames = Object.values(_.arguments ?? {}).map(k => k.name)
      return {
        type: 'prompt',
        name: 'mcp__' + client.name + '__' + _.name,
        description: _.description ?? '',
        isEnabled: true,
        isHidden: false,
        progressMessage: 'running',
        userFacingName() {
          return `${client.name}:${_.name} (MCP)`
        },
        argNames,
        async getPromptForCommand(args: string) {
          const argsArray = args.split(' ')
          return await runCommand(
            { name: _.name, client },
            zipObject(argNames, argsArray),
          )
        },
      }
    }),
  )
})

export async function runCommand(
  { name, client }: { name: string; client: ConnectedClient },
  args: Record<string, string>,
): Promise<MessageParam[]> {
  try {
    const result = await client.client.getPrompt({ name, arguments: args })
    return result.messages.map(
      (message): MessageParam => ({
        role: message.role,
        content: [
          message.content.type === 'text'
            ? {
                type: 'text',
                text: message.content.text,
              }
            : {
                type: 'image',
                source: {
                  data: String(message.content.data),
                  media_type: message.content
                    .mimeType as ImageBlockParam.Source['media_type'],
                  type: 'base64',
                },
              },
        ],
      }),
    )
  } catch (error) {
    logMCPError(
      client.name,
      `Error running command '${name}': ${error instanceof Error ? error.message : String(error)}`,
    )
    throw error
  }
}
```

**MCP 명령 특징:**
- 인수를 받아 메시지 배열 반환
- 사전 구성된 프롬프트 템플릿
- 텍스트 및 이미지 지원

---

## MCP 서버 승인 시스템

### 1. 승인 상태 관리

`.mcprc` 파일의 서버는 사용 전 승인이 필요합니다:

```typescript
export function getMcprcServerStatus(
  serverName: string,
): 'approved' | 'rejected' | 'pending' {
  const config = getCurrentProjectConfig()
  if (config.approvedMcprcServers?.includes(serverName)) {
    return 'approved'
  }
  if (config.rejectedMcprcServers?.includes(serverName)) {
    return 'rejected'
  }
  return 'pending'
}
```

**상태 종류:**
- **pending**: 아직 승인 대기 중
- **approved**: 사용자가 승인함
- **rejected**: 사용자가 거부함

### 2. 승인 프로세스

```typescript
export async function handleMcprcServerApprovals(): Promise<void> {
  const mcprcServers = getMcprcConfig()
  const pendingServers = Object.keys(mcprcServers).filter(
    serverName => getMcprcServerStatus(serverName) === 'pending',
  )

  if (pendingServers.length === 0) {
    return
  }

  await new Promise<void>(resolve => {
    const clearScreenAndResolve = () => {
      // 대화 상자 후 화면 지우기
      process.stdout.write('\x1b[2J\x1b[3J\x1b[H', () => {
        resolve()
      })
    }

    // 단일 서버 승인
    if (pendingServers.length === 1 && pendingServers[0] !== undefined) {
      const result = render(
        <MCPServerApprovalDialog
          serverName={pendingServers[0]}
          onDone={() => {
            result.unmount?.()
            clearScreenAndResolve()
          }}
        />,
        { exitOnCtrlC: false },
      )
    }
    // 다중 서버 선택
    else {
      const result = render(
        <MCPServerMultiselectDialog
          serverNames={pendingServers}
          onDone={() => {
            result.unmount?.()
            clearScreenAndResolve()
          }}
        />,
        { exitOnCtrlC: false },
      )
    }
  })
}
```

**승인 UI:**
- **단일 서버**: Yes/No 대화 상자
- **다중 서버**: 체크박스 선택 UI
- **승인 결과**: `.kode.json`의 `approvedMcprcServers` 또는 `rejectedMcprcServers`에 저장

### 3. 보안 모델

```typescript
// 승인된 서버만 연결
const approvedMcprcServers = pickBy(
  mcprcServers,
  (_, name) => getMcprcServerStatus(name) === 'approved',
)

const allServers = {
  ...globalServers,
  ...approvedMcprcServers, // 승인된 것만 포함
  ...projectServers,
}
```

**보안 설계 원칙:**
1. `.mcprc` 파일은 프로젝트에 포함되어 공유될 수 있음
2. 외부에서 온 MCP 서버를 자동으로 신뢰하지 않음
3. 사용자가 명시적으로 승인해야만 사용 가능
4. 거부한 서버는 다시 묻지 않음 (명시적 해제 필요)

---

## MCP 도구 시스템

### 1. MCPTool 기본 구조

```typescript
export const MCPTool = {
  async isEnabled() {
    return true
  },
  isReadOnly() {
    return false
  },
  isConcurrencySafe() {
    return false // MCP 도구는 상태 변경 가능, 동시 실행 불가
  },
  name: 'mcp', // mcpClient.ts에서 재정의됨
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  inputSchema: z.object({}).passthrough(), // 모든 입력 허용
  async *call() {
    yield {
      type: 'result',
      data: '',
      resultForAssistant: '',
    }
  },
  needsPermissions() {
    return true
  },
  renderToolUseMessage(input) {
    return Object.entries(input)
      .map(([key, value]) => `${key}: ${JSON.stringify(value)}`)
      .join(', ')
  },
  userFacingName: () => 'mcp',
  renderToolUseRejectedMessage() {
    return <FallbackToolUseRejectedMessage />
  },
  renderToolResultMessage(output) {
    const verbose = false
    if (Array.isArray(output)) {
      return (
        <Box flexDirection="column">
          {output.map((item, i) => {
            if (item.type === 'image') {
              return (
                <Box key={i} justifyContent="space-between" overflowX="hidden" width="100%">
                  <Box flexDirection="row">
                    <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
                    <Text>[Image]</Text>
                  </Box>
                </Box>
              )
            }
            const lines = item.text.split('\n').length
            return (
              <OutputLine key={i} content={item.text} lines={lines} verbose={verbose} />
            )
          })}
        </Box>
      )
    }

    if (!output) {
      return (
        <Box justifyContent="space-between" overflowX="hidden" width="100%">
          <Box flexDirection="row">
            <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
            <Text color={getTheme().secondaryText}>(No content)</Text>
          </Box>
        </Box>
      )
    }

    const lines = output.split('\n').length
    return <OutputLine content={output} lines={lines} verbose={verbose} />
  },
  renderResultForAssistant(content) {
    return content
  },
} satisfies Tool<typeof inputSchema, string>
```

### 2. 동적 도구 생성

MCP 도구는 `mcpClient.ts`에서 동적으로 생성됩니다:

```typescript
return toolsList.flatMap(({ client, result: { tools } }) =>
  tools.map(
    (tool): Tool => ({
      ...MCPTool, // 기본 MCPTool 확장
      name: 'mcp__' + client.name + '__' + tool.name,
      async description() {
        return tool.description ?? ''
      },
      inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
      async *call(args: Record<string, unknown>, context) {
        const data = await callMCPTool({ client, tool: tool.name, args })
        yield {
          type: 'result' as const,
          data,
          resultForAssistant: data,
        }
      },
      userFacingName() {
        return `${client.name}:${tool.name} (MCP)`
      },
    }),
  ),
)
```

**동적 생성 특징:**
- MCP 서버가 제공하는 스키마 사용
- 각 도구마다 고유한 Tool 인스턴스 생성
- 서버별 네임스페이스 분리

### 3. UI 렌더링

MCP 도구는 다양한 출력 형식을 지원합니다:

**텍스트 출력:**
```typescript
const lines = output.split('\n').length
return <OutputLine content={output} lines={lines} verbose={verbose} />
```

**이미지 출력:**
```typescript
if (item.type === 'image') {
  return (
    <Box key={i}>
      <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
      <Text>[Image]</Text>
    </Box>
  )
}
```

**빈 출력:**
```typescript
if (!output) {
  return (
    <Box>
      <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
      <Text color={getTheme().secondaryText}>(No content)</Text>
    </Box>
  )
}
```

---

## URL Fetcher 도구

URL Fetcher는 웹 페이지를 가져와서 AI로 분석하는 도구입니다.

### 1. 스키마 정의

```typescript
const inputSchema = z.strictObject({
  url: z.string().url().describe('The URL to fetch content from'),
  prompt: z.string().describe('The prompt to run on the fetched content'),
})

type Input = z.infer<typeof inputSchema>
type Output = {
  url: string
  fromCache: boolean
  aiAnalysis: string
}
```

### 2. URL 정규화

```typescript
function normalizeUrl(url: string): string {
  // HTTP를 HTTPS로 자동 업그레이드
  if (url.startsWith('http://')) {
    return url.replace('http://', 'https://')
  }
  return url
}
```

**정규화 규칙:**
- HTTP → HTTPS 자동 변환
- 보안 연결 우선

### 3. 웹 페이지 가져오기

```typescript
async *call({ url, prompt }: Input, {}: ToolUseContext) {
  const normalizedUrl = normalizeUrl(url)

  try {
    let content: string
    let fromCache = false

    // 1. 캐시 확인
    const cachedContent = urlCache.get(normalizedUrl)
    if (cachedContent) {
      content = cachedContent
      fromCache = true
    } else {
      // 2. URL에서 가져오기 (30초 타임아웃)
      const abortController = new AbortController()
      const timeout = setTimeout(() => abortController.abort(), 30000)

      const response = await fetch(normalizedUrl, {
        method: 'GET',
        headers: {
          'User-Agent': 'Mozilla/5.0 (compatible; URLFetcher/1.0)',
          'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
          'Accept-Language': 'en-US,en;q=0.5',
          'Accept-Encoding': 'gzip, deflate',
          'Connection': 'keep-alive',
          'Upgrade-Insecure-Requests': '1',
        },
        signal: abortController.signal,
        redirect: 'follow',
      })

      clearTimeout(timeout)

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      const contentType = response.headers.get('content-type') || ''
      if (!contentType.includes('text/') && !contentType.includes('application/')) {
        throw new Error(`Unsupported content type: ${contentType}`)
      }

      const html = await response.text()
      content = convertHtmlToMarkdown(html)

      // 3. 캐시에 저장
      urlCache.set(normalizedUrl, content)
      fromCache = false
    }

    // 4. 콘텐츠 크기 제한 (50,000자 ≈ 15k 토큰)
    const maxContentLength = 50000
    const truncatedContent = content.length > maxContentLength
      ? content.substring(0, maxContentLength) + '\n\n[Content truncated due to length]'
      : content

    // 5. AI 분석 (캐시된 콘텐츠도 항상 새로 분석)
    const systemPrompt = [
      'You are analyzing web content based on a user\'s specific request.',
      'The content has been extracted from a webpage and converted to markdown.',
      'Provide a focused response that directly addresses the user\'s prompt.',
    ]

    const userPrompt = `Here is the content from ${normalizedUrl}:

${truncatedContent}

User request: ${prompt}`

    const aiResponse = await queryQuick({
      systemPrompt,
      userPrompt,
      enablePromptCaching: false,
    })

    const output: Output = {
      url: normalizedUrl,
      fromCache,
      aiAnalysis: aiResponse.message.content[0]?.text || 'Unable to analyze content',
    }

    yield {
      type: 'result' as const,
      resultForAssistant: this.renderResultForAssistant(output),
      data: output,
    }
  } catch (error: any) {
    const output: Output = {
      url: normalizedUrl,
      fromCache: false,
      aiAnalysis: '',
    }

    yield {
      type: 'result' as const,
      resultForAssistant: `Error processing URL ${normalizedUrl}: ${error.message}`,
      data: output,
    }
  }
}
```

**프로세스 흐름:**
1. URL 정규화 (HTTP → HTTPS)
2. 캐시 확인
3. 캐시 미스 시 HTTP 요청
4. HTML → Markdown 변환
5. 콘텐츠 크기 제한
6. AI 분석 수행
7. 결과 반환

### 4. HTTP 요청 설정

```typescript
headers: {
  'User-Agent': 'Mozilla/5.0 (compatible; URLFetcher/1.0)',
  'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  'Accept-Language': 'en-US,en;q=0.5',
  'Accept-Encoding': 'gzip, deflate',
  'Connection': 'keep-alive',
  'Upgrade-Insecure-Requests': '1',
},
signal: abortController.signal,
redirect: 'follow',
```

**헤더 설정 이유:**
- **User-Agent**: 일부 사이트는 User-Agent 필수
- **Accept**: HTML 및 XML 우선 수락
- **redirect: 'follow'**: 리다이렉트 자동 추적
- **signal**: 타임아웃 구현

### 5. AI 분석 통합

```typescript
const aiResponse = await queryQuick({
  systemPrompt,
  userPrompt,
  enablePromptCaching: false,
})
```

**queryQuick 특징:**
- 빠른 모델 사용 (Haiku 등)
- 프롬프트 캐싱 비활성화 (매번 다른 콘텐츠)
- 웹 콘텐츠 분석에 최적화

### 6. UI 렌더링

```typescript
renderToolResultMessage(output: Output) {
  const statusText = output.fromCache ? 'from cache' : 'fetched'

  return (
    <Box justifyContent="space-between" width="100%">
      <Box flexDirection="row">
        <Text>&nbsp;&nbsp;⎿ &nbsp;Content </Text>
        <Text bold>{statusText} </Text>
        <Text>and analyzed</Text>
      </Box>
      <Cost costUSD={0} durationMs={0} debug={false} />
    </Box>
  )
}

renderResultForAssistant(output: Output) {
  if (!output.aiAnalysis.trim()) {
    return `No content could be analyzed from URL: ${output.url}`
  }

  return output.aiAnalysis
}
```

**사용자 UI:**
- 캐시 사용 여부 표시
- 간결한 상태 메시지

**AI 응답:**
- 분석 결과만 전달
- 에러 시 명확한 메시지

---

## Web Search 도구

Web Search는 DuckDuckGo를 통해 웹 검색을 수행합니다.

### 1. 스키마 정의

```typescript
const inputSchema = z.strictObject({
  query: z.string().describe('The search query'),
})

type Input = z.infer<typeof inputSchema>
type Output = {
  durationMs: number
  results: SearchResult[]
}
```

### 2. 검색 실행

```typescript
async *call({ query }: Input, {}: ToolUseContext) {
  const start = Date.now()

  try {
    const searchResults = await searchProviders.duckduckgo.search(query)

    const output: Output = {
      results: searchResults,
      durationMs: Date.now() - start,
    }

    yield {
      type: 'result' as const,
      resultForAssistant: this.renderResultForAssistant(output),
      data: output,
    }
  } catch (error: any) {
    const output: Output = {
      results: [],
      durationMs: Date.now() - start,
    }
    yield {
      type: 'result' as const,
      resultForAssistant: `An error occurred during web search with DuckDuckGo: ${error.message}`,
      data: output,
    }
  }
}
```

### 3. DuckDuckGo 검색 제공자

```typescript
export interface SearchResult {
  title: string
  snippet: string
  link: string
}

export interface SearchProvider {
  search: (query: string, apiKey?: string) => Promise<SearchResult[]>
  isEnabled: (apiKey?: string) => boolean
}

const duckDuckGoSearchProvider: SearchProvider = {
  isEnabled: () => true, // API 키 불필요
  search: async (query: string): Promise<SearchResult[]> => {
    const response = await fetch(`https://html.duckduckgo.com/html/?q=${encodeURIComponent(query)}`, {
      headers: {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
      }
    });

    if (!response.ok) {
      throw new Error(`DuckDuckGo search failed with status: ${response.status}`);
    }

    const html = await response.text();
    const root = parse(html);
    const results: SearchResult[] = [];

    const resultNodes = root.querySelectorAll('.result.web-result');

    for (const node of resultNodes) {
      const titleNode = node.querySelector('.result__a');
      const snippetNode = node.querySelector('.result__snippet');

      if (titleNode && snippetNode) {
        const title = titleNode.text;
        const link = titleNode.getAttribute('href');
        const snippet = snippetNode.text;

        if (title && link && snippet) {
          // 링크 정리 - DuckDuckGo 리다이렉트 URL 제거
          let cleanLink = link;
          if (link.startsWith('https://duckduckgo.com/l/?uddg=')) {
            try {
              const url = new URL(link);
              cleanLink = url.searchParams.get('uddg') || link;
            } catch {
              cleanLink = link;
            }
          }
          results.push({
            title: title.trim(),
            snippet: snippet.trim(),
            link: cleanLink
          });
        }
      }
    }

    return results;
  },
}

export const searchProviders = {
  duckduckgo: duckDuckGoSearchProvider,
}
```

**DuckDuckGo HTML 스크래핑:**
1. HTML 버전 DuckDuckGo 사용 (API 키 불필요)
2. `node-html-parser`로 HTML 파싱
3. `.result.web-result` CSS 선택자로 결과 추출
4. 제목, 스니펫, 링크 추출
5. DuckDuckGo 리다이렉트 URL 정리

### 4. 결과 포맷팅

```typescript
renderResultForAssistant(output: Output) {
  if (output.results.length === 0) {
    return `No results found using DuckDuckGo.`
  }

  let result = `Found ${output.results.length} search results using DuckDuckGo:\n\n`

  output.results.forEach((item, index) => {
    result += `${index + 1}. **${item.title}**\n`
    result += `   ${item.snippet}\n`
    result += `   Link: ${item.link}\n\n`
  })

  result += `You can reference these results to provide current, accurate information to the user.`
  return result
}
```

**Markdown 형식:**
```
Found 5 search results using DuckDuckGo:

1. **Example Title**
   Example snippet text...
   Link: https://example.com

2. **Another Result**
   ...
```

### 5. UI 메시지

```typescript
renderToolUseMessage({ query }: Input) {
  return `Searching for: "${query}" using DuckDuckGo`
}

renderToolResultMessage(output: Output) {
  return (
    <Box justifyContent="space-between" width="100%">
      <Box flexDirection="row">
        <Text>&nbsp;&nbsp;⎿ &nbsp;Found </Text>
        <Text bold>{output.results.length} </Text>
        <Text>
          {output.results.length === 1 ? 'result' : 'results'} using DuckDuckGo
        </Text>
      </Box>
      <Cost costUSD={0} durationMs={output.durationMs} debug={false} />
    </Box>
  )
}
```

**UI 표시:**
- 검색 쿼리 표시
- 결과 개수 표시
- 실행 시간 표시

---

## HTML to Markdown 변환

### 1. Turndown 서비스 설정

```typescript
import TurndownService from 'turndown'

const turndownService = new TurndownService({
  headingStyle: 'atx',         // # 스타일 헤딩
  hr: '---',                   // 수평선
  bulletListMarker: '-',       // 불릿 리스트
  codeBlockStyle: 'fenced',    // ``` 코드 블록
  fence: '```',                // 코드 펜스
  emDelimiter: '_',            // 이탤릭
  strongDelimiter: '**'        // 볼드
})
```

### 2. 커스텀 변환 규칙

#### 스크립트 및 스타일 제거

```typescript
turndownService.addRule('removeScripts', {
  filter: ['script', 'style', 'noscript'],
  replacement: () => ''
})
```

#### HTML 주석 제거

```typescript
turndownService.addRule('removeComments', {
  filter: (node) => node.nodeType === 8, // Comment nodes
  replacement: () => ''
})
```

#### 안전한 링크 처리

```typescript
turndownService.addRule('cleanLinks', {
  filter: 'a',
  replacement: (content, node) => {
    const href = node.getAttribute('href')
    // JavaScript 링크 및 앵커 제거
    if (!href || href.startsWith('javascript:') || href.startsWith('#')) {
      return content
    }
    return `[${content}](${href})`
  }
})
```

### 3. 변환 함수

```typescript
export function convertHtmlToMarkdown(html: string): string {
  try {
    // 1. HTML 정리
    const cleanHtml = html
      .replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '') // 스크립트 제거
      .replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '')   // 스타일 제거
      .replace(/<!--[\s\S]*?-->/g, '')                   // 주석 제거
      .replace(/\s+/g, ' ')                              // 공백 정규화
      .trim()

    // 2. Markdown 변환
    const markdown = turndownService.turndown(cleanHtml)

    // 3. Markdown 정리
    return markdown
      .replace(/\n{3,}/g, '\n\n')      // 과도한 줄바꿈 제거
      .replace(/^\s+|\s+$/gm, '')      // 줄 양끝 공백 제거
      .trim()
  } catch (error) {
    throw new Error(`Failed to convert HTML to markdown: ${error instanceof Error ? error.message : String(error)}`)
  }
}
```

**변환 프로세스:**
1. **HTML 전처리**
   - 스크립트 태그 제거
   - 스타일 태그 제거
   - HTML 주석 제거
   - 공백 정규화

2. **Markdown 변환**
   - Turndown으로 변환
   - 커스텀 규칙 적용

3. **Markdown 후처리**
   - 과도한 줄바꿈 제거
   - 공백 정리

**변환 예시:**
```html
<!-- 입력 HTML -->
<h1>Title</h1>
<p>This is a <strong>paragraph</strong> with a <a href="https://example.com">link</a>.</p>
<script>alert('removed')</script>

<!-- 출력 Markdown -->
# Title

This is a **paragraph** with a [link](https://example.com).
```

---

## 캐싱 메커니즘

### 1. URLCache 클래스

```typescript
interface CacheEntry {
  content: string
  timestamp: number
}

class URLCache {
  private cache = new Map<string, CacheEntry>()
  private readonly CACHE_DURATION = 15 * 60 * 1000 // 15분

  set(url: string, content: string): void {
    this.cache.set(url, {
      content,
      timestamp: Date.now()
    })
  }

  get(url: string): string | null {
    const entry = this.cache.get(url)
    if (!entry) {
      return null
    }

    // 만료 확인
    if (Date.now() - entry.timestamp > this.CACHE_DURATION) {
      this.cache.delete(url)
      return null
    }

    return entry.content
  }

  clear(): void {
    this.cache.clear()
  }

  // 만료된 항목 정리
  private cleanExpired(): void {
    const now = Date.now()
    for (const [url, entry] of this.cache.entries()) {
      if (now - entry.timestamp > this.CACHE_DURATION) {
        this.cache.delete(url)
      }
    }
  }

  // 5분마다 자동 정리
  constructor() {
    setInterval(() => {
      this.cleanExpired()
    }, 5 * 60 * 1000) // 5분
  }
}

// 싱글톤 인스턴스
export const urlCache = new URLCache()
```

### 2. 캐시 전략

**TTL (Time To Live):**
- 캐시 유효 시간: 15분
- 만료 후 자동 삭제

**자동 정리:**
- 5분마다 만료 항목 스캔
- 메모리 누수 방지

**싱글톤 패턴:**
- 전역 캐시 인스턴스
- 모든 URL Fetcher 호출 간 공유

### 3. 캐시 사용 흐름

```typescript
// 1. 캐시 확인
const cachedContent = urlCache.get(normalizedUrl)
if (cachedContent) {
  content = cachedContent
  fromCache = true
} else {
  // 2. HTTP 요청
  const html = await response.text()
  content = convertHtmlToMarkdown(html)

  // 3. 캐시 저장
  urlCache.set(normalizedUrl, content)
  fromCache = false
}
```

**캐시 적중 시:**
- HTTP 요청 생략
- 즉시 콘텐츠 반환
- `fromCache: true` 표시

**캐시 미스 시:**
- HTTP 요청 수행
- 변환 후 캐시 저장
- `fromCache: false` 표시

### 4. 캐시 이점

**성능:**
- 반복 요청 시 네트워크 비용 절감
- 응답 시간 단축

**비용:**
- AI 분석은 항상 수행 (캐시 안 함)
- 웹 페이지 다운로드만 캐시

**안정성:**
- 동일 URL 반복 요청 시 서버 부하 감소
- 일시적 네트워크 오류 완화

---

## 아키텍처 패턴 및 설계

### 1. 공통 Tool 인터페이스

모든 도구는 동일한 인터페이스를 구현합니다:

```typescript
interface Tool<TSchema extends ZodSchema, TOutput> {
  name: string
  description: () => Promise<string> | string
  userFacingName: () => string
  inputSchema: TSchema
  isReadOnly: () => boolean
  isConcurrencySafe: () => boolean
  isEnabled: () => Promise<boolean>
  needsPermissions: () => boolean
  prompt: () => Promise<string> | string
  renderToolUseMessage: (input: z.infer<TSchema>) => string | React.ReactElement
  renderToolUseRejectedMessage: () => React.ReactElement
  renderToolResultMessage: (output: TOutput) => React.ReactElement
  renderResultForAssistant: (output: TOutput) => string
  call: (
    input: z.infer<TSchema>,
    context: ToolUseContext
  ) => AsyncGenerator<ToolResult<TOutput>, void, unknown>
}
```

**주요 메서드:**

1. **메타데이터**
   - `name`: 도구 고유 식별자
   - `description`: 도구 설명
   - `userFacingName`: 사용자 표시 이름

2. **스키마**
   - `inputSchema`: Zod 스키마 검증
   - `isReadOnly`: 읽기 전용 여부
   - `isConcurrencySafe`: 동시 실행 가능 여부

3. **실행**
   - `call`: 도구 실행 (Generator 패턴)
   - `needsPermissions`: 권한 필요 여부

4. **UI 렌더링**
   - `renderToolUseMessage`: 도구 사용 메시지
   - `renderToolResultMessage`: 결과 메시지
   - `renderResultForAssistant`: AI에게 전달할 결과

### 2. Generator 패턴

모든 도구는 비동기 Generator를 사용합니다:

```typescript
async *call(input: Input, context: ToolUseContext) {
  // 작업 수행
  const result = await performWork(input)

  // 결과 yield
  yield {
    type: 'result' as const,
    data: result,
    resultForAssistant: this.renderResultForAssistant(result),
  }
}
```

**Generator 이점:**
1. **스트리밍**: 중간 결과 전송 가능
2. **취소**: 작업 중단 가능
3. **진행 상태**: 여러 단계 표시 가능

### 3. React/Ink UI 통합

UI는 React 컴포넌트로 렌더링됩니다:

```typescript
renderToolResultMessage(output: Output) {
  return (
    <Box justifyContent="space-between" width="100%">
      <Box flexDirection="row">
        <Text>&nbsp;&nbsp;⎿ &nbsp;Found </Text>
        <Text bold>{output.results.length} </Text>
        <Text>results</Text>
      </Box>
      <Cost costUSD={0} durationMs={output.durationMs} debug={false} />
    </Box>
  )
}
```

**Ink 컴포넌트:**
- `Box`: 레이아웃 컨테이너
- `Text`: 텍스트 렌더링
- `Cost`: 비용/시간 표시

### 4. 타입 안전성

**Zod 스키마 검증:**
```typescript
const inputSchema = z.strictObject({
  url: z.string().url().describe('The URL to fetch content from'),
  prompt: z.string().describe('The prompt to run on the fetched content'),
})

type Input = z.infer<typeof inputSchema>
```

**타입 추론:**
- Zod 스키마에서 TypeScript 타입 자동 생성
- 런타임 검증과 컴파일 타임 타입 체크 동시 제공

### 5. 에러 처리

**일관된 에러 응답:**
```typescript
try {
  // 작업 수행
} catch (error: any) {
  yield {
    type: 'result' as const,
    resultForAssistant: `Error: ${error.message}`,
    data: emptyOutput,
  }
}
```

**에러 처리 원칙:**
1. 항상 결과 반환 (에러도 결과)
2. 사용자에게 명확한 메시지
3. AI에게도 에러 정보 전달

### 6. 비동기 처리 패턴

**병렬 처리:**
```typescript
return await Promise.all(
  Object.entries(allServers).map(async ([name, serverRef]) => {
    try {
      const client = await connectToServer(name, serverRef)
      return { name, client, type: 'connected' as const }
    } catch (error) {
      return { name, type: 'failed' as const }
    }
  }),
)
```

**타임아웃:**
```typescript
const abortController = new AbortController()
const timeout = setTimeout(() => abortController.abort(), 30000)

const response = await fetch(url, {
  signal: abortController.signal,
})

clearTimeout(timeout)
```

**Promise.race:**
```typescript
await Promise.race([connectPromise, timeoutPromise])
```

### 7. Memoization 전략

```typescript
export const getClients = memoize(async (): Promise<WrappedClient[]> => {
  // 한 번만 실행되고 결과 캐시
})

export const getMCPTools = memoize(async (): Promise<Tool[]> => {
  // 도구 목록 캐시
})
```

**Memoization 이점:**
- 반복 호출 방지
- 성능 향상
- 일관된 결과

### 8. 설정 계층 구조

```typescript
// 우선순위: project > mcprc > global
const allServers = {
  ...globalServers,
  ...approvedMcprcServers,
  ...projectServers,
}
```

**설정 병합 전략:**
1. Global: 기본 설정
2. MCPRC: 프로젝트 공유 설정
3. Project: 로컬 오버라이드

---

## 정리

### MCP 통합 시스템의 핵심 특징

1. **확장성**
   - 외부 도구 동적 로드
   - 플러그인 아키텍처
   - 표준 프로토콜 (MCP)

2. **보안**
   - .mcprc 서버 승인 시스템
   - 권한 기반 도구 실행
   - 사용자 명시적 승인

3. **성능**
   - Memoization 캐싱
   - 병렬 연결 처리
   - URL 캐싱 (15분)

4. **안정성**
   - 타임아웃 처리
   - 에러 복구
   - 우아한 실패 (graceful failure)

5. **사용성**
   - React/Ink UI
   - 명확한 상태 표시
   - AI 친화적 결과 포맷

### 도구별 역할

- **MCP Tool**: 외부 도구 동적 통합
- **URL Fetcher**: 웹 페이지 분석
- **Web Search**: 최신 정보 검색
- **HTML to Markdown**: 콘텐츠 정규화
- **Cache**: 성능 최적화

### 설계 원칙

1. **타입 안전성**: TypeScript + Zod
2. **반응형 UI**: React/Ink
3. **비동기 처리**: Generator + Promise
4. **모듈화**: 독립적 도구 구현
5. **확장성**: 플러그인 아키텍처

이러한 MCP 통합 시스템은 Kode-CLI가 외부 도구 및 서비스와 원활하게 통합되어 AI 모델의 능력을 확장할 수 있도록 합니다.
