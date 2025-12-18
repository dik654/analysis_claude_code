# Open Claude Code - 기술 개발 구현 문서 (TDD)

## 1. 기술 아키텍처 설계

### 1.1 전체 시스템 아키텍처

Claude Code의 심층 역공학 분석을 기반으로 8계층 아키텍처(실시간 Steering 계층 포함)를 채택합니다:

```typescript
// 시스템 아키텍처 인터페이스 정의
interface SystemArchitecture {
  layers: {
    cli: CLILayer;           // 커맨드라인 인터페이스 계층
    ui: UILayer;             // 사용자 인터페이스 계층
    steering: SteeringLayer; // 실시간 Steering 계층(h2A 비동기 큐)
    event: EventLayer;       // 이벤트 시스템 계층
    message: MessageLayer;   // 메시지 처리 계층
    agent: AgentLayer;       // Agent 핵심 계층(nO 주 루프)
    tool: ToolLayer;         // Tool 실행 계층
    multiAgent: MultiAgentLayer; // 다중 Agent 조정 계층(Task Tool)
    api: APILayer;           // API 인터페이스 계층
  };
}
```

### 1.2 핵심 모듈 설계

#### 1.2.1 Agent 핵심 엔진
```typescript
// nO async generator 함수 기반 Agent 주 루프 설계
// h2A 비동기 메시지 큐 및 실시간 Steering 메커니즘 통합
class AgentCore {
  private steeringQueue: h2A; // 실시간 메시지 큐
  private abortController: AbortController; // 중단 컨트롤러

  async* mainLoop(
    messages: Message[],
    config: AgentConfig,
    context: Context
  ): AsyncGenerator<ResponseChunk> {
    // 1. 실시간 Steering 메커니즘 초기화
    this.steeringQueue = new h2A(this.cleanupResources.bind(this));
    this.abortController = new AbortController();
    this.startStdinListening(); // stdin 리스닝 시작

    // 2. 메시지 압축 확인
    const { messages: compactedMessages, wasCompacted } =
      await this.compressMessages(messages, config);

    // 3. 주 루프 실행 (비동기 제너레이터 패턴)
    while (this.shouldContinue && !this.abortController.signal.aborted) {
      try {
        // 4. 실시간 사용자 입력 확인
        const steeringMessage = await this.checkSteeringInput();
        if (steeringMessage) {
          messages.push(steeringMessage);
          // 동적으로 실행 전략 조정
          await this.adjustExecutionStrategy(steeringMessage);
        }

        // 5. 컨텍스트 주입
        const enrichedMessages = this.injectContext(compactedMessages, context);

        // 6. 스트림 제너레이터 호출 (kq5 스트리밍 처리)
        for await (const chunk of this.streamGenerator(enrichedMessages, config)) {
          // 중단 신호 확인
          if (this.abortController.signal.aborted) break;
          yield chunk;
        }

        // 7. 재귀 호출 확인
        if (this.needsContinuation(chunk)) {
          yield* this.mainLoop([...messages, ...responses], config, context);
        }
      } catch (error) {
        // 8. 모델 폴백 처리
        if (error instanceof ModelError && config.fallbackModel) {
          config.currentModel = config.fallbackModel;
          continue;
        }
        throw error;
      }
    }
  }

  private async checkSteeringInput(): Promise<Message | null> {
    try {
      const steeringData = await this.steeringQueue.next();
      if (steeringData.value) {
        return this.parseSteeringMessage(steeringData.value);
      }
    } catch (error) {
      // 실시간 입력 실패, 실행 계속
    }
    return null;
  }
}
```

#### 1.2.2 Tool 실행 엔진
```typescript
// Tool 실행 엔진 (gW5 복잡한 동시성 관리 메커니즘 통합)
class ToolExecutionEngine {
  private concurrencyManager: gW5ConcurrencyManager; // 복잡한 동시성 관리

  async executeTools(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    // 1. 동시성 안전성 분석
    const safeGroups = this.analyzeConcurrencySafety(toolCalls);

    // 2. 지능형 그룹 스케줄링
    const executionGroups = this.scheduleToolGroups(safeGroups);

    // 3. 동시 실행 제어
    const results: ToolResult[] = [];
    for (const group of executionGroups) {
      const groupResults = await this.executeConcurrently(group);
      results.push(...groupResults);
    }

    return results;
  }

  private async executeConcurrently(tools: ToolCall[]): Promise<ToolResult[]> {
    // gW5 복잡한 동시성 관리 메커니즘 사용 (동적 로드 밸런싱 포함)
    const concurrencyConfig = this.concurrencyManager.calculateOptimalConcurrency(tools);

    return Promise.all(tools.map(async (tool) => {
      await concurrencyConfig.semaphore.acquire();
      try {
        return await this.executeSingleTool(tool, concurrencyConfig);
      } finally {
        concurrencyConfig.semaphore.release();
      }
    }));
  }
}
```

### 1.3 실시간 Steering 통신 메커니즘 설계

