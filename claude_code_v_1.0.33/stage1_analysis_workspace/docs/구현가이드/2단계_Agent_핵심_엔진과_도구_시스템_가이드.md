# 2단계: Agent 핵심 엔진과 Tool 시스템 가이드

## 📋 대상 독자
**본 문서 대상: 초급 수준의 개발자**
- 깊이 있는 사고 불필요, 단계별로 엄격히 따라 실행
- 각 단계마다 명확한 파일 작업 지침 포함
- 필요한 코드 템플릿과 설정 포함

## 🎯 단계 목표
역분석 결과를 기반으로 Claude Code의 핵심 차별화 기술 구현:
- ✅ **실시간 Steering 메커니즘** (h2A 비동기 메시지 큐 시스템)
- ✅ **Agent 메인 루프 엔진** (nO async generator 함수)
- ✅ **계층적 Multi-Agent 아키텍처** (Task Tool과 I2A SubAgent 인스턴스화)
- ✅ **Tool 실행 엔진** (MH1 Tool 엔진과 gW5 동시성 제어)
- ✅ **Edit Tool 강제 읽기 메커니즘** (9계층 검증과 readFileState 관리)

**예상 결과물**:
- ✅ h2A 비동기 메시지 큐 완전 구현
- ✅ nO 메인 Agent 루프 엔진
- ✅ 15개 내장 Tool 완전 구현
- ✅ Task Tool 계층적 Multi-Agent 아키텍처
- ✅ 엔터프라이즈급 보안 메커니즘

**작업 시간**: 3주 (120시간)

---

## 📁 1주차: 실시간 Steering 메커니즘과 Agent 핵심

### 단계 2.1: h2A 비동기 메시지 큐 시스템 생성
**역분석 기반 h2A 클래스 정확한 구현**

