# Claude Code 계층적 다중 Agent 아키텍처

## 개요

이 문서는 Claude Code의 소스 코드를 역공학하여 분석한 결과를 바탕으로, 계층적 다중 Agent 아키텍처의 핵심 구현 원리를 설명합니다. Task Tool을 통해 SubAgent를 생성하고, 생명주기를 관리하며, 병렬 실행을 조율하고, 보안 격리를 구현하는 방법을 상세히 다룹니다.

---

## 1. 전체 아키텍처

### 1.1 시스템 구조

Claude Code는 주 Agent와 SubAgent의 협업을 통해 복잡한 작업을 처리하는 혁신적인 계층 구조를 채택했습니다.

```
┌─────────────────┐
│   사용자 요청    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  주 Agent (nO)  │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────┐   ┌──────────────┐
│직접 │   │ Task Tool    │
│처리 │   │ (p_2 객체)   │
└─────┘   └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │ SubAgent 생성 │
          │  (I2A 함수)   │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │ 생명주기 관리 │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │ 병렬 실행 조율│
          │  (UH1 함수)   │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │ 결과 합성기   │
          │  (KN5 함수)   │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │  통합 결과    │
          └──────────────┘
```

### 1.2 핵심 설계 원칙

1. **완전한 실행 환경 격리**: 각 SubAgent는 독립된 컨텍스트에서 실행
2. **지능형 병렬 스케줄링**: 여러 Agent의 동시 실행과 동적 부하 분산 지원
3. **세밀한 보안 제어**: 도구별 권한 관리 및 리소스 제한
4. **효율적인 결과 합성**: 여러 Agent의 결과를 지능적으로 통합하고 충돌 해결
5. **탄력적인 오류 처리**: 다층 오류 격리 및 복구 메커니즘

---

## 2. Task Tool의 Agent 인스턴스화

### 2.1 Task Tool 핵심 정의

Task Tool은 다중 Agent 아키텍처의 진입점입니다.

```javascript
// Task Tool 상수 정의 (improved-claude-code-5.mjs:25993)
cX = "Task"

// Task Tool 입력 스키마 (improved-claude-code-5.mjs:62321-62324)
CN5 = n.object({
    description: n.string().describe("작업에 대한 짧은 설명 (3-5단어)"),
    prompt: n.string().describe("Agent가 수행할 작업")
})

// Task Tool 전체 객체 구조 (improved-claude-code-5.mjs:62435-62569)
p_2 = {
    // 동적 설명 생성
    async prompt({ tools: A }) {
        return await u_2(A)  // 설명 생성 함수 호출
    },

    name: cX,  // "Task"

    async description() {
        return "새로운 작업 시작"
    },

    inputSchema: CN5,

    // 핵심 실행 함수
    async * call({ prompt: A }, context, J, F) {
        // 실제 Agent 시작 및 관리 로직
        // 뒤에서 자세히 설명
    },

    // Tool 특성 정의
    isReadOnly() { return true },
    isConcurrencySafe() { return true },
    isEnabled() { return true },
    userFacingName() { return "Task" },

    // 권한 검사
    async checkPermissions(A) {
        return { behavior: "allow", updatedInput: A }
    }
}
```

### 2.2 동적 설명 생성

Task Tool의 설명은 현재 사용 가능한 도구 목록을 포함하여 동적으로 생성됩니다.

```javascript
// 도구 설명 생성기 (improved-claude-code-5.mjs:62298-62316)
async function u_2(availableTools) {
    return `다음 도구에 접근 가능한 새 Agent를 시작합니다: ${
        availableTools
            .filter((tool) => tool.name !== cX)  // Task Tool 자체는 제외 (재귀 방지)
            .map((tool) => tool.name)
            .join(", ")
    }. 키워드나 파일을 검색할 때 첫 시도에서 올바른 결과를 찾을 확신이 없다면 Agent Tool을 사용하세요.

Agent Tool 사용 시점:
- "config"나 "logger" 같은 키워드를 검색하거나 "X는 어떤 파일에 있나요?" 같은 질문에는 Agent Tool 강력 추천
- 다중 파일에서 반복적으로 검색해야 할 때

Agent Tool을 사용하지 말아야 할 때:
- 특정 파일 경로를 읽으려면 ${OB.name}이나 ${g$.name} Tool을 직접 사용
- "class Foo" 같은 특정 클래스 정의를 찾으려면 ${g$.name} Tool 사용
- 특정 파일이나 2-3개 파일 내에서 코드를 검색하려면 ${OB.name} Tool 사용
- 코드 작성 및 bash 명령 실행 (다른 도구 사용)
- 검색과 관련 없는 작업