#### 1.3.1 h2A 비동기 메시지 큐 시스템
```typescript
// 역공학 분석 기반 h2A 클래스 구현
class h2A implements AsyncIterator<any> {
  private returned: (() => void) | null; // 정리 함수
  private queue: any[] = [];             // 메시지 큐 버퍼
  private readResolve?: (value: any) => void; // Promise resolve 콜백
  private readReject?: (reason: any) => void;  // Promise reject 콜백
  private isDone = false;                // 큐 완료 플래그
  private hasError?: any;                // 오류 상태
  private started = false;               // 시작 상태 플래그

  constructor(cleanupFn?: () => void) {
    this.returned = cleanupFn || null;
  }

  // AsyncIterator 인터페이스 구현
  [Symbol.asyncIterator](): AsyncIterator<any> {
    if (this.started) {
      throw new Error("Stream can only be iterated once");
    }
    this.started = true;
    return this;
  }

  // 핵심 비동기 이터레이터 메서드
  async next(): Promise<IteratorResult<any>> {
    if (this.hasError) {
      throw this.hasError;
    }

    if (this.queue.length > 0) {
      const value = this.queue.shift();
      return { value, done: false };
    }

    if (this.isDone) {
      return { value: undefined, done: true };
    }

    // 새 메시지 대기
    return new Promise((resolve, reject) => {
      this.readResolve = resolve;
      this.readReject = reject;
    });
  }

  // 큐에 메시지 추가
  enqueue(message: any): void {
    if (this.isDone) return;

    if (this.readResolve) {
      this.readResolve({ value: message, done: false });
      this.readResolve = undefined;
      this.readReject = undefined;
    } else {
      this.queue.push(message);
    }
  }

  // 큐 완료
  complete(): void {
    this.isDone = true;
    if (this.readResolve) {
      this.readResolve({ value: undefined, done: true });
      this.readResolve = undefined;
      this.readReject = undefined;
    }
    if (this.returned) {
      this.returned();
    }
  }

  // 오류 처리
  error(err: any): void {
    this.hasError = err;
    if (this.readReject) {
      this.readReject(err);
      this.readResolve = undefined;
      this.readReject = undefined;
    }
  }
}

interface MessageQueue extends h2A {
  priority: MessagePriority;
}

enum MessagePriority {
  SYSTEM_INTERRUPT = 0,  // 시스템 중단 (최고 우선순위)
  USER_STEERING = 1,     // 사용자 실시간 가이드
  USER_INPUT = 2,        // 사용자 입력
  TOOL_RESULT = 3,       // Tool 결과
  SYSTEM_UPDATE = 4      // 시스템 업데이트
}
```

#### 1.3.2 실시간 Steering 리스닝 시스템
```typescript
class SteeringSystem {
  private stdinListener: NodeJS.ReadStream;
  private messageQueue: h2A;
  private abortController: AbortController;

  constructor(messageQueue: h2A) {
    this.messageQueue = messageQueue;
    this.abortController = new AbortController();
    this.setupStdinListening();
  }

  // stdin 실시간 리스닝 설정
  private setupStdinListening(): void {
    this.stdinListener = process.stdin;
    this.stdinListener.setRawMode(true);
    this.stdinListener.resume();

    this.stdinListener.on('data', (chunk) => {
      try {
        const input = chunk.toString('utf8');
        // 특수 제어 문자 확인
        if (this.isSteeringInput(input)) {
          const steeringMessage = this.parseSteeringInput(input);
          this.messageQueue.enqueue({
            type: 'steering',
            content: steeringMessage,
            timestamp: Date.now(),
            priority: MessagePriority.USER_STEERING
          });
        }
      } catch (error) {
        console.error('Steering input parse error:', error);
      }
    });
  }

  // 실시간 가이드 입력 여부 판단
  private isSteeringInput(input: string): boolean {
    // 특수 제어 문자 또는 키 조합
    return input.includes('\u001b') || // ESC 키
           input.charCodeAt(0) === 3 || // Ctrl+C
           input.includes('\r') ||       // Enter 키
           input.length > 1;             // 다중 문자 입력
  }

  // 실시간 가이드 입력 파싱
  private parseSteeringInput(input: string): string {
    // 특수 키 조합 처리
    if (input.includes('\u001b[')) {
      // 화살표 키 등 특수 입력 처리
      return this.handleSpecialKeys(input);
    }

    return input.trim();
  }

  // 리스닝 중지
  stop(): void {
    this.abortController.abort();
    this.stdinListener.pause();
    if (this.stdinListener.setRawMode) {
      this.stdinListener.setRawMode(false);
    }
  }
}

// 이벤트 기반 통신 시스템
class EventSystem {
  private eventHandlers = new Map<string, EventHandler[]>();

  // 이벤트 디스패치 메커니즘 (실시간 Steering 지원 통합)
  async dispatch(event: SystemEvent): Promise<void> {
    const handlers = this.eventHandlers.get(event.type) || [];

    // system-reminder 생성
    if (this.shouldGenerateReminder(event)) {
      const reminder = this.generateSystemReminder(event);
      await this.injectReminder(reminder);
    }

    // 실시간 Steering 이벤트 처리
    if (event.type === 'steering_input') {
      await this.handleSteeringEvent(event);
    }

    // 이벤트 핸들러 실행
    await Promise.all(handlers.map(handler => handler(event)));
  }

  private async handleSteeringEvent(event: SystemEvent): Promise<void> {
    // 사용자 실시간 가이드 입력 처리
    const steeringContent = event.data.content;
    // Agent 실행 전략을 동적으로 조정
    await this.adjustAgentStrategy(steeringContent);
  }

  private generateSystemReminder(event: SystemEvent): SystemReminder {
    switch (event.type) {
      case 'todo_updated':
        return this.createTodoReminder(event.data);
      case 'file_edited':
        return this.createFileEditReminder(event.data);
      case 'plan_mode_activated':
        return this.createPlanModeReminder();
      case 'steering_input':
        return this.createSteeringReminder(event.data);
      default:
        return null;
    }
  }
}
```

## 2. 데이터 구조 설계

### 2.1 핵심 데이터 모델

#### 2.1.1 메시지 데이터 구조
```typescript
interface Message {
  id: string;
  type: 'user' | 'assistant' | 'system';
  role: 'user' | 'assistant' | 'system';
  content: string | ContentBlock[];
  timestamp: string;
  isMeta: boolean;                    // 메타 정보 마크
  isCompactSummary?: boolean;         // 압축 요약 마크
  toolUseResult?: ToolResult;         // Tool 실행 결과
  uuid: string;                       // 고유 식별자
}

interface SystemReminder extends Message {
  context: {
    directoryStructure?: string;
    gitStatus?: string;
    claudeMd?: string;
    todoList?: TodoItem[];
  };
}
```

