# 순차형 타이핑 커서 모양 변경 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task.

**Goal:** 글자가 작성될 때마다 기존 입력 위치 커서의 모양을 고정된 순서로 변경하고 실제 입력·자동 타이핑 모두에서 확인한다.

**Architecture:** `index.html`의 기존 `#cursorAnchor` 하나를 유지한다. CSS는 동일한 기준 박스 안에서 네 가지 커서 모양을 표현하고, JavaScript의 `cursorShapeIndex`가 작성 이벤트마다 다음 상태를 선택해 `data-cursor-shape`를 갱신한다. 기존 텍스트 렌더링과 타격 효과의 위치 측정은 그대로 사용한다.

**Tech Stack:** 단일 HTML, 인라인 CSS, 바닐라 JavaScript, Python 표준 HTTP 서버

## Global Constraints

- 수정 파일: `index.html`
- 기존 `#cursorAnchor` 요소를 유지한다.
- 외부 라이브러리, 빌드 도구, 별도 런타임 의존성은 추가하지 않는다.
- 커서 모양 변경은 실제 입력과 자동 타이핑에 공통 적용한다.
- 커밋과 push는 사용자의 화면 확인 전에는 하지 않는다.

## 현재 맥락

- 작업 루트: `/home/pne/.hermes/profiles/sam/workspace/platen`
- 작업 브랜치: `feat/sequential-cursor-shapes`
- 기존 `#cursorAnchor`는 inline-block 2px 세로 막대다.
- `updateScreen()`은 `committedText`, `composingText`, `cursorAnchor`를 갱신한다.
- `triggerTypeImpact()`은 조합 중이면 `composingEl`, 아니면 `cursorAnchor`의 `getBoundingClientRect()`를 사용한다.
- 프로젝트는 `index.html` 단일 파일이며 별도 테스트 프레임워크가 없다.

## 상태 순서와 인터페이스

- 상태 배열: `cursorShapes = ["bar", "underline", "box", "dot"]`
- 상태 인덱스: `cursorShapeIndex: number`
- 상태 갱신 함수: `advanceCursorShape(): void`
  - `cursorShapeIndex = (cursorShapeIndex + 1) % cursorShapes.length`
  - `cursorAnchor.dataset.cursorShape = cursorShapes[cursorShapeIndex]`
- 초기 상태: `cursorShapeIndex = 0`, `data-cursor-shape="bar"`
- 작성 이벤트 연결 함수: `handleTypingFeedback(): void`
  - 기존 작성 처리 직후 `advanceCursorShape()`를 호출한다.
  - 실제 키 입력과 자동 타이핑이 같은 처리 경로를 사용하도록 기존 공통 지점에만 연결한다.

### Task 1: 커서 상태 CSS와 상태 갱신 함수 추가

**Objective:** 기존 커서 위치를 보존하면서 네 가지 모양을 표현하고, 커서 상태를 순환할 수 있게 만든다.

**Files:**
- Modify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`의 `#cursorAnchor` CSS 및 화면 상태 초기화 구간

**Steps:**

- [ ] `#cursorAnchor`의 기존 inline-block·기준 크기 역할을 유지하면서 `position: relative`, 배경·테두리 전환에 필요한 스타일을 추가한다.
- [ ] `data-cursor-shape` 값에 따라 아래 모양을 정의한다.
  - `bar`: 기존 2px 세로 막대
  - `underline`: 커서 박스 아래쪽 2px 가로선
  - `box`: 2px 테두리 사각형
  - `dot`: 6px 원형 점
- [ ] 모양 전환에 `transition`을 사용하되 `prefers-reduced-motion: reduce`에서 전환 시간을 `0ms`로 만든다.
- [ ] `cursorShapes`, `cursorShapeIndex`, `advanceCursorShape()`를 기존 DOM 참조 뒤에 추가한다.
- [ ] 초기화 시 `cursorAnchor.dataset.cursorShape = cursorShapes[cursorShapeIndex]`를 설정한다.

**Verification:**

- [ ] `python3 - <<'PY'` 스크립트로 `index.html`에 `cursorShapes`, `advanceCursorShape`, 네 가지 `data-cursor-shape` 선택자가 모두 존재하는지 검사한다.
- [ ] 실행 결과가 누락 없이 통과해야 한다.

