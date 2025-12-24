# 실시간 Steering 메커니즘 함수 호출 체인 완전 분석

## 개요

Claude Code의 난독화된 소스코드를 심층 분석하여, 실시간 Steering 메커니즘의 완전한 함수 호출 흐름을 추적했습니다. 사용자 입력부터 시스템 응답까지의 전체 프로세스를 상세히 기록합니다.

## 1. 호출 체인 전체 개요

```
사용자 입력(stdin)
    ↓
입력 감지 레이어
    ↓
메시지 파싱 레이어
    ↓
큐 관리 레이어
    ↓
스트리밍 처리 레이어
    ↓
Agent 실행 루프
    ↓
AI 처리 레이어
    ↓
Tool 실행 레이어
    ↓
결과 출력
```

## 2. 상세 호출 체인 분석

### 2.1 입력 레이어 - 표준 입력 감지

#### 호출 경로 1: 전역 stdin 리스너 초기화

```
파일 위치: improved-claude-code-5.mjs:49065

┌──────────────────────────────────────────────────────────┐
│ JX5 = L0(() => process.stdin.on("data", Fc))             │
│ - 전역 초기화 함수로 stdin 데이터 이벤트 등록            │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ function pq2(A, B = dq2) {                               │
│   Y1A.useEffect(() => {                                  │
│     JX5();    // stdin 리스너 초기화                      │
│     Fc();     // 포커스 체크 트리거                        │
│   }, []);                                                │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: React의 `useEffect` 훅을 사용해 컴포넌트 마운트 시 stdin 리스너를 자동으로 등록합니다.

#### 호출 경로 2: 포커스 감지를 위한 stdin 리스너

```
파일 위치: improved-claude-code-5.mjs:53568-53570

┌──────────────────────────────────────────────────────────┐
│ if (yy.add(G), yy.size === 1) {                          │
│   process.stdout.write("\x1B[?1004h");  // 포커스 보고 활성화 │
│   process.stdin.on("data", eAA);        // 데이터 리스너 등록 │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ return () => {                                           │
│   if (yy.delete(G), yy.size === 0) {                    │
│     process.stdin.off("data", eAA);     // 리스너 정리    │
│     process.stdout.write("\x1B[?1004l"); // 포커스 보고 비활성화 │
│   }                                                      │
│ };                                                       │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 터미널 포커스 이벤트를 감지하기 위해 ANSI escape 코드를 사용합니다.

#### 호출 경로 3: 키보드 입력 리스너

```
파일 위치: improved-claude-code-5.mjs:67405-67407

┌──────────────────────────────────────────────────────────┐
│ useEffect(() => {                                        │
│   return process.stdin.on("data", Q), () => {           │
│     process.stdin.off("data", Q);                       │
│   };                                                     │
│ }, [A]);                                                 │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 키보드 입력을 실시간으로 감지하고, 컴포넌트 언마운트 시 자동으로 정리합니다.

### 2.2 파싱 레이어 - 메시지 파서 (g2A)

#### 메시지 파서 클래스 초기화

```
파일 위치: improved-claude-code-5.mjs:68893-68899

