---
name: produce-standard-auto
description: 2D Standard 컴포넌트를 서브에이전트 기반으로 한 컴포넌트씩 독립 컨텍스트에서 완전 자동 생산합니다. 메인은 Phase 0(대상 파악) → Agent 호출 → Phase 2(커밋) → 반복.
model: opus
---

# 2D Standard 컴포넌트 완전 자동 생산

> ⚠️ **사전 조건 — Opus 모델 권장**
>
> 이 SKILL은 매 사이클 서브에이전트에 `produce-component` 흐름을 위임하며, Sonnet에서 도메인 답습 + 구조 drift가 관측되었다 (2026-05-10 검증). 메인/서브에이전트 모두 Opus에서 실행할 것 (`/model opus`로 전환).

## 목표

DesignComponentSystem/Components 아래의 2D 컴포넌트를 폴더 알파벳 순서대로 Standard 생산한다.
한 번 실행하면 **남은 모든 2D Standard 대상**을 순차로 소화한다.
각 사이클은 **독립된 서브에이전트**가 처리하므로 메인 컨텍스트는 누적되지 않는다.

기존 `produce-standard-loop`(수동, 승인 기반)의 완전 자동 대체 버전.

> **공통 보일러플레이트는 [서브에이전트 공통 컨텍스트](03_파이프라인_서브에이전트컨텍스트.md)(Sub-agent 프롬프트) + [메인 루프 공통](03_파이프라인_메인루프공통.md)(메인 루프)에 단일 정의.** 본 문서는 변형 고유 절차만 inline으로 명시한다.

---

## 구조 원칙

```
[메인 루프]
    │
    ├─ Phase 0: 다음 대상 파악
    ├─ Phase 1: Agent(subagent_type=general-purpose) 호출 → 단일 컴포넌트 생산 위임
    ├─ Phase 2: 커밋
    └─ Phase 3: 다음 대상으로 반복 (남은 대상이 없을 때까지)
```

- **매 사이클마다 새 Agent 호출** — 서브에이전트는 독립된 컨텍스트에서 실행되고, 완료 후 요약만 반환
- **사용자 승인 포인트 없음** — 서브에이전트가 기능 분석/Mixin 매핑/CLAUDE.md/코드/manifest까지 자율 결정
- **실패 시 즉시 중단** — Hook 검증 실패, manifest JSON 오류, Agent 실패 보고 → 메인이 중단하고 사용자에게 알림

---

## 메인 루프 절차

### Phase 0. 다음 대상 파악

매 사이클 시작 시 실행한다. 대상 결정 규칙은 `_shared/phase0-2d.md` (본 첨부 미포함)를 따른다. 범주 폴더 구조를 동적 감지하여 `{컴포넌트경로}` 후보 목록을 만들고, `register.js`가 없는 첫 번째 항목을 **다음 대상**으로 삼는다. 남은 대상이 없으면 **전체 루프 종료** 후 사용자에게 완료 보고.

---

### Phase 1. 서브에이전트 호출

`Agent` 도구로 `subagent_type=general-purpose`에 위임한다.

**프롬프트 템플릿** (매 사이클 `{컴포넌트경로}`만 교체, `{차원}=2D` / `{단위}=컴포넌트` 고정):

