# DesignComponent 생산 품질 보장 — 강제 수단 설계

## 1. 목적

DesignComponent(Mixin + 컴포넌트)를 **일관된 품질로 대량 생산**하는 것이 이 프로젝트의 목표다.

생산 파이프라인:

```
Figma 디자인 → [1-figma] → 정적 HTML/CSS → [2-component] → 동적 컴포넌트
                                            ↑
                                       이 단계에서 register.js, beforeDestroy.js,
                                       preview.html이 반복 생성된다
```

강제 수단은 이 생산 과정에서 **반복적으로 발생하는 오류를 기계적으로 차단**하여, 사람의 감사에 의존하지 않고 품질을 보장한다.

---

## 2. 왜 텍스트 규칙만으로 부족한가

2026-03-31 전수 검토에서 실증되었다.

```
대상: 35개 컴포넌트 (Components/ 8개 + Examples/ 4 프로젝트)
발견: 17건의 규칙 위반
원인: 모든 규칙이 텍스트 문서에 존재했지만, 구현이 따르지 않음
```

텍스트 규칙의 구조적 한계:

- Claude가 참조 문서를 읽었더라도 **컨텍스트 압축 후 유실**될 수 있다
- 규칙을 "알고" 있어도 **실수로 위반**할 수 있다 (차트 4종의 Step 3 누락)
- 03-28 검토에서도 같은 파일을 읽고 **같은 위반을 놓쳤다** — 사람의 감사도 완벽하지 않다

결론: **규칙을 아는 것과 규칙을 지키는 것은 다르다.** 기계적 강제가 필요하다.

---

## 3. 선별 원칙

### 이전 접근 (도구 중심) — 폐기

```
생산 계약 전체 나열 → "정규식으로 잡을 수 있는 것"만 Hook으로 → 나머지는 "리뷰"

문제: 설계적으로 가장 중요한 계약이 "정규식으로 못 잡으니까" 리뷰로 빠졌다.
     var 금지, rem 금지 같은 표면적 규칙이 Hook으로 올라갔다.
     → 중요도가 아니라 자동화 난이도로 분류한 것.
```

### 현재 접근 (설계 중요도 중심)

```
1. 설계 문서에서 계약의 중요도를 결정한다
2. 중요도 순서대로 정렬한다
3. 각 계약에 맞는 강제 수단을 찾는다 (수단은 계약에 종속)

위반 시 아키텍처가 무너지는가?  → P0
위반 시 런타임이 깨지는가?      → P1
위반 시 생산 품질이 저하되는가?  → P2
위반 시 스타일이 불일치하는가?   → P3
```

### 강제 수단의 종류

```
┌─────────────────────┬──────────────────────────────────────────────┐
│ 수단                 │ 특성                                         │
├─────────────────────┼──────────────────────────────────────────────┤
│ Hook(command)        │ PostToolUse에서 셸 스크립트 실행.             │
│                     │ 파일 내용을 정규식으로 검사. 빠르고 저렴.      │
├─────────────────────┼──────────────────────────────────────────────┤
│ Hook(prompt)         │ PostToolUse에서 LLM 단일 턴 평가.            │
│                     │ 의미적 판단 가능. 느리고 비용 발생.            │
├─────────────────────┼──────────────────────────────────────────────┤
│ structural-script   │ PostToolUse에서 셸 스크립트로 파일 간 교차 검증.│
│                     │ 파일 수 비교, 참조 존재 확인 등. 중간 복잡도.  │
├─────────────────────┼──────────────────────────────────────────────┤
│ audit-project       │ SKILL 완료 후 수동 또는 자동 실행.            │
│                     │ LLM이 전체 프로젝트를 종합 검토.              │
└─────────────────────┴──────────────────────────────────────────────┘
```

---

## 4. 생산 계약 — 설계 중요도순

### P0 — 설계 철학 계약 (위반 시 아키텍처 붕괴)

핵심 원칙: **각 요소는 자신의 역할만 수행한다.**

```
HTML   = 시각 구조 + 약속된 선택자 (데이터를 모른다)
Mixin  = 기능 (HTML 구조를 모른다)
페이지 = 오케스트레이션 (렌더링을 모른다)
```

이 역할 경계가 무너지면, 형식은 완벽해도 설계가 무너진 것이다.