**파일 경로**: `src/core/message-queue.ts`
**파일 내용**:
```typescript
/**
 * h2A 비동기 메시지 큐 시스템
 * 역분석 기반 Claude Code h2A 클래스 정확한 구현
 * 실시간 Steering 메커니즘을 지원하는 핵심 컴포넌트
 */

export class h2A implements AsyncIterator<any> {
  private returned: (() => void) | null; // 정리 함수
  private queue: any[] = [];             // 메시지 큐 버퍼
  private readResolve?: (value: any) => void; // Promise resolve 콜백
  private readReject?: (reason: any) => void;  // Promise reject 콜백
  private isDone = false;                // 큐 완료 플래그
  private hasError?: any;                // 에러 상태
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

  // 핵심 비동기 반복자 메서드 - 역분석 기반 정확한 구현
  async next(): Promise<IteratorResult<any>> {
    // 큐에서 메시지 우선 처리
    if (this.queue.length > 0) {
      const value = this.queue.shift();
      return { value, done: false };
    }

    // 큐 완료 시 종료 플래그 반환
    if (this.isDone) {
      return { value: undefined, done: true };
    }

    // 에러 발생 시 Promise reject
    if (this.hasError) {
      throw this.hasError;
    }

    // 새 메시지 대기 - 핵심 논블로킹 메커니즘
    return new Promise((resolve, reject) => {
      this.readResolve = resolve;
      this.readReject = reject;
    });
  }

  // 메시지 큐 삽입 - 실시간 메시지 삽입 지원
  enqueue(message: any): void {
    if (this.isDone) return;

    if (this.readResolve) {
      // 대기 중인 읽기가 있으면 직접 메시지 반환
      const callback = this.readResolve;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback({ value: message, done: false });
    } else {
      // 그렇지 않으면 큐 버퍼에 추가
      this.queue.push(message);
    }
  }

  // 큐 완료
  complete(): void {
    this.isDone = true;
    if (this.readResolve) {
      const callback = this.readResolve;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback({ value: undefined, done: true });
    }

    // 정리 함수 실행
    if (this.returned) {
      this.returned();
    }
  }

  // 에러 처리
  error(err: any): void {
    this.hasError = err;
    if (this.readReject) {
      const callback = this.readReject;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback(err);
    }
  }

  // 상태 확인 메서드
  get isStarted(): boolean {
    return this.started;
  }

  get isCompleted(): boolean {
    return this.isDone;
  }

  get queueSize(): number {
    return this.queue.length;
  }
}

/**
 * 메시지 파서 g2A 클래스
 * 역분석 기반 스트리밍 메시지 파싱 구현
 */
export class g2A {
  private input: AsyncIterable<string>;
  private structuredInput: AsyncGenerator<any>;

  constructor(inputStream: AsyncIterable<string>) {
    this.input = inputStream;
    this.structuredInput = this.read();
  }

  // 비동기 제너레이터 - 스트리밍 입력 처리
  async *read(): AsyncGenerator<any> {
    let buffer = "";

    // 입력 스트림을 문자 단위로 처리
    for await (const chunk of this.input) {
      buffer += chunk;
      let lineEnd: number;

      // 라인 단위로 분할 처리
      while ((lineEnd = buffer.indexOf('\n')) !== -1) {
        const line = buffer.slice(0, lineEnd);
        buffer = buffer.slice(lineEnd + 1);

        const parsed = this.processLine(line);
        if (parsed) yield parsed;
      }
    }

    // 마지막 라인 처리
    if (buffer) {
      const parsed = this.processLine(buffer);
      if (parsed) yield parsed;
    }
  }

  // 단일 라인 메시지 파싱
  private processLine(line: string): any | null {
    try {
      const message = JSON.parse(line);

      // 엄격한 타입 검증 - 역분석 기반 검증 로직
      if (message.type !== "user") {
        throw new Error(`Expected message type 'user', got '${message.type}'`);
      }

      if (message.message?.role !== "user") {
        throw new Error(`Expected message role 'user', got '${message.message?.role}'`);
      }

      return message;
    } catch (error) {
      console.error(`Error parsing streaming input line: ${line}: ${error}`);
      // 역분석에서 파싱 실패 시 직접 프로세스 종료
      process.exit(1);
    }
  }

  // 구조화된 입력 스트림 가져오기
  getStructuredInput(): AsyncGenerator<any> {
    return this.structuredInput;
  }
}

/**
 * 실시간 Steering 리스너 시스템
 * 역분석 기반 stdin 리스닝 메커니즘
 */
export class SteeringListener {
  private steeringQueue: h2A;
  private stdinListener?: NodeJS.ReadStream;
  private isListening = false;

  constructor(steeringQueue: h2A) {
    this.steeringQueue = steeringQueue;
  }

  // stdin 실시간 리스닝 시작
  startListening(): void {
    if (this.isListening) return;

    this.stdinListener = process.stdin;
    this.stdinListener.setRawMode?.(true);
    this.stdinListener.resume();
    this.isListening = true;

    this.stdinListener.on('data', (chunk) => {
      try {
        const input = chunk.toString('utf8');

        // 실시간 Steering 입력 확인
        if (this.isSteeringInput(input)) {
          const steeringMessage = this.parseSteeringInput(input);
          this.steeringQueue.enqueue({
            type: 'steering',
            content: steeringMessage,
            timestamp: Date.now(),
            priority: 1 // 사용자 실시간 가이드 우선순위
          });
        }
      } catch (error) {
        console.error('Steering input parse error:', error);
      }
    });
  }

  // 리스닝 중지
  stopListening(): void {
    if (this.stdinListener && this.isListening) {
      this.stdinListener.pause();
      this.stdinListener.setRawMode?.(false);
      this.isListening = false;
    }
  }

  // 실시간 가이드 입력 여부 판단
  private isSteeringInput(input: string): boolean {
    // 역분석 기반 특수 제어 문자 감지
    return input.includes('\u001b') || // ESC 키
           input.charCodeAt(0) === 3 || // Ctrl+C
           input.includes('\r') ||       // Enter 키
           input.length > 1;             // 다중 문자 입력
  }

  // 실시간 가이드 입력 파싱
  private parseSteeringInput(input: string): string {
    // 특수 키 조합 처리
    if (input.includes('\u001b[')) {
      return this.handleSpecialKeys(input);
    }

    return input.trim();
  }

  // 특수 키 처리
  private handleSpecialKeys(input: string): string {
    // 간소화 처리: 정리된 입력 반환
    return input.replace(/\u001b\[[0-9;]*[a-zA-Z]/g, '').trim();
  }
}
```

### 단계 2.2: nO 메인 Agent 루프 엔진 생성
**역분석 기반 async generator 구현**

