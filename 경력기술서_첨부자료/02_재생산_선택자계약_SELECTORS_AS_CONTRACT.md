# 선택자 계약 (Selectors as Contract)

> `cssSelectors`와 `datasetAttrs`뿐 아니라 `elementAttrs`, `styleAttrs` 등 **부가 속성 객체들의 공통 메커니즘**을 다룬다. 대표 예시로 cssSelectors/datasetAttrs 쌍을 사용한다.

## 1. 두 객체의 역할

- **cssSelectors** — 대상 요소를 찾는다 (querySelector의 인자)
- **부가 속성 객체** (`datasetAttrs`, `elementAttrs`, `styleAttrs`) — 찾은 요소에 **어떻게** 값을 적용할지 지정한다
  - `datasetAttrs` — data-* 속성명 지정 (예시로 사용)
  - `elementAttrs` — 요소 속성명 (src, href, fill 등) 지정
  - `styleAttrs` — 인라인 스타일 속성명/단위 지정

---

## 2. key 공유 메커니즘

datasetAttrs는 data 속성명을 지정하지만, 그것만으로는 아무것도 할 수 없다.
data 속성은 태그에 입력되어야 하고, 그러려면 대상 요소를 찾아야 한다.

그래서 cssSelectors와 **key를 공유**한다.
같은 key로 cssSelectors에서 대상 요소를 찾아야 비로소 setAttribute가 가능해진다.

### 동작 예시: ListRenderMixin.renderData 내부 (datasetAttrs 사용 시)

```javascript
Object.entries(cssSelectors).forEach(([key, selector]) => {
    const el = clone.querySelector(selector);
    if (!el || itemData[key] == null) return;

    if (datasetAttrs[key]) {
        el.setAttribute('data-' + datasetAttrs[key], itemData[key]);
    } else {
        el.textContent = itemData[key];
    }
});
```

예시 — `key = 'active'`일 때:

```
cssSelectors['active']  → '.sidebar__item'  → 대상 요소
datasetAttrs['active']  → 'active'          → data-active 속성으로 설정
itemData['active']      → 'true'            → 설정할 값 (없으면 skip)
```

### 같은 대상, 다른 용도

하나의 DOM 요소에 여러 data 속성이 붙을 수 있다.
cssSelectors에서 같은 선택자를 여러 key가 공유하면 된다:

```javascript
cssSelectors: {
    active:  '.sidebar__item',
    menuid:  '.sidebar__item',
}
datasetAttrs: {
    active:  'active',
    menuid:  'menuid'
}
// active → data-active, menuid → data-menuid (같은 요소에 두 개의 data 속성)
```

두 객체와 key에 대한 처리 방식은 각 믹스인의 구현에 따른다.

---

## 3. 3형제 — datasetAttrs / elementAttrs / styleAttrs

같은 key 공유 패턴을 따르되, **VALUE의 형태와 소비 방식이 다르다.**

| 객체 | VALUE 타입 | 적용 방식 |
|------|------------|-----------|
| `datasetAttrs` | string (속성 이름) | `setAttribute('data-' + VALUE, 값)` |
| `elementAttrs` | string (속성 이름) | `setAttribute(VALUE, 값)` |
| `styleAttrs` | `{ property, unit? }` | `el.style[property] = 값 + (unit ?? '')` |

### 함께 쓴 예시 — 한 요소에 세 가지가 동시에 흐를 수 있다

```javascript
cssSelectors: {
    level:   '.event__level',
    icon:    '.event__icon',
    width:   '.event__bar'
}
datasetAttrs: {
    level:   'level'              // → data-level="ERROR" (CSS 타겟팅용)
}
elementAttrs: {
    icon:    'src'                // → <img src="error.svg">
}
styleAttrs: {
    width:   { property: 'width', unit: '%' }  // → style.width = "72%"
}
```

- `level`, `icon`, `width` 모두 `cssSelectors`에서 대상을 찾는 건 동일하다
- 그 다음 **어떻게 적용할지**만 각자의 객체가 책임진다
- 등록되지 않은 key는 [KEY_AS_CONNECTOR](02_재생산_KEY연결자_KEY_AS_CONNECTOR.md) 문서의 결정 순서에 따라 `textContent`(기본값)로 폴백된다

> **설계 귀결**: "어디에(cssSelectors) / 무엇을(itemData) / 어떻게(datasetAttrs·elementAttrs·styleAttrs)"를 하나의 key로 관통하기 때문에, 새로운 적용 방식(예: `classAttrs`)이 필요하면 **같은 패턴으로 4번째 객체를 추가**하면 된다. 구조는 그대로 유지된다.

*작성일: 2026-03-26*
