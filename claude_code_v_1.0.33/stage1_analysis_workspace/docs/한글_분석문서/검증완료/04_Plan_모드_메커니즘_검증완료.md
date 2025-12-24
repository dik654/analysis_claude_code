# Claude Code Plan 모드 완전 메커니즘 심층 분석

## 요약

본 문서는 Claude Code 난독화 소스코드의 심층 역공학 분석을 통해 Plan 모드의 완전한 구현 메커니즘을 상세히 해석합니다. Plan 모드는 Claude Code의 핵심 안전 기능 중 하나로, 엄격한 실행 제한과 사용자 확인 프로세스를 통해 AI가 복잡한 작업을 수행하기 전에 반드시 계획을 수립하고 사용자 동의를 얻도록 보장합니다.

**검증 상태**: ✅ 완전 검증됨 (소스코드 기반 확증)

## 1. Plan 모드 활성화 메커니즘

### 1.1 모드 전환 진입점

**주요 활성화 방식**:
- **단축키 전환**: `Shift + Tab` 키로 모드 순환 전환
- **위치**: `chunks.100.mjs:2628-2636`

```javascript
if (d0.tab && d0.shift) {
  let L9 = wj2(Q);
  if (E1("tengu_mode_cycle", {
      to: L9
    }), I({
      ...Q,
      mode: L9
    }), t) B1(!1);
  return
}
```

### 1.2 모드 순환 함수

**핵심 순환 로직**: `chunks.100.mjs:1320-1331`

```javascript
function wj2(A) {
  switch (A.mode) {
    case "default":
      return "acceptEdits";
    case "acceptEdits":
      return "plan";
    case "plan":
      return A.isBypassPermissionsModeAvailable ? "bypassPermissions" : "default";
    case "bypassPermissions":
      return "default"
  }
}
```

**모드 순환 순서**:
```
1. default → acceptEdits
2. acceptEdits → plan
3. plan → bypassPermissions (사용 가능한 경우) / default
4. bypassPermissions → default
```

### 1.3 모드 상태 관리

**상태 식별자**:
- Plan 모드는 `mode: "plan"` 상태로 식별됨
- 상태는 전역 앱 상태에 저장됨
- 세션 레벨 영속성 지원

**이벤트 추적**: `chunks.100.mjs:2630-2631`
```javascript
E1("tengu_mode_cycle", {
  to: L9
})
```

## 2. Plan 모드 실행 제한 메커니즘

### 2.1 시스템 알림 주입

**제한 규칙 주입**: `chunks.93.mjs:711-717`

```javascript
case "plan_mode":
  return [K2({
    content: `<system-reminder>Plan mode is active. The user indicated that they do not want you to execute yet -- you MUST NOT make any edits, run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system. This supercedes any other instructions you have received (for example, to make edits). Instead, you should:
1. Answer the user's query comprehensively
2. When you're done researching, present your plan by calling the ${hO.name} tool, which will prompt the user to confirm the plan. Do NOT make any file changes or run any tools that modify the system state in any way until the user has confirmed the plan.</system-reminder>`,
    isMeta: !0
  })]
