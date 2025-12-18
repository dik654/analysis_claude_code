# Claude Code 실시간 Steering 메커니즘 완전 기술 문서

## 개요

본 문서는 Claude Code 난독화 소스코드의 심층 분석을 기반으로 실시간 Steering 메커니즘의 완전한 구현을 상세히 설명합니다. 이 메커니즘은 사용자가 Claude가 작업을 실행하는 동안 실시간으로 메시지를 보내 상호작용하고 지시할 수 있게 하며, 전통적인 Agent 시스템의 동기 차단 제한을 돌파합니다.

## 핵심 아키텍처 개요

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   stdin 리스닝   │───▶│  메시지 파서     │───▶│  비동기 메시지   │
│  (실시간 입력)   │    │   (g2A)         │    │  큐 (h2A)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ AbortController │◀───│   스트림 처리    │◀───│   Agent 루프    │
│   (중단 제어)    │    │   (kq5)         │    │    (nO)         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 1. 비동기 메시지 큐 시스템 (h2A 클래스)

### 1.1 핵심 구현

```javascript
// 위치: improved-claude-code-5.mjs:68934-68993
class h2A {
  returned;           // 정리 함수
  queue = [];         // 메시지 큐 버퍼
  readResolve;        // Promise resolve 콜백
  readReject;         // Promise reject 콜백
  isDone = false;     // 큐 완료 플래그
  hasError;           // 오류 상태
  started = false;    // 시작 상태 플래그

  constructor(A) {
    this.returned = A  // 정리 콜백 설정
  }

  // AsyncIterator 인터페이스 구현
  [Symbol.asyncIterator]() {
    if (this.started)
      throw new Error("Stream can only be iterated once");
    this.started = true;
    return this;
  }

  // 핵심 비동기 반복자 메서드
  next() {
    // 큐에서 우선적으로 메시지 가져오기
    if (this.queue.length > 0) {
      return Promise.resolve({
        done: false,
        value: this.queue.shift()
      });
    }

    // 큐가 완료되면 종료 플래그 반환
    if (this.isDone) {
      return Promise.resolve({
        done: true,
        value: undefined
      });
    }

    // 오류가 있으면 Promise를 거부
    if (this.hasError) {
      return Promise.reject(this.hasError);
    }

    // 새 메시지 대기 - 핵심 논블로킹 메커니즘
    return new Promise((resolve, reject) => {
      this.readResolve = resolve;
      this.readReject = reject;
    });
  }

  // 메시지 인큐 - 실시간 메시지 삽입 지원
  enqueue(A) {
    if (this.readResolve) {
      // 대기 중인 읽기가 있으면 메시지를 직접 반환
      let callback = this.readResolve;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback({
        done: false,
        value: A
      });
    } else {
      // 그렇지 않으면 큐 버퍼에 추가
      this.queue.push(A);
    }
  }

  // 큐 완료
  done() {
    this.isDone = true;
    if (this.readResolve) {
      let callback = this.readResolve;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback({
        done: true,
        value: undefined
      });
    }
  }

  // 오류 처리
  error(A) {
    this.hasError = A;
    if (this.readReject) {
      let callback = this.readReject;
      this.readResolve = undefined;
      this.readReject = undefined;
      callback(A);
    }
  }
}
```

### 1.2 기술 특징

- **논블로킹 설계**: Promise와 콜백 메커니즘을 사용하여 논블로킹 메시지 읽기 구현
- **이중 버퍼링**: 큐 버퍼링과 직접 전달 두 가지 모드 지원
- **상태 관리**: 완전한 생명주기 상태 관리
- **오류 복구**: 내장 오류 처리 및 전파 메커니즘

## 2. 메시지 파서 (g2A 클래스)

### 2.1 스트림 파싱 구현

```javascript
// 위치: improved-claude-code-5.mjs:68893-68928
class g2A {
  input;            // 원시 입력 스트림
  structuredInput;  // 구조화된 입력 스트림

  constructor(A) {
    this.input = A;
    this.structuredInput = this.read();
  }

  // 비동기 제너레이터 - 스트림 처리 입력
  async *read() {
    let buffer = "";

    // 입력 스트림을 문자별로 처리
    for await (let chunk of this.input) {
      buffer += chunk;
      let lineEnd;

      // 줄 단위로 분할 처리
      while ((lineEnd = buffer.indexOf('\n')) !== -1) {
        let line = buffer.slice(0, lineEnd);
        buffer = buffer.slice(lineEnd + 1);

        let parsed = this.processLine(line);
        if (parsed) yield parsed;
      }
    }

    // 마지막 줄 처리
    if (buffer) {
      let parsed = this.processLine(buffer);
      if (parsed) yield parsed;
    }
  }

  // 한 줄 메시지 파싱
  processLine(line) {
    try {
      let message = JSON.parse(line);

      // 엄격한 타입 검증
      if (message.type !== "user") {
        throw new Error(`Expected message type 'user', got '${message.type}'`);
      }

      if (message.message.role !== "user") {
        throw new Error(`Expected message role 'user', got '${message.message.role}'`);
      }

      return message;
    } catch (error) {
      console.error(`Error parsing streaming input line: ${line}: ${error}`);
      process.exit(1);
    }
  }
}
```

