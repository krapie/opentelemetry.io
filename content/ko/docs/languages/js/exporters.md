---
title: 익스포터
weight: 50
description: 텔레메트리 데이터를 처리하고 내보내기
default_lang_commit: f49ec57e5a0ec766b07c7c8e8974c83531620af3
---

{{% docs/languages/exporters/intro %}}

## 의존성 {#otlp-dependencies}

([컬렉터(Collector)](#collector-setup), [Jaeger](#jaeger),
[Prometheus](#prometheus)와 같은) OTLP 엔드포인트로 텔레메트리 데이터를 보내고
싶다면, 데이터를 전송할 세 가지 프로토콜 중 하나를 선택할 수 있다.

- [HTTP/protobuf](https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-proto)
- [HTTP/JSON](https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-http)
- [gRPC](https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-grpc)

먼저 프로젝트의 의존성으로 해당 익스포터 패키지를 설치한다.

{{< tabpane text=true >}} {{% tab "HTTP/Proto" %}}

```shell
npm install --save @opentelemetry/exporter-trace-otlp-proto \
  @opentelemetry/exporter-metrics-otlp-proto
```

{{% /tab %}} {{% tab "HTTP/JSON" %}}

```shell
npm install --save @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http
```

{{% /tab %}} {{% tab gRPC %}}

```shell
npm install --save @opentelemetry/exporter-trace-otlp-grpc \
  @opentelemetry/exporter-metrics-otlp-grpc
```

{{% /tab %}} {{< /tabpane >}}

## Node.js에서 사용하기 {#usage-with-nodejs}

다음으로, OTLP 엔드포인트를 가리키도록 익스포터를 구성한다. 예를 들어
[시작하기](/docs/languages/js/getting-started/nodejs/)의 `instrumentation.ts`
파일(JavaScript를 사용한다면 `instrumentation.js`)을 다음과 같이 업데이트해
OTLP(`http/protobuf`)로 트레이스와 메트릭을 내보낼 수 있다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*instrumentation.ts*/
import * as opentelemetry from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-proto';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new OTLPTraceExporter({
    // optional - default url is http://localhost:4318/v1/traces
    url: '<your-otlp-endpoint>/v1/traces',
    // optional - collection of custom headers to be sent with each request, empty by default
    headers: {},
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: '<your-otlp-endpoint>/v1/metrics', // url is optional and can be omitted - default is http://localhost:4318/v1/metrics
      headers: {}, // an optional object containing custom headers to be sent with each request
    }),
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*instrumentation.js*/
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');
const {
  OTLPTraceExporter,
} = require('@opentelemetry/exporter-trace-otlp-proto');
const {
  OTLPMetricExporter,
} = require('@opentelemetry/exporter-metrics-otlp-proto');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new OTLPTraceExporter({
    // optional - default url is http://localhost:4318/v1/traces
    url: '<your-otlp-endpoint>/v1/traces',
    // optional - collection of custom headers to be sent with each request, empty by default
    headers: {},
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: '<your-otlp-endpoint>/v1/metrics', // url is optional and can be omitted - default is http://localhost:4318/v1/metrics
      headers: {}, // an optional object containing custom headers to be sent with each request
      concurrencyLimit: 1, // an optional limit on pending requests
    }),
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{< /tabpane >}}

## 브라우저에서 사용하기 {#usage-in-the-browser}

브라우저 기반 애플리케이션에서 OTLP 익스포터를 사용할 때는 다음을 유의해야 한다.

1. 내보내기에 gRPC를 사용하는 것은 지원되지 않는다
2. 웹사이트의 [콘텐츠 보안 정책][](CSP)이 내보내기를 차단할 수 있다
3. [교차 출처 리소스 공유][](CORS) 헤더가 내보내기 전송을 허용하지 않을 수 있다
4. 컬렉터를 퍼블릭 인터넷에 노출해야 할 수도 있다

아래에서 올바른 익스포터를 사용하는 방법, CSP와 CORS 헤더를 구성하는 방법,
컬렉터를 노출할 때 취해야 할 주의 사항에 대한 안내를 확인할 수 있다.