#### 2.1.2 Tool 정의 구조
```typescript
interface Tool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  userFacingName(): string;
  isConcurrencySafe(): boolean;

  // Tool 실행 인터페이스
  execute(input: any, context: ToolContext): Promise<ToolResult>;

  // 권한 검증 인터페이스
  checkPermissions(input: any, context: ToolContext): Promise<PermissionResult>;

  // 결과 매핑 인터페이스
  mapResult(result: any): ToolResultBlock;
}

// 내장 Tool 타입
enum BuiltinTools {
  READ = 'Read',
  WRITE = 'Write',
  EDIT = 'Edit',
  MULTI_EDIT = 'MultiEdit',
  BASH = 'Bash',
  LS = 'LS',
  GLOB = 'Glob',
  GREP = 'Grep',
  TODO_READ = 'TodoRead',
  TODO_WRITE = 'TodoWrite',
  TASK = 'Task',
  WEB_FETCH = 'WebFetch',
  WEB_SEARCH = 'WebSearch',
  NOTEBOOK_READ = 'NotebookRead',
  NOTEBOOK_EDIT = 'NotebookEdit',
  EXIT_PLAN_MODE = 'exit_plan_mode'
}
```

#### 2.1.3 MCP 프로토콜 데이터 구조
```typescript
interface MCPServer {
  name: string;
  config: MCPServerConfig;
  transport: MCPTransport;
  status: MCPServerStatus;
  tools: MCPTool[];
  resources: MCPResource[];
}

interface MCPServerConfig {
  command?: string;
  args?: string[];
  env?: Record<string, string>;
  transport: 'stdio' | 'http' | 'websocket' | 'sse';
  settings?: Record<string, any>;
  auth?: OAuthConfig;
}

interface MCPTool extends Tool {
  serverId: string;
  mcpName: string;           // 원본 MCP Tool명
  qualifiedName: string;     // mcp__server__tool 형식
}
```

### 2.2 상태 관리 설계

#### 2.2.1 애플리케이션 상태 구조
```typescript
interface ApplicationState {
  // UI 컴포넌트 분석 기반 11개 핵심 상태
  responseStatus: 'idle' | 'responding' | 'error';     // k1/Q1
  messageHistory: Message[];                           // v1/L1
  currentSession: Session | null;                      // BA/HA
  modeFlag: boolean;                                   // MA/t
  activeObject: any | null;                           // B1/W1
  userConfig: UserConfig | null;                      // w1/P1
  eventList: SystemEvent[];                           // e/y1
  messageQueue: Message[];                            // O1/h1
  temporaryData: any[];                               // o1/QA
  inputBuffer: string;                                // zA/Y0
  uiMode: 'prompt' | 'plan' | 'settings';            // fA/H0
}

interface Session {
  id: string;
  title: string;
  messages: Message[];
  createdAt: string;
  updatedAt: string;
  metadata: SessionMetadata;
}
```

## 3. 모듈 구현 상세 설계

### 3.1 CLI 명령 시스템 구현

#### 3.1.1 Commander.js 통합
```typescript
// CLI 시작 프로세스 분석 기반 구현
class CLIInterface {
  private program: Command;

  constructor() {
    this.program = new Command();
    this.setupCommands();
  }

  private setupCommands(): void {
    this.program
      .name('claude')
      .description('Open Claude Code - AI Programming Assistant')
      .version(VERSION)
      .argument('[prompt]', 'Your prompt', String)
      .option('-d, --debug', 'Enable debug mode')
      .option('--verbose', 'Enable verbose output')
      .option('-p, --print', 'Print response and exit')
      .option('-c, --continue', 'Continue recent conversation')
      .option('-r, --resume [sessionId]', 'Resume specific session')
      .option('--model <model>', 'Specify model to use')
      .option('--fallback-model <model>', 'Specify fallback model')
      .option('--mcp-config <config>', 'MCP server configuration')
      .action(this.handleMainCommand.bind(this));
  }

  private async handleMainCommand(prompt?: string, options?: CLIOptions): Promise<void> {
    // 모드 감지
    if (options.print) {
      await this.runPrintMode(prompt, options);
    } else {
      await this.runInteractiveMode(prompt, options);
    }
  }
}
```

#### 3.1.2 파라미터 검증 및 처리
```typescript
class ParameterValidator {
  static validateModel(model: string): boolean {
    const supportedModels = [
      'claude-3-5-sonnet-20241022',
      'claude-3-5-haiku-20241022',
      'gpt-4o',
      'gpt-4o-mini'
    ];
    return supportedModels.includes(model);
  }

  static validateMCPConfig(config: string): MCPServerConfig {
    try {
      const parsed = JSON.parse(config);
      // 구성 형식 검증
      return this.normalizeMCPConfig(parsed);
    } catch (error) {
      throw new Error(`Invalid MCP configuration: ${error.message}`);
    }
  }
}
```

### 3.2 UI 컴포넌트 시스템 구현

#### 3.2.1 React/Ink 프레임워크 통합
```typescript
// UI 컴포넌트 분석 기반 터미널 인터페이스 구현
import React from 'react';
import { render, Box, Text } from 'ink';

interface MainAppProps {
  initialState: ApplicationState;
}

const MainApp: React.FC<MainAppProps> = ({ initialState }) => {
  // 11개 핵심 상태 변수
  const [responseStatus, setResponseStatus] = useState<ResponseStatus>('idle');
  const [messageHistory, setMessageHistory] = useState<Message[]>([]);
  const [currentSession, setCurrentSession] = useState<Session | null>(null);
  const [modeFlag, setModeFlag] = useState<boolean>(false);
  const [activeObject, setActiveObject] = useState<any>(null);
  const [userConfig, setUserConfig] = useState<UserConfig | null>(null);
  const [eventList, setEventList] = useState<SystemEvent[]>([]);
  const [messageQueue, setMessageQueue] = useState<Message[]>([]);
  const [temporaryData, setTemporaryData] = useState<any[]>([]);
  const [inputBuffer, setInputBuffer] = useState<string>('');
  const [uiMode, setUIMode] = useState<UIMode>('prompt');

  return (
    <Box flexDirection="column" height="100%">
      <WelcomeHeader />
      <MessageDisplay messages={messageHistory} />
      <ProgressIndicator status={responseStatus} />
      <InputPanel
        value={inputBuffer}
        onChange={setInputBuffer}
        onSubmit={handleUserInput}
        mode={uiMode}
      />
      <StatusBar session={currentSession} mode={uiMode} />
    </Box>
  );
};
```

