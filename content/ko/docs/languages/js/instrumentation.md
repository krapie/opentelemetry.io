---
title: 계측
aliases:
  - /docs/languages/js/api/tracing
  - manual
weight: 30
description: 오픈텔레메트리(OpenTelemetry) JavaScript를 위한 계측
cSpell:ignore: dicelib Millis rolldice
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

{{% include instrumentation-intro.md %}}

> [!NOTE]
>
> 이 페이지에서는 트레이스, 메트릭, 로그를 코드에 _수동으로_ 추가하는 방법을
> 배운다. 하지만 계측 방식을 한 가지만 사용해야 하는 것은 아니다.
> [자동 계측](/docs/zero-code/js/)으로 시작한 다음, 필요에 따라 수동 계측으로
> 코드를 보강한다.
>
> 또한 코드가 의존하는 라이브러리는 직접 계측 코드를 작성하지 않아도 될 수 있다.
> 오픈텔레메트리가 _네이티브로_ 내장되어 있거나
> [계측 라이브러리](/docs/languages/js/libraries/)를 활용할 수 있기 때문이다.

## 예제 앱 준비 {#example-app}

이 페이지는 수동 계측을 배우는 데 도움이 되도록
[시작하기](/docs/languages/js/getting-started/nodejs/)의 예제 앱을 변형한 버전을
사용한다.

예제 앱을 반드시 사용할 필요는 없다. 자신의 앱이나 라이브러리를 계측하고 싶다면,
여기 있는 안내를 참고해 자신의 코드에 맞게 과정을 조정한다.

{{% include esm-support-note.md %}}

### 의존성 {#example-app-dependencies}

새 디렉터리에 빈 NPM `package.json` 파일을 생성한다.

```shell
npm init -y
```

다음으로 Express 의존성을 설치한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```sh
npm install express @types/express
npm install -D tsx  # a tool to run TypeScript (.ts) files directly with node
```

{{% /tab %}} {{% tab JavaScript %}}

```sh
npm install express
```

{{% /tab %}} {{< /tabpane >}}

### HTTP 서버 생성 및 실행 {#create-and-launch-an-http-server}

_라이브러리_를 계측하는 것과 독립형 _앱_을 계측하는 것의 차이를 보여주기 위해,
주사위 굴리기 로직을 _라이브러리 파일_로 분리하고, 이를 _앱 파일_에서 의존성으로
가져오도록 한다.

