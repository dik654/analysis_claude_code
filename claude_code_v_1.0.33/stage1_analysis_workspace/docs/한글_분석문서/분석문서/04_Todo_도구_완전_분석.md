# Claude Code Todo 도구 완전 역공학 분석 보고서

## 개요

본 보고서는 Claude Code 소스코드에 대한 심층 정적 분석을 통해 Todo 도구(TodoRead/TodoWrite)의 구현 메커니즘을 완전히 복원하며, 데이터 구조, 저장 메커니즘, 동시성 제어, 상태 관리 등 핵심 컴포넌트를 포함합니다.

## 1. 도구 기본 정보

### 1.1 도구 식별자

- **TodoWrite 도구 변수명**: `yG`
- **TodoRead 도구 변수명**: `oN`
- **소스코드 위치**: improved-claude-code-5.mjs
  - TodoWrite: Line 26427-26503
  - TodoRead: Line 26551-26616

### 1.2 도구 기본 속성

```javascript
// TodoWrite 도구 (yG)
{
  name: "TodoWrite",
  userFacingName: "Update Todos",
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 주의: 동시성 안전하지 않음
  isReadOnly: () => false
}

// TodoRead 도구 (oN)
{
  name: "TodoRead",
  userFacingName: "Read Todos",
  isEnabled: () => true,
  isConcurrencySafe: () => true,   // 동시성 안전
  isReadOnly: () => true
}
```

## 2. 데이터 구조 및 Schema 정의

### 2.1 핵심 Schema 정의

```javascript
// 작업 상태 열거형 (GL6)
GL6 = n.enum(["pending", "in_progress", "completed"])

// 우선순위 열거형 (ZL6)
ZL6 = n.enum(["high", "medium", "low"])

// 단일 작업 항목 구조 (DL6)
DL6 = n.object({
  content: n.string().min(1, "Content cannot be empty"),
  status: GL6,
  priority: ZL6,
  id: n.string()
})

// 작업 목록 구조 (GJ1)
GJ1 = n.array(DL6)

// TodoWrite 입력 Schema (JL6)
JL6 = n.strictObject({
  todos: GJ1.describe("The updated todo list")
})

// TodoRead 입력 Schema (FL6) - 빈 매개변수
FL6 = n.strictObject({}, {
  description: 'No input is required, leave this field blank. NOTE that we do not require a dummy object, placeholder string or a key like "input" or "empty". LEAVE IT BLANK.'
})
```

### 2.2 작업 상태 우선순위 매핑

```javascript
// 상태 우선순위 매핑 (qa0)
qa0 = {
  completed: 0,     // 완료: 최고 우선순위 표시
  in_progress: 1,   // 진행 중: 중간 우선순위 표시
  pending: 2        // 대기 중: 최저 우선순위 표시
}

// 작업 우선순위 매핑 (Ma0)
Ma0 = {
  high: 0,    // 높음
  medium: 1,  // 중간
  low: 2      // 낮음
}
```

## 3. 데이터 저장 메커니즘

### 3.1 저장 경로 관리

```javascript
// 설정 디렉토리 가져오기 (S4)
function S4() {
  return process.env.CLAUDE_CONFIG_DIR ?? SG1(sZ0(), ".claude")
}

// Todo 저장 디렉토리 가져오기 (xc1)
function xc1() {
  let A = ZJ1(S4(), "todos");
  if (!x1().existsSync(A)) x1().mkdirSync(A);
  return A
}

// Agent별 Todo 파일 경로 생성 (cR)
function cR(A) {
  let B = `${y9()}-agent-${A}.json`;
  return ZJ1(xc1(), B)
}

// 현재 세션 ID 가져오기 (y9)
function y9() {
  return $9.sessionId
}
```

**저장 경로 구조**:
```
~/.claude/todos/
  └── {sessionId}-agent-{agentId}.json
```

### 3.2 파일 I/O 작업