#### 3.2.2 터미널 적응 및 반응형 디자인
```typescript
// c9 함수 기반 터미널 크기 관리
const useTerminalSize = () => {
  const [size, setSize] = useState({
    columns: process.stdout.columns || 80,
    rows: process.stdout.rows || 24
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        columns: process.stdout.columns || 80,
        rows: process.stdout.rows || 24
      });
    };

    process.stdout.setMaxListeners(200);
    process.stdout.on('resize', handleResize);

    return () => {
      process.stdout.off('resize', handleResize);
    };
  }, []);

  return size;
};
```

### 3.3 Tool 시스템 구현

#### 3.3.1 Tool 등록 및 발견
```typescript
class ToolRegistry {
  private tools = new Map<string, Tool>();
  private mcpTools = new Map<string, MCPTool>();

  registerBuiltinTools(): void {
    // 15개 내장 Tool 등록
    this.register(new ReadTool());
    this.register(new WriteTool());
    this.register(new EditTool());
    this.register(new BashTool());
    this.register(new LSTools());
    this.register(new GlobTool());
    this.register(new GrepTool());
    this.register(new TodoReadTool());
    this.register(new TodoWriteTool());
    this.register(new TaskTool());
    this.register(new WebFetchTool());
    this.register(new WebSearchTool());
    this.register(new NotebookReadTool());
    this.register(new NotebookEditTool());
    this.register(new ExitPlanModeTool());
  }

  async discoverMCPTools(): Promise<void> {
    for (const server of this.mcpManager.getActiveServers()) {
      const tools = await server.listTools();
      for (const tool of tools) {
        const qualifiedName = `mcp__${server.name}__${tool.name}`;
        const mcpTool = new MCPTool(qualifiedName, tool, server);
        this.mcpTools.set(qualifiedName, mcpTool);
      }
    }
  }
}
```

#### 3.3.2 동시성 제어 구현
```typescript
// gW5 복잡한 동시성 관리 메커니즘 기반 구현
class ConcurrencyController {
  private readonly concurrencyManager: gW5ConcurrencyManager;
  private activeTasks = new Set<string>();
  private taskQueue: ToolCall[] = [];

  async execute(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    // 동시성 안전성 분석
    const { safeConcurrent, requiresSequential } = this.analyzeSafety(toolCalls);

    // 동시 실행 가능한 Tool 실행
    const concurrentResults = await this.executeConcurrent(safeConcurrent);

    // 안전하지 않은 Tool 순차 실행
    const sequentialResults = await this.executeSequential(requiresSequential);

    return [...concurrentResults, ...sequentialResults];
  }

  private analyzeSafety(toolCalls: ToolCall[]): {
    safeConcurrent: ToolCall[];
    requiresSequential: ToolCall[];
  } {
    const safe: ToolCall[] = [];
    const unsafe: ToolCall[] = [];

    for (const call of toolCalls) {
      const tool = this.toolRegistry.get(call.toolName);
      if (tool?.isConcurrencySafe()) {
        safe.push(call);
      } else {
        unsafe.push(call);
      }
    }

    return { safeConcurrent: safe, requiresSequential: unsafe };
  }
}
```

### 3.4 MCP 프로토콜 구현

#### 3.4.1 MCP 클라이언트 구현
```typescript
class MCPClient {
  private transport: MCPTransport;
  private messageId = 0;
  private pendingRequests = new Map<number, PendingRequest>();

  constructor(config: MCPServerConfig) {
    this.transport = this.createTransport(config);
  }

  private createTransport(config: MCPServerConfig): MCPTransport {
    switch (config.transport) {
      case 'stdio':
        return new STDIOTransport(config.command, config.args);
      case 'http':
        return new HTTPTransport(config.baseUrl);
      case 'websocket':
        return new WebSocketTransport(config.url);
      case 'sse':
        return new SSETransport(config.url);
      default:
        throw new Error(`Unsupported transport: ${config.transport}`);
    }
  }

  async callTool(name: string, arguments_: any): Promise<any> {
    const request: JSONRPCRequest = {
      jsonrpc: '2.0',
      id: ++this.messageId,
      method: 'tools/call',
      params: {
        name,
        arguments: arguments_
      }
    };

    return this.sendRequest(request);
  }
}
```

#### 3.4.2 다중 전송 프로토콜 지원
```typescript
abstract class MCPTransport {
  abstract connect(): Promise<void>;
  abstract disconnect(): Promise<void>;
  abstract send(message: JSONRPCMessage): Promise<void>;
  abstract receive(): AsyncGenerator<JSONRPCMessage>;
}

class STDIOTransport extends MCPTransport {
  private process: ChildProcess;

  constructor(private command: string, private args: string[]) {
    super();
  }

  async connect(): Promise<void> {
    this.process = spawn(this.command, this.args, {
      stdio: ['pipe', 'pipe', 'pipe']
    });

    // 메시지 처리 설정
    this.process.stdout.on('data', this.handleStdout.bind(this));
    this.process.stderr.on('data', this.handleStderr.bind(this));
  }
}

class HTTPTransport extends MCPTransport {
  constructor(private baseUrl: string) {
    super();
  }

  async send(message: JSONRPCMessage): Promise<void> {
    const response = await fetch(`${this.baseUrl}/rpc`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
  }
}
```

### 3.5 메시지 압축 시스템 구현

#### 3.5.1 압축 전략
```typescript
// wU2 함수 기반 메시지 압축 구현
class MessageCompressor {
  async shouldCompress(messages: Message[]): Promise<boolean> {
    // 다양한 조건을 기반으로 압축 필요 여부 판단
    const totalTokens = this.estimateTokens(messages);
    const messageCount = messages.length;
    const contextSize = this.calculateContextSize();

    return totalTokens > 100000 ||
           messageCount > 100 ||
           contextSize > 50000;
  }

  async compress(messages: Message[], config: CompressionConfig): Promise<{
    messages: Message[];
    wasCompacted: boolean;
  }> {
    try {
      // LLM을 호출하여 지능형 압축
      const summary = await this.generateSummary(messages, config);

      // 최근 메시지와 중요한 시스템 메시지 보존
      const recentMessages = messages.slice(-10);
      const systemMessages = messages.filter(m => m.isMeta);

      return {
        messages: [summary, ...systemMessages, ...recentMessages],
        wasCompacted: true
      };
    } catch (error) {
      // 압축 실패 시 원본 메시지 반환
      return {
        messages,
        wasCompacted: false
      };
    }
  }

  private async generateSummary(messages: Message[], config: CompressionConfig): Promise<Message> {
    // AU2 함수 기반 8단 요약 템플릿
    const summaryPrompt = this.createSummaryPrompt();

    const response = await this.llmClient.complete({
      messages: [
        { role: 'system', content: summaryPrompt },
        { role: 'user', content: this.formatMessagesForSummary(messages) }
      ],
      model: config.model,
      maxTokens: 4000
    });

    return {
      id: generateId(),
      type: 'user',
      role: 'user',
      content: response.content,
      timestamp: new Date().toISOString(),
      isMeta: true,
      isCompactSummary: true,
      uuid: generateUUID()
    };
  }
}
```