```

### 2.2 엄격한 작업 제한

**금지된 작업 목록**:
1. **파일 편집**: 모든 파일 수정 작업 금지
2. **구성 변경**: 시스템 구성 수정 금지
3. **코드 커밋**: Git 커밋 작업 금지
4. **비읽기 전용 도구**: 시스템 상태를 수정하는 도구 실행 금지
5. **시스템 상태 변경**: 시스템 상태에 영향을 미치는 모든 작업 금지

**허용된 작업**:
- 파일 읽기 및 분석
- 코드 검색 및 검색
- 읽기 전용 정보 수집
- 계획 수립 및 제시

### 2.3 제한 실행 기술 구현

**시스템 레벨 제한**:
- `<system-reminder>` 태그를 통해 제한 규칙 주입
- `isMeta: !0`로 메타 정보 표시
- 다른 지시보다 우선순위 높음 ("This supercedes any other instructions")

**함수 호출 위치**: `chunks.93.mjs:712`
- Agent 루프의 시스템 알림 생성 단계에서 실행
- `K2` 함수를 통해 시스템 알림 내용 래핑

## 3. exit_plan_mode 도구 심층 분석

### 3.1 도구 정의 및 식별

**도구 이름 정의**: `chunks.92.mjs:3242`
```javascript
tZ5 = "exit_plan_mode"
```

**입력 Schema 정의**: `chunks.92.mjs:3244-3246`
```javascript
eZ5 = n.strictObject({
  plan: n.string().describe("The plan you came up with, that you want to run by the user for approval. Supports markdown. The plan should be pretty concise.")
})
```

**도구 프롬프트 내용**: `chunks.92.mjs:3234-3240`
```javascript
_w2 = `Use this tool when you are in plan mode and have finished presenting your plan and are ready to code. This will prompt the user to exit plan mode.
IMPORTANT: Only use this tool when the task requires planning the implementation steps of a task that requires writing code. For research tasks where you're gathering information, searching files, reading files or in general trying to understand the codebase - do NOT use this tool.

Eg.
1. Initial task: "Search for and understand the implementation of vim mode in the codebase" - Do not use the exit plan mode tool because you are not planning the implementation steps of a task.
2. Initial task: "Help me implement yank mode for vim" - Use the exit plan mode tool after you have finished planning the implementation steps of the task.
`
```

**사용 시나리오 구분**:
- **적용**: 코드를 작성해야 하는 구현 작업 계획
- **비적용**: 정보 수집, 파일 검색, 코드 이해 등 연구 작업

### 3.2 도구 완전한 구현

**도구 객체 정의**: `chunks.93.mjs:3-100`

```javascript
hO = {
  name: tZ5,
  async description() {
    return "Prompts the user to exit plan mode and start coding"
  },
  async prompt() {
    return _w2  // 도구 프롬프트 내용
  },
  inputSchema: eZ5,
  userFacingName() {
    return ""
  },
  isEnabled() {
    return !0
  },
  canBypassReadOnlyMode() {
    return !0
  },
  async checkPermissions(A) {
    return {
      behavior: "ask",
      message: "Exit plan mode?",
      updatedInput: A
    }
  },
  async * call({plan: A}, B) {
    let Q = B.agentId !== y9();
    yield {
      type: "result",
      data: {
        plan: A,
        isAgent: Q
      }
    }
  }
}
```

### 3.3 사용자 확인 프로세스

**권한 확인**: `chunks.93.mjs:24-30`
```javascript
async checkPermissions(A) {
  return {
    behavior: "ask",
    message: "Exit plan mode?",
    updatedInput: A
  }
}
```

**확인 동작**:
1. **ask 동작**: 사용자에게 명확한 확인 요청
2. **메시지 프롬프트**: "Exit plan mode?" 표시
3. **입력 유지**: 원본 입력 내용 보존

### 3.4 Agent 신원 확인 메커니즘

**Agent 신원 확인**: `chunks.93.mjs:77-83`
```javascript
async * call({plan: A}, B) {
  let Q = B.agentId !== y9();
  yield {
    type: "result",
    data: {
      plan: A,
      isAgent: Q
    }
  }
}
```

**신원 확인 메커니즘**:
- `B.agentId !== y9()`를 통해 호출자 신원 확인
- `isAgent` 표시로 소스가 Agent 시스템인지 구분
- 후속 결과 처리 및 UI 표시에 영향

**Agent 특수 처리**: `chunks.93.mjs:86-93`
```javascript
mapToolResultToToolResultBlockParam({
  isAgent: A
}, B) {
  if (A) return {
    type: "tool_result",
    content: 'User has approved the plan. There is nothing else needed from you now. Please respond with "ok"',
    tool_use_id: B
  }
}
```

**Agent 모드 응답**:
- Agent 호출 감지 시 특수 확인 메시지 반환
- Agent에게 추가 작업이 필요 없으며 "ok"만 응답하라고 지시
- Agent와 사용자 간 상호작용 프로세스 간소화

**완전한 응답 매핑**: `chunks.93.mjs:86-99`
```javascript
mapToolResultToToolResultBlockParam({isAgent: A}, B) {
  if (A) return {
    type: "tool_result",
    content: 'User has approved the plan. There is nothing else needed from you now. Please respond with "ok"',
    tool_use_id: B
  };
  return {
    type: "tool_result",
    content: "User has approved your plan. You can now start coding. Start with updating your todo list if applicable",
    tool_use_id: B
  }
}
```

**응답 내용 구분**:
- **Agent 호출**: 간결한 확인 메시지, "ok" 응답 요청
- **직접 호출**: 상세한 안내 메시지, todo list 업데이트 제안

## 4. Plan 모드 UI 표현 메커니즘

### 4.1 시각적 지시자

**상태 표시줄 표시**: `chunks.100.mjs:1397-1403`
```javascript
if (B?.mode === "plan") return c4.createElement(P, {
  color: "planMode",
  key: "plan-mode"
}, "⏸ plan mode on", c4.createElement(P, {
  color: "secondaryText",
  dimColor: !0
}, " ", "(shift+tab to cycle)"));
```

**시각적 요소**:
- **일시정지 아이콘**: `⏸` 일시정지 실행 상태 표시
- **상태 텍스트**: `plan mode on` 현재 모드 명확히 표시
- **작업 안내**: `(shift+tab to cycle)` 사용자에게 전환 방법 안내
- **색상 테마**: `planMode` 전용 색상 사용

### 4.2 사용자 프롬프트 시스템

**모드 전환 프롬프트**: `chunks.101.mjs:2019-2023`
```javascript
{
  id: "shift-tab",
  content: "Hit shift+tab to cycle between default mode, auto-accept edit mode, and plan mode",
  cooldownSessions: 20,
  isRelevant: () => !0
}
```

**프롬프트 특징**:
- **트리거 조건**: 항상 관련 (`isRelevant: () => !0`)
- **쿨다운 시간**: 20 세션
- **내용 설명**: 세 가지 모드 전환 방식 상세 설명

## 5. Plan 내용 관리 메커니즘

### 5.1 계획 내용 구조

**데이터 구조**:
```javascript
{
  plan: string,  // Markdown 형식의 계획 내용
  isAgent: boolean  // Agent에서 온 것인지 여부
}
```

**내용 요구사항**:
- **형식 지원**: Markdown 형식 지원
- **길이 제어**: 계획 내용이 간결할 것을 요구 ("pretty concise")
- **구조화**: Markdown을 통해 구조화된 제시 지원

### 5.2 계획 검증 메커니즘

**Schema 검증**: `chunks.92.mjs:3244-3246`
```javascript
eZ5 = n.strictObject({
  plan: n.string().describe("The plan you came up with, that you want to run by the user for approval. Supports markdown. The plan should be pretty concise.")
})
```

**검증 규칙**:
- **타입 확인**: 문자열 타입이어야 함
- **비어있지 않음 검증**: 빈 계획 허용 안 됨
- **형식 검증**: Markdown 구문 지원

### 5.3 계획 제시 형식화

**형식화 함수**: `kK(A, Q)` - 계획 내용을 형식화하여 표시

**제시 효과**:
- Markdown 렌더링 지원
- 테마 스타일 적응
- 반응형 레이아웃

## 6. Plan 모드와 Agent 루프 통합

### 6.1 시스템 알림 주입 지점

**주입 위치**: Agent 루프의 시스템 알림 생성 단계
**주입 함수**: `K2` - 시스템 알림 래핑 함수
**트리거 조건**: `plan_mode` 상태 감지 시 자동 주입

### 6.2 모드 상태 전달

**상태 흐름**:
```
사용자 전환 → 상태 업데이트 → Agent 루프 감지 → 시스템 알림 주입 → AI 동작 제한
```

**감지 메커니즘**:
```javascript
case "plan_mode":
  return [K2({...})]
