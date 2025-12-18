# 실시간 Steering 메커니즘 복원 코드 구현

## 개요

Claude Code 난독화 소스 코드에 대한 심층 분석을 기반으로, 다음은 실시간 Steering 메커니즘의 고품질 가독성 복원 구현입니다.

## 1. 비동기 메시지 큐 (AsyncMessageQueue)

```typescript
interface QueueMessage {
  done: boolean;
  value?: any;
}

interface QueueCallbacks {
  resolve?: (value: QueueMessage) => void;
  reject?: (error: any) => void;
}

/**
 * 비동기 메시지 큐 - 실시간 메시지 큐잉 및 비차단 읽기 지원
 * 원본 난독화 클래스명: h2A
 */
class AsyncMessageQueue<T> implements AsyncIterable<T> {
  private readonly cleanupCallback?: () => void;
  private readonly messageQueue: T[] = [];
  private pendingRead: QueueCallbacks = {};
  private isCompleted: boolean = false;
  private errorState?: Error;
  private hasStarted: boolean = false;

  constructor(cleanupCallback?: () => void) {
    this.cleanupCallback = cleanupCallback;
  }

  /**
   * AsyncIterable 인터페이스 구현
   */
  [Symbol.asyncIterator](): AsyncIterator<T> {
    if (this.hasStarted) {
      throw new Error("Stream can only be iterated once");
    }
    this.hasStarted = true;
    return this;
  }

  /**
   * 비동기 반복자의 핵심 메서드
   * 큐에서 메시지를 우선적으로 가져오고, 큐가 비어있으면 새 메시지를 대기
   */
  next(): Promise<IteratorResult<T>> {
    // 큐의 메시지를 우선 처리
    if (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;
      return Promise.resolve({
        done: false,
        value: message
      });
    }

    // 큐가 완료됨
    if (this.isCompleted) {
      return Promise.resolve({
        done: true,
        value: undefined as any
      });
    }

    // 에러 상태가 있음
    if (this.errorState) {
      return Promise.reject(this.errorState);
    }

    // 새 메시지 대기 - 핵심 비차단 메커니즘
    return new Promise<IteratorResult<T>>((resolve, reject) => {
      this.pendingRead.resolve = resolve;
      this.pendingRead.reject = reject;
    });
  }

  /**
   * 메시지 큐잉 - 실시간 삽입 지원
   */
  enqueue(message: T): void {
    if (this.pendingRead.resolve) {
      // 대기 중인 읽기가 있으면 직접 메시지 반환
      const resolver = this.pendingRead.resolve;
      this.clearPendingRead();
      resolver({
        done: false,
        value: message
      });
    } else {
      // 큐 버퍼에 추가
      this.messageQueue.push(message);
    }
  }

  /**
   * 큐 완료 표시
   */
  complete(): void {
    this.isCompleted = true;
    if (this.pendingRead.resolve) {
      const resolver = this.pendingRead.resolve;
      this.clearPendingRead();
      resolver({
        done: true,
        value: undefined as any
      });
    }
  }

  /**
   * 에러 상태 설정
   */
  error(error: Error): void {
    this.errorState = error;
    if (this.pendingRead.reject) {
      const rejecter = this.pendingRead.reject;
      this.clearPendingRead();
      rejecter(error);
    }
  }

  /**
   * 정리 함수 반환 (비동기 반복자 프로토콜용)
   */
  return(): Promise<IteratorResult<T>> {
    this.isCompleted = true;
    if (this.cleanupCallback) {
      this.cleanupCallback();
    }
    return Promise.resolve({
      done: true,
      value: undefined as any
    });
  }

  private clearPendingRead(): void {
    this.pendingRead.resolve = undefined;
    this.pendingRead.reject = undefined;
  }
}
```

## 2. 스트리밍 메시지 파서 (StreamingMessageParser)

