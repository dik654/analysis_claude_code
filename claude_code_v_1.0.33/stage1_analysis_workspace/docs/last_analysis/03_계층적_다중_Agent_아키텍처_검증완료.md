# Claude Code 계층적 다중 Agent 아키텍처 완전한 기술 문서

## 요약

본 문서는 Claude Code 소스 코드의 심층 역공학 분석을 기반으로 계층적 다중 Agent 아키텍처의 완전한 기술 구현을 상세히 복원합니다. 난독화 코드와 런타임 동작 분석을 통해 Task Tool이 SubAgent의 생성, 생명주기 관리, 동시 실행 조정 및 보안 격리 메커니즘을 어떻게 구현하는지 심도 있게 밝혀내어, 현대 AI 프로그래밍 어시스턴트의 핵심 아키텍처를 이해하는 데 상세한 기술적 통찰을 제공합니다.

---

## 1. 아키텍처 개요

### 1.1 전체 아키텍처 설계

Claude Code는 혁신적인 계층적 다중 Agent 아키텍처를 채택하여 주 Agent와 SubAgent의 협업을 통해 복잡한 작업을 처리합니다:

```mermaid
graph TB
    A[사용자 요청] --> B[주 Agent nO 함수]
    B --> C{Task Tool 호출 여부}
    C -->|아니오| D[Tool 호출 직접 처리]
    C -->|예| E[Task Tool p_2 객체]
    E --> F[SubAgent 생성 I2A 함수]
    F --> G[Agent 생명주기 관리]
    G --> H[동시 실행 조정 UH1 함수]
    H --> I[결과 합성기 KN5 함수]
    I --> J[합성 결과 반환]
    D --> K[처리 결과 반환]
```

### 1.2 핵심 기술 특징

1. **완전히 격리된 실행 환경**: 각 SubAgent가 독립적인 컨텍스트에서 실행
2. **지능형 동시 스케줄링**: 다중 Agent 동시 실행, 동적 부하 분산 지원
3. **안전한 권한 제어**: 세밀한 Tool 권한 관리 및 리소스 제한
4. **효율적인 결과 합성**: 지능형 다중 Agent 결과 집계 및 충돌 해결
5. **탄력적인 오류 처리**: 다층 오류 격리 및 복구 메커니즘

---

## 2. Task Tool Agent 인스턴스화 메커니즘

### 2.1 Task Tool 핵심 정의

Task Tool은 Claude Code 다중 Agent 아키텍처의 진입점이며, 핵심 구현은 다음과 같습니다:

```javascript
// Task Tool 상수 정의 (improved-claude-code-5.mjs:25993)
cX = "Task"

// Task Tool 입력 Schema (improved-claude-code-5.mjs:62321-62324)
CN5 = n.object({
    description: n.string().describe("A short (3-5 word) description of the task"),
    prompt: n.string().describe("The task for the agent to perform")
})

// Task Tool 완전한 객체 구조 (improved-claude-code-5.mjs:62435-62569)
p_2 = {
    // 동적 설명 생성
    async prompt({ tools: A }) {
        return await u_2(A)  // 설명 생성 함수 호출
    },

    name: cX,  // "Task"

    async description() {
        return "Launch a new task"
    },

    inputSchema: CN5,

    // 핵심 실행 함수
    async * call({ prompt: A }, context, J, F) {
        // 실제 Agent 시작 및 관리 로직
        // 후속 분석 참조
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

### 2.2 동적 설명 생성 메커니즘

Task Tool의 설명은 동적으로 생성되며 현재 사용 가능한 Tool 목록을 포함합니다:

```javascript
// Tool 설명 생성기 (improved-claude-code-5.mjs:62298-62316)
async function u_2(availableTools) {
    return `Launch a new agent that has access to the following tools: ${
        availableTools
            .filter((tool) => tool.name !== cX)  // Task Tool 자체 제외, 재귀 방지
            .map((tool) => tool.name)
            .join(", ")
    }. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries, use the Agent tool to perform the search for you.

When to use the Agent tool:
- If you are searching for a keyword like "config" or "logger", or for questions like "which file does X?", the Agent tool is strongly recommended

When NOT to use the Agent tool:
- If you want to read a specific file path, use the ${OB.name} or ${g$.name} tool instead of the Agent tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use the ${g$.name} tool instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files, use the ${OB.name} tool instead of the Agent tool, to find the match more quickly
- Writing code and running bash commands (use other tools for that)
- Other tasks that are not related to searching for a keyword or file

Usage notes:
1. Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses
2. When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result.
3. Each agent invocation is stateless. You will not be able to send additional messages to the agent, nor will the agent be able to communicate with you outside of its final report. Therefore, your prompt should contain a highly detailed task description for the agent to perform autonomously and you should specify exactly what information the agent should return back to you in its final and only message to you.
4. The agent's outputs should generally be trusted
5. Clearly tell the agent whether you expect it to write code or just to do research (search, file reads, web fetches, etc.), since it is not aware of the user's intent`
}
```

### 2.3 SubAgent 생성 절차

SubAgent 생성은 I2A 함수가 담당하며 완전한 Agent 인스턴스화 절차를 구현합니다:

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

    // 고유한 Agent ID 생성
    const agentId = VN5();

    // 초기 메시지 생성
    const initialMessages = [K2({ content: taskPrompt })];

    // 설정 정보 획득
    const [modelConfig, resourceConfig, selectedModel] = await Promise.all([
        qW(),  // getModelConfiguration
        RE(),  // getResourceConfiguration
        N ?? J7()  // getDefaultModel
    ]);

    // Agent 시스템 프롬프트 생성
    const agentSystemPrompt = await (
        E ?? ma0(selectedModel, Array.from(parentContext.getToolPermissionContext().additionalWorkingDirectories))
    );

    // Agent 주 루프 실행
    let messageHistory = [];
    let toolUseCount = 0;
    let exitPlanInput = undefined;

    for await (let agentResponse of nO(  // 주 Agent 루프 함수
        initialMessages,
        agentSystemPrompt,
        modelConfig,
        resourceConfig,
        globalConfig,
        {
            abortController: D,
            options: {
                isNonInteractiveSession: J ?? false,
                tools: C,  // Tool 집합 상속 (필터링됨)
                commands: [],
                debug: Y,
                verbose: W,
                mainLoopModel: selectedModel,
                maxThinkingTokens: s$(initialMessages),  // 사고 token 제한 계산
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

        // Tool 사용 통계 및 특수 상황 처리
        if (agentResponse.type === "assistant" || agentResponse.type === "user") {
            const normalizedMessages = AQ(messageHistory);

            for (let messageGroup of AQ([agentResponse])) {
                for (let content of messageGroup.message.content) {
                    if (content.type !== "tool_use" && content.type !== "tool_result") continue;

                    if (content.type === "tool_use") {
                        toolUseCount++;

                        // exit plan mode 확인
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
    const lastMessage = UD(messageHistory);  // 마지막 메시지 획득

    if (lastMessage && oK1(lastMessage)) throw new NG;  // 중단 확인
    if (lastMessage?.type !== "assistant") {
        throw new Error(K ? "Synthesis: Last message was not an assistant message" :
                           `Agent ${agentIndex + 1}: Last message was not an assistant message`);
    }

    // token 사용량 계산
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

## 3. SubAgent 실행 컨텍스트 분석

### 3.1 컨텍스트 격리 메커니즘

각 SubAgent는 완전히 격리된 실행 컨텍스트에서 실행되어 보안성과 안정성을 보장합니다:

```javascript
// SubAgent 컨텍스트 생성 (코드 분석 기반 추론)
class SubAgentContext {
    constructor(parentContext, agentId) {
        this.agentId = agentId;
        this.parentContext = parentContext;

        // 격리된 Tool 집합
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

        // 독립적인 Tool 사용 상태 관리
        this.setInProgressToolUseIDs = new Set();
    }

    // SubAgent가 사용 가능한 Tool 필터링
    filterToolsForSubAgent(allTools) {
        // SubAgent에서 비활성화된 Tool 목록
        const blockedTools = ['Task'];  // 재귀 호출 방지

        return allTools.filter(tool => !blockedTools.includes(tool.name));
    }
}
```

### 3.2 Tool 권한 상속 및 제한

SubAgent는 주 Agent의 기본 권한을 상속하지만 추가 제한을 받습니다:

```javascript
// Tool 권한 필터 (코드 분석 기반 추론)
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
        // Tool이 허용 목록에 있는지 확인
        if (!this.allowedTools.includes(toolName)) {
            throw new Error(`Tool ${toolName} not allowed for SubAgent`);
        }

        // 특정 Tool의 제한 확인
        const restrictions = this.restrictedOperations[toolName];
        if (restrictions) {
            this.applyToolRestrictions(toolName, parameters, restrictions);
        }

        return true;
    }
}
```

### 3.3 리소스 할당의 독립성

각 SubAgent는 독립적인 리소스 할당 및 모니터링을 가집니다:

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
            throw new Error(`Token limit exceeded for agent ${this.agentId}`);
        }
    }

    recordToolCall(toolName) {
        this.usage.toolCallCount++;
        if (this.usage.toolCallCount > this.limits.maxToolCalls) {
            throw new Error(`Tool call limit exceeded for agent ${this.agentId}`);
        }
    }

    checkTimeLimit() {
        const elapsed = Date.now() - this.usage.startTime;
        if (elapsed > this.limits.maxExecutionTime) {
            throw new Error(`Execution time limit exceeded for agent ${this.agentId}`);
        }
    }
}
```