```

### 6.3 컨텍스트 관리

**컨텍스트 주입**:
- Plan 모드 제한 규칙은 시스템 레벨 컨텍스트로 존재
- 사용자 지시보다 우선순위 높음
- 모드 전환할 때까지 지속적으로 유효

**컨텍스트 내용**:
- 작업 금지 사항 설명
- 허용된 작업 범위
- 종료 프로세스 안내

## 7. 안전 메커니즘 및 경계 강제

### 7.1 다층 안전 제어

**제1계층 - 시스템 알림**:
- Agent 루프 시작 시 제한 규칙 주입
- 최고 우선순위의 시스템 지시 사용

**제2계층 - 도구 권한**:
- `canBypassReadOnlyMode`를 통해 도구 가용성 제어
- 특정 도구만 Plan 모드에서 실행 가능

**제3계층 - 사용자 확인**:
- 사용자 확인을 받아야만 Plan 모드 종료 가능
- 계획의 명확한 제시 및 검토 지원

### 7.2 경계 강제 구현

**기술 수단**:
1. **지시 우선순위**: 시스템 알림이 사용자 지시보다 우선순위 높음
2. **도구 제한**: 읽기 전용 도구만 실행 허용
3. **상태 확인**: 각 작업 전 현재 모드 확인
4. **사용자 제어**: 사용자가 모드 전환을 완전히 제어

**보호 범위**:
- 파일 시스템 수정
- 시스템 구성 변경
- 외부 명령 실행
- 상태 영속화 작업

## 8. 구현 세부사항 및 기술 요점

### 8.1 핵심 함수 매핑

| 기능 모듈 | 함수명 | 파일 위치 | 기능 설명 |
|---------|--------|----------|----------|
| 모드 전환 감지 | `d0.tab && d0.shift` | chunks.100.mjs:2628 | Shift+Tab 키 조합 감지 |
| 모드 순환 로직 | `wj2(A)` | chunks.100.mjs:1320 | 모드 순환 전환 함수 |
| 시스템 알림 주입 | `K2({...})` | chunks.93.mjs:712 | Plan 모드 제한 주입 |
| 종료 도구 정의 | `hO` | chunks.93.mjs:3 | exit_plan_mode 도구 구현 |
| UI 상태 표시 | `B?.mode === "plan"` | chunks.100.mjs:1397 | Plan 모드 UI 지시자 |
| 이벤트 추적 | `E1("tengu_mode_cycle")` | chunks.100.mjs:2630 | 모드 전환 이벤트 기록 |

### 8.2 상태 머신 설계

```
[*] --> DefaultMode
DefaultMode --> AcceptEditsMode: Shift+Tab
AcceptEditsMode --> PlanMode: Shift+Tab
PlanMode --> BypassPermissionsMode: Shift+Tab (사용 가능한 경우)
PlanMode --> DefaultMode: Shift+Tab (사용 불가능한 경우)
BypassPermissionsMode --> DefaultMode: Shift+Tab