### 2.2 보안 특성

- **타입 검증**: 메시지 타입과 역할을 엄격하게 검증
- **오류 처리**: 파싱 실패 시 상세한 오류 정보 제공
- **스트림 처리**: 큰 메시지의 청크 처리 지원
- **상태 안전성**: 무상태 설계로 상태 오염 방지

## 3. 스트림 메시지 처리 엔진 (kq5 함수)

### 3.1 핵심 스케줄링 로직

```javascript
// 위치: improved-claude-code-5.mjs:69363-69421
function kq5(inputStream, permissionContext, mcpClients, commands, tools,
            toolConfig, permissionTool, options) {
  let commandQueue = [];          // 명령 큐
  let getQueuedCommands = () => commandQueue;  // 큐 접근자
  let removeQueuedCommands = (commands) => {   // 큐 정리기
    commandQueue = commandQueue.filter(cmd => !commands.includes(cmd));
  };

  let isExecuting = false;        // 실행 상태 플래그
  let isCompleted = false;        // 완료 상태 플래그
  let outputStream = new h2A();   // 출력 큐
  let messageHistory = processInitialMessages(toolConfig);

  // 비동기 실행 엔진
  let executeCommands = async () => {
    isExecuting = true;
    try {
      // 큐의 모든 명령 처리
      while (commandQueue.length > 0) {
        let command = commandQueue.shift();

        // prompt 명령만 지원
        if (command.mode !== "prompt") {
          throw new Error("only prompt commands are supported in streaming mode");
        }

        let prompt = command.value;

        // 주 Agent 실행 루프 호출 - 핵심 호출 지점
        for await (let result of Zk2({
          commands: commands,
          prompt: prompt,
          cwd: getCurrentWorkingDirectory(),
          tools: tools,
          permissionContext: permissionContext,
          verbose: options.verbose,
          mcpClients: mcpClients,
          maxTurns: options.maxTurns,
          permissionPromptTool: permissionTool,
          userSpecifiedModel: options.userSpecifiedModel,
          fallbackModel: options.fallbackModel,
          initialMessages: messageHistory,
          customSystemPrompt: options.systemPrompt,
          appendSystemPrompt: options.appendSystemPrompt,
          getQueuedCommands: getQueuedCommands,
          removeQueuedCommands: removeQueuedCommands
        })) {
          messageHistory.push(result);    // 메시지 히스토리 업데이트
          outputStream.enqueue(result);   // 스트림에 출력
        }
      }
    } finally {
      isExecuting = false;  // 상태 재설정 보장
    }

    if (isCompleted) outputStream.done();
  };

  // 입력 처리 코루틴
  let processInput = async () => {
    for await (let message of inputStream) {
      let promptContent;

      // 메시지 내용 추출 - 여러 형식 지원
      if (typeof message.message.content === "string") {
        promptContent = message.message.content;
      } else {
        if (message.message.content.length !== 1) {
          console.error(`Error: Expected exactly one content item, got ${message.message.content.length}`);
          process.exit(1);
        }

        if (typeof message.message.content[0] === "string") {
          promptContent = message.message.content[0];
        } else if (message.message.content[0].type === "text") {
          promptContent = message.message.content[0].text;
        } else {
          console.error("Error: Expected string or text block content");
          process.exit(1);
        }
      }

      // 새 메시지 인큐
      commandQueue.push({
        mode: "prompt",
        value: promptContent
      });

      // 실행 중이 아니면 실행 시작
      if (!isExecuting) executeCommands();
    }

    isCompleted = true;
    if (!isExecuting) outputStream.done();
  };

  // 입력 처리 시작
  processInput();

  return outputStream;  // 출력 스트림 반환
}
```

### 3.2 동시성 제어 특성

- **상태 동기화**: 플래그 비트를 사용하여 실행 상태의 정확성 보장
- **큐 관리**: 동적 명령 인큐 및 디큐 지원
- **논블로킹 실행**: 새 메시지가 현재 실행을 차단하지 않음
- **스트림 출력**: 비동기 큐를 통한 스트림 출력 구현