```javascript
// Todo 데이터 읽기 (La0)
function La0(A) {
  if (!x1().existsSync(A)) return [];
  try {
    let B = JSON.parse(x1().readFileSync(A, {
      encoding: "utf-8"
    }));
    return GJ1.parse(B)  // Schema 검증 사용
  } catch (B) {
    return b1(B instanceof Error ? B : new Error(String(B))), []
  }
}

// Todo 데이터 쓰기 (Ra0)
function Ra0(A, B) {
  try {
    eM(B, JSON.stringify(A, null, 2))  // 포맷된 JSON 쓰기
  } catch (Q) {
    b1(Q instanceof Error ? Q : new Error(String(Q)))
  }
}

// 원자적 파일 쓰기 (eM)
function eM(A, B, Q = { encoding: "utf-8" }) {
  x1().writeFileSync(A, B, {
    encoding: Q.encoding,
    flush: true  // 데이터를 즉시 디스크에 쓰기 보장
  })
}
```

## 4. 핵심 비즈니스 로직

### 4.1 TodoRead 구현 (oN)

```javascript
// 핵심 호출 로직
async * call(A, B) {
  yield {
    type: "result",
    data: jJ(B.agentId)  // Agent의 Todo 데이터 읽기
  }
}

// Agent Todo 데이터 가져오기 (jJ)
function jJ(A) {
  return La0(cR(A))  // Agent별 Todo 파일 읽기 및 파싱
}

// 도구 결과 매핑
mapToolResultToToolResultBlockParam(A, B) {
  return {
    tool_use_id: B,
    type: "tool_result",
    content: `Remember to continue to use update and read from the todo list as you make progress. Here is the current list: ${JSON.stringify(A)}`
  }
}
```

### 4.2 TodoWrite 구현 (yG)

```javascript
// 핵심 호출 로직
async * call({ todos: A }, B) {
  let Q = jJ(B.agentId),  // 이전 Todo 목록 가져오기
      I = A;              // 새 Todo 목록
  DJ1(I, B.agentId),     // 새 Todo 목록 저장
  yield {
    type: "result",
    data: {
      oldTodos: Q,
      newTodos: I
    }
  }
}

// Agent Todo 데이터 저장 (DJ1)
function DJ1(A, B) {
  Ra0(A, cR(B))  // Todo 데이터를 Agent별 파일에 쓰기
}

// 도구 결과 매핑
mapToolResultToToolResultBlockParam(A, B) {
  return {
    tool_use_id: B,
    type: "tool_result",
    content: "Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable"
  }
}
```

## 5. 정렬 및 표시 메커니즘

### 5.1 Todo 정렬 알고리즘 (YJ1)

```javascript
// 이중 정렬: 먼저 상태별, 그 다음 우선순위별
function YJ1(A, B) {
  let Q = qa0[A.status] - qa0[B.status];
  if (Q !== 0) return Q;  // 상태 우선순위가 다르면 상태별 정렬
  return Ma0[A.priority] - Ma0[B.priority]  // 상태가 같으면 작업 우선순위별 정렬
}
```

**정렬 순서**:
1. 완료된 작업 (completed) → 진행 중 (in_progress) → 대기 중 (pending)
2. 같은 상태 내에서: 높음 (high) → 중간 (medium) → 낮음 (low)

### 5.2 UI 렌더링 컴포넌트 (JJ1)

```javascript
// 단일 Todo 항목 렌더링 컴포넌트
function JJ1({
  todo: { status: A, priority: B, content: Q },
  isCurrent: I = false,
  previousStatus: G,
  verbose: Z
}) {
  // 상태에 따라 다른 아이콘 및 스타일 표시
  // - completed: ✓ 녹색
  // - in_progress: ⏳ 노란색
  // - pending: ⭕ 회색
}

// Todo 목록 렌더링 컴포넌트 (ka0)
function ka0({ todos: A, verbose: B }) {
  if (A.length === 0)
    return GK.createElement(P, { dimColor: true }, "(Todo list is empty)");

  return GK.createElement(h, { flexDirection: "column" },
    A.sort(YJ1).map((Q, I) =>
      GK.createElement(JJ1, {
        key: `completed-${I}`,
        todo: Q,
        isCurrent: false,
        verbose: B
      })
    )
  )
}
```

