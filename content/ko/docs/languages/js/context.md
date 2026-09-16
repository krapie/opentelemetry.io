---
title: 컨텍스트
description: 오픈텔레메트리(OpenTelemetry) JavaScript 컨텍스트(Context) API 문서
aliases: [api/context]
weight: 60
default_lang_commit: bdaf520bae4197ff1384e9d9f7fd9b85cbceedbf
---

오픈텔레메트리(OpenTelemetry)가 동작하려면 중요한 텔레메트리 데이터를 저장하고
전파해야 한다. 예를 들어 요청을 수신하여 스팬이 시작되면, 그 자식 스팬을
생성하는 컴포넌트에서 해당 스팬을 사용할 수 있어야 한다. 이 문제를 해결하기 위해
오픈텔레메트리는 스팬을 컨텍스트에 저장한다. 이 문서는 JavaScript용
오픈텔레메트리 컨텍스트(Context) API와 그 사용법을 설명한다.

더 많은 정보:

- [컨텍스트 명세](/docs/specs/otel/context/)
- [컨텍스트 API 참조](https://open-telemetry.github.io/opentelemetry-js/interfaces/_opentelemetry_api._opentelemetry_api.ContextAPI.html)

## 컨텍스트 매니저 {#context-manager}

컨텍스트 API는 동작을 위해 컨텍스트 매니저(context manager)에 의존한다. 이
문서의 예시는 이미 컨텍스트 매니저를 구성했다고 가정한다. 일반적으로 컨텍스트
매니저는 SDK가 제공하지만, 다음과 같이 직접 등록할 수도 있다.

```typescript
import * as api from '@opentelemetry/api';
import { AsyncHooksContextManager } from '@opentelemetry/context-async-hooks';

const contextManager = new AsyncHooksContextManager();
contextManager.enable();
api.context.setGlobalContextManager(contextManager);
```

## 루트 컨텍스트 {#root-context}

`ROOT_CONTEXT`는 빈 컨텍스트이다. 활성 컨텍스트가 없으면 `ROOT_CONTEXT`가
활성화된다. 활성 컨텍스트는 아래 [활성 컨텍스트](#active-context)에서 설명한다.

## 컨텍스트 키 {#context-keys}

컨텍스트 엔트리는 키-값 쌍이다. 키는 `api.createContextKey(description)`을
호출하여 생성할 수 있다.

```typescript
import * as api from '@opentelemetry/api';

const key1 = api.createContextKey('My first key');
const key2 = api.createContextKey('My second key');
```

## 기본 연산 {#basic-operations}

### 엔트리 가져오기 {#get-entry}

엔트리는 `context.getValue(key)` 메서드로 접근한다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('some key');
// ROOT_CONTEXT is the empty context
const ctx = api.ROOT_CONTEXT;

const value = ctx.getValue(key);
```

### 엔트리 설정하기 {#set-entry}

엔트리는 `context.setValue(key, value)` 메서드를 사용해 생성한다. 컨텍스트
엔트리를 설정하면 이전 컨텍스트의 모든 엔트리를 포함하되 새 엔트리가 추가된
새로운 컨텍스트가 생성된다. 컨텍스트 엔트리를 설정해도 이전 컨텍스트는 수정되지
않는다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('some key');
const ctx = api.ROOT_CONTEXT;

// add a new entry
const ctx2 = ctx.setValue(key, 'context 2');

// ctx2 contains the new entry
console.log(ctx2.getValue(key)); // "context 2"

// ctx is unchanged
console.log(ctx.getValue(key)); // undefined
```

### 엔트리 삭제하기 {#delete-entry}

엔트리는 `context.deleteValue(key)`를 호출해 제거한다. 컨텍스트 엔트리를
삭제하면 이전 컨텍스트의 모든 엔트리를 포함하되 키로 식별된 엔트리가 제외된
새로운 컨텍스트가 생성된다. 컨텍스트 엔트리를 삭제해도 이전 컨텍스트는 수정되지
않는다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('some key');
const ctx = api.ROOT_CONTEXT;
const ctx2 = ctx.setValue(key, 'context 2');

// remove the entry
const ctx3 = ctx2.deleteValue(key);

// ctx3 does not contain the entry
console.log(ctx3.getValue(key)); // undefined

// ctx2 is unchanged
console.log(ctx2.getValue(key)); // "context 2"
// ctx is unchanged
console.log(ctx.getValue(key)); // undefined
```

## 활성 컨텍스트 {#active-context}

**중요**: 이는 컨텍스트 매니저를 구성했다고 가정한다. 컨텍스트 매니저가 없으면
`api.context.active()`는 _항상_ `ROOT_CONTEXT`를 반환한다.

활성 컨텍스트는 `api.context.active()`가 반환하는 컨텍스트이다. 컨텍스트 객체는
단일 실행 스레드를 트레이싱하는 트레이싱 컴포넌트들이 서로 통신하고 트레이스가
성공적으로 생성되도록 보장하는 데 사용하는 엔트리를 담고 있다. 예를 들어 스팬이
생성되면 이는 컨텍스트에 추가될 수 있다. 이후 다른 스팬이 생성되면, 컨텍스트에
있는 스팬을 자신의 부모 스팬으로 사용할 수 있다. 이는 node에서는
[async_hooks](https://nodejs.org/api/async_hooks.html)나
[AsyncLocalStorage](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage)와
같은 메커니즘을 통해, 웹에서는
[zone.js](https://github.com/angular/angular/tree/main/packages/zone.js)를 통해
단일 실행 전반에 걸쳐 컨텍스트를 전파함으로써 이루어진다. 활성 컨텍스트가 없으면
빈 컨텍스트 객체인 `ROOT_CONTEXT`가 반환된다.

### 활성 컨텍스트 가져오기 {#get-active-context}

활성 컨텍스트는 `api.context.active()`가 반환하는 컨텍스트이다.

```typescript
import * as api from '@opentelemetry/api';

// Returns the active context
// If no context is active, the ROOT_CONTEXT is returned
const ctx = api.context.active();
```

### 활성 컨텍스트 설정하기 {#set-active-context}

컨텍스트는 `api.context.with(ctx, callback)`을 사용해 활성화할 수 있다.
`callback`이 실행되는 동안 `with`에 전달된 컨텍스트가 `context.active`에 의해
반환된다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('Key to store a value');
const ctx = api.context.active();

api.context.with(ctx.setValue(key, 'context 2'), async () => {
  // "context 2" is active
  console.log(api.context.active().getValue(key)); // "context 2"
});
```

`api.context.with(context, callback)`의 반환값은 콜백의 반환값이다. 콜백은 항상
동기적으로 호출된다.

```typescript
import * as api from '@opentelemetry/api';

const name = await api.context.with(api.context.active(), async () => {
  const row = await db.getSomeValue();
  return row['name'];
});

console.log(name); // name returned by the db
```

활성 컨텍스트 실행은 중첩될 수 있다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('Key to store a value');
const ctx = api.context.active();

// No context is active
console.log(api.context.active().getValue(key)); // undefined

api.context.with(ctx.setValue(key, 'context 2'), () => {
  // "context 2" is active
  console.log(api.context.active().getValue(key)); // "context 2"
  api.context.with(ctx.setValue(key, 'context 3'), () => {
    // "context 3" is active
    console.log(api.context.active().getValue(key)); // "context 3"
  });
  // "context 2" is active
  console.log(api.context.active().getValue(key)); // "context 2"
});

// No context is active
console.log(api.context.active().getValue(key)); // undefined
```

### 예시 {#example}

이 좀 더 복잡한 예시는 컨텍스트가 수정되지 않고 새로운 컨텍스트 객체가 생성되는
방식을 보여준다.

```typescript
import * as api from '@opentelemetry/api';

const key = api.createContextKey('Key to store a value');

const ctx = api.context.active(); // Returns ROOT_CONTEXT when no context is active
const ctx2 = ctx.setValue(key, 'context 2'); // does not modify ctx

console.log(ctx.getValue(key)); //? undefined
console.log(ctx2.getValue(key)); //? "context 2"

const ret = api.context.with(ctx2, () => {
  const ctx3 = api.context.active().setValue(key, 'context 3');

  console.log(api.context.active().getValue(key)); //? "context 2"
  console.log(ctx.getValue(key)); //? undefined
  console.log(ctx2.getValue(key)); //? "context 2"
  console.log(ctx3.getValue(key)); //? "context 3"

  api.context.with(ctx3, () => {
    console.log(api.context.active().getValue(key)); //? "context 3"
  });
  console.log(api.context.active().getValue(key)); //? "context 2"

  return 'return value';
});

// The value returned by the callback is returned to the caller
console.log(ret); //? "return value"
```
