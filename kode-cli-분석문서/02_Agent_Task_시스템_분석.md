# Kode-CLI Agent 및 Task 시스템 심층 분석

## 목차
1. [개요](#개요)
2. [Task 도구의 SubAgent 생성 메커니즘](#task-도구의-subagent-생성-메커니즘)
3. [다중 Agent 아키텍처 구현 방식](#다중-agent-아키텍처-구현-방식)
4. [Architect 도구의 설계 및 분석 기능](#architect-도구의-설계-및-분석-기능)
5. [Think 도구의 추론 메커니즘](#think-도구의-추론-메커니즘)
6. [Agent 간 통신 및 결과 합성 방식](#agent-간-통신-및-결과-합성-방식)
7. [프롬프트 설계 전략](#프롬프트-설계-전략)
8. [병렬 실행 및 동시성 제어](#병렬-실행-및-동시성-제어)
9. [아키텍처 다이어그램](#아키텍처-다이어그램)
10. [실전 활용 예제](#실전-활용-예제)

---

## 개요

Kode-CLI는 **다중 Agent 시스템**을 통해 복잡한 작업을 효율적으로 처리합니다. 이 시스템의 핵심은 세 가지 특수 도구입니다:

- **TaskTool**: 새로운 서브 Agent를 생성하고 독립적인 작업을 위임
- **ArchitectTool**: 기술 요구사항을 분석하고 구현 계획을 수립
- **ThinkTool**: 복잡한 추론과 브레인스토밍을 수행

이 문서는 각 도구의 내부 메커니즘과 상호작용 방식을 상세히 분석합니다.

---

## Task 도구의 SubAgent 생성 메커니즘

### 1.1 입력 스키마 정의

TaskTool은 Zod 스키마를 사용하여 입력 매개변수를 정의합니다:

```typescript
const inputSchema = z.object({
  description: z
    .string()
    .describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  model_name: z
    .string()
    .optional()
    .describe(
      'Optional: Specific model name to use for this task. If not provided, uses the default task model pointer.',
    ),
  subagent_type: z
    .string()
    .optional()
    .describe(
      'The type of specialized agent to use for this task',
    ),
})
```

**주요 파라미터:**
- `description`: 작업의 간단한 설명 (3-5단어)
- `prompt`: Agent가 수행할 실제 작업 내용
- `model_name` (선택): 특정 모델 지정 (예: "claude-opus-4", "gpt-4")
- `subagent_type` (선택): 특수화된 Agent 타입 (예: "code-reviewer", "general-purpose")

### 1.2 동적 Agent 설정 로딩

TaskTool은 `subagent_type`에 따라 동적으로 Agent 설정을 로드합니다:

```typescript
// 기본값은 general-purpose
const agentType = subagent_type || 'general-purpose'

// 효과적인 설정 초기화
let effectivePrompt = prompt
let effectiveModel = model_name || 'task'
let toolFilter = null
let temperature = undefined

// Agent 설정을 동적으로 로드
if (agentType) {
  const agentConfig = await getAgentByType(agentType)

  if (!agentConfig) {
    // Agent 타입을 찾을 수 없는 경우 도움말 메시지 반환
    const availableTypes = await getAvailableAgentTypes()
    const helpMessage = `Agent type '${agentType}' not found.\n\nAvailable agents:\n${availableTypes.map(t => `  • ${t}`).join('\n')}\n\nUse /agents command to manage agent configurations.`

    yield {
      type: 'result',
      data: [{ type: 'text', text: helpMessage }] as TextBlock[],
      resultForAssistant: helpMessage,
    }
    return
  }

  // 시스템 프롬프트 적용
  if (agentConfig.systemPrompt) {
    effectivePrompt = `${agentConfig.systemPrompt}\n\n${prompt}`
  }

  // 모델 적용 (model_name으로 재정의되지 않은 경우)
  if (!model_name && agentConfig.model_name) {
    if (agentConfig.model_name !== 'inherit') {
      effectiveModel = agentConfig.model_name as string
    }
  }

  // 도구 필터 저장
  toolFilter = agentConfig.tools
}
```

**핵심 동작:**
1. Agent 타입이 존재하지 않으면 사용 가능한 Agent 목록을 반환
2. Agent 설정에서 시스템 프롬프트를 추출하여 사용자 프롬프트 앞에 추가
3. 모델 이름이 지정되지 않았고 Agent 설정에 모델이 있으면 해당 모델 사용
4. 도구 필터를 저장하여 나중에 적용

### 1.3 도구 필터링 메커니즘

Agent는 사용할 수 있는 도구를 제한할 수 있습니다:

```typescript
let tools = await getTaskTools(safeMode)

// Agent 설정에 따른 도구 필터링
if (toolFilter) {
  // 하위 호환성: ['*']는 모든 도구를 의미
  const isAllArray = Array.isArray(toolFilter) && toolFilter.length === 1 && toolFilter[0] === '*'
  if (toolFilter === '*' || isAllArray) {
    // no-op, 모든 도구 유지
  } else if (Array.isArray(toolFilter)) {
    tools = tools.filter(tool => toolFilter.includes(tool.name))
  }
}
```

**도구 필터 옵션:**
- `"*"` 또는 `["*"]`: 모든 도구 사용 가능
- `["FileRead", "Bash", "Grep"]`: 지정된 도구만 사용 가능
- 빈 배열 `[]`: 도구 없음 (순수 추론만)

### 1.4 SubAgent 실행 및 스트리밍

TaskTool은 AsyncGenerator를 사용하여 실시간으로 진행 상황을 스트리밍합니다:

```typescript
// 초기 작업 정보 표시
yield {
  type: 'progress',
  content: createAssistantMessage(`Starting agent: ${agentType}`),
  normalizedMessages: normalizeMessages(messages),
  tools,
}

yield {
  type: 'progress',
  content: createAssistantMessage(`Using model: ${modelToUse}`),
  normalizedMessages: normalizeMessages(messages),
  tools,
}

yield {
  type: 'progress',
  content: createAssistantMessage(`Task: ${description}`),
  normalizedMessages: normalizeMessages(messages),
  tools,
}

// Agent 프롬프트 및 컨텍스트 준비
const [taskPrompt, context, maxThinkingTokens] = await Promise.all([
  getAgentPrompt(),
  getContext(),
  getMaxThinkingTokens(messages),
])

// 자기 참조 방지를 위한 모델 컨텍스트 주입
taskPrompt.push(`\nIMPORTANT: You are currently running as ${modelToUse}. You do not need to consult ${modelToUse} via AskExpertModel since you ARE ${modelToUse}. Complete tasks directly using your capabilities.`)

// 쿼리 옵션 구성
const queryOptions = {
  safeMode,
  forkNumber,
  messageLogName,
  tools,
  commands: [],
  verbose,
  maxThinkingTokens,
  model: modelToUse,
}

// Agent 실행
for await (const message of query(
  messages,
  taskPrompt,
  context,
  hasPermissionsToUseTool,
  {
    abortController,
    options: queryOptions,
    messageId: getLastAssistantMessageId(messages),
    agentId: taskId,
    readFileTimestamps,
    setToolJSX: () => {}, // No-op implementation for TaskTool
  },
)) {
  messages.push(message)

  // 메시지 로그 저장
  overwriteLog(
    getMessagesPath(messageLogName, forkNumber, getSidechainNumber()),
    messages.filter(_ => _.type !== 'progress'),
  )

  // 도구 사용 및 텍스트 콘텐츠 처리
  for (const content of message.message.content) {
    if (content.type === 'text' && content.text && content.text !== INTERRUPT_MESSAGE) {
      // Agent의 응답 표시
      const preview = content.text.length > 200 ? content.text.substring(0, 200) + '...' : content.text
      yield {
        type: 'progress',
        content: createAssistantMessage(`${preview}`),
        normalizedMessages,
        tools,
      }
    } else if (content.type === 'tool_use') {
      toolUseCount++
      // 도구 사용 정보 표시
      yield {
        type: 'progress',
        content: modifiedMessage,
        normalizedMessages,
        tools,
      }
    }
  }
}
```

**스트리밍 이점:**
1. 실시간 진행 상황 모니터링
2. 긴 작업 중 사용자 피드백
3. 조기 중단 가능
4. 도구 사용 가시성 향상

### 1.5 결과 반환 및 요약

SubAgent 실행이 완료되면 결과를 주 Agent에게 반환합니다:

```typescript
const lastMessage = last(messages)
if (lastMessage?.type !== 'assistant') {
  throw new Error('Last message was not an assistant message')
}

// 중단 처리
if (
  lastMessage.message.content.some(
    _ => _.type === 'text' && _.text === INTERRUPT_MESSAGE,
  )
) {
  // 중단된 경우 진행 상황 yield 건너뛰기
} else {
  // 완료 통계 계산
  const result = [
    toolUseCount === 1 ? '1 tool use' : `${toolUseCount} tool uses`,
    formatNumber(
      (lastMessage.message.usage.cache_creation_input_tokens ?? 0) +
        (lastMessage.message.usage.cache_read_input_tokens ?? 0) +
        lastMessage.message.usage.input_tokens +
        lastMessage.message.usage.output_tokens,
    ) + ' tokens',
    formatDuration(Date.now() - startTime),
  ]
  yield {
    type: 'progress',
    content: createAssistantMessage(`Task completed (${result.join(' · ')})`),
    normalizedMessages,
    tools,
  }
}

// 텍스트 블록 추출 및 반환
const data = lastMessage.message.content.filter(_ => _.type === 'text')
yield {
  type: 'result',
  data,
  resultForAssistant: this.renderResultForAssistant(data),
}
```

**결과 구조:**
- `type: 'result'`: 최종 결과 타입
- `data`: TextBlock[] 형태의 원시 데이터
- `resultForAssistant`: AI가 읽을 수 있는 텍스트 형태로 변환된 결과

---

## 다중 Agent 아키텍처 구현 방식

### 2.1 Agent 로딩 시스템 개요

Kode-CLI의 Agent 시스템은 **5-tier 우선순위 계층**을 통해 Agent를 로드합니다:

```
1. Built-in (코드 내장)
2. ~/.claude/agents/ (Claude Code 사용자 디렉토리 호환성)
3. ~/.kode/agents/ (Kode 사용자 디렉토리)
4. ./.claude/agents/ (Claude Code 프로젝트 디렉토리 호환성)
5. ./.kode/agents/ (Kode 프로젝트 디렉토리)
```

**우선순위 규칙:**
- 하위 번호가 높은 우선순위
- 동일한 이름의 Agent가 여러 위치에 있으면 우선순위가 높은 것을 사용
- 프로젝트 레벨 Agent가 전역 Agent를 재정의 가능

### 2.2 Agent 정의 형식

Agent는 마크다운 파일로 정의되며 YAML frontmatter를 사용합니다:

```markdown
---
name: code-reviewer
description: "코드 리뷰를 수행하고 개선 사항을 제안합니다"
tools: ["FileRead", "Grep", "Glob"]
model: claude-opus-4
---

당신은 숙련된 코드 리뷰어입니다. 다음 기준에 따라 코드를 검토하세요:

1. **코드 품질**: 가독성, 유지보수성, 일관성
2. **버그 및 오류**: 잠재적 버그, 엣지 케이스, 오류 처리
3. **성능**: 비효율적인 알고리즘, 메모리 누수
4. **보안**: 취약점, 입력 검증, 인증/권한
5. **베스트 프랙티스**: 디자인 패턴, SOLID 원칙

각 발견 사항에 대해:
- **심각도**: Critical/High/Medium/Low
- **설명**: 무엇이 문제인지
- **제안**: 어떻게 개선할 수 있는지
- **예시 코드**: 가능한 경우 개선된 코드 제공
```

**YAML Frontmatter 필드:**
- `name` (필수): Agent의 고유 식별자
- `description` (필수): 언제 사용해야 하는지 설명
- `tools` (선택): 사용 가능한 도구 목록 또는 `"*"` (모든 도구)
- `model` (선택): 특정 모델 지정 또는 `"inherit"` (기본값 사용)

### 2.3 동적 Agent 로딩 구현

`getAgentByType` 함수는 Agent 타입에 따라 설정을 로드합니다:

```typescript
export async function getAgentByType(agentType: string): Promise<AgentConfig | null> {
  const agents = await getActiveAgents()
  const agent = agents.find(a => a.agentType === agentType)

  if (!agent) {
    return null
  }

  return {
    systemPrompt: agent.systemPrompt,
    model_name: agent.model || 'inherit',
    tools: agent.tools || '*',
  }
}

export async function getAvailableAgentTypes(): Promise<string[]> {
  const agents = await getActiveAgents()
  return agents.map(a => a.agentType)
}
```

**로딩 프로세스:**
1. 모든 Agent 디렉토리 스캔
2. 마크다운 파일 파싱 (YAML frontmatter + body)
3. 우선순위에 따라 병합
4. 메모이제이션을 통한 캐싱

### 2.4 프롬프트 생성 및 Agent 설명

TaskTool의 프롬프트는 사용 가능한 Agent 목록을 동적으로 생성합니다:

```typescript
export async function getPrompt(safeMode: boolean): Promise<string> {
  const agents = await getActiveAgents()

  // Agent 설명 포맷: (Tools: tool1, tool2)
  const agentDescriptions = agents.map(agent => {
    const toolsStr = Array.isArray(agent.tools)
      ? agent.tools.join(', ')
      : '*'
    return `- ${agent.agentType}: ${agent.whenToUse} (Tools: ${toolsStr})`
  }).join('\n')

  return `Launch a new agent to handle complex, multi-step tasks autonomously.

Available agent types and the tools they have access to:
${agentDescriptions}

When using the Task tool, you must specify a subagent_type parameter to select which agent type to use.

When to use the Agent tool:
- When you are instructed to execute custom slash commands. Use the Agent tool with the slash command invocation as the entire prompt.

When NOT to use the Agent tool:
- If you want to read a specific file path, use the FileRead or Glob tool instead
- If you are searching for a specific class definition, use the Glob tool instead
- If you are searching for code within a specific file or set of 2-3 files, use the FileRead tool instead

Usage notes:
1. Launch multiple agents concurrently whenever possible, to maximize performance
2. When the agent is done, it will return a single message back to you
3. Each agent invocation is stateless
4. Your prompt should contain a highly detailed task description
5. Clearly tell the agent whether you expect it to write code or just to do research
6. If the agent description mentions that it should be used proactively, then you should use it without the user having to ask`
}
```

**프롬프트 설계 원칙:**
1. 명확한 Agent 선택 가이드
2. 언제 사용해야/하지 말아야 하는지 명시
3. 병렬 실행 권장
4. 상태 비저장 특성 강조
5. 상세한 작업 설명 필요성 강조

### 2.5 입력 검증

TaskTool은 Agent 타입과 모델 이름을 검증합니다:

```typescript
async validateInput(input, context) {
  if (!input.description || typeof input.description !== 'string') {
    return {
      result: false,
      message: 'Description is required and must be a string',
    }
  }
  if (!input.prompt || typeof input.prompt !== 'string') {
    return {
      result: false,
      message: 'Prompt is required and must be a string',
    }
  }

  // 모델 검증
  if (input.model_name) {
    const modelManager = getModelManager()
    const availableModels = modelManager.getAllAvailableModelNames()

    if (!availableModels.includes(input.model_name)) {
      return {
        result: false,
        message: `Model '${input.model_name}' does not exist. Available models: ${availableModels.join(', ')}`,
        meta: {
          model_name: input.model_name,
          availableModels,
        },
      }
    }
  }

  // Agent 타입 검증
  if (input.subagent_type) {
    const availableTypes = await getAvailableAgentTypes()
    if (!availableTypes.includes(input.subagent_type)) {
      return {
        result: false,
        message: `Agent type '${input.subagent_type}' does not exist. Available types: ${availableTypes.join(', ')}`,
        meta: {
          subagent_type: input.subagent_type,
          availableTypes,
        },
      }
    }
  }

  return { result: true }
}
```

**검증 단계:**
1. 필수 필드 검증 (`description`, `prompt`)
2. 모델 이름 존재 여부 확인
3. Agent 타입 존재 여부 확인
4. 실패 시 사용 가능한 옵션 제공

---

## Architect 도구의 설계 및 분석 기능

### 3.1 Architect 도구 개요

ArchitectTool은 **기술 요구사항을 분석하고 구현 계획을 수립**하는 특수 도구입니다. 주요 특징:

- **읽기 전용**: 코드를 작성하지 않고 계획만 생성
- **제한된 도구**: 파일 시스템 탐색 도구만 사용 가능
- **전문가 역할**: 시니어 소프트웨어 아키텍트로서 동작

### 3.2 입력 스키마

```typescript
const inputSchema = z.strictObject({
  prompt: z
    .string()
    .describe('The technical request or coding task to analyze'),
  context: z
    .string()
    .describe('Optional context from previous conversation or system state')
    .optional(),
})
```

**파라미터:**
- `prompt`: 분석할 기술 요청 또는 코딩 작업
- `context` (선택): 이전 대화나 시스템 상태의 컨텍스트

### 3.3 시스템 프롬프트

ArchitectTool의 핵심은 잘 설계된 시스템 프롬프트입니다:

```typescript
export const ARCHITECT_SYSTEM_PROMPT = `You are an expert software architect. Your role is to analyze technical requirements and produce clear, actionable implementation plans.
These plans will then be carried out by a junior software engineer so you need to be specific and detailed. However do not actually write the code, just explain the plan.

Follow these steps for each request:
1. Carefully analyze requirements to identify core functionality and constraints
2. Define clear technical approach with specific technologies and patterns
3. Break down implementation into concrete, actionable steps at the appropriate level of abstraction

Keep responses focused, specific and actionable.

IMPORTANT: Do not ask the user if you should implement the changes at the end. Just provide the plan as described above.
IMPORTANT: Do not attempt to write the code or use any string modification tools. Just provide the plan.`
```

**프롬프트 설계 원칙:**
1. **역할 정의**: 전문 소프트웨어 아키텍트
2. **명확한 제약**: 코드 작성 금지, 계획만 생성
3. **구조화된 접근**: 3단계 분석 프로세스
4. **대상 청중**: 주니어 엔지니어가 실행할 수 있을 만큼 상세
5. **행동 제한**: 사용자에게 질문하지 않고 바로 계획 제공

### 3.4 도구 필터링

ArchitectTool은 파일 시스템 탐색 도구만 사용합니다:

```typescript
const FS_EXPLORATION_TOOLS: Tool[] = [
  BashTool,      // 명령 실행 (ls, find 등)
  LSTool,        // 디렉토리 목록
  FileReadTool,  // 파일 읽기
  FileWriteTool, // 파일 쓰기 (계획 문서 작성 용도)
  GlobTool,      // 파일 패턴 매칭
  GrepTool,      // 코드 검색
]

async *call({ prompt, context }, toolUseContext) {
  // 파일 탐색 도구만 허용
  const allowedTools = (toolUseContext.options?.tools ?? []).filter(_ =>
    FS_EXPLORATION_TOOLS.map(_ => _.name).includes(_.name),
  )

  // 더미 canUseTool 함수 (도구 사용 제어)
  const canUseTool = async () => ({ result: true as const })

  // 쿼리 실행
  const lastResponse = await lastX(
    query(
      messages,
      [ARCHITECT_SYSTEM_PROMPT],
      await getContext(),
      canUseTool,
      {
        ...toolUseContext,
        setToolJSX: () => {},
        options: {
          ...toolUseContext.options,
          tools: allowedTools
        },
      },
    ),
  )
}
```

**도구 제한 이유:**
1. 코드 작성 방지 (FileEdit, NotebookEdit 제외)
2. 코드베이스 탐색 및 분석에 집중
3. 실수로 코드 수정 방지
4. 계획 수립에 필요한 정보 수집만 허용

### 3.5 실행 흐름

```typescript
async *call({ prompt, context }, toolUseContext) {
  // 컨텍스트가 있으면 프롬프트에 추가
  const content = context
    ? `<context>${context}</context>\n\n${prompt}`
    : prompt

  const userMessage = createUserMessage(content)
  const messages: Message[] = [userMessage]

  // 파일 탐색 도구만 필터링
  const allowedTools = (toolUseContext.options?.tools ?? []).filter(_ =>
    FS_EXPLORATION_TOOLS.map(_ => _.name).includes(_.name),
  )

  // 쿼리 실행 (마지막 응답만 가져옴)
  const lastResponse = await lastX(
    query(
      messages,
      [ARCHITECT_SYSTEM_PROMPT],
      await getContext(),
      canUseTool,
      {
        ...toolUseContext,
        setToolJSX: () => {},
        options: {
          ...toolUseContext.options,
          tools: allowedTools
        },
      },
    ),
  )

  if (lastResponse.type !== 'assistant') {
    throw new Error(`Invalid response from API`)
  }

  // 텍스트 블록 추출 및 반환
  const data = lastResponse.message.content.filter(_ => _.type === 'text')
  yield {
    type: 'result',
    data,
    resultForAssistant: this.renderResultForAssistant(data),
  }
}
```

**주요 특징:**
1. **단일 쿼리**: 중간 진행 상황 없이 최종 결과만 반환
2. **컨텍스트 주입**: 이전 대화 컨텍스트를 XML 태그로 구분
3. **동기적 실행**: `lastX`를 사용하여 모든 메시지를 소비하고 마지막만 반환
4. **간단한 결과**: 텍스트 블록만 추출

### 3.6 활용 사례

**예제 1: 새로운 기능 설계**
```typescript
// 입력
{
  prompt: "사용자 인증 시스템을 추가하고 싶습니다. JWT 기반으로 구현해주세요.",
  context: "Express.js 기반 REST API, PostgreSQL 데이터베이스 사용 중"
}

// 출력 (예상)
/**
 * # 사용자 인증 시스템 구현 계획
 *
 * ## 1. 요구사항 분석
 * - JWT 기반 토큰 인증
 * - Express.js 미들웨어 통합
 * - PostgreSQL에 사용자 정보 저장
 * - 비밀번호 해싱 (bcrypt)
 *
 * ## 2. 기술 스택
 * - jsonwebtoken: JWT 생성/검증
 * - bcryptjs: 비밀번호 해싱
 * - express-validator: 입력 검증
 *
 * ## 3. 구현 단계
 *
 * ### 3.1 데이터베이스 스키마
 * - users 테이블 생성
 *   - id (UUID, PK)
 *   - email (VARCHAR, UNIQUE)
 *   - password_hash (VARCHAR)
 *   - created_at (TIMESTAMP)
 *
 * ### 3.2 인증 미들웨어
 * - src/middleware/auth.js 생성
 * - JWT 토큰 검증 로직
 * - Request 객체에 사용자 정보 주입
 *
 * ### 3.3 인증 라우트
 * - POST /api/auth/register: 회원가입
 * - POST /api/auth/login: 로그인
 * - POST /api/auth/refresh: 토큰 갱신
 *
 * ### 3.4 보안 고려사항
 * - 비밀번호 최소 8자, 특수문자 포함
 * - Rate limiting (express-rate-limit)
 * - HTTPS 강제
 * - 리프레시 토큰 별도 관리
 */
```

**예제 2: 버그 분석**
```typescript
// 입력
{
  prompt: "메모리 누수가 발생하는 것 같습니다. 어디를 확인해야 할까요?",
  context: "Node.js 애플리케이션, 장시간 실행 시 메모리 사용량 증가"
}

// 출력 (예상)
/**
 * # 메모리 누수 분석 계획
 *
 * ## 1. 의심 지점 식별
 * - 이벤트 리스너 누적
 * - 타이머/인터벌 정리 미흡
 * - 캐시 무제한 증가
 * - 클로저에 의한 참조 유지
 *
 * ## 2. 분석 단계
 *
 * ### 2.1 파일 검토 우선순위
 * 1. EventEmitter 사용 코드 (on/once 패턴)
 * 2. setInterval/setTimeout 사용
 * 3. 전역 캐시/맵 객체
 * 4. 데이터베이스 연결 풀
 *
 * ### 2.2 도구 활용
 * - Grep으로 'new EventEmitter' 검색
 * - Grep으로 'setInterval' 검색
 * - Glob으로 '*Cache*.js' 파일 찾기
 * - FileRead로 의심 파일 검토
 *
 * ### 2.3 확인 사항
 * - removeListener 호출 여부
 * - clearInterval/clearTimeout 호출 여부
 * - WeakMap 사용 가능성
 * - 주기적인 캐시 정리 로직
 */
```

---

## Think 도구의 추론 메커니즘

### 4.1 Think 도구 개요

ThinkTool은 **순수 추론 도구**로서 실제 작업을 수행하지 않고 사고 과정을 로깅합니다:

- **No-op 도구**: 실제 동작 없음
- **투명성 향상**: AI의 사고 과정을 사용자에게 노출
- **복잡한 추론 지원**: 브레인스토밍, 계획 수립, 가설 검증

### 4.2 입력 스키마

```typescript
const thinkToolSchema = z.object({
  thought: z.string().describe('Your thoughts.'),
})
```

**파라미터:**
- `thought`: AI의 생각이나 추론 과정

### 4.3 시스템 프롬프트

```typescript
export const PROMPT = `Use the tool to think about something. It will not obtain new information or make any changes to the repository, but just log the thought. Use it when complex reasoning or brainstorming is needed.

Common use cases:
1. When exploring a repository and discovering the source of a bug, call this tool to brainstorm several unique ways of fixing the bug, and assess which change(s) are likely to be simplest and most effective
2. After receiving test results, use this tool to brainstorm ways to fix failing tests
3. When planning a complex refactoring, use this tool to outline different approaches and their tradeoffs
4. When designing a new feature, use this tool to think through architecture decisions and implementation details
5. When debugging a complex issue, use this tool to organize your thoughts and hypotheses

The tool simply logs your thought process for better transparency and does not execute any code or make changes.`
```

**활용 시나리오:**
1. **버그 수정 브레인스토밍**: 여러 수정 방법 평가
2. **테스트 실패 분석**: 실패 원인 및 해결 방법 탐색
3. **리팩토링 계획**: 다양한 접근 방식과 트레이드오프 검토
4. **기능 설계**: 아키텍처 결정 및 구현 세부사항 검토
5. **디버깅**: 가설 정리 및 검증 계획 수립

### 4.4 구현

ThinkTool은 매우 간단하게 구현되어 있습니다:

```typescript
export const ThinkTool = {
  name: 'Think',
  userFacingName: () => 'Think',
  description: async () => DESCRIPTION,
  inputSchema: thinkToolSchema,
  isEnabled: async () => Boolean(process.env.THINK_TOOL), // 환경 변수로 활성화
  isReadOnly: () => true,  // 읽기 전용
  isConcurrencySafe: () => true,  // 동시 실행 안전
  needsPermissions: () => false,  // 권한 불필요
  prompt: async () => PROMPT,

  async *call(input, { messageId }) {
    // 즉시 결과 반환 (실제 작업 없음)
    yield {
      type: 'result',
      resultForAssistant: 'Your thought has been logged.',
      data: { thought: input.thought },
    }
  },

  renderToolUseMessage(input) {
    return input.thought
  },

  renderToolUseRejectedMessage() {
    return (
      <MessageResponse children={<Text color={getTheme().error}>Thought cancelled</Text>} />
    )
  },

  renderResultForAssistant: () => 'Your thought has been logged.',
}
```

**핵심 특징:**
1. **환경 변수 제어**: `THINK_TOOL=1`로 활성화
2. **즉시 반환**: 실제 처리 없이 바로 결과 반환
3. **사고 로깅**: UI에 사고 과정 표시
4. **간단한 응답**: "Your thought has been logged." 메시지

### 4.5 사용 예제

**예제 1: 버그 수정 전략 수립**
```typescript
// AI가 Think 도구 호출
{
  thought: `
  버그 수정을 위한 세 가지 접근 방법을 고려 중:

  1. **Quick Fix (난이도: 낮음, 위험: 중간)**
     - 증상: API 응답이 null일 때 크래시
     - 방법: null 체크 추가
     - 장점: 빠른 구현 (5분)
     - 단점: 근본 원인 미해결, 다른 곳에서 재발 가능

  2. **Defensive Programming (난이도: 중간, 위험: 낮음)**
     - 방법: 모든 API 호출에 에러 핸들링 추가
     - 장점: 견고성 향상, 재발 방지
     - 단점: 여러 파일 수정 필요 (30분)

  3. **Root Cause Fix (난이도: 높음, 위험: 낮음)**
     - 조사: API가 왜 null을 반환하는지 확인
     - 방법: API 계약 수정 또는 기본값 반환
     - 장점: 완전한 해결
     - 단점: 백엔드 팀 협업 필요 (2-3일)

  **권장 사항**:
  - 단기: 방법 2 (Defensive Programming) - 안정성과 속도의 균형
  - 장기: 방법 3 추진 - 백엔드 팀과 협의
  `
}
```

**예제 2: 리팩토링 트레이드오프 분석**
```typescript
// AI가 Think 도구 호출
{
  thought: `
  상태 관리 리팩토링 옵션 분석:

  **현재 상태**:
  - Redux 사용 중
  - 보일러플레이트 코드 과다
  - 학습 곡선 가파름

  **옵션 A: Zustand로 마이그레이션**
  - 장점:
    ✓ 단순한 API
    ✓ 보일러플레이트 최소화
    ✓ TypeScript 지원 우수
  - 단점:
    ✗ 팀원 학습 필요
    ✗ 미들웨어 생태계 작음
    ✗ 마이그레이션 비용 (3-4주)

  **옵션 B: Redux Toolkit 도입**
  - 장점:
    ✓ 기존 Redux 지식 활용
    ✓ 보일러플레이트 80% 감소
    ✓ 점진적 마이그레이션 가능
  - 단점:
    ✗ 여전히 Redux의 복잡성 존재
    ✗ 새로운 패턴 학습 필요

  **옵션 C: 현상 유지 + 헬퍼 함수**
  - 장점:
    ✓ 위험 없음
    ✓ 빠른 구현 (1주)
  - 단점:
    ✗ 근본적 개선 아님
    ✗ 기술 부채 누적

  **결론**:
  옵션 B (Redux Toolkit) 추천
  - 이유: 점진적 개선 + 팀 러닝 커브 최소화
  - 실행 계획: 신규 기능부터 적용, 4주 내 완료
  `
}
```

### 4.6 ThinkTool의 가치

ThinkTool은 다음과 같은 이점을 제공합니다:

1. **투명성**: AI의 의사결정 과정을 사용자에게 공개
2. **신뢰성**: 왜 특정 선택을 했는지 이해 가능
3. **검증 가능성**: 사용자가 AI의 추론을 검토하고 수정 가능
4. **학습**: AI의 문제 해결 접근 방식에서 배울 수 있음
5. **디버깅**: AI가 잘못된 가정을 하면 조기 발견 가능

---

## Agent 간 통신 및 결과 합성 방식

### 5.1 AsyncGenerator 기반 스트리밍

모든 Agent는 AsyncGenerator를 통해 결과를 스트리밍합니다:

```typescript
async *call(
  input,
  context,
): AsyncGenerator<
  | { type: 'result'; data: TextBlock[]; resultForAssistant?: string }
  | { type: 'progress'; content: any; normalizedMessages?: any[]; tools?: any[] },
  void,
  unknown
>
```

**메시지 타입:**

1. **Progress 메시지**: 중간 진행 상황
   ```typescript
   {
     type: 'progress',
     content: AssistantMessage,  // 진행 상황 메시지
     normalizedMessages: Message[],  // 현재까지의 대화 내역
     tools: Tool[]  // 사용 가능한 도구 목록
   }
   ```

2. **Result 메시지**: 최종 결과
   ```typescript
   {
     type: 'result',
     data: TextBlock[],  // 원시 결과 데이터
     resultForAssistant?: string  // AI가 읽을 수 있는 형태
   }
   ```

### 5.2 메시지 정규화

Agent 간 통신 시 메시지를 정규화합니다:

```typescript
const normalizedMessages = normalizeMessages(messages)

yield {
  type: 'progress',
  content: createAssistantMessage(`Processing...`),
  normalizedMessages,
  tools,
}
```

**정규화 목적:**
1. 일관된 메시지 형식 유지
2. UI 렌더링 최적화
3. 로그 저장 형식 표준화
4. API 호환성 보장

### 5.3 결과 직렬화

SubAgent의 결과는 주 Agent가 읽을 수 있는 형태로 변환됩니다:

```typescript
renderResultForAssistant(data: TextBlock[]): string {
  return data.map(block => block.type === 'text' ? block.text : '').join('\n')
}
```

**직렬화 과정:**
1. TextBlock 배열에서 텍스트만 추출
2. 여러 블록을 줄바꿈으로 연결
3. 주 Agent의 컨텍스트에 추가
4. 주 Agent가 결과를 분석하고 다음 작업 결정

### 5.4 도구 사용 추적

TaskTool은 SubAgent의 모든 도구 사용을 추적합니다:

```typescript
let toolUseCount = 0

for await (const message of query(...)) {
  messages.push(message)

  for (const content of message.message.content) {
    if (content.type === 'tool_use') {
      toolUseCount++

      // 도구 사용 정보를 progress로 스트리밍
      yield {
        type: 'progress',
        content: modifiedMessage,
        normalizedMessages,
        tools,
      }
    }
  }
}

// 최종 통계 계산
const result = [
  toolUseCount === 1 ? '1 tool use' : `${toolUseCount} tool uses`,
  formatNumber(totalTokens) + ' tokens',
  formatDuration(Date.now() - startTime),
]
```

**추적 정보:**
- 도구 사용 횟수
- 토큰 사용량 (입력/출력/캐시)
- 실행 시간

### 5.5 에러 처리 및 복구

Agent 실행 중 에러 발생 시:

```typescript
try {
  for await (const message of query(...)) {
    // Agent 실행
  }
} catch (error) {
  // 에러를 사용자 친화적 메시지로 변환
  yield {
    type: 'result',
    data: [{
      type: 'text',
      text: `Error: ${error.message}\n\nPlease try again with a different approach.`
    }],
    resultForAssistant: `Task failed: ${error.message}`,
  }
}
```

**에러 처리 전략:**
1. 에러를 캐치하여 중단 방지
2. 사용자 친화적 메시지 생성
3. 주 Agent에게 실패 정보 전달
4. 주 Agent가 대안 접근 방식 시도

### 5.6 중단 메커니즘

사용자가 Agent를 중단할 수 있습니다:

```typescript
const INTERRUPT_MESSAGE = '__INTERRUPT__'

// 중단 감지
if (
  lastMessage.message.content.some(
    _ => _.type === 'text' && _.text === INTERRUPT_MESSAGE,
  )
) {
  // 중단된 경우 처리
  yield {
    type: 'result',
    data: [{
      type: 'text',
      text: 'Task was interrupted by user'
    }],
    resultForAssistant: 'Task interrupted',
  }
  return
}
```

**중단 처리:**
1. 특수 메시지(`__INTERRUPT__`) 감지
2. 즉시 실행 중단
3. 부분 결과 반환 (가능한 경우)
4. 리소스 정리 (파일 핸들, 네트워크 연결 등)

### 5.7 병렬 Agent 실행

여러 Agent를 동시에 실행할 수 있습니다:

```typescript
// AI가 여러 Task를 동시에 실행하는 예제
// 실제로는 API 호출 시 여러 tool_use를 한 번에 보냄

// 사용자 요청: "파일 A, B, C를 각각 분석해주세요"

// AI 응답:
[
  {
    type: 'tool_use',
    id: 'task_1',
    name: 'Task',
    input: {
      description: 'Analyze file A',
      prompt: 'Read and analyze file A for potential issues',
      subagent_type: 'code-reviewer'
    }
  },
  {
    type: 'tool_use',
    id: 'task_2',
    name: 'Task',
    input: {
      description: 'Analyze file B',
      prompt: 'Read and analyze file B for potential issues',
      subagent_type: 'code-reviewer'
    }
  },
  {
    type: 'tool_use',
    id: 'task_3',
    name: 'Task',
    input: {
      description: 'Analyze file C',
      prompt: 'Read and analyze file C for potential issues',
      subagent_type: 'code-reviewer'
    }
  }
]

// 세 개의 Task가 병렬로 실행되고 각각 독립적으로 결과 반환
```

**병렬 실행 이점:**
1. **성능 향상**: 총 실행 시간 단축
2. **독립성**: 각 Agent가 독립적으로 작업
3. **확장성**: Agent 수에 제한 없음
4. **효율성**: I/O 대기 시간 활용

### 5.8 결과 합성

주 Agent는 여러 SubAgent의 결과를 합성합니다:

```typescript
// 세 개의 코드 리뷰 결과를 받은 후 주 Agent의 종합

// Task 1 결과:
"File A 분석 결과: 3개의 버그 발견, 5개의 개선 사항 제안"

// Task 2 결과:
"File B 분석 결과: 버그 없음, 2개의 개선 사항 제안"

// Task 3 결과:
"File C 분석 결과: 1개의 심각한 버그 발견, 3개의 개선 사항 제안"

// 주 Agent의 합성 응답:
/**
 * 전체 코드 리뷰 요약:
 *
 * ## 심각한 이슈
 * - File C: SQL Injection 취약점 (라인 45)
 *
 * ## 버그
 * - File A: Null pointer exception (라인 23)
 * - File A: Race condition (라인 67)
 * - File A: Memory leak (라인 89)
 *
 * ## 개선 사항 (우선순위 순)
 * 1. File A: 에러 핸들링 추가
 * 2. File C: 입력 검증 강화
 * 3. File B: 코드 중복 제거
 * ...
 *
 * ## 다음 단계
 * 1. File C의 SQL Injection 즉시 수정 (Critical)
 * 2. File A의 버그 수정 (High)
 * 3. 개선 사항 적용 (Medium)
 */
```

---

## 프롬프트 설계 전략

### 6.1 계층적 프롬프트 구조

Kode-CLI는 다층 프롬프트 시스템을 사용합니다:

```
1. 전역 Agent 프롬프트 (getAgentPrompt)
   └─ 모든 Agent에 공통 적용되는 기본 규칙

2. 도구별 프롬프트 (tool.prompt())
   └─ 각 도구의 사용 방법 및 제약사항

3. Agent 타입 프롬프트 (agentConfig.systemPrompt)
   └─ 특정 Agent 역할 및 전문성

4. 사용자 프롬프트 (input.prompt)
   └─ 구체적인 작업 지시
```

**프롬프트 병합 예제:**
```typescript
// 1. 전역 프롬프트
const globalPrompt = await getAgentPrompt()
// "You are an expert coding assistant. Follow best practices..."

// 2. 도구 프롬프트 (TaskTool)
const toolPrompt = await TaskTool.prompt()
// "Launch a new agent to handle complex tasks..."

// 3. Agent 타입 프롬프트 (code-reviewer)
const agentPrompt = agentConfig.systemPrompt
// "You are a code reviewer. Check for bugs, security issues..."

// 4. 사용자 프롬프트
const userPrompt = "Review the authentication module"

// 최종 프롬프트
const finalPrompt = `${agentPrompt}\n\n${userPrompt}`
// "You are a code reviewer. Check for bugs, security issues...
//
//  Review the authentication module"

// 전역 프롬프트는 시스템 레벨에서 적용됨
```

### 6.2 역할 기반 프롬프트 설계

각 Agent는 명확한 역할을 가집니다:

**Architect Agent:**
```typescript
`You are an expert software architect. Your role is to analyze technical requirements and produce clear, actionable implementation plans.
These plans will then be carried out by a junior software engineer so you need to be specific and detailed. However do not actually write the code, just explain the plan.`
```

**설계 원칙:**
- **명확한 역할 정의**: "expert software architect"
- **대상 청중**: "junior software engineer" → 상세한 설명 필요
- **행동 제약**: "do not actually write the code"
- **출력 형식**: "clear, actionable implementation plans"

### 6.3 Few-Shot 학습

프롬프트에 구체적인 예제를 포함합니다:

```typescript
export async function getPrompt(safeMode: boolean): Promise<string> {
  return `...

Example usage:

<example_agent_descriptions>
"code-reviewer": use this agent after you are done writing a significant piece of code
"greeting-responder": use this agent to respond to user greetings with a friendly joke
</example_agent_description>

<example>
user: "Please write a function that checks if a number is prime"
assistant: Sure let me write a function that checks if a number is prime
assistant: First let me use the FileWrite tool to write a function
assistant: I'm going to use the FileWrite tool to write the following code:
<code>
function isPrime(n) {
  if (n <= 1) return false
  for (let i = 2; i * i <= n; i++) {
    if (n % i === 0) return false
  }
  return true
}
</code>
<commentary>
Since a significant piece of code was written and the task was completed, now use the code-reviewer agent
</commentary>
assistant: Now let me use the code-reviewer agent to review the code
assistant: Uses the Task tool to launch with the code-reviewer agent
</example>

<example>
user: "Hello"
<commentary>
Since the user is greeting, use the greeting-responder agent
</commentary>
assistant: "I'm going to use the Task tool to launch with the greeting-responder agent"
</example>`
}
```

**Few-Shot 설계 원칙:**
1. **실제 사용 사례**: 일반적인 시나리오 제공
2. **단계별 사고**: `<commentary>` 태그로 추론 과정 표시
3. **올바른 패턴**: 권장되는 사용 방법 시연
4. **다양한 시나리오**: 여러 상황 커버

### 6.4 제약사항 명시

프롬프트에서 하지 말아야 할 것을 명확히 합니다:

```typescript
`When NOT to use the Agent tool:
- If you want to read a specific file path, use the FileRead or Glob tool instead of the Agent tool
- If you are searching for a specific class definition like "class Foo", use the Glob tool instead
- If you are searching for code within a specific file or set of 2-3 files, use the FileRead tool instead
- Other tasks that are not related to the agent descriptions above`
```

**제약사항 설계 원칙:**
1. **부정적 예제**: "하지 말아야 할 것" 명시
2. **대안 제시**: "대신 X를 사용하라"
3. **구체적 시나리오**: 일반적인 실수 케이스
4. **이유 설명**: "to find the match more quickly"

### 6.5 메타 지시사항

프롬프트에 작업 수행 방식을 지시합니다:

```typescript
`Usage notes:
1. Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses
2. When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result.
3. Each agent invocation is stateless. You will not be able to send additional messages to the agent, nor will the agent be able to communicate with you outside of its final report.
4. The agent's outputs should generally be trusted
5. Clearly tell the agent whether you expect it to write code or just to do research
6. If the agent description mentions that it should be used proactively, then you should try your best to use it without the user having to ask for it first.`
```

**메타 지시사항 카테고리:**
1. **성능 최적화**: "Launch multiple agents concurrently"
2. **통신 프로토콜**: "return a single message back to you"
3. **상태 관리**: "stateless"
4. **신뢰성**: "outputs should generally be trusted"
5. **명확성**: "Clearly tell the agent"
6. **주도성**: "used proactively"

### 6.6 동적 프롬프트 생성

Agent 목록에 따라 프롬프트를 동적으로 생성합니다:

```typescript
export async function getPrompt(safeMode: boolean): Promise<string> {
  const agents = await getActiveAgents()

  // Agent 설명 동적 생성
  const agentDescriptions = agents.map(agent => {
    const toolsStr = Array.isArray(agent.tools)
      ? agent.tools.join(', ')
      : '*'
    return `- ${agent.agentType}: ${agent.whenToUse} (Tools: ${toolsStr})`
  }).join('\n')

  return `...
Available agent types and the tools they have access to:
${agentDescriptions}
...`
}
```

**동적 생성 이점:**
1. **확장성**: 새로운 Agent 추가 시 자동 반영
2. **일관성**: Agent 정의와 프롬프트가 동기화
3. **정확성**: 실제 사용 가능한 Agent만 표시
4. **유지보수성**: 중복 제거, 단일 진실 소스

### 6.7 컨텍스트 주입

프롬프트에 동적 컨텍스트를 주입합니다:

```typescript
// 모델 자기 참조 방지
taskPrompt.push(`\nIMPORTANT: You are currently running as ${modelToUse}. You do not need to consult ${modelToUse} via AskExpertModel since you ARE ${modelToUse}.`)

// 컨텍스트 태그로 구분
const content = context
  ? `<context>${context}</context>\n\n${prompt}`
  : prompt
```

**컨텍스트 주입 전략:**
1. **명확한 구분**: XML 태그 사용 (`<context>`)
2. **메타 정보**: 현재 실행 환경 정보 제공
3. **순환 방지**: 자기 참조 경고
4. **우선순위**: 중요한 정보를 앞에 배치

---

## 병렬 실행 및 동시성 제어

### 7.1 동시성 안전성 플래그

각 도구는 동시 실행 가능 여부를 선언합니다:

```typescript
// TaskTool.tsx
isConcurrencySafe() {
  return true // Task tool supports concurrent execution
}

// ArchitectTool.tsx
isConcurrencySafe() {
  return true // ArchitectTool is read-only, safe for concurrent execution
}

// ThinkTool.tsx
isConcurrencySafe() {
  return true // ThinkTool is read-only, safe for concurrent execution
}
```

**안전성 판단 기준:**
- **읽기 전용 도구**: 항상 안전 (FileRead, Grep, Glob)
- **상태 비저장 도구**: 안전 (Task, Architect, Think)
- **쓰기 도구**: 조건부 안전 (FileWrite는 다른 파일이면 안전)
- **전역 상태 변경**: 불안전 (설정 변경, 모델 전환)

### 7.2 병렬 Task 실행 패턴

AI가 여러 Task를 동시에 실행하는 방법:

```typescript
// AI의 도구 호출 (한 번의 API 응답에 여러 tool_use)
{
  role: 'assistant',
  content: [
    {
      type: 'text',
      text: '세 개의 파일을 병렬로 분석하겠습니다.'
    },
    {
      type: 'tool_use',
      id: 'toolu_1',
      name: 'Task',
      input: {
        description: 'Analyze auth.ts',
        prompt: 'Review auth.ts for security issues',
        subagent_type: 'code-reviewer'
      }
    },
    {
      type: 'tool_use',
      id: 'toolu_2',
      name: 'Task',
      input: {
        description: 'Analyze database.ts',
        prompt: 'Review database.ts for performance issues',
        subagent_type: 'code-reviewer'
      }
    },
    {
      type: 'tool_use',
      id: 'toolu_3',
      name: 'Task',
      input: {
        description: 'Analyze api.ts',
        prompt: 'Review api.ts for error handling',
        subagent_type: 'code-reviewer'
      }
    }
  ]
}
```

**병렬 실행 메커니즘:**
1. Claude API는 여러 `tool_use`를 한 번에 반환
2. Kode-CLI는 각 `tool_use`에 대해 비동기적으로 도구 실행
3. 모든 결과가 수집될 때까지 대기
4. 결과를 `tool_result` 배열로 반환
5. AI가 모든 결과를 종합하여 응답 생성

### 7.3 AbortController를 통한 중단 제어

각 Agent는 중단 가능한 실행을 지원합니다:

```typescript
async *call(
  input,
  {
    abortController,  // 중단 컨트롤러
    options,
    readFileTimestamps,
  },
) {
  // AbortController를 하위 query에 전달
  for await (const message of query(
    messages,
    taskPrompt,
    context,
    hasPermissionsToUseTool,
    {
      abortController,  // 중단 신호 전파
      options: queryOptions,
      messageId: getLastAssistantMessageId(messages),
      agentId: taskId,
      readFileTimestamps,
      setToolJSX: () => {},
    },
  )) {
    // 중단 확인
    if (abortController.signal.aborted) {
      yield {
        type: 'result',
        data: [{ type: 'text', text: INTERRUPT_MESSAGE }],
        resultForAssistant: 'Task interrupted',
      }
      return
    }

    // 정상 처리
    messages.push(message)
  }
}
```

**중단 전파 메커니즘:**
1. 사용자가 Ctrl+C 누름
2. 최상위 REPL이 `abortController.abort()` 호출
3. 모든 실행 중인 Agent에게 중단 신호 전파
4. 각 Agent가 현재 작업 중단
5. 부분 결과 반환 (가능한 경우)
6. 리소스 정리 (파일, 네트워크 등)

### 7.4 읽기 전용 vs 쓰기 도구

동시성 안전성은 도구의 읽기/쓰기 특성에 따라 다릅니다:

```typescript
// 읽기 전용 도구 (항상 안전)
isReadOnly() {
  return true
}
isConcurrencySafe() {
  return true
}

// 쓰기 도구 (조건부 안전)
isReadOnly() {
  return false
}
isConcurrencySafe() {
  return true  // 다른 파일을 수정하면 안전
}
```

**동시성 규칙:**
- ✅ **안전**: 여러 Agent가 다른 파일 읽기
- ✅ **안전**: 여러 Agent가 다른 파일 쓰기
- ⚠️ **주의**: 여러 Agent가 같은 파일 읽기 (버전 불일치 가능)
- ❌ **불안전**: 여러 Agent가 같은 파일 쓰기 (경쟁 조건)

### 7.5 도구 필터링과 병렬성

Agent 타입에 따라 사용 가능한 도구를 제한하여 안전성을 보장합니다:

```typescript
// 읽기 전용 Agent (안전)
{
  name: 'code-analyzer',
  tools: ['FileRead', 'Grep', 'Glob'],  // 읽기만 가능
  // 병렬 실행 완전 안전
}

// 쓰기 가능 Agent (주의 필요)
{
  name: 'code-writer',
  tools: ['FileRead', 'FileWrite', 'FileEdit'],  // 쓰기 가능
  // 병렬 실행 시 다른 파일 작업 권장
}

// 전체 도구 Agent (조율 필요)
{
  name: 'general-purpose',
  tools: '*',  // 모든 도구 사용 가능
  // 병렬 실행 시 충돌 가능성 있음
}
```

### 7.6 로그 사이드체인

병렬 실행 시 각 Agent의 로그를 분리합니다:

```typescript
// 사이드체인 번호 생성 (메모이제이션)
const getSidechainNumber = memoize(() =>
  getNextAvailableLogSidechainNumber(messageLogName, forkNumber),
)

// 각 Agent의 로그를 별도 파일에 저장
overwriteLog(
  getMessagesPath(messageLogName, forkNumber, getSidechainNumber()),
  messages.filter(_ => _.type !== 'progress'),
)
```

**로그 구조:**
```
logs/
  main.json              # 주 Agent 로그
  main-fork-1.json       # SubAgent 1 로그
  main-fork-2.json       # SubAgent 2 로그
  main-fork-3.json       # SubAgent 3 로그
```

**로그 분리 이점:**
1. 병렬 실행 추적 가능
2. 각 Agent의 독립적 디버깅
3. 로그 파일 경쟁 조건 방지
4. 계층적 실행 추적

### 7.7 병렬 실행 최적화

프롬프트에서 병렬 실행을 권장합니다:

```typescript
`Usage notes:
1. Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses`
```

**최적화 전략:**

**순차 실행 (비효율적):**
```typescript
// AI가 순차적으로 Task 호출
1. Task(prompt: "Analyze file A")
   → 결과 대기 (30초)
2. Task(prompt: "Analyze file B")
   → 결과 대기 (30초)
3. Task(prompt: "Analyze file C")
   → 결과 대기 (30초)

총 시간: 90초
```

**병렬 실행 (효율적):**
```typescript
// AI가 동시에 여러 Task 호출
Task(prompt: "Analyze file A") ─┐
Task(prompt: "Analyze file B") ─┼─→ 모든 결과 대기
Task(prompt: "Analyze file C") ─┘

총 시간: 30초 (3배 빠름)
```

**병렬 실행 조건:**
1. **독립성**: 작업 간 의존성 없음
2. **안전성**: 동시 실행해도 충돌 없음
3. **리소스**: 충분한 메모리/CPU
4. **API 한도**: Rate limit 고려

---

## 아키텍처 다이어그램

### 8.1 전체 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     사용자 (User)                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  REPL (대화형 인터페이스)                        │
│  - 사용자 입력 처리                                              │
│  - 명령어 파싱                                                   │
│  - UI 렌더링 (Ink)                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│               주 Agent (Main Agent)                           │
│  - 사용자 의도 파악                                              │
│  - 작업 분해 및 위임                                             │
│  - 결과 합성 및 응답 생성                                         │
└─────┬───────────┬───────────┬───────────────────────────────┘
      │           │           │
      ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ TaskTool │ │Architect │ │ThinkTool │
│          │ │   Tool   │ │          │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     ▼            ▼            ▼
┌─────────────────────────────────────┐
│      SubAgent 생성 및 실행             │
│  - Agent 설정 로드                    │
│  - 프롬프트 구성                       │
│  - 도구 필터링                        │
│  - 모델 선택                          │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│         query() 실행 루프              │
│  - API 호출 (Claude/OpenAI)          │
│  - 도구 실행                          │
│  - 스트리밍 응답                       │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│           도구 계층                    │
│  FileRead  FileWrite  FileEdit       │
│  Bash      Grep       Glob           │
│  WebFetch  Memory     ...            │
└─────────────────────────────────────┘
```

### 8.2 TaskTool 실행 흐름

```
TaskTool.call(input)
      │
      ▼
┌─────────────────────────────────────┐
│  1. 입력 검증                         │
│     - description 확인                │
│     - prompt 확인                     │
│     - model_name 검증                 │
│     - subagent_type 검증              │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  2. Agent 설정 로드                   │
│     - getAgentByType(subagent_type)  │
│     - systemPrompt 추출               │
│     - model_name 결정                 │
│     - tools 필터 설정                 │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  3. 프롬프트 구성                      │
│     effectivePrompt =                │
│       agentPrompt + userPrompt      │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  4. 도구 필터링                        │
│     tools = getTaskTools(safeMode)   │
│     tools = tools.filter(toolFilter) │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  5. 초기 Progress 메시지              │
│     yield { type: 'progress', ... }  │
│     - Starting agent                 │
│     - Using model                    │
│     - Task description               │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  6. query() 실행                      │
│     for await (message of query())   │
│       ├─ 도구 사용 추적                │
│       ├─ 진행 상황 스트리밍             │
│       └─ 로그 저장                     │
└─────┬───────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  7. 결과 반환                         │
│     yield {                          │
│       type: 'result',                │
│       data: TextBlock[],             │
│       resultForAssistant: string     │
│     }                                │
└─────────────────────────────────────┘
```

### 8.3 Agent 로딩 우선순위

```
Agent 로딩 시스템
      │
      ▼
┌──────────────────────────────────────────┐
│  1. Built-in Agents                      │
│     - 코드에 하드코딩된 Agent              │
│     - 항상 사용 가능                       │
│     - 기본 기능 보장                       │
└─────┬────────────────────────────────────┘
      │ (덮어쓰기 가능)
      ▼
┌──────────────────────────────────────────┐
│  2. ~/.claude/agents/                    │
│     - Claude Code 호환성                  │
│     - 사용자 전역 Agent                    │
│     - 모든 프로젝트에 적용                  │
└─────┬────────────────────────────────────┘
      │ (덮어쓰기 가능)
      ▼
┌──────────────────────────────────────────┐
│  3. ~/.kode/agents/                      │
│     - Kode 전용 Agent                     │
│     - 사용자 커스터마이징                   │
└─────┬────────────────────────────────────┘
      │ (덮어쓰기 가능)
      ▼
┌──────────────────────────────────────────┐
│  4. ./.claude/agents/                    │
│     - Claude Code 프로젝트 Agent          │
│     - 프로젝트별 특수화                     │
└─────┬────────────────────────────────────┘
      │ (덮어쓰기 가능)
      ▼
┌──────────────────────────────────────────┐
│  5. ./.kode/agents/                      │
│     - Kode 프로젝트 Agent                 │
│     - 최고 우선순위                        │
│     - 프로젝트 특화 Agent                  │
└──────────────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────┐
│  최종 Agent 목록                          │
│  - 중복 제거 (이름 기준)                   │
│  - 우선순위 높은 것만 유지                  │
└──────────────────────────────────────────┘
```

### 8.4 병렬 Agent 실행

```
주 Agent
   │
   │ (여러 Task 동시 호출)
   ├─────────────┬─────────────┬─────────────┐
   │             │             │             │
   ▼             ▼             ▼             ▼
┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐
│Task 1│     │Task 2│     │Task 3│     │Task 4│
│      │     │      │     │      │     │      │
│Agent │     │Agent │     │Agent │     │Agent │
│  ID: │     │  ID: │     │  ID: │     │  ID: │
│ abc1 │     │ abc2 │     │ abc3 │     │ abc4 │
└──┬───┘     └──┬───┘     └──┬───┘     └──┬───┘
   │            │            │            │
   │ (독립적 실행)           │            │
   ▼            ▼            ▼            ▼
┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐
│File  │     │File  │     │Bash  │     │Grep  │
│Read  │     │Write │     │      │     │      │
└──┬───┘     └──┬───┘     └──┬───┘     └──┬───┘
   │            │            │            │
   │ (병렬 완료 대기)         │            │
   ├────────────┴────────────┴────────────┤
   │                                       │
   ▼                                       ▼
┌─────────────────────────────────────────┐
│         모든 결과 수집                     │
│  Result 1, Result 2, Result 3, Result 4 │
└─────┬───────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│         주 Agent가 결과 합성               │
│  - 패턴 분석                              │
│  - 우선순위 설정                          │
│  - 종합 보고서 생성                        │
└─────────────────────────────────────────┘
```

### 8.5 메시지 흐름

```
사용자 입력
   │
   ▼
┌─────────────────────────────────────────┐
│  UserMessage                             │
│  { role: 'user', content: '...' }        │
└─────┬───────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  AssistantMessage (도구 호출)             │
│  {                                       │
│    role: 'assistant',                    │
│    content: [                            │
│      { type: 'text', text: '...' },      │
│      { type: 'tool_use', name: 'Task' }  │
│    ]                                     │
│  }                                       │
└─────┬───────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  TaskTool 실행                            │
│  - yield { type: 'progress', ... }       │
│  - yield { type: 'progress', ... }       │
│  - yield { type: 'result', data: ... }   │
└─────┬───────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  ToolResultMessage                       │
│  {                                       │
│    role: 'user',                         │
│    content: [                            │
│      {                                   │
│        type: 'tool_result',              │
│        tool_use_id: '...',               │
│        content: TextBlock[]              │
│      }                                   │
│    ]                                     │
│  }                                       │
└─────┬───────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  AssistantMessage (최종 응답)             │
│  {                                       │
│    role: 'assistant',                    │
│    content: [                            │
│      {                                   │
│        type: 'text',                     │
│        text: '결과 요약 및 분석...'       │
│      }                                   │
│    ]                                     │
│  }                                       │
└─────────────────────────────────────────┘
```

---

## 실전 활용 예제

### 9.1 코드 리뷰 자동화

**시나리오**: PR에 포함된 여러 파일을 동시에 리뷰

```typescript
// 사용자 요청
"PR #123의 모든 변경 파일을 리뷰해주세요"

// 주 Agent의 작업 분해
1. PR 파일 목록 가져오기 (Bash: gh pr view 123 --json files)
2. 각 파일에 대해 병렬로 code-reviewer Agent 실행

// AI의 도구 호출
[
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Review auth.ts',
      prompt: 'Review src/auth.ts for security vulnerabilities, bugs, and code quality issues. Focus on authentication and authorization logic.',
      subagent_type: 'code-reviewer'
    }
  },
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Review api.ts',
      prompt: 'Review src/api.ts for error handling, input validation, and API design. Check for potential security issues.',
      subagent_type: 'code-reviewer'
    }
  },
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Review tests',
      prompt: 'Review tests/auth.test.ts for test coverage, edge cases, and test quality.',
      subagent_type: 'code-reviewer'
    }
  }
]

// 각 Agent의 결과
Task 1 (auth.ts):
"""
## 보안 이슈
1. **Critical**: JWT secret이 환경 변수가 아닌 하드코딩됨 (라인 15)
   - 위험: Secret 노출 시 전체 시스템 위협
   - 수정: process.env.JWT_SECRET 사용

2. **High**: 비밀번호 검증 로직 누락 (라인 45)
   - 위험: 약한 비밀번호 허용
   - 수정: 최소 8자, 특수문자 포함 검증 추가

## 코드 품질
1. **Medium**: 에러 메시지가 너무 상세 (라인 67)
   - 문제: 공격자에게 힌트 제공
   - 수정: 일반적인 메시지 사용
"""

Task 2 (api.ts):
"""
## 버그
1. **High**: Race condition in user update (라인 123)
   - 문제: 동시 요청 시 데이터 불일치
   - 수정: 트랜잭션 또는 낙관적 잠금 사용

## 개선 사항
1. **Medium**: Rate limiting 없음
   - 문제: DDoS 취약
   - 수정: express-rate-limit 추가
"""

Task 3 (tests):
"""
## 테스트 커버리지
1. **High**: 에러 케이스 테스트 부족
   - 문제: 정상 경로만 테스트
   - 수정: 실패 시나리오 테스트 추가

2. **Medium**: 통합 테스트 없음
   - 문제: 컴포넌트 간 통합 검증 안 됨
   - 수정: E2E 테스트 추가
"""

// 주 Agent의 결과 합성
"""
# PR #123 코드 리뷰 요약

## 🚨 Critical Issues (즉시 수정 필요)
1. **JWT Secret 하드코딩** (auth.ts:15)
   - 파일: src/auth.ts
   - 수정 방법: 환경 변수로 이동
   ```typescript
   const jwtSecret = process.env.JWT_SECRET
   if (!jwtSecret) throw new Error('JWT_SECRET not configured')
   ```

## ⚠️ High Priority Issues
1. **비밀번호 검증 누락** (auth.ts:45)
2. **Race Condition** (api.ts:123)
3. **에러 케이스 테스트 부족** (tests)

## 📋 Medium Priority Issues
1. 에러 메시지 보안
2. Rate limiting 추가
3. 통합 테스트 추가

## 다음 단계
1. Critical 이슈 즉시 수정 후 재커밋
2. High Priority 이슈는 이번 PR에서 처리
3. Medium Priority는 별도 이슈로 트래킹

총 3개 파일 리뷰 완료. 1개의 Critical, 3개의 High, 3개의 Medium 이슈 발견.
"""
```

### 9.2 프로젝트 아키텍처 분석

**시나리오**: 새로운 팀원을 위한 프로젝트 구조 문서화

```typescript
// 사용자 요청
"프로젝트 구조를 분석하고 신규 개발자를 위한 가이드를 작성해주세요"

// AI의 접근 방식
// 1단계: Architect로 프로젝트 구조 분석
{
  type: 'tool_use',
  name: 'Architect',
  input: {
    prompt: `Analyze the project structure and identify:
    1. Main entry points
    2. Core modules and their responsibilities
    3. Key dependencies and their purposes
    4. Configuration files and their roles
    5. Testing strategy`,
    context: 'This is for onboarding documentation'
  }
}

// Architect 결과
"""
# 프로젝트 구조 분석

## 진입점
1. **src/entrypoints/cli.tsx**: CLI 애플리케이션 시작점
   - Ink 앱 초기화
   - 설정 로드
   - REPL 시작

## 핵심 모듈

### 1. Agent 시스템 (src/tools/TaskTool/)
- **책임**: SubAgent 생성 및 관리
- **주요 파일**:
  - TaskTool.tsx: Agent 오케스트레이션
  - prompt.ts: Agent 프롬프트 관리
- **의존성**: agentLoader, query

### 2. 도구 계층 (src/tools/)
- **책임**: 파일 I/O, 코드 검색, 명령 실행
- **주요 도구**:
  - FileReadTool: 파일 읽기
  - FileWriteTool: 파일 쓰기
  - GrepTool: 코드 검색
  - BashTool: 명령 실행

### 3. 서비스 계층 (src/services/)
- **책임**: 외부 API 통합
- **주요 서비스**:
  - claude.ts: Anthropic API
  - openai.ts: OpenAI API
  - mcpClient.ts: MCP 프로토콜

## 주요 의존성
1. **@anthropic-ai/sdk**: Claude API 클라이언트
2. **ink**: 터미널 UI 프레임워크
3. **zod**: 스키마 검증
4. **chalk**: 터미널 색상

## 설정 시스템
1. **~/.kode.json**: 사용자 전역 설정
2. **./.kode.json**: 프로젝트 설정
3. **환경 변수**: API 키, 디버그 플래그

## 테스트 전략
- **단위 테스트**: bun test
- **타입 체크**: bun run typecheck
- **통합 테스트**: 수동 CLI 테스트
"""

// 2단계: Think로 문서 구조 계획
{
  type: 'tool_use',
  name: 'Think',
  input: {
    thought: `
    Architect의 분석 결과를 바탕으로 신규 개발자 가이드 구성:

    1. **Quick Start** (5분 내 실행)
       - 설치 방법
       - 첫 명령어 실행
       - 기본 사용법

    2. **프로젝트 구조** (15분 내 이해)
       - 디렉토리 구조 다이어그램
       - 주요 모듈 설명
       - 의존성 그래프

    3. **개발 워크플로우** (30분 내 기여)
       - 로컬 개발 환경 설정
       - 테스트 실행
       - PR 프로세스

    4. **핵심 개념** (1시간 내 숙지)
       - Agent 시스템
       - 도구 아키텍처
       - 프롬프트 설계

    5. **예제** (실습)
       - 새로운 도구 추가
       - Agent 커스터마이징
       - 디버깅 팁
    `
  }
}

// 3단계: 여러 Task로 예제 코드 생성
[
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Create tool example',
      prompt: 'Write a complete example of creating a new tool, including TypeScript code, schema definition, and prompt design.',
      subagent_type: 'general-purpose'
    }
  },
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Create agent example',
      prompt: 'Write an example of creating a custom agent with markdown format, YAML frontmatter, and system prompt.',
      subagent_type: 'general-purpose'
    }
  }
]