**파일 경로**: `src/core/agent-core.ts`
**파일 내용**:
```typescript
/**
 * Agent 핵심 엔진
 * 역분석 기반 nO async generator 함수 구현
 * 실시간 Steering 메커니즘과 모델 폴백 전략 통합
 */

import { h2A, SteeringListener } from './message-queue';
import type { Message, AgentConfig, AgentContext, AgentResult } from '../types/agent';
import type { ToolCall, ToolResult } from '../types/tool';

export class AgentCore {
  private steeringQueue: h2A;
  private steeringListener: SteeringListener;
  private abortController: AbortController;
  private messageHistory: Message[] = [];
  private config: AgentConfig;
  private context: AgentContext;

  constructor(config: AgentConfig, context: AgentContext) {
    this.config = config;
    this.context = context;
    this.abortController = new AbortController();
    this.steeringQueue = new h2A(this.cleanup.bind(this));
    this.steeringListener = new SteeringListener(this.steeringQueue);
  }

  /**
   * nO 메인 루프 - 역분석 기반 async generator 구현
   * 실시간 Steering과 모델 폴백 메커니즘 지원
   */
  async* executeMainLoop(
    messages: Message[],
    prompt?: string
  ): AsyncGenerator<AgentResult> {
    // 1. 실시간 Steering 메커니즘 시작
    if (this.config.enableSteering) {
      this.steeringListener.startListening();
    }

    // 2. 스트림 시작 마커
    yield {
      success: true,
      message: 'stream_request_start',
      data: { type: 'stream_start' }
    };

    let currentMessages = [...messages];
    if (prompt) {
      currentMessages.push({
        id: Date.now().toString(),
        role: 'user',
        content: prompt,
        timestamp: Date.now()
      });
    }

    // 3. 메시지 압축 확인
    const { messages: compactedMessages, wasCompacted } =
      await this.compressMessages(currentMessages);

    if (wasCompacted) {
      yield {
        success: true,
        message: 'context_compacted',
        data: {
          originalCount: currentMessages.length,
          compactedCount: compactedMessages.length
        }
      };
      currentMessages = compactedMessages;
    }

    let assistantResponses: Message[] = [];
    let currentModel = this.config.model;
    let shouldRetry = true;

    try {
      // 4. 메인 실행 루프 - 모델 폴백 재시도 지원
      while (shouldRetry) {
        shouldRetry = false;

        try {
          // 5. 실시간 Steering 입력 확인
          const steeringMessage = await this.checkSteeringInput();
          if (steeringMessage) {
            currentMessages.push(steeringMessage);
            yield {
              success: true,
              message: 'steering_input_received',
              data: { content: steeringMessage.content }
            };
          }

          // 6. 언어 모델 호출 처리
          for await (const response of this.processWithAI(
            currentMessages,
            currentModel,
            this.abortController.signal
          )) {
            // 실시간 중단 신호 확인
            if (this.abortController.signal.aborted) {
              yield {
                success: false,
                message: 'execution_aborted',
                error: new Error('Execution was aborted')
              };
              return;
            }

            yield response;

            if (response.data?.type === 'assistant') {
              assistantResponses.push(response.data as Message);
            }
          }
        } catch (error) {
          // 7. 모델 폴백 처리 - 역분석 기반 폴백 메커니즘
          if (this.isModelError(error) && this.config.fallbackModel) {
            currentModel = this.config.fallbackModel;
            shouldRetry = true;
            assistantResponses = [];

            yield {
              success: true,
              message: 'model_fallback_triggered',
              data: {
                originalModel: this.config.model,
                fallbackModel: this.config.fallbackModel,
                error: error.message
              }
            };
            continue;
          }
          throw error;
        }
      }
    } catch (error) {
      // 8. 에러 처리 및 Tool 결과 생성
      console.error('Agent execution error:', error);

      const errorMessage = error instanceof Error ? error.message : String(error);

      // 각 Tool 사용에 대한 에러 결과 생성
      let hasErrorResponse = false;
      for (const response of assistantResponses) {
        if (response.metadata?.toolCalls) {
          for (const toolCall of response.metadata.toolCalls) {
            yield {
              success: false,
              message: 'tool_error',
              data: {
                toolCallId: toolCall.id,
                error: errorMessage
              },
              error: error instanceof Error ? error : new Error(errorMessage)
            };
            hasErrorResponse = true;
          }
        }
      }

      if (!hasErrorResponse) {
        yield {
          success: false,
          message: 'general_error',
          error: error instanceof Error ? error : new Error(errorMessage)
        };
      }
      return;
    }

    // 9. Tool 호출 처리
    if (assistantResponses.length > 0) {
      const toolCalls = this.extractToolCalls(assistantResponses);

      if (toolCalls.length > 0) {
        // Tool 호출 실행
        yield* this.executeTools(toolCalls, currentMessages);

        // 대화 계속 여부 확인
        if (!this.abortController.signal.aborted && this.shouldContinue(toolCalls)) {
          // 재귀 호출로 대화 루프 계속
          const updatedMessages = [...currentMessages, ...assistantResponses];
          yield* this.executeMainLoop(updatedMessages);
        }
      }
    }
  }

  // 실시간 Steering 입력 확인
  private async checkSteeringInput(): Promise<Message | null> {
    if (!this.config.enableSteering) return null;

    try {
      const steeringData = await Promise.race([
        this.steeringQueue.next(),
        new Promise(resolve => setTimeout(() => resolve({ value: null, done: true }), 100))
      ]) as any;

      if (steeringData.value) {
        return {
          id: Date.now().toString(),
          role: 'user',
          content: steeringData.value.content,
          timestamp: Date.now(),
          metadata: { steeringMessage: true }
        };
      }
    } catch (error) {
      // Steering 입력 실패, 실행 계속
      console.debug('Steering input check failed:', error);
    }
    return null;
  }

  // 메시지 압축 처리
  private async compressMessages(messages: Message[]): Promise<{
    messages: Message[];
    wasCompacted: boolean;
  }> {
    // 역분석 기반 압축 임계값 확인
    const totalLength = this.estimateTokenCount(messages);
    const COMPRESSION_THRESHOLD = 40000; // k11 상수 값

    if (totalLength < COMPRESSION_THRESHOLD) {
      return { messages, wasCompacted: false };
    }

    try {
      // 스마트 압축 실행
      const summary = await this.generateSummary(messages);
      const recentMessages = messages.slice(-10);
      const systemMessages = messages.filter(m => m.role === 'system');

      return {
        messages: [summary, ...systemMessages, ...recentMessages],
        wasCompacted: true
      };
    } catch (error) {
      console.warn('Message compression failed:', error);
      return { messages, wasCompacted: false };
    }
  }

  // 메시지 요약 생성
  private async generateSummary(messages: Message[]): Promise<Message> {
    // 역분석 기반 AU2 함수 8단계 요약 템플릿
    const summaryPrompt = this.createSummaryPrompt();

    // 여기서 LLM API를 호출하여 요약 생성
    // 간소화 구현: 요약 메시지 반환
    return {
      id: Date.now().toString(),
      role: 'user',
      content: `[압축 요약] 이것은 이전 ${messages.length}개 메시지의 스마트 요약입니다...`,
      timestamp: Date.now()
    };
  }

  // 요약 프롬프트 생성
  private createSummaryPrompt(): string {
    return `다음 대화에 대한 간결한 요약을 생성하고 핵심 정보를 보존하세요:
1. 주요 사용자 요청과 의도
2. 중요한 기술 개념과 코드
3. 파일 작업과 시스템 상태
4. 에러와 해결 방법
5. 미완료 작업
6. 현재 작업 컨텍스트`;
  }

  // AI 처리 시뮬레이션
  private async* processWithAI(
    messages: Message[],
    model: string,
    abortSignal: AbortSignal
  ): AsyncGenerator<AgentResult> {
    // 여기서 실제 LLM API 호출을 통합해야 함
    // 간소화 구현: 스트리밍 응답 시뮬레이션
    yield {
      success: true,
      message: 'ai_thinking',
      data: { model, messageCount: messages.length }
    };

    await new Promise(resolve => setTimeout(resolve, 1000));

    if (abortSignal.aborted) return;

    yield {
      success: true,
      message: 'ai_response',
      data: {
        type: 'assistant',
        id: Date.now().toString(),
        role: 'assistant',
        content: '이것은 AI의 시뮬레이션 응답입니다',
        timestamp: Date.now()
      }
    };
  }

  // Tool 호출 추출
  private extractToolCalls(messages: Message[]): ToolCall[] {
    const toolCalls: ToolCall[] = [];

    for (const message of messages) {
      if (message.metadata?.toolCalls) {
        toolCalls.push(...message.metadata.toolCalls);
      }
    }

    return toolCalls;
  }

  // Tool 호출 실행
  private async* executeTools(
    toolCalls: ToolCall[],
    messages: Message[]
  ): AsyncGenerator<AgentResult> {
    for (const toolCall of toolCalls) {
      yield {
        success: true,
        message: 'tool_executing',
        data: {
          toolName: toolCall.name,
          toolCallId: toolCall.id
        }
      };

      // 여기서 후속 단계에서 구체적인 Tool 실행 로직 구현
      yield {
        success: true,
        message: 'tool_completed',
        data: {
          toolCallId: toolCall.id,
          result: 'Tool execution completed'
        }
      };
    }
  }

  // 토큰 수 추정
  private estimateTokenCount(messages: Message[]): number {
    return messages.reduce((count, msg) =>
      count + (typeof msg.content === 'string' ? msg.content.length / 4 : 0), 0
    );
  }

  // 모델 에러 여부 확인
  private isModelError(error: any): boolean {
    return error.name === 'ModelError' ||
           error.message?.includes('model') ||
           error.message?.includes('API');
  }

  // 대화 계속 여부 확인
  private shouldContinue(toolCalls: ToolCall[]): boolean {
    // Tool 호출 결과를 기반으로 계속 여부 결정
    return toolCalls.length > 0;
  }

  // 리소스 정리
  private cleanup(): void {
    this.steeringListener.stopListening();
    this.abortController.abort();
  }

  // 실행 중단
  abort(): void {
    this.abortController.abort();
  }

  // 메시지 히스토리 가져오기
  getMessageHistory(): Message[] {
    return [...this.messageHistory];
  }
}
```