## 4. 핵심 알고리즘 설계

### 4.1 컨텍스트 주입 알고리즘

```typescript
// Ie1 함수 기반 컨텍스트 주입 구현
class ContextInjector {
  async injectContext(messages: Message[], context: Context): Promise<Message[]> {
    const contextInfo = await this.gatherContextInfo(context);

    // 컨텍스트 정보가 있을 때만 주입
    if (Object.keys(contextInfo).length === 0) {
      return messages;
    }

    // system-reminder 생성
    const reminder = this.createSystemReminder(contextInfo);

    // 메시지 큐에 앞쪽으로 주입
    return [reminder, ...messages];
  }

  private async gatherContextInfo(context: Context): Promise<ContextInfo> {
    const info: ContextInfo = {};

    // 디렉토리 구조 수집
    if (context.includeDirectoryStructure) {
      info.directoryStructure = await this.getDirectoryStructure();
    }

    // Git 상태 수집
    if (context.includeGitStatus) {
      info.gitStatus = await this.getGitStatus();
    }

    // CLAUDE.md 구성 수집
    if (context.includeClaude()) {
      info.claudeMd = await this.getClaudeMd();
    }

    // 원격 측정 데이터 전송
    this.sendTelemetry(info);

    return info;
  }

  private createSystemReminder(contextInfo: ContextInfo): Message {
    const content = `<system-reminder>
As you answer the user's questions, you can use the following context:
${Object.entries(contextInfo).map(([key, value]) =>
  `# ${key}\n${value}`
).join('\n')}

IMPORTANT: this context may or may not be relevant to your tasks. You should not respond to this context or otherwise consider it in your response unless it is highly relevant to your task. Most of the time, it is not relevant.
</system-reminder>`;

    return {
      id: generateId(),
      type: 'user',
      role: 'user',
      content,
      timestamp: new Date().toISOString(),
      isMeta: true,
      uuid: generateUUID()
    };
  }
}
```

### 4.2 지능형 Tool 스케줄링 알고리즘

```typescript
// hW5 함수 기반 지능형 스케줄링 구현
class ToolScheduler {
  scheduleTools(toolCalls: ToolCall[]): ToolExecutionPlan {
    // 1. 종속성 분석
    const dependencies = this.analyzeDependencies(toolCalls);

    // 2. 동시성 안전성 그룹화
    const safetyGroups = this.groupBySafety(toolCalls);

    // 3. 리소스 요구사항 분석
    const resourceGroups = this.groupByResource(toolCalls);

    // 4. 스케줄 순서 최적화
    const executionGroups = this.optimizeSchedule(
      dependencies,
      safetyGroups,
      resourceGroups
    );

    return {
      groups: executionGroups,
      estimatedDuration: this.estimateDuration(executionGroups),
      resourceUsage: this.estimateResourceUsage(executionGroups)
    };
  }

  private analyzeDependencies(toolCalls: ToolCall[]): DependencyGraph {
    const graph = new DependencyGraph();

    for (let i = 0; i < toolCalls.length; i++) {
      for (let j = i + 1; j < toolCalls.length; j++) {
        if (this.hasDependency(toolCalls[i], toolCalls[j])) {
          graph.addEdge(toolCalls[i].id, toolCalls[j].id);
        }
      }
    }

    return graph;
  }

  private hasDependency(toolA: ToolCall, toolB: ToolCall): boolean {
    // 파일 종속성 확인
    if (this.hasFileConflict(toolA, toolB)) return true;

    // 순서 종속성 확인
    if (this.hasSequentialDependency(toolA, toolB)) return true;

    // 리소스 종속성 확인
    if (this.hasResourceConflict(toolA, toolB)) return true;

    return false;
  }
}
```

### 4.3 오류 복구 알고리즘

```typescript
class ErrorRecoveryManager {
  async handleError(error: Error, context: ExecutionContext): Promise<RecoveryAction> {
    // 오류 분류
    const errorType = this.classifyError(error);

    switch (errorType) {
      case ErrorType.MODEL_ERROR:
        return this.handleModelError(error as ModelError, context);

      case ErrorType.TOOL_ERROR:
        return this.handleToolError(error as ToolError, context);

      case ErrorType.NETWORK_ERROR:
        return this.handleNetworkError(error as NetworkError, context);

      case ErrorType.PERMISSION_ERROR:
        return this.handlePermissionError(error as PermissionError, context);

      default:
        return this.handleGenericError(error, context);
    }
  }

  private async handleModelError(error: ModelError, context: ExecutionContext): Promise<RecoveryAction> {
    if (context.config.fallbackModel) {
      // 모델 폴백
      await this.switchToFallbackModel(context.config.fallbackModel);

      // 현재 메시지 배치 지우기
      context.clearCurrentBatch();

      // 원격 측정 전송
      this.sendTelemetry('model_fallback_triggered', {
        originalModel: error.originalModel,
        fallbackModel: context.config.fallbackModel,
        error: error.message
      });

      return RecoveryAction.RETRY_WITH_FALLBACK;
    }

    return RecoveryAction.FAIL;
  }

  private async handleToolError(error: ToolError, context: ExecutionContext): Promise<RecoveryAction> {
    // Tool 수준 오류 복구
    if (error.isRetryable && context.retryCount < 3) {
      await this.delay(Math.pow(2, context.retryCount) * 1000); // 지수 백오프
      return RecoveryAction.RETRY;
    }

    // Tool을 실패로 표시하지만 다른 Tool 계속 실행
    context.markToolAsFailed(error.toolName);
    return RecoveryAction.CONTINUE_PARTIAL;
  }
}
```