| # | 계약 | 위반하면 | 강제 수단 |
|---|------|---------|----------|
| P0-1 | HTML은 데이터를 모른다 (런타임 데이터 하드코딩 금지) | Mixin 인터페이스가 장식품이 됨 | Hook(prompt) |
| P0-2 | register.js에서 fetch 금지, DOM 접근은 cssSelectors 계약 필수 | 데이터 출처 결합, 디자인 변형 호환 불가 | Hook(command) + Hook(prompt) |
| P0-3 | Mixin은 HTML 구조를 모른다 (선택자 계약으로만 접근) | 디자인 변형이 불가능해짐 | Hook(prompt) |
| P0-4 | 페이지는 렌더링하지 않는다 (오케스트레이션만) | 역할 경계 붕괴 | Hook(prompt) |
| P0-5 | Mixin은 다른 Mixin을 모른다 (조립 코드가 조합을 주도) | Mixin 간 의존성 발생 | Hook(prompt) |

**P0-1 상세 — HTML 데이터 하드코딩 검출:**

```
❌ 위반 — 형식만 지키고 설계를 무시:

  HTML에 데이터를 하드코딩:
    <span class="status" data-status="running">정상</span>

  register.js에 인터페이스를 "규칙을 지키기 위해" 정의:
    cssSelectors: { status: '.status' }
    datasetAttrs: { status: 'status' }

  → cssSelectors/datasetAttrs 객체는 존재하지만 장식품
  → renderData가 호출되어도 이미 있는 값 위에 덮어쓸 뿐
  → Mixin의 존재 이유가 소멸
```

```
✅ 올바른 사용 — 인터페이스가 실제로 데이터 흐름을 매개:

  HTML은 빈 상태 (약속된 선택자만 존재):
    <span class="status">-</span>

  register.js에서 Mixin이 데이터를 채우도록 연결:
    cssSelectors: { status: '.status' }
    datasetAttrs: { status: 'status' }
    → subscribe(topic, this, this.fieldRender.renderData)

  런타임 데이터 흐름:
    API 응답 { status: 'running' }
      → renderData가 호출됨
      → el.setAttribute('data-status', 'running')
      → CSS [data-status="running"] { color: green; }
```

**P0-2 상세 — register.js 역할 위반 검출:**

Hook(command)로 잡을 수 있는 표면적 신호:

```
fetch(, XMLHttpRequest, axios, fetchData, fetchAndPublish  → 데이터 직접 호출
```

Hook(command)로 잡을 수 없는 경우:

```
register.js 안에서 cssSelectors 계약을 거치지 않고 DOM에 접근하는 경우
→ 구문적으로는 합법이지만 디자인 변형 호환을 위반
→ Hook(prompt)가 필요
```

---

### P1 — 라이프사이클 계약 (위반 시 런타임 오류)

register.js에서 생성한 것이 beforeDestroy.js에서 정리되지 않으면 메모리 누수, 이벤트 중복, 구독 잔존이 발생한다.

| # | 계약 | 위반하면 | 강제 수단 |
|---|------|---------|----------|
| P1-1 | register.js에 구독 연결 존재 | 런타임 데이터 수신 불가 | Hook(command) |
| P1-2 | beforeDestroy.js가 register.js의 정확한 역순 | 정리 누락 → 메모리 누수 | structural-script |
| P1-3 | register.js에서 생성한 모든 것이 정리되는가 | 이벤트/구독 잔존 | structural-script |
| P1-4 | beforeDestroy에서 참조를 null 처리 | GC 방해 | Hook(command) |
| P1-5 | Mixin destroy가 주입한 모든 것을 null 처리 | 네임스페이스 잔존 | Hook(command) |
| P1-6 | Mixin이 네임스페이스를 주입하는가 | API 접근 불가 | Hook(command) |

**P1-2 상세 — 교차 파일 역순 검증:**

단일 파일 정규식으로는 불가능. register.js와 beforeDestroy.js를 동시에 읽어야 한다.

```
structural-script 로직:
  1. register.js에서 생성 순서 추출:
     - applyXxxMixin → Step 1
     - subscribe → Step 2
     - bindEvents → Step 3
  2. beforeDestroy.js에서 정리 순서 추출:
     - removeCustomEvents → Step 3 역순
     - unsubscribe → Step 2 역순
     - destroy → Step 1 역순
  3. 순서가 역순인지 검증
```