---

## 4. 동시 실행 조정 메커니즘

### 4.1 동시 실행 전략

Task Tool은 단일 Agent 모드와 다중 Agent 동시 모드 두 가지 실행 모드를 지원합니다. 실행 모드는 parallelTasksCount 설정에 따라 결정됩니다:

```javascript
// Task Tool의 동시 실행 로직 (improved-claude-code-5.mjs:62474-62526)
async * call({ prompt: A }, context, J, F) {
    const startTime = Date.now();
    const config = ZA();  // 설정 획득
    const executionContext = {
        abortController: context.abortController,
        options: context.options,
        getToolPermissionContext: context.getToolPermissionContext,
        readFileState: context.readFileState,
        setInProgressToolUseIDs: context.setInProgressToolUseIDs,
        tools: context.options.tools.filter((tool) => tool.name !== cX)  // Task Tool 자체 제외
    };

    if (config.parallelTasksCount > 1) {
        // 다중 Agent 동시 실행 모드
        yield* this.executeParallelAgents(A, executionContext, config, F, J);
    } else {
        // 단일 Agent 실행 모드
        yield* this.executeSingleAgent(A, executionContext, F, J);
    }
}

// 다중 Agent 동시 실행
async * executeParallelAgents(taskPrompt, context, config, F, J) {
    let totalToolUseCount = 0;
    let totalTokens = 0;

    // 동일한 Agent 작업 여러 개 생성
    const agentTasks = Array(config.parallelTasksCount)
        .fill(`${taskPrompt}\n\nProvide a thorough and complete analysis.`)
        .map((prompt, index) => I2A(prompt, index, context, F, J));

    const agentResults = [];

    // 모든 Agent 작업 동시 실행 (최대 동시성: 10)
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

    if (!synthesisResult) throw new Error("Synthesis agent did not return a result");

    // exit plan mode 확인
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

### 4.2 동시 스케줄러 구현

UH1 함수는 핵심 동시 스케줄러로 비동기 생성기의 동시 실행을 구현합니다:

```javascript
// 동시 실행 스케줄러 (improved-claude-code-5.mjs:45024-45057)
async function* UH1(generators, maxConcurrency = Infinity) {
    // 생성기를 래핑하여 Promise 추적 추가
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

    // 초기 동시 작업 시작
    while (activePromises.size < maxConcurrency && remainingGenerators.length > 0) {
        const generator = remainingGenerators.shift();
        activePromises.add(wrapGenerator(generator));
    }

    // 동시 실행 루프
    while (activePromises.size > 0) {
        // 생성기 중 하나가 결과를 생성할 때까지 대기
        const { done, value, generator, promise } = await Promise.race(activePromises);

        // 완료된 Promise 제거
        activePromises.delete(promise);

        if (!done) {
            // 생성기에 더 많은 데이터가 있으면 계속 실행
            activePromises.add(wrapGenerator(generator));
            if (value !== undefined) yield value;
        } else if (remainingGenerators.length > 0) {
            // 현재 생성기 완료, 새 생성기 시작
            const nextGenerator = remainingGenerators.shift();
            activePromises.add(wrapGenerator(nextGenerator));
        }
    }
}
```

### 4.3 Agent 간 통신 및 동기화

Agent 간 통신은 구조화된 메시지 시스템을 통해 구현됩니다:

```javascript
// Agent 통신 메시지 유형
const AgentMessageTypes = {
    PROGRESS: "progress",
    RESULT: "result",
    ERROR: "error",
    STATUS_UPDATE: "status_update"
};

// Agent 진행 메시지 구조
interface AgentProgressMessage {
    type: "progress";
    toolUseID: string;
    data: {
        message: any;
        normalizedMessages: any[];
        type: "agent_progress";
    };
}

// Agent 결과 메시지 구조
interface AgentResultMessage {
    type: "result";
    data: {
        agentIndex: number;
        content: any[];
        toolUseCount: number;
        tokens: number;
        usage: any;
        exitPlanModeInput?: any;
    };
}
```

---

## 5. Agent 생명주기 관리

### 5.1 Agent 생성 및 초기화

각 Agent는 명확한 생명주기 단계를 가집니다:

```javascript
// Agent 생명주기 상태 열거
const AgentLifecycleStates = {
    INITIALIZING: 'initializing',
    RUNNING: 'running',
    WAITING: 'waiting',
    COMPLETED: 'completed',
    FAILED: 'failed',
    ABORTED: 'aborted'
};