### 단계 2.3: 동시성 제어 시스템 생성
**gW5 복잡한 동시성 관리 메커니즘 기반**

**파일 경로**: `src/core/concurrency-manager.ts`
**파일 내용**:
```typescript
/**
 * gW5 복잡한 동시성 관리 시스템
 * 역분석 기반 스마트 동시성 제어 메커니즘
 * 동적 로드 밸런싱과 리소스 할당 지원
 */

export interface ConcurrencyConfig {
  maxConcurrent: number;      // 최대 동시 실행 수
  enableLoadBalancing: boolean; // 동적 로드 밸런싱
  resourceLimits: ResourceLimits;
  priorityQueues: boolean;    // 우선순위 큐
}

export interface ResourceLimits {
  maxMemoryMB: number;
  maxCpuUsage: number;
  maxNetworkConnections: number;
  maxFileHandles: number;
}

export interface TaskMetrics {
  id: string;
  priority: number;
  estimatedDuration: number;
  resourceRequirements: Partial<ResourceLimits>;
  dependencies: string[];
  retryCount: number;
}

/**
 * gW5 동시성 관리자 구현
 * 역분석 기반 복잡한 스케줄링 알고리즘
 */
export class gW5ConcurrencyManager {
  private readonly config: ConcurrencyConfig;
  private activeTasks = new Map<string, TaskExecution>();
  private taskQueue: PriorityQueue<TaskMetrics>;
  private resourceMonitor: ResourceMonitor;
  private loadBalancer: LoadBalancer;

  // 역분석 기반: gW5 = 10 (기본 최대 동시 실행 수)
  private static readonly DEFAULT_MAX_CONCURRENT = 10;

  constructor(config: Partial<ConcurrencyConfig> = {}) {
    this.config = {
      maxConcurrent: config.maxConcurrent || gW5ConcurrencyManager.DEFAULT_MAX_CONCURRENT,
      enableLoadBalancing: config.enableLoadBalancing ?? true,
      resourceLimits: {
        maxMemoryMB: 512,
        maxCpuUsage: 80,
        maxNetworkConnections: 100,
        maxFileHandles: 1000,
        ...config.resourceLimits
      },
      priorityQueues: config.priorityQueues ?? true
    };

    this.taskQueue = new PriorityQueue();
    this.resourceMonitor = new ResourceMonitor(this.config.resourceLimits);
    this.loadBalancer = new LoadBalancer();
  }

  /**
   * 최적 동시성 설정 계산
   * 역분석 기반 스마트 스케줄링 알고리즘
   */
  calculateOptimalConcurrency(tasks: TaskMetrics[]): OptimalConcurrencyResult {
    // 1. 작업 의존성 관계 분석
    const dependencyGraph = this.buildDependencyGraph(tasks);

    // 2. 시스템 리소스 상태 평가
    const resourceStatus = this.resourceMonitor.getCurrentStatus();

    // 3. 동시 실행 수 동적 조정
    const optimalConcurrency = this.calculateDynamicConcurrency(
      tasks,
      resourceStatus,
      dependencyGraph
    );

    // 4. 실행 계획 생성
    const executionPlan = this.generateExecutionPlan(
      tasks,
      optimalConcurrency,
      dependencyGraph
    );

    return {
      maxConcurrent: optimalConcurrency,
      executionGroups: executionPlan.groups,
      estimatedDuration: executionPlan.estimatedDuration,
      resourceAllocation: executionPlan.resourceAllocation,
      loadBalancingStrategy: this.loadBalancer.getStrategy()
    };
  }

  /**
   * 동시 작업 실행
   * 동적 로드 밸런싱과 우선순위 스케줄링 지원
   */
  async executeConcurrentTasks<T>(
    tasks: Array<() => Promise<T>>,
    options: ExecutionOptions = {}
  ): Promise<T[]> {
    const taskMetrics = tasks.map((task, index) => ({
      id: `task_${index}`,
      priority: options.priorities?.[index] || 0,
      estimatedDuration: options.estimatedDurations?.[index] || 1000,
      resourceRequirements: options.resourceRequirements?.[index] || {},
      dependencies: options.dependencies?.[index] || [],
      retryCount: 0
    }));

    const concurrencyConfig = this.calculateOptimalConcurrency(taskMetrics);

    // 세마포어로 동시성 제어 생성
    const semaphore = new Semaphore(concurrencyConfig.maxConcurrent);
    const results: T[] = new Array(tasks.length);
    const errors: Error[] = [];

    // 작업 실행
    const taskPromises = tasks.map(async (task, index) => {
      const metrics = taskMetrics[index];

      // 실행 허가 획득
      await semaphore.acquire();

      try {
        // 작업 시작 기록
        this.startTaskExecution(metrics);

        // 작업 실행
        const result = await this.executeWithTimeout(
          task,
          options.timeout || 30000,
          metrics
        );

        results[index] = result;

        // 작업 완료 기록
        this.completeTaskExecution(metrics.id, true);

      } catch (error) {
        errors.push(error);
        this.completeTaskExecution(metrics.id, false, error);

        // 재시도 로직
        if (options.retryOnFailure && metrics.retryCount < 3) {
          metrics.retryCount++;
          // 재실행 대기열에 추가
          setTimeout(() => this.retryTask(task, index, metrics), 1000 * Math.pow(2, metrics.retryCount));
        }
      } finally {
        semaphore.release();
      }
    });

    await Promise.allSettled(taskPromises);

    if (errors.length > 0 && !options.continueOnError) {
      throw new AggregateError(errors, 'Some tasks failed');
    }

    return results;
  }

  // 의존성 그래프 구축
  private buildDependencyGraph(tasks: TaskMetrics[]): DependencyGraph {
    const graph = new Map<string, string[]>();

    for (const task of tasks) {
      graph.set(task.id, task.dependencies);
    }

    return new DependencyGraph(graph);
  }

  // 동적 동시 실행 수 계산
  private calculateDynamicConcurrency(
    tasks: TaskMetrics[],
    resourceStatus: ResourceStatus,
    dependencyGraph: DependencyGraph
  ): number {
    let baseConcurrency = this.config.maxConcurrent;

    // 리소스 사용량에 따라 조정
    if (resourceStatus.memoryUsage > 80) baseConcurrency = Math.max(2, baseConcurrency / 2);
    if (resourceStatus.cpuUsage > 80) baseConcurrency = Math.max(2, baseConcurrency / 2);

    // 작업 복잡도에 따라 조정
    const avgComplexity = tasks.reduce((sum, task) => sum + task.estimatedDuration, 0) / tasks.length;
    if (avgComplexity > 10000) baseConcurrency = Math.max(2, baseConcurrency / 2);

    // 의존성 관계에 따라 조정
    const maxParallelizable = dependencyGraph.getMaxParallelizable();
    baseConcurrency = Math.min(baseConcurrency, maxParallelizable);

    return Math.max(1, Math.floor(baseConcurrency));
  }

  // 실행 계획 생성
  private generateExecutionPlan(
    tasks: TaskMetrics[],
    maxConcurrent: number,
    dependencyGraph: DependencyGraph
  ): ExecutionPlan {
    const groups: TaskGroup[] = [];
    const sortedTasks = dependencyGraph.topologicalSort(tasks);

    let currentGroup: TaskMetrics[] = [];
    let groupResourceUsage = { memory: 0, cpu: 0, network: 0, files: 0 };

    for (const task of sortedTasks) {
      // 현재 그룹에 추가 가능 여부 확인
      if (this.canAddToGroup(task, currentGroup, groupResourceUsage, maxConcurrent)) {
        currentGroup.push(task);
        this.updateGroupResources(groupResourceUsage, task.resourceRequirements);
      } else {
        // 현재 그룹 완료, 새 그룹 시작
        if (currentGroup.length > 0) {
          groups.push({
            tasks: [...currentGroup],
            estimatedDuration: Math.max(...currentGroup.map(t => t.estimatedDuration)),
            resourceUsage: { ...groupResourceUsage }
          });
        }

        currentGroup = [task];
        groupResourceUsage = { memory: 0, cpu: 0, network: 0, files: 0 };
        this.updateGroupResources(groupResourceUsage, task.resourceRequirements);
      }
    }

    // 마지막 그룹 추가
    if (currentGroup.length > 0) {
      groups.push({
        tasks: currentGroup,
        estimatedDuration: Math.max(...currentGroup.map(t => t.estimatedDuration)),
        resourceUsage: groupResourceUsage
      });
    }

    const totalDuration = groups.reduce((sum, group) => sum + group.estimatedDuration, 0);
    const totalResources = this.calculateTotalResources(groups);

    return {
      groups,
      estimatedDuration: totalDuration,
      resourceAllocation: totalResources
    };
  }

  // 작업을 그룹에 추가 가능 여부 확인
  private canAddToGroup(
    task: TaskMetrics,
    currentGroup: TaskMetrics[],
    groupResources: any,
    maxConcurrent: number
  ): boolean {
    // 동시 실행 수 제한 확인
    if (currentGroup.length >= maxConcurrent) return false;

    // 리소스 제한 확인
    const newMemory = groupResources.memory + (task.resourceRequirements.maxMemoryMB || 0);
    const newCpu = groupResources.cpu + (task.resourceRequirements.maxCpuUsage || 0);

    if (newMemory > this.config.resourceLimits.maxMemoryMB) return false;
    if (newCpu > this.config.resourceLimits.maxCpuUsage) return false;

    // 의존성 관계 확인
    const hasConflict = currentGroup.some(groupTask =>
      task.dependencies.includes(groupTask.id) ||
      groupTask.dependencies.includes(task.id)
    );

    return !hasConflict;
  }

  // 그룹 리소스 사용량 업데이트
  private updateGroupResources(groupResources: any, taskResources: Partial<ResourceLimits>): void {
    groupResources.memory += taskResources.maxMemoryMB || 0;
    groupResources.cpu += taskResources.maxCpuUsage || 0;
    groupResources.network += taskResources.maxNetworkConnections || 0;
    groupResources.files += taskResources.maxFileHandles || 0;
  }

  // 전체 리소스 요구 계산
  private calculateTotalResources(groups: TaskGroup[]): ResourceAllocation {
    return groups.reduce((total, group) => ({
      memory: Math.max(total.memory, group.resourceUsage.memory),
      cpu: Math.max(total.cpu, group.resourceUsage.cpu),
      network: Math.max(total.network, group.resourceUsage.network),
      files: Math.max(total.files, group.resourceUsage.files)
    }), { memory: 0, cpu: 0, network: 0, files: 0 });
  }

  // 타임아웃과 함께 작업 실행
  private async executeWithTimeout<T>(
    task: () => Promise<T>,
    timeout: number,
    metrics: TaskMetrics
  ): Promise<T> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Task ${metrics.id} timed out after ${timeout}ms`));
      }, timeout);

      task()
        .then(resolve)
        .catch(reject)
        .finally(() => clearTimeout(timer));
    });
  }

  // 작업 실행 시작 기록
  private startTaskExecution(metrics: TaskMetrics): void {
    this.activeTasks.set(metrics.id, {
      metrics,
      startTime: Date.now(),
      status: 'running'
    });
  }

  // 작업 완료 기록
  private completeTaskExecution(taskId: string, success: boolean, error?: any): void {
    const execution = this.activeTasks.get(taskId);
    if (execution) {
      execution.endTime = Date.now();
      execution.status = success ? 'completed' : 'failed';
      execution.error = error;

      // 활성 작업 기록 제거
      this.activeTasks.delete(taskId);
    }
  }

  // 작업 재시도
  private async retryTask<T>(
    task: () => Promise<T>,
    index: number,
    metrics: TaskMetrics
  ): Promise<void> {
    // 재시도 로직 구현
    console.log(`Retrying task ${metrics.id}, attempt ${metrics.retryCount}`);
  }
}

