# Key — 데이터와 화면을 연결하는 열쇠

---

## 이 문서의 목적

프로그래밍에서 key는 근본적인 개념이다. 이 문서는 key의 본질을 이해한 후, 그 이해를 바탕으로 이 시스템의 cssSelectors/datasetAttrs 설계가 왜 그렇게 생겼는지를 자연스럽게 도출한다.

> **읽는 순서:** key의 일반론 → 이 시스템에서의 key → 설계의 필연성

---

## 1. Key란 무엇인가

### 본질: 식별과 접근

key는 "여러 개 중에서 특정 하나를 지목한다."

```javascript
obj[key] → value
```

이것이 key의 전부다. 데이터는 잠긴 상자이고, key를 넣어야 값을 꺼낼 수 있다.

### 프로그래밍 전반에서의 key

| 맥락 | key | 역할 |
|------|-----|------|
| 객체/맵 | `obj[key]` | 값을 꺼낸다 |
| 배열 | `arr[index]` | 순서로 꺼낸다 (index = 숫자 key) |
| 데이터베이스 | `WHERE id = key` | 행을 찾는다 |
| API | `/users/:id` | 리소스를 식별한다 |
| 암호학 | `encrypt(data, key)` | 잠그고 푼다 |

형태는 다르지만 역할은 같다. **특정 대상을 식별하여 접근한다.**

---

## 2. 이 시스템에서 key가 하는 일

### 데이터에서 값을 꺼낸다

API가 다음 데이터를 반환한다고 하자:

```javascript
{ time: '14:30', level: 'ERROR', message: '연결 실패' }
```

`time`, `level`, `message`가 key다. `itemData['time']`으로 `'14:30'`을 꺼낸다.
여기까지는 일반적인 key의 역할과 같다.

### 화면에서 요소를 찾는다

이 시스템에서는 같은 key가 DOM 요소를 찾는 데도 쓰인다:

```javascript
cssSelectors: {
    time:    '.event__time',
    level:   '.event__level',
    message: '.event__message',
}
```

`cssSelectors['time']`으로 `.event__time` 선택자를 꺼내고, 그 선택자로 DOM 요소를 찾는다.

### 하나의 key가 양쪽을 관통한다

```
  데이터 (API 응답)          key           화면 (DOM)
  ──────────────────    ───────────    ──────────────────
  itemData['time']    ←── time ──→    querySelector('.event__time')
  itemData['level']   ←── level ──→   querySelector('.event__level')
  itemData['message'] ←── message ──→ querySelector('.event__message')
```

key 하나가 "어떤 값을 꺼내서 어디에 넣을지"를 동시에 결정한다. 데이터의 열쇠이면서 화면의 열쇠다.

---

## 3. 코드에서 일어나는 일

ListRenderMixin의 renderData 내부:

```javascript
Object.entries(cssSelectors).forEach(([key, selector]) => {
    const el = clone.querySelector(selector);   // key로 화면 요소를 찾고
    if (!el || itemData[key] == null) return;    // key로 데이터 값을 꺼내서
    _applyValue(el, key, itemData[key], paths);  // 연결한다
});
```

1. `cssSelectors`를 순회한다 — key와 selector 쌍을 꺼낸다
2. `selector`로 DOM 요소를 찾는다 — 화면 쪽 접근
3. `itemData[key]`로 값을 꺼낸다 — 데이터 쪽 접근
4. 값을 요소에 넣는다 — 연결 완료

cssSelectors에 등록되지 않은 key는 순회 대상이 아니므로 무시된다. 등록 = 렌더링 대상 선언이다.

---

## 4. 값을 "어떻게" 넣을지 — 같은 key로 결정한다

기본적으로 값은 `textContent`로 들어간다. 그러나 모든 값이 텍스트는 아니다. 이미지 경로(`src`), CSS 상태(`data-*`), 동적 크기(`width`) 등이 있다.

이때도 같은 key를 사용한다:

```javascript
function _applyValue(el, key, value, paths) {
    const { datasetAttrs, elementAttrs, styleAttrs } = paths;

    if (datasetAttrs[key]) {
        el.setAttribute('data-' + datasetAttrs[key], value);  // data-* 속성
    } else if (elementAttrs[key]) {
        el.setAttribute(elementAttrs[key], value);            // 요소 속성
    } else if (styleAttrs[key]) {
        const { property, unit = '' } = styleAttrs[key];
        el.style[property] = value + unit;                    // 스타일
    } else {
        el.textContent = value;                               // 기본값: 텍스트
    }
}
```