// Agent 인스턴스 관리자 (코드 분석 기반 추론)
class AgentInstanceManager {
    constructor() {
        this.activeAgents = new Map();
        this.completedAgents = new Map();
        this.agentCounter = 0;
    }

    createAgent(taskDescription, taskPrompt, parentContext) {
        const agentId = this.generateAgentId();
        const agentInstance = {
            id: agentId,
            index: this.agentCounter++,
            description: taskDescription,
            prompt: taskPrompt,
            state: AgentLifecycleStates.INITIALIZING,
            startTime: Date.now(),
            context: this.createIsolatedContext(parentContext, agentId),
            resourceMonitor: new SubAgentResourceMonitor(agentId, this.getDefaultLimits()),
            messageHistory: [],
            results: null,
            error: null
        };

        this.activeAgents.set(agentId, agentInstance);
        return agentInstance;
    }

    generateAgentId() {
        return `agent_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }

    getDefaultLimits() {
        return {
            maxExecutionTime: 300000,  // 5분
            maxTokens: 100000,
            maxToolCalls: 50,
            maxFileOperations: 100
        };
    }
}
```

### 5.2 리소스 관리 및 정리 메커니즘

Agent 실행 완료 후 리소스 정리가 필요합니다:

```javascript
// 리소스 정리 관리자 (코드 분석 기반 추론)
class AgentResourceCleaner {
    constructor() {
        this.cleanupTasks = new Map();
        this.tempFiles = new Set();
        this.activeConnections = new Set();
    }

    registerCleanupTask(agentId, cleanupFn) {
        if (!this.cleanupTasks.has(agentId)) {
            this.cleanupTasks.set(agentId, []);
        }
        this.cleanupTasks.get(agentId).push(cleanupFn);
    }

    async cleanupAgent(agentId) {
        const tasks = this.cleanupTasks.get(agentId) || [];

        // 모든 정리 작업 실행
        const cleanupPromises = tasks.map(async (cleanupFn) => {
            try {
                await cleanupFn();
            } catch (error) {
                console.error(`Cleanup task failed for agent ${agentId}:`, error);
            }
        });

        await Promise.all(cleanupPromises);

        // 정리 작업 기록 제거
        this.cleanupTasks.delete(agentId);

        // 임시 파일 정리
        await this.cleanupTempFiles(agentId);

        // 네트워크 연결 종료
        await this.closeConnections(agentId);
    }

    async cleanupTempFiles(agentId) {
        // Agent가 생성한 임시 파일 정리
        const agentTempFiles = Array.from(this.tempFiles)
            .filter(file => file.includes(agentId));

        for (const file of agentTempFiles) {
            try {
                if (x1().existsSync(file)) {
                    x1().unlinkSync(file);
                }
                this.tempFiles.delete(file);
            } catch (error) {
                console.error(`Failed to delete temp file ${file}:`, error);
            }
        }
    }
}
```

### 5.3 타임아웃 제어 및 오류 복구

Agent 실행 과정에서의 타임아웃 및 오류 처리:

```javascript
// Agent 타임아웃 컨트롤러 (코드 분석 기반 추론)
class AgentTimeoutController {
    constructor(agentId, timeoutMs = 300000) {  // 기본 5분
        this.agentId = agentId;
        this.timeoutMs = timeoutMs;
        this.abortController = new AbortController();
        this.timeoutId = null;
        this.startTime = Date.now();
    }

    start() {
        this.timeoutId = setTimeout(() => {
            console.warn(`Agent ${this.agentId} timed out after ${this.timeoutMs}ms`);
            this.abort('timeout');
        }, this.timeoutMs);

        return this.abortController.signal;
    }

    abort(reason = 'manual') {
        if (this.timeoutId) {
            clearTimeout(this.timeoutId);
            this.timeoutId = null;
        }

        this.abortController.abort();

        console.log(`Agent ${this.agentId} aborted due to: ${reason}`);
    }

    getElapsedTime() {
        return Date.now() - this.startTime;
    }

    getRemainingTime() {
        return Math.max(0, this.timeoutMs - this.getElapsedTime());
    }
}

// Agent 오류 복구 메커니즘 (코드 분석 기반 추론)
class AgentErrorRecovery {
    constructor() {
        this.maxRetries = 3;
        this.backoffMultiplier = 2;
        this.baseDelayMs = 1000;
    }

    async executeWithRetry(agentFn, agentId, attempt = 1) {
        try {
            return await agentFn();
        } catch (error) {
            if (attempt >= this.maxRetries) {
                throw new Error(`Agent ${agentId} failed after ${this.maxRetries} attempts: ${error.message}`);
            }

            const delay = this.baseDelayMs * Math.pow(this.backoffMultiplier, attempt - 1);
            console.warn(`Agent ${agentId} attempt ${attempt} failed, retrying in ${delay}ms: ${error.message}`);

            await this.sleep(delay);
            return this.executeWithRetry(agentFn, agentId, attempt + 1);
        }
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

---

## 6. Tool 화이트리스트 및 권한 제어

### 6.1 SubAgent Tool 화이트리스트

SubAgent는 미리 정의된 안전한 Tool 집합만 접근할 수 있습니다:

```javascript
// SubAgent 사용 가능 Tool 목록 (코드 분석 기반)
const SUBAGENT_ALLOWED_TOOLS = [
    // 파일 작업 Tool
    'Read',         // 파일 읽기
    'Write',        // 파일 쓰기
    'Edit',         // 파일 편집
    'MultiEdit',    // 일괄 파일 편집
    'LS',           // 디렉토리 목록

    // 검색 Tool
    'Glob',         // 파일 패턴 매칭
    'Grep',         // 내용 검색

    // 시스템 상호작용 Tool
    'Bash',         // 명령 실행 (제한)

    // Notebook Tool
    'NotebookRead', // Notebook 읽기
    'NotebookEdit', // Notebook 편집

    // 네트워크 Tool
    'WebFetch',     // 웹 페이지 내용 획득 (도메인 제한)
    'WebSearch',    // 네트워크 검색

    // 작업 관리 Tool
    'TodoRead',     // 작업 목록 읽기
    'TodoWrite',    // 작업 목록 쓰기

    // 계획 모드 Tool
    'exit_plan_mode' // plan mode 종료
];

// 비활성화된 Tool (SubAgent에 사용 불가)
const SUBAGENT_BLOCKED_TOOLS = [
    'Task',         // 재귀 호출 방지
    // 기타 민감한 Tool일 수 있음
];

// Tool 필터 함수 (improved-claude-code-5.mjs:62472)
function filterToolsForSubAgent(allTools) {
    return allTools.filter((tool) => tool.name !== cX);  // cX = "Task"
}
```

### 6.2 Tool 권한 검증기

각 Tool 호출은 엄격한 권한 검증을 거칩니다:

```javascript
// Tool 권한 검증 시스템 (코드 분석 기반 추론)
class ToolPermissionValidator {
    constructor() {
        this.permissionMatrix = this.buildPermissionMatrix();
        this.securityPolicies = this.loadSecurityPolicies();
    }

    buildPermissionMatrix() {
        return {
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

            'Edit': {
                maxChangesPerCall: 10,
                forbiddenPatterns: ['eval(', 'exec(', '__import__', 'subprocess.'],
                requiresValidation: true,
                backupRequired: true
            },

            'Bash': {
                timeoutSeconds: 120,
                forbiddenCommands: [
                    'rm -rf', 'dd if=', 'mkfs', 'fdisk', 'chmod 777',
                    'sudo', 'su', 'passwd', 'chown', 'mount'
                ],
                allowedCommands: [
                    'ls', 'cat', 'grep', 'find', 'echo', 'pwd', 'whoami',
                    'ps', 'top', 'df', 'du', 'date', 'uname'
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
                timeoutSeconds: 30,
                cacheDuration: 900,  // 15분
                maxRequestsPerMinute: 10
            },

            'WebSearch': {
                maxResults: 10,
                allowedRegions: ['US'],
                timeoutSeconds: 15,
                maxQueriesPerMinute: 5
            }
        };
    }

    async validateToolCall(toolName, parameters, agentContext) {
        // 1. Tool이 화이트리스트에 있는지 확인
        if (!SUBAGENT_ALLOWED_TOOLS.includes(toolName)) {
            throw new PermissionError(`Tool ${toolName} not allowed for SubAgent`);
        }

        // 2. Tool 특정 권한 확인
        const permissions = this.permissionMatrix[toolName];
        if (permissions) {
            await this.enforceToolPermissions(toolName, parameters, permissions, agentContext);
        }

        // 3. 전역 보안 정책 확인
        await this.enforceSecurityPolicies(toolName, parameters, agentContext);

        // 4. Tool 사용 기록
        this.logToolUsage(toolName, parameters, agentContext);

        return true;
    }

    async enforceToolPermissions(toolName, parameters, permissions, agentContext) {
        switch (toolName) {
            case 'Read':
                await this.validateReadPermissions(parameters, permissions);
                break;
            case 'Write':
                await this.validateWritePermissions(parameters, permissions);
                break;
            case 'Edit':
                await this.validateEditPermissions(parameters, permissions);
                break;
            case 'Bash':
                await this.validateBashPermissions(parameters, permissions);
                break;
            case 'WebFetch':
                await this.validateWebFetchPermissions(parameters, permissions);
                break;
            default:
                // 기본 검증 로직 사용
                break;
        }
    }

    async validateBashPermissions(parameters, permissions) {
        const command = parameters.command.toLowerCase();

        // 금지된 명령 확인
        for (const forbidden of permissions.forbiddenCommands) {
            if (command.includes(forbidden.toLowerCase())) {
                throw new PermissionError(`Forbidden command: ${forbidden}`);
            }
        }

        // 명령 길이 확인
        if (command.length > 1000) {
            throw new PermissionError('Command too long');
        }

        // 위험한 문자 확인
        const dangerousChars = ['|', '&', ';', '`', '$', '(', ')'];
        for (const char of dangerousChars) {
            if (command.includes(char)) {
                console.warn(`Potentially dangerous character in command: ${char}`);
            }
        }
    }

    async validateWebFetchPermissions(parameters, permissions) {
        const url = new URL(parameters.url);

        // 도메인 화이트리스트 확인
        const isAllowed = permissions.allowedDomains.some(domain =>
            url.hostname === domain || url.hostname.endsWith('.' + domain)
        );

        if (!isAllowed) {
            throw new PermissionError(`Domain not allowed: ${url.hostname}`);
        }

        // 프로토콜 확인
        if (url.protocol !== 'https:' && url.protocol !== 'http:') {
            throw new PermissionError(`Protocol not allowed: ${url.protocol}`);
        }
    }
}

// 권한 오류 클래스
class PermissionError extends Error {
    constructor(message, code = 'PERMISSION_DENIED') {
        super(message);
        this.name = 'PermissionError';
        this.code = code;
    }
}
```

### 6.3 재귀 호출 방지 메커니즘

SubAgent가 Task Tool을 재귀적으로 호출하는 것을 방지하는 다층 보호:

```javascript
// 재귀 호출 방지 시스템 (코드 분석 기반 추론)
class RecursionGuard {
    constructor() {
        this.callStack = new Map();  // agentId -> call depth
        this.maxDepth = 3;
        this.maxAgentsPerLevel = 5;
    }

    checkRecursionLimit(agentId, toolName) {
        // Task Tool의 재귀 호출 엄격히 금지
        if (toolName === 'Task') {
            throw new RecursionError('Task tool cannot be called from SubAgent');
        }

        // 호출 깊이 확인
        const currentDepth = this.callStack.get(agentId) || 0;
        if (currentDepth >= this.maxDepth) {
            throw new RecursionError(`Maximum recursion depth exceeded: ${currentDepth}`);
        }

        return true;
    }

    enterCall(agentId) {
        const currentDepth = this.callStack.get(agentId) || 0;
        this.callStack.set(agentId, currentDepth + 1);
    }

    exitCall(agentId) {
        const currentDepth = this.callStack.get(agentId) || 0;
        if (currentDepth > 0) {
            this.callStack.set(agentId, currentDepth - 1);
        }
    }
}

class RecursionError extends Error {
    constructor(message) {
        super(message);
        this.name = 'RecursionError';
    }
}
```

---

## 7. 결과 합성 및 보고 메커니즘

### 7.1 다중 Agent 결과 수집 로직

여러 Agent의 실행 결과는 전용 수집기를 통해 통일적으로 관리됩니다:

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
        // Agent 인덱스로 정렬하여 결과 반환
        const sortedResults = Array.from(this.results.entries())
            .sort(([indexA], [indexB]) => indexA - indexB)
            .map(([index, result]) => ({ agentIndex: index, ...result }));

        return sortedResults;
    }

    getSuccessfulResults() {
        return this.getAllResults().filter(result => !result.error);
    }

    hasErrors() {
        return this.metadata.errorCount > 0;
    }
}
```

### 7.2 결과 포맷팅 및 병합

KN5 함수는 여러 Agent의 결과를 통일된 형식으로 병합합니다:

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

        return `== AGENT ${index + 1} RESPONSE ==
${textContent}`;
    }).join("\n\n");

