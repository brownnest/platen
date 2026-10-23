# 붓 커서와 클래식 배경음악 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task.

**Goal:** 기존 도형 순환 커서를 붓 모양과 붓질 애니메이션으로 바꾸고, 사용자 조작으로 시작·일시정지할 수 있는 Web Audio 클래식 배경음악을 추가한다.

**Architecture:** `index.html` 단일 파일 안에서 기존 `#cursorAnchor`를 CSS pseudo-element 두 개로 붓 손잡이·붓털로 표현한다. 기존 작성 경로의 `advanceCursorShape()` 호출은 `triggerBrushStroke()`로 교체한다. 배경음악은 기존 `TypewriterAudio`와 별도의 `AmbientMusic` 클래스로 구현하고 HUD 버튼에서 수명주기를 제어한다.

**Tech Stack:** 인라인 CSS, 바닐라 JavaScript, Web Audio API, Python 표준 HTTP 서버

## Global Constraints

- 수정 파일: `index.html`
- 기존 `#cursorAnchor` 요소와 입력·한글 조합·램프·타자음 기능을 유지한다.
- 외부 라이브러리, 외부 음원 URL, 별도 빌드 도구는 추가하지 않는다.
- 배경음악은 브라우저 자동재생 제한을 고려해 사용자 클릭 이후 시작한다.
- 커밋과 push는 사용자의 화면 확인 전에는 하지 않는다.

## 현재 맥락

- 작업 루트: `/home/pne/.hermes/profiles/sam/workspace/platen`
- 브랜치: `feat/sequential-cursor-shapes`
- 기존 미커밋 변경: `index.html`에 도형 순환 커서 구현, `docs/`에 이전 설계·계획 문서
- 서버: `python3 -m http.server 8000`이 실행 중이며 표준 출력·에러는 `$TMPDIR/platen-http-server.log`로 분리되어 있다.
- 기존 오디오: `TypewriterAudio`가 타자음과 리턴 벨을 Web Audio API로 합성한다.

## 인터페이스

- `triggerBrushStroke(): void`
  - `cursorAnchor`의 `brush-stroke` 클래스를 제거하고 reflow 후 다시 추가한다.
  - 작성 성공 시 한 번 호출한다.
- `AmbientMusic.start(): void`
  - AudioContext를 초기화·재개하고 반복 스케줄러를 시작한다.
- `AmbientMusic.stop(): void`
  - 스케줄러를 중지하고 현재 gain을 닫는다.
- `AmbientMusic.toggle(): boolean`
  - 시작·정지를 전환하고 현재 재생 상태를 반환한다.
- `AmbientMusic.isSupported(): boolean`
  - `AudioContext` 또는 `webkitAudioContext` 지원 여부를 반환한다.

### Task 1: 붓 모양 커서와 붓질 애니메이션

**Objective:** 기존 네 가지 도형 커서를 붓 모양 하나로 바꾸고 작성 순간에만 붓질 동작을 보여준다.

**Files:**
- Modify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`의 `#cursorAnchor` CSS 및 화면 상태 함수

**Steps:**

- [ ] `#cursorAnchor`의 기존 `position`, inline-block, 폭·높이를 유지한다.
- [ ] 기존 `data-cursor-shape` 선택자와 `cursorShapes`, `cursorShapeIndex`, `advanceCursorShape()`를 제거한다.
- [ ] `#cursorAnchor::before`는 회전된 붓 손잡이, `#cursorAnchor::after`는 끝의 붓털로 스타일링한다.
- [ ] `.brush-stroke` 상태에서 두 pseudo-element가 짧게 기울고 이동하도록 `@keyframes brushStroke`를 추가한다.
- [ ] `prefers-reduced-motion: reduce`에서는 애니메이션을 끄고 붓 모양은 유지한다.
- [ ] `triggerBrushStroke(): void`를 추가하고 DOM reflow로 연속 입력에서도 애니메이션을 재시작한다.
- [ ] 공백·Enter·자모 입력에서 `advanceCursorShape()` 호출을 `triggerBrushStroke()`로 교체하고 백스페이스에는 연결하지 않는다.

**Verification:**

- [ ] `data-cursor-shape`, `cursorShapes`, `advanceCursorShape`가 더 이상 소스에 남지 않는지 확인한다.
- [ ] `brush-stroke`, `triggerBrushStroke`, `@keyframes brushStroke`가 존재하는지 Python 정적 검사로 확인한다.

### Task 2: Web Audio 클래식 배경음악 엔진

**Objective:** 외부 파일 없이 짧은 피아노풍 C장조 아르페지오를 반복 재생·정지할 수 있게 한다.