---

### P2 — 인터페이스 계약 (위반 시 생산 품질 저하)

SKILL이 참조하는 문서가 실제 구현과 불일치하면, 이후 생산되는 모든 컴포넌트가 잘못된 정보를 기반으로 생성된다.

| # | 계약 | 위반하면 | 강제 수단 |
|---|------|---------|----------|
| P2-1 | cssSelectors에 선언된 선택자가 HTML에 존재 | 런타임 null 참조 | structural-script |
| P2-2 | ListRenderMixin 사용 시 `<template>` 태그 존재 | 리스트 렌더링 실패 | structural-script |
| P2-3 | Mixin 추가/변경 시 관련 문서 동시 갱신 | SKILL이 잘못된 정보로 생산 | structural-script |
| P2-4 | spec과 구현(.js)의 public API 일치 | spec 기반 코드 생성 오류 | audit-project |
| P2-5 | 컨테이너 ID가 컴포넌트별 고유 | CSS 충돌 | Hook(command) |
| P2-6 | 컨테이너에 Figma 치수가 명시 (preview) | 시각적 검증 불가 | Hook(command) |

**P2-1 상세 — 선택자 교차 검증:**

```
structural-script 로직:
  1. register.js에서 cssSelectors 객체의 값 추출:
     cssSelectors: { title: '.card__title', status: '.card__status' }
     → ['.card__title', '.card__status']
  2. 대응하는 views/*.html에서 해당 선택자 존재 확인
  3. 누락된 선택자가 있으면 exit 2
```

**P2-3 상세 — Mixin 문서 동시 갱신:**

Mixin이 추가되면 갱신해야 하는 파일 목록:

```
  1. Mixins/MixinName.js          (구현)
  2. Mixins/MixinName.md          (문서)
  3. Mixins/specs/MixinName.md    (명세)
  4. SHARED_INSTRUCTIONS.md       (Mixin 목록)
  5. docs/README.md               (spec 수량)
```

```
structural-script 로직:
  트리거: Mixins/*.js 파일이 Write/Edit될 때
  1. Mixins/*.js 파일 수 카운트
  2. Mixins/*.md 파일 수 카운트
  3. SHARED_INSTRUCTIONS.md의 Mixin 목록 항목 수 카운트
  4. 불일치 시 exit 2 + "다음 파일도 갱신 필요: [목록]"
```

---

### P3 — 구문 규칙 (위반 시 스타일 불일치)

설계는 무너지지 않지만, RNBT 런타임 호환성이나 코드 일관성에 영향을 준다.

| # | 계약 | 위반하면 | 강제 수단 |
|---|------|---------|----------|
| P3-1 | CSS에 px 단위만 사용 (rem/em 금지) | RNBT 런타임 렌더링 오류 | Hook(command) |
| P3-2 | JS에 var 금지 | 스코프 오류 가능성 | Hook(command) |
| P3-3 | preview.html에 로컬 CSS `<link>` 금지 | 독립 실행 불가 | Hook(command) |
| P3-4 | Flexbox 우선 사용 (Grid/absolute 지양) | 레이아웃 일관성 | audit-project |
| P3-5 | preview `<script src>` ../개수 == 폴더 깊이 | preview ReferenceError (`loadComponentAssets is not defined`) | Hook(command) |

---

## 5. 강제 수단별 구현 명세

### Hook(command) — 셸 스크립트 검증

```
트리거: PostToolUse (Edit|Write)
동작: 파일 경로로 분기 → 해당 파일 읽기 → 정규식 검사 → exit 2 시 피드백
비용: 없음 (로컬 실행)
```

대상 계약:

| 우선순위 | 계약 | 파일 패턴 | 검출 방법 |
|---------|------|----------|----------|
| P0-2 | register.js fetch 호출 | `*/register.js` | `fetch(\|XMLHttpRequest\|axios\|fetchData\|fetchAndPublish` |
| P1-1 | register.js 구독 존재 | `*/register.js` | `subscribe(` 존재 확인 |
| P1-4 | beforeDestroy null 정리 | `*/beforeDestroy.js` | `= null` 존재 확인 |
| P1-5 | Mixin destroy null 처리 | `*/Mixin*.js` | `instance\.\w+ = null` 존재 |
| P1-6 | Mixin 네임스페이스 주입 | `*/Mixin*.js` | `instance\.\w+ = ns` 존재 |
| P2-5 | 컨테이너 ID 고유성 | `*/styles/*.css` | ID 선택자 추출 후 중복 확인 |
| P2-6 | preview Figma 치수 | `*/preview/*.html` | `width:.*px.*height:.*px` 존재 |
| P3-1 | CSS rem/em 금지 | `*.css` | `\d+rem\b\|\d+em\b` |
| P3-2 | var 금지 | `*.js` | `\bvar\b` (주석 행 제외) |
| P3-3 | preview link 금지 | `*/preview/*.html` | `<link.*href=` |

### Hook(prompt) — LLM 의미 검증

```
트리거: PostToolUse (Edit|Write)
동작: 파일 내용 + 설계 계약을 프롬프트로 전달 → LLM이 판단 → 위반 시 피드백
비용: LLM 호출 (파일 작성마다 발생 — 대상 파일 제한 필요)
```

대상 계약:

| 우선순위 | 계약 | 파일 패턴 | 프롬프트 핵심 |
|---------|------|----------|-------------|
| P0-1 | HTML 데이터 하드코딩 | `*/views/*.html` | "이 HTML에 런타임 데이터가 하드코딩되어 있는가? 약속된 선택자만 있고 실제 데이터 값은 비어있어야 한다." |
| P0-2 | register.js cssSelectors 계약 준수 | `*/register.js` | "이 register.js가 cssSelectors 계약을 거치지 않고 DOM에 접근하는가? 직접 fetch를 호출하는가?" |
| P0-3 | Mixin이 HTML 구조 무지 | `*/Mixin*.js` | "이 Mixin이 특정 HTML 구조에 의존하는가? cssSelectors 계약 외의 DOM 탐색(querySelector, parentNode, children 등)이 있으면 위반이다." |
| P0-4 | 페이지가 렌더링 안 함 | `*/before_load.js`, `*/loaded.js` | "이 페이지 코드가 직접 DOM을 조작하거나 렌더링하는가? 오케스트레이션(데이터 정의, interval 관리, param 관리)만 해야 한다." |
| P0-5 | Mixin 간 독립성 | `*/Mixin*.js` | "이 Mixin이 다른 Mixin의 메서드나 네임스페이스를 참조하는가?" |

**비용 제어:**

Hook(prompt)는 모든 파일 작성에 실행하면 비용이 과다하다. 대상을 제한한다:

```
views/*.html       → P0-1만 검사
register.js        → P0-2만 검사
Mixin*.js          → P0-3, P0-5만 검사
before_load.js     → P0-4만 검사
loaded.js          → P0-4만 검사
그 외 파일          → Hook(prompt) 미실행
```

### structural-script — 파일 간 교차 검증

```
트리거: PostToolUse (Edit|Write)
동작: 대상 파일 + 관련 파일을 함께 읽어서 정합성 검증
비용: 없음 (로컬 실행), 단 스크립트 복잡도 높음
```

대상 계약:

| 우선순위 | 계약 | 트리거 파일 | 교차 대상 | 검증 로직 |
|---------|------|-----------|----------|----------|
| P1-2 | beforeDestroy 역순 | `*/beforeDestroy.js` | 같은 컴포넌트의 `register.js` | register 생성 순서 추출 → beforeDestroy 정리 순서가 역순인지 |
| P1-3 | 생성-정리 대응 | `*/beforeDestroy.js` | 같은 컴포넌트의 `register.js` | register의 subscribe/bind 수 == beforeDestroy의 unsubscribe/remove 수 |
| P2-1 | 선택자 HTML 존재 | `*/register.js` | 같은 컴포넌트의 `views/*.html` | cssSelectors 값 추출 → HTML에서 존재 확인 |
| P2-2 | template 태그 존재 | `*/register.js` | 같은 컴포넌트의 `views/*.html` | ListRenderMixin 사용 시 → HTML에 `<template>` 존재 확인 |
| P2-3 | Mixin 문서 동시 갱신 | `*/Mixin*.js` | `SHARED_INSTRUCTIONS.md`, `*.md` | .js 수 vs .md 수 vs 목록 수 비교 |