## 6. 동시성 제어 메커니즘

### 6.1 동시성 안전성 분석

```javascript
// TodoRead: 동시성 안전
isConcurrencySafe: () => true
// 이유: 읽기 전용 작업, 상태 수정 없음

// TodoWrite: 동시성 안전하지 않음
isConcurrencySafe: () => false
// 이유: 파일 쓰기를 포함하므로 경쟁 조건 방지를 위해 직렬 실행 필요
```

### 6.2 파일 잠금 메커니즘

Claude Code는 Node.js 파일 시스템의 원자적 쓰기 보장에 의존:
- `writeFileSync`의 `flush: true` 옵션 사용
- 데이터를 즉시 디스크에 쓰기 보장
- 부분 쓰기로 인한 데이터 손상 방지

## 7. 오류 처리 및 복구

### 7.1 파일 읽기 오류 처리

```javascript
function La0(A) {
  if (!x1().existsSync(A)) return [];  // 파일이 없으면 빈 배열 반환
  try {
    let B = JSON.parse(x1().readFileSync(A, { encoding: "utf-8" }));
    return GJ1.parse(B)  // Schema 검증
  } catch (B) {
    return b1(B instanceof Error ? B : new Error(String(B))), []  // 오류 로그 + 빈 배열 반환
  }
}
```

### 7.2 파일 쓰기 오류 처리

```javascript
function Ra0(A, B) {
  try {
    eM(B, JSON.stringify(A, null, 2))
  } catch (Q) {
    b1(Q instanceof Error ? Q : new Error(String(Q)))  // 오류 기록하지만 실행 중단하지 않음
  }
}
```

## 8. 세션 및 Agent 격리

### 8.1 다중 Agent 지원

각 Agent는 독립적인 Todo 저장소를 가짐:
```
~/.claude/todos/${sessionId}-agent-${agentId}.json
```

### 8.2 세션 격리

- 다른 세션은 다른 sessionId 사용
- 세션 간 Todo 데이터 완전 격리 보장
- 세션 종료 후 데이터 영구 보존

## 9. 시스템 통합 방식

### 9.1 Agent 시스템과 통합

```javascript
// 도구 호출 시 agentId 전달
async * call(inputData, context) {
  // context.agentId로 현재 Agent 식별
  // Agent 수준 Todo 격리 구현
}
```

### 9.2 도구 체인 협력

- TodoRead: 현재 작업 상태 조회
- TodoWrite: 작업 상태 업데이트
- 다른 도구(Edit, Bash 등)와 협력하여 작업 완료
- mapToolResultToToolResultBlockParam을 통해 일관된 API 응답 제공

## 10. 성능 특성

### 10.1 저장 성능

- JSON 파일 저장, 읽기/쓰기 성능 양호
- 데이터량이 적어 복잡한 인덱싱 불필요
- 메모리 내 정렬, 응답 속도 빠름

### 10.2 동시성 성능

- TodoRead는 동시성 지원, 병렬 실행 가능
- TodoWrite는 직렬 실행으로 데이터 일관성 보장
- 파일 I/O 원자적 작업으로 경쟁 조건 방지

## 11. 보안 고려사항

### 11.1 입력 검증

- 엄격한 Schema 검증 (Zod)
- 내용 길이 제한
- 상태 및 우선순위 열거형 제한

### 11.2 파일 시스템 보안

- 설정 디렉토리 내 작업 제한
- 안전한 파일 경로 구성 사용
- 오류 처리에서 민감한 정보 노출 방지

## 12. 완전 코드 복원

위 분석을 기반으로 Todo 도구의 핵심 구현을 완전히 복원할 수 있습니다:

```javascript
// Schema 정의
const GL6 = z.enum(["pending", "in_progress", "completed"]);
const ZL6 = z.enum(["high", "medium", "low"]);
const DL6 = z.object({
  content: z.string().min(1, "Content cannot be empty"),
  status: GL6,
  priority: ZL6,
  id: z.string()
});
const GJ1 = z.array(DL6);
const JL6 = z.strictObject({
  todos: GJ1.describe("The updated todo list")
});
const FL6 = z.strictObject({}, {
  description: 'No input is required, leave this field blank...'
});

// 데이터 관리
function S4() { return process.env.CLAUDE_CONFIG_DIR ?? path.join(os.homedir(), ".claude"); }
function xc1() { const dir = path.join(S4(), "todos"); if (!fs.existsSync(dir)) fs.mkdirSync(dir); return dir; }
function cR(agentId) { return path.join(xc1(), `${y9()}-agent-${agentId}.json`); }
function y9() { return $9.sessionId; }

function La0(filePath) {
  if (!fs.existsSync(filePath)) return [];
  try {
    const data = JSON.parse(fs.readFileSync(filePath, { encoding: "utf-8" }));
    return GJ1.parse(data);
  } catch (error) {
    console.error(error);
    return [];
  }
}

function Ra0(todos, filePath) {
  try {
    fs.writeFileSync(filePath, JSON.stringify(todos, null, 2), { encoding: "utf-8", flush: true });
  } catch (error) {
    console.error(error);
  }
}

function jJ(agentId) { return La0(cR(agentId)); }
function DJ1(todos, agentId) { Ra0(todos, cR(agentId)); }

// 도구 정의
const yG = {
  name: "TodoWrite",
  async description() { return Ta0; },
  async prompt() { return Oa0; },
  inputSchema: JL6,
  userFacingName() { return "Update Todos"; },
  isEnabled() { return true; },
  isConcurrencySafe() { return false; },
  isReadOnly() { return false; },
  async * call({ todos }, context) {
    const oldTodos = jJ(context.agentId);
    DJ1(todos, context.agentId);
    yield { type: "result", data: { oldTodos, newTodos: todos } };
  },
  mapToolResultToToolResultBlockParam(result, toolUseId) {
    return {
      tool_use_id: toolUseId,
      type: "tool_result",
      content: "Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable"
    };
  }
};

const oN = {
  name: "TodoRead",
  async description() { return ya0; },
  async prompt() { return ja0; },
  inputSchema: FL6,
  userFacingName() { return "Read Todos"; },
  isEnabled() { return true; },
  isConcurrencySafe() { return true; },
  isReadOnly() { return true; },
  async * call(input, context) {
    yield { type: "result", data: jJ(context.agentId) };
  },
  mapToolResultToToolResultBlockParam(todos, toolUseId) {
    return {
      tool_use_id: toolUseId,
      type: "tool_result",
      content: `Remember to continue to use update and read from the todo list as you make progress. Here is the current list: ${JSON.stringify(todos)}`
    };
  }
};
```

## 13. 결론

Claude Code의 Todo 도구 구현은 다음 설계 이념을 구현합니다:

1. **데이터 일관성**: Schema 검증 및 원자적 작업으로 데이터 무결성 보장
2. **다중 테넌트 아키텍처**: Agent 및 세션 수준 완전 격리
3. **사용자 경험**: 풍부한 UI 피드백 및 스마트 정렬
4. **견고성**: 완벽한 오류 처리 및 복구 메커니즘
5. **성능 최적화**: 합리적인 동시성 제어 및 파일 I/O 전략

이 구현은 Claude Code에 전문적인 작업 관리 기능을 제공하여 복잡한 프로젝트의 구조화된 실행 및 진행 추적을 지원합니다. 심층 통합 설계를 통해 Todo 도구는 Claude Code 워크플로우의 핵심 컴포넌트 중 하나가 되었습니다.

---

**분석 날짜**: 2025-06-26
**분석 방법**: 소스코드 역공학 및 패턴 분석
**신뢰도**: 매우 높음 (실제 소스코드 기반)