## 4. Agent 메인 루프 (nO 함수)

### 4.1 Async Generator 구현

```javascript
// 위치: improved-claude-code-5.mjs:46187-46300+
async function* nO(messages, user, prompt, tools, context, agentContext,
                   turnData, fallbackModel, options) {
  // 스트림 시작 마커
  yield { type: "stream_request_start" };

  let currentMessages = messages;
  let currentTurnData = turnData;

  // 메시지 압축 처리
  let { messages: compactedMessages, wasCompacted } = await compactMessages(messages, agentContext);
  if (wasCompacted) {
    logEvent("tengu_auto_compact_succeeded", {
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

  let assistantResponses = [];
  let currentModel = agentContext.options.mainLoopModel;
  let shouldRetry = true;

  try {
    // 주 실행 루프 - 모델 강등 재시도 지원
    while (shouldRetry) {
      shouldRetry = false;

      try {
        // 핵심 AI 처리 루프 호출 - 핵심 yield 지점
        for await (let response of processWithAI(
          buildMessages(currentMessages, prompt),
          buildContext(user, tools),
          agentContext.options.maxThinkingTokens,
          agentContext.options.tools,
          agentContext.abortController.signal,  // 중단 신호 전달
          {
            getToolPermissionContext: agentContext.getToolPermissionContext,
            model: currentModel,
            prependCLISysprompt: true,
            toolChoice: undefined,
            isNonInteractiveSession: agentContext.options.isNonInteractiveSession,
            fallbackModel: fallbackModel
          }
        )) {
          // 각 응답을 스트림 출력 - 논블로킹 구현
          yield response;

          if (response.type === "assistant") {
            assistantResponses.push(response);
          }
        }
      } catch (error) {
        // 모델 강등 처리
        if (error instanceof ModelError && fallbackModel) {
          currentModel = fallbackModel;
          shouldRetry = true;
          assistantResponses.length = 0;
          agentContext.options.mainLoopModel = fallbackModel;

          logEvent("tengu_model_fallback_triggered", {
            original_model: error.originalModel,
            fallback_model: fallbackModel,
            entrypoint: "cli"
          });

          yield createLogMessage(`Model fallback triggered: switching from ${error.originalModel} to ${error.fallbackModel}`, "info");
          continue;
        }
        throw error;
      }
    }
  } catch (error) {
    // 오류 처리 및 도구 결과 생성
    logError(error instanceof Error ? error : new Error(String(error)));
    let errorMessage = error instanceof Error ? error.message : String(error);

    logEvent("tengu_query_error", {
      assistantMessages: assistantResponses.length,
      toolUses: assistantResponses.flatMap(resp =>
        resp.message.content.filter(content => content.type === "tool_use")).length
    });

    // 각 도구 사용에 대한 오류 결과 생성
    let hasToolResults = false;
    for (let response of assistantResponses) {
      let toolUses = response.message.content.filter(content => content.type === "tool_use");
      for (let toolUse of toolUses) {
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

  // 후속 도구 처리 로직...
  if (!assistantResponses.length) return;

  let toolUses = assistantResponses.flatMap(response =>
    response.message.content.filter(content => content.type === "tool_use"));

  if (!toolUses.length) return;

  let toolResults = [];
  let shouldPreventContinuation = false;

  // 도구 호출 실행
  for await (let result of executeTools(toolUses, assistantResponses, context, agentContext)) {
    yield result;  // 도구 결과를 스트림 출력

    if (result && result.type === "system" && result.preventContinuation) {
      shouldPreventContinuation = true;
    }

    toolResults.push(...filterUserMessages([result]));
  }

  // 중단되었는지 확인
  if (agentContext.abortController.signal.aborted) {
    yield createSystemMessage({
      toolUse: true,
      hardcodedMessage: undefined
    });
    return;
  }

  if (shouldPreventContinuation) return;

  // 계속 처리...
}
```

### 4.2 핵심 기술 특성

- **Async Generator**: yield를 사용하여 논블로킹 스트림 처리 구현
- **중단 검사**: 여러 핵심 지점에서 AbortController 신호 검사
- **상태 유지**: 제너레이터 상태를 통해 실행 컨텍스트 유지
- **오류 복구**: 완전한 오류 처리 및 복구 메커니즘
- **모델 강등**: 모델 장애 시 자동 강등 지원

## 5. AbortController 중단 메커니즘

### 5.1 중단 컨트롤러 생성

