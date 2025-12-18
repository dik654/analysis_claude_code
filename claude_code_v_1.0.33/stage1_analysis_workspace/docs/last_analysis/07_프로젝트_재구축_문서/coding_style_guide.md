# Open Claude Code - 코딩 스타일 가이드

## 1. 전체 원칙

### 1.1 설계 철학
- **일관성이 유연성보다 우선**: 코드 스타일의 일관성 유지
- **가독성이 간결성보다 우선**: 코드의 가독성과 유지보수성 우선 고려
- **타입 안전성이 동적성보다 우선**: TypeScript의 타입 시스템 충분히 활용
- **테스트 주도가 사후 테스트보다 우선**: 테스트 케이스 작성 우선

### 1.2 코드 품질 기준
- **테스트 커버리지**: ≥ 80%
- **순환 복잡도**: 단일 함수 ≤ 10
- **파일 길이**: 단일 파일 ≤ 500줄
- **함수 길이**: 단일 함수 ≤ 50줄

## 2. TypeScript 코딩 규범

### 2.1 명명 규칙

#### 2.1.1 파일 및 디렉토리 명명
```typescript
// ✅ 권장 - kebab-case
src/
├── agent-core/
│   ├── main-loop.ts
│   ├── message-compressor.ts
│   └── context-injector.ts
├── tool-system/
│   ├── tool-registry.ts
│   ├── execution-engine.ts
│   └── built-in-tools/
└── ui-components/
    ├── terminal-renderer.ts
    └── progress-indicator.ts

// ❌ 피하기
src/
├── AgentCore/           // PascalCase 디렉토리
├── tool_system/         // snake_case 디렉토리
├── UI_Components/       // 혼합 명명
```

#### 2.1.2 변수 및 함수 명명
```typescript
// ✅ 권장 - camelCase
const messageHistory: Message[] = [];
const toolExecutionEngine = new ToolExecutionEngine();

function processUserInput(input: string): Promise<ProcessingResult> {
  // 구현
}

async function compressMessages(messages: Message[]): Promise<CompressedMessages> {
  // 구현
}

// ❌ 피하기
const message_history = [];        // snake_case
const ToolExecutionEngine = {};    // PascalCase 변수
function ProcessUserInput() {}     // PascalCase 함수
```

#### 2.1.3 클래스 및 인터페이스 명명
```typescript
// ✅ 권장 - PascalCase
class AgentCore {
  private messageProcessor: MessageProcessor;
}

interface ToolExecutionContext {
  sessionId: string;
  permissions: Permission[];
}

interface MCPServerConfig {
  name: string;
  transport: TransportType;
}

// 타입 별칭
type MessageHandler = (message: Message) => Promise<void>;
type ToolResult = Success | Failure;

// 열거형
enum ExecutionStatus {
  PENDING = 'pending',
  RUNNING = 'running',
  COMPLETED = 'completed',
  FAILED = 'failed'
}
```

#### 2.1.4 상수 명명
```typescript
// ✅ 권장 - SCREAMING_SNAKE_CASE
const MAX_CONCURRENT_TOOLS = 10;
const DEFAULT_TIMEOUT_MS = 30000;
const SUPPORTED_MODELS = ['claude-3-5-sonnet', 'gpt-4o'] as const;

// 구성 객체
const DEFAULT_CONFIG = {
  maxConcurrency: 10,
  timeoutMs: 30000,
  retryAttempts: 3
} as const;
```

### 2.2 타입 정의 규범

#### 2.2.1 인터페이스 설계
```typescript
// ✅ 권장 - 명확한 인터페이스 정의
interface Message {
  readonly id: string;
  readonly type: 'user' | 'assistant' | 'system';
  readonly content: string | ContentBlock[];
  readonly timestamp: string;
  readonly isMeta?: boolean;
}

interface ToolExecutionOptions {
  readonly timeout?: number;
  readonly retryAttempts?: number;
  readonly permissions?: Permission[];
  readonly sandbox?: SandboxConfig;
}

// 제네릭을 사용하여 재사용성 향상
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<boolean>;
}

// ❌ 피하기 - 너무 광범위한 타입
interface Config {
  [key: string]: any;  // any 타입 피하기
}
```

