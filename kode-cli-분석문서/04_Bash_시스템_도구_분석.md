# Bash 및 시스템 도구 분석

## 목차
1. [BashTool - 명령어 실행 도구](#1-bashtool---명령어-실행-도구)
2. [NotebookEditTool - Jupyter 노트북 편집](#2-notebookedittool---jupyter-노트북-편집)
3. [NotebookReadTool - Jupyter 노트북 읽기](#3-notebookreadtool---jupyter-노트북-읽기)
4. [TodoWriteTool - 작업 관리 시스템](#4-todowritetool---작업-관리-시스템)
5. [MemoryTool - 컨텍스트 메모리 관리](#5-memorytool---컨텍스트-메모리-관리)

---

## 1. BashTool - 명령어 실행 도구

### 1.1 개요

BashTool은 Kode-cli에서 셸 명령어를 실행하는 핵심 도구입니다. 사용자의 컴퓨터에서 안전하게 bash 명령어를 실행하고, 출력을 처리하며, 보안 제어를 통해 악의적인 명령 실행을 방지합니다.

**파일 위치:**
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/BashTool/BashTool.tsx`

### 1.2 입력 스키마

```typescript
export const inputSchema = z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: z
    .number()
    .optional()
    .describe('Optional timeout in milliseconds (max 600000)'),
})
```

**파라미터:**
- `command` (필수): 실행할 bash 명령어
- `timeout` (선택): 밀리초 단위 타임아웃 (최대 600000ms = 10분, 기본값: 120000ms = 2분)

### 1.3 출력 구조

```typescript
type Out = {
  stdout: string                  // 표준 출력
  stdoutLines: number             // 원본 stdout의 총 라인 수
  stderr: string                  // 표준 에러
  stderrLines: number             // 원본 stderr의 총 라인 수
  interrupted: boolean            // 명령이 중단되었는지 여부
}
```

### 1.4 보안 메커니즘

#### 1.4.1 금지된 명령어 목록 (BANNED_COMMANDS)

보안상의 이유로 다음 명령어들이 차단됩니다:

```typescript
export const BANNED_COMMANDS = [
  'alias',       // 별칭 설정 방지
  'curl',        // HTTP 요청 방지
  'curlie',
  'wget',
  'axel',
  'aria2c',
  'nc',          // 네트워크 연결 방지
  'telnet',
  'lynx',        // 웹 브라우저 방지
  'w3m',
  'links',
  'httpie',
  'xh',
  'http-prompt',
  'chrome',      // GUI 브라우저 방지
  'firefox',
  'safari',
]
```

**차단 이유:**
- 프롬프트 인젝션 공격 방지
- 외부 네트워크 요청 제한
- 악의적인 스크립트 다운로드 방지

#### 1.4.2 디렉토리 변경 제한 (cd 명령어 검증)

```typescript
// cd 명령어에 대한 특별 처리
if (baseCmd === 'cd' && parts[1]) {
  const targetDir = parts[1]!.replace(/^['"]|['"]$/g, '') // 따옴표 제거
  const fullTargetDir = isAbsolute(targetDir)
    ? targetDir
    : resolve(getCwd(), targetDir)

  // 원본 작업 디렉토리의 하위 디렉토리로만 이동 허용
  if (!isInDirectory(
    relative(getOriginalCwd(), fullTargetDir),
    relative(getCwd(), getOriginalCwd()),
  )) {
    return {
      result: false,
      message: `ERROR: cd to '${fullTargetDir}' was blocked. For security, ${PRODUCT_NAME} may only change directories to child directories of the original working directory (${getOriginalCwd()}) for this session.`,
    }
  }
}
```

**보안 정책:**
- 세션 시작 시의 원본 작업 디렉토리(`getOriginalCwd()`)에서만 이동 가능
- 상위 디렉토리나 외부 경로로의 이동 차단
- 샌드박스 환경 유지

#### 1.4.3 디렉토리 자동 리셋

```typescript
if (!isInDirectory(getCwd(), getOriginalCwd())) {
  // 셸 디렉토리가 원본 작업 디렉토리 밖에 있으면 리셋
  await PersistentShell.getInstance().setCwd(getOriginalCwd())
  stderr = `${stderr.trim()}${EOL}Shell cwd was reset to ${getOriginalCwd()}`
}
```

**동작 방식:**
- 명령 실행 후 현재 디렉토리 검증
- 허용된 범위를 벗어나면 자동으로 원본 디렉토리로 복귀
- 사용자에게 리셋 메시지 출력

### 1.5 명령어 실행 메커니즘

#### 1.5.1 PersistentShell 사용

```typescript
const result = await PersistentShell.getInstance().exec(
  command,
  abortController.signal,
  timeout,
)
```

**특징:**
- **영구 셸 세션**: 여러 명령어 간 상태 유지 (환경 변수, 가상 환경 등)
- **비동기 실행**: Generator 함수로 스트리밍 출력 지원
- **취소 가능**: AbortController를 통한 중단 기능
- **타임아웃 제어**: 지정된 시간 초과 시 자동 종료

#### 1.5.2 출력 포맷팅 (`formatOutput` 함수)

```typescript
export function formatOutput(content: string): {
  totalLines: number
  truncatedContent: string
} {
  if (content.length <= MAX_OUTPUT_LENGTH) {
    return {
      totalLines: content.split('\n').length,
      truncatedContent: content,
    }
  }

  // 30000자 초과 시 중간 부분 잘라내기
  const halfLength = MAX_OUTPUT_LENGTH / 2
  const start = content.slice(0, halfLength)
  const end = content.slice(-halfLength)
  const truncated = `${start}\n\n... [${content.slice(halfLength, -halfLength).split('\n').length} lines truncated] ...\n\n${end}`

  return {
    totalLines: content.split('\n').length,
    truncatedContent: truncated,
  }
}
```

**출력 제한:**
- `MAX_OUTPUT_LENGTH = 30000` 문자
- 초과 시 앞부분 15000자 + 뒷부분 15000자만 표시
- 잘린 라인 수를 명시적으로 표시

### 1.6 HEREDOC 패턴 처리

```typescript
renderToolUseMessage({ command }) {
  // HEREDOC 패턴을 깔끔하게 정리
  if (command.includes("\"$(cat <<'EOF'")) {
    const match = command.match(
      /^(.*?)"?\$\(cat <<'EOF'\n([\s\S]*?)\n\s*EOF\n\s*\)"(.*)$/,
    )
    if (match && match[1] && match[2]) {
      const prefix = match[1]
      const content = match[2]
      const suffix = match[3] || ''
      return `${prefix.trim()} "${content.trim()}"${suffix.trim()}`
    }
  }
  return command
}
```

**용도:**
- Git commit 메시지와 같은 멀티라인 텍스트 처리
- UI에서 가독성 좋게 표시
- 복잡한 HEREDOC 구문을 간단한 형태로 변환

### 1.7 파일 경로 추출 (AI 기반)

```typescript
export async function getCommandFilePaths(
  command: string,
  output: string,
): Promise<string[]> {
  const response = await queryQuick({
    systemPrompt: [
      `Extract any file paths that this command reads or modifies.
      For commands like "git diff" and "cat", include the paths of files being shown.
      Use paths verbatim -- don't add any slashes or try to resolve them.
      Do not try to infer paths that were not explicitly listed in the command output.

      Format your response as:
      <filepaths>
      path/to/file1
      path/to/file2
      </filepaths>`
    ],
    userPrompt: `Command: ${command}\nOutput: ${output}`,
    enablePromptCaching: true,
  })

  const content = response.message.content
    .filter(_ => _.type === 'text')
    .map(_ => _.text)
    .join('')

  return (
    extractTag(content, 'filepaths')?.trim().split('\n').filter(Boolean) || []
  )
}
```

**혁신적인 접근:**
- AI 모델(Haiku)을 사용하여 명령어 출력에서 파일 경로 추출
- 정규식보다 더 유연하고 정확한 파싱
- 프롬프트 캐싱으로 성능 최적화
- 추출된 파일 경로는 `readFileTimestamps`에 기록되어 파일 변경 추적

### 1.8 UI 렌더링

#### 1.8.1 BashToolResultMessage 컴포넌트

```typescript
function BashToolResultMessage({ content, verbose }: Props): React.JSX.Element {
  const { stdout, stdoutLines, stderr, stderrLines } = content

  return (
    <Box flexDirection="column">
      {stdout !== '' ? (
        <OutputLine content={stdout} lines={stdoutLines} verbose={verbose} />
      ) : null}
      {stderr !== '' ? (
        <OutputLine
          content={stderr}
          lines={stderrLines}
          verbose={verbose}
          isError
        />
      ) : null}
      {stdout === '' && stderr === '' ? (
        <Box flexDirection="row">
          <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
          <Text color={getTheme().secondaryText}>(No content)</Text>
        </Box>
      ) : null}
    </Box>
  )
}
```

**렌더링 로직:**
- stdout과 stderr를 분리하여 표시
- 에러는 빨간색으로 강조
- 출력이 없을 경우 "(No content)" 메시지

#### 1.8.2 OutputLine 컴포넌트 (축약 표시)

```typescript
function renderTruncatedContent(content: string, totalLines: number): string {
  const allLines = content.split('\n')
  if (allLines.length <= MAX_RENDERED_LINES) {
    return allLines.join('\n')
  }

  // 기본적으로 마지막 5줄만 표시
  const lastLines = allLines.slice(-MAX_RENDERED_LINES)
  return [
    chalk.grey(
      `Showing last ${MAX_RENDERED_LINES} lines of ${totalLines} total lines`,
    ),
    ...lastLines,
  ].join('\n')
}
```

**UI 최적화:**
- `MAX_RENDERED_LINES = 5`: 기본적으로 마지막 5줄만 표시
- 긴 출력을 자동으로 축약하여 터미널 가독성 향상
- verbose 모드에서는 전체 내용 표시

### 1.9 Git 워크플로우 지원

BashTool의 프롬프트는 Git 작업에 대한 상세한 가이드를 제공합니다:

#### 1.9.1 커밋 생성 프로세스

```markdown
1. Start with a single message that contains exactly three tool_use blocks:
   - Run a git status command to see all untracked files.
   - Run a git diff command to see both staged and unstaged changes.
   - Run a git log command to see recent commit messages.

2. Use the git context to determine which files are relevant.

3. Analyze all staged changes and draft a commit message.

4. Create the commit with a message ending with:
   🤖 Generated with ${PRODUCT_NAME} & {MODEL_NAME}
   Co-Authored-By: ${PRODUCT_NAME} <noreply@${PRODUCT_NAME}.com>
```

**커밋 메시지 형식 (HEREDOC 사용):**

```bash
git commit -m "$(cat <<'EOF'
   Commit message here.

   🤖 Generated with Kode & Claude 3.5 Sonnet
   Co-Authored-By: Kode <noreply@Kode.com>
   EOF
   )"
```

#### 1.9.2 Pull Request 생성

```markdown
1. Understand the current state of the branch:
   - Run git status
   - Run git diff
   - Check if current branch tracks a remote branch
   - Run git log and git diff main...HEAD

2. Create new branch if needed
3. Commit changes if needed
4. Push to remote with -u flag if needed
5. Analyze all changes and draft a PR summary
6. Create PR using gh pr create
```

**PR 본문 형식:**

```bash
gh pr create --title "the pr title" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points>

## Test plan
[Checklist of TODOs for testing the pull request...]

🤖 Generated with [Kode](https://kode.dev) & Claude 3.5 Sonnet
EOF
)"
```

### 1.10 중요한 사용 지침

```markdown
Usage notes:
- VERY IMPORTANT: You MUST avoid using search commands like `find` and `grep`.
  Instead use Grep, Glob, or Task tools.
- You MUST avoid read tools like `cat`, `head`, `tail`, and `ls`,
  and use FileRead and LS tools.
- When issuing multiple commands, use ';' or '&&' operator.
  DO NOT use newlines (newlines are ok in quoted strings).
- All commands share the same shell session.
  Shell state (environment variables, virtual environments, current directory, etc.)
  persist between commands.
- Try to maintain your current working directory throughout the session
  by using absolute paths and avoiding usage of `cd`.
```

**권장 패턴:**

```bash
# 좋은 예
pytest /foo/bar/tests

# 나쁜 예
cd /foo/bar && pytest tests
```

---

## 2. NotebookEditTool - Jupyter 노트북 편집

### 2.1 개요

NotebookEditTool은 Jupyter 노트북(.ipynb 파일)의 셀을 편집하는 도구입니다. 코드 셀과 마크다운 셀을 생성, 수정, 삭제할 수 있습니다.

**파일 위치:**
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/NotebookEditTool/NotebookEditTool.tsx`

### 2.2 입력 스키마

```typescript
const inputSchema = z.strictObject({
  notebook_path: z
    .string()
    .describe('The absolute path to the Jupyter notebook file to edit (must be absolute, not relative)'),
  cell_number: z.number().describe('The index of the cell to edit (0-based)'),
  new_source: z.string().describe('The new source for the cell'),
  cell_type: z
    .enum(['code', 'markdown'])
    .optional()
    .describe('The type of the cell (code or markdown). If not specified, it defaults to the current cell type. If using edit_mode=insert, this is required.'),
  edit_mode: z
    .string()
    .optional()
    .describe('The type of edit to make (replace, insert, delete). Defaults to replace.'),
})
```

**파라미터:**
- `notebook_path`: 노트북 파일의 절대 경로 (필수)
- `cell_number`: 편집할 셀의 인덱스 (0부터 시작, 필수)
- `new_source`: 새로운 셀 내용 (필수)
- `cell_type`: 셀 유형 ('code' 또는 'markdown', 선택)
- `edit_mode`: 편집 모드 ('replace', 'insert', 'delete', 기본값: 'replace')

### 2.3 편집 모드

#### 2.3.1 Replace Mode (기본값)

```typescript
// 기존 셀 내용 교체
const targetCell = notebook.cells[cell_number]!
targetCell.source = new_source

// 실행 카운트와 출력 초기화 (셀이 수정되었으므로)
targetCell.execution_count = undefined
targetCell.outputs = []

// 셀 타입 변경 (지정된 경우)
if (cell_type && cell_type !== targetCell.cell_type) {
  targetCell.cell_type = cell_type
}
```

**동작:**
- 지정된 인덱스의 셀 내용을 새로운 내용으로 교체
- 셀 수정 시 실행 결과와 출력을 자동으로 제거
- 셀 타입 변경 가능

#### 2.3.2 Insert Mode

```typescript
// 새로운 셀 생성
const new_cell = {
  cell_type: cell_type!, // validateInput에서 검증됨
  source: new_source,
  metadata: {},
}

// 지정된 위치에 삽입
notebook.cells.splice(
  cell_number,
  0,
  cell_type == 'markdown' ? new_cell : { ...new_cell, outputs: [] },
)
```

**동작:**
- 지정된 인덱스 위치에 새로운 셀 삽입
- 코드 셀은 `outputs: []` 배열 포함
- 마크다운 셀은 outputs 없음
- `cell_type` 파라미터 필수

**검증:**
```typescript
// insert 모드는 노트북 끝에 추가하는 것도 허용
if (edit_mode === 'insert' && cell_number > notebook.cells.length) {
  return {
    result: false,
    message: `Cell number is out of bounds. For insert mode, the maximum value is ${notebook.cells.length} (to append at the end).`,
  }
}
```

#### 2.3.3 Delete Mode

```typescript
// 지정된 셀 삭제
notebook.cells.splice(cell_number, 1)
```

**동작:**
- 지정된 인덱스의 셀을 노트북에서 제거
- 이후 셀들의 인덱스가 자동으로 조정됨

### 2.4 입력 검증

```typescript
async validateInput({
  notebook_path,
  cell_number,
  cell_type,
  edit_mode = 'replace',
}) {
  // 1. 파일 존재 확인
  if (!existsSync(fullPath)) {
    return { result: false, message: 'Notebook file does not exist.' }
  }

  // 2. 파일 확장자 검증
  if (extname(fullPath) !== '.ipynb') {
    return {
      result: false,
      message: 'File must be a Jupyter notebook (.ipynb file). For editing other file types, use the FileEdit tool.',
    }
  }

  // 3. 셀 번호 검증
  if (cell_number < 0) {
    return { result: false, message: 'Cell number must be non-negative.' }
  }

  // 4. 편집 모드 검증
  if (edit_mode !== 'replace' && edit_mode !== 'insert' && edit_mode !== 'delete') {
    return { result: false, message: 'Edit mode must be replace, insert, or delete.' }
  }

  // 5. insert 모드에서 cell_type 필수 검증
  if (edit_mode === 'insert' && !cell_type) {
    return { result: false, message: 'Cell type is required when using edit_mode=insert.' }
  }

  // 6. JSON 유효성 검증
  const notebook = safeParseJSON(content) as NotebookContent | null
  if (!notebook) {
    return { result: false, message: 'Notebook is not valid JSON.' }
  }

  // 7. 셀 범위 검증
  if (edit_mode === 'insert' && cell_number > notebook.cells.length) {
    return {
      result: false,
      message: `Cell number is out of bounds. For insert mode, the maximum value is ${notebook.cells.length} (to append at the end).`,
    }
  } else if (
    (edit_mode === 'replace' || edit_mode === 'delete') &&
    (cell_number >= notebook.cells.length || !notebook.cells[cell_number])
  ) {
    return {
      result: false,
      message: `Cell number is out of bounds. Notebook has ${notebook.cells.length} cells.`,
    }
  }

  return { result: true }
}
```

### 2.5 파일 저장 및 추적

```typescript
// 줄바꿈 형식 감지 및 보존
const endings = detectLineEndings(fullPath)
const updatedNotebook = JSON.stringify(notebook, null, 1)
writeTextContent(fullPath, updatedNotebook, enc, endings!)

// 파일 편집 기록 (파일 신선도 추적)
recordFileEdit(fullPath, updatedNotebook)

// 시스템 리마인더 이벤트 발생
emitReminderEvent('file:edited', {
  filePath: fullPath,
  cellNumber: cell_number,
  newSource: new_source,
  cellType: cell_type,
  editMode: edit_mode || 'replace',
  timestamp: Date.now(),
  operation: 'notebook_edit',
})
```

**파일 처리:**
- 인코딩 자동 감지 (`detectFileEncoding`)
- 줄바꿈 형식 보존 (CRLF vs LF)
- 들여쓰기 1칸으로 JSON 포맷팅 (공간 절약)
- 파일 변경 추적 시스템과 통합

### 2.6 권한 관리

```typescript
needsPermissions({ notebook_path }) {
  return !hasWritePermission(notebook_path)
}
```

**동작:**
- 파일 쓰기 권한이 없으면 사용자에게 권한 요청
- 프로젝트별 권한 설정 존중
- 보안성 강화

### 2.7 UI 렌더링

```typescript
renderToolResultMessage({ cell_number, new_source, language, error }) {
  if (error) {
    return (
      <Box flexDirection="column">
        <Text color="red">{error}</Text>
      </Box>
    )
  }

  return (
    <Box flexDirection="column">
      <Text>Updated cell {cell_number}:</Text>
      <Box marginLeft={2}>
        <HighlightedCode code={new_source} language={language} />
      </Box>
    </Box>
  )
}
```

**특징:**
- 에러는 빨간색으로 표시
- 성공 시 셀 번호와 함께 업데이트된 코드 표시
- 언어별 구문 강조 (`HighlightedCode` 컴포넌트)

---

## 3. NotebookReadTool - Jupyter 노트북 읽기

### 3.1 개요

NotebookReadTool은 Jupyter 노트북 파일을 읽고 모든 셀과 출력을 추출하는 도구입니다.

**파일 위치:**
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/NotebookReadTool/NotebookReadTool.tsx`

### 3.2 입력 스키마

```typescript
const inputSchema = z.strictObject({
  notebook_path: z
    .string()
    .describe('The absolute path to the Jupyter notebook file to read (must be absolute, not relative)'),
})
```

**파라미터:**
- `notebook_path`: 노트북 파일의 절대 경로 (필수)

### 3.3 출력 데이터 구조

```typescript
type NotebookCellSource = {
  cell: number                                  // 셀 인덱스
  cellType: NotebookCellType                    // 'code' | 'markdown'
  source: string                                // 셀 소스 코드/텍스트
  language: string                              // 프로그래밍 언어 (예: 'python')
  execution_count?: number                      // 실행 카운트 (코드 셀만)
  outputs?: NotebookCellSourceOutput[]          // 셀 출력 (코드 셀만)
}

type NotebookCellSourceOutput = {
  output_type: string                           // 'stream', 'execute_result', 'display_data', 'error'
  text?: string                                 // 텍스트 출력
  image?: NotebookOutputImage                   // 이미지 출력
}

type NotebookOutputImage = {
  image_data: string                            // Base64 인코딩된 이미지
  media_type: 'image/png' | 'image/jpeg'        // 이미지 타입
}
```

### 3.4 셀 처리 로직

```typescript
function processCell(
  cell: NotebookCell,
  index: number,
  language: string,
): NotebookCellSource {
  const cellData: NotebookCellSource = {
    cell: index,
    cellType: cell.cell_type,
    source: Array.isArray(cell.source) ? cell.source.join('') : cell.source,
    language,
    execution_count: cell.execution_count,
  }

  if (cell.outputs?.length) {
    cellData.outputs = cell.outputs.map(processOutput)
  }

  return cellData
}
```

**처리 과정:**
1. 셀 소스가 배열이면 문자열로 결합
2. 메타데이터에서 언어 정보 추출 (기본값: 'python')
3. 코드 셀의 출력 처리
4. 실행 카운트 포함

### 3.5 출력 타입별 처리

```typescript
function processOutput(output: NotebookCellOutput) {
  switch (output.output_type) {
    case 'stream':
      return {
        output_type: output.output_type,
        text: processOutputText(output.text),
      }

    case 'execute_result':
    case 'display_data':
      return {
        output_type: output.output_type,
        text: processOutputText(output.data?.['text/plain']),
        image: output.data && extractImage(output.data),
      }

    case 'error':
      return {
        output_type: output.output_type,
        text: processOutputText(
          `${output.ename}: ${output.evalue}\n${output.traceback.join('\n')}`,
        ),
      }
  }
}
```

**출력 타입:**

1. **stream**: 표준 출력/에러 스트림
   ```python
   print("Hello World")  # stream output
   ```

2. **execute_result**: 실행 결과값
   ```python
   2 + 2  # execute_result: 4
   ```

3. **display_data**: 표시 데이터 (차트, 이미지 등)
   ```python
   plt.plot([1, 2, 3])  # display_data with image
   ```

4. **error**: 에러 출력
   ```python
   1 / 0  # error: ZeroDivisionError
   ```

### 3.6 이미지 추출

```typescript
function extractImage(
  data: Record<string, unknown>,
): NotebookOutputImage | undefined {
  if (typeof data['image/png'] === 'string') {
    return {
      image_data: data['image/png'] as string,
      media_type: 'image/png',
    }
  }
  if (typeof data['image/jpeg'] === 'string') {
    return {
      image_data: data['image/jpeg'] as string,
      media_type: 'image/jpeg',
    }
  }
  return undefined
}
```

**지원 이미지 형식:**
- PNG (image/png)
- JPEG (image/jpeg)
- Base64 인코딩으로 저장

### 3.7 출력 텍스트 처리

```typescript
function processOutputText(text: string | string[] | undefined): string {
  if (!text) return ''
  const rawText = Array.isArray(text) ? text.join('') : text
  const { truncatedContent } = formatOutput(rawText)
  return truncatedContent
}
```

**특징:**
- 배열 형태의 텍스트를 문자열로 결합
- `formatOutput` 함수로 30000자 제한 적용
- 긴 출력은 자동으로 잘라내기

### 3.8 어시스턴트용 렌더링

```typescript
renderResultForAssistant(data: NotebookCellSource[]) {
  return data.map((cell, index) => {
    let content = `Cell ${index + 1} (${cell.cellType}):\n${cell.source}`

    if (cell.outputs && cell.outputs.length > 0) {
      const outputText = cell.outputs
        .map(output => output.text)
        .filter(Boolean)
        .join('\n')

      if (outputText) {
        content += `\nOutput:\n${outputText}`
      }
    }

    return content
  }).join('\n\n')
}
```

**포맷:**
```
Cell 1 (code):
import numpy as np
print("Hello")

Output:
Hello

Cell 2 (markdown):
# Introduction
This is a markdown cell.
```

### 3.9 유사 파일 제안

```typescript
if (!existsSync(fullFilePath)) {
  const similarFilename = findSimilarFile(fullFilePath)
  let message = 'File does not exist.'

  if (similarFilename) {
    message += ` Did you mean ${similarFilename}?`
  }

  return { result: false, message }
}
```

**기능:**
- 파일이 존재하지 않으면 유사한 파일명 검색
- 오타나 확장자 실수 방지
- 사용자 경험 개선

---

## 4. TodoWriteTool - 작업 관리 시스템

### 4.1 개요

TodoWriteTool은 복잡한 작업을 추적하고 관리하는 도구입니다. AI 에이전트가 여러 단계의 작업을 체계적으로 진행하도록 돕습니다.

**파일 위치:**
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/TodoWriteTool/TodoWriteTool.tsx`

### 4.2 입력 스키마

```typescript
const TodoItemSchema = z.object({
  content: z.string().min(1).describe('The task description or content'),
  status: z
    .enum(['pending', 'in_progress', 'completed'])
    .describe('Current status of the task'),
  priority: z
    .enum(['high', 'medium', 'low'])
    .describe('Priority level of the task'),
  id: z.string().min(1).describe('Unique identifier for the task'),
})

const inputSchema = z.strictObject({
  todos: z.array(TodoItemSchema).describe('The updated todo list'),
})
```

**파라미터:**
- `todos`: TodoItem 배열
  - `content`: 작업 설명 (필수, 최소 1자)
  - `status`: 상태 ('pending', 'in_progress', 'completed')
  - `priority`: 우선순위 ('high', 'medium', 'low')
  - `id`: 고유 식별자 (필수)

### 4.3 사용 시나리오

프롬프트에 명시된 사용 기준:

```markdown
## When to Use This Tool

1. **Complex multi-step tasks** - 3개 이상의 단계가 필요한 작업
2. **Non-trivial and complex tasks** - 신중한 계획이나 여러 작업이 필요한 작업
3. **User explicitly requests todo list** - 사용자가 명시적으로 요청
4. **User provides multiple tasks** - 번호나 쉼표로 구분된 작업 목록
5. **After receiving new instructions** - 새로운 지시 받은 후 즉시 캡처
6. **When you start working on a task** - 작업 시작 전 in_progress로 표시
7. **After completing a task** - 완료 후 즉시 completed로 표시
```

**사용하지 말아야 할 경우:**
```markdown
1. 단일하고 직관적인 작업
2. 추적이 이점을 제공하지 않는 간단한 작업
3. 3단계 미만의 간단한 작업
4. 대화형 또는 정보 제공 목적의 작업
```

### 4.4 검증 로직

```typescript
function validateTodos(todos: TodoItem[]): ValidationResult {
  // 1. 중복 ID 검사
  const ids = todos.map(todo => todo.id)
  const uniqueIds = new Set(ids)
  if (ids.length !== uniqueIds.size) {
    return {
      result: false,
      errorCode: 1,
      message: 'Duplicate todo IDs found',
      meta: {
        duplicateIds: ids.filter((id, index) => ids.indexOf(id) !== index),
      },
    }
  }

  // 2. in_progress 작업은 하나만 허용
  const inProgressTasks = todos.filter(todo => todo.status === 'in_progress')
  if (inProgressTasks.length > 1) {
    return {
      result: false,
      errorCode: 2,
      message: 'Only one task can be in_progress at a time',
      meta: { inProgressTaskIds: inProgressTasks.map(t => t.id) },
    }
  }

  // 3. 각 todo 검증
  for (const todo of todos) {
    if (!todo.content?.trim()) {
      return {
        result: false,
        errorCode: 3,
        message: `Todo with ID "${todo.id}" has empty content`,
        meta: { todoId: todo.id },
      }
    }
    // ... 상태 및 우선순위 검증
  }

  return { result: true }
}
```

**핵심 규칙:**
1. **고유 ID**: 모든 todo는 고유한 ID를 가져야 함
2. **단일 진행 작업**: 한 번에 하나의 작업만 in_progress 상태 가능
3. **필수 내용**: content 필드는 비어있을 수 없음
4. **유효한 상태**: pending, in_progress, completed 중 하나
5. **유효한 우선순위**: high, medium, low 중 하나

### 4.5 상태 관리

#### 4.5.1 Todo 상태 전환

```
pending (대기 중)
    ↓
in_progress (진행 중)  ← 한 번에 하나만
    ↓
completed (완료)
```

**권장 워크플로우:**
```typescript
// 1. 작업 목록 생성
[
  { id: '1', content: '요구사항 분석', status: 'pending', priority: 'high' },
  { id: '2', content: '코드 구현', status: 'pending', priority: 'high' },
  { id: '3', content: '테스트 작성', status: 'pending', priority: 'medium' },
]

// 2. 첫 번째 작업 시작
[
  { id: '1', content: '요구사항 분석', status: 'in_progress', priority: 'high' },
  { id: '2', content: '코드 구현', status: 'pending', priority: 'high' },
  { id: '3', content: '테스트 작성', status: 'pending', priority: 'medium' },
]

// 3. 첫 번째 완료 후 다음 작업 시작
[
  { id: '1', content: '요구사항 분석', status: 'completed', priority: 'high' },
  { id: '2', content: '코드 구현', status: 'in_progress', priority: 'high' },
  { id: '3', content: '테스트 작성', status: 'pending', priority: 'medium' },
]
```

#### 4.5.2 완료 조건

프롬프트에서 명시한 완료 기준:

```markdown
ONLY mark a task as completed when you have FULLY accomplished it

Never mark a task as completed if:
- Tests are failing
- Implementation is partial
- You encountered unresolved errors
- You couldn't find necessary files or dependencies
```

### 4.6 에이전트별 Todo 격리

```typescript
async *call({ todos }: z.infer<typeof inputSchema>, context) {
  // 에이전트 ID 가져오기
  const agentId = context?.agentId

  // Todo 파일 감시 시작
  if (agentId) {
    startWatchingTodoFile(agentId)
  }

  // 이전 todos 가져오기 (에이전트별)
  const previousTodos = getTodos(agentId)

  // Todos 업데이트 (에이전트별)
  setTodos(todoItems, agentId)
}
```

**격리 메커니즘:**
- 각 에이전트(Task)는 독립적인 todo 리스트 보유
- `agentId`를 키로 사용하여 분리된 저장소 관리
- 메인 대화와 서브태스크의 todo가 섞이지 않음

### 4.7 변경 추적 및 이벤트

```typescript
// 변경 사항 감지
const hasChanged = JSON.stringify(previousTodos) !== JSON.stringify(todoItems)

if (hasChanged) {
  emitReminderEvent('todo:changed', {
    previousTodos,
    newTodos: todoItems,
    timestamp: Date.now(),
    agentId: agentId || 'default',
    changeType:
      todoItems.length > previousTodos.length
        ? 'added'
        : todoItems.length < previousTodos.length
          ? 'removed'
          : 'modified',
  })
}
```

**이벤트 타입:**
- `added`: 새로운 todo 추가
- `removed`: todo 삭제
- `modified`: 기존 todo 수정

### 4.8 UI 렌더링

```typescript
renderToolResultMessage(output) {
  const currentTodos = getTodos()

  if (currentTodos.length === 0) {
    return (
      <Box flexDirection="column" width="100%">
        <Box flexDirection="row">
          <Text color="#6B7280">&nbsp;&nbsp;⎿ &nbsp;</Text>
          <Text color="#9CA3AF">No todos currently</Text>
        </Box>
      </Box>
    )
  }

  // 정렬: [completed, in_progress, pending]
  const sortedTodos = [...currentTodos].sort((a, b) => {
    const order = ['completed', 'in_progress', 'pending']
    return (
      order.indexOf(a.status) - order.indexOf(b.status) ||
      a.content.localeCompare(b.content)
    )
  })

  // 다음 대기 중인 작업 찾기
  const nextPendingIndex = sortedTodos.findIndex(todo => todo.status === 'pending')

  return (
    <Box flexDirection="column" width="100%">
      {sortedTodos.map((todo: TodoItem, index: number) => {
        let checkbox: string
        let textColor: string
        let isBold = false
        let isStrikethrough = false

        if (todo.status === 'completed') {
          checkbox = '☒'
          textColor = '#6B7280'  // 회색
          isStrikethrough = true
        } else if (todo.status === 'in_progress') {
          checkbox = '☐'
          textColor = '#10B981'  // 녹색
          isBold = true
        } else if (todo.status === 'pending') {
          checkbox = '☐'
          if (index === nextPendingIndex) {
            textColor = '#8B5CF6'  // 보라색 (다음 작업 강조)
            isBold = true
          } else {
            textColor = '#9CA3AF'  // 연한 회색
          }
        }

        return (
          <Box key={todo.id || index} flexDirection="row">
            <Text color="#6B7280">&nbsp;&nbsp;⎿ &nbsp;</Text>
            <Box flexDirection="row" flexGrow={1}>
              <Text color={textColor} bold={isBold} strikethrough={isStrikethrough}>
                {checkbox}
              </Text>
              <Text> </Text>
              <Text color={textColor} bold={isBold} strikethrough={isStrikethrough}>
                {todo.content}
              </Text>
            </Box>
          </Box>
        )
      })}
    </Box>
  )
}
```

**시각적 표현:**

```
⎿ ☒ 요구사항 분석              (회색, 취소선 - completed)
⎿ ☐ 코드 구현                  (녹색, 굵게 - in_progress)
⎿ ☐ 테스트 작성                (보라색, 굵게 - 다음 pending)
⎿ ☐ 문서화                     (연한 회색 - 나머지 pending)
```

**색상 코드:**
- `#6B7280` (회색): 완료된 작업
- `#10B981` (녹색): 진행 중인 작업
- `#8B5CF6` (보라색): 다음에 할 작업 (첫 번째 pending)
- `#9CA3AF` (연한 회색): 나머지 대기 중인 작업

### 4.9 요약 생성

```typescript
function generateTodoSummary(todos: TodoItem[]): string {
  const stats = {
    total: todos.length,
    pending: todos.filter(t => t.status === 'pending').length,
    inProgress: todos.filter(t => t.status === 'in_progress').length,
    completed: todos.filter(t => t.status === 'completed').length,
  }

  let summary = `Updated ${stats.total} todo(s)`
  if (stats.total > 0) {
    summary += ` (${stats.pending} pending, ${stats.inProgress} in progress, ${stats.completed} completed)`
  }
  summary += '. Continue tracking your progress with the todo list.'

  return summary
}
```

**출력 예시:**
```
Updated 5 todo(s) (2 pending, 1 in progress, 2 completed). Continue tracking your progress with the todo list.
```

### 4.10 파일 감시

```typescript
// Todo 파일 변경 감시 시작
if (agentId) {
  startWatchingTodoFile(agentId)
}
```

**기능:**
- 파일 시스템에서 todo 파일 변경 감지
- 외부 편집 감지 및 동기화
- 파일 신선도 추적 시스템과 통합

---

## 5. MemoryTool - 컨텍스트 메모리 관리

### 5.1 개요

MemoryTool은 에이전트가 세션 간 정보를 저장하고 검색할 수 있도록 하는 영구 메모리 시스템입니다. 현재는 비활성화 상태이지만 향후 기능을 위한 인프라가 구축되어 있습니다.

**파일 위치:**
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/MemoryReadTool/MemoryReadTool.tsx`
- `/home/user/analysis_claude_code/kode-cli-analysis/src/tools/MemoryWriteTool/MemoryWriteTool.tsx`

### 5.2 MemoryReadTool

#### 5.2.1 입력 스키마

```typescript
const inputSchema = z.strictObject({
  file_path: z
    .string()
    .optional()
    .describe('Optional path to a specific memory file to read'),
})
```

**파라미터:**
- `file_path`: 읽을 메모리 파일 경로 (선택)
  - 지정하지 않으면 전체 메모리 인덱스 반환

#### 5.2.2 메모리 디렉토리 구조

```
{MEMORY_DIR}/
  └── agents/
      ├── {agentId}/
      │   ├── index.md           # 에이전트 메모리 인덱스
      │   ├── context.md         # 컨텍스트 정보
      │   └── notes.md           # 노트
      └── default/               # 기본 에이전트
          └── index.md
```

**에이전트별 격리:**
```typescript
const agentId = resolveAgentId(context?.agentId)
const agentMemoryDir = join(MEMORY_DIR, 'agents', agentId)
mkdirSync(agentMemoryDir, { recursive: true })
```

#### 5.2.3 특정 파일 읽기

```typescript
// 특정 파일 요청 시
if (file_path) {
  const fullPath = join(agentMemoryDir, file_path)

  // 경로 검증 (디렉토리 탈출 방지)
  if (!fullPath.startsWith(agentMemoryDir)) {
    throw new Error('Invalid memory file path')
  }

  if (!existsSync(fullPath)) {
    throw new Error('Memory file does not exist')
  }

  const content = readFileSync(fullPath, 'utf-8')

  yield {
    type: 'result',
    data: { content },
    resultForAssistant: this.renderResultForAssistant({ content }),
  }
  return
}
```

**보안:**
- 경로 검증으로 디렉토리 탈출 공격 방지
- 에이전트 메모리 디렉토리 내부로만 접근 제한

#### 5.2.4 인덱스 및 파일 목록 반환

```typescript
// 전체 메모리 인덱스 반환
const files = readdirSync(agentMemoryDir, { recursive: true })
  .map(f => join(agentMemoryDir, f.toString()))
  .filter(f => !lstatSync(f).isDirectory())
  .map(f => `- ${f}`)
  .join('\n')

const indexPath = join(agentMemoryDir, 'index.md')
const index = existsSync(indexPath) ? readFileSync(indexPath, 'utf-8') : ''

const content = `Here are the contents of the agent memory file, \`${indexPath}\`:
'''
${index}
'''

Files in the agent memory directory:
${files}`
```

**출력 형식:**
```markdown
Here are the contents of the agent memory file, `/path/to/index.md`:
'''
# Agent Memory Index
- Important context about the project
- User preferences
'''

Files in the agent memory directory:
- /path/to/agents/agent123/index.md
- /path/to/agents/agent123/context.md
- /path/to/agents/agent123/notes.md
```

### 5.3 MemoryWriteTool

#### 5.3.1 입력 스키마

```typescript
const inputSchema = z.strictObject({
  file_path: z.string().describe('Path to the memory file to write'),
  content: z.string().describe('Content to write to the file'),
})
```

**파라미터:**
- `file_path`: 쓸 메모리 파일 경로 (필수)
- `content`: 파일에 쓸 내용 (필수)

#### 5.3.2 파일 쓰기

```typescript
async *call({ file_path, content }, context) {
  const agentId = resolveAgentId(context?.agentId)
  const agentMemoryDir = join(MEMORY_DIR, 'agents', agentId)
  const fullPath = join(agentMemoryDir, file_path)

  // 디렉토리 생성 (재귀적)
  mkdirSync(dirname(fullPath), { recursive: true })

  // 파일 쓰기
  writeFileSync(fullPath, content, 'utf-8')

  // 파일 편집 기록
  recordFileEdit(fullPath, content)

  yield {
    type: 'result',
    data: 'Saved',
    resultForAssistant: 'Saved',
  }
}
```

**특징:**
- 중첩된 디렉토리 자동 생성
- UTF-8 인코딩 사용
- 파일 변경 추적 시스템과 통합

#### 5.3.3 입력 검증

```typescript
async validateInput({ file_path }, context) {
  const agentId = resolveAgentId(context?.agentId)
  const agentMemoryDir = join(MEMORY_DIR, 'agents', agentId)
  const fullPath = join(agentMemoryDir, file_path)

  // 경로 검증 (디렉토리 탈출 방지)
  if (!fullPath.startsWith(agentMemoryDir)) {
    return { result: false, message: 'Invalid memory file path' }
  }

  return { result: true }
}
```

### 5.4 활성화 상태

```typescript
async isEnabled() {
  // TODO: Gate with a setting or feature flag
  return false
}
```

**현재 상태:**
- 두 도구 모두 `isEnabled() = false`로 비활성화
- 향후 설정이나 기능 플래그로 활성화 예정
- 인프라는 완전히 구현되어 있어 쉽게 활성화 가능

### 5.5 프롬프트 (MCP 오버라이드)

```typescript
// prompt.ts
export const PROMPT = ''
export const DESCRIPTION = ''
```

**주석:**
```typescript
// Actual prompt and description are overridden in mcpClient.ts
```

**의미:**
- 실제 프롬프트와 설명은 MCP(Model Context Protocol) 클라이언트에서 동적으로 설정됨
- MCP를 통한 확장 가능한 도구 시스템의 일부

### 5.6 사용 사례 (활성화 시)

```typescript
// 사용자 선호도 저장
await MemoryWrite({
  file_path: 'preferences.md',
  content: `# User Preferences
- Preferred language: TypeScript
- Code style: 2 spaces
- Testing framework: Jest
`
})

// 프로젝트 컨텍스트 저장
await MemoryWrite({
  file_path: 'project_context.md',
  content: `# Project Context
- Framework: React + Ink
- Build tool: Bun
- Main entry: src/entrypoints/cli.tsx
`
})

// 나중에 검색
const preferences = await MemoryRead({ file_path: 'preferences.md' })
```

### 5.7 동시성 안전성

```typescript
// MemoryReadTool
isConcurrencySafe() {
  return true  // 읽기 전용, 동시 실행 안전
}

// MemoryWriteTool
isConcurrencySafe() {
  return false  // 상태 수정, 동시 실행 불안전
}
```

---

## 6. 공통 패턴 및 설계 원칙

### 6.1 도구 인터페이스 일관성

모든 도구는 `Tool` 인터페이스를 구현합니다:

```typescript
interface Tool<In, Out> {
  name: string
  description: () => Promise<string> | string
  prompt: () => Promise<string> | string
  inputSchema: ZodSchema<In>
  userFacingName: () => string
  isEnabled: () => Promise<boolean> | boolean
  isReadOnly: () => boolean
  isConcurrencySafe: () => boolean
  needsPermissions: (input: In) => boolean
  validateInput: (input: In, context?) => Promise<ValidationResult> | ValidationResult
  call: (input: In, context?) => AsyncGenerator<ToolResult<Out>>
  renderToolUseMessage: (input: In, options?) => string | React.JSX.Element
  renderToolUseRejectedMessage: () => React.JSX.Element
  renderToolResultMessage: (output: Out) => React.JSX.Element
  renderResultForAssistant: (output: Out) => string
}
```

### 6.2 검증 레이어

모든 도구는 다층 검증을 수행합니다:

1. **스키마 검증** (Zod)
   ```typescript
   const inputSchema = z.strictObject({ ... })
   ```

2. **커스텀 검증** (`validateInput`)
   ```typescript
   async validateInput(input) {
     if (!existsSync(input.file_path)) {
       return { result: false, message: 'File does not exist' }
     }
     return { result: true }
   }
   ```

3. **권한 검증** (`needsPermissions`)
   ```typescript
   needsPermissions(input) {
     return !hasWritePermission(input.file_path)
   }
   ```

### 6.3 에러 처리 패턴

```typescript
try {
  // 도구 실행 로직
  const result = await performOperation()

  yield {
    type: 'result',
    data: result,
    resultForAssistant: this.renderResultForAssistant(result),
  }
} catch (error) {
  const errorMessage = error instanceof Error
    ? error.message
    : 'Unknown error occurred'

  const errorData = {
    ...defaultData,
    error: errorMessage,
  }

  yield {
    type: 'result',
    data: errorData,
    resultForAssistant: this.renderResultForAssistant(errorData),
  }
}
```

### 6.4 파일 시스템 안전성

#### 6.4.1 절대 경로 사용

```typescript
const fullPath = isAbsolute(input_path)
  ? input_path
  : resolve(getCwd(), input_path)
```

#### 6.4.2 경로 검증

```typescript
// 디렉토리 탈출 방지
if (!fullPath.startsWith(allowedDirectory)) {
  return { result: false, message: 'Invalid path' }
}
```

#### 6.4.3 인코딩 및 줄바꿈 보존

```typescript
const enc = detectFileEncoding(fullPath)
const endings = detectLineEndings(fullPath)
writeTextContent(fullPath, content, enc, endings!)
```

### 6.5 이벤트 기반 추적

```typescript
// 파일 편집 추적
recordFileEdit(fullPath, content)

// 시스템 리마인더 이벤트
emitReminderEvent('file:edited', {
  filePath: fullPath,
  timestamp: Date.now(),
  operation: 'edit',
})
```

**추적 대상:**
- 파일 편집 (`recordFileEdit`)
- Todo 변경 (`emitReminderEvent('todo:changed')`)
- 에러 발생 (`emitReminderEvent('todo:error')`)

### 6.6 UI 렌더링 원칙

#### 6.6.1 일관된 프리픽스

```tsx
<Box flexDirection="row">
  <Text>&nbsp;&nbsp;⎿ &nbsp;</Text>
  <Text>{content}</Text>
</Box>
```

#### 6.6.2 색상 테마 사용

```tsx
import { getTheme } from '@utils/theme'

<Text color={getTheme().error}>Error message</Text>
<Text color={getTheme().success}>Success message</Text>
<Text color={getTheme().secondaryText}>Secondary info</Text>
```

#### 6.6.3 구문 강조

```tsx
import { HighlightedCode } from '@components/HighlightedCode'

<HighlightedCode code={sourceCode} language={language} />
```

### 6.7 동시성 제어

```typescript
// 읽기 전용 도구
isConcurrencySafe() {
  return true  // 안전하게 병렬 실행 가능
}

// 상태 수정 도구
isConcurrencySafe() {
  return false  // 순차 실행 필요
}
```

**영향:**
- `true`: 여러 도구 호출을 동시에 실행 가능
- `false`: 한 번에 하나씩만 실행

---

## 7. 성능 최적화

### 7.1 출력 길이 제한

```typescript
const MAX_OUTPUT_LENGTH = 30000  // BashTool
const MAX_RENDERED_LINES = 5     // OutputLine
```

**이유:**
- 터미널 UI 성능 유지
- 네트워크 대역폭 절약
- 응답 시간 단축

### 7.2 프롬프트 캐싱

```typescript
const response = await queryQuick({
  systemPrompt: [...],
  userPrompt: `Command: ${command}\nOutput: ${output}`,
  enablePromptCaching: true,  // 캐싱 활성화
})
```

**효과:**
- 반복적인 AI 호출 비용 감소
- 응답 속도 향상

### 7.3 변경 감지 최적화

```typescript
// 변경 사항이 없으면 이벤트 발생 안 함
const hasChanged = JSON.stringify(previousTodos) !== JSON.stringify(todoItems)
if (hasChanged) {
  emitReminderEvent('todo:changed', {...})
}
```

### 7.4 파일 감시 지연 로딩

```typescript
// 파일 감시는 필요할 때만 시작
if (agentId) {
  startWatchingTodoFile(agentId)
}
```

### 7.5 비동기 파일 경로 업데이트

```typescript
// 메인 스레드를 블록하지 않음
if (process.env.NODE_ENV !== 'test') {
  getCommandFilePaths(command, stdout).then(filePaths => {
    for (const filePath of filePaths) {
      try {
        readFileTimestamps[fullFilePath] = statSync(fullFilePath).mtimeMs
      } catch (e) {
        logError(e)
      }
    }
  })
}
```

---

## 8. 보안 고려사항

### 8.1 명령어 차단 (BashTool)

- 네트워크 요청 명령어 차단
- 브라우저 실행 방지
- 별칭 설정 방지

### 8.2 경로 검증

- 절대 경로 강제
- 디렉토리 탈출 방지
- 샌드박스 환경 유지

### 8.3 권한 시스템

```typescript
needsPermissions(input) {
  return !hasWritePermission(input.file_path)
}
```

- 파일 쓰기 전 권한 확인
- 사용자 승인 필요

### 8.4 입력 검증

- Zod 스키마를 통한 타입 안전성
- 커스텀 검증 로직
- 에러 코드와 메타데이터 제공

### 8.5 에이전트 격리

- 에이전트별 메모리 디렉토리
- 에이전트별 Todo 리스트
- 크로스 에이전트 접근 방지

---

## 9. 확장성 설계

### 9.1 MCP 통합

Memory 도구들은 MCP(Model Context Protocol)를 통해 프롬프트와 설명을 동적으로 로드합니다.

```typescript
// prompt.ts
// Actual prompt and description are overridden in mcpClient.ts
export const PROMPT = ''
export const DESCRIPTION = ''
```

### 9.2 플러그인 아키텍처

도구들은 독립적으로 활성화/비활성화 가능:

```typescript
async isEnabled() {
  // 설정이나 기능 플래그로 제어
  return false
}
```

### 9.3 컨텍스트 전달

모든 도구 호출은 컨텍스트를 받습니다:

```typescript
async *call(input, context) {
  const agentId = context?.agentId
  const abortController = context?.abortController
  // ...
}
```

**컨텍스트 정보:**
- `agentId`: 현재 에이전트 식별자
- `abortController`: 작업 취소 컨트롤러
- `readFileTimestamps`: 파일 읽기 타임스탬프

### 9.4 Generator 패턴

도구들은 AsyncGenerator를 사용하여 스트리밍 결과 지원:

```typescript
async *call(input, context) {
  // 중간 결과 yield
  yield { type: 'partial', data: partialResult }

  // 최종 결과 yield
  yield {
    type: 'result',
    data: finalResult,
    resultForAssistant: this.renderResultForAssistant(finalResult),
  }
}
```

---

## 10. 결론

Kode-cli의 Bash 및 시스템 도구들은 다음과 같은 특징을 가집니다:

### 10.1 핵심 강점

1. **보안 우선**: 다층 검증, 명령어 차단, 경로 검증
2. **사용자 경험**: 직관적인 UI, 색상 코딩, 구문 강조
3. **성능 최적화**: 출력 제한, 프롬프트 캐싱, 비동기 처리
4. **확장성**: MCP 통합, 플러그인 아키텍처, 컨텍스트 전달
5. **안정성**: 에러 처리, 입력 검증, 타입 안전성

### 10.2 혁신적인 기능

1. **AI 기반 파일 경로 추출**: 정규식 대신 AI 모델 사용
2. **영구 셸 세션**: 명령어 간 상태 유지
3. **에이전트별 격리**: Todo 및 메모리 분리
4. **동적 프롬프트**: MCP를 통한 도구 확장
5. **실시간 추적**: 파일 변경 감지 및 이벤트 시스템

### 10.3 설계 철학

- **일관성**: 모든 도구가 동일한 인터페이스 구현
- **안전성**: 보안과 검증이 최우선
- **확장성**: 새로운 도구 추가가 용이
- **사용자 중심**: 명확한 피드백과 에러 메시지
- **성능**: 대규모 출력도 원활하게 처리

### 10.4 향후 개선 방향

1. **Memory 도구 활성화**: 세션 간 컨텍스트 유지
2. **더 많은 노트북 지원**: R, Julia 등 다른 커널
3. **고급 Todo 기능**: 의존성, 태그, 마감일
4. **실시간 협업**: 여러 에이전트 간 Todo 공유
5. **플러그인 생태계**: 커뮤니티 도구 지원

이 문서는 Kode-cli의 Bash 및 시스템 도구들에 대한 포괄적인 분석을 제공합니다. 각 도구의 구현 세부사항, 보안 메커니즘, UI 렌더링, 그리고 실제 사용 예제를 포함하고 있어 개발자들이 시스템을 이해하고 확장하는 데 도움이 될 것입니다.