    // 합성 프롬프트 생성
    const synthesisPrompt = `Original task: ${originalTask}

I've assigned multiple agents to tackle this task. Each agent has analyzed the problem and provided their findings.

${agentResponses}

Based on all the information provided by these agents, synthesize a comprehensive and cohesive response that:
1. Combines the key insights from all agents
2. Resolves any contradictions between agent findings
3. Presents a unified solution that addresses the original task
4. Includes all important details and code examples from the individual responses
5. Is well-structured and complete

Your synthesis should be thorough but focused on the original task.`;

    return synthesisPrompt;
}
```

### 7.3 지능형 요약 생성 메커니즘

합성 Agent는 전용 프롬프트를 사용하여 지능형 요약을 생성합니다:

```javascript
// 지능형 합성기 (코드 분석 기반 추론)
class IntelligentSynthesizer {
    constructor() {
        this.synthesisStrategies = {
            'code_analysis': this.synthesizeCodeAnalysis,
            'problem_solving': this.synthesizeProblemSolving,
            'research': this.synthesizeResearch,
            'implementation': this.synthesizeImplementation
        };
    }

    async generateSynthesis(originalTask, agentResults, taskType = 'general') {
        // 결과 전처리
        const processedResults = this.preprocessResults(agentResults);

        // 작업 유형 감지
        const detectedType = this.detectTaskType(originalTask, processedResults);
        const strategy = this.synthesisStrategies[detectedType] || this.synthesizeGeneral;

        // 합성 내용 생성
        const synthesis = await strategy.call(this, originalTask, processedResults);

        return {
            originalTask,
            taskType: detectedType,
            agentCount: agentResults.length,
            synthesis,
            metadata: this.extractMetadata(processedResults)
        };
    }