// 최종 결과: 종합 온보딩 가이드
"""
# Kode-CLI 신규 개발자 가이드

## Quick Start (5분)

### 설치
```bash
npm install -g kode-cli
# 또는
bun install -g kode-cli
```

### API 키 설정
```bash
export ANTHROPIC_API_KEY=your-key-here
```

### 첫 명령어
```bash
kode
> Hello, analyze this project
```

## 프로젝트 구조 (15분)

[... Architect의 분석 결과 포함 ...]

## 개발 워크플로우 (30분)

[... 개발 환경 설정 가이드 ...]

## 핵심 개념 (1시간)

[... Agent 시스템 설명 ...]

## 실습 예제

### 예제 1: 새로운 도구 만들기

[... Task 1의 결과 포함 ...]

### 예제 2: 커스텀 Agent 만들기

[... Task 2의 결과 포함 ...]

## 추가 리소스
- GitHub: https://github.com/yourproject/kode-cli
- 문서: https://docs.kode-cli.dev
- Discord: https://discord.gg/kode-cli
"""
```

### 9.3 버그 수정 워크플로우

**시나리오**: 복잡한 버그를 체계적으로 분석하고 수정

```typescript
// 사용자 요청
"사용자가 로그아웃한 후에도 세션이 남아있는 버그를 수정해주세요"