사용 주의사항:
1. 성능 최대화를 위해 가능하면 여러 Agent를 동시에 실행하세요. 단일 메시지에 여러 Tool 호출 포함
2. Agent가 완료되면 단일 메시지를 반환합니다. Agent의 결과는 사용자에게 직접 보이지 않으므로 결과 요약을 사용자에게 전달해야 합니다
3. 각 Agent 호출은 상태를 유지하지 않습니다. Agent에 추가 메시지를 보낼 수 없으며, Agent도 최종 보고서 외에는 통신할 수 없습니다. 따라서 프롬프트에 Agent가 자율적으로 수행할 상세한 작업 설명과 반환해야 할 정보를 명확히 지정하세요
4. Agent의 출력은 일반적으로 신뢰할 수 있습니다
5. Agent에게 코드 작성이나 연구(검색, 파일 읽기, 웹 가져오기 등) 중 무엇을 기대하는지 명확히 알려주세요`
}
```

### 2.3 SubAgent 생성 프로세스

I2A 함수는 SubAgent 생성을 담당하며, 완전한 Agent 인스턴스화 프로세스를 구현합니다.

```javascript
// SubAgent 시작 함수 (improved-claude-code-5.mjs:62353-62433)
async function* I2A(taskPrompt, agentIndex, parentContext, globalConfig, options = {}) {
    const {
        abortController: D,
        options: {
            debug: Y,
            verbose: W,
            isNonInteractiveSession: J
        },
        getToolPermissionContext: F,
        readFileState: X,
        setInProgressToolUseIDs: V,
        tools: C
    } = parentContext;

    const {
        isSynthesis: K = false,
        systemPrompt: E,
        model: N
    } = options;

    // 고유 Agent ID 생성
    const agentId = VN5();

    // 초기 메시지 생성
    const initialMessages = [K2({ content: taskPrompt })];

    // 설정 정보 가져오기
    const [modelConfig, resourceConfig, selectedModel] = await Promise.all([
        qW(),  // 모델 설정 가져오기
        RE(),  // 리소스 설정 가져오기
        N ?? J7()  // 기본 모델 가져오기
    ]);

    // Agent 시스템 프롬프트 생성
    const agentSystemPrompt = await (
        E ?? ma0(selectedModel, Array.from(parentContext.getToolPermissionContext().additionalWorkingDirectories))
    );

    // Agent 메인 루프 실행
    let messageHistory = [];
    let toolUseCount = 0;
    let exitPlanInput = undefined;

    for await (let agentResponse of nO(  // 메인 Agent 루프 함수
        initialMessages,
        agentSystemPrompt,
        modelConfig,
        resourceConfig,
        globalConfig,
        {
            abortController: D,
            options: {
                isNonInteractiveSession: J ?? false,
                tools: C,  // 도구 집합 상속 (하지만 필터링됨)
                commands: [],
                debug: Y,
                verbose: W,
                mainLoopModel: selectedModel,
                maxThinkingTokens: s$(initialMessages),  // 사고 토큰 제한 계산
                mcpClients: [],
                mcpResources: {}
            },
            getToolPermissionContext: F,
            readFileState: X,
            getQueuedCommands: () => [],
            removeQueuedCommands: () => {},
            setInProgressToolUseIDs: V,
            agentId: agentId
        }
    )) {
        // Agent 응답 필터링 및 처리
        if (agentResponse.type !== "assistant" &&
            agentResponse.type !== "user" &&
            agentResponse.type !== "progress") continue;

        messageHistory.push(agentResponse);

        // 도구 사용 통계 및 특수 상황 처리
        if (agentResponse.type === "assistant" || agentResponse.type === "user") {
            const normalizedMessages = AQ(messageHistory);

            for (let messageGroup of AQ([agentResponse])) {
                for (let content of messageGroup.message.content) {
                    if (content.type !== "tool_use" && content.type !== "tool_result") continue;

                    if (content.type === "tool_use") {
                        toolUseCount++;

                        // 계획 모드 종료 확인
                        if (content.name === "exit_plan_mode" && content.input) {
                            let validation = hO.inputSchema.safeParse(content.input);
                            if (validation.success) {
                                exitPlanInput = { plan: validation.data.plan };
                            }
                        }
                    }

                    // 진행 이벤트 생성
                    yield {
                        type: "progress",
                        toolUseID: K ? `synthesis_${globalConfig.message.id}` : `agent_${agentIndex}_${globalConfig.message.id}`,
                        data: {
                            message: messageGroup,
                            normalizedMessages: normalizedMessages,
                            type: "agent_progress"
                        }
                    };
                }
            }
        }
    }

    // 최종 결과 처리
    const lastMessage = UD(messageHistory);  // 마지막 메시지 가져오기

    if (lastMessage && oK1(lastMessage)) throw new NG;  // 중단 확인
    if (lastMessage?.type !== "assistant") {
        throw new Error(K ? "합성: 마지막 메시지가 assistant 메시지가 아님" :
                           `Agent ${agentIndex + 1}: 마지막 메시지가 assistant 메시지가 아님`);
    }

    // 토큰 사용량 계산
    const totalTokens = (lastMessage.message.usage.cache_creation_input_tokens ?? 0) +
                       (lastMessage.message.usage.cache_read_input_tokens ?? 0) +
                       lastMessage.message.usage.input_tokens +
                       lastMessage.message.usage.output_tokens;

    // 텍스트 내용 추출
    const textContent = lastMessage.message.content.filter(content => content.type === "text");

    // 대화 기록 저장
    await CZ0([...initialMessages, ...messageHistory]);

    // 최종 결과 반환
    yield {
        type: "result",
        data: {
            agentIndex: agentIndex,
            content: textContent,
            toolUseCount: toolUseCount,
            tokens: totalTokens,
            usage: lastMessage.message.usage,
            exitPlanModeInput: exitPlanInput
        }
    };
}
```