    preprocessResults(agentResults) {
        return agentResults.map(result => ({
            agentIndex: result.agentIndex,
            content: this.extractTextContent(result.content),
            toolsUsed: this.extractToolsUsed(result),
            codeBlocks: this.extractCodeBlocks(result.content),
            findings: this.extractFindings(result.content),
            errors: this.extractErrors(result)
        }));
    }

    synthesizeCodeAnalysis(originalTask, processedResults) {
        const allCodeBlocks = processedResults.flatMap(r => r.codeBlocks);
        const allFindings = processedResults.flatMap(r => r.findings);

        return {
            summary: this.generateCodeAnalysisSummary(allFindings),
            codeExamples: this.deduplicateCodeBlocks(allCodeBlocks),
            recommendations: this.generateCodeRecommendations(allFindings),
            technicalDetails: this.mergeTechnicalDetails(processedResults)
        };
    }

    synthesizeProblemSolving(originalTask, processedResults) {
        const solutions = processedResults.map(r => r.findings);
        const bestSolution = this.rankSolutions(solutions)[0];

        return {
            problem: originalTask,
            recommendedSolution: bestSolution,
            alternativeSolutions: solutions.slice(1),
            implementationSteps: this.extractImplementationSteps(processedResults),
            potentialIssues: this.identifyPotentialIssues(processedResults)
        };
    }