// 1단계: Think로 조사 계획 수립
{
  type: 'tool_use',
  name: 'Think',
  input: {
    thought: `
    세션 버그 조사 계획:

    1. **증상 파악**
       - 로그아웃 후 세션 쿠키 확인
       - 서버 세션 스토어 상태 확인
       - 클라이언트 로컬 스토리지 확인

    2. **의심 지점**
       a) 로그아웃 API가 세션 삭제 안 함
       b) 쿠키 만료 설정 오류
       c) 클라이언트 상태 정리 누락
       d) 세션 스토어 동기화 이슈

    3. **조사 순서**
       1. 로그아웃 엔드포인트 코드 검토
       2. 세션 미들웨어 설정 검토
       3. 클라이언트 로그아웃 로직 검토
       4. 테스트 케이스 확인

    4. **병렬 실행 가능 작업**
       - Task 1: 서버 로그아웃 로직 분석
       - Task 2: 클라이언트 로그아웃 로직 분석
       - Task 3: 세션 관련 테스트 검토
    `
  }
}

// 2단계: 병렬로 코드 분석
[
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Analyze server logout',
      prompt: `
      Analyze the server-side logout implementation:
      1. Find logout endpoint (likely /api/auth/logout or /logout)
      2. Check if session.destroy() is called
      3. Verify cookie clearing (res.clearCookie)
      4. Check error handling

      Look in:
      - src/routes/auth.ts
      - src/controllers/auth.ts
      - src/middleware/session.ts
      `,
      subagent_type: 'general-purpose'
    }
  },
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Analyze client logout',
      prompt: `
      Analyze the client-side logout implementation:
      1. Find logout function in frontend code
      2. Check if localStorage/sessionStorage is cleared
      3. Verify API call to logout endpoint
      4. Check state management reset (Redux/Context)

      Look in:
      - src/auth/logout.js
      - src/store/authSlice.js
      - src/hooks/useAuth.js
      `,
      subagent_type: 'general-purpose'
    }
  },
  {
    type: 'tool_use',
    name: 'Task',
    input: {
      description: 'Review logout tests',
      prompt: `
      Review logout-related tests:
      1. Find logout test cases
      2. Check if session clearing is tested
      3. Verify test coverage
      4. Identify missing test scenarios

      Look in:
      - tests/auth.test.ts
      - tests/integration/logout.test.ts
      `,
      subagent_type: 'general-purpose'
    }
  }
]