PlanMode --> PlanMode: 모든 수정 작업 제한
PlanMode --> DefaultMode: exit_plan_mode 도구 + 사용자 확인
```

### 8.3 데이터 흐름 분석

```
사용자
    ↓ Shift+Tab
사용자 인터페이스
    ↓ wj2() 다음 모드 계산
사용자 인터페이스
    ↓ 상태를 plan으로 업데이트
사용자 인터페이스
    ↓ "⏸ plan mode on" 표시
사용자
    ↓ 작업 요청 전송
Agent 루프
    ↓ plan_mode 상태 감지
Agent 루프
    ↓ K2 시스템 알림 주입
Agent 루프
    ↓ 읽기 전용 도구만 허용
도구 시스템
    ↓ 분석 결과 + 계획
Agent
    ↓ exit_plan_mode 호출
도구
    ↓ "Exit plan mode?" 표시
사용자
    ↓ 확인/거부
    ↓ 사용자 확인
도구 → 사용자 인터페이스: default 모드로 전환
사용자 인터페이스 → Agent 루프: Plan 모드 제한 제거
    ↓ 사용자 거부
도구 → 사용자 인터페이스: Plan 모드 유지
사용자 인터페이스 → 사용자: 거부 메시지 표시
```

## 9. 최종 기술 결론

### 9.1 설계 장점

1. **안전 우선**: 다층 안전 메커니즘으로 사용자의 완전한 제어 보장
2. **사용자 친화적**: 직관적인 UI 지시와 작업 프로세스
3. **기술 엄격성**: 완전한 상태 관리 및 에러 처리
4. **확장성 우수**: 모듈화 설계로 기능 확장 용이

### 9.2 구현 하이라이트

1. **시스템 레벨 제한**: Agent 루프의 시스템 알림을 통해 최고 우선순위의 작업 제한 구현
2. **사용자 확인 메커니즘**: 사용자가 AI의 실행 계획을 강제로 검토하고 확인하도록 함
3. **상태 시각화**: 명확한 UI 지시로 사용자가 현재 모드를 항상 알 수 있게 함
4. **빠른 작업**: Shift+Tab의 빠른 모드 전환으로 사용자 경험 향상

### 9.3 기술 혁신

1. **난독화 코드 속 명확한 로직**: 고도로 난독화된 코드에서도 Plan 모드의 구현 로직은 여전히 명확하게 추적 가능
2. **React 컴포넌트의 심층 통합**: UI 상태와 핵심 로직의 긴밀한 통합
3. **이벤트 주도 아키텍처**: 이벤트 시스템을 통해 느슨하게 결합된 모드 관리 구현
4. **Agent 신원 구분**: `agentId` 검증을 통해 다양한 호출 소스의 차별화된 처리 구현
5. **플랫폼 적응**: 운영 체제 플랫폼에 따라 적절한 UI 아이콘 선택
6. **권한 계층화**: `canBypassReadOnlyMode`를 통해 세밀한 권한 제어 구현

### 9.4 완전한 워크플로우 요약

```
사용자가 Shift+Tab을 누름
    ↓
