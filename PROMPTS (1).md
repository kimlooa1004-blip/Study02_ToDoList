# 구현 프롬프트 (5단계)

[PRD.md](./PRD.md)를 클로드 코드에서 실행할 5개의 프롬프트로 나눈 것이다.

## 사용법

이 문서는 한 줄씩 붙여 넣는 대본이 아니라, 클로드에게 통째로 넘길 구현 계획서다.

- "이 계획에 따라 1단계부터 구현해 줘"라고 지시하면, 클로드가 단계마다 구현, 확인, 커밋을 한다.
- 한 단계가 끝나면 보고를 읽고 다음 단계로 넘어간다. 어긋난 것이 있으면 그 단계 안에서 고치게 한다. 뒤로 갈수록 원인을 찾기 어렵다.
- 구현이 끝나면 브라우저에서 직접 확인한다. 문서 끝의 「마무리 점검」이 그 자리다.
- 이 앱은 빌드 도구와 의존성이 없는 단일 HTML 파일이라 테스트 러너를 두지 않는다. 단계별 검증은 클로드가 하고, 마지막 확인은 사람이 한다.

## 공통 규약

아래는 5단계 전체에 적용되는 제약이다. 각 프롬프트에도 필요한 만큼 반복해 넣었다. 클로드가 중간에 어긋나면 이 절을 다시 붙여 넣는다.

**구조 제약**

- 파일은 `index.html` 하나뿐이다. CSS는 `<style>`, 자바스크립트는 `<script>`로 같은 파일 안에 둔다.
- 외부 라이브러리, CDN, 폰트를 쓰지 않는다.
- ES 모듈(`import` / `export`)을 쓰지 않는다. `file://`로 열면 막힌다.
- UI 문구는 한국어로 쓴다.
- 사용자가 입력한 글자는 `textContent`나 `value`로만 화면에 넣는다. `innerHTML`에 입력값을 쓰지 않는다.

**이름 규약** (단계마다 이름이 어긋나지 않도록 고정한다)

| 종류 | 이름 |
|---|---|
| 상수 | `STORAGE_KEY` (`"todo-app-v1"`), `CATEGORIES`, `MAX_LENGTH` (200) |
| 상태 | `todos`, `currentFilter`, `editingId` |
| 저장 | `loadTodos()`, `saveTodos()`, `isValidTodo()`, `showNotice()` |
| 화면 | `render()`, `renderList()`, `renderProgress()`, `getVisibleTodos()` |
| 동작 | `addTodo()`, `toggleTodo(id)`, `deleteTodo(id)`, `saveEdit(id)` |

**상태 변경 규칙**

데이터를 바꾸는 모든 동작은 예외 없이 같은 순서를 따른다.

> 배열 변경 → `saveTodos()` → `render()`

DOM을 부분적으로 고치지 않는다. 항목이 20개 안팎이라 전체를 다시 그려도 느려지지 않고, 화면과 데이터가 어긋날 여지가 없다.

**항목 하나의 형태**

```js
{ id: 문자열, text: 문자열, category: "work" | "personal" | "study", done: 불리언, createdAt: 숫자 }
```

---

## 1단계: 뼈대와 저장 계층

**목표** — 화면 틀을 잡고, 데이터를 `localStorage`에 저장하고 불러오는 부분을 먼저 완성한다. 아직 버튼은 동작하지 않는다.