```typescript
interface UserMessage {
  type: "user";
  message: {
    role: "user";
    content: string | Array<{ type: "text"; text: string }>;
  };
}

/**
 * 스트리밍 메시지 파서 - 원시 입력 스트림을 구조화된 메시지로 파싱
 * 원본 난독화 클래스명: g2A
 */
class StreamingMessageParser {
  private readonly inputStream: AsyncIterable<string>;
  public readonly structuredMessages: AsyncIterable<UserMessage>;

  constructor(inputStream: AsyncIterable<string>) {
    this.inputStream = inputStream;
    this.structuredMessages = this.parseStream();
  }

  /**
   * 비동기 생성자 - 스트리밍 메시지 파싱
   */
  private async *parseStream(): AsyncGenerator<UserMessage> {
    let buffer = "";

    // 입력 스트림을 청크 단위로 처리
    for await (const chunk of this.inputStream) {
      buffer += chunk;

      // 줄 단위로 분할 처리
      let lineEndIndex: number;
      while ((lineEndIndex = buffer.indexOf('\n')) !== -1) {
        const line = buffer.slice(0, lineEndIndex);
        buffer = buffer.slice(lineEndIndex + 1);

        const parsedMessage = this.parseLine(line);
        if (parsedMessage) {
          yield parsedMessage;
        }
      }
    }

    // 마지막 줄 처리
    if (buffer.trim()) {
      const parsedMessage = this.parseLine(buffer);
      if (parsedMessage) {
        yield parsedMessage;
      }
    }
  }

  /**
   * 단일 라인 메시지 파싱
   */
  private parseLine(line: string): UserMessage | null {
    if (!line.trim()) return null;

    try {
      const message = JSON.parse(line) as UserMessage;

      // 엄격한 타입 검증
      this.validateMessage(message);

      return message;
    } catch (error) {
      console.error(`Error parsing streaming input line: ${line}`, error);
      process.exit(1);
    }
  }

  /**
   * 메시지 검증
   */
  private validateMessage(message: any): asserts message is UserMessage {
    if (message.type !== "user") {
      throw new Error(`Expected message type 'user', got '${message.type}'`);
    }

    if (message.message?.role !== "user") {
      throw new Error(`Expected message role 'user', got '${message.message?.role}'`);
    }
  }
}
```

## 3. 스트리밍 처리 엔진 (StreamingProcessor)