// Task 1 결과 (서버)
"""
발견된 이슈:
1. **버그 발견**: src/routes/auth.ts의 logout 엔드포인트에서
   session.destroy() 호출은 있지만, 콜백 에러 처리가 없음 (라인 78)

   ```typescript
   // 현재 코드 (버그)
   app.post('/logout', (req, res) => {
     req.session.destroy()  // ❌ 에러 처리 없음
     res.clearCookie('sessionId')
     res.json({ success: true })
   })
   ```

   문제: session.destroy()가 비동기이며 에러 발생 시 세션이 삭제되지 않음

2. **추가 이슈**: 쿠키 옵션이 설정과 다름
   - 설정 시: { httpOnly: true, secure: true, sameSite: 'strict' }
   - 삭제 시: 옵션 없음 → 삭제 실패 가능
"""

// Task 2 결과 (클라이언트)
"""
클라이언트 코드는 정상:
- localStorage 정리: ✓
- 상태 관리 리셋: ✓
- API 호출: ✓

문제 없음. 서버 측 이슈로 판단됨.
"""

// Task 3 결과 (테스트)
"""
테스트 누락 발견:
1. 로그아웃 후 세션 접근 시도 테스트 없음
2. 세션 삭제 실패 시나리오 테스트 없음
3. 동시 로그아웃 요청 테스트 없음

기존 테스트는 정상 경로만 검증함.
"""