```javascript
// 위치: improved-claude-code-5.mjs:69070
let agentContext = {
  messages: messages,
  setMessages: () => {},
  onChangeAPIKey: () => {},
  options: {
    commands: commands,
    debug: false,
    tools: tools,
    verbose: verbose,
    mainLoopModel: getModel(),
    maxThinkingTokens: calculateMaxTokens(messages),
    mcpClients: mcpClients,
    mcpResources: {},
    ideInstallationStatus: null,
    isNonInteractiveSession: true,
    theme: getTheme().theme
  },
  getToolPermissionContext: () => permissionContext,
  getQueuedCommands: () => [],
  removeQueuedCommands: () => {},
  abortController: new AbortController(),  // 각 Agent 인스턴스의 독립 중단 컨트롤러
  readFileState: {},
  setInProgressToolUseIDs: () => {},
  setToolPermissionContext: () => {},
  agentId: generateAgentId()
};
```

### 5.2 중단 신호 전파

```javascript
// 중단 신호는 전체 호출 체인에서 전파됨
for await (let response of processWithAI(
  buildMessages(currentMessages, prompt),
  buildContext(user, tools),
  agentContext.options.maxThinkingTokens,
  agentContext.options.tools,
  agentContext.abortController.signal,  // 중단 신호 전달
  processingOptions
)) {
  yield response;
}

// 중단 상태 검사
if (agentContext.abortController.signal.aborted) {
  yield createSystemMessage({
    toolUse: true,
    hardcodedMessage: undefined
  });
  return;
}
```

### 5.3 중단 처리 메커니즘

```javascript
// 위치: improved-claude-code-5.mjs:4903-4906 및 관련 코드
// 중단 이벤트 리스닝
if (abortController.once("abort", cleanupCallback),
    options.cancelToken || options.signal) {

  if (options.cancelToken && options.cancelToken.subscribe(abortHandler)) {
    // 취소 토큰 처리
  }

  if (options.signal) {
    if (options.signal.aborted) {
      abortHandler();
    } else {
      options.signal.addEventListener("abort", abortHandler);
    }
  }
}

// 복합 중단 컨트롤러 - 다중 소스 중단 지원
function createCompositeAbortController(sources, cleanup) {
  let compositeController = new AbortController();
  let isAborted = false;

  let abortHandler = function(event) {
    if (!isAborted) {
      isAborted = true;
      cleanup();
      let error = event instanceof Error ?
        event : this.reason;
      compositeController.abort(
        error instanceof AbortError ?
        error : new AbortError(error instanceof Error ? error.message : error)
      );
    }
  };

  // 모든 소스의 중단 이벤트 리스닝
  sources.forEach(source => {
    source.addEventListener("abort", abortHandler);
  });

  let { signal } = compositeController;
  signal.unsubscribe = () => scheduleMicrotask(cleanup);
  return signal;
}
```

## 6. 실시간 입력 리스닝 시스템

### 6.1 표준 입력 리스닝

```javascript
// 위치: improved-claude-code-5.mjs:49065
// 전역 stdin 리스닝 초기화
let initializeStdinListener = once(() =>
  process.stdin.on("data", handleFocusChange)
);

// 위치: improved-claude-code-5.mjs:53568-53570
// 포커스 감지의 stdin 리스닝
if (focusListeners.add(callback), focusListeners.size === 1) {
  process.stdout.write("\x1B[?1004h");  // 포커스 보고 활성화
  process.stdin.on("data", handleFocusEvents);
}

// 정리 메커니즘
return () => {
  if (focusListeners.delete(callback), focusListeners.size === 0) {
    process.stdin.off("data", handleFocusEvents);
    process.stdout.write("\x1B[?1004l");  // 포커스 보고 비활성화
  }
};

// 위치: improved-claude-code-5.mjs:67405-67407
// 키보드 입력 리스닝
useEffect(() => {
  return process.stdin.on("data", handleKeyPress), () => {
    process.stdin.off("data", handleKeyPress);
  };
}, [onDismiss]);
```

### 6.2 입력 데이터 처리

```javascript
// 포커스 시퀀스 필터링
let filterFocusSequences = useCallback((data, hasTyped) => {
  // 터미널 포커스 제어 시퀀스 필터링
  if (data === "\x1B[I" || data === "\x1B[O" ||
      data === "[I" || data === "[O") {
    return "";
  }

  // 사용자 입력 상태 처리
  if ((data || hasTyped) && !isFocused) {
    setHasTyped(true);
  }

  return data;
}, [isFocused]);
```

## 7. 메시지 스트림 처리 파이프라인

### 7.1 메시지 플로우 경로