```
index.html 파일 하나를 만들어서 할 일 관리 앱의 뼈대와 데이터 저장 계층을 구현해 줘.
같은 폴더의 PRD.md에 전체 요구사항이 있으니 참고해.

제약:
- 파일은 index.html 하나뿐이다. CSS는 <style>, 자바스크립트는 <script>로 같은 파일 안에 둔다
- 외부 라이브러리, CDN, 폰트를 쓰지 않는다. ES 모듈(import/export)도 쓰지 않는다
- UI 문구는 한국어로 쓴다

HTML 구조 (내용은 비어 있어도 되고, 자리만 잡아 줘):
- 제목: "할 일 관리"
- 안내 문구 자리: <p id="notice" role="alert" hidden>
- 입력줄 영역 (빈 컨테이너만. 2단계에서 채운다)
- 필터 탭 영역과 진행률 영역 (빈 컨테이너만. 4단계에서 채운다)
- 목록: <ul id="todo-list">

CSS:
- 본문 최대 폭 640px, 가운데 정렬. 폭 360px에서도 가로 스크롤이 생기지 않게 한다
- 읽기 편한 기본 여백과 글꼴만. 화려하게 만들지 않는다
- 카테고리 배지 색 3가지: 업무 파랑, 개인 초록, 공부 보라. 글자 대비는 4.5:1 이상

자바스크립트로 아래를 정확히 이 이름으로 구현해 줘:
- const STORAGE_KEY = "todo-app-v1"
- const CATEGORIES = { work: "업무", personal: "개인", study: "공부" }
- const MAX_LENGTH = 200
- let todos = []
- let currentFilter = "all"   // 4단계에서 쓴다
- let editingId = null        // 5단계에서 쓴다
- showNotice(message)
    #notice에 문구를 넣고 보이게 한다
- isValidTodo(t)
    id, text는 문자열, category는 CATEGORIES의 키, done은 불리언, createdAt은 숫자인지 확인한다
- loadTodos()
    localStorage에서 STORAGE_KEY를 읽어 JSON 배열로 해석하고 isValidTodo를 통과한 항목만 돌려준다.
    저장값이 없으면 빈 배열이다.
    localStorage를 쓸 수 없거나 JSON이 깨졌거나 배열이 아니면 빈 배열로 시작하고,
    showNotice("저장된 데이터를 불러오지 못했습니다")를 호출한다. 예외를 밖으로 던지지 않는다.
    깨진 원본은 다른 키(STORAGE_KEY + "-broken")에 백업해 둔다
- saveTodos()
    todos를 JSON 배열로 STORAGE_KEY에 저장한다. 실패하면 예외를 잡아서
    showNotice("저장하지 못했습니다. 새로고침하면 방금 한 변경이 사라질 수 있습니다")를 호출한다
- renderList()
    todos를 #todo-list에 다시 그린다. 이번 단계에서는 읽기 전용이다.
    각 항목은 <li>로 만들고 안에 할 일 텍스트와 카테고리 배지를 넣는다.
    배지 텍스트는 CATEGORIES로 변환하고, 배지 클래스는 카테고리 키(work/personal/study)로 한다.
    체크박스와 버튼은 아직 넣지 않는다
- render()
    지금은 renderList()만 호출한다. 다음 단계에서 호출 대상이 늘어난다

마지막에 todos = loadTodos(); render(); 를 한 번 호출해서 페이지를 열면 저장된 데이터가 뜨게 해 줘.
```

**확인** (클로드가 수행하고 결과를 보고한다) — `index.html`을 브라우저로 열고 개발자 도구 콘솔에서 실행한다.

```js
todos = [
  { id: "a", text: "테스트 항목", category: "work",  done: false, createdAt: Date.now() },
  { id: "b", text: "두 번째",     category: "study", done: true,  createdAt: Date.now() }
];
saveTodos(); render();
```

- [ ] 두 항목이 화면에 뜨고, 배지가 각각 업무와 공부로 나온다
- [ ] 새로고침해도 두 항목이 그대로 있다 — 이 단계의 핵심이다
- [ ] 콘솔에서 `localStorage.setItem("todo-app-v1", "깨진값")`을 실행하고 새로고침한다. 오류 없이 빈 목록으로 뜨고 안내 문구가 보인다
- [ ] 360px 폭에서 가로 스크롤이 생기지 않는다

**커밋**

```bash
git add index.html && git commit -m "feat: 앱 뼈대와 localStorage 저장 계층 구현"
```

---

## 2단계: 할 일 추가