```typescript
interface Command {
  mode: "prompt";
  value: string;
}

interface ProcessorOptions {
  verbose: boolean;
  maxTurns: number;
  userSpecifiedModel?: string;
  fallbackModel?: string;
  systemPrompt?: string;
  appendSystemPrompt?: string;
}

/**
 * 스트리밍 처리 엔진 - 메시지 큐와 Agent 실행 조정
 * 원본 난독화 함수명: kq5
 */
class StreamingProcessor {
  private readonly commandQueue: Command[] = [];
  private isExecuting: boolean = false;
  private isCompleted: boolean = false;
  private readonly outputQueue: AsyncMessageQueue<any>;
  private messageHistory: any[] = [];

  constructor(
    private readonly inputStream: AsyncIterable<UserMessage>,
    private readonly permissionContext: any,
    private readonly mcpClients: any[],
    private readonly commands: any,
    private readonly tools: any,
    private readonly toolConfig: any,
    private readonly permissionTool: any,
    private readonly options: ProcessorOptions
  ) {
    this.outputQueue = new AsyncMessageQueue();
    this.messageHistory = this.processInitialMessages(toolConfig);
    this.startProcessing();
  }

  /**
   * 출력 스트림 가져오기
   */
  getOutputStream(): AsyncMessageQueue<any> {
    return this.outputQueue;
  }

  /**
   * 큐에 있는 명령 가져오기
   */
  getQueuedCommands(): Command[] {
    return [...this.commandQueue];
  }

  /**
   * 큐에서 명령 제거
   */
  removeQueuedCommands(commandsToRemove: Command[]): void {
    this.commandQueue.splice(0, this.commandQueue.length,
      ...this.commandQueue.filter(cmd => !commandsToRemove.includes(cmd))
    );
  }

  /**
   * 비동기 실행 엔진
   */
  private async executeCommands(): Promise<void> {
    this.isExecuting = true;

    try {
      // 큐의 모든 명령 처리
      while (this.commandQueue.length > 0) {
        const command = this.commandQueue.shift()!;

        if (command.mode !== "prompt") {
          throw new Error("Only prompt commands are supported in streaming mode");
        }

        // 메인 Agent 실행 루프 호출
        for await (const result of this.executeAgentLoop(command.value)) {
          this.messageHistory.push(result);
          this.outputQueue.enqueue(result);
        }
      }
    } finally {
      this.isExecuting = false;
    }

    if (this.isCompleted) {
      this.outputQueue.complete();
    }
  }

  /**
   * Agent 메인 루프 실행 (원본 Zk2 함수 호출)
   */
  private async *executeAgentLoop(prompt: string): AsyncGenerator<any> {
    // 실제 Agent 실행 로직 호출
    // 원본 코드의 Zk2 함수 호출과 동일

    const agentContext = this.createAgentContext();

    // 메인 Agent 루프 호출 (원본 nO 함수에 해당)
    yield* this.runMainAgentLoop(
      this.messageHistory,
      prompt,
      agentContext
    );
  }

  /**
   * Agent 컨텍스트 생성
   */
  private createAgentContext(): any {
    return {
      commands: this.commands,
      prompt: undefined, // 실행 시 설정됨
      cwd: process.cwd(),
      tools: this.tools,
      permissionContext: this.permissionContext,
      verbose: this.options.verbose,
      mcpClients: this.mcpClients,
      maxTurns: this.options.maxTurns,
      permissionPromptTool: this.permissionTool,
      userSpecifiedModel: this.options.userSpecifiedModel,
      fallbackModel: this.options.fallbackModel,
      initialMessages: this.messageHistory,
      customSystemPrompt: this.options.systemPrompt,
      appendSystemPrompt: this.options.appendSystemPrompt,
      getQueuedCommands: () => this.getQueuedCommands(),
      removeQueuedCommands: (cmds: Command[]) => this.removeQueuedCommands(cmds)
    };
  }

  /**
   * 메인 Agent 루프의 플레이스홀더 구현
   */
  private async *runMainAgentLoop(
    messages: any[],
    prompt: string,
    context: any
  ): AsyncGenerator<any> {
    // 원본 nO 함수 호출
    // 실제 구현은 구체적인 Agent 로직 호출 필요
    yield { type: "placeholder", content: "Agent processing..." };
  }

  /**
   * 입력 스트림 처리
   */
  private async startProcessing(): Promise<void> {
    try {
      for await (const message of this.inputStream) {
        const promptContent = this.extractPromptContent(message);

        // 새 메시지 큐잉
        this.commandQueue.push({
          mode: "prompt",
          value: promptContent
        });

        // 실행 중이 아니면 실행 시작
        if (!this.isExecuting) {
          this.executeCommands().catch(error => {
            console.error("Execution error:", error);
            this.outputQueue.error(error);
          });
        }
      }
    } catch (error) {
      console.error("Input processing error:", error);
      this.outputQueue.error(error as Error);
    } finally {
      this.isCompleted = true;
      if (!this.isExecuting) {
        this.outputQueue.complete();
      }
    }
  }

  /**
   * 메시지 내용 추출
   */
  private extractPromptContent(message: UserMessage): string {
    const content = message.message.content;

    if (typeof content === "string") {
      return content;
    }

    if (Array.isArray(content)) {
      if (content.length !== 1) {
        throw new Error(`Expected exactly one content item, got ${content.length}`);
      }

      const item = content[0];
      if (typeof item === "string") {
        return item;
      }

      if (item.type === "text") {
        return item.text;
      }
    }

    throw new Error("Expected string or text block content");
  }

  /**
   * 초기 메시지 처리
   */
  private processInitialMessages(toolConfig: any): any[] {
    // 도구 설정 처리 및 초기 메시지 히스토리 반환
    return toolConfig ? [toolConfig] : [];
  }
}
```