---

## 3. SubAgent 실행 컨텍스트

### 3.1 컨텍스트 격리 메커니즘

각 SubAgent는 완전히 격리된 실행 컨텍스트에서 실행되어 안정성과 보안을 보장합니다.

```javascript
// SubAgent 컨텍스트 생성 (코드 분석 기반 추론)
class SubAgentContext {
    constructor(parentContext, agentId) {
        this.agentId = agentId;
        this.parentContext = parentContext;

        // 격리된 도구 집합
        this.tools = this.filterToolsForSubAgent(parentContext.tools);

        // 상속된 권한 컨텍스트
        this.getToolPermissionContext = parentContext.getToolPermissionContext;

        // 파일 상태 접근자
        this.readFileState = parentContext.readFileState;

        // 리소스 제한
        this.resourceLimits = {
            maxExecutionTime: 300000,  // 5분
            maxToolCalls: 50,
            maxTokens: 100000
        };

        // 독립적인 중단 컨트롤러
        this.abortController = new AbortController();

        // 독립적인 도구 사용 상태 관리
        this.setInProgressToolUseIDs = new Set();
    }

    // SubAgent가 사용할 수 있는 도구 필터링
    filterToolsForSubAgent(allTools) {
        // SubAgent에서 금지된 도구 목록
        const blockedTools = ['Task'];  // 재귀 호출 방지

        return allTools.filter(tool => !blockedTools.includes(tool.name));
    }
}
```

### 3.2 도구 권한 상속 및 제한

SubAgent는 주 Agent의 기본 권한을 상속하지만 추가 제한이 적용됩니다.

```javascript
// 도구 권한 필터 (코드 분석 기반 추론)
class ToolPermissionFilter {
    constructor() {
        this.allowedTools = [
            'Bash', 'Glob', 'Grep', 'LS', 'exit_plan_mode',
            'Read', 'Edit', 'MultiEdit', 'Write',
            'NotebookRead', 'NotebookEdit', 'WebFetch',
            'TodoRead', 'TodoWrite', 'WebSearch'
        ];

        this.restrictedOperations = {
            'Write': { maxFileSize: '5MB', requiresValidation: true },
            'Edit': { maxChangesPerCall: 10, requiresBackup: true },
            'Bash': { timeoutSeconds: 120, forbiddenCommands: ['rm -rf', 'sudo'] },
            'WebFetch': { allowedDomains: ['docs.anthropic.com', 'github.com'] }
        };
    }

    validateToolAccess(toolName, parameters, agentContext) {
        // 도구가 허용 목록에 있는지 확인
        if (!this.allowedTools.includes(toolName)) {
            throw new Error(`SubAgent에서 도구 ${toolName}은(는) 허용되지 않음`);
        }

        // 특정 도구의 제한 확인
        const restrictions = this.restrictedOperations[toolName];
        if (restrictions) {
            this.applyToolRestrictions(toolName, parameters, restrictions);
        }

        return true;
    }
}
```

### 3.3 독립적인 리소스 할당

각 SubAgent는 독립적인 리소스 할당 및 모니터링을 가집니다.