// 3단계: Think로 수정 전략 수립
{
  type: 'tool_use',
  name: 'Think',
  input: {
    thought: `
    버그 원인 확정: 서버 logout 엔드포인트의 에러 처리 누락

    수정 전략:

    Option A: Promise 패턴
    - session.destroy()를 Promise로 래핑
    - async/await 사용
    - 명확한 에러 처리

    Option B: 콜백 패턴
    - session.destroy(callback) 사용
    - 전통적인 Express 패턴
    - 덜 현대적이지만 안정적

    **선택: Option A** (팀이 async/await 선호)

    수정 계획:
    1. logout 엔드포인트를 async 함수로 변경
    2. session.destroy()를 Promise로 래핑
    3. try-catch로 에러 처리
    4. 쿠키 삭제 시 올바른 옵션 사용
    5. 테스트 케이스 추가
    `
  }
}

// 4단계: 수정 적용
{
  type: 'tool_use',
  name: 'FileEdit',
  input: {
    path: 'src/routes/auth.ts',
    old_string: `app.post('/logout', (req, res) => {
  req.session.destroy()
  res.clearCookie('sessionId')
  res.json({ success: true })
})`,
    new_string: `app.post('/logout', async (req, res) => {
  try {
    // Promise로 session.destroy() 래핑
    await new Promise((resolve, reject) => {
      req.session.destroy((err) => {
        if (err) reject(err)
        else resolve()
      })
    })

    // 쿠키 생성 시와 동일한 옵션으로 삭제
    res.clearCookie('sessionId', {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict'
    })

    res.json({ success: true })
  } catch (error) {
    console.error('Logout error:', error)
    res.status(500).json({
      success: false,
      error: 'Failed to logout'
    })
  }
})`
  }
}