### Task 2: 작성 이벤트에 커서 순환 연결

**Objective:** 실제 입력과 자동 타이핑의 공통 작성 경로에서 글자가 작성될 때마다 커서 상태를 한 단계 진행한다.

**Files:**
- Modify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`의 키 입력 및 자동 타이핑 처리 구간

**Interface consumed:** Task 1의 `advanceCursorShape(): void`

**Steps:**

- [ ] 실제 입력에서 문자·자모·Enter·공백이 처리되는 공통 지점을 찾아 작성이 성공한 경우 `advanceCursorShape()`를 호출한다.
- [ ] 자동 타이핑이 실제 입력과 같은 처리 함수를 호출하는지 확인하고, 별도 호출 경로라면 동일한 공통 작성 지점으로 합친다.
- [ ] 백스페이스·리셋 동작은 기존 의미를 유지하며, 커서 모양 순환을 초기화하지 않는다.
- [ ] 한글 조합 중에도 자모 입력마다 모양이 다음 상태로 바뀌게 한다.
- [ ] 타격 효과는 기존 호출 순서와 위치 계산을 유지해 커서 모양 변경으로 대상 노드 선택이 바뀌지 않도록 한다.

**Verification:**

- [ ] JavaScript 구문 검사: `node --check <(sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d')`를 실행한다. 프로세스 치환을 지원하지 않는 경우 임시 파일을 만들어 같은 내용을 검사한다.
- [ ] 소스에서 `advanceCursorShape()`가 선언부 외에 실제 작성 처리 경로에서 호출되는지 확인한다.

### Task 3: 로컬 서버 실행 및 기능 검증

**Objective:** 수정된 단일 HTML을 포트 8000으로 제공하고 정적·동작 검증 결과를 확인한다.

**Files:**
- Verify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`

**Steps:**

- [ ] 프로젝트 루트에서 `python3 -m http.server 8000`을 실행한다.
- [ ] 서버 프로세스가 포트 8000에서 수신 중인지 확인한다.
- [ ] `curl -I http://127.0.0.1:8000/index.html`로 HTTP 200 응답을 확인한다.
- [ ] Windows 브라우저 접속 주소로 `http://localhost:8000`을 안내한다.
- [ ] 브라우저에서 실제 입력 시 바 → 밑줄 → 사각형 → 점 순서가 반복되는지 확인한다.
- [ ] 자동 타이핑에서도 같은 순서가 적용되는지 확인한다.
- [ ] 한글 조합, 백스페이스, 리셋, 램프 슬라이더, 램프 토글, 타격 효과, 타자 소리의 기존 동작을 확인한다.
- [ ] 마지막에 `git status --short --branch`로 변경 파일을 확인하고 커밋·push는 실행하지 않는다.

**Expected result:**

- 서버가 `0.0.0.0:8000`에서 실행된다.
- `curl -I`가 `HTTP/1.0 200 OK` 또는 동등한 200 응답을 반환한다.
- 변경 파일은 구현에 필요한 `index.html`과 계획·설계 문서로 제한된다.
- 커밋과 push가 발생하지 않는다.

## Risks and trade-offs

- 기존 커서의 폭·높이를 바꾸면 줄바꿈 위치가 달라질 수 있으므로 모든 상태를 동일한 inline-block 기준 상자 안에 둔다.
- 자동 타이핑과 실제 입력의 처리 경로가 분리되어 있으면 한쪽만 모양이 바뀔 수 있으므로 연결 지점을 확인한다.
- `prefers-reduced-motion` 사용자는 전환 애니메이션 없이도 상태 변화를 볼 수 있어야 한다.
- 서버 포트 8000이 이미 사용 중이면 임의의 다른 포트로 우회하지 않고 기존 프로세스 상태를 먼저 확인한다.

## 완료 조건

- [ ] 작성할 때마다 커서 모양이 정해진 순서로 변경된다.
- [ ] 실제 입력과 자동 타이핑에서 모두 동작한다.
- [ ] 기존 기능 회귀가 없다.
- [ ] 포트 8000 서버가 실행된다.
- [ ] 사용자가 화면 확인 전까지 커밋·push하지 않는다.