하나의 key가 세 가지를 결정한다:

| 질문 | 답을 주는 곳 | key |
|------|-------------|-----|
| 어디에 넣을지? | `cssSelectors[key]` → selector → DOM 요소 | 같은 key |
| 무엇을 넣을지? | `itemData[key]` → 값 | 같은 key |
| 어떻게 넣을지? | `datasetAttrs[key]` / `elementAttrs[key]` / `styleAttrs[key]` / 기본값 | 같은 key |

---

## 5. 왜 기본값이 textContent인가

대부분의 데이터 필드는 텍스트로 표시된다. 이름, 시간, 상태 라벨, 메시지.

`data-*` 속성이나 요소 속성(`src`, `fill`)이나 인라인 스타일은 특수한 경우다. 가장 흔한 경우가 가장 적은 설정을 요구해야 한다.

```javascript
// 5개 필드 중 특수한 것만 등록
cssSelectors: {
    menuid: '.sidebar__item',
    active: '.sidebar__item',
    icon:   '.sidebar__icon',
    label:  '.sidebar__label',
}
datasetAttrs: {
    menuid: 'menuid',   // data-menuid (특수)
    active: 'active',   // data-active (특수)
}
// icon, label → 등록 안 함 → textContent (기본값)
```

만약 기본값이 없었다면:

```javascript
textContentAttrs: { icon: true, label: true }  // ← 불필요한 선언
```

특수한 것을 명시하고 나머지는 기본값으로 흘러가는 것이 `switch-case`의 `default`와 같다.

---

## 6. 왜 key를 공유하는가

### 공유하지 않았다면

```javascript
cssSelectors:  { activeElement: '.sidebar__item' }
datasetAttrs:  { activeData: 'active' }
dataFieldMap:  { activeElement: 'activeData' }  // ← 연결 테이블 필요
```

key를 하나 추가할 때마다 세 곳을 동기화해야 한다. 연결 테이블이 커지면 실수가 발생한다.

### 공유하면

```javascript
cssSelectors:  { active: '.sidebar__item' }
datasetAttrs:  { active: 'active' }
```

`active`라는 이름 하나가 데이터 필드, 대상 요소, 적용 방식을 관통한다. 연결 테이블이 불필요하다. key 자체가 연결이다.

---

## 7. 데이터와 key의 관계

key가 데이터와 화면을 연결하려면, 데이터의 필드명과 cssSelectors의 key가 일치해야 한다.

### 일치하면 — 직결

```javascript
// API 응답: { time: '14:30', level: 'ERROR' }
// cssSelectors key: time, level → 일치

this.subscriptions = {
    events: [this.listRender.renderData]  // 변환 없이 직결
};
```

### 불일치하면 — 변환이 필요

```javascript
// API 응답: { timestamp: '2026-04-03T14:30', severity: 'ERR' }
// cssSelectors key: time, level → 불일치

this.transformData = function ({ response }) {
    const data = response.map(item => ({
        time:  item.timestamp.slice(11, 16),    // timestamp → time
        level: item.severity === 'ERR' ? 'ERROR' : item.severity,  // severity → level
    }));
    this.listRender.renderData({ response: data });
};

this.subscriptions = {
    events: [this.transformData]  // 변환 메서드를 거쳐 연결
};
```

데이터를 key에 맞추거나, key를 데이터에 맞추거나. 양쪽이 일치해야 연결이 성립한다.

---

## 요약

| 단계 | 이해 |
|------|------|
| key의 본질 | 여러 개 중 특정 하나를 식별하여 접근한다 |
| 이 시스템에서의 key | 데이터와 화면을 동시에 식별하는 양방향 열쇠 |
| key 공유 | 연결 테이블 없이 하나의 이름으로 관통한다 |
| 기본값 textContent | 가장 흔한 경우가 가장 적은 설정을 요구한다 |
| 적용 방식 분기 | 특수한 것만 명시, 나머지는 기본값으로 흘러간다 |
| 데이터-key 일치 | 양쪽이 일치해야 연결이 성립한다. 불일치하면 변환한다 |

key를 이해하면 이 설계가 선택이 아니라 필연임을 알 수 있다. key가 데이터와 화면을 연결하는 역할을 한다면, 같은 key로 양쪽을 관통하는 것이 가장 단순한 구조이고, 가장 흔한 적용 방식(textContent)을 기본값으로 두는 것이 가장 적은 설정을 요구하는 구조다.

*작성일: 2026-04-03*