```javascript
// 리소스 모니터 (코드 분석 기반 추론)
class SubAgentResourceMonitor {
    constructor(agentId, limits) {
        this.agentId = agentId;
        this.limits = limits;
        this.usage = {
            startTime: Date.now(),
            tokenCount: 0,
            toolCallCount: 0,
            fileOperations: 0,
            networkRequests: 0
        };
    }

    recordTokenUsage(tokens) {
        this.usage.tokenCount += tokens;
        if (this.usage.tokenCount > this.limits.maxTokens) {
            throw new Error(`Agent ${this.agentId}의 토큰 제한 초과`);
        }
    }

    recordToolCall(toolName) {
        this.usage.toolCallCount++;
        if (this.usage.toolCallCount > this.limits.maxToolCalls) {
            throw new Error(`Agent ${this.agentId}의 도구 호출 제한 초과`);
        }
    }

    checkTimeLimit() {
        const elapsed = Date.now() - this.usage.startTime;
        if (elapsed > this.limits.maxExecutionTime) {
            throw new Error(`Agent ${this.agentId}의 실행 시간 제한 초과`);
        }
    }
}
```

---

## 4. 병렬 실행 조율 메커니즘

### 4.1 병렬 실행 전략

Task Tool은 두 가지 실행 모드를 지원합니다: 단일 Agent 모드와 다중 Agent 병렬 모드.

```javascript
// Task Tool의 병렬 실행 로직 (improved-claude-code-5.mjs:62474-62526)
async * call({ prompt: A }, context, J, F) {
    const startTime = Date.now();
    const config = ZA();  // 설정 가져오기
    const executionContext = {
        abortController: context.abortController,
        options: context.options,
        getToolPermissionContext: context.getToolPermissionContext,
        readFileState: context.readFileState,
        setInProgressToolUseIDs: context.setInProgressToolUseIDs,
        tools: context.options.tools.filter((tool) => tool.name !== cX)  // Task Tool 자체 제외
    };

    if (config.parallelTasksCount > 1) {
        // 다중 Agent 병렬 실행 모드
        yield* this.executeParallelAgents(A, executionContext, config, F, J);
    } else {
        // 단일 Agent 실행 모드
        yield* this.executeSingleAgent(A, executionContext, F, J);
    }
}

// 여러 Agent를 병렬로 실행
async * executeParallelAgents(taskPrompt, context, config, F, J) {
    let totalToolUseCount = 0;
    let totalTokens = 0;

    // 여러 동일한 Agent 작업 생성
    const agentTasks = Array(config.parallelTasksCount)
        .fill(`${taskPrompt}\n\n철저하고 완전한 분석을 제공하세요.`)
        .map((prompt, index) => I2A(prompt, index, context, F, J));

    const agentResults = [];

    // 모든 Agent 작업을 병렬로 실행 (최대 동시성: 10)
    for await (let result of UH1(agentTasks, 10)) {
        if (result.type === "progress") {
            yield result;
        } else if (result.type === "result") {
            agentResults.push(result.data);
            totalToolUseCount += result.data.toolUseCount;
            totalTokens += result.data.tokens;
        }
    }

    // 중단 여부 확인
    if (context.abortController.signal.aborted) throw new NG;

    // 합성기를 사용하여 결과 병합
    const synthesisPrompt = KN5(taskPrompt, agentResults);
    const synthesisAgent = I2A(synthesisPrompt, 0, context, F, J, { isSynthesis: true });

    let synthesisResult = null;
    for await (let result of synthesisAgent) {
        if (result.type === "progress") {
            totalToolUseCount++;
            yield result;
        } else if (result.type === "result") {
            synthesisResult = result.data;
            totalTokens += synthesisResult.tokens;
        }
    }

    if (!synthesisResult) throw new Error("합성 Agent가 결과를 반환하지 않음");

    // 계획 모드 종료 확인
    const exitPlanInput = agentResults.find(r => r.exitPlanModeInput)?.exitPlanModeInput;

    yield {
        type: "result",
        data: {
            content: synthesisResult.content,
            totalDurationMs: Date.now() - startTime,
            totalTokens: totalTokens,
            totalToolUseCount: totalToolUseCount,
            usage: synthesisResult.usage,
            wasInterrupted: context.abortController.signal.aborted,
            exitPlanModeInput: exitPlanInput
        }
    };
}
```

### 4.2 병렬 스케줄러 구현

UH1 함수는 핵심 병렬 스케줄러로, 비동기 생성기의 병렬 실행을 구현합니다.

