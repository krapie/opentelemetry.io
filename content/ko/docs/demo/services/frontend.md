---
title: 프론트엔드
cSpell:ignore: typeof
default_lang_commit: 8dad29e2443b7c8739f3be322e5d5eec3baf999f
---

프론트엔드는 사용자를 위한 UI를 제공할 뿐만 아니라, UI 또는 다른 클라이언트가
활용하는 API도 제공하는 역할을 담당한다. 이 애플리케이션은
[Next.JS](https://nextjs.org/)를 기반으로 하여 React 기반의 웹 UI와 API 라우트를
제공한다.

[프론트엔드 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/frontend/)

## 서버 계측 {#server-instrumentation}

Node.js 애플리케이션을 시작할 때 SDK와 자동 계측을 초기화하려면 Node의 required
모듈을 사용하는 것이 권장된다. 오픈텔레메트리 Node.js SDK를 초기화할 때, 어떤
자동 계측 라이브러리를 활용할지 선택적으로 지정하거나, 가장 널리 사용되는
프레임워크들을 포함하는 `getNodeAutoInstrumentations()` 함수를 사용할 수 있다.
`utils/telemetry/Instrumentation.js` 파일에는 OTLP 내보내기, 리소스 속성, 서비스
이름에 대한 표준
[오픈텔레메트리 환경 변수](/docs/specs/otel/configuration/sdk-environment-variables/)를
기반으로 SDK와 자동 계측을 초기화하는 데 필요한 모든 코드가 들어 있다.

```javascript
const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  let resource = resourceFromAttributes({
    [ATTR_SERVICE_NAME]: NEXT_PUBLIC_OTEL_SERVICE_NAME,
  });
  const detectedResources = detectResources({ detectors: [browserDetector] });
  resource = resource.merge(detectedResources);

  const provider = new WebTracerProvider({
    resource,
    spanProcessors: [
      new SessionIdProcessor(),
      new BatchSpanProcessor(
        new OTLPTraceExporter({
          url:
            NEXT_PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT ||
            'http://localhost:4318/v1/traces',
        }),
        {
          scheduledDelayMillis: 500,
        },
      ),
    ],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', IS_SYNTHETIC_REQUEST);
          },
        },
      }),
    ],
  });
};
```

Node의 required 모듈은 `--require` 커맨드 라인 인수를 사용하여 로드된다. 이는
`package.json`의 `scripts.start` 섹션에서 설정하고, `npm start`로 애플리케이션을
시작하는 방식으로 수행할 수 있다.

```json
"scripts": {
  "start": "node --require ./Instrumentation.js server.js",
},
```

## 트레이스 {#traces}

### 스팬 예외 및 상태 {#span-exceptions-and-status}

스팬 객체의 `recordException` 함수를 사용하면 처리된 오류의 전체 스택
트레이스(stack trace)를 포함하는 스팬 이벤트를 생성할 수 있다. 예외를 기록할
때는 그에 맞게 스팬의 상태도 반드시 설정해야 한다. 이는
`utils/telemetry/InstrumentationMiddleware.ts` 파일의 `NextApiHandler` 함수 내
catch 블록에서 확인할 수 있다.

```typescript
span.recordException(error as Exception);
span.setStatus({ code: SpanStatusCode.ERROR });
```

### 새 스팬 생성 {#create-new-spans}

새로운 스팬은 `Tracer.startSpan("spanName", options)`를 사용하여 생성하고 시작할
수 있다. 스팬을 생성하는 방식을 지정하기 위해 여러 옵션을 사용할 수 있다.

- `root: true`는 새로운 트레이스를 생성하며, 이 스팬을 루트(root)로 설정한다.
- `links`는 참조해야 하는 다른 스팬(다른 트레이스 내의 스팬이라도)에 대한 링크를
  지정하는 데 사용된다.
- `attributes`는 스팬에 추가되는 키/값 쌍이며, 일반적으로 애플리케이션
  컨텍스트를 나타내는 데 사용된다.

```typescript
span = tracer.startSpan(`${method}`, {
  root: true,
  kind: SpanKind.SERVER,
  links: [{ context: syntheticSpan.spanContext() }],
  attributes: {
    'app.synthetic_request': true,
    [ATTR_HTTP_RESPONSE_STATUS_CODE]: response.statusCode,
    [ATTR_HTTP_REQUEST_METHOD]: method,
    [ATTR_USER_AGENT_ORIGINAL]: headers['user-agent'] || '',
    [ATTR_URL_PATH]: target,
    [ATTR_URL_FULL]: `${headers.host}${url}`,
    [ATTR_NETWORK_PROTOCOL_VERSION]: httpVersion,
  },
});
```

## 브라우저 계측 {#browser-instrumentation}

프론트엔드가 제공하는 웹 기반 UI는 웹 브라우저에 대해서도 계측되어 있다.
오픈텔레메트리 계측은 `pages/_app.tsx`의 Next.js App 컴포넌트의 일부로 포함되어
있다. 여기서 계측이 임포트되고 초기화된다.

```typescript
import FrontendTracer from '../utils/telemetry/FrontendTracer';

if (typeof window !== 'undefined') FrontendTracer();
```

`utils/telemetry/FrontendTracer.ts` 파일에는 TracerProvider를 초기화하고, OTLP
내보내기를 설정하고, 트레이스 컨텍스트 전파자를 등록하고, 웹에 특화된 자동 계측
라이브러리를 등록하는 코드가 들어 있다. 브라우저는 별도의 도메인에 있을 가능성이
높은 오픈텔레메트리 컬렉터로 데이터를 전송하므로, 이에 맞게 CORS 헤더도
설정된다.

백엔드 서비스에 `synthetic_request` 속성 플래그를 전달하기 위한 변경의 일환으로,
`instrumentation-fetch` 라이브러리의 커스텀 스팬 속성 로직에
`applyCustomAttributesOnSpan` 구성 함수가 추가되어, 브라우저 측의 모든 스팬이 이
속성을 포함하도록 한다.

```typescript
import {
  CompositePropagator,
  W3CBaggagePropagator,
  W3CTraceContextPropagator,
} from '@opentelemetry/core';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  const provider = new WebTracerProvider({
    resource: resourceFromAttributes({
      [ATTR_SERVICE_NAME]: process.env.NEXT_PUBLIC_OTEL_SERVICE_NAME,
    }),
    spanProcessors: [new SimpleSpanProcessor(new OTLPTraceExporter())],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', 'false');
          },
        },
      }),
    ],
  });
};

export default FrontendTracer;
```

## 메트릭 {#metrics}

TBD

## 로그 {#logs}

TBD

## 배기지(Baggage) {#baggage}

오픈텔레메트리 배기지는 프론트엔드에서 요청이 합성(synthetic) 요청인지(즉, 부하
생성기(load generator)로부터 온 것인지) 확인하는 데 활용된다. 합성 요청은 새로운
트레이스의 생성을 강제한다. 새로운 트레이스의 루트 스팬은 HTTP 요청이 계측된
스팬과 많은 속성을 동일하게 포함한다.

배기지 항목이 설정되어 있는지 확인하려면 `propagation` API를 활용하여 배기지
헤더를 파싱하고, `baggage` API를 활용하여 항목을 가져오거나 설정할 수 있다.

```typescript
const baggage = propagation.getBaggage(context.active());
if (baggage?.getEntry("synthetic_request")?.value == "true") {...}
```