#### 2.2.2 유니온 타입 및 판별 유니온
```typescript
// ✅ 권장 - 판별 유니온 타입
interface SuccessResult {
  readonly success: true;
  readonly data: any;
}

interface ErrorResult {
  readonly success: false;
  readonly error: string;
  readonly code: number;
}

type Result = SuccessResult | ErrorResult;

// 타입 가드
function isSuccessResult(result: Result): result is SuccessResult {
  return result.success === true;
}

// 사용 예제
function handleResult(result: Result): void {
  if (isSuccessResult(result)) {
    console.log(result.data);  // TypeScript가 여기는 SuccessResult임을 인식
  } else {
    console.error(result.error);  // TypeScript가 여기는 ErrorResult임을 인식
  }
}
```

#### 2.2.3 제네릭 제약
```typescript
// ✅ 권장 - 제네릭 제약 사용
interface Identifiable {
  id: string;
}

interface Timestamped {
  createdAt: string;
  updatedAt: string;
}

class EntityManager<T extends Identifiable & Timestamped> {
  private entities = new Map<string, T>();

  add(entity: T): void {
    this.entities.set(entity.id, entity);
  }

  findRecent(limit: number): T[] {
    return Array.from(this.entities.values())
      .sort((a, b) => b.updatedAt.localeCompare(a.updatedAt))
      .slice(0, limit);
  }
}
```

### 2.3 함수 설계 규범

#### 2.3.1 함수 시그니처 설계
```typescript
// ✅ 권장 - 명확한 함수 시그니처
interface CompressMessageOptions {
  maxTokens?: number;
  preserveSystemMessages?: boolean;
  compressionRatio?: number;
}

async function compressMessages(
  messages: readonly Message[],
  options: CompressMessageOptions = {}
): Promise<{
  compressedMessages: Message[];
  compressionRatio: number;
  tokensReduced: number;
}> {
  // 구현
}

// ✅ 권장 - 함수 오버로드 사용
function createTool(name: string, handler: ToolHandler): Tool;
function createTool(config: ToolConfig): Tool;
function createTool(
  nameOrConfig: string | ToolConfig,
  handler?: ToolHandler
): Tool {
  // 구현
}

// ❌ 피하기 - 파라미터가 너무 많음
function badFunction(a: string, b: number, c: boolean, d: object, e: any[]): void {
  // 3-4개 파라미터를 초과하지 않기
}
```

#### 2.3.2 오류 처리 규범
```typescript
// ✅ 권장 - 명확한 오류 타입
class ToolExecutionError extends Error {
  constructor(
    message: string,
    public readonly toolName: string,
    public readonly input: unknown,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'ToolExecutionError';
  }
}

class MCPConnectionError extends Error {
  constructor(
    message: string,
    public readonly serverName: string,
    public readonly transport: string,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'MCPConnectionError';
  }
}

// 오류 처리 함수
async function executeToolSafely(tool: Tool, input: unknown): Promise<Result> {
  try {
    const result = await tool.execute(input);
    return { success: true, data: result };
  } catch (error) {
    if (error instanceof ToolExecutionError) {
      return {
        success: false,
        error: error.message,
        code: 'TOOL_EXECUTION_FAILED',
        details: { toolName: error.toolName }
      };
    }
    throw error; // 알 수 없는 오류 재throw
  }
}
```

## 3. React/UI 컴포넌트 규범

### 3.1 컴포넌트 설계 원칙

#### 3.1.1 함수형 컴포넌트 우선
```typescript
// ✅ 권장 - 함수형 컴포넌트 + hooks
interface MessageDisplayProps {
  readonly messages: Message[];
  readonly onMessageClick?: (message: Message) => void;
  readonly loading?: boolean;
}

const MessageDisplay: React.FC<MessageDisplayProps> = ({
  messages,
  onMessageClick,
  loading = false
}) => {
  const [expandedMessages, setExpandedMessages] = useState<Set<string>>(new Set());

  const toggleExpanded = useCallback((messageId: string) => {
    setExpandedMessages(prev => {
      const next = new Set(prev);
      if (next.has(messageId)) {
        next.delete(messageId);
      } else {
        next.add(messageId);
      }
      return next;
    });
  }, []);

  return (
    <Box flexDirection="column">
      {messages.map(message => (
        <MessageItem
          key={message.id}
          message={message}
          expanded={expandedMessages.has(message.id)}
          onToggle={toggleExpanded}
          onClick={onMessageClick}
        />
      ))}
      {loading && <LoadingIndicator />}
    </Box>
  );
};
```