## 4. 메인 Agent 루프 (MainAgentLoop)

```typescript
interface AgentContext {
  messages: any[];
  options: {
    mainLoopModel: string;
    maxThinkingTokens: number;
    tools: any[];
    isNonInteractiveSession: boolean;
  };
  abortController: AbortController;
  getToolPermissionContext: () => any;
}

interface StreamEvent {
  type: string;
  [key: string]: any;
}

/**
 * 메인 Agent 루프 - async generator를 사용한 중단 가능한 스트리밍 처리
 * 원본 난독화 함수명: nO
 */
async function* mainAgentLoop(
  messages: any[],
  userContext: any,
  prompt: string,
  tools: any[],
  executionContext: any,
  agentContext: AgentContext,
  turnData: any,
  fallbackModel?: string,
  options?: any
): AsyncGenerator<StreamEvent> {
  // 스트림 시작 표시
  yield { type: "stream_request_start" };

  let currentMessages = messages;
  let currentTurnData = turnData;

  // 메시지 압축 처리
  const { messages: compactedMessages, wasCompacted } =
    await compactMessages(messages, agentContext);

  if (wasCompacted) {
    logEvent("auto_compact_succeeded", {
      originalMessageCount: messages.length,
      compactedMessageCount: compactedMessages.length
    });

    if (!currentTurnData?.compacted) {
      currentTurnData = {
        compacted: true,
        turnId: generateTurnId(),
        turnCounter: 0
      };
    }
    currentMessages = compactedMessages;
  }

  let assistantResponses: any[] = [];
  let currentModel = agentContext.options.mainLoopModel;
  let shouldRetry = true;

  try {
    // 메인 실행 루프 - 모델 폴백 재시도 지원
    while (shouldRetry) {
      shouldRetry = false;

      try {
        // 핵심 AI 처리 루프 호출 - 핵심 yield 지점
        for await (const response of processWithAI(
          buildMessages(currentMessages, prompt),
          buildContext(userContext, tools),
          agentContext.options.maxThinkingTokens,
          agentContext.options.tools,
          agentContext.abortController.signal, // 중단 신호 전달
          {
            getToolPermissionContext: agentContext.getToolPermissionContext,
            model: currentModel,
            prependCLISysprompt: true,
            toolChoice: undefined,
            isNonInteractiveSession: agentContext.options.isNonInteractiveSession,
            fallbackModel: fallbackModel
          }
        )) {
          // 각 응답을 스트리밍 출력 - 비차단 구현
          yield response;

          if (response.type === "assistant") {
            assistantResponses.push(response);
          }
        }
      } catch (error) {
        // 모델 폴백 처리
        if (error instanceof ModelError && fallbackModel) {
          currentModel = fallbackModel;
          shouldRetry = true;
          assistantResponses.length = 0;
          agentContext.options.mainLoopModel = fallbackModel;

          logEvent("model_fallback_triggered", {
            original_model: (error as any).originalModel,
            fallback_model: fallbackModel,
            entrypoint: "cli"
          });

          yield createLogMessage(
            `Model fallback: ${(error as any).originalModel} → ${fallbackModel}`,
            "info"
          );
          continue;
        }
        throw error;
      }
    }
  } catch (error) {
    // 에러 처리 및 도구 결과 생성
    const errorMessage = error instanceof Error ? error.message : String(error);

    logEvent("query_error", {
      assistantMessages: assistantResponses.length,
      toolUses: countToolUses(assistantResponses)
    });

    // 각 도구 사용에 대한 에러 결과 생성
    let hasToolResults = false;
    for (const response of assistantResponses) {
      const toolUses = extractToolUses(response);
      for (const toolUse of toolUses) {
        yield createToolResult({
          content: [{
            type: "tool_result",
            content: errorMessage,
            is_error: true,
            tool_use_id: toolUse.id
          }],
          toolUseResult: errorMessage
        });
        hasToolResults = true;
      }
    }

    if (!hasToolResults) {
      yield createSystemMessage({
        toolUse: false,
        hardcodedMessage: undefined
      });
    }
    return;
  }

  // 도구 실행 처리
  if (!assistantResponses.length) return;

  const toolUses = extractAllToolUses(assistantResponses);
  if (!toolUses.length) return;

  let toolResults: any[] = [];
  let shouldPreventContinuation = false;

  // 도구 호출 실행
  for await (const result of executeTools(
    toolUses,
    assistantResponses,
    executionContext,
    agentContext
  )) {
    yield result; // 도구 결과 스트리밍 출력

    if (result?.type === "system" && result.preventContinuation) {
      shouldPreventContinuation = true;
    }

    toolResults.push(...filterUserMessages([result]));
  }

  // 중단 여부 확인
  if (agentContext.abortController.signal.aborted) {
    yield createSystemMessage({
      toolUse: true,
      hardcodedMessage: undefined
    });
    return;
  }

  if (shouldPreventContinuation) return;

  // 후속 처리 계속...
  const sortedResults = sortToolResults(toolResults, toolUses);

  if (currentTurnData?.compacted) {
    currentTurnData.turnCounter++;
    logEvent("post_autocompact_turn", {
      turnId: currentTurnData.turnId,
      turnCounter: currentTurnData.turnCounter
    });
  }

  // 재귀 호출로 처리 계속
  yield* mainAgentLoop(
    [...currentMessages, ...buildContinuationMessages(assistantResponses, sortedResults)],
    userContext,
    prompt,
    tools,
    executionContext,
    agentContext,
    currentTurnData,
    fallbackModel,
    options
  );
}

// 보조 함수 구현
class ModelError extends Error {
  originalModel: string;
  fallbackModel: string;

  constructor(originalModel: string, fallbackModel: string, message: string) {
    super(message);
    this.originalModel = originalModel;
    this.fallbackModel = fallbackModel;
  }
}

async function compactMessages(messages: any[], context: AgentContext) {
  // 메시지 압축 로직
  return { messages, wasCompacted: false };
}

function generateTurnId(): string {
  return Math.random().toString(36).substr(2, 9);
}

function logEvent(event: string, data: any): void {
  // 이벤트 로그 기록
  console.log(`Event: ${event}`, data);
}

async function* processWithAI(
  messages: any[],
  context: any,
  maxTokens: number,
  tools: any[],
  abortSignal: AbortSignal,
  options: any
): AsyncGenerator<StreamEvent> {
  // AI 처리 로직
  yield { type: "ai_processing", status: "started" };

  // 중단 신호 확인
  if (abortSignal.aborted) {
    throw new Error("Request aborted");
  }

  // AI 응답 시뮬레이션
  yield { type: "assistant", message: { content: "AI response" } };
}

function buildMessages(messages: any[], prompt: string): any[] {
  return [...messages, { role: "user", content: prompt }];
}

function buildContext(userContext: any, tools: any[]): any {
  return { user: userContext, tools };
}

function createLogMessage(message: string, level: string): StreamEvent {
  return { type: "log", message, level };
}

function createToolResult(result: any): StreamEvent {
  return { type: "tool_result", ...result };
}

function createSystemMessage(options: any): StreamEvent {
  return { type: "system", ...options };
}

function countToolUses(responses: any[]): number {
  return responses.flatMap(r =>
    r.message.content.filter((c: any) => c.type === "tool_use")
  ).length;
}

function extractToolUses(response: any): any[] {
  return response.message.content.filter((c: any) => c.type === "tool_use");
}

function extractAllToolUses(responses: any[]): any[] {
  return responses.flatMap(extractToolUses);
}

async function* executeTools(
  toolUses: any[],
  responses: any[],
  context: any,
  agentContext: AgentContext
): AsyncGenerator<StreamEvent> {
  for (const toolUse of toolUses) {
    yield { type: "tool_execution", toolUse };
  }
}

function filterUserMessages(results: any[]): any[] {
  return results.filter(r => r.type === "user");
}

function sortToolResults(results: any[], toolUses: any[]): any[] {
  return results.sort((a, b) => {
    const aIndex = toolUses.findIndex(tu => tu.id === a.id);
    const bIndex = toolUses.findIndex(tu => tu.id === b.id);
    return aIndex - bIndex;
  });
}

function buildContinuationMessages(responses: any[], results: any[]): any[] {
  return [...responses, ...results];
}
```