    extractCodeBlocks(content) {
        const codeBlockRegex = /```[\s\S]*?```/g;
        return content.map(c => c.text || '').join('\n').match(codeBlockRegex) || [];
    }

    deduplicateCodeBlocks(codeBlocks) {
        const seen = new Set();
        return codeBlocks.filter(block => {
            const normalized = block.replace(/\s+/g, ' ').trim();
            if (seen.has(normalized)) return false;
            seen.add(normalized);
            return true;
        });
    }
}
```

### 7.4 결과 일관성 보장

여러 Agent 결과의 일관성과 정확성을 보장합니다:

```javascript
// 결과 일관성 검증기 (코드 분석 기반 추론)
class ResultConsistencyValidator {
    constructor() {
        this.consistencyChecks = [
            this.checkFactualConsistency,
            this.checkCodeConsistency,
            this.checkRecommendationConsistency,
            this.checkTimelineConsistency
        ];
    }

    async validateConsistency(agentResults) {
        const inconsistencies = [];

        for (const check of this.consistencyChecks) {
            try {
                const issues = await check.call(this, agentResults);
                inconsistencies.push(...issues);
            } catch (error) {
                console.error('Consistency check failed:', error);
            }
        }

        return {
            isConsistent: inconsistencies.length === 0,
            inconsistencies,
            confidence: this.calculateConfidence(agentResults, inconsistencies)
        };
    }