#### 3.1.2 Props 인터페이스 설계
```typescript
// ✅ 권장 - 명확한 Props 타입
interface BaseComponentProps {
  readonly className?: string;
  readonly testId?: string;
}

interface ProgressBarProps extends BaseComponentProps {
  readonly progress: number;  // 0-100
  readonly label?: string;
  readonly variant?: 'determinate' | 'indeterminate';
  readonly size?: 'small' | 'medium' | 'large';
  readonly onComplete?: () => void;
}

// 제네릭 Props 사용
interface ListProps<T> extends BaseComponentProps {
  readonly items: readonly T[];
  readonly renderItem: (item: T, index: number) => React.ReactNode;
  readonly keyExtractor: (item: T) => string;
  readonly emptyMessage?: string;
}
```

### 3.2 Hooks 사용 규범

#### 3.2.1 커스텀 Hooks
```typescript
// ✅ 권장 - 커스텀 hooks
function useTerminalSize() {
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

    process.stdout.on('resize', handleResize);
    return () => process.stdout.off('resize', handleResize);
  }, []);

  return size;
}

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

#### 3.2.2 성능 최적화
```typescript
// ✅ 권장 - useMemo 및 useCallback 사용하여 최적화
const ExpensiveComponent: React.FC<Props> = ({ data, onUpdate }) => {
  // 계산 결과 캐싱
  const processedData = useMemo(() => {
    return data.map(item => ({
      ...item,
      processed: expensiveProcessing(item)
    }));
  }, [data]);

  // 이벤트 핸들러 캐싱
  const handleUpdate = useCallback((id: string, value: any) => {
    onUpdate(id, value);
  }, [onUpdate]);

  // 렌더링 결과 캐싱
  const renderedItems = useMemo(() => {
    return processedData.map(item => (
      <ItemComponent
        key={item.id}
        item={item}
        onUpdate={handleUpdate}
      />
    ));
  }, [processedData, handleUpdate]);

  return <Box>{renderedItems}</Box>;
};
```

## 4. 아키텍처 패턴 규범

### 4.1 모듈 조직 규범

#### 4.1.1 디렉토리 구조
```
src/
├── cli/                    # CLI 진입점 및 명령 처리
│   ├── commands/
│   ├── parsers/
│   └── main.ts
├── core/                   # 핵심 비즈니스 로직
│   ├── agent/              # Agent 주 루프
│   ├── message/            # 메시지 처리
│   ├── compression/        # 메시지 압축
│   └── context/            # 컨텍스트 관리
├── tools/                  # Tool 시스템
│   ├── registry/
│   ├── execution/
│   ├── built-in/
│   └── mcp/
├── ui/                     # 사용자 인터페이스
│   ├── components/
│   ├── hooks/
│   └── themes/
├── utils/                  # 유틸리티 함수
│   ├── logger.ts
│   ├── crypto.ts
│   └── file-system.ts
├── types/                  # 타입 정의
│   ├── core.ts
│   ├── tools.ts
│   └── mcp.ts
└── __tests__/              # 테스트 파일
    ├── unit/
    ├── integration/
    └── e2e/
```

#### 4.1.2 모듈 내보내기 규범
```typescript
// ✅ 권장 - index.ts 통일 내보내기
// src/tools/index.ts
export { ToolRegistry } from './registry/tool-registry';
export { ExecutionEngine } from './execution/execution-engine';
export { type Tool, type ToolResult } from './types';

// 분류 내보내기
export * as BuiltInTools from './built-in';
export * as MCPTools from './mcp';

// src/tools/built-in/index.ts
export { ReadTool } from './read-tool';
export { WriteTool } from './write-tool';
export { EditTool } from './edit-tool';
export { BashTool } from './bash-tool';

// 기본 내보내기 집계
export const DEFAULT_TOOLS = [
  ReadTool,
  WriteTool,
  EditTool,
  BashTool
] as const;
```

### 4.2 의존성 주입 규범

#### 4.2.1 컨테이너 설계
```typescript
// ✅ 권장 - 간단한 의존성 주입 컨테이너
class Container {
  private instances = new Map<string, any>();
  private factories = new Map<string, () => any>();

  register<T>(key: string, factory: () => T): void {
    this.factories.set(key, factory);
  }