// 5단계: 테스트 추가
{
  type: 'tool_use',
  name: 'FileEdit',
  input: {
    path: 'tests/auth.test.ts',
    old_string: `describe('Logout', () => {
  it('should logout successfully', async () => {
    // 기존 테스트
  })
})`,
    new_string: `describe('Logout', () => {
  it('should logout successfully', async () => {
    // 기존 테스트
  })

  it('should clear session on logout', async () => {
    const agent = request.agent(app)

    // 로그인
    await agent.post('/login').send({
      username: 'test',
      password: 'password'
    })

    // 로그아웃
    await agent.post('/logout').expect(200)

    // 세션이 삭제되었는지 확인
    const res = await agent.get('/api/protected')
    expect(res.status).toBe(401)
  })

  it('should handle session destroy errors', async () => {
    // session.destroy() 실패 시뮬레이션
    jest.spyOn(session, 'destroy').mockImplementation((cb) => {
      cb(new Error('Session store error'))
    })

    const res = await request(app)
      .post('/logout')
      .set('Cookie', validSessionCookie)

    expect(res.status).toBe(500)
    expect(res.body.success).toBe(false)
  })
})`
  }
}

// 6단계: 검증
{
  type: 'tool_use',
  name: 'Bash',
  input: {
    command: 'npm test -- auth.test.ts'
  }
}