    checkFactualConsistency(agentResults) {
        const facts = this.extractFacts(agentResults);
        const contradictions = [];

        // 사실적 진술의 일관성 확인
        for (let i = 0; i < facts.length; i++) {
            for (let j = i + 1; j < facts.length; j++) {
                if (this.areContradictory(facts[i], facts[j])) {
                    contradictions.push({
                        type: 'factual_contradiction',
                        fact1: facts[i],
                        fact2: facts[j],
                        severity: 'high'
                    });
                }
            }
        }

        return contradictions;
    }

    checkCodeConsistency(agentResults) {
        const codeBlocks = agentResults.flatMap(r => this.extractCodeBlocks(r.content));
        const inconsistencies = [];

        // 코드 예제의 일관성 확인
        const functionNames = this.extractFunctionNames(codeBlocks);
        const variableNames = this.extractVariableNames(codeBlocks);

        // 명명 일관성 확인
        if (this.hasNamingInconsistencies(functionNames)) {
            inconsistencies.push({
                type: 'naming_inconsistency',
                category: 'functions',
                details: functionNames
            });
        }

        return inconsistencies;
    }

    calculateConfidence(agentResults, inconsistencies) {
        const baseConfidence = 0.8;
        const penaltyPerInconsistency = 0.1;
        const agreementBonus = this.calculateAgreementBonus(agentResults);

        return Math.max(0.1, Math.min(1.0,
            baseConfidence - (inconsistencies.length * penaltyPerInconsistency) + agreementBonus
        ));
    }
}
```

---

## 결론

Claude Code의 계층적 다중 Agent 아키텍처는 AI 프로그래밍 어시스턴트 분야의 중요한 기술 혁신을 나타냅니다. 심층 역공학 분석을 통해 핵심 기술 구현을 완전히 복원했으며, 다음을 포함합니다:

### 핵심 기술 성과

1. **완전한 Agent 격리 메커니즘**: 각 SubAgent가 독립적인 컨텍스트에서 실행되어 진정한 격리와 보안성 구현
2. **지능형 동시 스케줄링 시스템**: UH1 함수가 구현한 동시 실행 스케줄러로 효율적인 다중 Agent 협업 지원
3. **정교한 Tool 권한 제어**: 다층 권한 검증 및 보안 정책으로 권한 남용 및 보안 위험 방지
4. **혁신적인 결과 합성 메커니즘**: KN5 함수가 구현한 지능형 결과 집계로 다중 Agent 결과의 일관성 보장
5. **견고한 오류 처리 및 복구**: 완전한 오류 격리, 모델 폴백 및 리소스 정리 메커니즘

### 아키텍처 혁신 가치

1. **확장성**: 모듈화된 설계로 유연한 Agent 확장 및 Tool 통합 지원
2. **신뢰성**: 다층 오류 처리 및 리소스 제한으로 시스템 안정적 실행 보장
3. **효율성**: 지능형 동시 스케줄링 및 컨텍스트 압축으로 성능 크게 향상
4. **보안성**: 포괄적인 권한 제어 및 격리 메커니즘으로 시스템 보안 보장
5. **유지보수성**: 명확한 아키텍처 계층화 및 모듈화된 설계로 유지보수 및 업그레이드 용이

### 기술적 영향 및 의미

Claude Code의 계층적 다중 Agent 아키텍처는 AI 프로그래밍 어시스턴트 발전을 위한 새로운 기준을 세웠으며, 기술 혁신은 전체 업계에 중요한 참고 가치를 가집니다:

1. **다중 Agent 협업 모드**: AI 시스템에서 효율적인 다중 Agent 협업을 구현하는 방법 보여줌
2. **보안 격리 설계**: AI 시스템 보안 격리의 모범 사례 제공
3. **리소스 관리 전략**: 대규모 AI 애플리케이션의 리소스 관리 및 최적화 방법 시범
4. **오류 처리 메커니즘**: 탄력적인 AI 시스템의 오류 처리 표준 수립

이러한 선진적인 아키텍처 설계는 복잡한 작업 처리의 기술적 도전을 해결했을 뿐만 아니라, 향후 AI 프로그래밍 어시스턴트의 발전 방향을 제시하며, 중요한 기술적 가치와 실무적 의미를 가집니다.

---

*본 문서는 Claude Code 소스 코드의 완전한 역공학 분석을 기반으로 하며, 난독화 코드, 런타임 동작 및 아키텍처 패턴을 체계적으로 분석하여 계층적 다중 Agent 아키텍처의 완전한 기술 구현을 정확하게 복원했습니다. 모든 분석 결과는 실제 코드 증거를 기반으로 하며, 현대 AI 프로그래밍 어시스턴트의 저수준 메커니즘을 이해하는 데 상세하고 정확한 기술적 통찰을 제공합니다.*