  registerSingleton<T>(key: string, factory: () => T): void {
    this.register(key, () => {
      if (!this.instances.has(key)) {
        this.instances.set(key, factory());
      }
      return this.instances.get(key);
    });
  }

  resolve<T>(key: string): T {
    const factory = this.factories.get(key);
    if (!factory) {
      throw new Error(`No registration found for ${key}`);
    }
    return factory();
  }
}

// 서비스 등록
const container = new Container();

container.registerSingleton('logger', () => new Logger());
container.registerSingleton('config', () => loadConfiguration());
container.register('toolRegistry', () =>
  new ToolRegistry(container.resolve('logger'))
);
```

### 4.3 이벤트 시스템 규범

#### 4.3.1 이벤트 정의
```typescript
// ✅ 권장 - 타입 안전 이벤트 시스템
interface EventMap {
  'tool:started': { toolName: string; input: unknown };
  'tool:completed': { toolName: string; result: ToolResult; duration: number };
  'tool:failed': { toolName: string; error: Error };
  'message:received': { message: Message };
  'session:saved': { sessionId: string };
}

class TypedEventEmitter<T extends Record<string, any>> {
  private listeners = new Map<keyof T, Set<(data: any) => void>>();

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    const eventListeners = this.listeners.get(event);
    if (eventListeners) {
      eventListeners.forEach(listener => listener(data));
    }
  }

  off<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    const eventListeners = this.listeners.get(event);
    if (eventListeners) {
      eventListeners.delete(listener);
    }
  }
}

// 사용
const eventBus = new TypedEventEmitter<EventMap>();

eventBus.on('tool:completed', ({ toolName, result, duration }) => {
  console.log(`Tool ${toolName}이 ${duration}ms 안에 완료되었습니다`);
});
```

## 5. 테스트 규범

### 5.1 테스트 구조 규범

#### 5.1.1 테스트 파일 조직
```typescript
// ✅ 권장 - 소스 코드에 대응하는 테스트 구조
__tests__/
├── unit/
│   ├── core/
│   │   ├── agent-core.test.ts
│   │   └── message-compressor.test.ts
│   └── tools/
│       ├── read-tool.test.ts
│       └── execution-engine.test.ts
├── integration/
│   ├── mcp-integration.test.ts
│   └── tool-coordination.test.ts
└── e2e/
    ├── user-workflows.test.ts
    └── session-management.test.ts
```

#### 5.1.2 테스트 작성 규범
```typescript
// ✅ 권장 - 명확한 테스트 구조
describe('MessageCompressor', () => {
  let compressor: MessageCompressor;
  let mockLLMClient: jest.Mocked<LLMClient>;

  beforeEach(() => {
    mockLLMClient = createMockLLMClient();
    compressor = new MessageCompressor(mockLLMClient);
  });

  describe('shouldCompress', () => {
    it('should return true when message count exceeds threshold', async () => {
      // Arrange
      const messages = Array(101).fill(null).map((_, i) =>
        createTestMessage(`Message ${i}`)
      );

      // Act
      const result = await compressor.shouldCompress(messages);

      // Assert
      expect(result).toBe(true);
    });

    it('should return false for small message sets', async () => {
      // Arrange
      const messages = [createTestMessage('Single message')];

      // Act
      const result = await compressor.shouldCompress(messages);

      // Assert
      expect(result).toBe(false);
    });
  });

  describe('compress', () => {
    it('should preserve system messages during compression', async () => {
      // Arrange
      const messages = [
        createTestMessage('User message 1'),
        createSystemMessage('System reminder'),
        createTestMessage('User message 2')
      ];

      mockLLMClient.complete.mockResolvedValue(
        createMockCompletion('Compressed summary')
      );

      // Act
      const result = await compressor.compress(messages, {});

      // Assert
      expect(result.wasCompacted).toBe(true);
      expect(result.messages).toHaveLength(2); // summary + system message
      expect(result.messages.some(m => m.isMeta)).toBe(true);
    });
  });
});
```

### 5.2 Mock 및 테스트 도구

#### 5.2.1 팩토리 함수
```typescript
// ✅ 권장 - 테스트 데이터 팩토리
export class TestDataFactory {
  static createMessage(overrides: Partial<Message> = {}): Message {
    return {
      id: `msg-${Date.now()}-${Math.random()}`,
      type: 'user',
      role: 'user',
      content: 'Test message',
      timestamp: new Date().toISOString(),
      isMeta: false,
      uuid: crypto.randomUUID(),
      ...overrides
    };
  }

