# Mixin 명세서: FieldRenderMixin

> 이 문서는 `MIXIN_SPEC_TEMPLATE.md` (본 첨부 미포함)의 모범답안이다.

---

## 1. 기능 정의

| 항목 | 내용 |
|------|------|
| **목적** | 데이터를 보여준다 |
| **기능** | 데이터 객체의 각 필드를 화면 요소에 1:1로 표시한다 |

### 기존 Mixin과의 관계

| 항목 | 내용 |
|------|------|
| **목적이 같은 기존 Mixin** | ListRenderMixin (데이터를 보여준다) |
| **기능의 차이** | ListRenderMixin은 template을 복제하여 배열 데이터로 DOM을 생성함. FieldRenderMixin은 이미 존재하는 DOM 요소에 값을 채움. 데이터 형태도 다름 (배열 vs 플랫 객체). |

---

## 2. 인터페이스

### cssSelectors

| KEY | 종류 | 의미 |
|-----|------|------|
| `name` | 사용자 정의 | 시스템 이름을 표시할 요소 |
| `statusLabel` | 사용자 정의 | 상태 라벨을 표시할 요소 |
| `version` | 사용자 정의 | 버전을 표시할 요소 |
| `card` | 사용자 정의 | 이벤트 매핑 전용 (데이터 없이 사용 가능) |

> **규약 KEY 없음.** 모든 KEY는 사용자가 정의한다. Mixin이 정의하는 구조적 KEY는 없다. KEY는 "이 데이터를 이 위치에 표시하라"는 지정이다. 데이터와 무관한 KEY(예: `card`)는 customEvents나 페이지 핸들러에서 요소 접근용으로 사용된다.

### datasetAttrs

| KEY | 종류 | 의미 |
|-----|------|------|
| `status` | 사용자 정의 | cssSelectors['status']가 가리키는 요소에 `data-status` 속성 설정. CSS에서 `[data-status="RUNNING"]` 등으로 스타일링에 활용 |

> 규약 KEY 없음. 모든 KEY는 사용자가 정의한다. cssSelectors와 key를 공유하여 대상 요소를 찾고, VALUE가 data 속성명이 된다.

### 기타 옵션

없음.

---

## 3. renderData 기대 데이터

### 데이터 형태

```
플랫 객체. KEY가 cssSelectors/datasetAttrs의 KEY와 일치해야 한다.
```

### 예시

```javascript
// renderData({ response: ??? })에 전달되는 response의 형태:
{
    name:        'RNBT-01',     // → cssSelectors['name'] → textContent
    status:      'RUNNING',     // → datasetAttrs['status'] → dataset
    statusLabel: '정상',        // → cssSelectors['statusLabel'] → textContent
    version:     'v2.4.1'      // → cssSelectors['version'] → textContent
}
```

### KEY 매칭 규칙

```
Object.entries(data)를 순회하며:

  datasetAttrs에 키가 있으면 → [data-속성명] 선택자로 요소를 찾아 dataset에 반영
  cssSelectors에 키가 있으면 → 해당 선택자로 요소를 찾아 textContent에 반영

data 기준 순회이므로 data에 없는 KEY는 건너뛴다.
cssSelectors/datasetAttrs에 없는 data KEY도 건너뛴다.
```

---

## 4. 주입 네임스페이스

### 네임스페이스 이름

`this.fieldRender`

### 메서드/속성

| 속성/메서드 | 역할 |
|------------|------|
| `cssSelectors` | 주입된 cssSelectors. customEvents에서 computed property로 참조 |
| `datasetAttrs` | 주입된 datasetAttrs |
| `elementAttrs` | 주입된 elementAttrs |
| `styleAttrs` | 주입된 styleAttrs |
| `renderData({ response })` | 플랫 객체를 받아 각 필드를 DOM 요소에 반영 |
| `destroy()` | Mixin이 주입한 모든 속성과 메서드를 null 처리 |

---

## 5. destroy 범위

```
- ns.renderData = null
- ns.cssSelectors = null
- ns.datasetAttrs = null
- ns.elementAttrs = null
- ns.styleAttrs = null
- instance.fieldRender = null
```

---

## 6. 사용 예시

### HTML

```html
<div class="system-info">
    <div class="system-info__name">-</div>
    <span class="system-info__status" data-status="unknown">-</span>
    <span class="system-info__version">-</span>
</div>
```

### register.js

```javascript
applyFieldRenderMixin(this, {
    cssSelectors: {
        name:        '.system-info__name',
        statusLabel: '.system-info__status',
        version:     '.system-info__version'
    },
    datasetAttrs: {
        status:      'status'
    }
});

this.subscriptions = {
    systemInfo: [this.fieldRender.renderData]
};
```

---