---

## 6. 구현 우선순위

```
즉시 구현 (P0 커버리지 확보):
  1. Hook(command) — register.js, Mixin.js 표면 검사 [P0-2 부분, P1 전체, P3 전체]
  2. structural-script — 선택자 교차 검증, 역순 검증 [P1-2, P1-3, P2-1, P2-2]

효과 검증 후 도입:
  3. Hook(prompt) — P0 설계 철학 의미 검증 [P0-1~5]
  4. structural-script — Mixin 문서 정합성 [P2-3]

기존 유지:
  5. audit-project — 종합 진단 (SKILL 완료 후 실행)
```

---

## 7. 구현 상태 (2026-04-05)

### 구현 완료

| 스크립트 | 커�� 계약 | 대상 파일 | 검증 결과 |
|----------|----------|----------|----------|
| `check-p3.sh` | P3-1, P3-2, P3-3 | *.css, *.js, preview.html | 기존 인라인 Hook 3���를 ��합 |
| `check-register.sh` | P0-2, P1-1 | register.js | fetch 차단 + subscribe 존재 확인 |
| `check-beforeDestroy.sh` | P1-4 | beforeDestroy.js | null 정리 + destroy() + unsubscribe 존재 확인 |
| `check-page-loaded.sh` | P0-4 | loaded.js (page) | DOM 조작 차단 + 데이터 매핑 존재 확인 |
| `check-page-before-load.sh` | P0-4 | before_load.js (page) | DOM 조작 차단 + 이벤트 핸들러 등록 확인 |
| `check-page-before-unload.sh` | P1 | before_unload.js (page) | offEventBus + unregisterMapping + null 확인 |
| `cross-register-destroy.sh` | P1-2, P1-3, P1-4 | register.js ↔ beforeDestroy.js | Mixin수=destroy수, bind↔remove, 역순, 함수 null |
| `cross-selectors-html.sh` | P2-1, P2-2 | register.js ↔ views/*.html | cssSelectors↔HTML + ListRender↔template |
| `cross-page-lifecycle.sh` | P1-2, P1-4 | loaded.js ↔ before_unload.js | 생성-정리 매핑 + 3D raycasting/Three 정리 확인 |
| `check-preview-script-depth.sh` | P3-5 | preview/*.html | `<script src>` ../개수 == Components 이후 segment 수 (DesignComponentSystem 루트 점프 거리) |
| `check-key-trinity.sh` | P0-2(설계 수준) | register.js | datasetAttrs / elementAttrs / styleAttrs의 KEY가 cssSelectors의 부분집합인지. cssSelectors가 모든 KEY의 진실 출처. (REVIEW_COMMIT_DIRECTION L61–69 ack 패턴 위반 자인의 자동화 — 설계 수준 정합성 검증의 첫 사례) |

### 전수 테스트 결과

```
대상: 모든 DesignComponentSystem 라이���사이클 파일
  - Components/ 2D: 8개 컴포넌트
  - Components/ 3D: 7개 장비 × 3 변형 = 21개
  - Examples/ 2D: 4개 프로젝트 = 29개 컴포넌트
  - 페이지: 25개

오탐: 0건
진탐: 14건 — 3D 03_status_popup beforeDestroy.js 전수
  - P1-4: this.customEvents = null 누락
  - P1-2: removeCustomEvents → unsubscribe 역순 위반
```

### 미구현

```
Hook(prompt):   P0-1, P0-2 의미, P0-3, P0-4 의미, P0-5 — Phase 1·2 효과 검증 후 도입
structural:     P2-3 — Mixin 문서 동시 갱신 정합성
audit-project:  P2-4, P3-4 — SKILL 완료 후 수동 실행으로 유지
```

---

*작성일: 2026-03-31*
*개정일: 2026-04-04 — 선별 기준을 "정규식 가능 여부"에서 "설계 중요도"로 전환*
*근거: 03-31 전수 검토 22건 분석 — 실제 위반의 95%가 기존 Hook 선별 범위 밖*
*개정���: 2026-04-05 — Phase 1(단일 파일) + Phase 2(교차 검증) 구현 완료, 9개 스크립트 배��*
