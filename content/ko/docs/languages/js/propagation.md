---
title: 전파
description: JS SDK를 위한 컨텍스트 전파
weight: 65
cSpell:ignore: rolldice
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

{{% docs/languages/propagation %}}

{{% include esm-support-note.md %}}

## 자동 컨텍스트 전파 {#automatic-context-propagation}

[`@opentelemetry/instrumentation-http`](https://www.npmjs.com/package/@opentelemetry/instrumentation-http)나
[`@opentelemetry/instrumentation-express`](https://www.npmjs.com/package/@opentelemetry/instrumentation-express)
같은 [계측 라이브러리](../libraries/)는 서비스 간 컨텍스트를 대신 전파해준다.

[시작하기 가이드](../getting-started/nodejs)를 따라했다면, `/rolldice`
엔드포인트를 조회하는 클라이언트 애플리케이션을 만들 수 있다.

> [!NOTE]
>
> 이 예제는 다른 언어의 시작하기 가이드에 있는 샘플 애플리케이션과도 결합할 수
> 있다. 상관 관계 추적은 서로 다른 언어로 작성된 애플리케이션 사이에서도 차이
> 없이 동작한다.

`dice-client`라는 새 폴더를 생성하는 것으로 시작하고, 필요한 의존성을 설치한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```sh
npm init -y
npm install undici \
  @opentelemetry/instrumentation-undici \
  @opentelemetry/sdk-node
npm install -D tsx  # a tool to run TypeScript (.ts) files directly with node
```

{{% /tab %}} {{% tab JavaScript %}}

```sh
npm init -y
npm install undici \
  @opentelemetry/instrumentation-undici \
  @opentelemetry/sdk-node
```

{{% /tab %}} {{< /tabpane >}}

다음으로, 다음 내용을 담은 `client.ts`(또는 `client.js`)라는 새 파일을 생성한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/* client.ts */
import { NodeSDK } from '@opentelemetry/sdk-node';
import {
  SimpleSpanProcessor,
  ConsoleSpanExporter,
} from '@opentelemetry/sdk-trace-node';
import { UndiciInstrumentation } from '@opentelemetry/instrumentation-undici';

const sdk = new NodeSDK({
  spanProcessors: [new SimpleSpanProcessor(new ConsoleSpanExporter())],
  instrumentations: [new UndiciInstrumentation()],
});
sdk.start();

import { request } from 'undici';

request('http://localhost:8080/rolldice').then((response) => {
  response.body.json().then((json: any) => console.log(json));
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/* instrumentation.mjs */
import { NodeSDK } from '@opentelemetry/sdk-node';
import {
  SimpleSpanProcessor,
  ConsoleSpanExporter,
} from '@opentelemetry/sdk-trace-node';
import { UndiciInstrumentation } from '@opentelemetry/instrumentation-undici';

const sdk = new NodeSDK({
  spanProcessors: [new SimpleSpanProcessor(new ConsoleSpanExporter())],
  instrumentations: [new UndiciInstrumentation()],
});
sdk.start();

const { request } = require('undici');

request('http://localhost:8080/rolldice').then((response) => {
  response.body.json().then((json) => console.log(json));
});
```

{{% /tab %}} {{% /tabpane %}}

[시작하기](../getting-started/nodejs)에서 계측한 버전의 `app.ts`(또는
`app.js`)가 한 셸에서 실행 중인지 확인한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```console
$ npx tsx --import ./instrumentation.ts app.ts
Listening for requests on http://localhost:8080
```

{{% /tab %}} {{% tab JavaScript %}}

```console
$ node --import ./instrumentation.mjs app.js
Listening for requests on http://localhost:8080
```

{{% /tab %}} {{< /tabpane >}}

두 번째 셸을 열어 `client.ts`(또는 `client.js`)를 실행한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```shell
npx tsx client.ts
```

{{% /tab %}} {{% tab JavaScript %}}

```shell
node client.js
```

{{% /tab %}} {{< /tabpane >}}

두 셸 모두 콘솔에 스팬 정보를 출력해야 한다. 클라이언트의 출력은 다음과
비슷하다.

```javascript {hl_lines=[7,11]}
{
  resource: {
    attributes: {
      // ...
    }
  },
  traceId: 'cccd19c3a2d10e589f01bfe2dc896dc2',
  parentSpanContext: undefined,
  traceState: undefined,
  name: 'GET',
  id: '6f64ce484217a7bf',
  kind: 2,
  timestamp: 1718875320295000,
  duration: 19836.833,
  attributes: {
    'url.full': 'http://localhost:8080/rolldice',
    // ...
  },
  status: { code: 0 },
  events: [],
  links: []
}
```

traceId(`cccd19c3a2d10e589f01bfe2dc896dc2`)와 ID(`6f64ce484217a7bf`)를
확인해둔다. 둘 다 클라이언트의 출력에서도 찾을 수 있다.

```javascript {hl_lines=[6,9]}
{
  resource: {
    attributes: {
      // ...
    }
  },
  traceId: 'cccd19c3a2d10e589f01bfe2dc896dc2',
  parentSpanContext: {
    traceId: 'cccd19c3a2d10e589f01bfe2dc896dc2',
    spanId: '6f64ce484217a7bf',
    traceFlags: 1,
    isRemote: true
  },
  traceState: undefined,
  name: 'GET /rolldice',
  id: '027c5c8b916d29da',
  kind: 1,
  timestamp: 1718875320310000,
  duration: 3894.792,
  attributes: {
    'http.url': 'http://localhost:8080/rolldice',
    // ...
  },
  status: { code: 0 },
  events: [],
  links: []
}
```

이제 클라이언트와 서버 애플리케이션이 서로 연결된 스팬을 성공적으로 보고한다.
지금 둘 다 백엔드로 전송하면, 시각화 화면에서 이 의존 관계를 보여준다.

## 수동 컨텍스트 전파 {#manual-context-propagation}

경우에 따라 앞 절에서 설명한 것처럼 컨텍스트를 자동으로 전파할 수 없을 때가
있다. 서비스 간 통신에 사용하는 라이브러리에 맞는 계측 라이브러리가 없을 수도
있고, 설령 그런 라이브러리가 있더라도 충족할 수 없는 요구 사항이 있을 수도 있다.

컨텍스트를 수동으로 전파해야 한다면,
[컨텍스트 API](/docs/languages/js/context)를 사용할 수 있다.

### 일반적인 예제 {#generic-example}

다음의 일반적인 예제는 트레이스 컨텍스트를 수동으로 전파하는 방법을 보여준다.

먼저, 전송하는 서비스에서 현재 `context`를 주입해야 한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```typescript
// Sending service
import { context, propagation, trace } from '@opentelemetry/api';

// Define an interface for the output object that will hold the trace information.
interface Carrier {
  traceparent?: string;
  tracestate?: string;
}

// Create an output object that conforms to that interface.
const output: Carrier = {};

// Serialize the traceparent and tracestate from context into
// an output object.
//
// This example uses the active trace context, but you can
// use whatever context is appropriate to your scenario.
propagation.inject(context.active(), output);

// Extract the traceparent and tracestate values from the output object.
const { traceparent, tracestate } = output;

// You can then pass the traceparent and tracestate
// data to whatever mechanism you use to propagate
// across services.
```

{{% /tab %}} {{% tab JavaScript %}}

```js
// Sending service
const { context, propagation } = require('@opentelemetry/api');
const output = {};

// Serialize the traceparent and tracestate from context into
// an output object.
//
// This example uses the active trace context, but you can
// use whatever context is appropriate to your scenario.
propagation.inject(context.active(), output);

const { traceparent, tracestate } = output;
// You can then pass the traceparent and tracestate
// data to whatever mechanism you use to propagate
// across services.
```

{{% /tab %}} {{< /tabpane >}}

수신하는 서비스에서는 (예를 들어 파싱된 HTTP 헤더로부터) `context`를 추출한
다음, 이를 현재 트레이스 컨텍스트로 설정해야 한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```typescript
// Receiving service
import {
  type Context,
  propagation,
  trace,
  Span,
  context,
} from '@opentelemetry/api';

// Define an interface for the input object that includes 'traceparent' & 'tracestate'.
interface Carrier {
  traceparent?: string;
  tracestate?: string;
}

// Assume "input" is an object with 'traceparent' & 'tracestate' keys.
const input: Carrier = {};

// Extracts the 'traceparent' and 'tracestate' data into a context object.
//
// You can then treat this context as the active context for your
// traces.
let activeContext: Context = propagation.extract(context.active(), input);

let tracer = trace.getTracer('app-name');

let span: Span = tracer.startSpan(
  spanName,
  {
    attributes: {},
  },
  activeContext,
);

// Set the created span as active in the deserialized context.
trace.setSpan(activeContext, span);
```

{{% /tab %}} {{% tab JavaScript %}}

```js
// Receiving service
import { context, propagation, trace } from '@opentelemetry/api';

// Assume "input" is an object with 'traceparent' & 'tracestate' keys
const input = {};

// Extracts the 'traceparent' and 'tracestate' data into a context object.
//
// You can then treat this context as the active context for your
// traces.
let activeContext = propagation.extract(context.active(), input);

let tracer = trace.getTracer('app-name');

let span = tracer.startSpan(
  spanName,
  {
    attributes: {},
  },
  activeContext,
);

// Set the created span as active in the deserialized context.
trace.setSpan(activeContext, span);
```

{{% /tab %}} {{< /tabpane >}}

이렇게 역직렬화된 활성 컨텍스트가 있으면, 다른 서비스로부터의 동일한 트레이스에
속하는 스팬을 생성할 수 있다.

[Context](/docs/languages/js/context) API를 사용해 역직렬화된 컨텍스트를 다른
방식으로 수정하거나 설정할 수도 있다.

### 커스텀 프로토콜 예제 {#custom-protocol-example}

컨텍스트를 수동으로 전파해야 하는 흔한 사용 사례는 서비스 간 통신에 커스텀
프로토콜을 사용하는 경우이다. 다음 예제는 기본적인 텍스트 기반 TCP 프로토콜을
사용해 한 서비스에서 다른 서비스로 직렬화된 객체를 전송한다.

`propagation-example`라는 새 폴더를 생성하고 다음과 같이 의존성을 설정하여
초기화하는 것으로 시작한다.

```shell
npm init -y
npm install @opentelemetry/api @opentelemetry/sdk-node
```

다음으로, 다음 내용을 담은 `client.js`와 `server.js` 파일을 생성한다.

```javascript
// client.js
const net = require('net');
const { context, propagation, trace } = require('@opentelemetry/api');

let tracer = trace.getTracer('client');

// Connect to the server
const client = net.createConnection({ port: 8124 }, () => {
  // Send the serialized object to the server
  let span = tracer.startActiveSpan('send', { kind: 1 }, (span) => {
    const output = {};
    propagation.inject(context.active(), output);
    const { traceparent, tracestate } = output;

    const objToSend = { key: 'value' };

    if (traceparent) {
      objToSend._meta = { traceparent, tracestate };
    }

    client.write(JSON.stringify(objToSend), () => {
      client.end();
      span.end();
    });
  });
});
```

```javascript
// server.js
const net = require('net');
const { context, propagation, trace } = require('@opentelemetry/api');

let tracer = trace.getTracer('server');

const server = net.createServer((socket) => {
  socket.on('data', (data) => {
    const message = data.toString();
    // Parse the JSON object received from the client
    try {
      const json = JSON.parse(message);
      let activeContext = context.active();
      if (json._meta) {
        activeContext = propagation.extract(context.active(), json._meta);
        delete json._meta;
      }
      span = tracer.startSpan('receive', { kind: 1 }, activeContext);
      trace.setSpan(activeContext, span);
      console.log('Parsed JSON:', json);
    } catch (e) {
      console.error('Error parsing JSON:', e.message);
    } finally {
      span.end();
    }
  });
});

// Listen on port 8124
server.listen(8124, () => {
  console.log('Server listening on port 8124');
});
```

첫 번째 셸을 열어 서버를 실행한다.

```console
$ node server.js
Server listening on port 8124
```

그런 다음 두 번째 셸에서 클라이언트를 실행한다.

```shell
node client.js
```

클라이언트는 즉시 종료되어야 하며, 서버는 다음을 출력해야 한다.

```text
Parsed JSON: { key: 'value' }
```

지금까지의 예제는 오픈텔레메트리 API에만 의존했으므로, 이에 대한 모든 호출은
[no-op 명령](<https://en.wikipedia.org/wiki/NOP_(code)>)이며 클라이언트와 서버는
오픈텔레메트리를 사용하지 않는 것처럼 동작한다.

> [!IMPORTANT]
>
> 서버와 클라이언트 코드가 라이브러리라면 이 점이 특히 중요한데, 라이브러리는
> 오픈텔레메트리 API만 사용해야 하기 때문이다. 그 이유를 이해하려면
> [라이브러리에 계측을 추가하는 방법에 대한 개념 페이지](/docs/concepts/instrumentation/libraries/)를
> 읽어본다.

오픈텔레메트리를 활성화하고 컨텍스트 전파가 동작하는 모습을 보려면, 다음 내용을
담은 `instrumentation.js`라는 추가 파일을 생성한다.

```javascript
// instrumentation.mjs
import { NodeSDK } from '@opentelemetry/sdk-node';
import {
  ConsoleSpanExporter,
  SimpleSpanProcessor,
} from '@opentelemetry/sdk-trace-node';

const sdk = new NodeSDK({
  spanProcessors: [new SimpleSpanProcessor(new ConsoleSpanExporter())],
});

sdk.start();
```

이 파일을 사용해 계측을 활성화한 상태로 서버와 클라이언트를 모두 실행한다.

```console
$ node --import ./instrumentation.mjs server.js
Server listening on port 8124
```

그리고

```shell
node --import ./instrumentation.mjs client.js
```

클라이언트가 서버로 데이터를 전송하고 종료되면, 두 셸의 콘솔 출력에서 스팬을
확인할 수 있어야 한다.

클라이언트의 출력은 다음과 같다.

```javascript {hl_lines=[7,11]}
{
  resource: {
    attributes: {
      // ...
    }
  },
  traceId: '4b5367d540726a70afdbaf49240e6597',
  parentId: undefined,
  traceState: undefined,
  name: 'send',
  id: '92f125fa335505ec',
  kind: 1,
  timestamp: 1718879823424000,
  duration: 1054.583,
  // ...
}
```

서버의 출력은 다음과 같다.

```javascript {hl_lines=[7,8]}
{
  resource: {
    attributes: {
      // ...
    }
  },
  traceId: '4b5367d540726a70afdbaf49240e6597',
  parentId: '92f125fa335505ec',
  traceState: undefined,
  name: 'receive',
  id: '53da0c5f03cb36e5',
  kind: 1,
  timestamp: 1718879823426000,
  duration: 959.541,
  // ...
}
```

[수동 예제](#manual-context-propagation)와 마찬가지로, 스팬은 `traceId`와
`id`/`parentId`를 사용해 연결된다.

## 다음 단계 {#next-steps}

전파에 대해 더 알아보려면
[전파자 API 명세](/docs/specs/otel/context/api-propagators/)를 읽어본다.