**목표** — 입력줄을 만들어 실제로 할 일을 쌓을 수 있게 한다. 이 단계가 끝나면 콘솔 없이 앱을 쓸 수 있다. (PRD F1, F5)

```
index.html에 할 일 추가 기능을 구현해 줘. PRD.md의 F1(추가)과 F5(카테고리)다.
기존 이름 규약(todos, saveTodos, render, renderList, CATEGORIES, MAX_LENGTH)을 그대로 쓰고,
새 파일을 만들지 마.

입력줄 UI를 입력줄 영역에 <form id="add-form">으로 넣어 줘:
- 텍스트 입력칸: <input id="todo-input">, placeholder는 "할 일을 입력하세요", maxlength 200, aria-label 붙이기
- 카테고리 선택: <select id="category-select">, 업무/개인/공부 세 개.
  value는 각각 work / personal / study, 기본 선택은 업무
- 추가 버튼: <button type="submit">추가</button>

Enter와 추가 버튼은 form의 submit 이벤트 하나로 처리해 줘 (keydown을 따로 걸지 마.
한글 입력 중 Enter가 두 번 처리되는 문제를 피하려는 것이다).

addTodo(rawText, category) 함수를 만들어서 아래대로 동작하게 해 줘:
- 입력 텍스트의 앞뒤 공백을 잘라 내고 MAX_LENGTH(200)자까지만 쓴다
- 잘라 낸 결과가 빈 문자열이면 아무 일도 하지 않고 false를 돌려준다. 경고창을 띄우지 마
- category가 CATEGORIES의 키가 아니면 false를 돌려준다
- 새 항목 { id, text, category, done: false, createdAt: Date.now() } 를 todos의 맨 앞에 넣는다 (unshift).
  id는 Date.now() + "-" + 랜덤 문자열로 만든다. crypto.randomUUID()는 쓰지 마
- saveTodos() 후 render(). 성공하면 true를 돌려준다

submit 처리:
- event.preventDefault()
- addTodo가 성공하면 입력칸을 비운다. 성공이든 실패든 입력칸에 포커스를 다시 준다.
  카테고리 선택은 그대로 둔다
- 마지막에 고른 카테고리를 localStorage("todo-app-v1-last-category")에 저장하고,
  페이지를 열 때 불러와서 select에 적용한다. 저장·불러오기 실패는 조용히 무시한다

renderList()에 빈 상태 처리를 추가해 줘:
- todos가 비어 있으면 목록 자리에 "할 일을 추가하세요" 안내 문구를 표시한다 (흐린 색, 가운데 정렬)
```

**확인** (클로드가 수행하고 결과를 보고한다)

- [ ] 할 일 3개를 서로 다른 카테고리로 추가한다. 배지 색이 카테고리마다 다르다
- [ ] 마지막에 추가한 것이 맨 위에 있다
- [ ] 추가한 뒤 바로 다음 할 일을 타이핑할 수 있다 (포커스가 입력칸에 남아 있다)
- [ ] Enter 키와 추가 버튼 모두 추가된다
- [ ] 빈 칸이나 스페이스만 입력하고 추가를 누르면 아무 일도 일어나지 않는다
- [ ] `<b>테스트</b>`를 입력하면 굵은 글씨가 아니라 태그 문자 그대로 보인다
- [ ] 새로고침해도 3개가 그대로이고, 마지막에 고른 카테고리가 선택되어 있다
- [ ] 콘솔에서 `todos = []; saveTodos(); render();`를 실행하면 안내 문구가 나온다

**커밋**

```bash
git add index.html && git commit -m "feat: 할 일 추가 기능과 빈 상태 표시"
```

---

## 3단계: 완료 체크와 삭제

**목표** — 목록 안의 조작을 붙인다. 이벤트를 항목마다 붙이지 않고 한 곳에서 처리하는 구조를 여기서 잡는다. (PRD F3, F4)

