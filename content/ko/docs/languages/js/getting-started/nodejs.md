---
title: Node.js
description: 5분 이내에 애플리케이션의 텔레메트리를 확보한다!
aliases: [/docs/js/getting_started/nodejs]
weight: 10
cSpell:ignore: autoinstrumentations rolldice
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

이 페이지는 Node.js에서 오픈텔레메트리(OpenTelemetry)를 시작하는 방법을
보여준다.

[트레이스][traces]와 [메트릭][metrics]를 모두 계측하고 이를 콘솔에 출력하는
방법을 배운다.

> [!NOTE]
>
> Node.js용 오픈텔레메트리 로깅 라이브러리는 아직 개발 중이므로, 아래에는 이에
> 대한 예제를 제공하지 않는다. 상태에 대한 자세한 내용은
> [상태 및 릴리스](/docs/languages/js/#status-and-releases)를 참고한다.

## 사전 요구 사항 {#prerequisites}

다음이 로컬에 설치되어 있는지 확인한다.

- [Node.js](https://nodejs.org/en/download/)
- TypeScript를 사용할 예정이라면
  [TypeScript](https://www.typescriptlang.org/download)

## 예제 애플리케이션 {#example-application}

다음 예제는 기본적인 [Express](https://expressjs.com/) 애플리케이션을 사용한다.
Express를 사용하지 않아도 괜찮다. OpenTelemetry JavaScript는 Koa나 Nest.JS 같은
다른 웹 프레임워크에서도 사용할 수 있다. 지원되는 프레임워크의 전체 라이브러리
목록은
[레지스트리](/ecosystem/registry/?component=instrumentation&language=js)를
참고한다.

더 정교한 예제는 [예제](/docs/languages/js/examples/)를 참고한다.

### 의존성 {#dependencies}

시작하려면, 새 디렉터리에 빈 `package.json`을 설정한다.

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

`app.ts`(TypeScript를 사용하지 않는다면 `app.js`)라는 이름의 파일을 생성하고
다음 코드를 추가한다.

{{% tabpane text=true %}} {{% tab TypeScript %}}

```ts
/*app.ts*/
import express, { Express } from 'express';

const PORT: number = parseInt(process.env.PORT || '8080');
const app: Express = express();

function getRandomNumber(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

app.get('/rolldice', (req, res) => {
  res.send(getRandomNumber(1, 6).toString());
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*app.js*/
const express = require('express');

const PORT = parseInt(process.env.PORT || '8080');
const app = express();

function getRandomNumber(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

app.get('/rolldice', (req, res) => {
  res.send(getRandomNumber(1, 6).toString());
});

app.listen(PORT, () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```

{{% /tab %}} {{% /tabpane %}}

다음 명령어로 애플리케이션을 실행하고, 웹 브라우저에서
<http://localhost:8080/rolldice>를 열어 정상적으로 동작하는지 확인한다.

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

## 계측 {#instrumentation}

다음은 오픈텔레메트리로 계측한 애플리케이션을 설치하고 초기화하고 실행하는
방법을 보여준다.

### 추가 의존성 {#more-dependencies}

먼저 Node SDK와 autoinstrumentations 패키지를 설치한다.

Node SDK를 사용하면 대부분의 사용 사례에 적합한 여러 구성 기본값으로
오픈텔레메트리를 초기화할 수 있다.

`auto-instrumentations-node` 패키지는 라이브러리에서 호출되는 코드에 대응하는
스팬을 자동으로 생성하는 계측 라이브러리를 설치한다. 이 경우 Express에 대한
계측을 제공하여, 예제 앱이 들어오는 각 요청에 대해 스팬을 자동으로 생성하도록
해준다.

```shell
npm install @opentelemetry/sdk-node \
  @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/sdk-metrics \
  @opentelemetry/sdk-trace-node
```

모든 자동 계측 모듈을 찾으려면
[레지스트리](/ecosystem/registry/?language=js&component=instrumentation)를
참고한다.

### 설정 {#setup}

계측 설정과 구성은 애플리케이션 코드보다 _먼저_ 실행되어야 한다. 이 작업에 흔히
사용되는 도구 중 하나가
[--import](https://nodejs.org/api/cli.html#--importmodule) 플래그이다.

계측 설정 코드를 담을 `instrumentation.ts`(TypeScript를 사용하지 않는다면
`instrumentation.mjs`)라는 이름의 파일을 생성한다.

> [!NOTE]
>
> `--import instrumentation.ts`(TypeScript)를 사용하는 다음 예제는 Node.js v.20
> 이상이 필요하다. Node.js v.18을 사용한다면 JavaScript 예제를 사용한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*instrumentation.ts*/
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter } from '@opentelemetry/sdk-trace-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import {
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} from '@opentelemetry/sdk-metrics';

const sdk = new NodeSDK({
  traceExporter: new ConsoleSpanExporter(),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new ConsoleMetricExporter(),
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*instrumentation.mjs*/
import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter } from '@opentelemetry/sdk-trace-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import {
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} from '@opentelemetry/sdk-metrics';

const sdk = new NodeSDK({
  traceExporter: new ConsoleSpanExporter(),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new ConsoleMetricExporter(),
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

{{% /tab %}} {{< /tabpane >}}

## 계측된 애플리케이션 실행하기 {#run-the-instrumented-app}

이제 평소처럼 애플리케이션을 실행할 수 있지만, 애플리케이션 코드보다 먼저 계측을
로드하도록 `--import` 플래그를 사용할 수 있다. `NODE_OPTIONS` 환경 변수에
`--require @opentelemetry/auto-instrumentations-node/register`처럼 충돌하는 다른
`--import`나 `--require` 플래그가 없는지 확인한다.

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

(참고: 애플리케이션이 ECMAScript 모듈(ESM) 형태의 JavaScript로 작성되었거나
TypeScript에서 ESM으로 컴파일된 경우, 계측을 제대로 지원하려면 로더 훅이
필요하다.
`node --experimental-loader=@opentelemetry/instrumentation/hook.mjs --require ./instrumentation.js app.js`를
사용한다. 오픈텔레메트리의 ESM 지원에 대한 자세한 내용은
[ESM 지원 문서](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/esm-support.md)를
참고한다.)

웹 브라우저에서 <http://localhost:8080/rolldice>를 열고 페이지를 몇 번
새로고침한다. 잠시 후 `ConsoleSpanExporter`가 콘솔에 출력하는 스팬을 볼 수
있어야 한다.

<details>
<summary>출력 예시 보기</summary>

```js
{
  resource: {
    attributes: {
      'host.arch': 'arm64',
      'host.id': '8FEBBC33-D6DA-57FC-8EF0-1A9C14B919F8',
      'process.pid': 12460,
      // ... some resource attributes elided ...
      'process.runtime.version': '22.17.1',
      'process.runtime.name': 'nodejs',
      'process.runtime.description': 'Node.js',
      'telemetry.sdk.language': 'nodejs',
      'telemetry.sdk.name': 'opentelemetry',
      'telemetry.sdk.version': '2.0.1'
    }
  },
  instrumentationScope: {
    name: '@opentelemetry/instrumentation-express',
    version: '0.52.0',
    schemaUrl: undefined
  },
  traceId: '61e8960c349ca2a3a51289e050fd3b82',
  parentSpanContext: {
    traceId: '61e8960c349ca2a3a51289e050fd3b82',
    spanId: '631b666604f933bc',
    traceFlags: 1,
    traceState: undefined
  },
  traceState: undefined,
  name: 'request handler - /rolldice',
  id: 'd8fcc05ac4f60c99',
  kind: 0,
  timestamp: 1755719307779000,
  duration: 2801.5,
  attributes: {
    'http.route': '/rolldice',
    'express.name': '/rolldice',
    'express.type': 'request_handler'
  },
  status: { code: 0 },
  events: [],
  links: []
}
{
  resource: {
    attributes: {
      'host.arch': 'arm64',
      'host.id': '8FEBBC33-D6DA-57FC-8EF0-1A9C14B919F8',
      'process.pid': 12460,
      // ... some resource attributes elided ...
      'process.runtime.version': '22.17.1',
      'process.runtime.name': 'nodejs',
      'process.runtime.description': 'Node.js',
      'telemetry.sdk.language': 'nodejs',
      'telemetry.sdk.name': 'opentelemetry',
      'telemetry.sdk.version': '2.0.1'
    }
  },
  instrumentationScope: {
    name: '@opentelemetry/instrumentation-http',
    version: '0.203.0',
    schemaUrl: undefined
  },
  traceId: '61e8960c349ca2a3a51289e050fd3b82',
  parentSpanContext: undefined,
  traceState: undefined,
  name: 'GET /rolldice',
  id: '631b666604f933bc',
  kind: 1,
  timestamp: 1755719307777000,
  duration: 4705.75,
  attributes: {
    'http.url': 'http://localhost:8080/rolldice',
    'http.host': 'localhost:8080',
    'net.host.name': 'localhost',
    'http.method': 'GET',
    'http.scheme': 'http',
    'http.target': '/rolldice',
    'http.user_agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:141.0) Gecko/20100101 Firefox/141.0',
    'http.flavor': '1.1',
    'net.transport': 'ip_tcp',
    'net.host.ip': '::ffff:127.0.0.1',
    'net.host.port': 8080,
    'net.peer.ip': '::ffff:127.0.0.1',
    'net.peer.port': 63067,
    'http.status_code': 200,
    'http.status_text': 'OK',
    'http.route': '/rolldice'
  },
  status: { code: 0 },
  events: [],
  links: []
}
```

</details>

생성된 스팬은 `/rolldice` 라우트에 대한 요청의 생명 주기를 추적한다.

엔드포인트에 요청을 몇 번 더 보낸다. 잠시 후 다음과 같은 메트릭이 콘솔 출력에
나타난다.

<details>
<summary>출력 예시 보기</summary>

```yaml
{
  descriptor: {
    name: 'http.server.duration',
    type: 'HISTOGRAM',
    description: 'Measures the duration of inbound HTTP requests.',
    unit: 'ms',
    valueType: 1,
    advice: {}
  },
  dataPointType: 0,
  dataPoints: [
    {
      attributes: {
        'http.scheme': 'http',
        'http.method': 'GET',
        'net.host.name': 'localhost',
        'http.flavor': '1.1',
        'http.status_code': 200,
        'net.host.port': 8080,
        'http.route': '/rolldice'
      },
      startTime: [ 1755719307, 782000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: {
        min: 1.439792,
        max: 5.775,
        sum: 15.370167,
        buckets: {
          boundaries: [
               0,    5,    10,   25,
              50,   75,   100,  250,
             500,  750,  1000, 2500,
            5000, 7500, 10000
          ],
          counts: [
            0, 5, 1, 0, 0, 0,
            0, 0, 0, 0, 0, 0,
            0, 0, 0, 0
          ]
        },
        count: 6
      }
    },
    {
      attributes: {
        'http.scheme': 'http',
        'http.method': 'GET',
        'net.host.name': 'localhost',
        'http.flavor': '1.1',
        'http.status_code': 304,
        'net.host.port': 8080,
        'http.route': '/rolldice'
      },
      startTime: [ 1755719433, 609000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: {
        min: 1.39575,
        max: 1.39575,
        sum: 1.39575,
        buckets: {
          boundaries: [
               0,    5,    10,   25,
              50,   75,   100,  250,
             500,  750,  1000, 2500,
            5000, 7500, 10000
          ],
          counts: [
            0, 1, 0, 0, 0, 0,
            0, 0, 0, 0, 0, 0,
            0, 0, 0, 0
          ]
        },
        count: 1
      }
    }
  ]
}
{
  descriptor: {
    name: 'nodejs.eventloop.utilization',
    type: 'OBSERVABLE_GAUGE',
    description: 'Event loop utilization',
    unit: '1',
    valueType: 1,
    advice: {}
  },
  dataPointType: 2,
  dataPoints: [
    {
      attributes: {},
      startTime: [ 1755719362, 939000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: 0.00843049454565211
    }
  ]
}
{
  descriptor: {
    name: 'v8js.gc.duration',
    type: 'HISTOGRAM',
    description: 'Garbage collection duration by kind, one of major, minor, incremental or weakcb.',
    unit: 's',
    valueType: 1,
    advice: { explicitBucketBoundaries: [ 0.01, 0.1, 1, 10 ] }
  },
  dataPointType: 0,
  dataPoints: [
    {
      attributes: { 'v8js.gc.type': 'minor' },
      startTime: [ 1755719303, 5000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: {
        min: 0.0005120840072631835,
        max: 0.0022552499771118163,
        sum: 0.006526499509811401,
        buckets: { boundaries: [ 0.01, 0.1, 1, 10 ], counts: [ 6, 0, 0, 0, 0 ] },
        count: 6
      }
    },
    {
      attributes: { 'v8js.gc.type': 'incremental' },
      startTime: [ 1755719310, 812000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: {
        min: 0.0003403329849243164,
        max: 0.0012867081165313721,
        sum: 0.0016270411014556885,
        buckets: { boundaries: [ 0.01, 0.1, 1, 10 ], counts: [ 2, 0, 0, 0, 0 ] },
        count: 2
      }
    },
    {
      attributes: { 'v8js.gc.type': 'major' },
      startTime: [ 1755719310, 830000000 ],
      endTime: [ 1755719482, 940000000 ],
      value: {
        min: 0.0025888750553131105,
        max: 0.005744750022888183,
        sum: 0.008333625078201293,
        buckets: { boundaries: [ 0.01, 0.1, 1, 10 ], counts: [ 2, 0, 0, 0, 0 ] },
        count: 2
      }
    }
  ]
}
```

</details>

## 다음 단계 {#next-steps}

자동으로 생성된 계측을 자신의 코드베이스에 대한
[수동 계측](/docs/languages/js/instrumentation)으로 보강한다. 이렇게 하면
맞춤화된 옵저버빌리티 데이터를 얻을 수 있다.

또한 하나 이상의 텔레메트리 백엔드로
[텔레메트리 데이터를 내보내](/docs/languages/js/exporters)도록 적절한 익스포터를
구성해야 한다.

더 복잡한 예제를 살펴보고 싶다면, JavaScript 기반의
[결제 서비스](/docs/demo/services/payment/)와 TypeScript 기반의
[프론트엔드 서비스](/docs/demo/services/frontend/)를 포함하는
[오픈텔레메트리 데모](/docs/demo/)를 확인해본다.

## 문제 해결 {#troubleshooting}

무언가 잘못되었는가? 오픈텔레메트리가 올바르게 초기화되었는지 확인하기 위해 진단
로깅을 활성화할 수 있다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*instrumentation.ts*/
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api';

// For troubleshooting, set the log level to DiagLogLevel.DEBUG
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.INFO);

// const sdk = new NodeSDK({...
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*instrumentation.mjs*/
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api';

// For troubleshooting, set the log level to DiagLogLevel.DEBUG
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.INFO);

// const sdk = new NodeSDK({...
```

{{% /tab %}} {{< /tabpane >}}

{{% include esm-support-note.md %}}

[traces]: /docs/concepts/signals/traces/
[metrics]: /docs/concepts/signals/metrics/
