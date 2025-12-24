# Kode-CLI API 어댑터 시스템 분석

## 목차
1. [시스템 개요](#1-시스템-개요)
2. [기본 어댑터 인터페이스](#2-기본-어댑터-인터페이스)
3. [OpenAI API 어댑터](#3-openai-api-어댑터)
4. [Chat Completions API 처리](#4-chat-completions-api-처리)
5. [Responses API 처리](#5-responses-api-처리)
6. [스트리밍 응답 처리](#6-스트리밍-응답-처리)
7. [모델 어댑터 팩토리](#7-모델-어댑터-팩토리)
8. [응답 상태 관리](#8-응답-상태-관리)
9. [OAuth 인증 시스템](#9-oauth-인증-시스템)
10. [통합 서비스 레이어](#10-통합-서비스-레이어)

---

## 1. 시스템 개요

### 1.1 아키텍처 설계 원칙

Kode-CLI의 API 어댑터 시스템은 다양한 AI 모델 제공자의 API를 통합하기 위한 추상화 레이어입니다.

```
┌─────────────────────────────────────────────────────────────┐
│                     통합 인터페이스                          │
│  (UnifiedRequestParams / UnifiedResponse)                   │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Model Adapter Factory                      │
│  • API 타입 결정 (Chat Completions vs Responses API)       │
│  • 모델 기반 어댑터 생성                                     │
└─────────────────────────────────────────────────────────────┘
                            ▼
        ┌──────────────────┴──────────────────┐
        ▼                                     ▼
┌──────────────────┐              ┌──────────────────────┐
│ OpenAIAdapter    │              │ AnthropicAdapter     │
│ (추상 클래스)     │              │ (Native SDK)         │
└──────────────────┘              └──────────────────────┘
        │
        ├── ChatCompletionsAdapter
        └── ResponsesAPIAdapter
```

### 1.2 주요 설계 목표

1. **다중 API 지원**: 단일 인터페이스로 다양한 AI API 통합
2. **스트리밍 처리**: 실시간 응답 스트리밍 지원
3. **토큰 정규화**: 다양한 토큰 형식을 통합된 구조로 변환
4. **상태 관리**: 대화 상태 및 응답 연속성 유지
5. **에러 처리**: 강력한 재시도 및 복구 메커니즘

### 1.3 핵심 구성 요소

| 컴포넌트 | 역할 | 파일 |
|---------|------|------|
| **ModelAPIAdapter** | 기본 어댑터 추상 클래스 | base.ts |
| **OpenAIAdapter** | OpenAI 호환 API 기본 클래스 | openaiAdapter.ts |
| **ChatCompletionsAdapter** | Chat Completions API 구현 | chatCompletions.ts |
| **ResponsesAPIAdapter** | Responses API 구현 | responsesAPI.ts |
| **ModelAdapterFactory** | 어댑터 생성 팩토리 | modelAdapterFactory.ts |
| **ResponseStateManager** | 응답 상태 관리자 | responseStateManager.ts |

---

## 2. 기본 어댑터 인터페이스

### 2.1 ModelAPIAdapter 추상 클래스

**파일**: `/src/services/adapters/base.ts`

#### 핵심 역할
- 모든 어댑터의 기본 인터페이스 정의
- 토큰 사용량 추적 및 정규화
- 공통 유틸리티 메서드 제공

#### 주요 구조

```typescript
// 정규화된 토큰 표현
interface TokenUsage {
  input: number          // 입력 토큰
  output: number         // 출력 토큰
  total?: number         // 총 토큰
  reasoning?: number     // 추론 토큰 (GPT-5 등)
}

// 스트리밍 이벤트 타입
export type StreamingEvent =
  | { type: 'message_start', message: any, responseId: string }
  | { type: 'text_delta', delta: string, responseId: string }
  | { type: 'tool_request', tool: any }
  | { type: 'usage', usage: TokenUsage }
  | { type: 'message_stop', message: any }
  | { type: 'error', error: string }
```

#### 토큰 정규화 시스템

**목적**: 다양한 API의 토큰 필드명을 통일된 형식으로 변환

```typescript
function normalizeTokens(apiResponse: any): TokenUsage {
  // 입력 검증
  if (!apiResponse || typeof apiResponse !== 'object') {
    return { input: 0, output: 0 }
  }

  // 다양한 필드명 처리
  const input = Number(
    apiResponse.prompt_tokens ??        // OpenAI
    apiResponse.input_tokens ??         // Anthropic
    apiResponse.promptTokens            // 대체 형식
  ) || 0

  const output = Number(
    apiResponse.completion_tokens ??    // OpenAI
    apiResponse.output_tokens ??        // Anthropic
    apiResponse.completionTokens        // 대체 형식
  ) || 0

  const total = Number(
    apiResponse.total_tokens ??
    apiResponse.totalTokens
  ) || undefined

  const reasoning = Number(
    apiResponse.reasoning_tokens ??
    apiResponse.reasoningTokens
  ) || undefined

  return {
    input,
    output,
    total: total && total > 0 ? total : undefined,
    reasoning: reasoning && reasoning > 0 ? reasoning : undefined
  }
}
```

#### 추상 클래스 구현

```typescript
export abstract class ModelAPIAdapter {
  protected cumulativeUsage: TokenUsage = { input: 0, output: 0 }

  constructor(
    protected capabilities: ModelCapabilities,
    protected modelProfile: ModelProfile
  ) {}

  // 서브클래스가 구현해야 하는 메서드
  abstract createRequest(params: UnifiedRequestParams): any
  abstract parseResponse(response: any): Promise<UnifiedResponse>
  abstract buildTools(tools: Tool[]): any

  // 선택적 스트리밍 지원
  async *parseStreamingResponse?(
    response: any,
    signal?: AbortSignal
  ): AsyncGenerator<StreamingEvent> {
    return
    yield // TypeScript 만족용
  }

  // 누적 사용량 관리
  protected resetCumulativeUsage(): void {
    this.cumulativeUsage = { input: 0, output: 0 }
  }

  protected updateCumulativeUsage(usage: TokenUsage): void {
    this.cumulativeUsage.input += usage.input
    this.cumulativeUsage.output += usage.output
    if (usage.total) {
      this.cumulativeUsage.total = (this.cumulativeUsage.total || 0) + usage.total
    }
    if (usage.reasoning) {
      this.cumulativeUsage.reasoning = (this.cumulativeUsage.reasoning || 0) + usage.reasoning
    }
  }

  // 유틸리티 메서드
  protected getMaxTokensParam(): string {
    return this.capabilities.parameters.maxTokensField
  }

  protected getTemperature(): number {
    if (this.capabilities.parameters.temperatureMode === 'fixed_one') {
      return 1
    }
    if (this.capabilities.parameters.temperatureMode === 'restricted') {
      return Math.min(1, 0.7)
    }
    return 0.7
  }

  protected shouldIncludeReasoningEffort(): boolean {
    return this.capabilities.parameters.supportsReasoningEffort
  }
}
```

### 2.2 설계 패턴

#### 템플릿 메서드 패턴
- 공통 로직은 기본 클래스에 구현
- 세부 구현은 서브클래스에 위임

#### 전략 패턴
- 모델 capabilities를 통한 동적 동작 결정
- 런타임에 API 특성에 맞게 동작 조정

---

## 3. OpenAI API 어댑터

### 3.1 OpenAIAdapter 추상 클래스

**파일**: `/src/services/adapters/openaiAdapter.ts`

#### 핵심 역할
- OpenAI 호환 API의 공통 기능 제공
- SSE(Server-Sent Events) 스트리밍 파싱
- Chat Completions와 Responses API 공통 로직

#### 통합 응답 처리

```typescript
export abstract class OpenAIAdapter extends ModelAPIAdapter {
  /**
   * 스트리밍/비스트리밍 응답 통합 처리
   */
  async parseResponse(response: any): Promise<UnifiedResponse> {
    // 스트리밍 응답 감지
    if (response?.body instanceof ReadableStream) {
      const { assistantMessage } = await this.parseStreamingOpenAIResponse(response)

      return {
        id: assistantMessage.responseId,
        content: assistantMessage.message.content,
        toolCalls: assistantMessage.message.content
          .filter((block: any) => block.type === 'tool_use')
          .map((block: any) => ({
            id: block.id,
            type: 'function',
            function: {
              name: block.name,
              arguments: JSON.stringify(block.input)
            }
          })),
        usage: this.normalizeUsageForAdapter(assistantMessage.message.usage),
        responseId: assistantMessage.responseId
      }
    }

    // 비스트리밍 응답은 서브클래스로 위임
    return this.parseNonStreamingResponse(response)
  }
}
```

### 3.2 SSE 스트리밍 파싱

#### SSE 형식
```
data: {"id": "chatcmpl-123", "choices": [...]}
data: {"id": "chatcmpl-123", "choices": [...]}
data: [DONE]
```

#### 파싱 구현

```typescript
async *parseStreamingResponse(response: any): AsyncGenerator<StreamingEvent> {
  const reader = response.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''

  let responseId = response.id || `openai_${Date.now()}`
  let hasStarted = false
  let accumulatedContent = ''

  // Responses API용 추론 컨텍스트
  const reasoningContext: ReasoningStreamingContext = {
    thinkOpen: false,
    thinkClosed: false,
    sawAnySummary: false,
    pendingSummaryParagraph: false
  }

  try {
    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      // 버퍼에 청크 추가
      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''

      for (const line of lines) {
        if (line.trim()) {
          const parsed = this.parseSSEChunk(line)
          if (parsed) {
            // 응답 ID 추출
            if (parsed.id) {
              responseId = parsed.id
            }

            // 서브클래스로 처리 위임
            yield* this.processStreamingChunk(
              parsed,
              responseId,
              hasStarted,
              accumulatedContent,
              reasoningContext
            )

            // 상태 업데이트
            const stateUpdate = this.updateStreamingState(parsed, accumulatedContent)
            if (stateUpdate.content) accumulatedContent = stateUpdate.content
            if (stateUpdate.hasStarted) hasStarted = true
          }
        }
      }
    }
  } catch (error) {
    console.error('Error reading streaming response:', error)
    yield {
      type: 'error',
      error: error instanceof Error ? error.message : String(error)
    }
  } finally {
    reader.releaseLock()
  }

  // 최종 메시지
  const finalContent = accumulatedContent
    ? [{ type: 'text', text: accumulatedContent, citations: [] }]
    : [{ type: 'text', text: '', citations: [] }]

  yield {
    type: 'message_stop',
    message: {
      id: responseId,
      role: 'assistant',
      content: finalContent,
      responseId
    }
  }
}
```

#### SSE 청크 파싱

```typescript
protected parseSSEChunk(line: string): any | null {
  if (line.startsWith('data: ')) {
    const data = line.slice(6).trim()

    // 종료 신호
    if (data === '[DONE]') {
      return null
    }

    if (data) {
      try {
        return JSON.parse(data)
      } catch (error) {
        console.error('Error parsing SSE chunk:', error)
        return null
      }
    }
  }
  return null
}
```

### 3.3 텍스트 델타 처리

```typescript
protected handleTextDelta(
  delta: string,
  responseId: string,
  hasStarted: boolean
): StreamingEvent[] {
  const events: StreamingEvent[] = []

  // 첫 델타 시 message_start 이벤트
  if (!hasStarted && delta) {
    events.push({
      type: 'message_start',
      message: {
        role: 'assistant',
        content: []
      },
      responseId
    })
  }

  // 텍스트 델타 이벤트
  if (delta) {
    events.push({
      type: 'text_delta',
      delta,
      responseId
    })
  }

  return events
}
```

### 3.4 사용량 정규화

```typescript
protected normalizeUsageForAdapter(usage?: any) {
  if (!usage) {
    return {
      input_tokens: 0,
      output_tokens: 0,
      promptTokens: 0,
      completionTokens: 0,
      totalTokens: 0,
      reasoningTokens: 0
    }
  }

  const inputTokens =
    usage.input_tokens ??
    usage.prompt_tokens ??
    usage.promptTokens ??
    0
  const outputTokens =
    usage.output_tokens ??
    usage.completion_tokens ??
    usage.completionTokens ??
    0

  return {
    ...usage,
    input_tokens: inputTokens,
    output_tokens: outputTokens,
    promptTokens: inputTokens,
    completionTokens: outputTokens,
    totalTokens: usage.totalTokens ?? (inputTokens + outputTokens),
    reasoningTokens: usage.reasoningTokens ?? 0
  }
}
```

### 3.5 도구 빌드

```typescript
public buildTools(tools: Tool[]): any[] {
  return tools.map(tool => ({
    type: 'function',
    function: {
      name: tool.name,
      description: tool.description,
      parameters: zodToJsonSchema(tool.inputSchema)
    }
  }))
}
```

---

## 4. Chat Completions API 처리

### 4.1 ChatCompletionsAdapter 구현

**파일**: `/src/services/adapters/chatCompletions.ts`

#### 특징
- 표준 OpenAI Chat Completions API 구현
- 대부분의 OpenAI 호환 모델 지원
- 스트리밍 및 비스트리밍 모드 지원

### 4.2 요청 생성

```typescript
export class ChatCompletionsAdapter extends OpenAIAdapter {
  createRequest(params: UnifiedRequestParams): any {
    const { messages, systemPrompt, tools, maxTokens, stream } = params

    // 메시지 빌드
    const fullMessages = this.buildMessages(systemPrompt, messages)

    // 기본 요청 구조
    const request: any = {
      model: this.modelProfile.modelName,
      messages: fullMessages,
      [this.getMaxTokensParam()]: maxTokens,
      temperature: this.getTemperature()
    }

    // 도구 추가
    if (tools && tools.length > 0) {
      request.tools = this.buildTools(tools)
      request.tool_choice = 'auto'
    }

    // 추론 노력 (O1 모델 등)
    if (this.capabilities.parameters.supportsReasoningEffort && params.reasoningEffort) {
      request.reasoning_effort = params.reasoningEffort
    }

    // 상세도 제어
    if (this.capabilities.parameters.supportsVerbosity && params.verbosity) {
      request.verbosity = params.verbosity
    }

    // 스트리밍 옵션
    if (stream && this.capabilities.streaming.supported) {
      request.stream = true
      if (this.capabilities.streaming.includesUsage) {
        request.stream_options = {
          include_usage: true
        }
      }
    }

    // 모델 특정 제약 사항 적용
    if (this.capabilities.parameters.temperatureMode === 'fixed_one') {
      delete request.temperature
    }

    if (!this.capabilities.streaming.supported) {
      delete request.stream
      delete request.stream_options
    }

    return request
  }
}
```

### 4.3 메시지 빌드

```typescript
private buildMessages(systemPrompt: string[], messages: any[]): any[] {
  // 시스템 메시지 생성
  const systemMessages = systemPrompt.map(prompt => ({
    role: 'system',
    content: prompt
  }))

  // 도구 메시지 정규화
  const normalizedMessages = this.normalizeToolMessages(messages)

  return [...systemMessages, ...normalizedMessages]
}

private normalizeToolMessages(messages: any[]): any[] {
  if (!Array.isArray(messages)) {
    return []
  }

  return messages.map(msg => {
    if (!msg || typeof msg !== 'object') {
      return msg
    }

    // 도구 응답 메시지 정규화
    if (msg.role === 'tool') {
      if (Array.isArray(msg.content)) {
        return {
          ...msg,
          content:
            msg.content
              .map(c => c?.text || '')
              .filter(Boolean)
              .join('\n\n') || '(empty content)',
        }
      } else if (typeof msg.content !== 'string') {
        return {
          ...msg,
          content:
            msg.content === null || msg.content === undefined
              ? '(empty content)'
              : JSON.stringify(msg.content),
        }
      }
    }
    return msg
  })
}
```

### 4.4 비스트리밍 응답 파싱

```typescript
protected parseNonStreamingResponse(response: any): UnifiedResponse {
  // 응답 검증
  if (!response || typeof response !== 'object') {
    throw new Error('Invalid response: response must be an object')
  }

  const choice = response.choices?.[0]
  if (!choice) {
    throw new Error('Invalid response: no choices found in response')
  }

  // 메시지 내용 추출
  const message = choice.message || {}
  const content = typeof message.content === 'string' ? message.content : ''
  const toolCalls = Array.isArray(message.tool_calls) ? message.tool_calls : []

  // 사용량 추출
  const usage = response.usage || {}
  const promptTokens = Number(usage.prompt_tokens) || 0
  const completionTokens = Number(usage.completion_tokens) || 0

  return {
    id: response.id || `chatcmpl_${Date.now()}`,
    content,
    toolCalls,
    usage: {
      promptTokens,
      completionTokens
    }
  }
}
```

### 4.5 스트리밍 청크 처리

```typescript
protected async *processStreamingChunk(
  parsed: any,
  responseId: string,
  hasStarted: boolean,
  accumulatedContent: string,
  reasoningContext?: ReasoningStreamingContext
): AsyncGenerator<StreamingEvent> {
  // 입력 검증
  if (!parsed || typeof parsed !== 'object') {
    return
  }

  // 내용 델타 처리 (Chat Completions 형식)
  const choice = parsed.choices?.[0]
  if (choice?.delta && typeof choice.delta === 'object') {
    const delta = typeof choice.delta.content === 'string' ? choice.delta.content : ''
    const reasoningDelta = typeof choice.delta.reasoning_content === 'string'
      ? choice.delta.reasoning_content
      : ''
    const fullDelta = delta + reasoningDelta

    if (fullDelta) {
      const textEvents = this.handleTextDelta(fullDelta, responseId, hasStarted)
      for (const event of textEvents) {
        yield event
      }
    }
  }

  // 도구 호출 처리 (Chat Completions 형식)
  if (choice?.delta?.tool_calls && Array.isArray(choice.delta.tool_calls)) {
    for (const toolCall of choice.delta.tool_calls) {
      if (toolCall && typeof toolCall === 'object') {
        yield {
          type: 'tool_request',
          tool: {
            id: toolCall.id || `tool_${Date.now()}`,
            name: toolCall.function?.name || 'unknown',
            input: toolCall.function?.arguments || '{}'
          }
        }
      }
    }
  }

  // 사용량 정보 처리
  if (parsed.usage && typeof parsed.usage === 'object') {
    const normalizedUsage = normalizeTokens(parsed.usage)
    this.updateCumulativeUsage(normalizedUsage)
    yield {
      type: 'usage',
      usage: { ...this.cumulativeUsage }
    }
  }
}
```

### 4.6 스트리밍 응답 완전 파싱

```typescript
protected async parseStreamingOpenAIResponse(
  response: any,
  signal?: AbortSignal
): Promise<{ assistantMessage: any; rawResponse: any }> {
  const contentBlocks: any[] = []
  const usage: any = {
    prompt_tokens: 0,
    completion_tokens: 0,
  }

  let responseId = response.id || `chatcmpl_${Date.now()}`
  const pendingToolCalls: any[] = []

  try {
    this.resetCumulativeUsage()

    for await (const event of this.parseStreamingResponse(response)) {
      // 중단 신호 확인
      if (signal?.aborted) {
        throw new Error('Stream aborted by user')
      }

      // 이벤트 타입별 처리
      if (event.type === 'message_start') {
        responseId = event.responseId || responseId
        continue
      }

      if (event.type === 'text_delta') {
        const last = contentBlocks[contentBlocks.length - 1]
        if (!last || last.type !== 'text') {
          contentBlocks.push({ type: 'text', text: event.delta, citations: [] })
        } else {
          last.text += event.delta
        }
        continue
      }

      if (event.type === 'tool_request') {
        pendingToolCalls.push(event.tool)
        continue
      }

      if (event.type === 'usage') {
        usage.prompt_tokens = event.usage.input
        usage.completion_tokens = event.usage.output
        usage.totalTokens = event.usage.total ?? (event.usage.input + event.usage.output)
        usage.promptTokens = event.usage.input
        usage.completionTokens = event.usage.output
        continue
      }
    }
  } catch (error) {
    if (signal?.aborted) {
      // 중단 시 부분 응답 반환
      const assistantMessage = {
        type: 'assistant',
        message: {
          role: 'assistant',
          content: contentBlocks,
          usage: {
            input_tokens: usage.prompt_tokens ?? 0,
            output_tokens: usage.completion_tokens ?? 0,
            prompt_tokens: usage.prompt_tokens ?? 0,
            completion_tokens: usage.completion_tokens ?? 0,
            totalTokens: (usage.prompt_tokens || 0) + (usage.completion_tokens || 0),
          },
        },
        costUSD: 0,
        durationMs: Date.now() - Date.now(),
        uuid: `${Date.now()}-${Math.random().toString(36).slice(2, 11)}` as any,
        responseId,
      }

      return {
        assistantMessage,
        rawResponse: {
          id: responseId,
          content: contentBlocks,
          usage,
          aborted: true,
        },
      }
    }
    throw error
  }

  // 도구 호출 블록 추가
  for (const toolCall of pendingToolCalls) {
    let toolArgs = {}
    try {
      toolArgs = toolCall.input ? JSON.parse(toolCall.input) : {}
    } catch {}

    contentBlocks.push({
      type: 'tool_use',
      id: toolCall.id,
      name: toolCall.name,
      input: toolArgs,
    })
  }

  // 최종 메시지 생성
  const assistantMessage = {
    type: 'assistant',
    message: {
      role: 'assistant',
      content: contentBlocks,
      usage: {
        input_tokens: usage.prompt_tokens ?? 0,
        output_tokens: usage.completion_tokens ?? 0,
        prompt_tokens: usage.prompt_tokens ?? 0,
        completion_tokens: usage.completion_tokens ?? 0,
        totalTokens:
          usage.totalTokens ??
          (usage.prompt_tokens || 0) + (usage.completion_tokens || 0),
      },
    },
    costUSD: 0,
    durationMs: Date.now() - Date.now(),
    uuid: `${Date.now()}-${Math.random().toString(36).slice(2, 11)}` as any,
    responseId,
  }

  return {
    assistantMessage,
    rawResponse: {
      id: responseId,
      content: contentBlocks,
      usage,
    },
  }
}
```

---

## 5. Responses API 처리

### 5.1 ResponsesAPIAdapter 구현

**파일**: `/src/services/adapters/responsesAPI.ts`

#### 특징
- GPT-5 등 신규 Responses API 지원
- 추론 과정(reasoning) 처리
- 상태 연속성(previous_response_id) 지원
- 도구 호출 형식 변환

### 5.2 요청 생성

```typescript
export class ResponsesAPIAdapter extends OpenAIAdapter {
  createRequest(params: UnifiedRequestParams): any {
    const { messages, systemPrompt, tools, maxTokens, reasoningEffort } = params

    // 기본 요청
    const request: any = {
      model: this.modelProfile.modelName,
      input: this.convertMessagesToInput(messages),
      instructions: this.buildInstructions(systemPrompt)
    }

    // 토큰 제한
    const maxTokensField = this.getMaxTokensParam()
    request[maxTokensField] = maxTokens

    // 스트리밍 지원
    request.stream = params.stream !== false && this.capabilities.streaming.supported

    // 온도
    const temperature = this.getTemperature()
    if (temperature !== undefined) {
      request.temperature = temperature
    }

    // 추론 제어
    const include: string[] = []
    if (this.capabilities.parameters.supportsReasoningEffort &&
        (this.shouldIncludeReasoningEffort() || reasoningEffort)) {
      include.push('reasoning.encrypted_content')
      request.reasoning = {
        effort: reasoningEffort || this.modelProfile.reasoningEffort || 'medium'
      }
    }

    // 상세도 제어
    if (this.capabilities.parameters.supportsVerbosity && this.shouldIncludeVerbosity()) {
      let defaultVerbosity: 'low' | 'medium' | 'high' = 'medium'
      if (params.verbosity) {
        defaultVerbosity = params.verbosity
      } else {
        const modelNameLower = this.modelProfile.modelName.toLowerCase()
        if (modelNameLower.includes('high')) {
          defaultVerbosity = 'high'
        } else if (modelNameLower.includes('low')) {
          defaultVerbosity = 'low'
        }
      }

      request.text = {
        verbosity: defaultVerbosity
      }
    }

    // 도구
    if (tools && tools.length > 0) {
      request.tools = this.buildTools(tools)
    }

    request.tool_choice = 'auto'

    // 병렬 도구 호출
    if (this.capabilities.toolCalling.supportsParallelCalls) {
      request.parallel_tool_calls = true
    }

    request.store = false

    // 상태 관리
    if (params.previousResponseId &&
        this.capabilities.stateManagement.supportsPreviousResponseId) {
      request.previous_response_id = params.previousResponseId
    }

    if (include.length > 0) {
      request.include = include
    }

    return request
  }
}
```

### 5.3 메시지 형식 변환

**Chat Completions → Responses API 형식 변환**

```typescript
private convertMessagesToInput(messages: any[]): any[] {
  const inputItems = []

  for (const message of messages) {
    const role = message.role

    // 도구 응답 처리
    if (role === 'tool') {
      const callId = message.tool_call_id || message.id
      if (typeof callId === 'string' && callId) {
        let content = message.content || ''
        if (Array.isArray(content)) {
          const texts = []
          for (const part of content) {
            if (typeof part === 'object' && part !== null) {
              const t = part.text || part.content
              if (typeof t === 'string' && t) {
                texts.push(t)
              }
            }
          }
          content = texts.join('\n')
        }
        if (typeof content === 'string') {
          inputItems.push({
            type: 'function_call_output',
            call_id: callId,
            output: content
          })
        }
      }
      continue
    }

    // 어시스턴트 도구 호출 처리
    if (role === 'assistant' && Array.isArray(message.tool_calls)) {
      for (const tc of message.tool_calls) {
        if (typeof tc !== 'object' || tc === null) {
          continue
        }
        const tcType = tc.type || 'function'
        if (tcType !== 'function') {
          continue
        }
        const callId = tc.id || tc.call_id
        const fn = tc.function
        const name = typeof fn === 'object' && fn !== null ? fn.name : null
        const args = typeof fn === 'object' && fn !== null ? fn.arguments : null

        if (typeof callId === 'string' && typeof name === 'string' && typeof args === 'string') {
          inputItems.push({
            type: 'function_call',
            name: name,
            arguments: args,
            call_id: callId
          })
        }
      }
      continue
    }

    // 일반 텍스트 내용 처리
    const content = message.content || ''
    const contentItems = []

    if (Array.isArray(content)) {
      for (const part of content) {
        if (typeof part !== 'object' || part === null) continue
        const ptype = part.type
        if (ptype === 'text') {
          const text = part.text || part.content || ''
          if (typeof text === 'string' && text) {
            const kind = role === 'assistant' ? 'output_text' : 'input_text'
            contentItems.push({ type: kind, text: text })
          }
        } else if (ptype === 'image_url') {
          const image = part.image_url
          const url = typeof image === 'object' && image !== null ? image.url : image
          if (typeof url === 'string' && url) {
            contentItems.push({ type: 'input_image', image_url: url })
          }
        }
      }
    } else if (typeof content === 'string' && content) {
      const kind = role === 'assistant' ? 'output_text' : 'input_text'
      contentItems.push({ type: kind, text: content })
    }

    if (contentItems.length) {
      const roleOut = role === 'assistant' ? 'assistant' : 'user'
      inputItems.push({ type: 'message', role: roleOut, content: contentItems })
    }
  }

  return inputItems
}
```

### 5.4 도구 빌드 (Responses API 형식)

```typescript
buildTools(tools: Tool[]): any[] {
  // 평면 구조, 중첩된 'function' 객체 없음
  return tools.map(tool => {
    let parameters = tool.inputJSONSchema

    // Zod 스키마 변환
    if (!parameters && tool.inputSchema) {
      const isPlainObject = (obj: any): boolean => {
        return obj !== null && typeof obj === 'object' && !Array.isArray(obj)
      }

      if (isPlainObject(tool.inputSchema) &&
          ('type' in tool.inputSchema || 'properties' in tool.inputSchema)) {
        parameters = tool.inputSchema
      } else {
        try {
          parameters = zodToJsonSchema(tool.inputSchema)
        } catch (error) {
          console.warn(`Failed to convert Zod schema for tool ${tool.name}:`, error)
          parameters = { type: 'object', properties: {} }
        }
      }
    }

    return {
      type: 'function',
      name: tool.name,
      description: getToolDescription(tool),
      parameters: (parameters as any) || { type: 'object', properties: {} }
    }
  })
}
```

### 5.5 스트리밍 청크 처리 (Responses API)

```typescript
protected async *processStreamingChunk(
  parsed: any,
  responseId: string,
  hasStarted: boolean,
  accumulatedContent: string,
  reasoningContext?: ReasoningStreamingContext
): AsyncGenerator<StreamingEvent> {
  // 추론 요약 파트 이벤트 처리
  if (parsed.type === 'response.reasoning_summary_part.added') {
    const partIndex = parsed.summary_index || 0

    if (!reasoningContext?.thinkingContent) {
      reasoningContext!.thinkingContent = ''
      reasoningContext!.currentPartIndex = -1
    }

    reasoningContext!.currentPartIndex = partIndex

    // 파트 간 구분자
    if (partIndex > 0 && reasoningContext!.thinkingContent) {
      reasoningContext!.thinkingContent += '\n\n'

      yield {
        type: 'text_delta',
        delta: '\n\n',
        responseId
      }
    }

    return
  }

  // 추론 요약 텍스트 델타
  if (parsed.type === 'response.reasoning_summary_text.delta') {
    const delta = parsed.delta || ''

    if (delta && reasoningContext) {
      reasoningContext.thinkingContent += delta

      yield {
        type: 'text_delta',
        delta,
        responseId
      }
    }

    return
  }

  // 추론 텍스트 델타
  if (parsed.type === 'response.reasoning_text.delta') {
    const delta = parsed.delta || ''

    if (delta && reasoningContext) {
      reasoningContext.thinkingContent += delta

      yield {
        type: 'text_delta',
        delta,
        responseId
      }
    }

    return
  }

  // 텍스트 내용 델타 (Responses API 형식)
  if (parsed.type === 'response.output_text.delta') {
    const delta = parsed.delta || ''
    if (delta) {
      const textEvents = this.handleTextDelta(delta, responseId, hasStarted)
      for (const event of textEvents) {
        yield event
      }
    }
  }

  // 도구 호출 (Responses API 형식)
  if (parsed.type === 'response.output_item.done') {
    const item = parsed.item || {}
    if (item.type === 'function_call') {
      const callId = item.call_id || item.id
      const name = item.name
      const args = item.arguments

      if (typeof callId === 'string' && typeof name === 'string' && typeof args === 'string') {
        yield {
          type: 'tool_request',
          tool: {
            id: callId,
            name: name,
            input: args
          }
        }
      }
    }
  }

  // 사용량 정보
  if (parsed.usage) {
    const normalizedUsage = normalizeTokens(parsed.usage)

    // 추론 토큰 추가
    if (parsed.usage.output_tokens_details?.reasoning_tokens) {
      normalizedUsage.reasoning = parsed.usage.output_tokens_details.reasoning_tokens
    }

    yield {
      type: 'usage',
      usage: normalizedUsage
    }
  }
}
```

### 5.6 비스트리밍 응답 파싱

```typescript
protected parseNonStreamingResponse(response: any): UnifiedResponse {
  let content = response.output_text || ''

  // 추론 내용 추출
  let reasoningContent = ''
  if (response.output && Array.isArray(response.output)) {
    const messageItems = response.output.filter(item => item.type === 'message')
    if (messageItems.length > 0) {
      content = messageItems
        .map(item => {
          if (item.content && Array.isArray(item.content)) {
            return item.content
              .filter(c => c.type === 'text')
              .map(c => c.text)
              .join('\n')
          }
          return item.content || ''
        })
        .filter(Boolean)
        .join('\n\n')
    }

    // 추론 내용 추출
    const reasoningItems = response.output.filter(item => item.type === 'reasoning')
    if (reasoningItems.length > 0) {
      reasoningContent = reasoningItems
        .map(item => item.content || '')
        .filter(Boolean)
        .join('\n\n')
    }
  }

  // 추론 포맷 적용
  if (reasoningContent) {
    const thinkBlock = `

${reasoningContent}

`
    content = thinkBlock + content
  }

  // 도구 호출 파싱
  const toolCalls = this.parseToolCalls(response)

  // 통합 응답 빌드
  const contentArray = content
    ? [{ type: 'text', text: content, citations: [] }]
    : [{ type: 'text', text: '', citations: [] }]

  const promptTokens = response.usage?.input_tokens || 0
  const completionTokens = response.usage?.output_tokens || 0
  const totalTokens = response.usage?.total_tokens ?? (promptTokens + completionTokens)

  return {
    id: response.id || `resp_${Date.now()}`,
    content: contentArray,
    toolCalls,
    usage: {
      promptTokens,
      completionTokens,
      reasoningTokens: response.usage?.output_tokens_details?.reasoning_tokens
    },
    responseId: response.id
  }
}

private parseToolCalls(response: any): any[] {
  if (!response.output || !Array.isArray(response.output)) {
    return []
  }

  const toolCalls = []

  for (const item of response.output) {
    if (item.type === 'function_call') {
      const callId = item.call_id || item.id
      const name = item.name || ''
      const args = item.arguments || '{}'

      if (typeof callId === 'string' && typeof name === 'string' && typeof args === 'string') {
        toolCalls.push({
          id: callId,
          type: 'function',
          function: {
            name: name,
            arguments: args
          }
        })
      }
    }
  }

  return toolCalls
}
```

---

## 6. 스트리밍 응답 처리

### 6.1 processResponsesStream 함수

**파일**: `/src/services/adapters/responsesStreaming.ts`

#### 역할
- 스트리밍 이벤트를 AssistantMessage로 변환
- 도구 호출 블록 조립
- 사용량 정규화

```typescript
export async function processResponsesStream(
  stream: AsyncGenerator<StreamingEvent>,
  startTime: number,
  fallbackResponseId: string,
): Promise<{ assistantMessage: AssistantMessage; rawResponse: any }> {
  const contentBlocks: any[] = []
  const usage: any = {
    prompt_tokens: 0,
    completion_tokens: 0,
  }

  let responseId = fallbackResponseId
  const pendingToolCalls: any[] = []

  // 스트리밍 이벤트 처리
  for await (const event of stream) {
    if (event.type === 'message_start') {
      responseId = event.responseId || responseId
      continue
    }

    if (event.type === 'text_delta') {
      const last = contentBlocks[contentBlocks.length - 1]
      if (!last || last.type !== 'text') {
        contentBlocks.push({ type: 'text', text: event.delta, citations: [] })
      } else {
        last.text += event.delta
      }
      continue
    }

    if (event.type === 'tool_request') {
      pendingToolCalls.push(event.tool)
      continue
    }

    if (event.type === 'usage') {
      // 사용량은 정규화된 형식으로 전달됨
      usage.prompt_tokens = event.usage.input
      usage.completion_tokens = event.usage.output
      usage.promptTokens = event.usage.input
      usage.completionTokens = event.usage.output
      usage.totalTokens = event.usage.total ?? (event.usage.input + event.usage.output)
      if (event.usage.reasoning !== undefined) {
        usage.reasoningTokens = event.usage.reasoning
      }
      continue
    }
  }

  // 도구 호출 블록 추가
  for (const toolCall of pendingToolCalls) {
    let toolArgs = {}
    try {
      toolArgs = toolCall.input ? JSON.parse(toolCall.input) : {}
    } catch {}

    contentBlocks.push({
      type: 'tool_use',
      id: toolCall.id,
      name: toolCall.name,
      input: toolArgs,
    })
  }

  // AssistantMessage 생성
  const assistantMessage: AssistantMessage = {
    type: 'assistant',
    message: {
      role: 'assistant',
      content: contentBlocks,
      usage: {
        input_tokens: usage.prompt_tokens ?? 0,
        output_tokens: usage.completion_tokens ?? 0,
        prompt_tokens: usage.prompt_tokens ?? 0,
        completion_tokens: usage.completion_tokens ?? 0,
        totalTokens:
          usage.totalTokens ??
          (usage.prompt_tokens || 0) + (usage.completion_tokens || 0),
        reasoningTokens: usage.reasoningTokens,
      },
    },
    costUSD: 0,
    durationMs: Date.now() - startTime,
    uuid: `${Date.now()}-${Math.random().toString(36).slice(2, 11)}` as any,
    responseId,
  }

  return {
    assistantMessage,
    rawResponse: {
      id: responseId,
      content: contentBlocks,
      usage,
    },
  }
}
```

### 6.2 스트리밍 처리 흐름

```
SSE Stream → parseSSEChunk → processStreamingChunk
    ↓
StreamingEvent 생성
    ↓
├── message_start: 응답 시작 신호
├── text_delta: 텍스트 조각
├── tool_request: 도구 호출
├── usage: 토큰 사용량
└── message_stop: 응답 완료

processResponsesStream 처리
    ↓
└── AssistantMessage 생성
```

---

## 7. 모델 어댑터 팩토리

### 7.1 ModelAdapterFactory 클래스

**파일**: `/src/services/modelAdapterFactory.ts`

#### 역할
- 모델 프로필 기반 어댑터 생성
- API 타입 자동 결정
- 모델 capabilities 활용

```typescript
export class ModelAdapterFactory {
  /**
   * 모델 설정 기반 적절한 어댑터 생성
   */
  static createAdapter(modelProfile: ModelProfile): ModelAPIAdapter {
    const capabilities = getModelCapabilities(modelProfile.modelName)

    // 사용할 API 결정
    const apiType = this.determineAPIType(modelProfile, capabilities)

    // 해당 어댑터 생성
    switch (apiType) {
      case 'responses_api':
        return new ResponsesAPIAdapter(capabilities, modelProfile)
      case 'chat_completions':
      default:
        return new ChatCompletionsAdapter(capabilities, modelProfile)
    }
  }

  /**
   * 사용할 API 결정
   */
  private static determineAPIType(
    modelProfile: ModelProfile,
    capabilities: ModelCapabilities
  ): 'responses_api' | 'chat_completions' {
    // Responses API 미지원 모델은 Chat Completions 사용
    if (capabilities.apiArchitecture.primary !== 'responses_api') {
      return 'chat_completions'
    }

    // 공식 OpenAI 엔드포인트 확인
    const isOfficialOpenAI = !modelProfile.baseURL ||
      modelProfile.baseURL.includes('api.openai.com')

    // 비공식 엔드포인트는 fallback 옵션 확인
    if (!isOfficialOpenAI) {
      if (capabilities.apiArchitecture.fallback === 'chat_completions') {
        return capabilities.apiArchitecture.primary  // primary 사용
      }
      return capabilities.apiArchitecture.primary
    }

    // 공식 엔드포인트는 primary API 사용
    return capabilities.apiArchitecture.primary
  }

  /**
   * Responses API 사용 여부 확인
   */
  static shouldUseResponsesAPI(modelProfile: ModelProfile): boolean {
    const capabilities = getModelCapabilities(modelProfile.modelName)
    const apiType = this.determineAPIType(modelProfile, capabilities)
    return apiType === 'responses_api'
  }
}
```

### 7.2 사용 예제

```typescript
// 모델 프로필로 어댑터 생성
const modelProfile = modelManager.getModel('main')
const adapter = ModelAdapterFactory.createAdapter(modelProfile)

// 요청 생성
const request = adapter.createRequest({
  messages: openaiMessages,
  systemPrompt: systemPrompt,
  tools: tools,
  maxTokens: 8000,
  stream: true,
  reasoningEffort: 'medium'
})

// API 호출
const response = await callAPI(modelProfile, request)

// 응답 파싱
const unifiedResponse = await adapter.parseResponse(response)
```

---

## 8. 응답 상태 관리

### 8.1 ResponseStateManager 클래스

**파일**: `/src/services/responseStateManager.ts`

#### 역할
- GPT-5 Responses API의 `previous_response_id` 관리
- 대화 연속성 유지
- 추론 컨텍스트 재사용

```typescript
interface ConversationState {
  previousResponseId?: string
  lastUpdate: number
}

class ResponseStateManager {
  private conversationStates = new Map<string, ConversationState>()

  // 1시간 후 자동 정리
  private readonly CLEANUP_INTERVAL = 60 * 60 * 1000

  constructor() {
    // 주기적 정리
    setInterval(() => {
      this.cleanup()
    }, this.CLEANUP_INTERVAL)
  }

  /**
   * 대화의 이전 응답 ID 설정
   */
  setPreviousResponseId(conversationId: string, responseId: string): void {
    this.conversationStates.set(conversationId, {
      previousResponseId: responseId,
      lastUpdate: Date.now()
    })
  }

  /**
   * 대화의 이전 응답 ID 조회
   */
  getPreviousResponseId(conversationId: string): string | undefined {
    const state = this.conversationStates.get(conversationId)
    if (state) {
      // 마지막 접근 시간 업데이트
      state.lastUpdate = Date.now()
      return state.previousResponseId
    }
    return undefined
  }

  /**
   * 대화 상태 삭제
   */
  clearConversation(conversationId: string): void {
    this.conversationStates.delete(conversationId)
  }

  /**
   * 모든 대화 상태 삭제
   */
  clearAll(): void {
    this.conversationStates.clear()
  }

  /**
   * 오래된 대화 정리
   */
  private cleanup(): void {
    const now = Date.now()
    for (const [conversationId, state] of this.conversationStates.entries()) {
      if (now - state.lastUpdate > this.CLEANUP_INTERVAL) {
        this.conversationStates.delete(conversationId)
      }
    }
  }

  /**
   * 현재 상태 크기 (디버깅/모니터링용)
   */
  getStateSize(): number {
    return this.conversationStates.size
  }
}

// 싱글톤 인스턴스
export const responseStateManager = new ResponseStateManager()

/**
 * 컨텍스트에서 대화 ID 생성
 */
export function getConversationId(agentId?: string, messageId?: string): string {
  return agentId || messageId || `conv_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
}
```

### 8.2 사용 흐름

```typescript
// 1. 도구 사용 컨텍스트 초기화
if (toolUseContext && !toolUseContext.responseState) {
  const conversationId = getConversationId(
    toolUseContext.agentId,
    toolUseContext.messageId
  )
  const previousResponseId = responseStateManager.getPreviousResponseId(conversationId)

  toolUseContext.responseState = {
    previousResponseId,
    conversationId
  }
}

// 2. 요청에 previous_response_id 포함
const request = adapter.createRequest({
  ...params,
  previousResponseId: toolUseContext?.responseState?.previousResponseId
})

// 3. 응답 받은 후 상태 업데이트
if (toolUseContext?.responseState?.conversationId && result.responseId) {
  responseStateManager.setPreviousResponseId(
    toolUseContext.responseState.conversationId,
    result.responseId
  )
}
```

### 8.3 상태 관리 다이어그램

```
┌──────────────────────────────────────────────────┐
│ 첫 번째 요청                                      │
│  previousResponseId: undefined                   │
└────────────────┬─────────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────────┐
│ 응답: responseId = "resp_abc123"                 │
└────────────────┬─────────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────────┐
│ ResponseStateManager.setPreviousResponseId()     │
│  conversationId → "resp_abc123"                  │
└────────────────┬─────────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────────┐
│ 두 번째 요청                                      │
│  previousResponseId: "resp_abc123"               │
│  → 추론 컨텍스트 재사용                           │
└──────────────────────────────────────────────────┘
```

---

## 9. OAuth 인증 시스템

### 9.1 OAuthService 클래스

**파일**: `/src/services/oauth.ts`

#### 특징
- PKCE (Proof Key for Code Exchange) 플로우 구현
- 로컬 서버를 통한 콜백 처리
- 수동 리디렉션 지원
- API 키 자동 생성 및 저장

### 9.2 PKCE 플로우 구현

```typescript
// Base64URL 인코딩
function base64URLEncode(buffer: Buffer): string {
  return buffer
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '')
}

// Code Verifier 생성 (32바이트 랜덤)
function generateCodeVerifier(): string {
  return base64URLEncode(crypto.randomBytes(32))
}

// Code Challenge 생성 (SHA-256 해시)
async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder()
  const data = encoder.encode(verifier)
  const digest = await crypto.subtle.digest('SHA-256', data)
  return base64URLEncode(Buffer.from(digest))
}
```

### 9.3 OAuth 플로우

```typescript
export class OAuthService {
  private server: http.Server | null = null
  private codeVerifier: string
  private expectedState: string | null = null
  private pendingCodePromise: {
    resolve: (result: { authorizationCode: string; useManualRedirect: boolean }) => void
    reject: (err: Error) => void
  } | null = null

  constructor() {
    this.codeVerifier = generateCodeVerifier()
  }

  async startOAuthFlow(
    authURLHandler: (url: string) => Promise<void>
  ): Promise<OAuthResult> {
    // 1. Code Challenge 생성
    const codeChallenge = await generateCodeChallenge(this.codeVerifier)
    const state = base64URLEncode(crypto.randomBytes(32))
    this.expectedState = state

    // 2. 인증 URL 생성
    const { autoUrl, manualUrl } = this.generateAuthUrls(codeChallenge, state)

    // 3. 로컬 서버 시작 및 브라우저 열기
    const onReady = async () => {
      await authURLHandler(manualUrl)
      await openBrowser(autoUrl)
    }

    // 4. Authorization Code 대기
    const { authorizationCode, useManualRedirect } = await new Promise((resolve, reject) => {
      this.pendingCodePromise = { resolve, reject }
      this.startLocalServer(state, onReady)
    })

    // 5. Code를 Token으로 교환
    const { access_token: accessToken, account, organization } =
      await this.exchangeCodeForTokens(authorizationCode, state, useManualRedirect)

    // 6. 계정 정보 저장
    if (account) {
      const accountInfo: AccountInfo = {
        accountUuid: account.uuid,
        emailAddress: account.email_address,
        organizationUuid: organization?.uuid,
      }
      const config = getGlobalConfig()
      config.oauthAccount = accountInfo
      saveGlobalConfig(config)
    }

    return { accessToken }
  }
}
```

### 9.4 로컬 서버 콜백 처리

```typescript
private startLocalServer(state: string, onReady?: () => void): void {
  if (this.server) {
    this.closeServer()
  }

  this.server = http.createServer((req: IncomingMessage, res: ServerResponse) => {
    const parsedUrl = url.parse(req.url || '', true)

    if (parsedUrl.pathname === '/callback') {
      const authorizationCode = parsedUrl.query.code as string
      const returnedState = parsedUrl.query.state as string

      // Authorization Code 검증
      if (!authorizationCode) {
        res.writeHead(400)
        res.end('Authorization code not found')
        if (this.pendingCodePromise) {
          this.pendingCodePromise.reject(new Error('No authorization code received'))
        }
        return
      }

      // State 파라미터 검증 (CSRF 방지)
      if (returnedState !== state) {
        res.writeHead(400)
        res.end('Invalid state parameter')
        if (this.pendingCodePromise) {
          this.pendingCodePromise.reject(new Error('Invalid state parameter'))
        }
        return
      }

      // 성공 페이지로 리디렉션
      res.writeHead(302, {
        Location: OAUTH_CONFIG.SUCCESS_URL,
      })
      res.end()

      // 콜백 처리
      this.processCallback({
        authorizationCode,
        state,
        useManualRedirect: false,
      })
    } else {
      res.writeHead(404)
      res.end()
    }
  })

  this.server.listen(OAUTH_CONFIG.REDIRECT_PORT, async () => {
    onReady?.()
  })

  this.server.on('error', (err: Error) => {
    // 포트 충돌 처리
    const portError = err as NodeJS.ErrnoException
    if (portError.code === 'EADDRINUSE') {
      const error = new Error(
        `Port ${OAUTH_CONFIG.REDIRECT_PORT} is already in use.`
      )
      this.closeServer()
      if (this.pendingCodePromise) {
        this.pendingCodePromise.reject(error)
      }
    }
  })
}
```

### 9.5 Token 교환

```typescript
private async exchangeCodeForTokens(
  authorizationCode: string,
  state: string,
  useManualRedirect: boolean = false,
): Promise<OAuthTokenExchangeResponse> {
  const requestBody = {
    grant_type: 'authorization_code',
    code: authorizationCode,
    redirect_uri: useManualRedirect
      ? OAUTH_CONFIG.MANUAL_REDIRECT_URL
      : `http://localhost:${OAUTH_CONFIG.REDIRECT_PORT}/callback`,
    client_id: OAUTH_CONFIG.CLIENT_ID,
    code_verifier: this.codeVerifier,  // PKCE 검증
    state,
  }

  const response = await fetch(OAUTH_CONFIG.TOKEN_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(requestBody),
  })

  if (!response.ok) {
    throw new Error(`Token exchange failed: ${response.statusText}`)
  }

  const data = await response.json()
  return data
}
```

### 9.6 API 키 생성 및 저장

```typescript
export async function createAndStoreApiKey(
  accessToken: string,
): Promise<string | null> {
  try {
    // API 키 생성
    const createApiKeyResp = await fetch(OAUTH_CONFIG.API_KEY_URL, {
      method: 'POST',
      headers: { Authorization: `Bearer ${accessToken}` },
    })

    const apiKeyData = await createApiKeyResp.json()

    if (createApiKeyResp.ok && apiKeyData && apiKeyData.raw_key) {
      const apiKey = apiKeyData.raw_key

      // 전역 설정에 저장
      const config = getGlobalConfig()

      // 승인된 API 키 목록에 추가
      if (!config.customApiKeyResponses) {
        config.customApiKeyResponses = { approved: [], rejected: [] }
      }

      const normalizedKey = normalizeApiKeyForConfig(apiKey)
      if (!config.customApiKeyResponses.approved.includes(normalizedKey)) {
        config.customApiKeyResponses.approved.push(normalizedKey)
      }

      // 설정 저장
      saveGlobalConfig(config)

      // Anthropic 클라이언트 재설정
      resetAnthropicClient()

      return apiKey
    }

    return null
  } catch (error) {
    throw error
  }
}
```

### 9.7 OAuth 플로우 다이어그램

```
┌──────────────┐
│ OAuth Start  │
└──────┬───────┘
       ▼
┌─────────────────────┐
│ Generate:           │
│ • Code Verifier     │
│ • Code Challenge    │
│ • State             │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Start Local Server  │
│ (Port 3000)         │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Open Browser        │
│ → Auth URL          │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ User Authorizes     │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Callback:           │
│ • code              │
│ • state             │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Validate State      │
│ (CSRF Protection)   │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Exchange Code       │
│ for Access Token    │
│ (with Verifier)     │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Create API Key      │
└──────┬──────────────┘
       ▼
┌─────────────────────┐
│ Save to Config      │
└─────────────────────┘
```

---

## 10. 통합 서비스 레이어

### 10.1 OpenAI 서비스

**파일**: `/src/services/openai.ts`

#### 주요 기능
- OpenAI 호환 API 호출
- 재시도 메커니즘
- 에러 처리 및 복구
- GPT-5 특수 처리

#### 재시도 메커니즘

```typescript
const RETRY_CONFIG = {
  BASE_DELAY_MS: 1000,
  MAX_DELAY_MS: 32000,
  MAX_SERVER_DELAY_MS: 60000,
  JITTER_FACTOR: 0.1,
} as const

function getRetryDelay(attempt: number, retryAfter?: string | null): number {
  // 서버 지정 재시도 시간
  if (retryAfter) {
    const retryAfterMs = parseInt(retryAfter) * 1000
    if (!isNaN(retryAfterMs) && retryAfterMs > 0) {
      return Math.min(retryAfterMs, RETRY_CONFIG.MAX_SERVER_DELAY_MS)
    }
  }

  // 지수 백오프 + 지터
  const delay = RETRY_CONFIG.BASE_DELAY_MS * Math.pow(2, attempt - 1)
  const jitter = Math.random() * RETRY_CONFIG.JITTER_FACTOR * delay

  return Math.min(delay + jitter, RETRY_CONFIG.MAX_DELAY_MS)
}
```

#### 중단 가능한 지연

```typescript
function abortableDelay(delayMs: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    // 이미 중단된 경우
    if (signal?.aborted) {
      reject(new Error('Request was aborted'))
      return
    }

    const timeoutId = setTimeout(() => {
      resolve()
    }, delayMs)

    // 중단 신호 리스너
    if (signal) {
      const abortHandler = () => {
        clearTimeout(timeoutId)
        reject(new Error('Request was aborted'))
      }
      signal.addEventListener('abort', abortHandler, { once: true })
    }
  })
}
```

#### 에러 핸들러

```typescript
// GPT-5 특정 에러 핸들러
const GPT5_ERROR_HANDLERS: ErrorHandler[] = [
  {
    type: ModelErrorType.MaxCompletionTokens,
    detect: errMsg => {
      const lowerMsg = errMsg.toLowerCase()
      return (
        (lowerMsg.includes("unsupported parameter: 'max_tokens'") &&
         lowerMsg.includes("'max_completion_tokens'")) ||
        (lowerMsg.includes("max_tokens") && lowerMsg.includes("max_completion_tokens"))
      )
    },
    fix: async opts => {
      console.log(`🔧 GPT-5 Fix: Converting max_tokens to max_completion_tokens`)
      if ('max_tokens' in opts) {
        opts.max_completion_tokens = opts.max_tokens
        delete opts.max_tokens
      }
    },
  },
  {
    type: ModelErrorType.TemperatureRestriction,
    detect: errMsg => {
      const lowerMsg = errMsg.toLowerCase()
      return (
        lowerMsg.includes("temperature") &&
        (lowerMsg.includes("only supports") || lowerMsg.includes("must be 1"))
      )
    },
    fix: async opts => {
      console.log(`🔧 GPT-5 Fix: Adjusting temperature to 1`)
      opts.temperature = 1
    },
  },
]
```

#### 모델 특성 감지 및 변환

```typescript
function applyModelSpecificTransformations(
  opts: OpenAI.ChatCompletionCreateParams,
): void {
  if (!opts.model || typeof opts.model !== 'string') {
    return
  }

  const features = getModelFeatures(opts.model)
  const isGPT5 = opts.model.toLowerCase().includes('gpt-5')

  // GPT-5 변환
  if (isGPT5 || features.usesMaxCompletionTokens) {
    // max_completion_tokens 강제 사용
    if ('max_tokens' in opts && !('max_completion_tokens' in opts)) {
      console.log(`🔧 Transforming max_tokens to max_completion_tokens for ${opts.model}`)
      opts.max_completion_tokens = opts.max_tokens
      delete opts.max_tokens
    }

    // temperature = 1 강제
    if (features.requiresTemperatureOne && 'temperature' in opts) {
      if (opts.temperature !== 1 && opts.temperature !== undefined) {
        console.log(`🔧 Adjusting temperature to 1 for ${opts.model}`)
        opts.temperature = 1
      }
    }

    // 미지원 파라미터 제거
    if (isGPT5) {
      delete opts.frequency_penalty
      delete opts.presence_penalty
      delete opts.logit_bias
      delete opts.user

      // reasoning_effort 추가
      if (!opts.reasoning_effort && features.supportsVerbosityControl) {
        opts.reasoning_effort = 'medium'
      }
    }
  }
}
```

#### 엔드포인트 폴백

```typescript
async function tryWithEndpointFallback(
  baseURL: string,
  opts: OpenAI.ChatCompletionCreateParams,
  headers: Record<string, string>,
  provider: string,
  proxy: any,
  signal?: AbortSignal,
): Promise<{ response: Response; endpoint: string }> {
  const endpointsToTry = []

  if (provider === 'minimax') {
    endpointsToTry.push('/text/chatcompletion_v2', '/chat/completions')
  } else {
    endpointsToTry.push('/chat/completions')
  }

  let lastError = null

  for (const endpoint of endpointsToTry) {
    try {
      const response = await fetch(`${baseURL}${endpoint}`, {
        method: 'POST',
        headers,
        body: JSON.stringify(opts.stream ? { ...opts, stream: true } : opts),
        dispatcher: proxy,
        signal: signal,
      })

      // 성공 시 즉시 반환
      if (response.ok) {
        return { response, endpoint }
      }

      // 404는 다음 엔드포인트 시도
      if (response.status === 404 && endpointsToTry.length > 1) {
        console.log(`Endpoint ${endpoint} returned 404, trying next...`)
        continue
      }

      // 다른 에러 코드는 즉시 반환
      return { response, endpoint }
    } catch (error) {
      lastError = error
      // 네트워크 에러는 다음 엔드포인트 시도
      if (endpointsToTry.indexOf(endpoint) < endpointsToTry.length - 1) {
        console.log(`Network error on ${endpoint}, trying next...`)
        continue
      }
    }
  }

  throw lastError || new Error('All endpoints failed')
}
```

#### 메인 호출 함수

```typescript
export async function getCompletionWithProfile(
  modelProfile: any,
  opts: OpenAI.ChatCompletionCreateParams,
  attempt: number = 0,
  maxAttempts: number = 10,
  signal?: AbortSignal,
): Promise<OpenAI.ChatCompletion | AsyncIterable<OpenAI.ChatCompletionChunk>> {
  if (attempt >= maxAttempts) {
    throw new Error('Max attempts reached')
  }

  const provider = modelProfile?.provider || 'anthropic'
  const baseURL = modelProfile?.baseURL
  const apiKey = modelProfile?.apiKey
  const proxy = getGlobalConfig().proxy
    ? new ProxyAgent(getGlobalConfig().proxy)
    : undefined

  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
  }

  if (apiKey) {
    if (provider === 'azure') {
      headers['api-key'] = apiKey
    } else {
      headers['Authorization'] = `Bearer ${apiKey}`
    }
  }

  // 모델 특정 변환 적용
  applyModelSpecificTransformations(opts)
  await applyModelErrorFixes(opts, baseURL || '')

  // 디버그 로깅
  debugLogger.api('OPENAI_API_CALL_START', {
    endpoint: baseURL || 'DEFAULT_OPENAI',
    model: opts.model,
    provider,
    apiKeyConfigured: !!apiKey,
    streamMode: opts.stream,
    timestamp: new Date().toISOString(),
  })

  // 도구 메시지 정규화
  opts.messages = opts.messages.map(msg => {
    if (msg.role === 'tool') {
      if (Array.isArray(msg.content)) {
        return {
          ...msg,
          content:
            msg.content
              .map(c => c.text || '')
              .filter(Boolean)
              .join('\n\n') || '(empty content)',
        }
      } else if (typeof msg.content !== 'string') {
        return {
          ...msg,
          content:
            typeof msg.content === 'undefined'
              ? '(empty content)'
              : JSON.stringify(msg.content),
        }
      }
    }
    return msg
  })

  try {
    if (opts.stream) {
      // 스트리밍 요청
      const isOpenAICompatible = [
        'minimax', 'kimi', 'deepseek', 'siliconflow',
        'qwen', 'glm', 'openai', 'mistral', 'xai', 'groq'
      ].includes(provider)

      let response: Response
      let usedEndpoint: string

      if (isOpenAICompatible && provider !== 'azure') {
        const result = await tryWithEndpointFallback(
          baseURL, opts, headers, provider, proxy, signal
        )
        response = result.response
        usedEndpoint = result.endpoint
      } else {
        response = await fetch(`${baseURL}/chat/completions`, {
          method: 'POST',
          headers,
          body: JSON.stringify({ ...opts, stream: true }),
          dispatcher: proxy,
          signal: signal,
        })
        usedEndpoint = '/chat/completions'
      }

      if (!response.ok) {
        // 중단 확인
        if (signal?.aborted) {
          throw new Error('Request cancelled by user')
        }

        // 에러 파싱 및 처리
        try {
          const errorData = await response.json()
          const errorMessage = errorData.error?.message || errorData.message || `HTTP ${response.status}`

          // 에러 핸들러 적용
          const isGPT5 = opts.model.startsWith('gpt-5')
          const handlers = isGPT5 ? [...GPT5_ERROR_HANDLERS, ...ERROR_HANDLERS] : ERROR_HANDLERS

          for (const handler of handlers) {
            if (handler.detect(errorMessage)) {
              console.log(`🔧 Detected ${handler.type} error: ${errorMessage}`)

              // 에러 저장
              setModelError(baseURL || '', opts.model, handler.type, errorMessage)

              // 수정 적용 후 재시도
              await handler.fix(opts)
              console.log(`🔧 Applied fix for ${handler.type}, retrying...`)

              return getCompletionWithProfile(
                modelProfile, opts, attempt + 1, maxAttempts, signal
              )
            }
          }

          logAPIError({
            model: opts.model,
            endpoint: `${baseURL}${usedEndpoint}`,
            status: response.status,
            error: errorMessage,
            request: opts,
            response: errorData,
            provider: provider
          })
        } catch (parseError) {
          console.log(`⚠️  Could not parse error response (${response.status})`)
        }

        // 재시도
        const delayMs = getRetryDelay(attempt)
        console.log(`  ⎿  API error (${response.status}), retrying in ${Math.round(delayMs / 1000)}s...`)

        try {
          await abortableDelay(delayMs, signal)
        } catch (error) {
          if (error.message === 'Request was aborted') {
            throw new Error('Request cancelled by user')
          }
          throw error
        }

        return getCompletionWithProfile(
          modelProfile, opts, attempt + 1, maxAttempts, signal
        )
      }

      const stream = createStreamProcessor(response.body as any, signal)
      return stream
    }

    // 비스트리밍 요청 (유사한 로직)
    // ...
  } catch (error) {
    if (signal?.aborted) {
      throw new Error('Request cancelled by user')
    }

    if (attempt < maxAttempts) {
      const delayMs = getRetryDelay(attempt)
      console.log(`  ⎿  Network error, retrying in ${Math.round(delayMs / 1000)}s...`)

      try {
        await abortableDelay(delayMs, signal)
      } catch (error) {
        if (error.message === 'Request was aborted') {
          throw new Error('Request cancelled by user')
        }
        throw error
      }

      return getCompletionWithProfile(
        modelProfile, opts, attempt + 1, maxAttempts, signal
      )
    }
    throw error
  }
}
```

### 10.2 Claude 서비스

**파일**: `/src/services/claude.ts`

#### 주요 기능
- Anthropic/OpenAI API 통합
- 프롬프트 캐싱
- 컨텍스트 관리
- 응답 상태 추적

#### queryLLM 통합 함수

```typescript
export async function queryLLM(
  messages: (UserMessage | AssistantMessage)[],
  systemPrompt: string[],
  maxThinkingTokens: number,
  tools: Tool[],
  signal: AbortSignal,
  options: {
    safeMode: boolean
    model: string | ModelPointerType
    prependCLISysprompt: boolean
    toolUseContext?: ToolUseContext
  },
): Promise<AssistantMessage> {
  const modelManager = getModelManager()
  const modelResolution = modelManager.resolveModelWithInfo(options.model)

  if (!modelResolution.success || !modelResolution.profile) {
    throw new Error(
      modelResolution.error || `Failed to resolve model: ${options.model}`
    )
  }

  const modelProfile = modelResolution.profile
  const resolvedModel = modelProfile.modelName

  // 응답 상태 초기화
  const toolUseContext = options.toolUseContext
  if (toolUseContext && !toolUseContext.responseState) {
    const conversationId = getConversationId(
      toolUseContext.agentId,
      toolUseContext.messageId
    )
    const previousResponseId = responseStateManager.getPreviousResponseId(conversationId)

    toolUseContext.responseState = {
      previousResponseId,
      conversationId
    }
  }

  debugLogger.api('LLM_REQUEST_START', {
    messageCount: messages.length,
    model: resolvedModel,
    hasResponseState: !!toolUseContext?.responseState,
  })

  try {
    const result = await queryLLMWithPromptCaching(
      messages,
      systemPrompt,
      maxThinkingTokens,
      tools,
      signal,
      { ...options, model: resolvedModel, modelProfile, toolUseContext },
    )

    // 응답 상태 업데이트
    if (toolUseContext?.responseState?.conversationId && result.responseId) {
      responseStateManager.setPreviousResponseId(
        toolUseContext.responseState.conversationId,
        result.responseId
      )

      debugLogger.api('RESPONSE_STATE_UPDATED', {
        conversationId: toolUseContext.responseState.conversationId,
        responseId: result.responseId,
      })
    }

    return result
  } catch (error) {
    logErrorWithDiagnosis(error, {
      messageCount: messages.length,
      model: options.model,
    })
    throw error
  }
}
```

#### 프롬프트 캐싱

```typescript
function applyCacheControlWithLimits(
  systemBlocks: TextBlockParam[],
  messageParams: MessageParam[]
): { systemBlocks: TextBlockParam[]; messageParams: MessageParam[] } {
  if (!PROMPT_CACHING_ENABLED) {
    return { systemBlocks, messageParams }
  }

  const maxCacheBlocks = 4
  let usedCacheBlocks = 0

  // 시스템 프롬프트 캐싱 (최우선)
  const processedSystemBlocks = systemBlocks.map((block, index) => {
    if (usedCacheBlocks < maxCacheBlocks && block.text.length > 1000) {
      usedCacheBlocks++
      return {
        ...block,
        cache_control: { type: 'ephemeral' as const }
      }
    }
    const { cache_control, ...blockWithoutCache } = block
    return blockWithoutCache
  })

  // 메시지 내용 캐싱
  const processedMessageParams = messageParams.map((message, messageIndex) => {
    if (Array.isArray(message.content)) {
      const processedContent = message.content.map((contentBlock, blockIndex) => {
        const shouldCache =
          usedCacheBlocks < maxCacheBlocks &&
          contentBlock.type === 'text' &&
          typeof contentBlock.text === 'string' &&
          (
            // 긴 문서 (2000자 이상)
            contentBlock.text.length > 2000 ||
            // 마지막 메시지의 마지막 블록
            (messageIndex === messageParams.length - 1 &&
             blockIndex === message.content.length - 1 &&
             contentBlock.text.length > 500)
          )

        if (shouldCache) {
          usedCacheBlocks++
          return {
            ...contentBlock,
            cache_control: { type: 'ephemeral' as const }
          }
        }

        const { cache_control, ...blockWithoutCache } = contentBlock as any
        return blockWithoutCache
      })

      return {
        ...message,
        content: processedContent
      }
    }

    return message
  })

  return {
    systemBlocks: processedSystemBlocks,
    messageParams: processedMessageParams
  }
}
```

#### Anthropic 네이티브 호출

```typescript
async function queryAnthropicNative(
  messages: (UserMessage | AssistantMessage)[],
  systemPrompt: string[],
  maxThinkingTokens: number,
  tools: Tool[],
  signal: AbortSignal,
  options?: {
    safeMode: boolean
    model: string
    prependCLISysprompt: boolean
    modelProfile?: ModelProfile | null
    toolUseContext?: ToolUseContext
  },
): Promise<AssistantMessage> {
  const modelProfile = options?.modelProfile
  const model = modelProfile?.modelName || options?.model

  // Anthropic 클라이언트 생성
  const anthropic = new Anthropic({
    apiKey: modelProfile?.apiKey,
    baseURL: modelProfile?.baseURL,
    maxRetries: 0,
    timeout: 60000,
    defaultHeaders: {
      'x-app': 'cli',
      'User-Agent': USER_AGENT,
    },
  })

  // 시스템 프롬프트
  const system: TextBlockParam[] = systemPrompt.map(_ => ({
    text: _,
    type: 'text',
  }))

  // 도구 스키마
  const toolSchemas = await Promise.all(
    tools.map(async tool =>
      ({
        name: tool.name,
        description: getToolDescription(tool),
        input_schema: 'inputJSONSchema' in tool && tool.inputJSONSchema
          ? tool.inputJSONSchema
          : zodToJsonSchema(tool.inputSchema),
      }) as Anthropic.Beta.Messages.BetaTool,
    )
  )

  // 메시지 변환
  const anthropicMessages = messages.map(msg =>
    msg.type === 'user'
      ? { role: 'user', content: msg.message.content }
      : { role: 'assistant', content: msg.message.content }
  )

  // 캐시 제어 적용
  const { systemBlocks, messageParams } =
    applyCacheControlWithLimits(system, anthropicMessages)

  try {
    const response = await withRetry(async attempt => {
      const params: Anthropic.Beta.Messages.MessageCreateParams = {
        model,
        max_tokens: getMaxTokensFromProfile(modelProfile),
        messages: messageParams,
        system: systemBlocks,
        tools: toolSchemas.length > 0 ? toolSchemas : undefined,
        tool_choice: toolSchemas.length > 0 ? { type: 'auto' } : undefined,
      }

      if (maxThinkingTokens > 0) {
        (params as any).thinking = { max_tokens: maxThinkingTokens }
      }

      if (config.stream) {
        // 스트리밍
        const stream = await anthropic.beta.messages.create({
          ...params,
          stream: true,
        }, {
          signal: signal
        })

        // 스트림 처리 (수동 조립)
        let finalResponse: any | null = null
        const contentBlocks: any[] = []
        const inputJSONBuffers = new Map<number, string>()

        for await (const event of stream) {
          if (signal.aborted) {
            throw new Error('Request was cancelled')
          }

          // 이벤트 타입별 처리
          // ... (스트림 조립 로직)
        }

        return finalResponse
      } else {
        // 비스트리밍
        return await anthropic.beta.messages.create(params, {
          signal: signal
        })
      }
    }, { signal })

    // AssistantMessage 생성
    const assistantMessage: AssistantMessage = {
      message: {
        id: response.id,
        content: response.content,
        role: 'assistant',
        usage: response.usage,
      },
      type: 'assistant',
      uuid: nanoid() as UUID,
      durationMs: Date.now() - start,
      costUSD: calculateCost(response.usage, model),
    }

    return assistantMessage
  } catch (error) {
    return getAssistantMessageFromError(error)
  }
}
```

---

## 결론

### 시스템 장점

1. **확장성**: 새로운 API 추가가 용이한 어댑터 패턴
2. **유지보수성**: 공통 로직의 중앙화 및 재사용
3. **안정성**: 강력한 재시도 메커니즘 및 에러 처리
4. **성능**: 스트리밍 지원 및 프롬프트 캐싱
5. **호환성**: 다양한 AI 모델 제공자 지원

### 핵심 설계 원칙

1. **단일 책임 원칙**: 각 어댑터는 특정 API만 담당
2. **개방-폐쇄 원칙**: 새 기능 추가 시 기존 코드 수정 최소화
3. **의존성 역전**: 추상화에 의존, 구체적 구현에 의존하지 않음
4. **인터페이스 분리**: 클라이언트가 필요한 메서드만 노출

### 활용 시나리오

1. **다중 모델 지원**: 단일 인터페이스로 여러 AI 모델 활용
2. **실시간 스트리밍**: 빠른 응답 시작 및 사용자 경험 개선
3. **상태 연속성**: 대화 컨텍스트 유지 및 추론 재사용
4. **에러 복구**: 자동 재시도 및 대체 엔드포인트 사용

---

## 참고 자료

- **Anthropic API 문서**: https://docs.anthropic.com
- **OpenAI API 문서**: https://platform.openai.com/docs
- **GPT-5 Responses API**: https://platform.openai.com/docs/api-reference/responses
- **OAuth 2.0 PKCE**: https://oauth.net/2/pkce/