  static createToolCall(toolName: string, input: unknown = {}): ToolCall {
    return {
      id: `tool-${Date.now()}`,
      toolName,
      input,
      timestamp: new Date().toISOString()
    };
  }

  static createMCPServer(name: string): MCPServer {
    return {
      name,
      config: {
        command: 'test-server',
        transport: 'stdio'
      },
      status: 'connected',
      tools: [],
      resources: []
    };
  }
}
```

## 6. 로그 및 오류 처리 규범

### 6.1 로그 시스템

#### 6.1.1 로그 레벨 및 형식
```typescript
// ✅ 권장 - 구조화된 로그
enum LogLevel {
  ERROR = 0,
  WARN = 1,
  INFO = 2,
  DEBUG = 3,
  TRACE = 4
}

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  context?: Record<string, any>;
  error?: Error;
}

class Logger {
  constructor(
    private context: Record<string, any> = {},
    private minLevel: LogLevel = LogLevel.INFO
  ) {}

  error(message: string, error?: Error, context?: Record<string, any>): void {
    this.log(LogLevel.ERROR, message, { ...context, error });
  }

  info(message: string, context?: Record<string, any>): void {
    this.log(LogLevel.INFO, message, context);
  }

  debug(message: string, context?: Record<string, any>): void {
    this.log(LogLevel.DEBUG, message, context);
  }

  private log(level: LogLevel, message: string, context?: Record<string, any>): void {
    if (level > this.minLevel) return;

    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      context: { ...this.context, ...context }
    };

    console.log(JSON.stringify(entry));
  }

  child(context: Record<string, any>): Logger {
    return new Logger({ ...this.context, ...context }, this.minLevel);
  }
}

// 사용 예제
const logger = new Logger({ component: 'AgentCore' });
const toolLogger = logger.child({ tool: 'ReadTool' });

toolLogger.info('Tool execution started', {
  input: '/path/to/file.txt',
  timeout: 30000
});
```

### 6.2 오류 처리 패턴

#### 6.2.1 Result 패턴
```typescript
// ✅ 권장 - Result 패턴으로 예외 피하기
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

async function safeExecute<T>(
  operation: () => Promise<T>
): Promise<Result<T>> {
  try {
    const data = await operation();
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error(String(error))
    };
  }
}

// 사용 예제
async function processFile(path: string): Promise<Result<string>> {
  const readResult = await safeExecute(() => fs.readFile(path, 'utf-8'));
  if (!readResult.success) {
    return { success: false, error: readResult.error };
  }

  const processResult = await safeExecute(() => processContent(readResult.data));
  if (!processResult.success) {
    return { success: false, error: processResult.error };
  }

  return { success: true, data: processResult.data };
}
```

## 7. 성능 최적화 규범

### 7.1 비동기 작업 최적화

#### 7.1.1 동시성 제어
```typescript
// ✅ 권장 - 세마포어로 동시성 제어
class Semaphore {
  private permits: number;
  private waiting: Array<() => void> = [];

  constructor(permits: number) {
    this.permits = permits;
  }

  async acquire(): Promise<void> {
    if (this.permits > 0) {
      this.permits--;
      return;
    }

    return new Promise<void>(resolve => {
      this.waiting.push(resolve);
    });
  }

  release(): void {
    if (this.waiting.length > 0) {
      const next = this.waiting.shift()!;
      next();
    } else {
      this.permits++;
    }
  }
}

// 사용 예제
class ConcurrentExecutor {
  private semaphore = new Semaphore(10); // 최대 10개 동시 실행

  async executeMany<T>(
    tasks: Array<() => Promise<T>>
  ): Promise<T[]> {
    return Promise.all(tasks.map(task => this.executeOne(task)));
  }