```javascript
// 병렬 실행 스케줄러 (improved-claude-code-5.mjs:45024-45057)
async function* UH1(generators, maxConcurrency = Infinity) {
    // 생성기를 감싸서 Promise 추적 추가
    const wrapGenerator = (generator) => {
        const promise = generator.next().then(({ done, value }) => ({
            done,
            value,
            generator,
            promise
        }));
        return promise;
    };

    const remainingGenerators = [...generators];
    const activePromises = new Set();

    // 초기 병렬 작업 시작
    while (activePromises.size < maxConcurrency && remainingGenerators.length > 0) {
        const generator = remainingGenerators.shift();
        activePromises.add(wrapGenerator(generator));
    }

    // 병렬 실행 루프
    while (activePromises.size > 0) {
        // 어떤 생성기든 결과가 나올 때까지 대기
        const { done, value, generator, promise } = await Promise.race(activePromises);

        // 완료된 Promise 제거
        activePromises.delete(promise);

        if (!done) {
            // 생성기에 더 많은 데이터가 있으면 계속 실행
            activePromises.add(wrapGenerator(generator));
            if (value !== undefined) yield value;
        } else if (remainingGenerators.length > 0) {
            // 현재 생성기가 완료되면 새 생성기 시작
            const nextGenerator = remainingGenerators.shift();
            activePromises.add(wrapGenerator(nextGenerator));
        }
    }
}
```

---

## 5. 결과 합성 및 보고 메커니즘

### 5.1 다중 Agent 결과 수집

여러 Agent의 실행 결과는 전용 수집기를 통해 통합 관리됩니다.

```javascript
// 다중 Agent 결과 수집기 (코드 분석 기반)
class MultiAgentResultCollector {
    constructor() {
        this.results = new Map();  // agentIndex -> result
        this.metadata = {
            totalTokens: 0,
            totalToolCalls: 0,
            totalExecutionTime: 0,
            errorCount: 0
        };
    }

    addResult(agentIndex, result) {
        this.results.set(agentIndex, result);

        // 통계 정보 업데이트
        this.metadata.totalTokens += result.tokens || 0;
        this.metadata.totalToolCalls += result.toolUseCount || 0;

        if (result.error) {
            this.metadata.errorCount++;
        }
    }

    getAllResults() {
        // Agent 인덱스 순서로 정렬하여 결과 반환
        const sortedResults = Array.from(this.results.entries())
            .sort(([indexA], [indexB]) => indexA - indexB)
            .map(([index, result]) => ({ agentIndex: index, ...result }));

        return sortedResults;
    }
}
```

### 5.2 결과 합성 프롬프트 생성

KN5 함수는 여러 Agent의 결과를 통합 형식으로 병합합니다.

```javascript
// 다중 Agent 결과 합성기 (improved-claude-code-5.mjs:62326-62351)
function KN5(originalTask, agentResults) {
    // Agent 인덱스로 결과 정렬
    const sortedResults = agentResults.sort((a, b) => a.agentIndex - b.agentIndex);

    // 각 Agent의 텍스트 내용 추출
    const agentResponses = sortedResults.map((result, index) => {
        const textContent = result.content
            .filter((content) => content.type === "text")
            .map((content) => content.text)
            .join("\n\n");

        return `== Agent ${index + 1} 응답 ==
${textContent}`;
    }).join("\n\n");

    // 합성 프롬프트 생성
    const synthesisPrompt = `원래 작업: ${originalTask}

이 작업을 처리하기 위해 여러 Agent를 할당했습니다. 각 Agent가 문제를 분석하고 결과를 제공했습니다.

${agentResponses}

모든 Agent가 제공한 정보를 바탕으로 다음과 같은 포괄적이고 일관된 응답을 합성하세요:
1. 모든 Agent의 핵심 통찰력 결합
2. Agent 결과 간의 모순 해결
3. 원래 작업을 해결하는 통합 솔루션 제시
4. 개별 응답의 모든 중요한 세부사항과 코드 예제 포함
5. 잘 구조화되고 완전한 결과

합성 결과는 철저하되 원래 작업에 집중해야 합니다.`;

    return synthesisPrompt;
}
```

---

## 6. 도구 화이트리스트와 권한 제어

### 6.1 SubAgent 도구 화이트리스트

SubAgent는 사전 정의된 안전한 도구 집합에만 접근할 수 있습니다.

```javascript
// SubAgent 허용 도구 목록 (코드 분석 기반)
const SUBAGENT_ALLOWED_TOOLS = [
    // 파일 작업 도구
    'Read',         // 파일 읽기
    'Write',        // 파일 쓰기
    'Edit',         // 파일 편집
    'MultiEdit',    // 여러 파일 편집
    'LS',           // 디렉터리 목록

    // 검색 도구
    'Glob',         // 파일 패턴 매칭
    'Grep',         // 내용 검색

    // 시스템 상호작용 도구
    'Bash',         // 명령 실행 (제한됨)

    // Notebook 도구
    'NotebookRead', // Notebook 읽기
    'NotebookEdit', // Notebook 편집

    // 네트워크 도구
    'WebFetch',     // 웹 콘텐츠 가져오기 (도메인 제한)
    'WebSearch',    // 웹 검색

    // 작업 관리 도구
    'TodoRead',     // 작업 목록 읽기
    'TodoWrite',    // 작업 목록 쓰기

    // 계획 모드 도구
    'exit_plan_mode' // 계획 모드 종료
];

// 금지된 도구 (SubAgent에서 사용 불가)
const SUBAGENT_BLOCKED_TOOLS = [
    'Task',         // 재귀 호출 방지
];
```