```
대상: 2D 컴포넌트 `{컴포넌트경로}/Standard`를 처음부터 끝까지 생산한다.
예: Buttons/SplitButtons/Standard

## 배경 / 공통 필수 제약 / 공통 반환 형식

`/.claude/skills/0-produce/_shared/agent-prompt-common.md`의 §배경(차원=2D, 단위=컴포넌트) / §공통 필수 제약 / §공통 반환 형식을 **먼저 읽고 그대로 적용한다.**

## 필수 읽어야 할 문서 (순서대로)

1. `/.claude/skills/0-produce/_shared/agent-prompt-common.md` — **본 프롬프트의 §배경/§공통 필수 제약/§공통 반환 형식 진실 소스 (먼저 읽음)**
2. `/.claude/skills/SHARED_INSTRUCTIONS.md` — 공통 규칙
3. `/.claude/skills/0-produce/produce-component/SKILL.md` — 생산 프로세스. **Step 5-1 "디자인 페르소나 & CSS 조달 규칙"이 5종 페르소나 / 필수 참고 CSS / 시각 토큰 원칙의 진실 소스다. 반드시 해당 섹션이 지시하는 문서들을 모두 읽어라.**
4. `/.claude/skills/2-component/create-2d-component/SKILL.md` — 2D 구현
5. `/.claude/guides/CODING_STYLE.md` — 코딩 스타일
6. `/RNBT_architecture/DesignComponentSystem/Mixins/README.md` — Mixin 카탈로그
7. `/RNBT_architecture/DesignComponentSystem/docs/architecture/COMPONENT_SYSTEM_DESIGN.md` — 시스템 설계
8. `Components/{범주}/CLAUDE.md` — 범주 역할
9. `Components/{범주}/{컴포넌트}/CLAUDE.md` — MD3 정의 + 역할 (이미 존재)

## 참고 사례 (직전 완성품) — 구조적 참고만

> CSS 시각 토큰 원칙은 `produce-component/SKILL.md` Step 5-1을 따른다. 아래 사례는 **구조적 참고용**(Mixin 조립 · HTML 구조 · cssSelectors 계약 · 이벤트 패턴)이며, 색/타이포/간격/그림자/모서리 같은 시각 토큰은 Persona 프로파일이 최종 근거다.

- `Components/Buttons/ButtonGroups/Standard/` — ListRenderMixin 패턴 (항목 배열 렌더 + 선택)
- `Components/Buttons/IconButtons/Standard/` — FieldRenderMixin 패턴 (단일 객체 렌더 + 클릭)
- `Components/Buttons/SegmentedButtons/Standard/` — ListRenderMixin + Connected 스타일
- `Components/Buttons/SplitButtons/Standard/` — FieldRender + ListRender 조립
- `Components/Cards/Standard/` — FieldRender(본문) + ListRender(액션) 조립
- `Components/Checkbox/Standard/` — ListRender + tri-state(data-checked) + 인라인 SVG 체크마크

## 산출물 (모두 자동으로 작성)

1. `Standard/CLAUDE.md` — 기능 정의 + 구현 명세
2. `Standard/scripts/register.js` — Mixin 조립 + 구독 + 이벤트
3. `Standard/scripts/beforeDestroy.js` — 정리 코드
4. `Standard/views/0{1..5}_{persona}.html` — 5개 페르소나 변형
5. `Standard/styles/0{1..5}_{persona}.css` — 5개 페르소나 변형
6. `Standard/preview/0{1..5}_{persona}.html` — 5개 독립 실행 프리뷰. **`<script>` 내부 코드는 `_shared/preview-area-labeling.md` (본 첨부 미포함)의 4종 라벨(`[PREVIEW 인프라]` / `[PAGE]` / `[COMPONENT register.js 본문]` / `[PREVIEW 전용]`)로 영역 분리 필수 — 빌더가 페이지 작성 코드와 컴포넌트 register.js 본문을 한눈에 구분할 수 있도록**
7. `DesignComponentSystem/manifest.json` — `{컴포넌트}`의 `sets` 배열에 Standard 엔트리 추가

## 페르소나 5종 — 정의 출처

`produce-component/SKILL.md` **Step 5-1 "디자인 페르소나 & CSS 조달 규칙"**을 그대로 따른다. 해당 섹션이 5종 페르소나(01_refined ~ 05_glass) ↔ `create-html-css/SKILL.md` Persona A~E 매핑 · 필수 참고 CSS · 시각 토큰 원칙의 단일 진실 소스다.

## 추가 필수 제약 (공통 5개에 더해서)

- MD3 명세가 필요하면 WebFetch → 실패 시 WebSearch로 대체. 근거를 명시.
- 컴포넌트 CLAUDE.md의 cssSelectors 계약을 HTML과 JS에서 일치시킨다.
- CSS는 `#{component}-container { ... nesting ... }` 형태 (컨테이너 크기는 CSS가 소유).

## 추가 반환 항목 (공통 5개에 더해서)

- 주요 cssSelectors KEY 목록
- 5개 변형 요약 (각 변형의 의도 1줄)
- 페르소나 근거: 각 변형에서 SKILL.md Persona A~E의 어떤 항목(예: "깊이=그라디언트", "모서리=0~4px", "밀도=Compact")을 핵심 근거로 삼았는지 1~2줄
```

**호출**:

```javascript
Agent({
    description: "{컴포넌트경로} Standard 생산",
    subagent_type: "general-purpose",
    model: "opus",
    prompt: "(위 템플릿)"
})
```

---

### Phase 2. 결과 확인 + 커밋

[메인 루프 공통](03_파이프라인_메인루프공통.md) §Phase 2 공통 절차를 따른다 (파일 존재 확인 → manifest.json JSON 유효성 → 커밋 → 공통 실패 감지).

산출물 ls 패턴:

```bash
ls RNBT_architecture/DesignComponentSystem/Components/{경로}/Standard/{scripts,views,styles,preview}/
```

커밋 메시지:

```
feat: {컴포넌트경로}/Standard 컴포넌트 자동 생산 — {Agent 요약 첫 줄}
```

---

### Phase 3. 다음 사이클

[메인 루프 공통](03_파이프라인_메인루프공통.md) §Phase 3 공통 — 남은 대상이 있으면 Phase 0부터 다시 실행. 없으면 종료.

---

## 종료 보고

[메인 루프 공통](03_파이프라인_메인루프공통.md) §종료 보고 template를 따른다. 치환:

- `{차원}` = `2D`
- `{단계}` = `Standard`
- `{단위}` = `컴포넌트`
- `{SKILL 이름}` = `produce-standard-auto`

---

## 금지 사항

[메인 루프 공통](03_파이프라인_메인루프공통.md) §공통 금지 사항 3개 + 본 SKILL 추가:

- ❌ 한 사이클 안에서 여러 컴포넌트 생산 (Agent는 반드시 1 컴포넌트만)

---

## 참조 문서

- 공통 보일러플레이트: [서브에이전트 공통 컨텍스트](03_파이프라인_서브에이전트컨텍스트.md), [메인 루프 공통](03_파이프라인_메인루프공통.md)
- Phase 0 규칙: `_shared/phase0-2d.md` (본 첨부 미포함)
- Preview 영역 라벨: `_shared/preview-area-labeling.md` (본 첨부 미포함)
- 수동 버전: `/.claude/skills/0-produce/produce-standard-loop/SKILL.md` (승인 기반, 기존 유지)
- 생산 프로세스: `/.claude/skills/0-produce/produce-component/SKILL.md`
- 2D 구현: `/.claude/skills/2-component/create-2d-component/SKILL.md`
- SHARED_INSTRUCTIONS: `/.claude/skills/SHARED_INSTRUCTIONS.md`