### HTTP/JSON 또는 HTTP/protobuf로 OTLP 익스포터 사용하기 {#use-otlp-exporter-with-httpjson-or-httpprotobuf}

[gRPC를 사용하는 오픈텔레메트리 컬렉터 익스포터][]는 Node.js에서만 동작하므로,
브라우저에서는 [HTTP/JSON을 사용하는 오픈텔레메트리 컬렉터 익스포터][]나
[HTTP/protobuf를 사용하는 오픈텔레메트리 컬렉터 익스포터][]만 사용할 수 있다.

[HTTP/JSON을 사용하는 오픈텔레메트리 컬렉터 익스포터][]를 사용한다면 익스포터의
수신 측(컬렉터 또는 옵저버빌리티 백엔드)이 `http/json`을 허용하는지, 포트를
4318로 설정한 올바른 엔드포인트로 데이터를 내보내는지 확인한다.

### CSP 구성하기 {#configure-csps}

웹사이트가 콘텐츠 보안 정책(CSP)을 사용하고 있다면, OTLP 엔드포인트의 도메인이
포함되어 있는지 확인한다. 컬렉터 엔드포인트가
`https://collector.example.com:4318/v1/traces`라면, 다음 디렉티브를 추가한다.

```text
connect-src collector.example.com:4318/v1/traces
```

CSP에 OTLP 엔드포인트가 포함되어 있지 않으면, 엔드포인트로의 요청이 CSP
디렉티브를 위반한다는 오류 메시지가 표시된다.

### CORS 헤더 구성하기 {#configure-cors-headers}

웹사이트와 컬렉터가 서로 다른 출처에서 호스팅된다면, 브라우저가 컬렉터로 나가는
요청을 차단할 수 있다. 교차 출처 리소스 공유(CORS)를 위한 특별한 헤더를 구성해야
한다.

오픈텔레메트리 컬렉터는 HTTP 기반 리시버가 웹 브라우저로부터 트레이스를 받을 수
있도록 필요한 헤더를 추가하는 [기능][]을 제공한다.

```yaml
receivers:
  otlp:
    protocols:
      http:
        include_metadata: true
        cors:
          allowed_origins:
            - https://foo.bar.com
            - https://*.test.com
          allowed_headers:
            - Example-Header
          max_age: 7200
```

### 컬렉터를 안전하게 노출하기 {#securely-expose-your-collector}

웹 애플리케이션으로부터 텔레메트리를 받으려면, 최종 사용자의 브라우저가 컬렉터로
데이터를 보낼 수 있도록 허용해야 한다. 웹 애플리케이션이 퍼블릭 인터넷에서 접근
가능하다면, 컬렉터도 모두에게 접근 가능하도록 만들어야 한다.

컬렉터를 직접 노출하지 말고, 그 앞에 리버스 프록시(NGINX, Apache HTTP Server
등)를 두는 것을 권장한다. 리버스 프록시는 SSL 오프로딩, 올바른 CORS 헤더 설정 등
웹 애플리케이션에 특화된 다양한 기능을 처리할 수 있다.

아래에서 시작하는 데 도움이 될 인기 있는 NGINX 웹 서버 구성을 확인할 수 있다.

```nginx
server {
    listen 80 default_server;
    server_name _;
    location / {
        # Take care of preflight requests
        if ($request_method = 'OPTIONS') {
             add_header 'Access-Control-Max-Age' 1728000;
             add_header 'Access-Control-Allow-Origin' 'name.of.your.website.example.com' always;
             add_header 'Access-Control-Allow-Headers' 'Accept,Accept-Language,Content-Language,Content-Type' always;
             add_header 'Access-Control-Allow-Credentials' 'true' always;
             add_header 'Content-Type' 'text/plain charset=UTF-8';
             add_header 'Content-Length' 0;
             return 204;
        }

        add_header 'Access-Control-Allow-Origin' 'name.of.your.website.example.com' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Allow-Headers' 'Accept,Accept-Language,Content-Language,Content-Type' always;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://collector:4318;
    }
}
```

## 콘솔 {#console}

계측을 디버깅하거나 개발 중에 로컬에서 값을 확인하려면, 텔레메트리 데이터를
콘솔(stdout)에 기록하는 익스포터를 사용할 수 있다.