## 5. 데이터베이스 및 스토리지 설계

### 5.1 세션 스토리지 방안

```typescript
interface SessionStorage {
  // 로컬 파일 기반 세션 스토리지
  sessionsDir: string;      // ~/.claude/sessions/
  configDir: string;       // ~/.claude/
  cacheDir: string;        // ~/.claude/cache/
}

class SessionManager {
  private storage: SessionStorage;

  async saveSession(session: Session): Promise<void> {
    const sessionPath = path.join(this.storage.sessionsDir, `${session.id}.json`);
    const sessionData = {
      ...session,
      messages: await this.compressMessages(session.messages)
    };

    await fs.writeFile(sessionPath, JSON.stringify(sessionData, null, 2));
  }

  async loadSession(sessionId: string): Promise<Session> {
    const sessionPath = path.join(this.storage.sessionsDir, `${sessionId}.json`);
    const data = await fs.readFile(sessionPath, 'utf-8');
    const sessionData = JSON.parse(data);

    return {
      ...sessionData,
      messages: await this.decompressMessages(sessionData.messages)
    };
  }

  async listSessions(): Promise<SessionMetadata[]> {
    const files = await fs.readdir(this.storage.sessionsDir);
    const sessions: SessionMetadata[] = [];

    for (const file of files) {
      if (file.endsWith('.json')) {
        const metadata = await this.getSessionMetadata(file);
        sessions.push(metadata);
      }
    }

    return sessions.sort((a, b) => b.updatedAt.localeCompare(a.updatedAt));
  }
}
```

### 5.2 구성 관리 방안

```typescript
// 3단계 구성 시스템 구현
class ConfigurationManager {
  private localConfig: Config | null = null;
  private projectConfig: Config | null = null;
  private userConfig: Config | null = null;

  async loadConfiguration(): Promise<Config> {
    // 우선순위에 따라 구성 로드
    await this.loadUserConfig();      // ~/.claude/settings.json
    await this.loadProjectConfig();   // ./.claude/settings.json
    await this.loadLocalConfig();     // ./CLAUDE.md

    // 구성 병합
    return this.mergeConfigurations();
  }

  private mergeConfigurations(): Config {
    return {
      ...this.userConfig,
      ...this.projectConfig,
      ...this.localConfig,
      // MCP 구성 특수 처리
      mcp: {
        ...this.userConfig?.mcp,
        ...this.projectConfig?.mcp,
        ...this.localConfig?.mcp
      }
    };
  }

  async saveMCPConfiguration(config: MCPConfig): Promise<void> {
    const configPath = path.join(process.cwd(), '.mcp.json');
    await fs.writeFile(configPath, JSON.stringify(config, null, 2));
  }
}
```

## 6. 보안 메커니즘 설계

### 6.1 권한 제어 시스템

```typescript
class PermissionManager {
  async checkToolPermission(tool: Tool, input: any, context: ToolContext): Promise<PermissionResult> {
    // 1. 내장 권한 확인
    const builtinCheck = await this.checkBuiltinPermissions(tool, input);
    if (builtinCheck.behavior !== 'allow') {
      return builtinCheck;
    }

    // 2. 사용자 구성 권한 확인
    const userCheck = await this.checkUserPermissions(tool, input, context);
    if (userCheck.behavior !== 'allow') {
      return userCheck;
    }

    // 3. 동적 보안 분석
    const securityCheck = await this.performSecurityAnalysis(tool, input);
    if (securityCheck.behavior !== 'allow') {
      return securityCheck;
    }

    return { behavior: 'allow', updatedInput: input };
  }

  private async performSecurityAnalysis(tool: Tool, input: any): Promise<PermissionResult> {
    // 위험한 명령 감지
    if (tool.name === 'Bash') {
      const dangerousPatterns = [
        /rm\s+-rf\s+\//,
        /sudo\s+rm/,
        /dd\s+if=.*of=\/dev/,
        /:(){ :|:& };:/  // fork bomb
      ];

      for (const pattern of dangerousPatterns) {
        if (pattern.test(input.command)) {
          return {
            behavior: 'deny',
            message: `Dangerous command detected: ${input.command}`,
            ruleSuggestions: ['Review the command for safety', 'Use safer alternatives']
          };
        }
      }
    }

    // 파일 작업 보안 확인
    if (['Write', 'Edit'].includes(tool.name)) {
      if (this.isSystemFile(input.filePath)) {
        return {
          behavior: 'ask',
          message: `Attempting to modify system file: ${input.filePath}`,
          confirmation: 'This operation may affect system stability. Continue?'
        };
      }
    }

    return { behavior: 'allow', updatedInput: input };
  }
}
```

### 6.2 샌드박스 실행 환경

```typescript
class SandboxExecutor {
  async executeTool(tool: Tool, input: any, options: SandboxOptions): Promise<ToolResult> {
    const sandbox = await this.createSandbox(options);

    try {
      // 샌드박스에서 Tool 실행
      const result = await sandbox.execute(async () => {
        return await tool.execute(input, sandbox.context);
      });

      // 결과 안전성 검증
      await this.validateResult(result);

      return result;
    } finally {
      await sandbox.destroy();
    }
  }

  private async createSandbox(options: SandboxOptions): Promise<Sandbox> {
    return new Sandbox({
      // 파일 시스템 제한
      allowedPaths: options.allowedPaths || [process.cwd()],
      readOnlyPaths: options.readOnlyPaths || ['/etc', '/usr'],

      // 네트워크 제한
      allowNetworking: options.allowNetworking || false,
      allowedHosts: options.allowedHosts || [],

      // 리소스 제한
      maxMemory: options.maxMemory || 512 * 1024 * 1024, // 512MB
      maxCpuTime: options.maxCpuTime || 30000, // 30초

      // 프로세스 제한
      allowSubprocesses: options.allowSubprocesses || false,
      maxProcesses: options.maxProcesses || 10
    });
  }
}
```