  private async executeOne<T>(task: () => Promise<T>): Promise<T> {
    await this.semaphore.acquire();
    try {
      return await task();
    } finally {
      this.semaphore.release();
    }
  }
}
```

### 7.2 메모리 관리

#### 7.2.1 LRU 캐시
```typescript
// ✅ 권장 - LRU 캐시 구현
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;

  constructor(maxSize: number) {
    this.maxSize = maxSize;
  }

  get(key: K): V | undefined {
    const value = this.cache.get(key);
    if (value !== undefined) {
      // 맨 앞으로 이동
      this.cache.delete(key);
      this.cache.set(key, value);
    }
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // 가장 오래된 항목 삭제
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  clear(): void {
    this.cache.clear();
  }
}
```

## 8. 도구 구성

### 8.1 ESLint 구성
```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    '@typescript-eslint/recommended-requiring-type-checking'
  ],
  parserOptions: {
    ecmaVersion: 2022,
    sourceType: 'module',
    project: './tsconfig.json'
  },
  rules: {
    // 타입 안전성 강제
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unsafe-assignment': 'error',
    '@typescript-eslint/no-unsafe-call': 'error',

    // 명명 규칙
    '@typescript-eslint/naming-convention': [
      'error',
      {
        selector: 'interface',
        format: ['PascalCase']
      },
      {
        selector: 'class',
        format: ['PascalCase']
      },
      {
        selector: 'variable',
        format: ['camelCase', 'UPPER_CASE']
      }
    ],

    // 코드 품질
    'complexity': ['error', 10],
    'max-lines': ['error', 500],
    'max-params': ['error', 4],
    'max-depth': ['error', 4]
  }
};
```

### 8.2 Prettier 구성
```javascript
// .prettierrc.js
module.exports = {
  semi: true,
  trailingComma: 'es5',
  singleQuote: true,
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  bracketSpacing: true,
  arrowParens: 'avoid',
  endOfLine: 'lf'
};
```

### 8.3 TypeScript 구성
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "declaration": true,
    "outDir": "./dist",
    "sourceMap": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "__tests__"]
}
```

## 9. 코드 리뷰 체크리스트

### 9.1 기능 리뷰
- [ ] 코드가 예상 기능 구현
- [ ] 경계 조건 처리 정확
- [ ] 오류 처리 완전
- [ ] 성능이 요구사항 충족
- [ ] 보안 고려 충분

### 9.2 코드 품질 리뷰
- [ ] 명명이 명확하고 이해하기 쉬움
- [ ] 함수 책임이 단일
- [ ] 클래스 설계 합리적
- [ ] 종속성 관계 명확
- [ ] 중복 코드 제거됨

### 9.3 테스트 리뷰
- [ ] 단위 테스트가 주요 로직 커버
- [ ] 통합 테스트가 컴포넌트 협업 검증
- [ ] 경계 조건에 테스트 커버리지 있음
- [ ] 예외 상황에 테스트 검증 있음
- [ ] 테스트 코드 품질 양호

### 9.4 문서 리뷰
- [ ] 코드 주석이 명확하고 정확
- [ ] API 문서 완전
- [ ] 사용 예제 충분
- [ ] 변경 기록 적시 업데이트

## 10. 지속적 통합 및 품질 보증

### 10.1 사전 제출 검사
```bash
#!/bin/bash
# pre-commit hook

echo "Running pre-commit checks..."

# 1. 타입 검사
npm run type-check
if [ $? -ne 0 ]; then
  echo "❌ Type check failed"
  exit 1
fi

# 2. 코드 스타일 검사
npm run lint
if [ $? -ne 0 ]; then
  echo "❌ Linting failed"
  exit 1
fi

# 3. 포맷 검사
npm run format:check
if [ $? -ne 0 ]; then
  echo "❌ Format check failed"
  exit 1
fi

# 4. 테스트 실행
npm run test:unit
if [ $? -ne 0 ]; then
  echo "❌ Unit tests failed"
  exit 1
fi

echo "✅ All pre-commit checks passed"
```

### 10.2 품질 게이트
```yaml
# 품질 게이트 기준
quality_gates:
  test_coverage:
    threshold: 80%
    type: "line_coverage"

  code_smells:
    threshold: 0
    severity: "major"

  complexity:
    threshold: 10
    type: "cyclomatic"

  duplications:
    threshold: 3%
    type: "duplicated_lines"

  maintainability:
    threshold: "A"
    type: "maintainability_rating"
```

이 코딩 스타일 가이드는 Open Claude Code 프로젝트의 코드 품질과 일관성을 보장하며, 팀 협업과 프로젝트 유지보수를 위한 통일된 표준을 제공합니다.

---

*본 코딩 스타일 가이드는 TypeScript 및 현대적인 프론트엔드 개발 모범 사례를 기반으로 작성되었습니다.*