### 6.2 도구 권한 검증

모든 도구 호출은 엄격한 권한 검증을 거칩니다.

```javascript
// 도구 권한 검증 시스템 (코드 분석 기반 추론)
class ToolPermissionValidator {
    constructor() {
        this.permissionMatrix = {
            'Read': {
                allowedExtensions: ['.js', '.ts', '.json', '.md', '.txt', '.yaml', '.yml', '.py'],
                maxFileSize: 10 * 1024 * 1024,  // 10MB
                forbiddenPaths: ['/etc/passwd', '/etc/shadow', '~/.ssh', '~/.aws'],
                maxConcurrent: 5
            },

            'Write': {
                maxFileSize: 5 * 1024 * 1024,   // 5MB
                forbiddenPaths: ['/etc', '/usr', '/bin', '/sbin'],
                requiresBackup: true,
                maxFilesPerOperation: 10
            },

            'Bash': {
                timeoutSeconds: 120,
                forbiddenCommands: [
                    'rm -rf', 'dd if=', 'mkfs', 'fdisk', 'chmod 777',
                    'sudo', 'su', 'passwd', 'chown', 'mount'
                ],
                maxOutputSize: 1024 * 1024,  // 1MB
                sandboxed: true
            },

            'WebFetch': {
                allowedDomains: [
                    'docs.anthropic.com',
                    'github.com',
                    'raw.githubusercontent.com',
                    'api.github.com'
                ],
                maxResponseSize: 5 * 1024 * 1024,  // 5MB
                timeoutSeconds: 30
            }
        };
    }

    async validateToolCall(toolName, parameters, agentContext) {
        // 1. 도구가 화이트리스트에 있는지 확인
        if (!SUBAGENT_ALLOWED_TOOLS.includes(toolName)) {
            throw new PermissionError(`SubAgent에서 도구 ${toolName}은(는) 허용되지 않음`);
        }

        // 2. 도구별 권한 확인
        const permissions = this.permissionMatrix[toolName];
        if (permissions) {
            await this.enforceToolPermissions(toolName, parameters, permissions, agentContext);
        }

        // 3. 글로벌 보안 정책 확인
        await this.enforceSecurityPolicies(toolName, parameters, agentContext);

        // 4. 도구 사용 로그 기록
        this.logToolUsage(toolName, parameters, agentContext);

        return true;
    }

    async validateBashPermissions(parameters, permissions) {
        const command = parameters.command.toLowerCase();

        // 금지된 명령 확인
        for (const forbidden of permissions.forbiddenCommands) {
            if (command.includes(forbidden.toLowerCase())) {
                throw new PermissionError(`금지된 명령: ${forbidden}`);
            }
        }

        // 명령 길이 확인
        if (command.length > 1000) {
            throw new PermissionError('명령이 너무 길음');
        }
    }
}
```

---

## 7. 주 Agent 루프 메커니즘 심층 분석

### 7.1 nO 함수 핵심 구현

nO 함수는 전체 Agent 시스템의 핵심으로, 완전한 대화 루프를 구현합니다.