## 7. 성능 최적화 전략

### 7.1 동시 처리 최적화

```typescript
class PerformanceOptimizer {
  // gW5 복잡한 동시성 관리 메커니즘 기반 최적화
  private readonly concurrencyManager: gW5ConcurrencyManager;
  private dynamicSemaphore: DynamicSemaphore;

  async optimizeConcurrentExecution(tasks: Task[]): Promise<TaskResult[]> {
    // 1. 작업 분석 및 그룹화
    const groups = this.analyzeAndGroup(tasks);

    // 2. 지능형 동시 실행 최적화 (동적 로드 밸런싱)
    const results: TaskResult[] = [];
    for (const group of groups) {
      const concurrencyConfig = this.concurrencyManager.calculateOptimalConcurrency(group);
      const groupResults = await this.executeConcurrentGroup(group, concurrencyConfig);
      results.push(...groupResults);
    }

    return results;
  }

  private async executeConcurrentGroup(tasks: Task[]): Promise<TaskResult[]> {
    // 세마포어를 사용하여 동시성 수 제어
    return Promise.all(tasks.map(async (task) => {
      await this.semaphore.acquire();
      try {
        return await this.executeWithTimeout(task);
      } finally {
        this.semaphore.release();
      }
    }));
  }

  private async executeWithTimeout(task: Task, timeout = 30000): Promise<TaskResult> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Task timeout: ${task.name}`));
      }, timeout);

      task.execute()
        .then(resolve)
        .catch(reject)
        .finally(() => clearTimeout(timer));
    });
  }
}
```

### 7.2 메모리 관리 최적화

```typescript
class MemoryManager {
  private cache = new LRUCache<string, any>({ max: 1000 });
  private messageHistory: Message[] = [];
  private readonly maxHistorySize = 1000;

  optimizeMemoryUsage(): void {
    // 1. 메시지 히스토리 관리
    this.pruneMessageHistory();

    // 2. 캐시 정리
    this.cleanupCache();

    // 3. 강제 가비지 컬렉션 (개발 모드)
    if (process.env.NODE_ENV === 'development') {
      if (global.gc) {
        global.gc();
      }
    }
  }

  private pruneMessageHistory(): void {
    if (this.messageHistory.length > this.maxHistorySize) {
      // 최근 메시지와 중요한 시스템 메시지 보존
      const recentMessages = this.messageHistory.slice(-800);
      const systemMessages = this.messageHistory
        .slice(0, -800)
        .filter(m => m.isMeta || m.type === 'system');

      this.messageHistory = [...systemMessages, ...recentMessages];
    }
  }

  private cleanupCache(): void {
    // 만료된 캐시 항목 정리
    const now = Date.now();
    for (const [key, value] of this.cache.entries()) {
      if (value.timestamp && now - value.timestamp > 3600000) { // 1시간
        this.cache.delete(key);
      }
    }
  }
}
```

## 8. 모니터링 및 원격 측정 구현

### 8.1 원격 측정 데이터 수집

```typescript
// E1 함수 기반 원격 측정 구현
class TelemetryManager {
  private events: TelemetryEvent[] = [];
  private batchSize = 100;
  private flushInterval = 30000; // 30초

  logEvent(eventName: string, properties: Record<string, any>): void {
    const event: TelemetryEvent = {
      name: eventName,
      properties,
      timestamp: Date.now(),
      sessionId: this.getSessionId(),
      userId: this.getUserId()
    };

    this.events.push(event);

    // 일괄 전송
    if (this.events.length >= this.batchSize) {
      this.flush();
    }
  }

  // 핵심 이벤트 기록
  logToolUsage(toolName: string, duration: number, success: boolean): void {
    this.logEvent('tool_usage', {
      toolName,
      duration,
      success,
      concurrentTools: this.getConcurrentToolCount()
    });
  }

  logModelFallback(originalModel: string, fallbackModel: string): void {
    this.logEvent('model_fallback', {
      originalModel,
      fallbackModel,
      reason: 'api_error'
    });
  }

  logContextSize(size: ContextSizeMetrics): void {
    this.logEvent('context_size', {
      directoryStructureSize: size.directoryStructure,
      gitStatusSize: size.gitStatus,
      claudeMdSize: size.claudeMd,
      totalSize: size.total,
      projectFileCount: size.projectFileCount
    });
  }
}
```

### 8.2 성능 모니터링

```typescript
class PerformanceMonitor {
  private metrics = new Map<string, PerformanceMetric>();

  startOperation(operationId: string, operationType: string): void {
    this.metrics.set(operationId, {
      type: operationType,
      startTime: performance.now(),
      memoryStart: process.memoryUsage()
    });
  }

  endOperation(operationId: string, success: boolean = true): PerformanceMetric {
    const metric = this.metrics.get(operationId);
    if (!metric) {
      throw new Error(`Operation ${operationId} not found`);
    }

    const endTime = performance.now();
    const memoryEnd = process.memoryUsage();

    const completedMetric: PerformanceMetric = {
      ...metric,
      endTime,
      duration: endTime - metric.startTime,
      memoryEnd,
      memoryDelta: {
        rss: memoryEnd.rss - metric.memoryStart.rss,
        heapUsed: memoryEnd.heapUsed - metric.memoryStart.heapUsed,
        heapTotal: memoryEnd.heapTotal - metric.memoryStart.heapTotal
      },
      success
    };

    this.metrics.delete(operationId);
    this.recordMetric(completedMetric);

    return completedMetric;
  }
}
```

## 9. 테스트 전략 설계

### 9.1 단위 테스트 설계

```typescript
// Tool 테스트 예제
describe('ReadTool', () => {
  let readTool: ReadTool;
  let mockFileSystem: MockFileSystem;

  beforeEach(() => {
    mockFileSystem = new MockFileSystem();
    readTool = new ReadTool(mockFileSystem);
  });

  test('should read file content successfully', async () => {
    // Arrange
    const filePath = '/test/file.txt';
    const expectedContent = 'Hello, World!';
    mockFileSystem.setFileContent(filePath, expectedContent);

    // Act
    const result = await readTool.execute({ filePath });

    // Assert
    expect(result.success).toBe(true);
    expect(result.content).toBe(expectedContent);
  });

  test('should handle file not found error', async () => {
    // Arrange
    const filePath = '/nonexistent/file.txt';

    // Act & Assert
    await expect(readTool.execute({ filePath }))
      .rejects.toThrow('File not found');
  });
});