**Files:**
- Modify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`의 `<script>` 내 오디오 엔진 구간

**Steps:**

- [ ] `AmbientMusic` 클래스를 `TypewriterAudio`와 분리해 추가한다.
- [ ] `AudioContext`와 `webkitAudioContext`를 감지하고 지원되지 않으면 `isSupported()`가 false를 반환하게 한다.
- [ ] C장조 음계 기반 패턴 `[261.63, 329.63, 392.0, 523.25, 392.0, 329.63]`을 순환시킨다.
- [ ] 각 음에 oscillator와 gain envelope를 만들고 attack 0.02초, release 0.45초, 기본 음악 gain 0.035를 사용한다.
- [ ] `setTimeout` 기반 스케줄러가 한 음씩 예약하고, `stop()`에서 타이머를 지우며 gain을 0으로 낮춘다.
- [ ] `start()`와 `toggle()`은 예외를 잡아 음악 실패가 입력·타자음 기능을 막지 않게 한다.

**Verification:**

- [ ] 소스에 `AmbientMusic`, `start`, `stop`, `toggle`, `isSupported`가 있고 외부 음원 URL이 없는지 정적 검사한다.
- [ ] Node로 추출한 `<script>`의 구문 검사를 실행한다.

### Task 3: HUD 연결, 서버 재시작, 브라우저 검증

**Objective:** 사용자가 배경음악을 제어할 수 있는 버튼을 추가하고 개발 서버에서 두 기능을 확인한다.

**Files:**
- Modify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`의 HUD 마크업과 이벤트 연결
- Verify: `/home/pne/.hermes/profiles/sam/workspace/platen/index.html`

**Interface consumed:** Task 2의 `AmbientMusic.toggle(): boolean`, `AmbientMusic.isSupported(): boolean`

**Steps:**

- [ ] HUD에 `musicToggleBtn` 버튼을 추가하고 기존 `demoToggleBtn`과 시각적 스타일을 맞춘다.
- [ ] 버튼 클릭 시 `music.toggle()`을 호출하고 텍스트를 `배경음악 ▶` 또는 `배경음악 ⏸`로 갱신한다.
- [ ] Web Audio 미지원 환경에서는 버튼을 disabled 처리하고 `배경음악 미지원`을 표시한다.
- [ ] 버튼 이벤트가 종이 입력으로 전파되지 않게 하고 기존 포커스 동작을 유지한다.
- [ ] 프로젝트 루트에서 현재 8000 서버를 종료한 뒤 `python3 -m http.server 8000 > "$TMPDIR/platen-http-server.log" 2>&1`로 재시작한다.
- [ ] `curl --fail http://127.0.0.1:8000/index.html`이 HTTP 200을 반환하는지 확인한다.
- [ ] 브라우저에서 붓 커서가 보이고 입력 시 붓질 애니메이션이 실행되는지 확인한다.
- [ ] 배경음악 버튼 클릭 후 오디오 컨텍스트가 running 상태가 되고, 다시 클릭하면 suspended 또는 음악 정지 상태가 되는지 확인한다.
- [ ] 한글 조합, 자동 타이핑, 타격음, 램프 컨트롤, 리셋이 유지되는지 확인한다.
- [ ] `git diff --check`와 `git status --short --branch`를 실행하고 커밋·push는 하지 않는다.

**Expected result:**

- `http://localhost:8000/`에서 붓 커서와 배경음악 버튼이 보인다.
- 사용자 클릭 후 클래식 아르페지오가 재생되고 다시 클릭하면 정지한다.
- HTTP 200과 JavaScript 구문 검사가 통과한다.
- 변경은 현재 작업 브랜치에 남고 커밋·push는 발생하지 않는다.

## Risks and trade-offs

- 브라우저는 사용자 조작 전 오디오 재생을 막을 수 있으므로 버튼 클릭을 명시적 시작점으로 사용한다.
- CSS 붓털이 커서 위치를 넓혀 보이게 하더라도 inline-block 기준 폭은 유지해 줄바꿈을 바꾸지 않는다.
- Web Audio 스케줄러 실패 시 음악만 중단하고 텍스트 입력은 계속 동작해야 한다.
- 기존 미커밋 도형 커서 변경 위에 적용하므로 최종 diff에서 이전 기능과 이번 변경을 함께 검증한다.

## 완료 조건

- [ ] 커서가 네 가지 도형이 아닌 붓 모양으로 표시된다.
- [ ] 글자 작성 시 붓질 애니메이션이 실행된다.
- [ ] 사용자 클릭으로 클래식 배경음악을 시작·일시정지할 수 있다.
- [ ] 기존 기능이 유지된다.
- [ ] 개발 서버가 포트 8000에서 정상 응답한다.
- [ ] 사용자의 화면 확인 전 커밋·push가 없다.
