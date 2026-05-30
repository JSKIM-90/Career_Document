# Why Mixin — Renobit 컴포넌트 확장 설계 원칙

## 1. 문제: 프레임워크가 소유한 상속 체인

Renobit 컴포넌트는 프레임워크 위에서 동작하는 팩(Pack) 컴포넌트다.
상속 체인은 프레임워크가 정의하며, 팩 개발자는 이를 변경할 수 없다.

```
WVDisplayObject                    ← 프레임워크
  └── WVDisplayObjectContainer     ← 프레임워크
        └── WVComponent            ← 프레임워크
              ├── WVDOMComponent    ← 프레임워크 (2D)
              │     └── MyComponent ← 팩 코드
              └── NWV3DComponent    ← 프레임워크 (3D)
                    └── WV3DResourceComponent  ← 프레임워크
                          └── MyComponent      ← 팩 코드
```

팩 개발자가 제어할 수 있는 것은 최하위 클래스 하나뿐이다.

---

## 2. 제약: 왜 클래스 기반 확장이 불가능한가

### 2.1 프레임워크가 인스턴스를 생성한다

클래스 팩토리 Mixin 패턴은 JavaScript에서 문법적으로 가능하다:

```javascript
// JavaScript 문법으로는 가능하다
class MyComponent extends ShadowPopupMixin(ListRenderMixin(WV3DResourceComponent)) { }
```

그러나 Renobit에서는 **프레임워크가 인스턴스를 생성**한다.
팩 개발자는 `new MyComponent()`를 직접 호출하지 않는다.
프레임워크가 등록된 클래스를 기반으로 인스턴스를 만들고, 생명주기를 관리한다.

이 과정에서:
- 프레임워크는 특정 상속 체인을 전제로 동작한다
- 팩 개발자가 중간에 클래스를 끼워넣으면 프레임워크의 전제가 깨진다
- `_onCreateProperties()` → `_onCreateElement()` → `_onViewerReady()` 호출 순서가 보장되지 않는다

**클래스 팩토리 Mixin이 불가능한 이유는 JavaScript의 한계가 아니라, 프레임워크가 상속 체인을 소유하기 때문이다.**

### 2.2 팩 개발자의 진입점

팩 개발자가 코드를 작성할 수 있는 시점은 두 곳이다:

| 시점 | 메서드 | 용도 |
|------|--------|------|
| 인스턴스 생성 | `constructor()` | 초기 속성 설정 |
| 뷰어 준비 완료 | `_onViewerReady()` | DOM/3D 접근 가능, 기능 초기화 |

이 두 시점 모두 **이미 만들어진 인스턴스**에 대해 작업한다.
클래스를 바꾸는 것이 아니라, 인스턴스에 기능을 붙여야 한다.

---

## 3. 해법: 함수 기반 Mixin

이미 만들어진 인스턴스에 기능을 붙이는 유일한 방법은 인스턴스 데코레이션이다.

```javascript
_onViewerReady() {
    // Mixin 적용 → 네임스페이스로 기능 주입
    applyShadowPopupMixin(this, { ... });   // → this.shadowPopup
    applyListRenderMixin(this, { ... });    // → this.listRender
    applyEChartsMixin(this, { ... });       // → this.chart

    // Mixin이 만든 메서드를 구독에 연결
    this.subscriptions = {
        devices: [this.listRender.renderData]
    };
}
```

### 3.1 네임스페이스 격리

각 Mixin은 고유한 네임스페이스를 가진다.

```
this.shadowPopup.show()
this.shadowPopup.hide()
this.shadowPopup.query(selector)

this.listRender.renderData({ response })
this.listRender.clear()

this.chart.bindDataset()
this.chart.bindSeries()
```

메서드 이름이 충돌할 수 없다. `show`가 `shadowPopup`과 `chart` 양쪽에 있어도 문제없다.

### 3.2 클로저 기반 — this 바인딩 문제 제거

Mixin 내부 함수는 프로토타입 메서드가 아니라 클로저다.
`instance`를 캡처하므로 this 바인딩 문제를 구조적으로 제거한다.

```javascript
function applyListRenderMixin(instance, options) {
    const ns = {};
    instance.listRender = ns;

    // instance를 클로저로 캡처 — .bind() 불필요
    ns.renderData = function({ response }) {
        const containerEl = instance.appendElement.querySelector(container);
        // ...
    };
}
```

이 함수를 어디서 꺼내 쓰든 (`this.listRender.renderData`든, 배열에 넣든, 콜백으로 넘기든) `instance` 참조는 유지된다.

### 3.3 기존 문자열 방식의 한계

현재 컴포넌트들은 render 콜백을 문자열로 정의한다:

```javascript
// 현재 방식
render: ['renderBasicInfo']

// 소비 코드
fx.each((fn) => this[fn](response), render);
```

`this[fn]()` — 문자열을 키로 `this`의 프로퍼티를 조회한다.
이 방식은 `this`에 직접 바인딩된 메서드에서만 동작한다.

Mixin 네임스페이스의 메서드는 문자열로 접근할 수 없다:

```javascript
// 불가능 — this['listRender.renderData']는 undefined
render: ['listRender.renderData']

// 가능 — 함수 참조를 직접 전달
render: [this.listRender.renderData]
```

**이것이 함수 참조 방식으로 전환해야 하는 직접적인 이유다.**

---

## 4. 조립 코드에 의한 Mixin 조합

Mixin이 함수 기반이므로, 조립 코드가 여러 Mixin을 자유롭게 조합할 수 있다.

```javascript
// Shadow DOM 팝업 안에서 리스트 렌더링이 필요한 경우
// 조립 코드(register.js)가 콜백을 통해 다른 Mixin을 적용한다
applyShadowPopupMixin(this, {
    onCreated: (shadowRoot) => {
        // 팝업 내부 전용 스코프 생성
        this._popupScope = { appendElement: shadowRoot };

        // 같은 Mixin을 다른 스코프에 적용
        applyListRenderMixin(this._popupScope, {
            cssSelectors: { ... }
        });
    }
});
```

핵심 원칙: **Mixin은 다른 Mixin의 존재를 모른다.** 조합을 결정하는 것은 조립 코드(register.js)다.

- `ShadowPopupMixin`은 `onCreated` 콜백을 실행할 뿐, 그 안에서 무엇이 일어나는지 모른다
- `ListRenderMixin`은 자신이 Shadow DOM 안에서 동작하는지 모른다. `appendElement`만 알 뿐이다
- 일반 DOM의 `this.listRender`와 Shadow DOM의 `this._popupScope.listRender`가 독립적으로 존재한다
- 클래스 상속으로는 이런 "같은 기능의 다중 인스턴스"를 표현할 수 없다

---

## 5. 생명주기 정리

Mixin은 스스로 정리할 수 있어야 한다.
프레임워크의 `_onViewerDestroy()` 시점에 각 Mixin의 `destroy()`를 호출한다.

```javascript
// ShadowPopupMixin.destroy()
ns.destroy = function() {
    ns.removePopupEvents();          // 이벤트 리스너 해제
    if (host) host.remove();         // DOM 제거
    shadowRoot = null;               // 참조 해제
    instance.shadowPopup = null;     // 네임스페이스 제거
};
```

클로저가 캡처한 `instance`, `shadowRoot`, `host` 등의 참조를 명시적으로 끊어,
가비지 컬렉션이 가능하도록 한다.

---

## 6. 전환 전략: 문자열 → 함수 참조

기존 컴포넌트와의 호환성을 유지하면서 전환하려면, 소비 코드에서 두 방식을 모두 지원한다:

```javascript
// 변경 전
fx.each((fn) => this[fn](response), render);

// 변경 후 — 문자열과 함수 참조 모두 지원
fx.each(
    (fn) => (typeof fn === 'string' ? this[fn](response) : fn(response)),
    render
);
```

이후 새 컴포넌트부터 함수 참조 방식을 사용하고, 기존 컴포넌트는 점진적으로 전환한다.

---

## 7. 초기화 순서

함수 참조 방식에서는 Mixin 적용 순서가 중요하다.

```
_onViewerReady() {
    ① config 정의
    ② DOM 생성 / 상태 초기화
    ③ Mixin 적용          ← this.listRender, this.shadowPopup 등 생성됨
    ④ datasetInfo 정의    ← render: [this.listRender.renderData] 참조 가능
    ⑤ 구독 / 이벤트 연결
}
```

③이 ④보다 먼저 와야 함수 참조가 존재한다.
클로저 기반이므로 별도의 `.bind(this)` 단계는 필요 없다.

---

## 요약

| 질문 | 답 |
|------|-----|
| 왜 클래스가 아닌가? | 상속 체인이 프레임워크 소유이고, 클래스 체인 변경이 허용되지 않는다 |
| 왜 프로토타입 확장이 아닌가? | 인스턴스가 이미 만들어진 후에만 개입할 수 있다 |
| 왜 네임스페이스인가? | 여러 Mixin의 메서드 이름 충돌을 방지한다 |
| 왜 클로저인가? | this 바인딩 문제를 구조적으로 제거한다 |
| 왜 함수 참조로 전환하는가? | 중첩 객체의 메서드를 문자열로 참조할 수 없다 |

*작성일: 2026-03-26*