// Agent 주 루프 테스트
describe('AgentCore', () => {
  let agentCore: AgentCore;
  let mockLLMClient: MockLLMClient;

  beforeEach(() => {
    mockLLMClient = new MockLLMClient();
    agentCore = new AgentCore(mockLLMClient);
  });

  test('should handle recursive calls correctly', async () => {
    // 재귀 호출 메커니즘 테스트
    const messages = [{ role: 'user', content: 'Test message' }];
    const responses: any[] = [];

    for await (const chunk of agentCore.mainLoop(messages, config, context)) {
      responses.push(chunk);
      if (responses.length > 5) break; // 무한 루프 방지
    }

    expect(responses.length).toBeGreaterThan(0);
  });
});
```

### 9.2 통합 테스트 설계

```typescript
// MCP 통합 테스트
describe('MCP Integration', () => {
  let mcpManager: MCPManager;
  let testServer: TestMCPServer;

  beforeAll(async () => {
    testServer = new TestMCPServer();
    await testServer.start();

    mcpManager = new MCPManager();
    await mcpManager.addServer('test-server', {
      transport: 'http',
      baseUrl: testServer.url
    });
  });

  afterAll(async () => {
    await testServer.stop();
  });

  test('should discover and execute MCP tools', async () => {
    // Tool 발견 테스트
    const tools = await mcpManager.discoverTools('test-server');
    expect(tools.length).toBeGreaterThan(0);

    // Tool 실행 테스트
    const result = await mcpManager.executeTool('test-server', 'echo', {
      message: 'Hello MCP'
    });
    expect(result.output).toBe('Hello MCP');
  });
});
```

### 9.3 엔드투엔드 테스트 설계

```typescript
// 완전한 사용자 플로우 테스트
describe('End-to-End User Flows', () => {
  let app: CLIApplication;

  beforeEach(async () => {
    app = new CLIApplication();
    await app.initialize();
  });

  test('complete coding assistance workflow', async () => {
    // 1. 사용자가 프로그래밍 작업 입력
    const userRequest = "피보나치 수열을 계산하는 함수 생성";
    const response = await app.processUserInput(userRequest);

    // 2. 시스템이 관련 Tool 호출 확인
    expect(response.toolsUsed).toContain('Write');
    expect(response.filesCreated.length).toBeGreaterThan(0);

    // 3. 생성된 코드 검증
    const generatedCode = response.filesCreated[0].content;
    expect(generatedCode).toContain('fibonacci');
    expect(generatedCode).toContain('function');

    // 4. 코드 실행 테스트
    const testResult = await app.processUserInput("방금 생성한 코드 실행");
    expect(testResult.success).toBe(true);
  });

  test('session management workflow', async () => {
    // 세션 저장 및 복원 테스트
    const sessionId = await app.saveCurrentSession();
    expect(sessionId).toBeDefined();

    await app.clearSession();
    const restoredSession = await app.resumeSession(sessionId);

    expect(restoredSession.messages.length).toBeGreaterThan(0);
  });
});
```

## 10. 배포 및 운영 설계

### 10.1 빌드 및 패키징

```typescript
// 빌드 구성
const buildConfig: BuildConfig = {
  entry: 'src/cli.ts',
  output: {
    path: 'dist',
    filename: 'claude.js'
  },
  target: 'node18',
  externals: {
    // 대형 종속성 제외, 동적 임포트 사용
    'sharp': 'commonjs sharp',
    'puppeteer': 'commonjs puppeteer'
  },
  optimization: {
    minimize: true,
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};

// npm 게시 구성
const packageConfig = {
  name: 'open-claude-code',
  version: '1.0.0',
  bin: {
    'claude': './dist/claude.js'
  },
  engines: {
    node: '>=18.0.0'
  },
  dependencies: {
    // 런타임에 필요한 종속성만 포함
  },
  peerDependencies: {
    // 선택적인 향상 기능 종속성
  }
};
```

### 10.2 CI/CD 파이프라인

```yaml
# GitHub Actions 구성 예제
name: Build and Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run tests
      run: npm test

    - name: Build application
      run: npm run build

    - name: E2E tests
      run: npm run test:e2e

  publish:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18
        registry-url: 'https://registry.npmjs.org'

    - name: Install dependencies
      run: npm ci

    - name: Build
      run: npm run build

    - name: Publish to npm
      run: npm publish
      env:
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 10.3 모니터링 및 운영

```typescript
// 상태 확인 서비스
class HealthCheckService {
  async performHealthCheck(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkLLMAPIConnection(),
      this.checkMCPServers(),
      this.checkFileSystemAccess(),
      this.checkMemoryUsage(),
      this.checkDiskSpace()
    ]);

    const results = checks.map((check, index) => ({
      name: this.checkNames[index],
      status: check.status === 'fulfilled' ? 'healthy' : 'unhealthy',
      details: check.status === 'fulfilled' ? check.value : check.reason
    }));

    return {
      overall: results.every(r => r.status === 'healthy') ? 'healthy' : 'degraded',
      checks: results,
      timestamp: new Date().toISOString()
    };
  }

  async checkLLMAPIConnection(): Promise<APIHealthStatus> {
    try {
      const response = await this.llmClient.ping();
      return {
        status: 'healthy',
        responseTime: response.responseTime,
        model: response.model
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        error: error.message
      };
    }
  }
}
```

이 TDD 문서는 Open Claude Code 프로젝트의 완전한 기술 구현 프레임워크를 제공하며, 핵심 아키텍처부터 구체적인 구현까지 모든 기술 세부 사항을 다룹니다. 이 문서를 기반으로 개발 팀은 프로젝트 개발을 신속하게 시작하고 기술 구현의 일관성과 품질을 보장할 수 있습니다.

---

*본 TDD는 Claude Code v1.0.34의 심층 역공학 분석을 기반으로 하여 기술 솔루션의 실현 가능성과 정확성을 보장합니다.*