┌──────────────────────────────────────────────────────────┐
│ class g2A {                                              │
│   input;                    // 원본 입력 스트림            │
│   structuredInput;          // 구조화된 입력 스트림        │
│                                                          │
│   constructor(A) {                                       │
│     this.input = A;                                      │
│     this.structuredInput = this.read();  // 비동기 읽기 시작 │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

#### 비동기 스트림 읽기 메서드

```
파일 위치: improved-claude-code-5.mjs:68900-68917

┌──────────────────────────────────────────────────────────┐
│ async *read() {                                          │
│   let buffer = "";                                       │
│                                                          │
│   for await (let chunk of this.input) {  // 스트림 순회   │
│     buffer += chunk;                                     │
│                                                          │
│     let lineEndIndex;                                    │
│     while ((lineEndIndex = buffer.indexOf('\n')) !== -1) { │
│       let line = buffer.slice(0, lineEndIndex);         │
│       buffer = buffer.slice(lineEndIndex + 1);          │
│                                                          │
│       let parsed = this.processLine(line);  // 라인 처리  │
│       if (parsed) yield parsed;             // 결과 반환  │
│     }                                                    │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: Async Generator 패턴을 사용해 스트리밍 방식으로 메시지를 처리합니다.

#### 라인 처리 및 검증

```
파일 위치: improved-claude-code-5.mjs:68918-68927

┌──────────────────────────────────────────────────────────┐
│ processLine(line) {                                      │
│   try {                                                  │
│     let message = JSON.parse(line);      // JSON 파싱    │
│                                                          │
│     // 메시지 타입 검증                                   │
│     if (message.type !== "user") {                       │
│       throw new Error(`Expected 'user', got '${message.type}'`); │
│     }                                                    │
│                                                          │
│     // 메시지 role 검증                                   │
│     if (message.message.role !== "user") {               │
│       throw new Error(`Expected role 'user', got '${message.message.role}'`); │
│     }                                                    │
│                                                          │
│     return message;                      // 검증된 메시지 반환 │
│   } catch (error) {                                      │
│     console.error(`Error parsing: ${line}: ${error}`);  │
│     process.exit(1);                                     │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 엄격한 타입 검증을 통해 잘못된 메시지 형식을 조기에 차단합니다.

### 2.3 큐 레이어 - 비동기 메시지 큐 (h2A)

#### 큐 초기화 및 핵심 메서드

```
파일 위치: improved-claude-code-5.mjs:68934-68993

┌──────────────────────────────────────────────────────────┐
│ class h2A {                                              │
│   queue = [];              // 메시지 큐                   │
│   readResolve;             // 대기 중인 읽기 Promise resolve │
│   readReject;              // 대기 중인 읽기 Promise reject  │
│   isDone = false;          // 완료 상태                   │
│   hasError;                // 에러 상태                    │
│   started = false;         // 시작 상태                    │
│                                                          │
│   constructor(cleanup) {                                 │
│     this.returned = cleanup;                             │
│   }                                                      │
│                                                          │
│   [Symbol.asyncIterator]() {                            │
│     if (this.started) {                                  │
│       throw new Error("Stream can only be iterated once"); │
│     }                                                    │
│     this.started = true;                                 │
│     return this;                                         │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

#### 비동기 읽기 메서드 - 핵심 로직

```
파일 위치: improved-claude-code-5.mjs:68945-68962

┌──────────────────────────────────────────────────────────┐
│ next() {                                                 │
│   // 우선순위 1: 큐에 메시지가 있으면 즉시 반환            │
│   if (this.queue.length > 0) {                           │
│     return Promise.resolve({                             │
│       done: false,                                       │
│       value: this.queue.shift()  // 큐에서 메시지 추출    │
│     });                                                  │
│   }                                                      │
│                                                          │
│   // 우선순위 2: 큐가 완료되었으면 완료 신호 반환          │
│   if (this.isDone) {                                     │
│     return Promise.resolve({                             │
│       done: true,                                        │
│       value: undefined                                   │
│     });                                                  │
│   }                                                      │
│                                                          │
│   // 우선순위 3: 에러 상태면 Promise reject              │
│   if (this.hasError) {                                   │
│     return Promise.reject(this.hasError);                │
│   }                                                      │
│                                                          │
│   // 우선순위 4: 새 메시지를 기다림 (비차단 핵심!)        │
│   return new Promise((resolve, reject) => {              │
│     this.readResolve = resolve;                          │
│     this.readReject = reject;                            │
│   });                                                    │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: Promise를 사용한 비차단 대기 메커니즘이 실시간 처리의 핵심입니다.

#### 메시지 인큐 메서드

```
파일 위치: improved-claude-code-5.mjs:68963-68971

┌──────────────────────────────────────────────────────────┐
│ enqueue(message) {                                       │
│   if (this.readResolve) {                                │
│     // 대기 중인 읽기가 있으면 즉시 메시지 전달            │
│     let resolver = this.readResolve;                     │
│     this.readResolve = undefined;                        │
│     this.readReject = undefined;                         │
│     resolver({                                           │
│       done: false,                                       │
│       value: message                                     │
│     });                                                  │
│   } else {                                               │
│     // 대기 중인 읽기가 없으면 큐에 버퍼링                 │
│     this.queue.push(message);                            │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 대기 중인 읽기에 즉시 전달하거나 버퍼링하는 이중 전략으로 실시간성을 보장합니다.

### 2.4 처리 레이어 - 스트리밍 처리 엔진 (kq5)

#### 스트리밍 프로세서 초기화

```
파일 위치: improved-claude-code-5.mjs:69363-69421

┌──────────────────────────────────────────────────────────┐
│ function kq5(A, B, Q, I, G, Z, D, Y) {                   │
│   let commandQueue = [];           // 명령 큐             │
│   let getCommands = () => commandQueue;                  │
│   let removeCommands = (cmds) => {  // 큐 정리 함수       │
│     commandQueue = commandQueue.filter(c => !cmds.includes(c)); │
│   };                                                     │
│   let isExecuting = false;         // 실행 중 플래그      │
│   let isCompleted = false;         // 완료 플래그          │
│   let outputQueue = new h2A();     // 출력 큐 생성        │
│   let history = processInitialMessages(Z);               │
└──────────────────────────────────────────────────────────┘
```

#### 비동기 실행 엔진

```
파일 위치: improved-claude-code-5.mjs:69373-69402

┌──────────────────────────────────────────────────────────┐
│   executeCommands = async () => {                        │
│     isExecuting = true;            // 실행 상태 설정      │
│                                                          │
│     try {                                                │
│       // 큐의 모든 명령 처리                              │
│       while (commandQueue.length > 0) {                  │
│         let command = commandQueue.shift();              │
│                                                          │
│         if (command.mode !== "prompt") {                 │
│           throw new Error("Only prompt commands supported"); │
│         }                                                │
│                                                          │
│         let prompt = command.value;                      │
│                                                          │
│         // 메인 Agent 루프 호출                           │
│         for await (let result of Zk2({                   │
│           commands: I,                                   │
│           prompt: prompt,                                │
│           cwd: process.cwd(),                            │
│           tools: G,                                      │
│           permissionContext: B,                          │
│           verbose: D,                                    │
│           mcpClients: Q,                                 │
│           // ... 기타 설정                                │
│         })) {                                            │
│           history.push(result);    // 히스토리에 추가     │
│           outputQueue.enqueue(result); // 출력 큐에 추가  │
│         }                                                │
│       }                                                  │
│     } finally {                                          │
│       isExecuting = false;         // 실행 상태 리셋      │
│     }                                                    │
│                                                          │
│     if (isCompleted) {                                   │
│       outputQueue.complete();      // 큐 완료 처리        │
│     }                                                    │
│   };                                                     │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: `finally` 블록으로 예외 발생 시에도 상태를 안전하게 리셋합니다.

#### 입력 처리 코루틴

```
파일 위치: improved-claude-code-5.mjs:69403-69421

┌──────────────────────────────────────────────────────────┐
│   return (async () => {                                  │
│     for await (let message of A) {  // 입력 스트림 순회   │
│       let prompt = extractPromptContent(message);        │
│                                                          │
│       // 명령 큐에 추가                                   │
│       commandQueue.push({                                │
│         mode: "prompt",                                  │
│         value: prompt                                    │
│       });                                                │
│                                                          │
│       // 실행 중이 아니면 실행 시작                        │
│       if (!isExecuting) {                                │
│         executeCommands().catch(error => {               │
│           outputQueue.error(error);                      │
│         });                                              │
│       }                                                  │
│     }                                                    │
│                                                          │
│     isCompleted = true;                                  │
│     if (!isExecuting) {                                  │
│       outputQueue.complete();                            │
│     }                                                    │
│   })(), outputQueue;                                     │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 입력 처리와 명령 실행이 별도 코루틴으로 분리되어 동시성을 확보합니다.

### 2.5 실행 레이어 - Agent 메인 루프 (nO & Zk2)

#### Agent 컨텍스트 생성

```
파일 위치: improved-claude-code-5.mjs:69025-69105

┌──────────────────────────────────────────────────────────┐
│ async function* Zk2({ commands, permissionContext, ... }) { │
│   // Agent 컨텍스트 구축                                  │
│   let context = {                                        │
│     messages: initialMessages,                           │
│     options: {                                           │
│       commands: commands,                                │
│       tools: tools,                                      │
│       verbose: verbose,                                  │
│       mainLoopModel: model,                              │
│       maxThinkingTokens: calculateMaxTokens(messages),   │
│       mcpClients: mcpClients,                            │
│       isNonInteractiveSession: true,                     │
│       // ... 기타 옵션                                    │
│     },                                                   │
│     abortController: new AbortController(), // 중단 컨트롤러 생성 │
│     getToolPermissionContext: () => permissionContext,   │
│     // ... 기타 메서드                                    │
│   };                                                     │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 각 Agent 실행마다 독립적인 `AbortController`를 생성하여 중단을 제어합니다.

#### 메인 루프 호출

```
파일 위치: improved-claude-code-5.mjs:69090-69095

┌──────────────────────────────────────────────────────────┐
│   // 메인 루프 호출 및 스트리밍 출력                       │
│   for await (let event of nO(                            │
│     messages, permissionContext, prompt, tools,          │
│     executionContext, context, null, fallbackModel, options │
│   )) {                                                   │
│     yield event;                   // 스트리밍 출력        │
│   }                                                      │
└──────────────────────────────────────────────────────────┘
```

#### 메인 Agent 루프 구현

```
파일 위치: improved-claude-code-5.mjs:46187-46300

┌──────────────────────────────────────────────────────────┐
│ async function* nO(A, B, Q, I, G, Z, D, Y, W) {          │
│   yield { type: "stream_request_start" }; // 스트림 시작 신호 │
│                                                          │
│   let currentMessages = A;                               │
│   let currentTurnData = D;                               │
│                                                          │
│   // 메시지 압축 처리                                     │
│   let { messages: compacted, wasCompacted } =            │
│       await wU2(A, Z);                                   │
│                                                          │
│   let responses = [];                                    │
│   let currentModel = Z.options.mainLoopModel;            │
│   let shouldRetry = true;                                │
└──────────────────────────────────────────────────────────┘
```

#### 메인 실행 루프

```
파일 위치: improved-claude-code-5.mjs:46211-46260

┌──────────────────────────────────────────────────────────┐
│   try {                                                  │
│     while (shouldRetry) {          // 재시도 루프         │
│       shouldRetry = false;                               │
│                                                          │
│       try {                                              │
│         // AI 처리 루프 호출 - 여기가 핵심 yield 지점      │
│         for await (let event of wu(                      │
│           buildMessages(currentMessages, Q),             │
│           buildContext(B, I),                            │
│           Z.options.maxThinkingTokens,                   │
│           Z.options.tools,                               │
│           Z.abortController.signal,  // 중단 신호 전달    │
│           {                                              │
│             getToolPermissionContext: Z.getToolPermissionContext, │
│             model: currentModel,                         │
│             fallbackModel: Y,                            │
│             // ... 기타 옵션                              │
│           }                                              │
│         )) {                                             │
│           yield event;             // 이벤트 스트리밍 출력 │
│           if (event.type === "assistant") {              │
│             responses.push(event);                       │
│           }                                              │
│         }                                                │
│       } catch (error) {            // 모델 강등 로직      │
│         if (error instanceof ModelFailureError && Y) {   │
│           currentModel = Y;        // 대체 모델로 전환    │
│           shouldRetry = true;                            │
│           responses.length = 0;    // 응답 리셋           │
│           continue;                                      │
│         }                                                │
│         throw error;                                     │
│       }                                                  │
│     }                                                    │
│   } catch (error) {                                      │
│     // 에러 처리 로직                                     │
│   }                                                      │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: `while` + `for await` 중첩 구조로 모델 강등 재시도와 스트리밍을 동시에 처리합니다.

#### Tool 실행 처리

```
파일 위치: improved-claude-code-5.mjs:46261-46280

┌──────────────────────────────────────────────────────────┐
│   // Tool 사용 추출                                       │
│   let toolUses = responses.flatMap(response =>           │
│     response.message.content.filter(                     │
│       item => item.type === "tool_use"                   │
│     )                                                    │
│   );                                                     │
│                                                          │
│   // Tool 실행                                            │
│   for await (let result of hW5(                          │
│     toolUses, responses, G, Z                            │
│   )) {                                                   │
│     yield result;                  // Tool 결과 출력      │
│                                                          │
│     if (result?.type === "system" &&                     │
│         result.preventContinuation) {                    │
│       shouldStop = true;           // 계속 진행 중단 플래그 │
│     }                                                    │
│   }                                                      │
│                                                          │
│   // 중단 신호 체크                                       │
│   if (Z.abortController.signal.aborted) {                │
│     yield createAbortMessage();                          │
│     return;                                              │
│   }                                                      │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: Tool 실행 후에도 중단 신호를 체크하여 즉시 중단할 수 있습니다.

### 2.6 AI 처리 레이어 - 핵심 AI 호출 (wu 함수)

#### AI 처리 호출 체인 (추론)

```
추정 위치: improved-claude-code-5.mjs의 wu 함수

┌──────────────────────────────────────────────────────────┐
│ async function* wu(                                      │
│   messages,              // AI에 전달할 메시지 목록        │
│   context,               // 실행 컨텍스트                 │
│   maxTokens,             // 최대 토큰 수                  │
│   tools,                 // 사용 가능한 도구 목록          │
│   abortSignal,           // 중단 신호                     │
│   options                // 추가 옵션                     │
│ ) {                                                      │
│   // 중단 신호 체크                                       │
│   if (abortSignal.aborted) {                             │
│     throw new Error("Request aborted");                  │
│   }                                                      │
│                                                          │
│   // AI 요청 구성                                         │
│   let request = buildAIRequest(messages, context, options); │
│                                                          │
│   // AI API 호출 및 스트리밍 처리                          │
│   for await (let chunk of callAIAPI(request, abortSignal)) { │
│     // 중단 신호 체크 (각 청크마다)                        │
│     if (abortSignal.aborted) {                           │
│       throw new Error("Request aborted");                │
│     }                                                    │
│                                                          │
│     yield processAIChunk(chunk);  // AI 응답 청크 처리     │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 각 스트리밍 청크마다 중단 신호를 체크하여 즉시 중단할 수 있습니다.

### 2.7 Tool 실행 레이어 - Tool 호출 (hW5 함수)

#### Tool 실행 호출 체인 (추론)

```
추정 위치: improved-claude-code-5.mjs의 hW5 함수

┌──────────────────────────────────────────────────────────┐
│ async function* hW5(                                     │
│   toolUses,              // 실행할 Tool 목록              │
│   responses,             // Assistant 응답 목록            │
│   context,               // 실행 컨텍스트                 │
│   agentContext           // Agent 컨텍스트                │
│ ) {                                                      │
│   for (let toolUse of toolUses) {  // 각 Tool 순회       │
│     try {                                                │
│       // 중단 상태 체크                                   │
│       if (agentContext.abortController.signal.aborted) { │
│         yield createAbortMessage();                      │
│         return;                                          │
│       }                                                  │
│                                                          │
│       // 구체적인 Tool 실행                               │
│       let result = await executeTool(toolUse, context);  │
│       yield createToolResult(result);  // Tool 결과 출력  │
│                                                          │
│     } catch (error) {                                    │
│       // 에러 결과 출력                                   │
│       yield createErrorResult(error);                    │
│     }                                                    │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: 각 Tool 실행 전에 중단 신호를 체크하여 불필요한 작업을 방지합니다.

## 3. 중단 신호 전파 체인

### 3.1 AbortController 생성 및 전파

```
Agent 컨텍스트 생성 (69070행)
    ↓
abortController: new AbortController()
    ↓
AI 처리 전달 (46215행)
    ↓
Z.abortController.signal
    ↓
Tool 실행 전달
    ↓
Tool 내부 중단 체크
    ↓
중단 상태 체크 (46270행)
    ↓
if (Z.abortController.signal.aborted)
```

### 3.2 중단 체크 포인트 분포

```
┌──────────────────────────────────────────────────────────┐
│ 체크 포인트 1: AI 처리 시작 전                            │
│ 위치: wu 함수 진입점                                      │
│ 검사: abortSignal.aborted                                │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 체크 포인트 2: 각 AI 응답 청크 처리 시                     │
│ 위치: AI API 호출 루프 내부                               │
│ 검사: abortSignal.aborted                                │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 체크 포인트 3: Tool 실행 시작 전                          │
│ 위치: Tool 실행 루프 시작                                 │
│ 검사: agentContext.abortController.signal.aborted        │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 체크 포인트 4: 메인 루프 계속 진행 전                      │
│ 위치: nO 함수 Tool 실행 후 (46270행)                     │
│ 검사: Z.abortController.signal.aborted                   │
└──────────────────────────────────────────────────────────┘
```

## 4. 에러 처리 및 복구 체인

### 4.1 에러 포착 계층 구조

```
┌──────────────────────────────────────────────────────────┐
│ 레벨 1: 입력 파싱 에러                                    │
│ 위치: g2A.processLine (68918-68927행)                   │
│ 처리: console.error + process.exit(1)                   │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 레벨 2: 큐 연산 에러                                      │
│ 위치: h2A.error 메서드 (68980-68984행)                   │
│ 처리: Promise.reject를 통한 에러 전파                     │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 레벨 3: 스트리밍 처리 에러                                 │
│ 위치: kq5 실행 엔진의 try-finally (69373-69402행)        │
│ 처리: 상태 리셋 보장                                      │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 레벨 4: Agent 실행 에러                                   │
│ 위치: nO 메인 루프의 try-catch (46211-46260행)           │
│ 처리: 모델 강등 + 에러 결과 생성                          │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 레벨 5: Tool 실행 에러                                    │
│ 위치: 각 Tool의 내부 에러 처리                            │
│ 처리: 에러 결과 래핑 + 실행 계속                          │
└──────────────────────────────────────────────────────────┘
```

## 5. 핵심 시퀀스 다이어그램

### 5.1 정상 흐름 시퀀스

```
시간 흐름 →

입력         파서         큐          프로세서      Agent       AI          Tool
───────────────────────────────────────────────────────────────────────────────
stdin.data
    │
    ├──────→ processLine()
             │
             ├──────→ enqueue()
                      │
                      ├──────→ executeCommands()
                               │
                               ├──────→ nO()
                                        │
                                        ├──────→ wu()
                                                 │
                                                 ├──────→ executeTool()
                                                          │
                                                          └─→ result
                                                          │
                                                 ←────────┘
                                                 │
                                        ←────────┘
                                        │
                               ←────────┘
                               │
                      ←────────┘
                      │
             ←────────┘
             │
    ←────────┘
```

### 5.2 중단 흐름 시퀀스

```
시간 흐름 →

사용자       AbortController  Agent       AI          Tool        출력
────────────────────────────────────────────────────────────────────
abort()
    │
    ├──────→ signal.abort
             │
             ├──────→ signal.aborted check
                      │
                      ├──────→ abort check
                               │
                               ├──────→ abort check
                                        │
                                        └──────→ abort message
                                                 │
                                                 └─→ user
```

## 6. 성능 핵심 경로

### 6.1 핫 패스 분석

1. **메시지 인큐 핫 패스** (최빈도 호출):
   ```
   stdin.data → g2A.processLine → h2A.enqueue → (즉시 콜백 또는 큐 버퍼링)
   ```

2. **메시지 디큐 핫 패스** (최빈도 호출):
   ```
   h2A.next → (큐 추출 또는 Promise 대기) → kq5 실행 엔진
   ```

3. **AI 처리 핫 패스** (연산 집약):
   ```
   nO 메인 루프 → wu(AI 처리) → 스트리밍 응답 → yield 출력
   ```

### 6.2 병목 지점 식별

1. **잠재적 병목 1**: JSON 파싱
   - 위치: g2A.processLine
   - 최적화: 스트리밍 JSON 파서 사용

2. **잠재적 병목 2**: 큐 연산
   - 위치: h2A 큐 연산
   - 최적화: 링 버퍼 구조 적용

3. **잠재적 병목 3**: AI API 호출
   - 위치: wu 함수 내부
   - 최적화: 커넥션 풀 + 캐싱

## 7. 메모리 관리 경로

### 7.1 메모리 할당 지점

```
┌──────────────────────────────────────────────────────────┐
│ 할당 지점 1: 메시지 큐 버퍼                               │
│ 위치: h2A.queue.push(message)                            │
│ 크기: 메시지 객체당                                       │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 할당 지점 2: 명령 큐                                      │
│ 위치: kq5의 commandQueue.push({mode, value})             │
│ 크기: 명령 객체당                                         │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 할당 지점 3: 메시지 히스토리                              │
│ 위치: kq5의 history.push(result)                         │
│ 크기: 누적 메시지 히스토리                                │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 할당 지점 4: Agent 응답                                   │
│ 위치: nO의 responses.push(event)                         │
│ 크기: AI 응답 객체당                                      │
└──────────────────────────────────────────────────────────┘
```

### 7.2 메모리 해제 지점

```
┌──────────────────────────────────────────────────────────┐
│ 해제 지점 1: 큐 소비                                      │
│ 위치: h2A.next()의 this.queue.shift()                    │
│ 메커니즘: 자동 가비지 컬렉션                              │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 해제 지점 2: 명령 완료                                    │
│ 위치: kq5의 commandQueue.shift()                         │
│ 메커니즘: 자동 가비지 컬렉션                              │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│ 해제 지점 3: 클린업 콜백                                  │
│ 위치: h2A.return()의 this.returned()                     │
│ 메커니즘: 수동 정리                                       │
└──────────────────────────────────────────────────────────┘
```

## 8. 결론

Claude Code의 실시간 Steering 메커니즘은 정교하게 설계된 다층 호출 체인을 통해 다음을 구현합니다:

1. **비차단 처리**: 모든 레이어가 비동기 메커니즘으로 차단 방지
2. **실시간 응답**: 직접 콜백과 Promise 메커니즘으로 저지연 구현
3. **상태 일관성**: 다층 상태 관리로 시스템 일관성 보장
4. **에러 복구**: 계층화된 에러 처리로 우아한 강등 구현
5. **리소스 관리**: 자동 및 수동 정리로 메모리 누수 방지

이 호출 체인 분석은 Claude Code의 기술 구현을 이해하고 오픈소스로 재구현하기 위한 완전한 기술 청사진을 제공합니다.