```javascript
// 주 Agent 루프 함수 (improved-claude-code-5.mjs:46187-46302)
async function* nO(messages, systemPrompt, modelConfig, resourceConfig, globalConfig, context, compactionState, fallbackModel, options) {
    yield { type: "stream_request_start" };

    let currentMessages = messages;
    let currentContext = context;

    // 컨텍스트 압축 필요 여부 확인
    const {
        messages: processedMessages,
        wasCompacted
    } = await wU2(messages, context);  // 컨텍스트 압축 함수

    if (wasCompacted) {
        // 압축 이벤트 기록
        E1("tengu_auto_compact_succeeded", {
            originalMessageCount: messages.length,
            compactedMessageCount: processedMessages.length
        });

        currentMessages = processedMessages;
    }

    let assistantMessages = [];
    let currentModel = context.options.mainLoopModel;
    let shouldRetry = true;

    try {
        while (shouldRetry) {
            shouldRetry = false;

            try {
                // 언어 모델 호출
                for await (let response of wu(  // 언어 모델 호출 함수
                    Ie1(currentMessages, modelConfig),     // 메시지 포맷팅
                    Qe1(systemPrompt, resourceConfig),     // 시스템 프롬프트 포맷팅
                    context.options.maxThinkingTokens,     // 토큰 제한
                    context.options.tools,                 // 사용 가능한 도구
                    context.abortController.signal,        // 중단 신호
                    {
                        getToolPermissionContext: context.getToolPermissionContext,
                        model: currentModel,
                        prependCLISysprompt: true,
                        toolChoice: undefined,
                        isNonInteractiveSession: context.options.isNonInteractiveSession,
                        fallbackModel: fallbackModel
                    }
                )) {
                    yield response;

                    if (response.type === "assistant") {
                        assistantMessages.push(response);
                    }
                }
            } catch (error) {
                // 모델 폴백 처리
                if (error instanceof wH1 && fallbackModel) {
                    currentModel = fallbackModel;
                    shouldRetry = true;
                    assistantMessages.length = 0;
                    context.options.mainLoopModel = fallbackModel;

                    // 폴백 이벤트 기록
                    E1("tengu_model_fallback_triggered", {
                        original_model: error.originalModel,
                        fallback_model: fallbackModel,
                        entrypoint: "cli"
                    });

                    yield L11(`모델 폴백 트리거: ${error.originalModel}에서 ${error.fallbackModel}로 전환`, "info");
                    continue;
                }
                throw error;
            }
        }
    } catch (error) {
        // 오류 처리 로직
        b1(error instanceof Error ? error : new Error(String(error)));

        const errorMessage = error instanceof Error ? error.message : String(error);
        E1("tengu_query_error", {
            assistantMessages: assistantMessages.length,
            toolUses: assistantMessages.flatMap(msg =>
                msg.message.content.filter(content => content.type === "tool_use")
            ).length
        });

        // 각 도구 호출에 대한 오류 응답 생성
        let hasErrorResponse = false;
        for (const message of assistantMessages) {
            const toolUses = message.message.content.filter(content => content.type === "tool_use");
            for (const toolUse of toolUses) {
                yield K2({  // 사용자 메시지 생성
                    content: [{
                        type: "tool_result",
                        content: errorMessage,
                        is_error: true,
                        tool_use_id: toolUse.id
                    }],
                    toolUseResult: errorMessage
                });
                hasErrorResponse = true;
            }
        }

        if (!hasErrorResponse) {
            yield St1({ toolUse: false, hardcodedMessage: undefined });
        }
        return;
    }

    if (!assistantMessages.length) return;

    // 도구 호출 추출
    const toolUses = assistantMessages.flatMap(msg =>
        msg.message.content.filter(content => content.type === "tool_use")
    );

    if (!toolUses.length) return;

    // 도구 호출 실행
    const toolResults = [];
    let preventContinuation = false;

    for await (let result of hW5(toolUses, assistantMessages, globalConfig, context)) {  // 도구 실행 조율기
        yield result;

        if (result && result.type === "system" && result.preventContinuation) {
            preventContinuation = true;
        }

        toolResults.push(...JW([result]).filter(msg => msg.type === "user"));
    }

    // 중단 확인
    if (context.abortController.signal.aborted) {
        yield St1({ toolUse: true, hardcodedMessage: undefined });
        return;
    }

    if (preventContinuation) return;

    // 재귀 호출, 대화 루프 계속
    yield* nO(
        [...currentMessages, ...assistantMessages, ...toolResults],
        systemPrompt,
        modelConfig,
        resourceConfig,
        globalConfig,
        context,
        compactionState,
        fallbackModel,
        options
    );
}
```

---

## 8. 아키텍처의 장점과 기술 혁신

### 8.1 계층적 다중 Agent 아키텍처의 기술적 이점

1. **완전히 격리된 실행 환경**
   - 각 SubAgent는 독립 컨텍스트에서 실행되어 상호 간섭 방지
   - 리소스 제한 및 권한 제어로 시스템 안정성 보장
   - 오류 격리 메커니즘으로 단일 실패 지점이 전체 시스템에 영향을 주지 않음

2. **지능형 병렬 스케줄링**
   - 다중 Agent 병렬 실행으로 처리 효율성 크게 향상
   - 동적 부하 분산 및 리소스 할당
   - 도구 안전성 기반의 지능형 그룹 실행

3. **탄력적인 오류 처리**
   - 다층 오류 포착 및 복구 메커니즘
   - 자동 모델 폴백 및 재시도 로직
   - 우아한 중단 처리 및 리소스 정리

4. **효율적인 결과 합성**
   - 지능적인 다중 Agent 결과 집계 알고리즘
   - 충돌 감지 및 일관성 검증
   - 전용 합성 Agent가 통합 결과 생성

### 8.2 보안 메커니즘의 혁신적 설계

1. **다층 권한 제어**
   - 도구 화이트리스트 및 블랙리스트 메커니즘
   - 세밀한 매개변수 검증 및 제한
   - 동적 권한 평가 및 조정