```
index.html에 완료 체크와 삭제 기능을 구현해 줘. PRD.md의 F3(삭제)과 F4(완료 체크)다.

renderList()가 그리는 각 <li>에 아래를 추가해 줘:
- 왼쪽에 체크박스. 항목의 done 값이 체크 상태에 반영되어야 한다. aria-label 붙이기
- 오른쪽에 삭제 버튼. aria-label은 "삭제"

이벤트 처리 방식이 중요해. 항목마다 addEventListener를 붙이지 마.
<ul id="todo-list">에 이벤트 위임 리스너 하나만 걸어 줘.
- 각 <li>에 data-id 속성으로 항목 id를 넣는다
- 체크박스와 삭제 버튼에 data-action 속성을 넣는다 ("toggle", "delete")
- 리스너에서 event.target.closest("[data-action]")의 data-action을 보고 분기한다

함수 두 개:
- toggleTodo(id): 해당 항목의 done을 뒤집는다. saveTodos() 후 render()
- deleteTodo(id): 확인 창 없이 바로 배열에서 제거한다. saveTodos() 후 render().
  (PRD가 삭제 확인 창을 두지 않기로 했다. confirm()을 쓰지 마)

완료된 항목의 표시:
- 텍스트에 취소선을 긋고 글자를 흐리게 한다
- 목록에서는 미완료가 위, 완료가 아래에 오게 한다. 각 묶음 안에서는 지금 순서(최근 추가순)를 유지한다.
  이 정렬은 todos 배열 자체를 바꾸지 말고, 그릴 때만 적용한다

키보드로도 체크와 삭제가 가능해야 한다 (체크박스와 버튼은 기본 요소를 쓰면 된다).
```

**확인** (클로드가 수행하고 결과를 보고한다)

- [ ] 체크박스를 누르면 취소선이 생기고, 다시 누르면 없어진다
- [ ] 체크한 항목이 완료 묶음(아래쪽)으로 내려가고, 체크를 풀면 원래 순서의 자리로 돌아온다
- [ ] 체크한 뒤 새로고침해도 체크 상태가 유지된다
- [ ] 삭제 버튼을 누르면 항목이 바로 사라지고, 새로고침해도 사라진 상태다
- [ ] 개발자 도구 Elements 탭에서 리스너가 `<ul id="todo-list">` 하나에만 붙어 있다 (항목마다 붙어 있지 않다)
- [ ] 키보드(Tab, Space, Enter)만으로 체크와 삭제가 된다

**커밋**

```bash
git add index.html && git commit -m "feat: 완료 체크와 삭제 기능 (이벤트 위임)"
```

---

## 4단계: 필터와 진행률

**목표** — 카테고리별로 걸러 보고, 지금 얼마나 했는지 보여 준다. 이 앱에서 가장 틀리기 쉬운 단계다. (PRD F6, F7, F8)

