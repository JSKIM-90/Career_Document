# fx.go / reduce 기반 에러 핸들링 가이드 (v2)

## 핵심 이해 코드

```javascript
fx.go(
    [
        Promise.resolve(1),
        Promise.reject('new error').catch(err => (console.warn(err), 'then again')),
        Promise.resolve(2)
    ],
    fx.each(console.log)
).catch(err => console.log(err,'-----outside'))
```

## 0. 목적

이 문서는 fx.go, reduce, fx.each로 구성된 파이프라인에서 에러가 언제 전파되어 파이프라인을 중단시키는지, 그리고 어디에서 어떤 방식으로 처리해야 하는지를 규약으로 정의한다.

## 1. 핵심 원리 (반드시 이해해야 하는 3가지)

### 1.1 async는 "resolve/reject를 자동으로 만든다"

async function은 반환값이 무엇이든 Promise를 반환한다.

- 정상 종료 → `Promise.resolve(value)` (value 없으면 undefined)
- 예외 발생 / throw → `Promise.reject(error)`

따라서 fetchAndPublish는 실패 시 반드시 rejected Promise가 된다.

### 1.2 catch는 에러 처리 연산자가 아니라 "상태를 다시 쓰는 then"이다

다음은 기능적으로 동일하다:

```javascript
p.catch(onRejected)
// ===
p.then(undefined, onRejected)
```

그리고 onRejected가 throw 하지 않으면, catch는 에러를 소비하고 fulfilled Promise로 복구한다.

- `catch(() => {})` → fulfilled(undefined)
- `catch(e => fallback)` → fulfilled(fallback)
- `catch(e => { throw e })` → rejected(에러 유지)

즉, **중간 catch는 에러를 "없애는" 것이 아니라, rejected를 fulfilled로 "변환(복구)"한다.**

### 1.3 reduce / fx.go는 rejected를 "처리하지 않고 전파"한다 (fail-fast)

reduce는 rejected Promise를 만나면 내부 recur가 더 이상 실행되지 않아 즉시 중단하고, 그 rejected를 그대로 호출자에게 반환한다.

- ✅ "reduce가 reject를 받는다"
- ✅ "하지만 복구하지 않는다"
- ✅ "그래서 fail-fast로 멈춘다"

## 2. 규칙: 에러 처리 책임

### 2.1 유틸은 에러를 복구하지 않는다

fx.go, reduce, each, reduceF는 에러를 삼키지 않고 전파한다.
(nop은 필터 스킵을 위한 내부 시그널로만 예외 처리)

### 2.2 호출부는 반드시 에러 전략을 명시해야 한다

파이프라인 호출부는 반드시 아래 중 하나를 선택해야 한다:

- **Fail-fast**: 한 번 실패하면 전체 중단
- **Fail-safe(격리)**: 실패를 개별 소비하여 전체는 계속 진행

## 3. 패턴 A: Fail-fast (한 번 실패하면 전체 중단)

### 3.1 언제 쓰나

- "초기 로딩이 완전해야만 화면을 보여준다"
- "중간 실패가 이후 단계의 의미를 무효화한다"
- "데이터 파이프라인(변환/집계)"

### 3.2 규약

- 중간에서 `.catch(...)`로 에러를 소비하지 않는다.
- 최상단(호출부)에서만 `.catch(...)` 또는 try/catch로 처리한다.

### 3.3 예시

```javascript
fx.go(
  mappings,
  fx.each(({ topic }) => GlobalDataPublisher.fetchAndPublish(topic, page))
).catch(e => {
  console.error('[page_loaded] init fetch failed', e);
  // UI fail state, retry, fallback navigation 등 도메인 처리
});
```

**효과**: fetchAndPublish가 throw → reduce 체인이 rejected를 받음 → recur가 실행되지 않음 → 이후 topic 실행이 중단될 수 있음.

## 4. 패턴 B: Fail-safe (topic별 격리, "전부 시도" 보장)

### 4.1 언제 쓰나

- topic들이 서로 독립적이고 "가능한 것만" 표시해도 된다
- 대시보드/모니터링처럼 일부 데이터 유실을 허용한다
- "전부 요청을 시도하는 것"이 UX/운영상 중요하다

### 4.2 핵심 규약

- 실패를 중간에서 소비하여 fulfilled로 복구한다.
- 그렇지 않으면 fail-fast로 전체가 중단된다.

### 4.3 예시 (직렬, 전부 시도)

```javascript
fx.go(
  mappings,
  fx.each(({ topic }) =>
    GlobalDataPublisher.fetchAndPublish(topic, page)
      .catch(e => console.error(`[topic:${topic}]`, e))
  )
);
```

- 각 항목은 Promise를 반환하므로 each가 대기한다.
- 하지만 `.catch(...)`가 에러를 소비하므로 전체는 reject되지 않는다.
- 결과적으로 끝까지 순회한다.

**주의**: `.catch(console.log)`는 "로깅"이 아니라 "에러를 성공(undefined)으로 변환"하는 동작이다.

## 5. 중요한 함정: Promise를 반환하지 않으면 파이프라인은 비동기를 모른다

### 5.1 금지 패턴

```javascript
fx.each(() => {
  fetch(url); // return 없음
})
```

- f(a)가 undefined를 반환
- reduce는 비동기 체인을 만들지 못한다
- 요청 실패는 파이프라인 밖에서 unhandled rejection으로 터질 수 있다

### 5.2 올바른 패턴

```javascript
fx.each(() => fetch(url))
```

또는(격리 필요 시):

```javascript
fx.each(() => fetch(url).catch(log))
```
## 6. interval / 이벤트 핸들러의 필수 규칙 (Unhandled 방지)

### 6.1 규약

interval/이벤트 핸들러 내부에서 호출하는 Promise는 반드시 catch로 소비하거나, async 함수라면 내부에서 try/catch로 처리한다.

### 6.2 예시

```javascript
const run = () =>
  GlobalDataPublisher.fetchAndPublish(topic, page, params)
    .catch(e => {
      console.error(`[fetchAndPublish:${topic}]`, e);
      // retry/backoff/toast/metric 등
    });
setInterval(run, refreshMs);
run();
```
## 7. 체크리스트 (리뷰용)

- [ ] 모든 fx.go(...) 호출부에 **정책(Fail-fast vs Fail-safe)**이 명시돼 있는가?
- [ ] 비동기 함수는 반드시 Promise를 return하는가? (return 누락 없음)
- [ ] 중간 `.catch(...)`가 "의도적 복구"인가? (그렇다면 복구값 의미가 정의돼 있는가?)
- [ ] interval/이벤트 핸들러에서 unhandled rejection 가능성이 제거돼 있는가?
- [ ] "전부 시도"가 요구사항이면, fail-fast 경로가 섞여 있지 않은가?

## 8. 한 줄 요약

- reduce/fx.go는 rejected를 복구하지 않으며 rejected를 받으면 fail-fast로 중단한다.
- 중간 catch는 에러를 "잡는" 게 아니라 rejected를 fulfilled로 복구하여 순회를 계속하게 만든다.
- "전부 요청 시도"를 보장하려면 Promise를 반환하고, 필요한 곳에서 개별 catch로 격리해야 한다.