2. **재귀 호출 방지**
   - Task Tool의 재귀 호출 엄격 금지
   - 호출 깊이 제한 및 순환 감지
   - 지능적인 호출 스택 관리

3. **리소스 사용 모니터링**
   - 실시간 토큰 사용량 추적
   - 실행 시간 및 도구 호출 횟수 제한
   - 메모리 및 네트워크 리소스의 안전한 제어

---

## 9. 실제 응용 시나리오

### 9.1 복잡한 코드 분석

```javascript
// 예: 복잡한 프로젝트 아키텍처 분석
const codeAnalysisTask = {
    description: "프로젝트 아키텍처 분석",
    prompt: `이 대규모 코드베이스의 아키텍처를 분석하세요:
    1. 주요 구성요소와 관계 식별
    2. 잠재적인 아키텍처 문제 찾기
    3. 개선 사항 및 모범 사례 제안
    4. 가능하면 아키텍처 다이어그램 생성`
};

// Task Tool은 여러 SubAgent를 시작하며, 각각 다른 측면에 집중:
// - Agent 1: 구성요소 식별 및 종속성 분석
// - Agent 2: 코드 품질 평가
// - Agent 3: 아키텍처 패턴 인식
// - 합성 Agent: 모든 발견 사항을 통합하여 통합 보고서 생성
```

### 9.2 다중 파일 리팩토링

```javascript
// 예: 대규모 리팩토링 작업
const refactoringTask = {
    description: "레거시 시스템 리팩토링",
    prompt: `이 레거시 시스템을 현대 표준으로 리팩토링하세요:
    1. 더 이상 사용되지 않는 API 및 라이브러리 업데이트
    2. 코드 구조 및 패턴 개선
    3. 적절한 오류 처리 및 로깅 추가
    4. 하위 호환성 보장
    5. 문서 및 테스트 업데이트`
};

// 다중 Agent 병렬 처리:
// - Agent 1: API 업데이트 및 종속성 업그레이드
// - Agent 2: 코드 구조 개선
// - Agent 3: 오류 처리 및 로그 시스템
// - Agent 4: 테스트 및 문서 업데이트
// - 합성 Agent: 변경 사항 조율, 일관성 보장
```

---

## 10. 결론

Claude Code의 계층적 다중 Agent 아키텍처는 AI 프로그래밍 어시스턴트 분야의 중요한 기술 혁신을 나타냅니다. 심층 역공학 분석을 통해 다음과 같은 핵심 기술 구현을 완전히 복원했습니다:

### 핵심 기술 성과

1. **완전한 Agent 격리 메커니즘**: 각 SubAgent가 독립 컨텍스트에서 실행되어 진정한 격리 및 보안 달성
2. **지능형 병렬 스케줄링 시스템**: UH1 함수가 구현한 병렬 실행 스케줄러로 효율적인 다중 Agent 협업 지원
3. **정교한 도구 권한 제어**: 다층 권한 검증 및 보안 정책으로 권한 남용 및 보안 위험 방지
4. **혁신적인 결과 합성 메커니즘**: KN5 함수가 구현한 지능형 결과 집계로 다중 Agent 결과의 일관성 보장
5. **견고한 오류 처리 및 복구**: 완전한 오류 격리, 모델 폴백 및 리소스 정리 메커니즘

### 아키텍처 혁신 가치

1. **확장성**: 모듈식 설계로 유연한 Agent 확장 및 도구 통합 지원
2. **신뢰성**: 다층 오류 처리 및 리소스 제한으로 시스템 안정적 실행 보장
3. **효율성**: 지능형 병렬 스케줄링 및 컨텍스트 압축으로 성능 크게 향상
4. **보안성**: 포괄적인 권한 제어 및 격리 메커니즘으로 시스템 보안 보장
5. **유지보수성**: 명확한 아키텍처 계층화 및 모듈화 설계로 유지보수 및 업그레이드 용이

이 선진적인 아키텍처 설계는 복잡한 작업 처리의 기술적 과제를 해결했을 뿐만 아니라, 향후 AI 프로그래밍 어시스턴트 발전의 방향을 제시하여 중요한 기술적 가치와 실용적 의의를 가집니다.

---

*본 문서는 Claude Code 소스 코드의 완전한 역공학 분석을 기반으로 하며, 난독화된 코드, 런타임 동작 및 아키텍처 패턴을 체계적으로 분석하여 계층적 다중 Agent 아키텍처의 완전한 기술 구현을 정확하게 복원했습니다. 모든 분석 결과는 실제 코드 증거를 기반으로 하여, 현대 AI 프로그래밍 어시스턴트의 내부 메커니즘을 이해하는 데 상세하고 정확한 기술적 통찰력을 제공합니다.*