`dice.ts`라는 이름의 _라이브러리 파일_(TypeScript를 사용하지 않는다면
`dice.js`)을 생성하고 다음 코드를 추가한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*dice.ts*/
function rollOnce(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

export function rollTheDice(rolls: number, min: number, max: number) {
  const result: number[] = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*dice.js*/
function rollOnce(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

function rollTheDice(rolls, min, max) {
  const result = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}

module.exports = { rollTheDice };
```

{{% /tab %}} {{< /tabpane >}}

`app.ts`라는 이름의 _앱 파일_(TypeScript를 사용하지 않는다면 `app.js`)을
생성하고 다음 코드를 추가한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*app.ts*/
import express, { type Express } from 'express';
import { rollTheDice } from './dice';

const PORT: number = parseInt(process.env.PORT || '8080');
const app: Express = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*app.js*/
const express = require('express');
const { rollTheDice } = require('./dice.js');

const PORT = parseInt(process.env.PORT || '8080');
const app = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{< /tabpane >}}

정상적으로 동작하는지 확인하려면, 다음 명령어로 애플리케이션을 실행하고 웹
브라우저에서 <http://localhost:8080/rolldice?rolls=12>를 연다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```console
$ npx tsx app.ts
Listening for requests on http://localhost:8080
```

{{% /tab %}} {{% tab JavaScript %}}

```console
$ node app.js
Listening for requests on http://localhost:8080
```

{{% /tab %}} {{< /tabpane >}}

## 수동 계측 설정 {#manual-instrumentation-setup}

### 의존성 {#dependencies}

오픈텔레메트리 API 패키지를 설치한다.

```shell
npm install @opentelemetry/api @opentelemetry/resources @opentelemetry/semantic-conventions
```

### SDK 초기화 {#initialize-the-sdk}

> [!NB] 라이브러리를 계측하는 경우 **이 단계를 건너뛴다**.

Node.js 애플리케이션을 계측한다면
[Node.js용 오픈텔레메트리 SDK](https://www.npmjs.com/package/@opentelemetry/sdk-node)를
설치한다.

```shell
npm install @opentelemetry/sdk-node
```

애플리케이션의 다른 모듈이 로드되기 전에 반드시 SDK를 초기화해야 한다. SDK를
초기화하지 않거나 너무 늦게 초기화하면, API에서 트레이서나 미터를 얻으려는
라이브러리에는 아무 동작도 하지 않는(no-op) 구현체가 제공된다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*instrumentation.ts*/
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter } from '@opentelemetry/sdk-trace-node';
import {
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} from '@opentelemetry/sdk-metrics';
import { resourceFromAttributes } from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'yourServiceName',
    [ATTR_SERVICE_VERSION]: '1.0',
  }),
  traceExporter: new ConsoleSpanExporter(),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new ConsoleMetricExporter(),
  }),
});

sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*instrumentation.mjs*/
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter } from '@opentelemetry/sdk-trace-node';
import {
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} from '@opentelemetry/sdk-metrics';
import { resourceFromAttributes } from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'dice-server',
    [ATTR_SERVICE_VERSION]: '0.1.0',
  }),
  traceExporter: new ConsoleSpanExporter(),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new ConsoleMetricExporter(),
  }),
});

sdk.start();
```

{{% /tab %}} {{< /tabpane >}}

디버깅과 로컬 개발을 위해, 다음 예제는 텔레메트리를 콘솔로 내보낸다. 수동 계측
설정을 마친 후에는,
[앱의 텔레메트리 데이터를 내보내](/docs/languages/js/exporters/) 하나 이상의
텔레메트리 백엔드로 전송하도록 적절한 익스포터를 구성해야 한다.

이 예제는 서비스의 논리적 이름을 담는 필수 SDK 기본 속성 `service.name`과,
서비스 API나 구현체의 버전을 담는 선택적(하지만 강력히 권장하는!) 속성
`service.version`도 함께 설정한다.

리소스 속성을 설정하는 다른 방법도 있다. 자세한 내용은
[리소스](/docs/languages/js/resources/)를 참고한다.

> [!NOTE]
>
> `--import instrumentation.ts`(TypeScript)를 사용하는 다음 예제는 Node.js v20
> 이상이 필요하다. Node.js v18을 사용한다면 JavaScript 예제를 사용한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```sh
npx tsx --import ./instrumentation.ts app.ts
```

{{% /tab %}} {{% tab JavaScript %}}

```sh
node --import ./instrumentation.mjs app.js
```

{{% /tab %}} {{< /tabpane >}}

이 기본 설정만으로는 아직 앱에 아무 효과가 없다. [트레이스](#traces),
[메트릭](#metrics), [로그](#logs)에 대한 코드를 추가해야 한다.

의존성에 대한 텔레메트리 데이터를 생성하려면 Node.js용 오픈텔레메트리 SDK에 계측
라이브러리를 등록할 수 있다. 자세한 내용은
[라이브러리](/docs/languages/js/libraries/)를 참고한다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initialize-tracing}

> [!NB] 라이브러리를 계측하는 경우 **이 단계를 건너뛴다**.

앱에서 [트레이싱](/docs/concepts/signals/traces/)을 활성화하려면,
[`Tracer`](/docs/concepts/signals/traces/#tracer)를 생성할 수 있게 해주는
초기화된 [`TracerProvider`](/docs/concepts/signals/traces/#tracer-provider)가
필요하다.

`TracerProvider`가 생성되지 않으면, 트레이싱을 위한 오픈텔레메트리 API는 no-op
구현체를 사용하여 데이터를 생성하지 못한다. 다음에서 설명하듯이, Node와
브라우저에서 모든 SDK 초기화 코드를 포함하도록 `instrumentation.ts`(또는
`instrumentation.js`) 파일을 수정한다.

#### Node.js {#nodejs}

위에서 [SDK 초기화](#initialize-the-sdk) 안내를 따라했다면, 이미
`TracerProvider`가 설정되어 있다. 이제 [트레이서 얻기](#acquiring-a-tracer)로
넘어가면 된다.

#### 브라우저 {#browser}

{{% include browser-instrumentation-warning.md %}}

먼저 알맞은 패키지가 설치되어 있는지 확인한다.

```shell
npm install @opentelemetry/sdk-trace-web
```

다음으로, 모든 SDK 초기화 코드를 담도록 `instrumentation.ts`(또는
`instrumentation.js`)를 업데이트한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import {
  defaultResource,
  resourceFromAttributes,
} from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import {
  BatchSpanProcessor,
  ConsoleSpanExporter,
} from '@opentelemetry/sdk-trace-base';

const resource = defaultResource().merge(
  resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'service-name-here',
    [ATTR_SERVICE_VERSION]: '0.1.0',
  }),
);

const exporter = new ConsoleSpanExporter();
const processor = new BatchSpanProcessor(exporter);

const provider = new WebTracerProvider({
  resource: resource,
  spanProcessors: [processor],
});

provider.register();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');
const {
  defaultResource,
  resourceFromAttributes,
} = require('@opentelemetry/resources');
const {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} = require('@opentelemetry/semantic-conventions');
const { WebTracerProvider } = require('@opentelemetry/sdk-trace-web');
const {
  ConsoleSpanExporter,
  BatchSpanProcessor,
} = require('@opentelemetry/sdk-trace-base');

const resource = defaultResource().merge(
  resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'service-name-here',
    [ATTR_SERVICE_VERSION]: '0.1.0',
  }),
);

const exporter = new ConsoleSpanExporter();
const processor = new BatchSpanProcessor(exporter);

const provider = new WebTracerProvider({
  resource: resource,
  spanProcessors: [processor],
});

provider.register();
```

{{% /tab %}} {{< /tabpane >}}

웹 애플리케이션의 나머지 부분에서 트레이싱을 사용할 수 있으려면 이 파일을 웹
애플리케이션과 함께 번들링해야 한다.

이것만으로는 아직 앱에 아무 효과가 없다. 앱에서 텔레메트리가 방출되게 하려면
[스팬을 생성](#create-spans)해야 한다.

#### 알맞은 스팬 프로세서 선택하기 {#picking-the-right-span-processor}

기본적으로 Node SDK는 `BatchSpanProcessor`를 사용하며, Web SDK 예제에서도 이
스팬 프로세서를 선택한다. `BatchSpanProcessor`는 스팬을 내보내기 전에 배치로
묶어 처리한다. 대부분의 애플리케이션에서는 이 프로세서를 사용하는 것이 적절하다.

반면 `SimpleSpanProcessor`는 스팬이 생성되는 즉시 처리한다. 즉, 5개의 스팬을
생성하면 코드에서 다음 스팬이 생성되기 전에 각 스팬이 처리되고 내보내진다. 이는
배치를 잃을 위험을 감수하고 싶지 않은 경우나, 개발 중에 오픈텔레메트리를
실험해볼 때 도움이 될 수 있다. 하지만 특히 네트워크를 통해 스팬을 내보내는 경우
상당한 오버헤드가 발생할 수 있다. 스팬을 생성하는 호출이 있을 때마다 앱의 실행이
계속되기 전에 처리되어 네트워크로 전송되기 때문이다.

대부분의 경우 `SimpleSpanProcessor`보다 `BatchSpanProcessor`를 사용하는 것이
좋다.

### 트레이서 얻기 {#acquiring-a-tracer}

애플리케이션에서 수동 트레이싱 코드를 작성하는 곳이라면 어디서든 `getTracer`를
호출해 트레이서를 얻어야 한다. 예를 들면 다음과 같다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry from '@opentelemetry/api';
//...

const tracer = opentelemetry.trace.getTracer(
  'instrumentation-scope-name',
  'instrumentation-scope-version',
);

// You can now use a 'tracer' to do tracing!
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');
//...

const tracer = opentelemetry.trace.getTracer(
  'instrumentation-scope-name',
  'instrumentation-scope-version',
);

// You can now use a 'tracer' to do tracing!
```

{{% /tab %}} {{< /tabpane >}}

`instrumentation-scope-name`과 `instrumentation-scope-version`의 값은 패키지,
모듈, 클래스 이름처럼 [계측 스코프](/docs/concepts/instrumentation-scope/)를
고유하게 식별할 수 있어야 한다. 이름은 필수이지만, 버전은 선택 사항이라도
지정하는 것이 권장된다.

앱의 나머지 부분으로 `tracer` 인스턴스를 내보내기보다는, 필요할 때 앱에서 직접
`getTracer`를 호출하는 것이 일반적으로 권장된다. 이렇게 하면 다른 필수 의존성이
관련될 때 발생할 수 있는 까다로운 애플리케이션 로드 문제를 피할 수 있다.

[예제 앱](#example-app)의 경우, 적절한 계측 스코프로 트레이서를 얻을 수 있는
곳이 두 군데 있다.

첫 번째는 _애플리케이션 파일_ `app.ts`(또는 `app.js`)이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts {hl_lines=[6]}
/*app.ts*/
import { trace } from '@opentelemetry/api';
import express, { type Express } from 'express';
import { rollTheDice } from './dice';

const tracer = trace.getTracer('dice-server', '0.1.0');

const PORT: number = parseInt(process.env.PORT || '8080');
const app: Express = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js {hl_lines=[6]}
/*app.js*/
const { trace } = require('@opentelemetry/api');
const express = require('express');
const { rollTheDice } = require('./dice.js');

const tracer = trace.getTracer('dice-server', '0.1.0');

const PORT = parseInt(process.env.PORT || '8080');
const app = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{< /tabpane >}}

두 번째는 _라이브러리 파일_ `dice.ts`(또는 `dice.js`)이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts {hl_lines=[4]}
/*dice.ts*/
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('dice-lib');

function rollOnce(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

export function rollTheDice(rolls: number, min: number, max: number) {
  const result: number[] = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js {hl_lines=[4]}
/*dice.js*/
const { trace } = require('@opentelemetry/api');

const tracer = trace.getTracer('dice-lib');

function rollOnce(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

function rollTheDice(rolls, min, max) {
  const result = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}

module.exports = { rollTheDice };
```

{{% /tab %}} {{< /tabpane >}}

### 스팬 생성하기 {#create-spans}

이제 [트레이서](/docs/concepts/signals/traces/#tracer)가 초기화되었으니
[스팬](/docs/concepts/signals/traces/#spans)을 생성할 수 있다.

OpenTelemetry JavaScript의 API는 스팬을 생성할 수 있는 두 가지 메서드를
제공한다.

- [`tracer.startSpan`](https://open-telemetry.github.io/opentelemetry-js/interfaces/_opentelemetry_api._opentelemetry_api.Tracer.html#startspan):
  컨텍스트에 설정하지 않고 새 스팬을 시작한다.
- [`tracer.startActiveSpan`](https://open-telemetry.github.io/opentelemetry-js/interfaces/_opentelemetry_api._opentelemetry_api.Tracer.html#startactivespan):
  새 스팬을 시작하고, 생성된 스팬을 첫 번째 인수로 전달하며 주어진 콜백 함수를
  호출한다. 새 스팬은 컨텍스트에 설정되며, 이 컨텍스트는 함수 호출이 지속되는
  동안 활성화된다.

대부분의 경우 스팬과 그 컨텍스트를 활성 상태로 설정해주는 후자
(`tracer.startActiveSpan`)를 사용하는 것이 좋다.

아래 코드는 활성 스팬을 생성하는 방법을 보여준다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import { trace, type Span } from '@opentelemetry/api';

/* ... */

export function rollTheDice(rolls: number, min: number, max: number) {
  // Create a span. A span must be closed.
  return tracer.startActiveSpan('rollTheDice', (span: Span) => {
    const result: number[] = [];
    for (let i = 0; i < rolls; i++) {
      result.push(rollOnce(min, max));
    }
    // Be sure to end the span!
    span.end();
    return result;
  });
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
function rollTheDice(rolls, min, max) {
  // Create a span. A span must be closed.
  return tracer.startActiveSpan('rollTheDice', (span) => {
    const result = [];
    for (let i = 0; i < rolls; i++) {
      result.push(rollOnce(min, max));
    }
    // Be sure to end the span!
    span.end();
    return result;
  });
}
```

{{% /tab %}} {{< /tabpane >}}

지금까지 [예제 앱](#example-app)을 사용하는 안내를 따라했다면, 위 코드를
라이브러리 파일 `dice.ts`(또는 `dice.js`)에 복사할 수 있다. 이제 앱에서 방출되는
스팬을 확인할 수 있어야 한다.

다음과 같이 앱을 시작한 다음, 브라우저나 `curl`로
<http://localhost:8080/rolldice?rolls=12>에 접속해 요청을 보낸다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```sh
npx tsx --import ./instrumentation.ts app.ts
```

{{% /tab %}} {{% tab JavaScript %}}

```sh
node --import ./instrumentation.mjs app.js
```

{{% /tab %}} {{< /tabpane >}}

잠시 후 `ConsoleSpanExporter`가 콘솔에 출력하는 스팬을 다음과 비슷하게 볼 수
있다.

```js
{
  resource: {
    attributes: {
      'service.name': 'dice-server',
      'service.version': '0.1.0',
      // ...
    }
  },
  instrumentationScope: { name: 'dice-lib', version: undefined, schemaUrl: undefined },
  traceId: '30d32251088ba9d9bca67b09c43dace0',
  parentSpanContext: undefined,
  traceState: undefined,
  name: 'rollTheDice',
  id: 'cc8a67c2d4840402',
  kind: 0,
  timestamp: 1756165206470000,
  duration: 35.584,
  attributes: {},
  status: { code: 0 },
  events: [],
  links: []
}
```

### 중첩된 스팬 생성하기 {#create-nested-spans}

중첩된 [스팬](/docs/concepts/signals/traces/#spans)을 사용하면 본질적으로 중첩된
작업을 추적할 수 있다. 예를 들어 아래의 `rollOnce()` 함수는 중첩된 연산을
나타낸다. 다음 예제는 `rollOnce()`를 추적하는 중첩된 스팬을 생성한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
function rollOnce(i: number, min: number, max: number) {
  return tracer.startActiveSpan(`rollOnce:${i}`, (span: Span) => {
    const result = Math.floor(Math.random() * (max - min + 1) + min);
    span.end();
    return result;
  });
}

export function rollTheDice(rolls: number, min: number, max: number) {
  // Create a span. A span must be closed.
  return tracer.startActiveSpan('rollTheDice', (parentSpan: Span) => {
    const result: number[] = [];
    for (let i = 0; i < rolls; i++) {
      result.push(rollOnce(i, min, max));
    }
    // Be sure to end the span!
    parentSpan.end();
    return result;
  });
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
function rollOnce(i, min, max) {
  return tracer.startActiveSpan(`rollOnce:${i}`, (span) => {
    const result = Math.floor(Math.random() * (max - min + 1) + min);
    span.end();
    return result;
  });
}

function rollTheDice(rolls, min, max) {
  // Create a span. A span must be closed.
  return tracer.startActiveSpan('rollTheDice', (parentSpan) => {
    const result = [];
    for (let i = 0; i < rolls; i++) {
      result.push(rollOnce(i, min, max));
    }
    // Be sure to end the span!
    parentSpan.end();
    return result;
  });
}
```

{{% /tab %}} {{< /tabpane >}}

이 코드는 각 _굴리기_마다 `parentSpan`의 ID를 부모 ID로 갖는 자식 스팬을
생성한다.

```js
{
  traceId: '6469e115dc1562dd768c999da0509615',
  parentSpanContext: {
    traceId: '6469e115dc1562dd768c999da0509615',
    spanId: '38691692d6bc3395',
    // ...
  },
  name: 'rollOnce:0',
  id: '36423bc1ce7532b0',
  timestamp: 1756165362215000,
  duration: 85.667,
  // ...
}
{
  traceId: '6469e115dc1562dd768c999da0509615',
  parentSpanContext: {
    traceId: '6469e115dc1562dd768c999da0509615',
    spanId: '38691692d6bc3395',
    // ...
  },
  name: 'rollOnce:1',
  id: 'ed9bbba2264d6872',
  timestamp: 1756165362215000,
  duration: 16.834,
  // ...
}
{
  traceId: '6469e115dc1562dd768c999da0509615',
  parentSpanContext: undefined,
  name: 'rollTheDice',
  id: '38691692d6bc3395',
  timestamp: 1756165362214000,
  duration: 1022.209,
  // ...
}
```

### 독립적인 스팬 생성하기 {#create-independent-spans}

앞의 예제들은 활성 스팬을 생성하는 방법을 보여주었다. 경우에 따라 중첩되지 않고
서로 형제 관계인 비활성 스팬을 생성하고 싶을 수도 있다.

```js
const doWork = () => {
  const span1 = tracer.startSpan('work-1');
  // do some work
  const span2 = tracer.startSpan('work-2');
  // do some more work
  const span3 = tracer.startSpan('work-3');
  // do even more work

  span1.end();
  span2.end();
  span3.end();
};
```

이 예제에서 `span1`, `span2`, `span3`는 형제 스팬이며, 그중 어느 것도 현재 활성
스팬으로 간주되지 않는다. 이들은 서로 중첩되는 대신 같은 부모를 공유한다.

이런 구조는 함께 그룹화되어 있지만 개념적으로는 서로 독립적인 작업 단위가 있을
때 유용할 수 있다.

### 현재 스팬 얻기 {#get-the-current-span}

프로그램 실행 중 특정 시점에 현재/활성
[스팬](/docs/concepts/signals/traces/#spans)으로 무언가를 수행하는 것이 유용할
때가 있다.

```js
const activeSpan = opentelemetry.trace.getActiveSpan();

// do something with the active span, optionally ending it if that is appropriate for your use case.
```

### 컨텍스트에서 스팬 얻기 {#get-a-span-from-context}

반드시 활성 스팬이 아니더라도, 주어진 컨텍스트에서
[스팬](/docs/concepts/signals/traces/#spans)을 얻는 것이 유용할 수도 있다.

```js
const ctx = getContextFromSomewhere();
const span = opentelemetry.trace.getSpan(ctx);

// do something with the acquired span, optionally ending it if that is appropriate for your use case.
```

### 속성 {#attributes}

[속성](/docs/concepts/signals/traces/#attributes)을 사용하면
[`Span`](/docs/concepts/signals/traces/#spans)에 키/값 쌍을 첨부하여, 추적 중인
현재 작업에 대한 더 많은 정보를 담을 수 있다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
function rollOnce(i: number, min: number, max: number) {
  return tracer.startActiveSpan(`rollOnce:${i}`, (span: Span) => {
    const result = Math.floor(Math.random() * (max - min + 1) + min);

    // Add an attribute to the span
    span.setAttribute('dicelib.rolled', result.toString());

    span.end();
    return result;
  });
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
function rollOnce(i, min, max) {
  return tracer.startActiveSpan(`rollOnce:${i}`, (span) => {
    const result = Math.floor(Math.random() * (max - min + 1) + min);

    // Add an attribute to the span
    span.setAttribute('dicelib.rolled', result.toString());

    span.end();
    return result;
  });
}
```

{{% /tab %}} {{< /tabpane >}}

스팬을 생성할 때 속성을 함께 추가할 수도 있다.

```javascript
tracer.startActiveSpan(
  'app.new-span',
  { attributes: { attribute1: 'value1' } },
  (span) => {
    // do some work...

    span.end();
  },
);
```

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
function rollTheDice(rolls: number, min: number, max: number) {
  return tracer.startActiveSpan(
    'rollTheDice',
    { attributes: { 'dicelib.rolls': rolls.toString() } },
    (span: Span) => {
      /* ... */
    },
  );
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
function rollTheDice(rolls, min, max) {
  return tracer.startActiveSpan(
    'rollTheDice',
    { attributes: { 'dicelib.rolls': rolls.toString() } },
    (span) => {
      /* ... */
    },
  );
}
```

{{% /tab %}} {{< /tabpane >}}

#### 시맨틱 속성 {#semantic-attributes}

HTTP나 데이터베이스 호출처럼 잘 알려진 프로토콜에서의 작업을 나타내는 스팬에는
시맨틱 컨벤션이 존재한다. 이러한 스팬에 대한 시맨틱 컨벤션은 명세의
[트레이스 시맨틱 컨벤션](/docs/specs/semconv/general/trace/)에 정의되어 있다. 이
가이드의 간단한 예제에서는 소스 코드 속성을 사용할 수 있다.

먼저 시맨틱 컨벤션을 애플리케이션의 의존성으로 추가한다.

```shell
npm install --save @opentelemetry/semantic-conventions
```

애플리케이션 파일 상단에 다음을 추가한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import {
  ATTR_CODE_FUNCTION_NAME,
  ATTR_CODE_FILE_PATH,
} from '@opentelemetry/semantic-conventions';
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const {
  ATTR_CODE_FUNCTION_NAME,
  ATTR_CODE_FILE_PATH,
} = require('@opentelemetry/semantic-conventions');
```

{{% /tab %}} {{< /tabpane >}}

마지막으로, 시맨틱 속성을 포함하도록 파일을 업데이트할 수 있다.

```javascript
const doWork = () => {
  tracer.startActiveSpan('app.doWork', (span) => {
    span.setAttribute(ATTR_CODE_FUNCTION_NAME, 'doWork');
    span.setAttribute(ATTR_CODE_FILE_PATH, __filename);

    // Do some work...

    span.end();
  });
};
```

### 스팬 이벤트 {#span-events}

[스팬 이벤트](/docs/concepts/signals/traces/#span-events)는
[`Span`](/docs/concepts/signals/traces/#spans)에 남기는, 사람이 읽을 수 있는
메시지로, 지속 시간 없이 단일 타임스탬프로 추적할 수 있는 개별 이벤트를
나타낸다. 원시적인 형태의 로그라고 생각하면 된다.

```js
span.addEvent('Doing something');

const result = doWork();
```

추가 [속성](/docs/concepts/signals/traces/#attributes)과 함께 스팬 이벤트를
생성할 수도 있다.

```js
span.addEvent('some log', {
  'log.severity': 'error',
  'log.message': 'Data not found',
  'request.id': requestId,
});
```

### 스팬 링크 {#span-links}

[`Span`](/docs/concepts/signals/traces/#spans)은 인과적으로 관련된 다른
스팬으로의 [`Link`](/docs/concepts/signals/traces/#span-links)를 0개 이상 가지고
생성될 수 있다. 흔한 시나리오는 하나 이상의 트레이스를 현재 스팬과 연관 짓는
것이다.

```js
const someFunction = (spanToLinkFrom) => {
  const options = {
    links: [
      {
        context: spanToLinkFrom.spanContext(),
      },
    ],
  };

  tracer.startActiveSpan('app.someFunction', options, (span) => {
    // Do some work...

    span.end();
  });
};
```

### 스팬 상태 {#span-status}

{{% include "span-status-preamble.md" %}}

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry, { SpanStatusCode } from '@opentelemetry/api';

// ...

tracer.startActiveSpan('app.doWork', (span) => {
  for (let i = 0; i <= Math.floor(Math.random() * 40000000); i += 1) {
    if (i > 10000) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: 'Error',
      });
    }
  }

  span.end();
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');

// ...

tracer.startActiveSpan('app.doWork', (span) => {
  for (let i = 0; i <= Math.floor(Math.random() * 40000000); i += 1) {
    if (i > 10000) {
      span.setStatus({
        code: opentelemetry.SpanStatusCode.ERROR,
        message: 'Error',
      });
    }
  }

  span.end();
});
```

{{% /tab %}} {{< /tabpane >}}

### 예외 기록하기 {#recording-exceptions}

예외가 발생했을 때 이를 기록해두는 것이 좋다. 이는 [스팬 상태](#span-status)
설정과 함께 하는 것이 권장된다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry, { SpanStatusCode } from '@opentelemetry/api';

// ...

try {
  doWork();
} catch (ex) {
  if (ex instanceof Error) {
    span.recordException(ex);
  }
  span.setStatus({ code: SpanStatusCode.ERROR });
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');

// ...

try {
  doWork();
} catch (ex) {
  if (ex instanceof Error) {
    span.recordException(ex);
  }
  span.setStatus({ code: opentelemetry.SpanStatusCode.ERROR });
}
```

{{% /tab %}} {{< /tabpane >}}

### `sdk-trace-base`를 사용해 스팬 컨텍스트 수동으로 전파하기 {#using-sdk-trace-base-and-manually-propagating-span-context}

경우에 따라 Node.js SDK나 Web SDK를 모두 사용할 수 없을 때가 있다. 초기화 코드를
제외하면 가장 큰 차이점은, 중첩된 스팬을 생성할 수 있도록 현재 컨텍스트에서
스팬을 수동으로 활성 상태로 설정해야 한다는 점이다.

#### `sdk-trace-base`로 트레이싱 초기화하기 {#initializing-tracing-with-sdk-trace-base}

트레이싱을 초기화하는 방법은 Node.js나 Web SDK에서 하는 방식과 비슷하다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry from '@opentelemetry/api';
import {
  CompositePropagator,
  W3CTraceContextPropagator,
  W3CBaggagePropagator,
} from '@opentelemetry/core';
import {
  BasicTracerProvider,
  BatchSpanProcessor,
  ConsoleSpanExporter,
} from '@opentelemetry/sdk-trace-base';

opentelemetry.trace.setGlobalTracerProvider(
  new BasicTracerProvider({
    // Configure span processor to send spans to the exporter
    spanProcessors: [new BatchSpanProcessor(new ConsoleSpanExporter())],
  }),
);

opentelemetry.propagation.setGlobalPropagator(
  new CompositePropagator({
    propagators: [new W3CTraceContextPropagator(), new W3CBaggagePropagator()],
  }),
);

// This is what we'll access in all instrumentation code
const tracer = opentelemetry.trace.getTracer('example-basic-tracer-node');
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');
const {
  CompositePropagator,
  W3CTraceContextPropagator,
  W3CBaggagePropagator,
} = require('@opentelemetry/core');
const {
  BasicTracerProvider,
  ConsoleSpanExporter,
  BatchSpanProcessor,
} = require('@opentelemetry/sdk-trace-base');

opentelemetry.trace.setGlobalTracerProvider(
  new BasicTracerProvider({
    // Configure span processor to send spans to the exporter
    spanProcessors: [new BatchSpanProcessor(new ConsoleSpanExporter())],
  }),
);

opentelemetry.propagation.setGlobalPropagator(
  new CompositePropagator({
    propagators: [new W3CTraceContextPropagator(), new W3CBaggagePropagator()],
  }),
);

// This is what we'll access in all instrumentation code
const tracer = opentelemetry.trace.getTracer('example-basic-tracer-node');
```

{{% /tab %}} {{< /tabpane >}}

이 문서의 다른 예제들과 마찬가지로, 이 코드는 앱 전체에서 사용할 수 있는
트레이서를 내보낸다.

#### `sdk-trace-base`로 중첩된 스팬 생성하기 {#creating-nested-spans-with-sdk-trace-base}

중첩된 스팬을 생성하려면, 현재 생성된 스팬이 무엇이든 현재 컨텍스트에서 활성
스팬으로 설정해야 한다. `startActiveSpan`은 이를 대신 해주지 않으므로 사용할
필요가 없다.

```javascript
const mainWork = () => {
  const parentSpan = tracer.startSpan('main');

  for (let i = 0; i < 3; i += 1) {
    doWork(parentSpan, i);
  }

  // Be sure to end the parent span!
  parentSpan.end();
};

const doWork = (parent, i) => {
  // To create a child span, we need to mark the current (parent) span as the active span
  // in the context, then use the resulting context to create a child span.
  const ctx = opentelemetry.trace.setSpan(
    opentelemetry.context.active(),
    parent,
  );
  const span = tracer.startSpan(`doWork:${i}`, undefined, ctx);

  // simulate some random work.
  for (let i = 0; i <= Math.floor(Math.random() * 40000000); i += 1) {
    // empty
  }

  // Make sure to end this child span! If you don't,
  // it will continue to track work beyond 'doWork'!
  span.end();
};
```

`sdk-trace-base`를 사용할 때도 다른 모든 API는 Node.js나 Web SDK를 사용할 때와
동일하게 동작한다.

## 메트릭 {#metrics}

[메트릭](/docs/concepts/signals/metrics)은 개별 측정값을 집계로 결합하여, 시스템
부하에 대한 함수로서 일정한 데이터를 만들어낸다. 집계는 낮은 수준의 문제를
진단하는 데 필요한 세부 정보는 부족하지만, 추세를 파악하고 애플리케이션 런타임
텔레메트리를 제공함으로써 스팬을 보완한다.

### 메트릭 초기화 {#initialize-metrics}

> [!NB] 라이브러리를 계측하는 경우 **이 단계를 건너뛴다**.

앱에서 [메트릭](/docs/concepts/signals/metrics/)을 활성화하려면,
[`Meter`](/docs/concepts/signals/metrics/#meter)를 생성할 수 있게 해주는
초기화된 [`MeterProvider`](/docs/concepts/signals/metrics/#meter-provider)가
필요하다.

`MeterProvider`가 생성되지 않으면, 메트릭을 위한 오픈텔레메트리 API는 no-op
구현체를 사용하여 데이터를 생성하지 못한다. 다음에서 설명하듯이, Node와
브라우저에서 모든 SDK 초기화 코드를 포함하도록 `instrumentation.ts`(또는
`instrumentation.js`) 파일을 수정한다.

#### Node.js {#initialize-metrics-nodejs}

위에서 [SDK 초기화](#initialize-the-sdk) 안내를 따라했다면, 이미
`MeterProvider`가 설정되어 있다. 이제 [미터 얻기](#acquiring-a-meter)로 넘어가면
된다.

##### `sdk-metrics`로 메트릭 초기화하기 {#initializing-metrics-with-sdk-metrics}

경우에 따라
[Node.js용 오픈텔레메트리 SDK 전체](https://www.npmjs.com/package/@opentelemetry/sdk-node)를
사용할 수 없거나 사용하고 싶지 않을 수 있다. 브라우저에서 OpenTelemetry
JavaScript를 사용하고 싶은 경우에도 마찬가지다.

이런 경우 `@opentelemetry/sdk-metrics` 패키지로 메트릭을 초기화할 수 있다.

```shell
npm install @opentelemetry/sdk-metrics
```

트레이싱을 위해 아직 만들지 않았다면, 모든 SDK 초기화 코드를 담은 별도의
`instrumentation.ts`(또는 `instrumentation.js`) 파일을 생성한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry from '@opentelemetry/api';
import {
  ConsoleMetricExporter,
  MeterProvider,
  PeriodicExportingMetricReader,
} from '@opentelemetry/sdk-metrics';
import {
  defaultResource,
  resourceFromAttributes,
} from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions';

const resource = defaultResource().merge(
  resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'dice-server',
    [ATTR_SERVICE_VERSION]: '0.1.0',
  }),
);

const metricReader = new PeriodicExportingMetricReader({
  exporter: new ConsoleMetricExporter(),
  // Default is 60000ms (60 seconds). Set to 10 seconds for demonstrative purposes only.
  exportIntervalMillis: 10000,
});

const myServiceMeterProvider = new MeterProvider({
  resource: resource,
  readers: [metricReader],
});

// Set this MeterProvider to be global to the app being instrumented.
opentelemetry.metrics.setGlobalMeterProvider(myServiceMeterProvider);
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');
const {
  MeterProvider,
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} = require('@opentelemetry/sdk-metrics');
const {
  defaultResource,
  resourceFromAttributes,
} = require('@opentelemetry/resources');
const {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} = require('@opentelemetry/semantic-conventions');

const resource = defaultResource().merge(
  resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'service-name-here',
    [ATTR_SERVICE_VERSION]: '0.1.0',
  }),
);

const metricReader = new PeriodicExportingMetricReader({
  exporter: new ConsoleMetricExporter(),

  // Default is 60000ms (60 seconds). Set to 10 seconds for demonstrative purposes only.
  exportIntervalMillis: 10000,
});

const myServiceMeterProvider = new MeterProvider({
  resource: resource,
  readers: [metricReader],
});

// Set this MeterProvider to be global to the app being instrumented.
opentelemetry.metrics.setGlobalMeterProvider(myServiceMeterProvider);
```

{{% /tab %}} {{< /tabpane >}}

앱을 실행할 때 다음과 같이 이 파일을 `--import`해야 한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```sh
npx tsx --import ./instrumentation.ts app.ts
```

{{% /tab %}} {{% tab JavaScript %}}

```sh
node --import ./instrumentation.mjs app.js
```

{{% /tab %}} {{< /tabpane >}}

이제 `MeterProvider`가 구성되었으니 `Meter`를 얻을 수 있다.

### 미터 얻기 {#acquiring-a-meter}

애플리케이션에서 수동으로 계측한 코드가 있는 곳이라면 어디서든 `getMeter`를
호출해 미터를 얻을 수 있다. 예를 들면 다음과 같다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import opentelemetry from '@opentelemetry/api';

const myMeter = opentelemetry.metrics.getMeter(
  'instrumentation-scope-name',
  'instrumentation-scope-version',
);

// You can now use a 'meter' to create instruments!
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/api');

const myMeter = opentelemetry.metrics.getMeter(
  'instrumentation-scope-name',
  'instrumentation-scope-version',
);

// You can now use a 'meter' to create instruments!
```

{{% /tab %}} {{< /tabpane >}}

`instrumentation-scope-name`과 `instrumentation-scope-version`의 값은 패키지,
모듈, 클래스 이름처럼 [계측 스코프](/docs/concepts/instrumentation-scope/)를
고유하게 식별할 수 있어야 한다. 이름은 필수이지만, 버전은 선택 사항이라도
지정하는 것이 권장된다.

앱의 나머지 부분으로 미터 인스턴스를 내보내기보다는, 필요할 때 앱에서 직접
`getMeter`를 호출하는 것이 일반적으로 권장된다. 이렇게 하면 다른 필수 의존성이
관련될 때 발생할 수 있는 까다로운 애플리케이션 로드 문제를 피할 수 있다.

[예제 앱](#example-app)의 경우, 적절한 계측 스코프로 미터를 얻을 수 있는 곳이 두
군데 있다.

첫 번째는 _애플리케이션 파일_ `app.ts`(또는 `app.js`)이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*app.ts*/
import { metrics, trace } from '@opentelemetry/api';
import express, { type Express } from 'express';
import { rollTheDice } from './dice';

const tracer = trace.getTracer('dice-server', '0.1.0');
const meter = metrics.getMeter('dice-server', '0.1.0');

const PORT: number = parseInt(process.env.PORT || '8080');
const app: Express = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*app.js*/
const { trace, metrics } = require('@opentelemetry/api');
const express = require('express');
const { rollTheDice } = require('./dice.js');

const tracer = trace.getTracer('dice-server', '0.1.0');
const meter = metrics.getMeter('dice-server', '0.1.0');

const PORT = parseInt(process.env.PORT || '8080');
const app = express();

app.get('/rolldice', (req, res) => {
  const rolls = req.query.rolls ? parseInt(req.query.rolls.toString()) : NaN;
  if (isNaN(rolls)) {
    res
      .status(400)
      .send("Request parameter 'rolls' is missing or not a number.");
    return;
  }
  res.send(JSON.stringify(rollTheDice(rolls, 1, 6)));
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{< /tabpane >}}

두 번째는 _라이브러리 파일_ `dice.ts`(또는 `dice.js`)이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*dice.ts*/
import { trace, metrics } from '@opentelemetry/api';

const tracer = trace.getTracer('dice-lib');
const meter = metrics.getMeter('dice-lib');

function rollOnce(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

export function rollTheDice(rolls: number, min: number, max: number) {
  const result: number[] = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*dice.js*/
const { trace, metrics } = require('@opentelemetry/api');

const tracer = trace.getTracer('dice-lib');
const meter = metrics.getMeter('dice-lib');

function rollOnce(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

function rollTheDice(rolls, min, max) {
  const result = [];
  for (let i = 0; i < rolls; i++) {
    result.push(rollOnce(min, max));
  }
  return result;
}

module.exports = { rollTheDice };
```

{{% /tab %}} {{< /tabpane >}}

이제 [미터](/docs/concepts/signals/metrics/#meter)가 초기화되었으니
[메트릭 계측기](/docs/concepts/signals/metrics/#metric-instruments)를 생성할 수
있다.

### 카운터 사용하기 {#using-counters}

카운터는 음수가 아니며 증가하는 값을 측정하는 데 사용할 수 있다.

[예제 앱](#example-app)의 경우, 이를 사용해 주사위를 굴린 횟수를 셀 수 있다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*dice.ts*/
const counter = meter.createCounter('dice-lib.rolls.counter');

function rollOnce(min: number, max: number) {
  counter.add(1);
  return Math.floor(Math.random() * (max - min + 1) + min);
}
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*dice.js*/
const counter = meter.createCounter('dice-lib.rolls.counter');

function rollOnce(min, max) {
  counter.add(1);
  return Math.floor(Math.random() * (max - min + 1) + min);
}
```

{{% /tab %}} {{< /tabpane >}}

### UpDown 카운터 사용하기 {#using-updown-counters}

UpDown 카운터는 증가와 감소가 모두 가능하여, 오르내리는 누적 값을 관측할 수 있게
해준다.

```js
const counter = myMeter.createUpDownCounter('events.counter');

//...

counter.add(1);

//...

counter.add(-1);
```

### 히스토그램 사용하기 {#using-histograms}

히스토그램은 시간에 따른 값의 분포를 측정하는 데 사용된다.

예를 들어, Express로 만든 API 라우트의 응답 시간 분포를 다음과 같이 보고할 수
있다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import express from 'express';

const app = express();

app.get('/', (_req, _res) => {
  const histogram = myMeter.createHistogram('task.duration');
  const startTime = new Date().getTime();

  // do some work in an API call

  const endTime = new Date().getTime();
  const executionTime = endTime - startTime;

  // Record the duration of the task operation
  histogram.record(executionTime);
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const express = require('express');

const app = express();

app.get('/', (_req, _res) => {
  const histogram = myMeter.createHistogram('task.duration');
  const startTime = new Date().getTime();

  // do some work in an API call

  const endTime = new Date().getTime();
  const executionTime = endTime - startTime;

  // Record the duration of the task operation
  histogram.record(executionTime);
});
```

{{% /tab %}} {{< /tabpane >}}

### 관측 가능한(비동기) 카운터 사용하기 {#using-observable-async-counters}

관측 가능한 카운터는 가산적이고, 음수가 아니며, 단조 증가하는 값을 측정하는 데
사용할 수 있다.

```js
const events = [];

const addEvent = (name) => {
  events.push(name);
};

const counter = myMeter.createObservableCounter('events.counter');

counter.addCallback((result) => {
  result.observe(events.length);
});

//... calls to addEvent
```

### 관측 가능한(비동기) UpDown 카운터 사용하기 {#using-observable-async-updown-counters}

관측 가능한 UpDown 카운터는 증가와 감소가 모두 가능하여, 가산적이고 음수가
아니며 비단조적으로 증가하는 누적 값을 측정할 수 있게 해준다.

```js
const events = [];

const addEvent = (name) => {
  events.push(name);
};

const removeEvent = () => {
  events.pop();
};

const counter = myMeter.createObservableUpDownCounter('events.counter');

counter.addCallback((result) => {
  result.observe(events.length);
});

//... calls to addEvent and removeEvent
```

### 관측 가능한(비동기) 게이지 사용하기 {#using-observable-async-gauges}

관측 가능한 게이지는 비가산적인 값을 측정하는 데 사용해야 한다.

```js
let temperature = 32;

const gauge = myMeter.createObservableGauge('temperature.gauge');

gauge.addCallback((result) => {
  result.observe(temperature);
});

//... temperature variable is modified by a sensor
```

### 계측기 설명하기 {#describing-instruments}

카운터, 히스토그램 등의 계측기를 생성할 때 설명을 부여할 수 있다.

```js
const httpServerResponseDuration = myMeter.createHistogram(
  'http.server.duration',
  {
    description: 'A distribution of the HTTP server response times',
    unit: 'milliseconds',
    valueType: ValueType.INT,
  },
);
```

JavaScript에서 각 설정 값은 다음을 의미한다.

- `description` - 계측기에 대한 사람이 읽을 수 있는 설명
- `unit` - 값이 나타내고자 하는 측정 단위에 대한 설명이다. 예를 들어 지속 시간을
  측정할 때는 `milliseconds`를, 바이트 수를 셀 때는 `bytes`를 사용한다.
- `valueType` - 측정에 사용되는 숫자 값의 종류이다.

생성하는 각 계측기마다 설명을 붙이는 것이 일반적으로 권장된다.

### 속성 추가하기 {#adding-attributes}

메트릭이 생성될 때 속성을 추가할 수 있다.

```js
const counter = myMeter.createCounter('my.counter');

counter.add(1, { 'some.optional.attribute': 'some value' });
```

### 메트릭 뷰 구성하기 {#configure-metric-views}

메트릭 뷰(View)는 개발자가 메트릭 SDK에서 노출하는 메트릭을 커스터마이즈할 수
있게 해준다.

#### 셀렉터 {#selectors}

뷰를 인스턴스화하려면 먼저 대상 계측기를 선택해야 한다. 메트릭에 사용할 수 있는
셀렉터는 다음과 같다.

- `instrumentType`
- `instrumentName`
- `meterName`
- `meterVersion`
- `meterSchemaUrl`

`instrumentName`(문자열 타입)으로 선택할 때는 와일드카드를 지원하므로, `*`를
사용해 모든 계측기를 선택하거나 `http*`를 사용해 이름이 `http`로 시작하는 모든
계측기를 선택할 수 있다.

#### 예제 {#examples}

모든 메트릭 유형에서 속성을 필터링하는 예이다.

```js
const limitAttributesView = {
  // only export the attribute 'environment'
  attributeKeys: ['environment'],
  // apply the view to all instruments
  instrumentName: '*',
};
```

미터 이름이 `pubsub`인 모든 계측기를 제외하는 예이다.

```js
const dropView = {
  aggregation: { type: AggregationType.DROP },
  meterName: 'pubsub',
};
```

`http.server.duration`이라는 이름의 히스토그램에 대해 명시적인 버킷 크기를
정의하는 예이다.

```js
const histogramView = {
  aggregation: {
    type: AggregationType.EXPLICIT_BUCKET_HISTOGRAM,
    options: { boundaries: [0, 1, 5, 10, 15, 20, 25, 30] },
  },
  instrumentName: 'http.server.duration',
  instrumentType: InstrumentType.HISTOGRAM,
};
```

#### 미터 프로바이더에 연결하기 {#attach-to-meter-provider}

뷰를 구성했다면, 해당 미터 프로바이더에 연결한다.

```js
const meterProvider = new MeterProvider({
  views: [limitAttributesView, dropView, histogramView],
});
```

## 로그 {#logs}

로그 API 및 SDK는 현재 개발 중이다.

## 다음 단계 {#next-steps}

또한 하나 이상의 텔레메트리 백엔드로
[텔레메트리 데이터를 내보내](/docs/languages/js/exporters)도록 적절한 익스포터를
구성해야 한다.