[시작하기](/docs/languages/js/getting-started/nodejs/)나
[수동 계측](/docs/languages/js/instrumentation) 가이드를 따라 했다면, 이미 콘솔
익스포터가 설치되어 있다.

`ConsoleSpanExporter`는
[`@opentelemetry/sdk-trace-node`](https://www.npmjs.com/package/@opentelemetry/sdk-trace-node)
패키지에, `ConsoleMetricExporter`는
[`@opentelemetry/sdk-metrics`](https://www.npmjs.com/package/@opentelemetry/sdk-metrics)
패키지에 포함되어 있다.

{{% include "exporters/jaeger.md" %}}

{{% include "exporters/prometheus-setup.md" %}}

## 의존성 {#prometheus-dependencies}

[익스포터 패키지](https://www.npmjs.com/package/@opentelemetry/exporter-prometheus)를
애플리케이션의 의존성으로 설치한다.

```shell
npm install --save @opentelemetry/exporter-prometheus
```

오픈텔레메트리 구성을 업데이트해 익스포터를 사용하고 Prometheus 백엔드로
데이터를 보내도록 한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import * as opentelemetry from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { PrometheusExporter } from '@opentelemetry/exporter-prometheus';

const sdk = new opentelemetry.NodeSDK({
  metricReader: new PrometheusExporter({
    port: 9464, // optional - default is 9464
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');
const { PrometheusExporter } = require('@opentelemetry/exporter-prometheus');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');
const sdk = new opentelemetry.NodeSDK({
  metricReader: new PrometheusExporter({
    port: 9464, // optional - default is 9464
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{< /tabpane >}}

위와 같이 하면 <http://localhost:9464/metrics>에서 메트릭에 접근할 수 있다.
Prometheus나 Prometheus 리시버를 사용하는 오픈텔레메트리 컬렉터가 이
엔드포인트에서 메트릭을 스크레이핑할 수 있다.

{{% include "exporters/zipkin-setup.md" %}}

## 의존성 {#zipkin-dependencies}

트레이스 데이터를 [Zipkin](https://zipkin.io/)으로 보내려면, `ZipkinExporter`를
사용할 수 있다.

[익스포터 패키지](https://www.npmjs.com/package/@opentelemetry/exporter-zipkin)를
애플리케이션의 의존성으로 설치한다.

```shell
npm install --save @opentelemetry/exporter-zipkin
```

오픈텔레메트리 구성을 업데이트해 익스포터를 사용하고 Zipkin 백엔드로 데이터를
보내도록 한다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import * as opentelemetry from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { ZipkinExporter } from '@opentelemetry/exporter-zipkin';

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new ZipkinExporter({}),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');
const { ZipkinExporter } = require('@opentelemetry/exporter-zipkin');

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new ZipkinExporter({}),
  instrumentations: [getNodeAutoInstrumentations()],
});
```

{{% /tab %}} {{< /tabpane >}}

{{% include "exporters/outro.md" `https://open-telemetry.github.io/opentelemetry-js/interfaces/_opentelemetry_sdk-trace-base.SpanExporter.html` %}}

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
/*instrumentation.ts*/
import * as opentelemetry from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  spanProcessors: [new SimpleSpanProcessor(exporter)],
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{% tab JavaScript %}}

```js
/*instrumentation.js*/
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');

const sdk = new opentelemetry.NodeSDK({
  spanProcessors: [new SimpleSpanProcessor(exporter)],
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

{{% /tab %}} {{< /tabpane >}}

[콘텐츠 보안 정책]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/
[교차 출처 리소스 공유]: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
[gRPC를 사용하는 오픈텔레메트리 컬렉터 익스포터]:
  https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-grpc
[HTTP/protobuf를 사용하는 오픈텔레메트리 컬렉터 익스포터]:
  https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-proto
[HTTP/JSON을 사용하는 오픈텔레메트리 컬렉터 익스포터]:
  https://www.npmjs.com/package/@opentelemetry/exporter-trace-otlp-http
[기능]:
  https://github.com/open-telemetry/opentelemetry-collector/blob/main/config/confighttp/README.md
