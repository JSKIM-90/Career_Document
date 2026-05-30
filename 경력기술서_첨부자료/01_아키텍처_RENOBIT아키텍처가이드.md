# RENOBIT 아키텍처 가이드

## 목차

- [Q. RENOBIT의 역할은 무엇인가요?](#q-renobit의-역할은-무엇인가요)
- [컴포넌트는 무엇인가요?](#컴포넌트는-무엇인가요)
- [Q. 페이지와 컴포넌트의 역할이 정해져 있나요?](#q-페이지와-컴포넌트의-역할이-정해져-있나요)
  - [핵심 원칙](#핵심-원칙)
  - [완전한 라이프사이클 흐름](#완전한-라이프사이클-흐름)
- [Q. 컴포넌트를 배치하긴 했는데, 어디서부터 어떻게 작업해야 하는지 모르겠어요.](#q-컴포넌트를-배치하긴-했는데-어디서부터-어떻게-작업해야-하는지-모르겠어요)
- [컴포넌트 소스 레벨 라이프사이클](#컴포넌트-소스-레벨-라이프사이클)
- [프로젝트 설계 템플릿](#프로젝트-설계-템플릿)
  - [Param 관리](#param-관리)
  - [Interval 관리](#interval-관리)
  - [YAGNI 원칙](#yagni-원칙)
- [컴포넌트 라이프사이클 패턴](#컴포넌트-라이프사이클-패턴)
- [Default JS 템플릿 (Mixin 기반)](#default-js-템플릿-mixin-기반)
- [fx.go 기반 에러 핸들링 가이드](#fxgo-기반-에러-핸들링-가이드)
- [Component Structure Guide](#component-structure-guide)
- [Utils 디렉토리](#utils-디렉토리)
- [부록 A: 라이프사이클 상세](#부록-a-라이프사이클-상세)
- [부록 F: 인스턴스 스코프, 생명주기, GC 분석](#부록-f-인스턴스-스코프-생명주기-gc-분석)

---

## Q. RENOBIT의 역할은 무엇인가요?

RENOBIT은 단순하게 생각하면 페이지와 컴포넌트로 웹 결과물을 만들어내는 플랫폼이다.

## 컴포넌트는 무엇인가요?

RENOBIT에서 컴포넌트는 클래스다. 클래스는 data와 data를 다루는 코드의 모음이다.

## Q. 페이지와 컴포넌트의 역할이 정해져 있나요?

페이지는 컨트롤러로서 컴포넌트를 운영한다.

컴포넌트는 수동적이며, 자신의 콘텐츠를 가지고 있다.

### 핵심 원칙

**페이지 = 오케스트레이터**
- 데이터 정의 (pageDataMappings)
- Interval 관리 (pageIntervals)
- Param 관리 (pageParams)

**컴포넌트 = 독립적 구독자**
- 필요한 topic만 구독
- 데이터 렌더링만 집중
- 페이지의 내부 구조 몰라도 됨

**Topic 기반 pub-sub**
- 중복 fetch 방지
- 여러 컴포넌트 공유 가능
- 느슨한 결합

### 완전한 라이프사이클 흐름

```
[페이지 로드]
  MASTER before_load → PAGE before_load
    → 이벤트 핸들러 등록 (onEventBusHandlers)
    ↓
  컴포넌트 register (MASTER + PAGE 모두)
    → GlobalDataPublisher.subscribe() (구독 등록)
    ↓
  리소스 로딩 → 컴포넌트 completed
    ↓
  MASTER loaded → PAGE loaded
    → 데이터셋 정의 (pageDataMappings)
    → pageParams 초기화
    → GlobalDataPublisher.registerMapping()
    → 최초 데이터 발행 (fetchAndPublish)
    → Interval 시작 (startAllIntervals)

[User Interaction]
  → DOM Event → Weventbus.emit() → Page EventBus Handler
  → pageParams 업데이트 → 즉시 fetchAndPublish

[페이지 언로드]
  MASTER before_unload → PAGE before_unload
    → stopAllIntervals()
    → offEventBusHandlers()
    → unregisterMapping()
    ↓
  컴포넌트 beforeDestroy (MASTER + PAGE 모두)
```

> 같은 마스터를 사용하는 페이지 간 전환 시, 마스터 스크립트는 스킵됩니다.

---

## Q. 컴포넌트를 배치하긴 했는데, 어디서부터 어떻게 작업해야 하는지 모르겠어요.

작업을 한다는 것은 컴포넌트의 행동양식을 결정한다는 것과 같다. 이벤트를 정의해야하고, 탄생과 죽음까지 행동양식을 결정해야한다.

태어날땐 무엇을 하며, 살아있는 동안 어떤 상호작용을 할 것이며, 죽는 순간에는 어떤 것을 정리해야하는지를 결정하는 것이 개발자의 몫이다.

페이지와 컴포넌트는 각각 라이프 사이클을 가지고 있다. 먼저 주로 활용되는 페이지의 라이프사이클은 다음과 같다.

| 단계 | 시점 |
|------|------|
| Before Load | 모든 개별 컴포넌트 register 이전 |
| Loaded | 모든 개별 컴포넌트 completed 이후 |
| Before Unload | 모든 개별 컴포넌트 beforeDestroy 이전 |

활용할 컴포넌트의 라이프 사이클은 다음과 같다.

| 단계 | 설명 |
|------|------|
| register | this.appendElement에 접근 가능 |
| beforeDestroy | this.appendElement에 접근 가능 |

### appendElement? → Component Container

`this.appendElement`는 컴포넌트의 가장 최상단이며, 컨테이너다.

- 2D의 경우 instance id를 id 속성으로 가진 HTMLElement(div)이고,
- 3D의 경우 "MainGroup"을 이름으로 가진 THREE.Object3D다.

인스턴스의 이름은 `this.name`으로 접근 가능하다.

---

## 컴포넌트 소스 레벨 라이프사이클

컴포넌트는 소스에서도 라이프사이클을 가지고 있다. (컴포넌트 커스텀 제작 시 중요)

참조: `DesignComponentSystem/Mixins/ComponentMixin.js`

### 뷰어 전용 라이프사이클 훅 (2D/3D 공통)

커스텀 컴포넌트에서 뷰어 전용 로직을 작성할 때 사용하는 훅입니다.

#### 등록 시점 (초기화)

```
1. _onViewerReady()        ← 컴포넌트 소스 (뷰어 전용)
2. WScript REGISTER        ← Codebox
```

- 두 시점 모두 `this.appendElement` 접근 가능
- `_onViewerReady()`는 뷰어 모드에서만 실행됨

#### 소멸 시점 (정리)

```
1. WScript BEFORE_DESTROY  ← Codebox
2. _onViewerDestroy()      ← 컴포넌트 소스 (뷰어 전용)
3. WScript DESTROY         ← Codebox
```

- `BEFORE_DESTROY`, `_onViewerDestroy()`: `this.appendElement` 접근 가능
- `DESTROY`: `this.appendElement` 접근 불가 (이미 제거됨)
- `_onViewerDestroy()`는 뷰어 모드에서만 실행됨

#### 사용 예시

```javascript
class MyChart extends WVDOMComponent {
  constructor() {
    super();
  }

  // 클래스 메서드로 정의 (WScript의 function 정의를 대체)
  renderChart(data) {
    const option = { /* ECharts 옵션 */ };
    this.chart.setOption(option);
  }

  _onViewerReady() {
    // 뷰어 전용 초기화 (super 호출 불필요)
    const chart = echarts.init(this.appendElement.querySelector('#echarts'));
    this.chart = chart;
  }

  _onViewerDestroy() {
    // 뷰어 전용 정리 (super 호출 불필요)
    if (this.chart) {
      this.chart.dispose();
      this.chart = null;
    }
  }
}
```

> **WScript에서 마이그레이션 시 주의**
> - WScript register에서 정의하던 `function`들은 **클래스 메서드**로 정의
> - `_onViewerReady()`에는 **초기화 로직만** 작성 (메서드 정의 X)

라이프사이클에 대해 정리하였다. 라이프사이클에 대해 정리한 이유는 작업자가 RENOBIT에서 코드를 작업하는 영역에 대해 이해할 수 있어야 자신이 원하는 코드 작업을 원하는 위치에 작성할 수 있기 때문이다. 이러한 라이프사이클을 기반으로 다음의 아키텍쳐에 따라 코드를 작업할 수 있다.

> **참고:** 컴포넌트의 register.js 내부 구조는 **Mixin 기반**으로 전환되었다. register.js는 Mixin 적용, 자체 메서드 정의, 구독 연결, 이벤트 매핑을 수행한다. Mixin과 자체 메서드 모두 cssSelectors 계약으로 DOM에 접근하며, beforeDestroy.js에서 정리한다. 상세는 [컴포넌트 라이프사이클 패턴](#컴포넌트-라이프사이클-패턴) 및 [COMPONENT_SYSTEM_DESIGN.md](01_아키텍처_컴포넌트시스템설계.md) 참조.

---

## 역할은 알겠는데 이 역할을 나눈 것의 의미는 무엇이죠? 각 탭에서는 그래서 무슨 코드를 써야하나요?

## 프로젝트 설계 템플릿

페이지 라이프사이클의 논리적 설계 의도를 설명합니다.

> **코드 템플릿**: [Default JS 템플릿](#default-js-템플릿) 참조

### 페이지 라이프사이클 구현

| 파일 | 역할 | 핵심 논리 |
|------|------|----------|
| `page_before_load.js` | 컴포넌트 생성 전 초기 설정 | 이벤트 핸들러 등록 영역 제공 |
| `page_loaded.js` | 데이터 발행 및 갱신 관리 | 데이터마다 독립적인 interval 관리 |
| `page_before_unload.js` | 모든 리소스 정리 | 생성된 리소스 1:1 매칭 정리 |

### 설계 원칙

1. **이벤트 정의**: 빈 구조 제공 → 샘플로 패턴 명시 → 선택적 기능은 주석 처리
2. **데이터 갱신**: `refreshInterval` 있으면 주기적 갱신, 없으면 한 번만 fetch
3. **정리 순서**: Interval 중단 → EventBus 정리 → DataPublisher 정리 → Three.js 정리

### 페이지 생성/정리 매칭

| 생성 (before_load / loaded) | 정리 (before_unload) |
|-----------------------------|----------------------|
| `this.pageEventBusHandlers = {...}` | `this.pageEventBusHandlers = null` |
| `onEventBusHandlers(handlers)` | `offEventBusHandlers(handlers)` |
| `this.pageDataMappings = [...]` | `this.pageDataMappings = null` |
| `registerMapping(mapping)` | `unregisterMapping(topic)` |
| `this.pageParams = {}` | `this.pageParams = null` |
| `this.startAllIntervals()` | `this.stopAllIntervals()` |
| `this.pageIntervals = {}` | `this.pageIntervals = null` |
| `initThreeRaycasting(canvas, type)` | `canvas.removeEventListener(type, handler)` |
| `this.raycastingEvents = [...]` | `this.raycastingEvents = null` |
| *(3D 컴포넌트 존재 시)* | `disposeAllThreeResources(this)` |

### Param 관리

**문제:** param은 호출 시점마다 달라질 수 있어야 함 (필터, 시간 범위 등)

**해결:** `pageParams`로 topic별 param 관리

```javascript
// Initialize param storage
this.pageParams = {};

fx.go(
    this.pageDataMappings,
    each(GlobalDataPublisher.registerMapping),           // 1. Register
    each(({ topic }) => this.pageParams[topic] = {}), // 2. Init params
    each(({ topic }) => GlobalDataPublisher.fetchAndPublish(topic, this)) // 3. Fetch
);
```

| 항목 | 설명 |
|------|------|
| 관리 주체 | 페이지 (데이터셋 정보를 소유하므로) |
| 관리 구조 | `this.pageParams[topic]` |
| 사용 | `fetchAndPublish(topic, this, this.pageParams[topic])` |

#### 동적 Param 변경

**핵심: Stop/Start 불필요!** `pageParams`는 참조(Reference)이므로 interval이 자동으로 업데이트된 param을 사용합니다.

```javascript
'@filterChanged': ({ event }) => {
    const filter = event.target.value;

    // 1. Update pageParams
    this.pageParams['myTopic'] = {
        ...this.pageParams['myTopic'],
        filter
    };

    // 2. Immediate fetch - 사용자가 즉시 새 데이터 봄
    GlobalDataPublisher.fetchAndPublish('myTopic', this, this.pageParams['myTopic']);

    // 3. Interval은 자동으로 업데이트된 param 사용
}
```

### Interval 관리

**문제:** 데이터마다 갱신 주기가 다를 수 있음. `setInterval`은 응답 완료 전 다음 요청이 누적될 수 있음.

**해결:** `setTimeout` 체이닝 + `_stopped` 가드

```javascript
this.startAllIntervals = () => {
    this.pageIntervals = {};

    go(
        this.pageDataMappings,
        each(({ topic, refreshInterval }) => {
            if (refreshInterval) {
                const state = { _stopped: false, _timerId: null };
                this.pageIntervals[topic] = state;

                const scheduleNext = () => {
                    if (state._stopped) return;    // 좀비 방지 가드
                    state._timerId = setTimeout(() => {
                        fetchAndPublish(
                            topic,
                            this,
                            this.pageParams[topic] || {}
                        )
                        .catch(err => console.error(`[Page] ${topic}:`, err))
                        .finally(scheduleNext);    // 완료 후 다음 예약
                    }, refreshInterval);
                };
                scheduleNext();
            }
        })
    );
};

this.stopAllIntervals = () => {
    go(
        Object.values(this.pageIntervals || {}),
        each(state => {
            state._stopped = true;             // 반드시 clearTimeout보다 먼저
            clearTimeout(state._timerId);
            state._timerId = null;
        })
    );
};

this.startAllIntervals();
```

**왜 setTimeout 체이닝인가:**

| 방식 | 문제 |
|------|------|
| `setInterval` | 응답 완료 전 다음 요청이 누적됨 |
| `setTimeout` + `.finally()` | 응답 완료 후에만 다음 요청 예약 |
| + `_stopped` 가드 | `stopAllIntervals` 후 `.finally()`가 좀비 타이머를 재등록하는 것을 방지 |

### YAGNI 원칙

"필요할 때 추가하라. 미리 추가하지 마라."

| 기능 | 복잡도 | 실용성 | 권장 |
|------|--------|--------|------|
| Param 변경 | 낮음 | 매우 높음 | 필수 |
| Interval on/off | 낮음 | 높음 | 유용 |
| Interval 주기 변경 | 높음 | 매우 낮음 | 불필요 |

---

## 컴포넌트 라이프사이클 패턴

컴포넌트는 register와 beforeDestroy 두 개의 라이프사이클 단계를 활용합니다.
register.js는 Mixin 적용, 자체 메서드 정의, 구독/이벤트 연결을 수행합니다.

> **설계 문서**: [COMPONENT_SYSTEM_DESIGN.md](01_아키텍처_컴포넌트시스템설계.md) 참조

### Register 패턴 (Mixin 기반)

register.js의 순서는 중요합니다. Mixin 적용이 모든 것의 전제 조건입니다.

```
1. Mixin 적용     — 반드시 먼저 (네임스페이스 생성)
2. 구독 연결      — Mixin 메서드를 함수 참조로 연결
3. 이벤트 매핑    — Mixin의 선택자를 computed property로 참조
```

### beforeDestroy 패턴 (생성의 역순, 3단계)

| 생성 (register) | 정리 (beforeDestroy) |
|-----------------|----------------------|
| `bindEvents(this, customEvents)` | `removeCustomEvents(this, customEvents)` |
| `subscribe(topic, this, handler)` | `unsubscribe(topic, this)` |
| `applyXxxMixin(this, options)` | `this.xxxMixin.destroy()` |

---

## Default JS 템플릿 (Mixin 기반)

새 컴포넌트/페이지 생성 시 복사하여 시작하는 기본 스크립트 구조.
컴포넌트의 register.js는 Mixin 적용과 자체 메서드 정의를 수행한다.

```
Figma 디자인
    ↓
HTML/CSS 퍼블리싱 (약속된 선택자 포함)
    ↓
Mixin 조립 코드 적용 ← 이 문서
```

### 1. 컴포넌트 Default JS (Mixin 기반)

#### register.js

```javascript
const { subscribe } = GlobalDataPublisher;
const { bindEvents } = Wkit;
const { each, go } = fx;

// ======================
// 1. MIXIN 적용 — 반드시 먼저
// ======================

applyFieldRenderMixin(this, {
    cssSelectors: {
        // Mixin 인터페이스 KEY: 'CSS 선택자'
    },
    datasetAttrs: {
        // Mixin 인터페이스 KEY: 'data-* 속성명'
    }
});

// ======================
// 2. 구독 연결 — Mixin 메서드를 함수 참조로
// ======================

this.subscriptions = {
    // topic: [this.fieldRender.renderData]
};

go(
    Object.entries(this.subscriptions),
    each(([topic, handlers]) =>
        each(handler => subscribe(topic, this, handler), handlers)
    )
);

// ======================
// 3. 이벤트 매핑 — Mixin의 선택자를 computed property로
// ======================

this.customEvents = {
    // click: {
    //     [this.listRender.cssSelectors.item]: '@itemClicked'
    // }
};

bindEvents(this, this.customEvents);
```

#### beforeDestroy.js

```javascript
const { unsubscribe } = GlobalDataPublisher;
const { removeCustomEvents } = Wkit;
const { each, go } = fx;

// 3. 이벤트 제거
removeCustomEvents(this, this.customEvents);
this.customEvents = null;

// 2. 구독 해제
go(
    Object.entries(this.subscriptions),
    each(([topic, _]) => unsubscribe(topic, this))
);
this.subscriptions = null;

// 1. Mixin 정리
this.fieldRender.destroy();
// this.listRender.destroy();
```

### 2. 페이지 Default JS

#### before_load.js

```javascript
const { onEventBusHandlers } = Wkit;

// ======================
// EVENT BUS HANDLERS
// ======================

this.pageEventBusHandlers = {
    // Mixin 메서드에 네임스페이스로 직접 접근
    // '@clearClicked': ({ targetInstance }) => {
    //     targetInstance.listRender.clear();
    // },

    // Param 업데이트 패턴
    // '@filterChanged': ({ event }) => {
    //     this.pageParams['myTopic'] = {
    //         ...this.pageParams['myTopic'],
    //         filter: event.target.value
    //     };
    //     GlobalDataPublisher.fetchAndPublish('myTopic', this, this.pageParams['myTopic']);
    // }
};

onEventBusHandlers(this.pageEventBusHandlers);
```

#### loaded.js

```javascript
const { registerMapping, fetchAndPublish } = GlobalDataPublisher;
const { each, go } = fx;

// ======================
// DATA MAPPINGS
// ======================

this.pageDataMappings = [
    // {
    //     topic: 'myTopic',
    //     datasetInfo: {
    //         datasetName: 'myDataset',
    //         param: { baseUrl: 'localhost:4010' }
    //     },
    //     refreshInterval: 5000  // 없으면 한 번만 fetch
    //     // dataFormat — API 응답을 selector KEY에 맞춰 변환하는 함수를 필요 시 정의
    // }
];

// ======================
// PARAM MANAGEMENT
// ======================

this.pageParams = {};

go(
    this.pageDataMappings,
    each(registerMapping),
    each(({ topic }) => this.pageParams[topic] = {}),
    each(({ topic }) =>
        fetchAndPublish(topic, this)
            .catch(err => console.error(`[Page] ${topic}:`, err))
    )
);

// ======================
// INTERVAL MANAGEMENT (setTimeout 체이닝)
// ======================

this.startAllIntervals = () => {
    this.pageIntervals = {};

    go(
        this.pageDataMappings,
        each(({ topic, refreshInterval }) => {
            if (refreshInterval) {
                const state = { _stopped: false, _timerId: null };
                this.pageIntervals[topic] = state;

                const scheduleNext = () => {
                    if (state._stopped) return;
                    state._timerId = setTimeout(() => {
                        fetchAndPublish(topic, this, this.pageParams[topic] || {})
                            .catch(err => console.error(`[Page] ${topic}:`, err))
                            .finally(scheduleNext);
                    }, refreshInterval);
                };
                scheduleNext();
            }
        })
    );
};

this.stopAllIntervals = () => {
    go(
        Object.values(this.pageIntervals || {}),
        each(state => {
            state._stopped = true;
            clearTimeout(state._timerId);
            state._timerId = null;
        })
    );
};

this.startAllIntervals();
```

#### before_unload.js

```javascript
const { unregisterMapping } = GlobalDataPublisher;
const { offEventBusHandlers } = Wkit;
const { each, go } = fx;

// ======================
// INTERVAL CLEANUP
// ======================

if (this.stopAllIntervals) {
    this.stopAllIntervals();
}
this.pageIntervals = null;

// ======================
// EVENT BUS CLEANUP
// ======================

offEventBusHandlers(this.pageEventBusHandlers);
this.pageEventBusHandlers = null;

// ======================
// DATA PUBLISHER CLEANUP
// ======================

go(
    this.pageDataMappings,
    each(({ topic }) => unregisterMapping(topic))
);

this.pageDataMappings = null;
this.pageParams = null;
```

### 3. 페이지 3D Default JS ( 페이지 Default JS를 기반으로 )

3D 컴포넌트가 있는 페이지는 위 페이지 Default JS에 아래 내용을 추가합니다.

#### before_load.js 추가

```javascript
const { initThreeRaycasting } = Wkit;

// 3D RAYCASTING SETUP
const { withSelector } = Wkit;

this.raycastingEvents = withSelector(this.appendElement, 'canvas', canvas =>
    fx.go(
        [{ type: 'click' }],
        fx.map(event => ({
            ...event,
            handler: initThreeRaycasting(canvas, event.type)
        }))
    )
);
```

#### before_unload.js 추가

```javascript
const { disposeAllThreeResources, withSelector } = Wkit;

// 3D RAYCASTING CLEANUP
withSelector(this.appendElement, 'canvas', canvas => {
    if (this.raycastingEvents) {
        go(
            this.raycastingEvents,
            each(({ type, handler }) => canvas.removeEventListener(type, handler))
        );
        this.raycastingEvents = null;
    }
});

// 3D RESOURCES CLEANUP
disposeAllThreeResources(this);
```

---

## fx.go 기반 에러 핸들링 가이드

> **fx.js 출처:** [indongyoo/functional-javascript-01](https://github.com/indongyoo/functional-javascript-01)

### 목적

이 문서는 fx.go / reduce 기반 파이프라인에서 에러가 어떻게 전파되고, 어디에서 처리해야 하는지를 명확히 정의한다.

### 1. 기본 원칙

#### 1.1 유틸은 에러를 처리하지 않는다

fx.go, reduce, reduceF 등 유틸 레벨에서는 에러를 복구하지 않는다.

에러를 삼키지 않고 그대로 전파한다.

유틸의 책임은 다음 두 가지뿐이다:
- 정상 값은 다음 단계로 전달
- 에러는 호출자에게 전파

> 재시도, fallback, 사용자 알림, 로그 레벨 등은 도메인 컨텍스트를 아는 호출자의 책임이다.

#### 1.2 호출자는 반드시 에러를 처리한다

`fx.go(...)`는 실패 시 rejected Promise를 반환할 수 있다.

모든 호출부는 반드시 다음 중 하나로 에러를 처리해야 한다:
- async / await + try-catch
- `.catch(...)`

```javascript
// async / await
try {
  await fx.go(...);
} catch (e) {
  // 도메인별 처리
}

// Promise 체인
fx.go(...)
  .catch(e => {
    // 도메인별 처리
  });
```

### 2. fx.go 파이프라인의 내부 동작 (이해를 위한 설명)

#### 2.1 fx.go는 reduce 기반 파이프라인

```javascript
const go = (...args) => reduce((a, f) => f(a), args);
```

- 각 함수의 반환값이 다음 함수의 입력이 된다
- Promise가 반환되면 비동기 파이프라인으로 연결된다

#### 2.2 비동기 처리의 핵심: reduceF

```javascript
const reduceF = (acc, a, f) =>
  a instanceof Promise
    ? a.then(
        a => f(acc, a),
        e => e == nop ? acc : Promise.reject(e)
      )
    : f(acc, a);
```

**의미 정리:**

| 구분 | 설명 |
|------|------|
| nop | 필터(L.filter) 전용 내부 시그널. "조건 불충족 → 스킵"을 표현. 에러가 아님. 순회를 계속하기 위해 acc를 그대로 반환 |
| 진짜 에러 | `Promise.reject(e)`로 그대로 전파. 복구하지 않음 |

> reduceF는 에러를 처리하지 않는다. nop만 예외적으로 스킵을 위해 복구한다.

#### 2.3 순회가 중단되는 이유 (fail-fast 동작)

reduce는 acc가 Promise일 경우 다음과 같이 동작한다:

```javascript
return acc.then(recur);
```

그러나 acc가 rejected Promise이면:
- `then(recur)`의 recur는 실행되지 않는다
- rejected 상태가 그대로 반환된다

**결과적으로:**
- 다음 함수 적용 ❌
- 순회 중단
- 최종적으로 fx.go는 rejected Promise를 반환

> reduce는 내부에서 catch를 하지 않기 때문에 기본 동작은 fail-fast이다.

### 3. nop의 정확한 역할

nop은 오직 필터를 구현하기 위한 내부 메커니즘이다.

목적은 비동기 조건식에서 "false → 스킵"을 표현하는 것이다.

**특징:**
- 진짜 에러 ❌
- 외부에서 처리 대상 ❌
- reduceF에서만 특별 취급

```javascript
e => e == nop ? acc : Promise.reject(e)
```

### 4. catch 위치와 파이프라인 의미

#### 4.1 중요한 구분

- 파이프라인의 정상 흐름의 의미는 함수 구성으로 결정된다
- catch의 위치는 에러 발생 시의 동작 방식을 결정한다

> **파이프라인의 정상 흐름은 함수 구성으로,**
> **파이프라인의 에러 처리 전략은 catch를 어디에 두느냐로 결정된다.**

#### 4.2 파이프라인 중간 catch (주의)

```javascript
fx.go(
  items,
  fx.each(item =>
    fx.go(fetchData(item), process)
      .catch(err => {
        console.error(err);
        // 반환값 없음 → resolved(undefined)
      })
  )
);
```

- 진짜 에러가 fulfilled 값으로 변환됨
- 파이프라인은 "성공"으로 인식하고 계속 진행
- 의도하지 않으면 버그의 원인이 됨

#### 4.3 기본 패턴: 파이프라인 끝에서 catch

```javascript
fx.go(
  items,
  fx.each(item =>
    fx.go(fetchData(item), process)
  )
).catch(e => {
  console.error('[Component]', e);
  // 상태 복구 / 에러 UI / 중단 처리
});
```

- 에러는 끝까지 전파된다
- 한 지점에서 일관되게 처리한다
- 기본값으로 권장되는 패턴이다

#### 4.4 예외: 부분 실패를 허용하는 경우

일부 실패를 허용해야 하는 도메인에서는 의도적으로 중간 catch를 사용할 수 있다.

단, 반드시 명시적인 대체값을 반환해야 한다:

```javascript
.catch(e => ({
  ok: false,
  error: e
}));
```

> 에러를 삼키는 것이 아니라 의미 있는 값으로 변환하는 것이다.

### 5. interval / 이벤트 핸들러

이 컨텍스트에서는 최상단 catch가 필수다.

목적은 unhandled rejection 방지다.

```javascript
const run = () =>
  GlobalDataPublisher.fetchAndPublish(topic, page, params)
    .catch(e => {
      console.error(`[fetchAndPublish:${topic}]`, e);
      // 재시도 / 백오프 / 사용자 알림 등
    });

setInterval(run, refreshMs);
run();
```

#### 5.1 §1.1 "에러 전파"와의 양립성 — startAllIntervals 패턴

`startAllIntervals`의 `.finally(scheduleNext)` 패턴(이 README L283–326 참조)은 §1.1의 "유틸은 에러를 처리하지 않는다"와 모순처럼 보이지만 실제로는 **계층 분리**에 의한 양립이다.

| 계층 | 정책 | 예 |
|---|---|---|
| **유틸 (fx.go / reduce)** | Fail-fast — 에러를 전파만 | reduceF는 진짜 에러를 `Promise.reject(e)`로 그대로 |
| **호출자 (페이지의 startAllIntervals)** | Fail-safe — 도메인 결정으로 격리 | `.catch(err => console.error)` + `.finally(scheduleNext)`로 다음 주기 보장 |

> 즉 fx.go 자체는 에러를 복구하지 않지만, **페이지 작성자가 도메인 컨텍스트**(여러 토픽이 독립적이고 한 토픽 실패가 다른 토픽 fetch 중단으로 이어지면 안 됨)를 알기 때문에 의도적으로 Fail-safe를 선택한다. `.finally`는 catch가 아니므로 에러 자체는 console.error로 노출되며, 다음 사이클이 자동으로 재시도한다.

#### 5.2 페이지 작성자 — Fail-fast vs Fail-safe 의사결정

새 페이지의 interval / 이벤트 핸들러를 작성할 때 다음 결정 트리를 따른다:

```
이 토픽이 다른 토픽과 독립적인가?
├─ YES → 한 토픽 실패가 페이지의 나머지 동작을 막으면 안 되는가?
│        ├─ YES → Fail-safe (개별 .catch + .finally(scheduleNext))
│        │        예: 대시보드의 N개 위젯 — 위젯 A의 fetch 실패가 위젯 B를 멈추면 안 됨
│        └─ NO  → 페이지 종료 시점에 일괄 catch (단순 패턴)
└─ NO  → Fail-fast (await + 단일 try-catch — 한 토픽이 실패하면 다음 단계 의미 없음)
         예: 페이지 초기 로딩의 인증 토픽 → 사용자 정보 토픽 (의존 사슬)
```

> **혼합 패턴 금지**: 같은 페이지에서 일부 토픽은 Fail-safe, 일부는 Fail-fast로 섞으면 디버깅이 어렵다. 의존 사슬이 있는 토픽은 별도의 Fail-fast 파이프라인으로 분리하고, 독립 토픽은 모두 Fail-safe로 통일한다.

> **선택 결과를 페이지의 `before_load.js` 상단에 한 줄 주석으로 명시**:
> ```javascript
> // [에러 전략] Fail-safe — 4개 토픽이 독립적이며, 한 토픽 실패는 다른 위젯 동작을 막지 않는다.
> ```

### 6. 체크리스트

- [ ] 모든 fx.go 호출부에 try-catch 또는 `.catch()`가 있는가?
- [ ] 파이프라인 중간 catch가 의도적인 복구인가?
- [ ] catch에서 반환값이 명확한가?
- [ ] nop을 진짜 에러 처리와 혼동하지 않았는가?
- [ ] interval / 이벤트 핸들러에 catch가 있는가?

### 핵심 요약

1. fx.go는 에러를 처리하지 않고 전파한다
2. reduceF는 nop만 복구하고 진짜 에러는 fail-fast로 중단시킨다
3. 파이프라인의 정상 흐름은 함수 구성으로 결정된다
4. 에러 처리 전략은 catch 위치로 결정된다

### 추가 자료

**[fail_fast_safe_error.md](01_아키텍처_에러전략_FailFastSafe.md)** - Fail-fast vs Fail-safe 에러 전략 상세 가이드

이 문서는 **"언제 Fail-fast를 쓰고, 언제 Fail-safe(격리)를 써야 하는가"**에 대한 판단 기준과 구체적인 패턴을 제공합니다.

| 전략 | 사용 시점 | 특징 |
|------|----------|------|
| **Fail-fast** | 초기 로딩이 완전해야 할 때, 중간 실패가 이후 단계를 무효화할 때 | 한 번 실패하면 전체 중단 |
| **Fail-safe** | topic들이 독립적일 때, 일부 데이터 유실을 허용할 때 | 개별 catch로 격리, 전부 시도 보장 |

---

## Component Structure Guide

컴포넌트 자산을 쌓기 위한 구조 가이드입니다.

### 핵심 원칙

#### Figma 선택 요소 = 컨테이너

```
┌─────────────────────────────────────────────────────────────────────┐
│  Figma 링크 제공 = 컴포넌트 단위 선택                                  │
│                                                                      │
│  사용자가 Figma 링크를 제공하면:                                       │
│  - 선택된 요소의 가장 바깥 = div 컨테이너                              │
│  - 선택된 요소의 크기 = 컨테이너 크기                                  │
│  - 내부 요소 = innerHTML (Figma 스타일 그대로)                        │
└─────────────────────────────────────────────────────────────────────┘
```

```html
<div id="[component-name]-container">  <!-- 컴포넌트별 고유 ID (예: transaction-table-container) -->
    <div class="transaction-table">    <!-- Figma 내부 요소 (스타일 그대로) -->
        ...
    </div>
</div>
```

> **컨테이너 ID 명명 규칙**: `#[component-name]-container` 형태로 컴포넌트마다 고유한 ID를 부여합니다.
> 예: `#header-container`, `#sidebar-container`, `#stats-cards-container`

### 웹 빌더 기본 구조

웹 빌더는 컴포넌트마다 div 컨테이너를 기본 단위로 가집니다.

웹 빌더에서 컴포넌트를 배치하면:

```html
<div id="component-xxx">   ← 웹 빌더가 자동 생성하는 컨테이너
    <!-- innerHTML -->     ← 사용자 정의 내용
</div>
```

따라서 Figma 선택 요소의 크기가 곧 컨테이너 크기가 되어야 스타일링이 그대로 유지됩니다.

### 컨테이너 크기 규칙

**컨테이너 크기 = Figma 선택 요소 크기**

```css
/* Container: Figma 선택 요소 크기 */
/* 컨테이너 ID는 컴포넌트별 고유: #[component-name]-container */
#transaction-table-container {
    width: 524px;   /* Figma 선택 요소 width */
    height: 350px;  /* Figma 선택 요소 height */
    overflow: auto; /* 동적 렌더링 대응 */
}
```

- Figma에서 선택한 요소의 크기가 곧 컨테이너 크기
- 레이아웃 조립 시 크기 조정은 에디터가 담당
- preview.html은 Figma 크기로 디자인 검증 목적

### 설계 철학

#### Figma 스타일 그대로 유지

- 컨테이너 크기 = Figma 선택 요소 크기
- 내부 요소 스타일 = Figma에서 추출한 그대로
- 임의로 `width: 100%`, `height: 100%`로 변경하지 않음

#### 박스 단위 조합

컨테이너가 있으면 조합이 단순해집니다:

```html
<!-- 컨테이너 없이 -->
<button>Click</button>
<!-- 조합 시 버튼 자체 스타일이 레이아웃에 간섭 -->

<!-- 컨테이너 있음 -->
<div class="button-container">
    <button>Click</button>
</div>
<!-- 박스끼리 조합 → 내부는 신경 안 써도 됨 -->
```

- 외부에서 보면: 그냥 박스
- 내부에서 보면: 버튼이든 테이블이든 상관없음
- 조합하는 쪽에서 내부 구현을 알 필요 없음

#### CSS Box Model과의 일관성

- 컨테이너 = Containing Block 역할
- 컨테이너가 명시적 크기를 가지므로 레이아웃 예측 가능
- `overflow: auto`로 동적 콘텐츠 대응

### 런타임 동작

현재 런타임 애플리케이션에서:

```
사용자가 컴포넌트 HTML 작성
    ↓
container.innerHTML = 사용자 정의 HTML
    ↓
외부에서 보면 container 하나
```

- 사용자가 컴포넌트 단위로 HTML을 작성
- HTML이 컨테이너의 innerHTML로 포함됨
- 사용자 정의 HTML이 얼마나 복잡하든, 외부에서는 container 하나로 취급

### 파일 구성

하나의 컴포넌트는 다음 구조로 구성됩니다. 디자인 변형은 넘버링으로 관리하며, scripts/는 디자인이 달라져도 변하지 않습니다.

```
ComponentName/
├─ views/
│   ├─ 01_bar.html             # 디자인 변형 A
│   └─ 02_card.html            # 디자인 변형 B
├─ styles/
│   ├─ 01_bar.css
│   └─ 02_card.css
├─ scripts/
│   ├─ register.js             # Mixin 조립 코드 (불변)
│   └─ beforeDestroy.js        # 정리 코드 (불변)
└─ preview/
    ├─ 01_bar.html
    └─ 02_card.html
```

| 파일 | 역할 |
|------|------|
| views/NN_name.html | 디자인 변형별 HTML (약속된 선택자 포함) |
| styles/NN_name.css | 디자인 변형별 CSS |
| scripts/register.js | Mixin 조립 코드 (디자인 무관, 불변) |
| scripts/beforeDestroy.js | 정리 코드 (디자인 무관, 불변) |
| preview/NN_name.html | 디자인 변형별 독립 테스트 |

> **핵심:** 약속된 선택자(cssSelectors, datasetAttrs의 값)를 HTML에서 유지하면 디자인은 자유

### 컴포넌트 템플릿

#### HTML (views/component.html)

```html
<div class="component-name">
    <!-- Figma 내부 구조 그대로 -->
</div>
```

#### CSS (styles/component.css)

```css
/* 컨테이너 ID 중심 nesting 구조 */
/* 컨테이너 ID: #[component-name]-container (예: #transaction-table-container) */
#transaction-table-container {
    .transaction-table {
        /* Figma에서 추출한 스타일 그대로 적용 */
        display: flex;
        flex-direction: column;
        /* ... */
    }
}
```

#### Preview (preview.html)

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        /* Container: Figma 선택 요소 크기 */
        /* 컨테이너 ID: #[component-name]-container */
        #transaction-table-container {
            width: 524px;   /* Figma 크기 */
            height: 350px;
            overflow: auto;
        }

        /* Component CSS - Figma 스타일 그대로 */
        #transaction-table-container {
            .transaction-table {
                /* Figma에서 추출한 스타일 그대로 */
            }
        }
    </style>
</head>
<body>
    <!-- 컴포넌트만 배치 (page-root 없이) -->
    <div id="transaction-table-container">
        <div class="transaction-table">
            ...
        </div>
    </div>
</body>
</html>
```

### 트레이드오프

#### 장점

- **디자인 일관성:** Figma 스타일을 그대로 유지
- **독립성:** 각 컴포넌트가 자신의 경계 안에서 완결됨
- **조합성:** 컨테이너 크기만 조정하면 어떤 레이아웃에도 배치 가능
- **예측 가능성:** 일관된 구조로 유지보수 용이
- **캡슐화:** 내부 복잡도가 외부에 노출되지 않음

#### 단점

- **DOM 깊이 증가:** 모든 컴포넌트마다 컨테이너 div 추가
- **단순 컴포넌트 오버헤드:** 아이콘 하나도 컨테이너 필요

### 결론

비주얼 빌더에서는 Figma 스타일 유지와 예측 가능한 구조의 가치가 트레이드오프보다 큽니다.
일관된 구조를 유지하면 컴포넌트를 자산으로 쌓을 수 있습니다.

---

## Utils 디렉토리

프레임워크 전반에서 사용되는 유틸리티 모듈입니다. 페이지와 컴포넌트의 코드에서 직접 참조됩니다.

| 파일 | 역할 | 주요 API |
|------|------|----------|
| `fx.js` | 함수형 유틸리티 라이브러리. `go`, `pipe`, `each`, `map`, `filter`, `reduce` 등 이터러블 기반 파이프라인과 지연 평가(`L.*`)를 제공한다. 비동기(Promise) 파이프라인을 자연스럽게 지원한다. | `fx.go`, `fx.each`, `fx.map`, `fx.filter`, `fx.pipe`, `fx.reduce`, `fx.L.*` |
| `Wkit.js` | 컴포넌트/페이지 공통 도구 모음. 2D 이벤트 위임(`bindEvents`/`removeCustomEvents`), 3D 레이캐스팅(`initThreeRaycasting`/`bind3DEvents`), 3D 리소스 정리(`dispose3DTree`/`disposeAllThreeResources`), EventBus 일괄 등록/해제(`onEventBusHandlers`/`offEventBusHandlers`), 데이터 fetch(`fetchData`), 인스턴스 조회(`makeIterator`/`getInstanceByName`/`getInstanceById`) 등을 제공한다. | `Wkit.bindEvents`, `Wkit.removeCustomEvents`, `Wkit.fetchData`, `Wkit.dispose3DTree`, `Wkit.onEventBusHandlers`, `Wkit.offEventBusHandlers` |
| `Weventbus.js` | 경량 이벤트 버스. `on`/`off`/`emit`/`once` 4개 메서드로 컴포넌트 간 느슨한 결합 통신을 제공한다. 컴포넌트에서 DOM 이벤트를 `@` 접두사 커스텀 이벤트로 발행하면, 페이지가 핸들러로 수신하는 구조에서 중개 역할을 한다. | `Weventbus.on`, `Weventbus.off`, `Weventbus.emit`, `Weventbus.once` |
| `GlobalDataPublisher.js` | Topic 기반 pub-sub 데이터 발행기. 페이지가 topic-dataset 매핑을 등록(`registerMapping`)하고, 컴포넌트가 topic을 구독(`subscribe`)하면, `fetchAndPublish`가 데이터를 fetch하여 모든 구독자에게 분배한다. `publish`는 페이지가 여러 dataset을 직접 합성한 결과를 fetch 없이 분배할 때 사용(preview 시뮬에서는 미노출, 금지). | `GlobalDataPublisher.registerMapping`, `GlobalDataPublisher.subscribe`, `GlobalDataPublisher.unsubscribe`, `GlobalDataPublisher.fetchAndPublish`, `GlobalDataPublisher.publish` |

> **출처:** fx.js는 [indongyoo/functional-javascript-01](https://github.com/indongyoo/functional-javascript-01) 기반이다.

---

## 부록 A: 라이프사이클 상세

### 뷰어 훅과 WScript 이벤트 실행 순서

#### 등록 시점

```
1. _onViewerReady()        ← 컴포넌트 소스 (뷰어 전용)
2. WScript REGISTER        ← Codebox
```

#### 소멸 시점

```
1. WScript BEFORE_DESTROY  ← Codebox
2. _onViewerDestroy()      ← 컴포넌트 소스 (뷰어 전용)
3. WScript DESTROY         ← Codebox
```

### 각 시점별 appendElement 접근성

| 시점 | appendElement | 비고 |
|------|---------------|------|
| `_onViewerReady()` | 접근 가능 | 뷰어 전용 초기화 |
| `WScript REGISTER` | 접근 가능 | |
| `WScript BEFORE_DESTROY` | 접근 가능 | 리소스 정리 권장 시점 |
| `_onViewerDestroy()` | 접근 가능 | 뷰어 전용 정리 |
| `WScript DESTROY` | **접근 불가** | 이미 제거됨 |

> **권장:** `WScript DESTROY` 대신 `WScript BEFORE_DESTROY` 또는 `_onViewerDestroy()`에서 리소스 정리를 수행하세요.

---

## 부록 F: 인스턴스 스코프, 생명주기, GC 분석

뷰어 런타임에서 컴포넌트 인스턴스(2D/3D)의 스코프 체인, GC 판정 조건, 그리고
외부에서 인스턴스 속성을 null 처리할 때 발생하는 캡슐화 위반 문제를 분석한 문서.

> **상세 문서:** `INSTANCE_LIFECYCLE_GC.md` (.legacy_ref/docs/ — 본 첨부 미포함)

**핵심 요약:**
- 인스턴스는 레이어의 `_childs[]`와 역참조 Map에 의해 참조되며, `removeChildAll()` 후 GC 대상이 됨
- 인스턴스가 GC되면 모든 속성도 함께 수거되므로, 수동 null 처리는 불필요
- 외부에서 `instance.datasetInfo = null` 등으로 속성을 직접 null 처리하면, 컴포넌트 내부의 정리 로직(`_onViewerDestroy`)이 실패할 수 있음 (캡슐화 위반)

**외부 접근 원칙:**

| 패턴 | 예시 | 안전성 |
|------|------|--------|
| 읽기 | `targetInstance.datasetInfo` | 안전 |
| 메서드 호출 | `targetInstance.showDetail()` | 안전 |
| 공유 리소스 해제 | `unsubscribe(topic, instance)` | 안전 |
| Mixin 소유 쓰기 | `instance._popup.host = null` | 안전 (생성자 = 정리자) |
| 비소유 속성 직접 null | `instance.datasetInfo = null` | 위험 (소유권 불일치) |

