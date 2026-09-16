---
title: 계측 라이브러리 사용하기
linkTitle: 라이브러리
weight: 40
description: 앱이 의존하는 라이브러리를 계측하는 방법
cSpell:ignore: metapackage metapackages
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

{{% docs/languages/libraries-intro "js" %}}

## 계측 라이브러리 사용하기 {#use-instrumentation-libraries}

라이브러리가 기본적으로 오픈텔레메트리(OpenTelemetry)를 지원하지 않는다면,
[계측 라이브러리](/docs/specs/otel/glossary/#instrumentation-library)를 사용해
해당 라이브러리나 프레임워크의 텔레메트리 데이터를 생성할 수 있다.

예를 들어
[Express용 계측 라이브러리](https://www.npmjs.com/package/@opentelemetry/instrumentation-express)는
인바운드 HTTP 요청을 기반으로 [스팬](/docs/concepts/signals/traces/#spans)을
자동으로 생성한다.

{{% include esm-support-note.md %}}

### 설정 {#setup}

각 계측 라이브러리는 NPM 패키지이다. 예를 들어 인바운드 및 아웃바운드 HTTP
트래픽을 계측하기 위해
[instrumentation-express](https://www.npmjs.com/package/@opentelemetry/instrumentation-express)와
[instrumentation-http](https://www.npmjs.com/package/@opentelemetry/instrumentation-http)
계측 라이브러리를 설치하는 방법은 다음과 같다.

```sh
npm install --save @opentelemetry/instrumentation-http @opentelemetry/instrumentation-express
```

OpenTelemetry JavaScript는 또한
[auto-instrumentation-node](https://www.npmjs.com/package/@opentelemetry/auto-instrumentations-node)와
[auto-instrumentation-web](https://www.npmjs.com/package/@opentelemetry/auto-instrumentations-web)
메타패키지를 정의하는데, 이는 Node.js 또는 웹 기반 계측 라이브러리를 모두 하나의
패키지로 묶은 것이다. 최소한의 노력으로 모든 라이브러리에 대해 자동으로 생성되는
텔레메트리를 추가할 수 있는 편리한 방법이다.

{{< tabpane text=true >}}

{{% tab Node.js %}}

```shell
npm install --save @opentelemetry/auto-instrumentations-node
```

{{% /tab %}}

{{% tab Browser %}}

```shell
npm install --save @opentelemetry/auto-instrumentations-web
```

{{% /tab %}} {{< /tabpane >}}

이러한 메타패키지를 사용하면 의존성 그래프의 크기가 커진다는 점에 유의한다.
정확히 어떤 것이 필요한지 알고 있다면 개별 계측 라이브러리를 사용한다.

### 등록 {#registration}

필요한 계측 라이브러리를 설치한 후에는, 이를 Node.js용 오픈텔레메트리 SDK에
등록해야 한다. [시작하기](/docs/languages/js/getting-started/nodejs/)를 따라
했다면 이미 메타패키지를 사용하고 있을 것이다.
[수동 계측을 위해 SDK를 초기화하는 방법](/docs/languages/js/instrumentation/#initialize-tracing)을
따랐다면, `instrumentation.ts`(또는 `instrumentation.js`)를 다음과 같이
업데이트한다.

{{< tabpane text=true >}}

{{% tab TypeScript %}}

```typescript
/*instrumentation.ts*/
...
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  ...
  // This registers all instrumentation packages
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start()
```

{{% /tab %}}

{{% tab JavaScript %}}

```javascript
/*instrumentation.js*/
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  ...
  // This registers all instrumentation packages
  instrumentations: [getNodeAutoInstrumentations()]
});
```

{{% /tab %}}

{{< /tabpane >}}

개별 계측 라이브러리를 비활성화하려면 다음과 같이 변경한다.

{{< tabpane text=true >}}

{{% tab TypeScript %}}

```typescript
/*instrumentation.ts*/
...
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  ...
  // This registers all instrumentation packages
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': {
        enabled: false,
      },
    }),
  ],
});

sdk.start()
```

{{% /tab %}}

{{% tab JavaScript %}}

```javascript
/*instrumentation.js*/
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  ...
  // This registers all instrumentation packages
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': {
        enabled: false,
      },
    }),
  ],
});
```

{{% /tab %}}

{{< /tabpane >}}

개별 계측 라이브러리만 불러오려면, `[getNodeAutoInstrumentations()]`를 필요한
것들의 목록으로 바꾼다.

{{< tabpane text=true >}}

{{% tab TypeScript %}}

```typescript
/*instrumentation.ts*/
...
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";

const sdk = new NodeSDK({
  ...
  instrumentations: [
    // Express instrumentation expects HTTP layer to be instrumented
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ]
});

sdk.start()
```

{{% /tab %}} {{% tab JavaScript %}}

```javascript
/*instrumentation.js*/
const { HttpInstrumentation } = require("@opentelemetry/instrumentation-http");
const { ExpressInstrumentation } = require("@opentelemetry/instrumentation-express");

const sdk = new NodeSDK({
  ...
  instrumentations: [
    // Express instrumentation expects HTTP layer to be instrumented
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ]
});
```

{{% /tab %}}

{{< /tabpane >}}

### 구성 {#configuration}

일부 계측 라이브러리는 추가 구성 옵션을 제공한다.

예를 들어
[Express 계측](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/instrumentation-express#express-instrumentation-options)은
특정 미들웨어를 무시하거나 요청 훅으로 자동 생성된 스팬을 보강하는 방법을
제공한다.

{{< tabpane text=true >}}

{{% tab TypeScript %}}

```typescript
import { Span } from '@opentelemetry/api';
import {
  ATTR_HTTP_REQUEST_METHOD,
  ATTR_URL_FULL,
} from '@opentelemetry/semantic-conventions';
import {
  ExpressInstrumentation,
  ExpressLayerType,
  ExpressRequestInfo,
} from '@opentelemetry/instrumentation-express';

const expressInstrumentation = new ExpressInstrumentation({
  requestHook: function (span: Span, info: ExpressRequestInfo) {
    if (info.layerType === ExpressLayerType.REQUEST_HANDLER) {
      span.setAttribute(ATTR_HTTP_REQUEST_METHOD, info.request.method);
      span.setAttribute(ATTR_URL_FULL, info.request.baseUrl);
    }
  },
});
```

{{% /tab %}}

{{% tab JavaScript %}}

```javascript
/*instrumentation.js*/
const {
  ATTR_HTTP_REQUEST_METHOD,
  ATTR_URL_FULL,
} = require('@opentelemetry/semantic-conventions');
const {
  ExpressInstrumentation,
  ExpressLayerType,
} = require('@opentelemetry/instrumentation-express');

const expressInstrumentation = new ExpressInstrumentation({
  requestHook: function (span, info) {
    if (info.layerType === ExpressLayerType.REQUEST_HANDLER) {
      span.setAttribute(ATTR_HTTP_REQUEST_METHOD, info.request.method);
      span.setAttribute(ATTR_URL_FULL, info.request.baseUrl);
    }
  },
});
```

{{% /tab %}}

{{< /tabpane >}}

고급 구성에 대해서는 각 계측 라이브러리의 문서를 참고해야 한다.

### 사용 가능한 계측 라이브러리 {#available-instrumentation-libraries}

사용 가능한 계측 목록은
[레지스트리](/ecosystem/registry/?language=js&component=instrumentation)에서
찾을 수 있다.

## 라이브러리를 네이티브로 계측하기 {#instrument-a-library-natively}

라이브러리에 네이티브 계측을 추가하고 싶다면, 다음 문서를 검토해야 한다.

- 개념 페이지 [라이브러리](/docs/concepts/instrumentation/libraries/)는 언제
  계측해야 하고 무엇을 계측해야 하는지에 대한 통찰을 제공한다
- [수동 계측](/docs/languages/js/instrumentation/)은 라이브러리를 위한 트레이스,
  메트릭, 로그를 생성하는 데 필요한 코드 예시를 제공한다
- Node.js와 브라우저를 위한
  [계측 구현 가이드](https://github.com/open-telemetry/opentelemetry-js-contrib/blob/main/GUIDELINES.md)에는
  라이브러리 계측을 만들기 위한 JavaScript 특화 모범 사례가 담겨 있다.

## 계측 라이브러리 만들기 {#create-an-instrumentation-library}

애플리케이션에 대해 기본적으로 옵저버빌리티를 확보하는 것이 바람직한 방식이지만,
항상 가능하거나 바람직한 것은 아니다. 이런 경우 인터페이스 래핑, 라이브러리 특화
콜백 구독, 기존 텔레메트리를 오픈텔레메트리 모델로 변환하는 것과 같은 메커니즘을
사용해 계측 호출을 주입하는 계측 라이브러리를 만들 수 있다.

이러한 라이브러리를 만들려면 Node.js와 브라우저를 위한
[계측 구현 가이드](https://github.com/open-telemetry/opentelemetry-js-contrib/blob/main/GUIDELINES.md)를
따른다.