// 보조 클래스 구현
class Semaphore {
  private permits: number;
  private waitQueue: Array<() => void> = [];

  constructor(permits: number) {
    this.permits = permits;
  }

  async acquire(): Promise<void> {
    if (this.permits > 0) {
      this.permits--;
      return Promise.resolve();
    }

    return new Promise(resolve => {
      this.waitQueue.push(resolve);
    });
  }

  release(): void {
    this.permits++;
    const next = this.waitQueue.shift();
    if (next) {
      this.permits--;
      next();
    }
  }
}

class PriorityQueue<T> {
  private items: Array<{ item: T; priority: number }> = [];

  enqueue(item: T, priority: number): void {
    this.items.push({ item, priority });
    this.items.sort((a, b) => b.priority - a.priority);
  }

  dequeue(): T | undefined {
    return this.items.shift()?.item;
  }

  get length(): number {
    return this.items.length;
  }
}

class ResourceMonitor {
  constructor(private limits: ResourceLimits) {}

  getCurrentStatus(): ResourceStatus {
    // 간소화 구현: 시뮬레이션된 리소스 상태 반환
    return {
      memoryUsage: process.memoryUsage().heapUsed / 1024 / 1024, // MB
      cpuUsage: Math.random() * 100, // CPU 사용률 시뮬레이션
      networkConnections: 10,
      fileHandles: 50
    };
  }
}

