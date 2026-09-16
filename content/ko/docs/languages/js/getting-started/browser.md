---
title: 브라우저
aliases: [/docs/js/getting_started/browser]
description:
  브라우저 앱에 오픈텔레메트리(OpenTelemetry)를 추가하는 방법을 알아본다
weight: 20
default_lang_commit: 4c9af5912f276b79a489a10b44c53f720c7927d7
---

{{% include browser-instrumentation-warning.md %}}

이 가이드는 아래 제시된 예제 애플리케이션을 사용하지만, 자신의 애플리케이션을
계측하는 단계도 이와 비슷할 것이다.

## 사전 요구 사항 {#prerequisites}

다음이 로컬에 설치되어 있는지 확인한다.

- [Node.js](https://nodejs.org/en/download/)
- TypeScript를 사용할 예정이라면
  [TypeScript](https://www.typescriptlang.org/download)

## 예제 애플리케이션 {#example-application}

이는 매우 간단한 가이드이며, 더 복잡한 예제를 보고 싶다면
[examples/opentelemetry-web](https://github.com/open-telemetry/opentelemetry-js/tree/main/examples/opentelemetry-web)로
이동한다.

다음 파일을 빈 디렉터리에 복사하고 `index.html`이라는 이름을 붙인다.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Document Load Instrumentation Example</title>
    <base href="/" />
    <!--
      https://www.w3.org/TR/trace-context/
      Set the `traceparent` in the server's HTML template code. It should be
      dynamically generated server side to have the server's request trace ID,
      a parent span ID that was set on the server's request span, and the trace
      flags to indicate the server's sampling decision
      (01 = sampled, 00 = not sampled).
      '{version}-{traceId}-{spanId}-{sampleDecision}'
    -->
    <meta
      name="traceparent"
      content="00-ab42124a3c573678d4d8b21ba52df3bf-d21f7bc17caa5aba-01"
    />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body>
    Example of using Web Tracer with document load instrumentation with console
    exporter and collector exporter
  </body>
</html>
```

### 설치 {#installation}

브라우저에서 트레이스를 생성하려면 `@opentelemetry/sdk-trace-web`과 계측
라이브러리인 `@opentelemetry/instrumentation-document-load`가 필요하다.

```shell
npm init -y
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/instrumentation-document-load \
  @opentelemetry/context-zone
```

### 초기화 및 구성 {#initialization-and-configuration}

TypeScript로 코딩하고 있다면, 다음 명령을 실행한다.

```shell
tsc --init
```

그런 다음 [parcel](https://parceljs.org/)을 준비한다. 이는 (다른 기능들과 함께)
TypeScript로 작업할 수 있게 해준다.

```shell
npm install --save-dev parcel
```

작성하려는 앱의 언어에 따라 `.ts` 또는 `.js` 확장자를 가진 `document-load`라는
빈 코드 파일을 만든다. HTML의 `</body>` 닫는 태그 바로 앞에 다음 코드를
추가한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```html
<script type="module" src="document-load.ts"></script>
```

{{% /tab %}} {{% tab JavaScript %}}

```html
<script type="module" src="document-load.js"></script>
```

{{% /tab %}} {{< /tabpane >}}

문서 로드 타이밍을 트레이싱하고 이를 오픈텔레메트리 스팬으로 출력하는 코드를
추가할 것이다.

### 트레이서 프로바이더 생성하기 {#creating-a-tracer-provider}

문서 로드를 트레이싱하는 계측을 도입하는 트레이서 프로바이더를 생성하기 위해
`document-load.ts|js`에 다음 코드를 추가한다.

```js
/* document-load.ts|js file - the code snippet is the same for both the languages */
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { DocumentLoadInstrumentation } from '@opentelemetry/instrumentation-document-load';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { registerInstrumentations } from '@opentelemetry/instrumentation';

const provider = new WebTracerProvider();

provider.register({
  // Changing default contextManager to use ZoneContextManager - supports asynchronous operations - optional
  contextManager: new ZoneContextManager(),
});

// Registering instrumentations
registerInstrumentations({
  instrumentations: [new DocumentLoadInstrumentation()],
});
```

이제 parcel로 앱을 빌드한다.

```shell
npx parcel index.html
```

개발 웹 서버(예: `http://localhost:1234`)를 열어서 코드가 동작하는지 확인한다.

아직 익스포터를 추가하지 않았으므로 트레이스 출력은 나타나지 않는다.

### 익스포터 생성하기 {#creating-an-exporter}

다음 예시에서는 모든 스팬을 콘솔에 출력하는 `ConsoleSpanExporter`를 사용한다.

트레이스를 시각화하고 분석하려면 트레이싱 백엔드로 내보내야 한다. 백엔드와
익스포터를 설정하려면 [이 안내](../../exporters)를 따른다.

리소스를 더 효율적으로 사용하기 위해 스팬을 배치로 내보내는
`BatchSpanProcessor`를 사용하고 싶을 수도 있다.

트레이스를 콘솔로 내보내려면, `document-load.ts|js`를 다음 코드 스니펫과
일치하도록 수정한다.

```js
/* document-load.ts|js file - the code is the same for both the languages */
import {
  ConsoleSpanExporter,
  SimpleSpanProcessor,
} from '@opentelemetry/sdk-trace-base';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { DocumentLoadInstrumentation } from '@opentelemetry/instrumentation-document-load';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { registerInstrumentations } from '@opentelemetry/instrumentation';

const provider = new WebTracerProvider({
  spanProcessors: [new SimpleSpanProcessor(new ConsoleSpanExporter())],
});

provider.register({
  // Changing default contextManager to use ZoneContextManager - supports asynchronous operations - optional
  contextManager: new ZoneContextManager(),
});

// Registering instrumentations
registerInstrumentations({
  instrumentations: [new DocumentLoadInstrumentation()],
});
```

이제 애플리케이션을 다시 빌드하고 브라우저를 다시 연다. 개발자 도구바의 콘솔에서
트레이스가 내보내지는 것을 확인할 수 있어야 한다.

```json
{
  "traceId": "ab42124a3c573678d4d8b21ba52df3bf",
  "parentId": "cfb565047957cb0d",
  "name": "documentFetch",
  "id": "5123fc802ffb5255",
  "kind": 0,
  "timestamp": 1606814247811266,
  "duration": 9390,
  "attributes": {
    "component": "document-load",
    "http.response_content_length": 905
  },
  "status": {
    "code": 0
  },
  "events": [
    {
      "name": "fetchStart",
      "time": [1606814247, 811266158]
    },
    {
      "name": "domainLookupStart",
      "time": [1606814247, 811266158]
    },
    {
      "name": "domainLookupEnd",
      "time": [1606814247, 811266158]
    },
    {
      "name": "connectStart",
      "time": [1606814247, 811266158]
    },
    {
      "name": "connectEnd",
      "time": [1606814247, 811266158]
    },
    {
      "name": "requestStart",
      "time": [1606814247, 819101158]
    },
    {
      "name": "responseStart",
      "time": [1606814247, 819791158]
    },
    {
      "name": "responseEnd",
      "time": [1606814247, 820656158]
    }
  ]
}
```

### 계측 추가하기 {#add-instrumentations}

Ajax 요청, 사용자 상호작용 등을 계측하고 싶다면, 추가 계측 라이브러리를 설치하고
등록할 수 있다.

```sh
npm install @opentelemetry/instrumentation-user-interaction \
  @opentelemetry/instrumentation-xml-http-request \
```

```javascript
import { UserInteractionInstrumentation } from '@opentelemetry/instrumentation-user-interaction';
import { XMLHttpRequestInstrumentation } from '@opentelemetry/instrumentation-xml-http-request';

registerInstrumentations({
  instrumentations: [
    new DocumentLoadInstrumentation(),
    new UserInteractionInstrumentation(),
    new XMLHttpRequestInstrumentation(),
  ],
});
```

## 웹을 위한 메타 패키지 {#meta-packages-for-web}

가장 흔한 계측들을 한 번에 활용하려면
[웹을 위한 오픈텔레메트리 메타 패키지](https://www.npmjs.com/package/@opentelemetry/auto-instrumentations-web)를
그냥 사용하면 된다.