// 최종 응답
"""
✅ 버그 수정 완료!

## 발견된 문제
세션 삭제 시 에러 처리가 없어서, 세션 스토어에서 삭제 실패 시 세션이 남아있었습니다.

## 적용된 수정
1. **src/routes/auth.ts**:
   - logout을 async 함수로 변경
   - session.destroy()를 Promise로 래핑하여 에러 처리 추가
   - 쿠키 삭제 시 올바른 옵션 사용

2. **tests/auth.test.ts**:
   - 세션 정리 검증 테스트 추가
   - 에러 시나리오 테스트 추가

## 테스트 결과
```
✓ should logout successfully
✓ should clear session on logout
✓ should handle session destroy errors

3 passing (45ms)
```

## 다음 단계
1. 수동 테스트로 재확인
2. PR 생성 및 코드 리뷰 요청
3. 스테이징 환경 배포 및 검증
"""
```

---

## 결론

Kode-CLI의 Agent 및 Task 시스템은 다음과 같은 핵심 특징을 가집니다:

### 주요 강점

1. **확장 가능한 Agent 시스템**
   - 동적 Agent 로딩
   - 마크다운 기반 설정
   - 5-tier 우선순위 계층

2. **병렬 실행 최적화**
   - 동시성 안전한 도구
   - 독립적인 SubAgent 실행
   - 로그 사이드체인

3. **명확한 역할 분리**
   - TaskTool: 오케스트레이션
   - ArchitectTool: 계획 수립
   - ThinkTool: 추론 및 브레인스토밍

4. **견고한 프롬프트 설계**
   - 계층적 프롬프트 구조
   - Few-Shot 학습
   - 명확한 제약사항

5. **실시간 피드백**
   - AsyncGenerator 스트리밍
   - Progress 메시지
   - 도구 사용 추적

### 설계 철학

1. **상태 비저장**: 각 Agent는 독립적으로 실행
2. **명시적 통신**: 명확한 입력/출력 계약
3. **실패 복원력**: 우아한 에러 처리
4. **사용자 제어**: 중단 및 권한 관리
5. **투명성**: 모든 작업 가시화

### 향후 개선 방향

1. **Agent 간 직접 통신**: SubAgent가 다른 SubAgent 호출
2. **상태 공유**: Agent 간 컨텍스트 공유
3. **학습 및 개선**: Agent 성능 메트릭 수집
4. **동적 도구 생성**: 런타임에 도구 정의
5. **분산 실행**: 여러 머신에서 Agent 실행

이 시스템은 복잡한 소프트웨어 개발 작업을 효율적으로 자동화하면서도, 사용자에게 완전한 제어권과 투명성을 제공합니다.