class LoadBalancer {
  getStrategy(): string {
    return 'round_robin';
  }
}

class DependencyGraph {
  constructor(private graph: Map<string, string[]>) {}

  topologicalSort(tasks: TaskMetrics[]): TaskMetrics[] {
    // 간소화된 위상 정렬 구현
    const visited = new Set<string>();
    const result: TaskMetrics[] = [];

    const visit = (task: TaskMetrics) => {
      if (visited.has(task.id)) return;
      visited.add(task.id);

      // 먼저 의존성 방문
      for (const depId of task.dependencies) {
        const depTask = tasks.find(t => t.id === depId);
        if (depTask) visit(depTask);
      }

      result.push(task);
    };

    for (const task of tasks) {
      visit(task);
    }

    return result;
  }

  getMaxParallelizable(): number {
    // 최대 병렬 가능 작업 수 계산
    return Math.max(1, Math.floor(this.graph.size / 2));
  }
}

// 타입 정의
interface TaskExecution {
  metrics: TaskMetrics;
  startTime: number;
  endTime?: number;
  status: 'running' | 'completed' | 'failed';
  error?: any;
}

interface OptimalConcurrencyResult {
  maxConcurrent: number;
  executionGroups: TaskGroup[];
  estimatedDuration: number;
  resourceAllocation: ResourceAllocation;
  loadBalancingStrategy: string;
}

interface ExecutionOptions {
  priorities?: number[];
  estimatedDurations?: number[];
  resourceRequirements?: Array<Partial<ResourceLimits>>;
  dependencies?: string[][];
  timeout?: number;
  retryOnFailure?: boolean;
  continueOnError?: boolean;
}

interface TaskGroup {
  tasks: TaskMetrics[];
  estimatedDuration: number;
  resourceUsage: any;
}

interface ExecutionPlan {
  groups: TaskGroup[];
  estimatedDuration: number;
  resourceAllocation: ResourceAllocation;
}

interface ResourceStatus {
  memoryUsage: number;
  cpuUsage: number;
  networkConnections: number;
  fileHandles: number;
}

interface ResourceAllocation {
  memory: number;
  cpu: number;
  network: number;
  files: number;
}
```

(파일 계속...)