```
index.html에 카테고리 필터와 진행률을 구현해 줘. PRD.md의 F6, F7, F8이다.

필터 탭 UI를 필터 영역에 넣어 줘:
- 버튼 4개: 전체 / 업무 / 개인 / 공부, 각각 data-filter 속성 (all / work / personal / study)
- 현재 선택된 탭은 색이나 밑줄로 구분하고 aria-pressed를 맞춘다
- 클릭하면 currentFilter를 바꾸고 render()를 호출한다. 탭 버튼은 다시 만들지 말고 클래스와 aria-pressed만 바꿔서
  키보드 포커스가 탭 버튼에 남게 한다. 클릭은 <nav> 하나에서 이벤트 위임으로 처리한다
- 카테고리 탭을 누르면 입력줄의 카테고리 선택도 그 카테고리로 맞춘다
- 지금 보는 탭에 안 보일 카테고리로 할 일을 추가하면 전체 탭으로 돌아간다
  (방금 추가한 항목이 사라진 것처럼 보이지 않게 하려는 것이다)
- **선택한 탭은 localStorage("todo-app-v1-filter")에 저장해서 새로고침 후에도 유지한다.**
  저장된 값이 all/work/personal/study가 아니면 "all"로 시작한다

getVisibleTodos():
- currentFilter가 "all"이면 todos 전체, 아니면 category가 같은 항목만 돌려준다
- 3단계의 "미완료 위, 완료 아래" 정렬도 여기에서 적용한다

renderList()를 고쳐 줘:
- todos 대신 getVisibleTodos() 결과를 그린다
- 빈 상태 문구를 둘로 나눈다. 전체가 비었으면 "할 일을 추가하세요",
  특정 카테고리가 비었으면 "이 분류에는 할 일이 없습니다"

renderProgress()를 새로 만들어 진행률 영역에 그려 줘:
- 진행률 막대와 "완료 7 / 전체 12 (58%)" 문구. 퍼센트는 Math.round로 정수 반올림한다
- **계산 기준은 getVisibleTodos()가 아니라 todos 전체다.** 업무 탭을 보고 있어도
  전체 진행률은 바뀌지 않는다. 화면에 보이는 목록과 진행률의 기준이 다른 것이 맞다
- 전체 항목이 0개이면 0%가 아니라 "할 일을 추가하세요"를 보여 주고 막대는 숨긴다.
  0으로 나누지 않도록 주의한다
- 그 아래에 카테고리별로 "업무 1/3 · 개인 2/2 · 공부 0/0" 형식의 작은 줄을 넣는다.
  항목이 없는 카테고리는 "0/0"으로 보여 준다
- 막대는 CSS width 퍼센트로 표현하고 role="progressbar"와 aria-valuenow를 붙인다

지금까지 render() 하나에 들어 있던 목록 그리기를 renderList()로 옮기고,
render()가 renderProgress()와 renderList()를 둘 다 호출하게 고쳐 줘.
```

**확인** (클로드가 수행하고 결과를 보고한다) — 업무 3개 중 1개 완료, 개인 2개 중 2개 완료, 공부 0개로 데이터를 정확히 만들어 놓고 확인한다.

| 화면 | 나와야 하는 값 |
|---|---|
| 전체 탭 | 완료 3 / 전체 5 (60%), 카테고리별 업무 1/3 · 개인 2/2 · 공부 0/0 |
| 업무 탭 | 목록에 업무 3개만. 진행률은 그대로 3 / 5 (60%) |
| 공부 탭 | "이 분류에는 할 일이 없습니다". 진행률은 그대로 3 / 5 (60%) |

- [ ] 업무 탭에서 항목 하나를 체크하면 완료 4 / 전체 5 (80%)로 바뀐다
- [ ] 새로고침해도 선택한 탭이 유지된다
- [ ] 개인 탭에서 업무 카테고리로 추가하면 전체 탭으로 돌아가고, 새 항목이 보인다
- [ ] 탭을 키보드(Tab, Enter)로 바꿔도 포커스가 탭 버튼에 남는다
- [ ] 항목을 모두 지우면 "할 일을 추가하세요"가 보이고, 오류가 나지 않는다
- [ ] 콘솔에서 `localStorage.setItem("todo-app-v1-filter", "엉뚱한값")` 후 새로고침하면 전체 탭으로 열린다

**커밋**

```bash
git add index.html && git commit -m "feat: 카테고리 필터와 전체 기준 진행률"
```

---

## 5단계: 수정과 마무리

**목표** — 마지막 기능인 인라인 수정을 붙이고, PRD의 인수 기준 12개를 전부 통과시킨다. (PRD F2)

