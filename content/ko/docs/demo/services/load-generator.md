---
title: 부하 생성기
aliases: [loadgenerator]
cSpell:ignore: baggage goroutines loadgenerator otelHeaders xk6
default_lang_commit: c060ef7682b152a285d3a2f0c6a84c93ff877070
---

부하 생성기는 JavaScript로 작성된 테스트 시나리오를 실행하는 Go 부하 테스트
도구인 [k6](https://k6.io)를 기반으로 한다. 기본적으로 프론트엔드에 대해 여러
다른 경로를 요청하는 사용자를 시뮬레이션한다. 부하 생성기의 모든
오픈텔레메트리(OpenTelemetry) 계측은 아래에서 설명하는 `xk6-otel`
익스텐션(extension)에 의해 k6 바이너리에 내장된 Go SDK에서 나온다.

[부하 생성기 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/load-generator/)

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

부하 생성기의 트레이싱은 오픈텔레메트리 Go SDK를 감싸서 k6 JavaScript 스크립트에
노출하는 커스텀 [xk6](https://github.com/grafana/xk6) 익스텐션(`xk6-otel`)이
제공한다. 이 익스텐션은 이미지 빌드 시점에 k6 바이너리에 컴파일되어 포함된다.

이 익스텐션은 처음 사용될 때 OTLP HTTP 익스포터를 사용하는 `TracerProvider`를
초기화한다. 컬렉터 엔드포인트, 프로토콜, 리소스 속성, 서비스 이름은 표준
[오픈텔레메트리 환경 변수](/docs/specs/otel/configuration/sdk-environment-variables/)(`OTEL_EXPORTER_OTLP_ENDPOINT`,
`OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_RESOURCE_ATTRIBUTES`,
`OTEL_SERVICE_NAME`)에서 읽어온다.

### 스팬 생성 {#creating-spans}

스크립트는 `k6/x/otel` 모듈에서 `Tracer` 클래스를 임포트하여 시뮬레이션된 각
사용자 작업 주위에 수동으로 스팬을 생성한다.

```javascript
import { Tracer } from 'k6/x/otel';

const tracer = new Tracer();

function browseProduct() {
  const span = tracer.startSpan('user_browse_product', {
    'product.id': product,
  });
  http.get(`${BASE_URL}/api/products/${product}`, {
    headers: otelHeaders(span.traceParent()),
  });
  span.end();
}
```

`startSpan(name, attrs?)` 메서드는 새 클라이언트 스팬을 시작하고, 세 가지
메서드를 가진 객체를 반환한다.

- `traceParent()` — 트레이스 컨텍스트를 백엔드 서비스로 전파하는 데 사용되는,
  해당 스팬의 W3C `traceparent` 헤더 값을 반환한다.
- `log(message)` — 스팬의 트레이스 ID 및 스팬 ID와 연관된 OTel 로그 레코드를
  발생시킨다.
- `end()` — 스팬을 종료하고 익스포터로 플러시(flush)한다.

## 메트릭 {#metrics}

부하 생성기는 두 종류의 메트릭을 발생시킨다.

- **k6 내장 테스트 메트릭**(요청 지속 시간, 오류율, 처리량 등)은 k6의 내장
  `opentelemetry` 출력(`--out opentelemetry`)을 통해 오픈텔레메트리 컬렉터로
  내보내진다. 출력 프로토콜과 컬렉터 엔드포인트는 `K6_OTEL_EXPORTER_PROTOCOL` 및
  `K6_OTEL_HTTP_EXPORTER_ENDPOINT` 환경 변수를 통해 구성한다.
- **Go 런타임 메트릭**(메모리, 가비지 컬렉션, 고루틴(goroutine))은 `xk6-otel`
  익스텐션이 오픈텔레메트리 `runtime` 계측을 사용하여 발생시킨다.

## 로그 {#logs}

로그 레코드는 활성 스팬에서 `span.log(message)`를 호출하여 발생시킨다.
`xk6-otel` 익스텐션은 스팬의 트레이스 ID와 스팬 ID를 각 로그 레코드에 주입한다.

## 배기지 {#baggage}

오픈텔레메트리 배기지(Baggage)는 부하 생성기가 트레이스가
합성으로(synthetically) 생성되었음을 나타내는 데 사용된다. 나가는 각 HTTP 요청은
`otelHeaders` 헬퍼가 구성하는 `baggage` 헤더와 `traceparent` 헤더를 함께
전달한다.

```javascript
function otelHeaders(traceParent, extra) {
  return Object.assign(
    {
      baggage: `synthetic_request=true,session.id=${sessionId}`,
      traceparent: traceParent,
    },
    extra,
  );
}
```

배기지 자체만으로는 텔레메트리에 표시를 남기지 않는다. 각 백엔드 서비스는
들어오는 배기지에서 `synthetic_request` 항목을 읽어 이를 속성(attribute)으로
자신의 스팬과 로그 레코드에 복사하며, 텔레메트리가 합성 흐름에서 나온 것인지
여부를 기록하는 것은 바로 이 속성이다. 프론트엔드는 `demo.synthetic_request`를
설정하고, 체크아웃 서비스와 결제 서비스는 `user_agent.synthetic.type`을 `test`로
설정한다. 이 표시가 텔레메트리 자체에 남기 때문에, 옵저버빌리티 백엔드에서 어떤
쿼리에서든 부하 생성기 트래픽을 포함하거나 제외하도록 필터링할 수 있다.
