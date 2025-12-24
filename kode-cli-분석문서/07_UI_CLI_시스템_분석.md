# Kode-CLI UI 및 CLI 시스템 분석

## 목차
1. [개요](#1-개요)
2. [React/Ink 기반 터미널 UI 시스템](#2-reactink-기반-터미널-ui-시스템)
3. [주요 UI 컴포넌트](#3-주요-ui-컴포넌트)
4. [React Hooks 시스템](#4-react-hooks-시스템)
5. [화면(Screen) 구성과 전환](#5-화면screen-구성과-전환)
6. [명령어 시스템](#6-명령어-시스템)
7. [커스텀 명령어 시스템](#7-커스텀-명령어-시스템)
8. [알림 시스템](#8-알림-시스템)
9. [컴포넌트 구조도](#9-컴포넌트-구조도)

---

## 1. 개요

Kode-CLI는 **React와 Ink**를 사용하여 터미널에서 실행되는 고급 대화형 UI를 제공합니다. Ink는 React의 컴포넌트 기반 아키텍처를 터미널 환경에 적용한 라이브러리로, 선언적 UI 프로그래밍을 터미널에서 가능하게 합니다.

### 핵심 기술 스택
- **React**: 컴포넌트 기반 UI 로직
- **Ink**: 터미널용 React 렌더러
- **TypeScript**: 타입 안전성
- **Zod**: 런타임 검증

---

## 2. React/Ink 기반 터미널 UI 시스템

### 2.1 Ink의 역할

Ink는 React 컴포넌트를 터미널 출력으로 렌더링합니다:

```typescript
// 기본 렌더링 패턴
import { render } from 'ink'
import { REPL } from './REPL'

render(
  <REPL
    commands={commands}
    tools={tools}
    initialPrompt={prompt}
    // ... props
  />,
  {
    exitOnCtrlC: false,  // Ctrl+C 동작 커스터마이징
  }
)
```

### 2.2 Ink 컴포넌트 기본 요소

```typescript
import { Box, Text, Newline } from 'ink'

// Box: 레이아웃 컨테이너 (flexbox 기반)
<Box flexDirection="column" gap={1} paddingX={2}>
  {/* 컨텐츠 */}
</Box>

// Text: 텍스트 렌더링 (색상, 스타일 지원)
<Text color="blue" bold dimColor>
  텍스트 내용
</Text>

// Newline: 줄바꿈
<Newline />
```

### 2.3 정적(Static) vs 동적(Transient) 렌더링

Kode-CLI는 성능 최적화를 위해 **Static**과 **Transient** 두 가지 렌더링 방식을 사용합니다:

```typescript
// REPL.tsx - 렌더링 전략
const messagesJSX = useMemo(() => {
  return [
    {
      type: 'static',  // 한 번 렌더링되면 고정
      jsx: (
        <Box flexDirection="column" key={`logo${forkNumber}`}>
          <Logo mcpClients={mcpClients} />
          <ProjectOnboarding workspaceDir={getOriginalCwd()} />
        </Box>
      ),
    },
    ...reorderMessages(normalizedMessages).map(_ => {
      const type = shouldRenderStatically(_, normalizedMessages, unresolvedToolUseIDs)
        ? 'static'
        : 'transient'

      return {
        type,
        jsx: <Message message={_} /* ... */ />
      }
    }),
  ]
}, [messages, tools, verbose])

// Static 컴포넌트: 한 번만 렌더링
<Static
  items={messagesJSX.filter(_ => _.type === 'static')}
  children={(item: any) => item.jsx}
/>

// Transient 컴포넌트: 매 렌더링마다 업데이트
{messagesJSX.filter(_ => _.type === 'transient').map(_ => _.jsx)}
```

**Static 렌더링 조건:**
- 툴 사용이 완료된 메시지
- 더 이상 변경되지 않을 메시지
- 로고, 프로젝트 온보딩 등 고정 컨텐츠

**Transient 렌더링 조건:**
- 진행 중인 툴 사용
- 애니메이션이 필요한 메시지
- 사용자 입력 중인 컴포넌트

---

## 3. 주요 UI 컴포넌트

### 3.1 PromptInput 컴포넌트

사용자 입력을 받는 핵심 컴포넌트입니다.

**파일 위치:** `/src/components/PromptInput.tsx`

#### 주요 기능

1. **다중 모드 입력**
```typescript
// 3가지 입력 모드
type InputMode = 'prompt' | 'bash' | 'koding'

// 모드 전환
if (value.startsWith('!')) {
  onModeChange('bash')    // Bash 명령어 모드
}
if (value.startsWith('#')) {
  onModeChange('koding')  // Koding 메모 모드
}
```

2. **통합 자동완성 시스템**
```typescript
const {
  suggestions,
  selectedIndex,
  isActive: completionActive,
  emptyDirMessage,
} = useUnifiedCompletion({
  input,
  cursorOffset,
  onInputChange,
  setCursorOffset,
  commands,
  onSubmit,
})
```

3. **특수 키 바인딩**
```typescript
// Ctrl+M: 모델 전환
if (key.ctrl && inputChar === 'm') {
  handleQuickModelSwitch()
}

// Ctrl+G: 외부 에디터 열기
if (key.ctrl && inputChar === 'g') {
  void handleExternalEdit()
}

// Shift+Enter: 줄바꿈
if (key.return && key.shift) {
  insertNewlineAtCursor()
}
```

4. **이미지 붙여넣기 지원**
```typescript
function onImagePaste(image: string) {
  onModeChange('prompt')
  setPastedImage(image)
}
```

#### UI 구조

```typescript
<Box flexDirection="column">
  {/* 모델 정보 표시 */}
  {modelInfo && (
    <Box justifyContent="flex-end" marginBottom={1}>
      <Text dimColor>
        [{modelInfo.provider}] {modelInfo.name}:
        {Math.round(modelInfo.currentTokens / 1000)}k /
        {Math.round(modelInfo.contextLength / 1000)}k
      </Text>
    </Box>
  )}

  {/* 입력 박스 */}
  <Box
    borderColor={mode === 'bash' ? theme.bashBorder : theme.secondaryBorder}
    borderStyle="round"
  >
    <Box width={3}>
      {mode === 'bash' ? (
        <Text color={theme.bashBorder}>&nbsp;!&nbsp;</Text>
      ) : mode === 'koding' ? (
        <Text color={theme.noting}>&nbsp;#&nbsp;</Text>
      ) : (
        <Text>&nbsp;&gt;&nbsp;</Text>
      )}
    </Box>
    <TextInput
      multiline
      value={input}
      onChange={onChange}
      onSubmit={onSubmit}
      // ... props
    />
  </Box>

  {/* 자동완성 제안 */}
  {suggestions.length > 0 && (
    <Box flexDirection="column">
      {suggestions.map((suggestion, index) => (
        <Text key={index} color={index === selectedIndex ? theme.suggestion : undefined}>
          {index === selectedIndex ? '◆ ' : '  '}
          {suggestion.displayValue}
        </Text>
      ))}
    </Box>
  )}
</Box>
```

### 3.2 Message 컴포넌트

메시지를 렌더링하는 컴포넌트입니다.

**파일 위치:** `/src/components/Message.tsx`

#### 구조

```typescript
export function Message({
  message,
  messages,
  tools,
  verbose,
  erroredToolUseIDs,
  inProgressToolUseIDs,
  unresolvedToolUseIDs,
  shouldAnimate,
  shouldShowDot,
}: Props): React.ReactNode {
  if (message.type === 'assistant') {
    return (
      <Box flexDirection="column">
        {message.message.content.map((block, index) => (
          <AssistantMessage
            key={index}
            param={block}
            costUSD={message.costUSD}
            durationMs={message.durationMs}
            tools={tools}
            // ... props
          />
        ))}
      </Box>
    )
  }

  // User message
  return (
    <Box flexDirection="column">
      {content.map((block, index) => (
        <UserMessage
          key={index}
          message={message}
          param={block}
          tools={tools}
        />
      ))}
    </Box>
  )
}
```

#### 메시지 타입별 렌더링

1. **Assistant 메시지**
   - `AssistantTextMessage`: 일반 텍스트
   - `AssistantToolUseMessage`: 툴 사용
   - `AssistantThinkingMessage`: 사고 과정
   - `AssistantRedactedThinkingMessage`: 축약된 사고

2. **User 메시지**
   - `UserTextMessage`: 사용자 텍스트
   - `UserToolResultMessage`: 툴 실행 결과

### 3.3 MessageResponse 컴포넌트

툴 응답을 시각적으로 구분하는 간단한 래퍼입니다.

```typescript
export function MessageResponse({ children }: Props): React.ReactNode {
  return (
    <Box flexDirection="row" height={1} overflow="hidden">
      <Text>{'  '}⎿ &nbsp;</Text>
      {children}
    </Box>
  )
}
```

**사용 예:**
```
  ⎿  Running bash command: ls -la
```

### 3.4 Logo 컴포넌트

애플리케이션 시작 시 표시되는 로고와 정보를 담당합니다.

**파일 위치:** `/src/components/Logo.tsx`

```typescript
export function Logo({
  mcpClients,
  isDefaultModel,
  updateBannerVersion,
  updateBannerCommands,
}: Props): React.ReactNode {
  return (
    <Box flexDirection="column">
      <Box
        borderColor={theme.kode}
        borderStyle="round"
        flexDirection="column"
      >
        {/* 업데이트 배너 */}
        {updateBannerVersion && (
          <Text color="yellow">
            New version available: {updateBannerVersion}
          </Text>
        )}

        {/* 환영 메시지 */}
        <Text>
          <Text color={theme.kode}>✻</Text> Welcome to{' '}
          <Text bold>{PRODUCT_NAME}</Text>
        </Text>

        {/* 현재 디렉토리 */}
        <Text color={theme.secondaryText}>
          cwd: {getCwd()}
        </Text>

        {/* MCP 서버 상태 */}
        {mcpClients.map((client, idx) => (
          <Box key={idx}>
            <Text>• {client.name}</Text>
            <Text color={client.type === 'connected' ? theme.success : theme.error}>
              {client.type === 'connected' ? 'connected' : 'failed'}
            </Text>
          </Box>
        ))}
      </Box>
    </Box>
  )
}
```

### 3.5 PermissionRequest 컴포넌트

툴 사용 권한을 요청하는 컴포넌트입니다.

**파일 위치:** `/src/components/permissions/PermissionRequest.tsx`

```typescript
export function PermissionRequest({
  toolUseConfirm,
  onDone,
  verbose,
}: Props): React.ReactNode {
  // Ctrl+C로 취소
  useInput((input, key) => {
    if (key.ctrl && input === 'c') {
      onDone()
      toolUseConfirm.onReject()
    }
  })

  const PermissionComponent = permissionComponentForTool(toolUseConfirm.tool)

  return (
    <PermissionComponent
      toolUseConfirm={toolUseConfirm}
      onDone={onDone}
      verbose={verbose}
    />
  )
}
```

**툴별 권한 요청 컴포넌트:**
- `FileEditPermissionRequest`: 파일 편집
- `FileWritePermissionRequest`: 파일 쓰기
- `BashPermissionRequest`: Bash 명령어
- `FilesystemPermissionRequest`: 파일시스템 접근
- `FallbackPermissionRequest`: 기본 권한 요청

---

## 4. React Hooks 시스템

### 4.1 터미널 관련 Hooks

#### useTerminalSize

터미널 크기를 추적하는 Hook입니다.

**파일 위치:** `/src/hooks/useTerminalSize.ts`

```typescript
// 전역 상태로 공유
let globalSize = {
  columns: process.stdout.columns || 80,
  rows: process.stdout.rows || 24,
}

const listeners = new Set<() => void>()

export function useTerminalSize() {
  const [size, setSize] = useState(globalSize)

  useEffect(() => {
    const updateSize = () => setSize({ ...globalSize })
    listeners.add(updateSize)

    // 첫 번째 Hook만 이벤트 리스너 등록
    if (!isListenerAttached) {
      process.stdout.on('resize', updateAllListeners)
      isListenerAttached = true
    }

    return () => {
      listeners.delete(updateSize)
      if (listeners.size === 0) {
        process.stdout.off('resize', updateAllListeners)
        isListenerAttached = false
      }
    }
  }, [])

  return size
}
```

**사용 예:**
```typescript
const { columns, rows } = useTerminalSize()
const maxWidth = columns - 10  // 여백 고려
```

#### useTextInput

고급 텍스트 입력 처리를 위한 Hook입니다.

**파일 위치:** `/src/hooks/useTextInput.ts`

**주요 기능:**
1. **커서 관리**: 멀티라인 텍스트에서 커서 위치 추적
2. **키보드 단축키**: Emacs 스타일 키 바인딩 (Ctrl+A, Ctrl+E 등)
3. **이미지 붙여넣기**: 클립보드에서 이미지 감지 및 처리
4. **히스토리 탐색**: 위/아래 화살표로 명령어 히스토리

```typescript
export function useTextInput({
  value,
  onChange,
  onSubmit,
  onHistoryUp,
  onHistoryDown,
  cursorChar,
  columns,
}: Props): UseTextInputResult {
  const cursor = Cursor.fromText(value, columns, offset)

  const handleCtrl = mapInput([
    ['a', () => cursor.startOfLine()],      // Ctrl+A: 줄 시작
    ['e', () => cursor.endOfLine()],        // Ctrl+E: 줄 끝
    ['b', () => cursor.left()],             // Ctrl+B: 왼쪽
    ['f', () => cursor.right()],            // Ctrl+F: 오른쪽
    ['k', () => cursor.deleteToLineEnd()],  // Ctrl+K: 줄 끝까지 삭제
    ['u', () => cursor.deleteToLineStart()],// Ctrl+U: 줄 시작까지 삭제
    ['w', () => cursor.deleteWordBefore()], // Ctrl+W: 단어 삭제
    ['v', tryImagePaste],                   // Ctrl+V: 붙여넣기
  ])

  function onInput(input: string, key: Key): void {
    // 백스페이스 처리
    if (key.backspace || key.delete) {
      const nextCursor = cursor.backspace()
      setOffset(nextCursor.offset)
      onChange(nextCursor.text)
      return
    }

    // 키 매핑 적용
    const nextCursor = mapKey(key)(input)
    if (nextCursor) {
      setOffset(nextCursor.offset)
      onChange(nextCursor.text)
    }
  }

  return {
    onInput,
    renderedValue: cursor.render(cursorChar, mask, invert),
    offset,
    setOffset,
  }
}
```

### 4.2 제어 관련 Hooks

#### useCancelRequest

요청 취소를 처리하는 Hook입니다.

**파일 위치:** `/src/hooks/useCancelRequest.ts`

```typescript
export function useCancelRequest(
  setToolJSX: SetToolJSXFn,
  setToolUseConfirm: (toolUseConfirm: ToolUseConfirm | null) => void,
  setBinaryFeedbackContext: (context: BinaryFeedbackContext | null) => void,
  onCancel: () => void,
  isLoading: boolean,
  isMessageSelectorVisible: boolean,
  abortSignal?: AbortSignal,
) {
  useInput((_, key) => {
    if (!key.escape) return
    if (abortSignal?.aborted) return
    if (!isLoading) return
    if (isMessageSelectorVisible) return

    // 모든 UI 상태 초기화
    setToolJSX(null)
    setToolUseConfirm(null)
    setBinaryFeedbackContext(null)
    onCancel()
  })
}
```

#### useExitOnCtrlCD

Ctrl+C/D로 종료하는 Hook입니다 (더블 프레스 패턴).

**파일 위치:** `/src/hooks/useExitOnCtrlCD.ts`

```typescript
export function useExitOnCtrlCD(onExit: () => void): ExitState {
  const [exitState, setExitState] = useState({
    pending: false,
    keyName: null,
  })

  const handleCtrlC = useDoublePress(
    pending => setExitState({ pending, keyName: 'Ctrl-C' }),
    onExit,
  )

  const handleCtrlD = useDoublePress(
    pending => setExitState({ pending, keyName: 'Ctrl-D' }),
    onExit,
  )

  useInput((input, key) => {
    if (key.ctrl && input === 'c') handleCtrlC()
    if (key.ctrl && input === 'd') handleCtrlD()
  })

  return exitState
}
```

#### useInterval

주기적 작업을 위한 Hook입니다.

**파일 위치:** `/src/hooks/useInterval.ts`

```typescript
export function useInterval(callback: () => void, delay: number): void {
  const savedCallback = useRef(callback)

  useEffect(() => {
    savedCallback.current = callback
  }, [callback])

  useEffect(() => {
    function tick() {
      savedCallback.current()
    }

    const id = setInterval(tick, delay)
    return () => clearInterval(id)
  }, [delay])
}
```

### 4.3 비즈니스 로직 Hooks

#### useUnifiedCompletion

통합 자동완성 시스템을 제공하는 복잡한 Hook입니다.

**파일 위치:** `/src/hooks/useUnifiedCompletion.ts`

**주요 기능:**
1. **명령어 자동완성**: `/` 로 시작하는 슬래시 명령어
2. **에이전트 자동완성**: `@` 로 시작하는 멘션
3. **파일 경로 자동완성**: Unix 스타일 경로 탐색
4. **스마트 매칭**: 퍼지 검색과 우선순위 기반 정렬

```typescript
interface UnifiedSuggestion {
  value: string
  displayValue: string
  type: 'command' | 'agent' | 'file' | 'ask'
  icon?: string
  score: number
  metadata?: any
}

export function useUnifiedCompletion({
  input,
  cursorOffset,
  onInputChange,
  setCursorOffset,
  commands,
  onSubmit,
}: Props) {
  const [state, setState] = useState<CompletionState>({
    suggestions: [],
    selectedIndex: 0,
    isActive: false,
    context: null,
    preview: null,
  })

  // 컨텍스트 감지
  const getWordAtCursor = useCallback((): CompletionContext | null => {
    if (!input) return null

    let start = cursorOffset
    while (start > 0) {
      const char = input[start - 1]
      if (/\s/.test(char)) break
      if (char === '@' && start < cursorOffset) {
        start--
        break
      }
      start--
    }

    const word = input.slice(start, cursorOffset)

    // 타입 감지
    if (word.startsWith('/')) {
      return { type: 'command', prefix: word.slice(1), startPos: start, endPos: cursorOffset }
    }
    if (word.startsWith('@')) {
      return { type: 'agent', prefix: word.slice(1), startPos: start, endPos: cursorOffset }
    }

    return { type: 'file', prefix: word, startPos: start, endPos: cursorOffset }
  }, [input, cursorOffset])

  // Tab 키 처리
  useInput((input_str, key) => {
    if (!key.tab) return false

    const context = getWordAtCursor()
    if (!context) return false

    if (state.isActive && state.suggestions.length > 0) {
      // 다음 제안으로 순환
      const nextIndex = (state.selectedIndex + 1) % state.suggestions.length
      updateState({ selectedIndex: nextIndex })
      return true
    }

    // 새 제안 생성
    const suggestions = generateSuggestions(context)
    if (suggestions.length === 0) return false

    if (suggestions.length === 1) {
      completeWith(suggestions[0], context)
    } else {
      activateCompletion(suggestions, context)
    }
    return true
  })

  return {
    suggestions: state.suggestions,
    selectedIndex: state.selectedIndex,
    isActive: state.isActive,
    emptyDirMessage: state.emptyDirMessage,
  }
}
```

**자동완성 예시:**

```bash
# 명령어 완성
> /he[Tab]
  ◆ /help
    /hello

# 에이전트 멘션
> @ag[Tab]
  ◆ 👤 run-agent-general-purpose :: General purpose coding assistant
    👤 run-agent-code-review :: Code review specialist

# 파일 경로
> src/to[Tab]
  ◆ 📁 src/tools/
    📁 src/toast/

# Unix 명령어
> gi[Tab]
  ◆ $ git
    $ gimp
```

#### useCanUseTool

툴 사용 권한을 관리하는 Hook입니다.

**파일 위치:** `/src/hooks/useCanUseTool.ts`

```typescript
export function useCanUseTool(
  setToolUseConfirm: SetState<ToolUseConfirm | null>,
): CanUseToolFn {
  return useCallback(async (tool, input, toolUseContext, assistantMessage) => {
    return new Promise(resolve => {
      // 이미 취소된 경우
      if (toolUseContext.abortController.signal.aborted) {
        resolve({ result: false, message: REJECT_MESSAGE })
        return
      }

      // 권한 확인
      hasPermissionsToUseTool(tool, input, toolUseContext, assistantMessage)
        .then(async result => {
          if (result.result) {
            resolve({ result: true })
            return
          }

          // 권한 요청 UI 표시
          const description = await tool.description(input)

          setToolUseConfirm({
            tool,
            description,
            input,
            onAllow(type) {
              resolve({ result: true })
            },
            onReject() {
              resolve({ result: false, message: REJECT_MESSAGE })
            },
            onAbort() {
              toolUseContext.abortController.abort()
            },
          })
        })
    })
  }, [setToolUseConfirm])
}
```

#### useArrowKeyHistory

명령어 히스토리 탐색을 위한 Hook입니다.

**파일 위치:** `/src/hooks/useArrowKeyHistory.ts`

```typescript
export function useArrowKeyHistory(
  onSetInput: (value: string, mode: 'bash' | 'prompt') => void,
  currentInput: string,
) {
  const [historyIndex, setHistoryIndex] = useState(0)
  const [lastTypedInput, setLastTypedInput] = useState('')

  function onHistoryUp() {
    const latestHistory = getHistory()
    if (historyIndex < latestHistory.length) {
      if (historyIndex === 0 && currentInput.trim() !== '') {
        setLastTypedInput(currentInput)  // 현재 입력 저장
      }
      const newIndex = historyIndex + 1
      setHistoryIndex(newIndex)
      updateInput(latestHistory[historyIndex])
    }
  }

  function onHistoryDown() {
    if (historyIndex > 1) {
      const newIndex = historyIndex - 1
      setHistoryIndex(newIndex)
      updateInput(latestHistory[newIndex - 1])
    } else if (historyIndex === 1) {
      setHistoryIndex(0)
      updateInput(lastTypedInput)  // 저장된 입력 복원
    }
  }

  return {
    historyIndex,
    onHistoryUp,
    onHistoryDown,
    resetHistory: () => {
      setLastTypedInput('')
      setHistoryIndex(0)
    },
  }
}
```

---

## 5. 화면(Screen) 구성과 전환

### 5.1 REPL Screen

메인 대화형 화면입니다.

**파일 위치:** `/src/screens/REPL.tsx`

#### 상태 관리

```typescript
export function REPL({
  commands,
  tools,
  initialPrompt,
  messageLogName,
  verbose,
}: Props): React.ReactNode {
  // 핵심 상태
  const [messages, setMessages] = useState<MessageType[]>(initialMessages ?? [])
  const [inputValue, setInputValue] = useState('')
  const [inputMode, setInputMode] = useState<'bash' | 'prompt' | 'koding'>('prompt')
  const [isLoading, setIsLoading] = useState(false)
  const [abortController, setAbortController] = useState<AbortController | null>(null)

  // UI 상태
  const [toolJSX, setToolJSX] = useState<{
    jsx: React.ReactNode | null
    shouldHidePromptInput: boolean
  } | null>(null)
  const [toolUseConfirm, setToolUseConfirm] = useState<ToolUseConfirm | null>(null)
  const [binaryFeedbackContext, setBinaryFeedbackContext] = useState<BinaryFeedbackContext | null>(null)

  // ... 로직
}
```

#### 렌더링 구조

```typescript
return (
  <PermissionProvider>
    {/* 모드 인디케이터 */}
    <ModeIndicator />

    {/* 정적 메시지 (로고, 히스토리) */}
    <Static
      items={messagesJSX.filter(_ => _.type === 'static')}
      children={(item: any) => item.jsx}
    />

    {/* 동적 메시지 (진행 중인 대화) */}
    {messagesJSX.filter(_ => _.type === 'transient').map(_ => _.jsx)}

    {/* 로딩 스피너 */}
    {!toolJSX && !toolUseConfirm && isLoading && <Spinner />}

    {/* 툴 JSX (로컬 명령어 UI) */}
    {toolJSX ? toolJSX.jsx : null}

    {/* 바이너리 피드백 (모델 선택) */}
    {binaryFeedbackContext && (
      <BinaryFeedback
        m1={binaryFeedbackContext.m1}
        m2={binaryFeedbackContext.m2}
        resolve={binaryFeedbackContext.resolve}
      />
    )}

    {/* 권한 요청 */}
    {toolUseConfirm && (
      <PermissionRequest
        toolUseConfirm={toolUseConfirm}
        onDone={() => setToolUseConfirm(null)}
      />
    )}

    {/* 비용 경고 다이얼로그 */}
    {showCostDialog && (
      <CostThresholdDialog onDone={() => setShowCostDialog(false)} />
    )}

    {/* 프롬프트 입력 */}
    {!toolUseConfirm && shouldShowPromptInput && (
      <PromptInput
        commands={commands}
        tools={tools}
        isLoading={isLoading}
        onQuery={onQuery}
        input={inputValue}
        onInputChange={setInputValue}
        mode={inputMode}
        onModeChange={setInputMode}
      />
    )}

    {/* 메시지 선택기 */}
    {isMessageSelectorVisible && (
      <MessageSelector
        messages={messages}
        onSelect={handleMessageSelect}
        onEscape={() => setIsMessageSelectorVisible(false)}
      />
    )}
  </PermissionProvider>
)
```

### 5.2 LogList Screen

로그 목록을 표시하는 화면입니다.

**파일 위치:** `/src/screens/LogList.tsx`

```typescript
export function LogList({ context, type, logNumber }: Props): React.ReactNode {
  const [logs, setLogs] = useState<LogOption[]>([])

  useEffect(() => {
    loadLogList(
      type === 'messages' ? CACHE_PATHS.messages() : CACHE_PATHS.errors()
    ).then(logs => {
      // 특정 로그 번호가 지정된 경우 즉시 출력
      if (logNumber !== undefined) {
        const log = logs[logNumber]
        if (log) {
          console.log(JSON.stringify(log.messages, null, 2))
          process.exit(0)
        }
      }
      setLogs(logs)
    })
  }, [type, logNumber])

  function onSelect(index: number): void {
    const log = logs[index]
    console.log(JSON.stringify(log.messages, null, 2))
    process.exit(0)
  }

  return <LogSelector logs={logs} onSelect={onSelect} />
}
```

### 5.3 ResumeConversation Screen

이전 대화를 복원하는 화면입니다.

**파일 위치:** `/src/screens/ResumeConversation.tsx`

```typescript
export function ResumeConversation({
  context,
  commands,
  logs,
  tools,
  verbose,
}: Props): React.ReactNode {
  async function onSelect(index: number) {
    const log = logs[index]
    if (!log) return

    // 메시지 역직렬화
    const messages = deserializeMessages(log.messages, tools)

    // 새 REPL 시작
    context.unmount?.()
    render(
      <REPL
        messageLogName={log.date}
        initialPrompt=""
        shouldShowPromptInput={true}
        commands={commands}
        tools={tools}
        initialMessages={messages}
        initialForkNumber={getNextAvailableLogForkNumber(log.date, log.forkNumber, 0)}
      />,
      { exitOnCtrlC: false }
    )
  }

  return <LogSelector logs={logs} onSelect={onSelect} />
}
```

### 5.4 Doctor Screen

시스템 상태를 확인하는 화면입니다.

**파일 위치:** `/src/screens/Doctor.tsx`

```typescript
export function Doctor({ onDone }: Props): React.ReactNode {
  const [checked, setChecked] = useState(false)
  const theme = getTheme()

  useEffect(() => {
    setChecked(true)
  }, [])

  useInput((_, key) => {
    if (key.return) onDone()
  })

  if (!checked) {
    return (
      <Box paddingX={1} paddingTop={1}>
        <Text color={theme.secondaryText}>Running checks…</Text>
      </Box>
    )
  }

  return (
    <Box flexDirection="column" gap={1} paddingX={1} paddingTop={1}>
      <Text color={theme.success}>✓ Installation checks passed</Text>
      <Text dimColor>Note: Auto-update is disabled by design.</Text>
      <PressEnterToContinue />
    </Box>
  )
}
```

---

## 6. 명령어 시스템

### 6.1 Command 인터페이스

**파일 위치:** `/src/commands.ts`

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  argNames?: string[]
  getPromptForCommand(args: string): Promise<MessageParam[]>
}

type LocalCommand = {
  type: 'local'
  call(args: string, context: Context): Promise<string>
}

type LocalJSXCommand = {
  type: 'local-jsx'
  call(onDone: (result?: string) => void, context: Context): Promise<React.ReactNode>
}

export type Command = {
  description: string
  isEnabled: boolean
  isHidden: boolean
  name: string
  aliases?: string[]
  userFacingName(): string
} & (PromptCommand | LocalCommand | LocalJSXCommand)
```

### 6.2 명령어 타입

#### 1. PromptCommand
AI에게 전달할 프롬프트를 생성합니다.

```typescript
// 예: review 명령어
{
  type: 'prompt',
  name: 'review',
  description: 'Review code changes',
  progressMessage: 'Reviewing code...',
  async getPromptForCommand(args: string) {
    return [{
      role: 'user',
      content: `Please review the following changes: ${args}`
    }]
  }
}
```

#### 2. LocalCommand
로컬에서 실행되어 문자열을 반환합니다.

```typescript
// 예: clear 명령어
{
  type: 'local',
  name: 'clear',
  description: 'Clear the screen',
  async call(args: string, context: Context) {
    await clearTerminal()
    return 'Screen cleared'
  }
}
```

#### 3. LocalJSXCommand
React 컴포넌트를 반환하는 명령어입니다.

```typescript
// 예: help 명령어
{
  type: 'local-jsx',
  name: 'help',
  description: 'Show help',
  async call(onDone, context) {
    return <Help commands={context.options.commands} onClose={onDone} />
  }
}
```

### 6.3 내장 명령어 목록

**파일 위치:** `/src/commands/`

| 명령어 | 타입 | 설명 |
|--------|------|------|
| `/help` | local-jsx | 도움말 표시 |
| `/model` | local-jsx | 모델 설정 |
| `/config` | local-jsx | 설정 열기 |
| `/agents` | local-jsx | 에이전트 관리 |
| `/cost` | local | 비용 표시 |
| `/clear` | local | 화면 지우기 |
| `/compact` | local | 컴팩트 모드 토글 |
| `/doctor` | local-jsx | 시스템 체크 |
| `/login` | local-jsx | 로그인 |
| `/logout` | local-jsx | 로그아웃 |
| `/mcp` | local | MCP 서버 관리 |
| `/onboarding` | local-jsx | 온보딩 |
| `/resume` | local-jsx | 대화 복원 |
| `/review` | prompt | 코드 리뷰 |

### 6.4 명령어 로딩 및 실행

```typescript
// 명령어 로딩
export const getCommands = memoize(async (): Promise<Command[]> => {
  const [mcpCommands, customCommands] = await Promise.all([
    getMCPCommands(),
    loadCustomCommands(),
  ])

  return [...mcpCommands, ...customCommands, ...COMMANDS()].filter(
    _ => _.isEnabled,
  )
})

// 명령어 검색
export function getCommand(commandName: string, commands: Command[]): Command {
  const command = commands.find(
    _ => _.userFacingName() === commandName || _.aliases?.includes(commandName),
  )

  if (!command) {
    throw ReferenceError(`Command ${commandName} not found`)
  }

  return command
}
```

---

## 7. 커스텀 명령어 시스템

**파일 위치:** `/src/services/customCommands.ts`

### 7.1 커스텀 명령어 구조

커스텀 명령어는 **마크다운 파일**로 정의됩니다:

```markdown
---
name: mycommand
description: "My custom command"
aliases: [mc, mycmd]
enabled: true
hidden: false
progressMessage: "Running my command..."
argNames: [file, action]
allowed-tools: ["FileRead", "Bash"]
---

# My Custom Command

This is the prompt that will be sent to the AI.

You can use $ARGUMENTS to insert user arguments.
You can use {file} and {action} for named arguments.
You can use @filepath to include file contents.
You can use !`command` to execute bash commands.
```

### 7.2 Frontmatter 인터페이스

```typescript
export interface CustomCommandFrontmatter {
  name?: string                    // 명령어 이름
  description?: string             // 설명
  aliases?: string[]               // 별칭
  enabled?: boolean                // 활성화 여부 (기본: true)
  hidden?: boolean                 // 숨김 여부 (기본: false)
  progressMessage?: string         // 진행 메시지
  argNames?: string[]              // 인자 이름 목록
  'allowed-tools'?: string[]       // 허용된 툴 목록
}
```

### 7.3 커스텀 명령어 디렉토리

```
~/.claude/commands/       # Claude Code 호환 (사용자 레벨)
~/.kode/commands/         # Kode 전용 (사용자 레벨)
./.claude/commands/       # Claude Code 호환 (프로젝트 레벨)
./.kode/commands/         # Kode 전용 (프로젝트 레벨)
```

**우선순위**: 프로젝트 > 사용자, Kode > Claude

### 7.4 커스텀 명령어 기능

#### 1. 파일 참조 (@filepath)

```markdown
---
name: review-file
---

Please review this file: @src/index.ts

The file content will be automatically included.
```

파일 참조 처리:
```typescript
export async function resolveFileReferences(content: string): Promise<string> {
  const fileRefRegex = /@([a-zA-Z0-9/._-]+(?:\.[a-zA-Z0-9]+)?)/g
  const matches = [...content.matchAll(fileRefRegex)]

  let result = content
  for (const match of matches) {
    const filePath = match[1]
    const fullPath = join(getCwd(), filePath)

    if (existsSync(fullPath)) {
      const fileContent = readFileSync(fullPath, 'utf-8')
      const formattedContent = `\n\n## File: ${filePath}\n\`\`\`\n${fileContent}\n\`\`\`\n`
      result = result.replace(match[0], formattedContent)
    }
  }

  return result
}
```

#### 2. Bash 명령어 실행 (!`command`)

```markdown
---
name: git-status-review
---

Current git status:
!`git status`

Please analyze the changes.
```

Bash 명령어 처리:
```typescript
export async function executeBashCommands(content: string): Promise<string> {
  const bashCommandRegex = /!\`([^`]+)\`/g
  const matches = [...content.matchAll(bashCommandRegex)]

  let result = content
  for (const match of matches) {
    const command = match[1].trim()
    const parts = command.split(/\s+/)

    try {
      const { stdout } = await execFileAsync(parts[0], parts.slice(1), {
        timeout: 5000,
        cwd: getCwd(),
      })

      result = result.replace(match[0], stdout.trim())
    } catch (error) {
      result = result.replace(match[0], `(error executing: ${command})`)
    }
  }

  return result
}
```

#### 3. 인자 치환

```markdown
---
name: test-file
argNames: [filename, testType]
---

Run {testType} tests for {filename}

# 사용 예
> /test-file utils.ts unit
# -> Run unit tests for utils.ts
```

인자 치환 로직:
```typescript
async getPromptForCommand(args: string): Promise<MessageParam[]> {
  let prompt = content.trim()

  // $ARGUMENTS 치환
  if (prompt.includes('$ARGUMENTS')) {
    prompt = prompt.replace(/\$ARGUMENTS/g, args || '')
  }

  // 명명된 인자 치환
  if (argNames && argNames.length > 0) {
    const argValues = args.trim().split(/\s+/)
    argNames.forEach((argName, index) => {
      const value = argValues[index] || ''
      prompt = prompt.replace(new RegExp(`\\{${argName}\\}`, 'g'), value)
    })
  }

  return [{ role: 'user', content: prompt }]
}
```

### 7.5 커스텀 명령어 로딩

```typescript
export const loadCustomCommands = memoize(async (): Promise<CustomCommandWithScope[]> => {
  const userClaudeDir = join(homedir(), '.claude', 'commands')
  const projectClaudeDir = join(getCwd(), '.claude', 'commands')
  const userKodeDir = join(homedir(), '.kode', 'commands')
  const projectKodeDir = join(getCwd(), '.kode', 'commands')

  // 모든 디렉토리 스캔
  const [projectClaudeFiles, userClaudeFiles, projectKodeFiles, userKodeFiles] =
    await Promise.all([
      existsSync(projectClaudeDir) ? scanMarkdownFiles(projectClaudeDir) : [],
      existsSync(userClaudeDir) ? scanMarkdownFiles(userClaudeDir) : [],
      existsSync(projectKodeDir) ? scanMarkdownFiles(projectKodeDir) : [],
      existsSync(userKodeDir) ? scanMarkdownFiles(userKodeDir) : [],
    ])

  // 우선순위: 프로젝트 > 사용자, Kode > Claude
  const projectFiles = [...projectKodeFiles, ...projectClaudeFiles]
  const userFiles = [...userKodeFiles, ...userClaudeFiles]

  const commands: CustomCommandWithScope[] = []

  // 파일 파싱 및 명령어 생성
  for (const filePath of [...projectFiles, ...userFiles]) {
    const content = readFileSync(filePath, 'utf-8')
    const { frontmatter, content: commandContent } = parseFrontmatter(content)
    const command = createCustomCommand(frontmatter, commandContent, filePath, baseDir)

    if (command) {
      commands.push(command)
    }
  }

  return commands.filter(cmd => cmd.isEnabled)
})
```

### 7.6 Frontmatter 파싱

```typescript
export function parseFrontmatter(content: string): {
  frontmatter: CustomCommandFrontmatter
  content: string
} {
  const frontmatterRegex = /^---\s*\n([\s\S]*?)---\s*\n?/
  const match = content.match(frontmatterRegex)

  if (!match) {
    return { frontmatter: {}, content }
  }

  const yamlContent = match[1]
  const markdownContent = content.slice(match[0].length)
  const frontmatter: CustomCommandFrontmatter = {}

  // 간단한 YAML 파서
  const lines = yamlContent.split('\n')
  for (const line of lines) {
    const trimmed = line.trim()
    if (!trimmed || trimmed.startsWith('#')) continue

    const colonIndex = trimmed.indexOf(':')
    if (colonIndex === -1) continue

    const key = trimmed.slice(0, colonIndex).trim()
    const value = trimmed.slice(colonIndex + 1).trim()

    // 인라인 배열 [item1, item2]
    if (value.startsWith('[') && value.endsWith(']')) {
      const items = value.slice(1, -1).split(',').map(s => s.trim().replace(/['"]/g, ''))
      frontmatter[key] = items
    }
    // 불린값
    else if (value === 'true' || value === 'false') {
      frontmatter[key] = value === 'true'
    }
    // 문자열
    else {
      frontmatter[key] = value.replace(/['"]/g, '')
    }
  }

  return { frontmatter, content: markdownContent }
}
```

---

## 8. 알림 시스템

**파일 위치:** `/src/services/notifier.ts`

### 8.1 알림 타입

```typescript
export type NotificationOptions = {
  message: string
  title?: string
}
```

### 8.2 알림 채널

사용자 설정에 따라 다양한 알림 방식을 지원합니다:

```typescript
export async function sendNotification(notif: NotificationOptions): Promise<void> {
  const channel = getGlobalConfig().preferredNotifChannel

  switch (channel) {
    case 'iterm2':
      sendITerm2Notification(notif)
      break

    case 'terminal_bell':
      sendTerminalBell()
      break

    case 'iterm2_with_bell':
      sendITerm2Notification(notif)
      sendTerminalBell()
      break

    case 'notifications_disabled':
      // 아무것도 하지 않음
      break
  }
}
```

### 8.3 iTerm2 알림

```typescript
function sendITerm2Notification({ message, title }: NotificationOptions): void {
  const displayString = title ? `${title}:\n${message}` : message

  try {
    // iTerm2 전용 escape sequence
    process.stdout.write(`\x1b]9;\n\n${displayString}\x07`)
  } catch {
    // 에러 무시
  }
}
```

### 8.4 터미널 벨

```typescript
function sendTerminalBell(): void {
  process.stdout.write('\x07')
}
```

### 8.5 사용 예시

```typescript
// useNotifyAfterTimeout Hook
export function useNotifyAfterTimeout(message: string, timeout: number = 10000) {
  useEffect(() => {
    const timer = setTimeout(() => {
      sendNotification({
        title: 'Kode',
        message,
      })
    }, timeout)

    return () => clearTimeout(timer)
  }, [message, timeout])
}

// 권한 요청 시 알림
const toolName = toolUseConfirm.tool.userFacingName?.() || 'Tool'
useNotifyAfterTimeout(
  `Kode needs your permission to use ${toolName}`,
)
```

---

## 9. 컴포넌트 구조도

### 9.1 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                        Kode CLI App                         │
│                     (Ink Renderer)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Screen Layer                           │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   REPL   │  │ LogList  │  │  Doctor  │  │  Resume  │   │
│  │  Screen  │  │  Screen  │  │  Screen  │  │  Screen  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Component Layer                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │  PromptInput    │  │   Message      │  │     Logo     │ │
│  │  - TextInput    │  │   - Assistant  │  │   - Banner   │ │
│  │  - Completion   │  │   - User       │  │   - MCP Info │ │
│  │  - Mode Toggle  │  │   - ToolUse    │  │              │ │
│  └─────────────────┘  └────────────────┘  └──────────────┘ │
│                                                              │
│  ┌─────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │  Permission     │  │  BinaryFeedback│  │   Spinner    │ │
│  │  Request        │  │                │  │              │ │
│  └─────────────────┘  └────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Hooks Layer                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  Terminal Hooks  │  │  Control Hooks   │                │
│  │  - TerminalSize  │  │  - CancelRequest │                │
│  │  - TextInput     │  │  - ExitOnCtrlCD  │                │
│  │  - Interval      │  │                  │                │
│  └──────────────────┘  └──────────────────┘                │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  Business Hooks  │  │  Utility Hooks   │                │
│  │  - Completion    │  │  - ArrowHistory  │                │
│  │  - CanUseTool    │  │  - ApiKeyVerify  │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Service Layer                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  Command System │  │  Custom Commands│  │  Notifier   │ │
│  │  - Built-in     │  │  - Loader       │  │  - iTerm2   │ │
│  │  - MCP          │  │  - Parser       │  │  - Bell     │ │
│  │  - Custom       │  │  - Executor     │  │             │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 REPL 컴포넌트 상세 구조

```
REPL
├── PermissionProvider (Context)
│   │
│   ├── ModeIndicator
│   │   └── 현재 모드 표시 (Bash/Koding/Prompt)
│   │
│   ├── Static Messages (한 번만 렌더링)
│   │   ├── Logo
│   │   │   ├── Welcome Banner
│   │   │   ├── Update Notice
│   │   │   ├── Current Directory
│   │   │   └── MCP Server Status
│   │   │
│   │   └── ProjectOnboarding
│   │       └── 프로젝트 설정 가이드
│   │
│   ├── Transient Messages (동적 렌더링)
│   │   └── Message[]
│   │       ├── UserMessage
│   │       │   ├── UserTextMessage
│   │       │   └── UserToolResultMessage
│   │       │
│   │       └── AssistantMessage
│   │           ├── AssistantTextMessage
│   │           ├── AssistantToolUseMessage
│   │           ├── AssistantThinkingMessage
│   │           └── AssistantRedactedThinkingMessage
│   │
│   ├── Interactive Components (조건부 렌더링)
│   │   ├── Spinner (isLoading)
│   │   │
│   │   ├── ToolJSX (로컬 명령어 UI)
│   │   │   ├── Help
│   │   │   ├── ModelConfig
│   │   │   ├── Config
│   │   │   └── ...
│   │   │
│   │   ├── BinaryFeedback (모델 선택)
│   │   │   ├── Option 1
│   │   │   └── Option 2
│   │   │
│   │   ├── PermissionRequest (툴 권한 요청)
│   │   │   ├── FileEditPermissionRequest
│   │   │   ├── BashPermissionRequest
│   │   │   └── ...
│   │   │
│   │   └── CostThresholdDialog (비용 경고)
│   │
│   ├── PromptInput (메인 입력)
│   │   ├── ModelInfo (상단)
│   │   │   └── [Provider] Model: 50k / 200k
│   │   │
│   │   ├── InputBox (중앙)
│   │   │   ├── ModeIndicator (! / # / >)
│   │   │   └── TextInput
│   │   │       └── useTextInput
│   │   │           ├── Cursor Management
│   │   │           ├── Key Bindings
│   │   │           └── Image Paste
│   │   │
│   │   ├── Completion Suggestions (하단)
│   │   │   └── useUnifiedCompletion
│   │   │       ├── Command Completion
│   │   │       ├── Agent Completion
│   │   │       ├── File Completion
│   │   │       └── Unix Command Completion
│   │   │
│   │   └── Helper Text (최하단)
│   │       ├── Mode Help (! for bash, # for AGENTS.md)
│   │       ├── Shortcuts (/commands, ctrl+m, ctrl+g)
│   │       └── TokenWarning
│   │
│   └── MessageSelector (Esc로 열기)
│       └── 이전 메시지 선택 및 Fork
```

### 9.3 데이터 흐름

```
사용자 입력
    │
    ▼
TextInput (useTextInput)
    │
    ├──► useUnifiedCompletion ──► Suggestions
    │         │
    │         └──► Tab/Enter ──► Complete
    │
    ▼
PromptInput.onSubmit
    │
    ├──► processUserInput ──► Message[]
    │         │
    │         ├──► Slash Command? ──► Execute Command
    │         ├──► Bash Command? ──► BashTool
    │         └──► Normal Prompt ──► API Query
    │
    ▼
REPL.onQuery
    │
    ├──► getSystemPrompt
    ├──► getContext
    └──► query() ──► AI API
              │
              ▼
         Response Stream
              │
              ├──► Text Block ──► AssistantTextMessage
              ├──► Tool Use ──► PermissionRequest
              │                      │
              │                      ├──► Allow ──► Execute Tool
              │                      └──► Reject ──► Skip
              │
              └──► Update Messages ──► Re-render
```

### 9.4 Hook 의존성 그래프

```
useUnifiedCompletion
├── useState (CompletionState)
├── useCallback (많은 함수들)
├── useInput (Tab, Enter, Arrows, Esc)
└── useEffect (자동 트리거)
    ├── getWordAtCursor
    ├── generateSuggestions
    │   ├── generateCommandSuggestions
    │   ├── generateMentionSuggestions
    │   ├── generateFileSuggestions
    │   └── generateUnixCommandSuggestions
    └── shouldAutoTrigger

useTextInput
├── useState (offset)
├── useDoublePress (Ctrl+C, Esc)
└── Cursor Utility
    ├── startOfLine
    ├── endOfLine
    ├── left / right
    ├── up / down
    ├── backspace / delete
    └── insert

useCancelRequest
└── useInput (Esc)
    └── Abort + Cleanup

useExitOnCtrlCD
└── useInput (Ctrl+C, Ctrl+D)
    └── useDoublePress
        └── Exit

useArrowKeyHistory
├── useState (historyIndex, lastTypedInput)
└── getHistory()

useCanUseTool
├── useCallback
└── hasPermissionsToUseTool
    └── setToolUseConfirm
```

---

## 마무리

Kode-CLI의 UI/CLI 시스템은 다음과 같은 특징을 가집니다:

### 핵심 강점

1. **선언적 UI**: React/Ink를 통한 컴포넌트 기반 아키텍처
2. **최적화된 렌더링**: Static/Transient 분리로 성능 향상
3. **풍부한 상호작용**: 다양한 키보드 단축키와 자동완성
4. **확장 가능한 명령어 시스템**: 내장/MCP/커스텀 명령어 통합
5. **타입 안전성**: TypeScript와 Zod를 통한 런타임 검증

### 설계 원칙

- **모듈화**: 각 컴포넌트와 Hook은 독립적으로 작동
- **재사용성**: 공통 패턴을 Hook과 유틸리티로 추출
- **성능**: 메모이제이션과 최적화된 렌더링 전략
- **확장성**: 플러그인 시스템(MCP, 커스텀 명령어)
- **사용자 경험**: 직관적인 인터페이스와 즉각적인 피드백

이러한 아키텍처는 터미널 환경에서도 현대적이고 반응성 높은 UI를 제공하며, 개발자가 AI 어시스턴트와 자연스럽게 상호작용할 수 있게 합니다.