```
index.html에 할 일 수정 기능을 구현하고 마무리해 줘. PRD.md의 F2(수정)다.

수정 진입:
- 항목 텍스트를 더블클릭하면 편집 모드가 된다. 이미 있는 이벤트 위임 리스너(<ul id="todo-list">)에
  dblclick 처리를 추가해서 처리해 줘. 항목마다 리스너를 붙이지 마
- 편집 모드는 editingId 변수 하나로 관리한다. 다른 항목을 더블클릭하면 앞의 편집은
  저장하지 않고 닫힌다. 이 동작이 맞다

renderList()에서 id가 editingId와 같은 <li>는 편집 모드로 그려 줘:
- 텍스트 자리에 현재 텍스트가 들어 있는 <input> (maxlength 200)
- 배지 자리에 현재 카테고리가 선택되어 있는 <select> (업무/개인/공부)
- 이 <li>를 그린 직후 input에 포커스를 주고 글자를 전체 선택한다

saveEdit(id):
- input 값의 앞뒤 공백을 잘라 내고 200자까지만 쓴다
- 빈 문자열이면 저장하지 않고 편집을 취소한다 (editingId를 null로 하고 render)
- 아니면 항목의 text와 category를 바꾸고, editingId를 null로 만든 뒤 saveTodos()와 render()

키보드: 편집 input에서 Enter는 저장, Esc는 취소 (editingId = null 후 render()).
수정 중에 한글 조합이 끝나기 전의 Enter(event.isComposing)는 저장으로 처리하지 마.

마무리 CSS:
- 편집 모드로 바뀔 때 <li> 높이가 크게 출렁이지 않게 한다
- 마우스를 올렸을 때 항목이 반응하는 정도의 hover 효과만 넣는다. 애니메이션은 넣지 마

작업이 끝나면 PRD.md의 "인수 기준과 수동 테스트" 12개를 하나씩 직접 점검하고,
통과 여부를 표로 정리해서 보고해 줘. 통과하지 못한 항목이 있으면 고친 뒤 다시 확인해.
```

**확인** (클로드가 수행하고 결과를 보고한다) — PRD의 인수 기준 12개를 전부 통과해야 한다. 그중 이 단계에서 새로 생긴 것들이다.

- [ ] 항목을 더블클릭하면 입력상자로 바뀌고, 텍스트와 카테고리를 함께 바꿔 Enter로 저장하면 목록과 배지에 모두 반영된다
- [ ] 새로고침해도 수정한 내용이 유지된다
- [ ] 편집 중 Esc를 누르면 원래 값으로 돌아온다
- [ ] 편집 중 텍스트를 모두 지우고 저장하면 수정이 취소되고 원래 값이 남는다
- [ ] 항목 A를 편집하다가 항목 B를 더블클릭하면 A는 원래 값으로 닫히고 B가 열린다
- [ ] 개발자 도구 Elements 탭에서 리스너가 여전히 `<ul>` 하나에만 붙어 있다

**커밋**

```bash
git add index.html && git commit -m "feat: 할 일 수정 기능과 스타일 마무리"
```

---

## 단계 요약

| 단계 | 내용 | PRD 항목 | 끝나면 할 수 있는 것 |
|---|---|---|---|
| 1 | 뼈대와 저장 계층 | 저장 방식 | 콘솔로 넣은 데이터가 새로고침 후에도 남는다 |
| 2 | 할 일 추가 | F1, F5 | 앱에서 직접 할 일을 쌓을 수 있다 |
| 3 | 완료 체크와 삭제 | F3, F4 | 할 일을 끝내고 지울 수 있다 |
| 4 | 필터와 진행률 | F6, F7, F8 | 카테고리별로 보고 진척을 확인한다 |
| 5 | 수정과 마무리 | F2 | PRD의 필수 기능이 모두 채워진다 |

선택 기능 F9(완료 항목 일괄 삭제)와 F10(내보내기/가져오기)은 5단계 뒤에 필요하면 같은 방식으로 추가한다.

## 마무리 점검 (사람이 브라우저에서 확인)

구현이 끝나면 배포 전에 직접 확인한다.

PRD의 「인수 기준과 수동 테스트」 12개를 순서대로 통과시킨다. 하나라도 걸리면 그 자리에서 고치라고 지시한 뒤 다시 확인한다. 클로드의 완료 보고와 실제 화면이 다를 수 있고, 그 차이를 찾는 것이 이 단계의 목적이다.

여기에 하나를 더한다.

- [ ] 개발자 도구 Application → Local Storage에서 `todo-app-v1` 키를 연다. 값이 JSON 배열 문자열 하나로 들어 있다