```
사용자 입력(stdin)
    ↓
입력 리스너(process.stdin.on)
    ↓
메시지 파서(g2A.processLine)
    ↓
큐 인큐(h2A.enqueue)
    ↓
스트림 프로세서(kq5)
    ↓
Agent 루프(nO)
    ↓
AI 처리(wu/processWithAI)
    ↓
도구 실행(executeTools)
    ↓
결과 출력(yield)
```

### 7.2 동시성 안전 메커니즘

```javascript
// 큐 상태 동기화
let commandQueue = [];
let isExecuting = false;
let isCompleted = false;

// 안전한 큐 작업
let addCommand = (command) => {
  commandQueue.push(command);
  if (!isExecuting) {
    executeCommands();  // 논블로킹 실행 시작
  }
};

// 실행 상태 보호
let executeCommands = async () => {
  isExecuting = true;
  try {
    while (commandQueue.length > 0) {
      let command = commandQueue.shift();
      // 명령 실행...
    }
  } finally {
    isExecuting = false;  // 상태 재설정 보장
  }

  if (isCompleted) {
    outputStream.done();
  }
};
```

## 8. 기술 우위 및 혁신점

### 8.1 아키텍처 우위

1. **진정한 비동기 처리**:
   - 동기 루프가 아닌 async generator 사용
   - Promise 기반 논블로킹 큐 메커니즘
   - 스트림 처리로 메모리 적체 방지

2. **다층 상태 관리**:
   - 큐 레벨 상태(h2A)
   - 프로세서 레벨 상태(kq5)
   - Agent 레벨 상태(nO)
   - 도구 레벨 상태(AbortController)

3. **완전한 오류 복구**:
   - 계층화된 오류 처리
   - 우아한 중단 메커니즘
   - 모델 강등 전략
   - 상태 일관성 보장

### 8.2 성능 특성

1. **저지연 응답**:
   - 직접 콜백 메커니즘으로 폴링 방지
   - 큐 버퍼링으로 대기 감소
   - yield 지점이 적시 중단 기회 제공

2. **메모리 효율성**:
   - 스트림 처리로 대량 캐싱 방지
   - 필요에 따른 처리로 메모리 사용 감소
   - 자동 정리 메커니즘

3. **동시성 안전성**:
   - 잠금 없는 설계로 교착 상태 방지
   - 상태 플래그로 일관성 보장
   - 비동기 메커니즘으로 처리량 향상

## 9. 구현 가이드 원칙

### 9.1 오픈소스 구현 요점

1. **핵심 컴포넌트 구현**:
   ```javascript
   // 구현해야 할 핵심 클래스
   class AsyncMessageQueue {
     // h2A 기능 구현
   }

   class MessageParser {
     // g2A 기능 구현
   }

   class StreamProcessor {
     // kq5 기능 구현
   }

   async function* agentLoop() {
     // nO 기능 구현
   }
   ```

2. **핵심 디자인 패턴**:
   - AsyncIterator 패턴으로 스트림 처리
   - Promise/콜백 혼합으로 논블로킹
   - Generator로 중단 가능한 실행
   - ObserverPattern으로 상태 알림

3. **필수 인프라**:
   - AbortController 통합
   - 프로세스 신호 처리
   - 오류 경계 처리
   - 상태 동기화 메커니즘

### 9.2 테스트 검증 전략

1. **기능 검증**:
   - 메시지 큐의 인큐/디큐 정확성
   - 중단 신호의 전파 및 응답
   - 동시성 상황에서의 상태 일관성
   - 오류 상황에서의 복구 능력

2. **성능 검증**:
   - 고빈도 메시지 처리 능력
   - 메모리 사용의 안정성
   - 응답 지연 측정
   - 장기 실행의 안정성

## 10. 결론

Claude Code의 실시간 Steering 메커니즘은 Agent 시스템 아키텍처의 중요한 기술 혁신을 나타냅니다:

1. **전통적 한계 돌파**: 동기 실행의 제약을 깨고 진정한 실시간 상호작용 구현
2. **아키텍처 선진성**: 현대 비동기 프로그래밍 패턴을 사용하여 효율적인 동시성 처리 구현
3. **엔지니어링 완전성**: 완전한 오류 처리, 상태 관리 및 성능 최적화 포함
4. **실용적 가치**: 오픈소스 Agent 시스템에 실행 가능한 구현 경로 제공

이 메커니즘의 심층 분석은 Claude Code의 기술력을 보여줄 뿐만 아니라 전체 AI Agent 분야에 귀중한 기술 참조와 구현 가이드를 제공합니다.

---

**문서 버전**: 1.0
**분석 날짜**: 2025-06-27
**기반 소스코드**: improved-claude-code-5.mjs (및 관련 파일)
**분석 깊이**: 완전한 소스코드 레벨 검증
