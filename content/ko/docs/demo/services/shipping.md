---
title: 배송 서비스
linkTitle: 배송
aliases: [shippingservice]
cSpell:ignore: sdktrace
default_lang_commit: f92688f04b7162bd5ab6dbf7578bfb9e5c4263b2
---

이 서비스는 체크아웃 서비스에서 요청받았을 때 가격 및 추적 정보를 포함한 배송
정보를 제공하는 역할을 담당한다.

배송 서비스는 [Actix Web](https://actix.rs/), 로그를 위한
[Tracing](https://tracing.rs/), 오픈텔레메트리(OpenTelemetry) 라이브러리로
빌드된다. 그 밖의 모든 하위 의존성은 `Cargo.toml`에 포함되어 있다.

사용 중인 프레임워크와 런타임에 따라, 보충 자료로
[Rust 문서](/docs/languages/rust/)를 참고할 수 있다. 견적 요청과 추적 ID에서
각각 비동기(async) 스팬과 동기(sync) 스팬의 예시를 확인할 수 있다.

[배송 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/shipping/)

## 계측 {#instrumentation}

오픈텔레메트리 SDK는 `telemetry_conf` 파일에서 구성된다.

기본 리소스 감지기(Resource Detector)에 `OS` 및 `Process` 감지기를 더해 리소스를
생성하는 `get_resource()` 함수가 구현되어 있다.

```rust
fn get_resource() -> Resource {
    let detectors: Vec<Box<dyn ResourceDetector>> = vec![
        Box::new(OsResourceDetector),
        Box::new(ProcessResourceDetector),
    ];

    Resource::builder().with_detectors(&detectors).build()
}
```

`get_resource()`를 만들어 두면, 모든 프로바이더 초기화 과정에서 이 함수를 여러
번 호출할 수 있다.

### 트레이서 프로바이더 초기화 {#initializing-tracer-provider}

```rust
fn init_tracer_provider() {
    global::set_text_map_propagator(TraceContextPropagator::new());

    let tracer_provider = opentelemetry_sdk::trace::SdkTracerProvider::builder()
        .with_resource(get_resource())
        .with_batch_exporter(
            opentelemetry_otlp::SpanExporter::builder()
                .with_tonic()
                .build()
                .expect("Failed to initialize tracing provider"),
        )
        .build();

    global::set_tracer_provider(tracer_provider);
}
```

### 미터 프로바이더 초기화 {#initializing-meter-provider}

```rust
fn init_meter_provider() -> opentelemetry_sdk::metrics::SdkMeterProvider {
    let meter_provider = opentelemetry_sdk::metrics::SdkMeterProvider::builder()
        .with_resource(get_resource())
        .with_periodic_exporter(
            opentelemetry_otlp::MetricExporter::builder()
                .with_temporality(opentelemetry_sdk::metrics::Temporality::Delta)
                .with_tonic()
                .build()
                .expect("Failed to initialize metric exporter"),
        )
        .build();
    global::set_meter_provider(meter_provider.clone());

    meter_provider
}
```

### 로거 프로바이더 초기화 {#initializing-logger-provider}

배송 서비스는 로그에 Tracing을 사용하므로, Tracing 크레이트(crate)의 로그를
오픈텔레메트리로 연결하기 위해 `OpenTelemetryTracingBridge`를 사용한다.

```rust
fn init_logger_provider() {
    let logger_provider = opentelemetry_sdk::logs::SdkLoggerProvider::builder()
        .with_resource(get_resource())
        .with_batch_exporter(
            opentelemetry_otlp::LogExporter::builder()
                .with_tonic()
                .build()
                .expect("Failed to initialize logger provider"),
        )
        .build();

    let otel_layer = OpenTelemetryTracingBridge::new(&logger_provider);
    let filter_otel = EnvFilter::new("info");
    let otel_layer = otel_layer.with_filter(filter_otel);

    tracing_subscriber::registry().with(otel_layer).init();
}
```

### 계측 초기화 {#instrumentation-initialization}

트레이스, 메트릭, 로그용 프로바이더를 초기화하는 함수를 정의한 뒤, 공개 함수
`init_otel()`이 생성된다.

```rust
pub fn init_otel() -> Result<()> {
    init_logger_provider();
    init_tracer_provider();
    init_meter_provider();
    Ok(())
}
```

이 함수는 모든 초기화 함수를 호출하고, 모든 것이 정상적으로 시작되면 `OK(())`를
반환한다.

이후 `init_otel()` 함수는 `main`에서 호출된다.

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    match init_otel() {
        Ok(_) => {
            info!("Successfully configured OTel");
        }
        Err(err) => {
            panic!("Couldn't start OTel: {0}", err);
        }
    };

    [...]

}
```

### 계측 구성 {#instrumentation-configuration}

이제 프로바이더가 구성되고 초기화되었으므로, 배송 서비스는 서버 측 및 클라이언트
측 구성 과정에서 애플리케이션을 계측하기 위해
[`opentelemetry-instrumentation-actix-web` 크레이트](https://crates.io/crates/opentelemetry-instrumentation-actix-web)를
사용한다.

#### 서버 측 {#server-side}

서버는 요청을 받을 때 트레이스와 메트릭을 자동으로 생성하도록 `RequestTracing`과
`RequestMetrics`로 감싸져 있다.

```rust
HttpServer::new(|| {
    App::new()
        .wrap(RequestTracing::new())
        .wrap(RequestMetrics::default())
        .service(get_quote)
        .service(ship_order)
})
```

#### 클라이언트 측 {#client-side}

다른 서비스로 요청을 보낼 때는 해당 호출에 `trace_request()`가 추가된다.

```rust
let mut response = client
    .post(quote_service_addr)
    .trace_request()
    .send_json(&reqbody)
    .await
    .map_err(|err| anyhow::anyhow!("Failed to call quote service: {err}"))?;
```

### 수동 계측 {#manual-instrumentation}

`opentelemetry-instrumentation-actix-web` 크레이트는 앞 절에서 언급한 명령을
추가함으로써 서버 측과 클라이언트 측을 계측할 수 있게 해준다.

이 데모에서는 자동으로 생성된 스팬을 수동으로 보강하는 방법과 애플리케이션에서
수동 메트릭을 생성하는 방법도 함께 보여준다.

#### 수동 스팬 {#manual-spans}

다음 스니펫에서는 현재 활성 스팬에 스팬 이벤트와 스팬 속성이 추가되어 보강된다.

```rust
Ok(get_active_span(|span| {
    let q = create_quote_from_float(f);
    span.add_event(
        "Received Quote".to_string(),
        vec![KeyValue::new("app.shipping.cost.total", format!("{}", q))],
    );
    span.set_attribute(KeyValue::new("app.shipping.cost.total", format!("{}", q)));
    q
}))
```

#### 수동 메트릭 {#manual-metrics}

배송 요청에 포함된 항목 수를 세기 위한 커스텀 메트릭 카운터가 생성된다.

```rust
let meter = global::meter("otel_demo.shipping.quote");
let counter = meter.u64_counter("app.shipping.items_count").build();
counter.add(count as u64, &[]);
```

### 로그 {#logs}

배송 서비스는 로그 인터페이스로 Tracing을 사용하므로, Tracing 로그를
오픈텔레메트리 로그로 연결하기 위해 `opentelemetry-appender-tracing` 크레이트를
사용한다.

이 어펜더(appender)는 [로거 프로바이더 초기화](#initializing-logger-provider)
과정에서 다음 두 줄로 이미 구성되었다.

```rust
let otel_layer = OpenTelemetryTracingBridge::new(&logger_provider);
tracing_subscriber::registry().with(otel_layer).init();
```

이렇게 구성되면, 평소처럼 Tracing을 사용할 수 있다. 예를 들면 다음과 같다.

```rust
info!(
    name = "SendingQuoteValue",
    quote.dollars = quote.dollars,
    quote.cents = quote.cents,
    message = "Sending Quote"
);
```

`opentelemetry-appender-tracing` 크레이트는 오픈텔레메트리 컨텍스트를 로그
엔트리에 추가하는 작업을 처리하며, 최종적으로 내보내지는 로그에는 구성된 모든
리소스 속성과 `TraceContext` 정보가 포함된다.