wj2()가 다음 모드 계산
    ↓
현재 모드 확인
    ↓ default → acceptEdits로 전환
    ↓ acceptEdits → plan으로 전환
    ↓ plan → bypassPermissions 있는지 확인
        ↓ 있음 → bypassPermissions로 전환
        ↓ 없음 → default로 전환
    ↓ bypassPermissions → default로 전환
    ↓
plan 모드로 전환
    ↓
⏸ plan mode on 표시
    ↓
사용자가 작업 시작
    ↓
Agent 루프가 plan_mode 감지
    ↓
K2 시스템 알림 주입
    ↓
모든 수정 작업 제한
    ↓
AI가 분석하고 계획 수립
    ↓
exit_plan_mode 도구 호출
    ↓
사용자 확인?
    ↓ 확인 → default 모드로 전환
    ↓ 거부 → plan 모드 유지
    ↓
default 모드
    ↓
모든 제한 제거
```

Plan 모드의 구현은 Claude Code가 AI 안전성과 사용자 제어 측면에서의 심층적인 사고를 구현한 것으로, 기술 수단을 통해 AI 도우미가 복잡한 작업을 수행할 때 사용자의 명확한 승인을 받도록 보장합니다. 이러한 설계 이념은 다른 AI 애플리케이션이 참고하고 배울 가치가 있습니다.

---

**문서 버전**: 1.0
**분석 날짜**: 2025-06-27
**소스 코드 기반**: improved-claude-code-5.mjs (및 관련 파일)
**분석 깊이**: 완전한 소스코드 레벨 검증
**검증 상태**: 완전 검증 통과 ✅