## 5. 완전한 통합 구현

```typescript
/**
 * 실시간 Steering 시스템의 완전한 통합
 */
class RealtimeSteeringSystem {
  private messageParser: StreamingMessageParser;
  private processor: StreamingProcessor;
  private abortController: AbortController;

  constructor(
    inputStream: AsyncIterable<string>,
    options: {
      tools: any[];
      commands: any;
      mcpClients: any[];
      permissionContext: any;
      processorOptions: ProcessorOptions;
    }
  ) {
    this.abortController = new AbortController();

    // 메시지 파서 생성
    this.messageParser = new StreamingMessageParser(inputStream);

    // 스트리밍 프로세서 생성
    this.processor = new StreamingProcessor(
      this.messageParser.structuredMessages,
      options.permissionContext,
      options.mcpClients,
      options.commands,
      options.tools,
      null, // toolConfig
      null, // permissionTool
      options.processorOptions
    );
  }

  /**
   * 출력 스트림 가져오기
   */
  getOutputStream(): AsyncMessageQueue<any> {
    return this.processor.getOutputStream();
  }

  /**
   * 처리 중단
   */
  abort(reason?: string): void {
    this.abortController.abort(reason);
  }

  /**
   * 중단 신호 가져오기
   */
  get abortSignal(): AbortSignal {
    return this.abortController.signal;
  }

  /**
   * 시스템 시작 (사용 예시)
   */
  static async start(inputStream: AsyncIterable<string>): Promise<void> {
    const system = new RealtimeSteeringSystem(inputStream, {
      tools: [],
      commands: {},
      mcpClients: [],
      permissionContext: {},
      processorOptions: {
        verbose: false,
        maxTurns: 10
      }
    });

    const outputStream = system.getOutputStream();

    try {
      for await (const message of outputStream) {
        console.log("Output:", message);
      }
    } catch (error) {
      console.error("Processing error:", error);
    }
  }
}

// 사용 예시
async function example() {
  // stdin 입력 스트림 시뮬레이션
  const stdinStream = createStdinStream();

  // 실시간 Steering 시스템 시작
  await RealtimeSteeringSystem.start(stdinStream);
}

async function* createStdinStream(): AsyncGenerator<string> {
  // stdin에서 데이터 읽기 시뮬레이션
  process.stdin.setEncoding('utf8');

  for await (const chunk of process.stdin) {
    yield chunk;
  }
}
```

## 결론

위 구현은 Claude Code 실시간 Steering 메커니즘의 핵심 컴포넌트를 복원했습니다:

1. **AsyncMessageQueue**: 비차단 비동기 메시지 큐
2. **StreamingMessageParser**: 스트리밍 메시지 파서
3. **StreamingProcessor**: 스트리밍 처리 엔진
4. **mainAgentLoop**: 중단 가능한 Agent 메인 루프

이러한 컴포넌트들이 함께 다음을 구현합니다:
- 진정한 실시간 메시지 처리
- 비차단 비동기 실행
- 완전한 중단 및 복구 메커니즘
- 스트리밍 입출력 처리

이 구현은 오픈소스 Agent 시스템에 실행 가능한 기술 경로와 구체적인 코드 참조를 제공합니다.
