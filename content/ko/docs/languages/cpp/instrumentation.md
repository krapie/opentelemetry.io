---
title: 계측
linkTitle: 계측
aliases: [manual]
weight: 30
description: 오픈텔레메트리(OpenTelemetry) C++를 위한 계측
cSpell:ignore: decltype labelkv nostd nullptr
default_lang_commit: db1fe601e5f86c3e4a0c9517e9950dd64bbdd850
---

<!-- markdownlint-disable no-duplicate-heading -->

{{% include instrumentation-intro.md %}}

> [!NOTE]
>
> 계측하려는 라이브러리의 소스 코드를 사용할 수 없는 경우, OpenTelemetry C++는
> 자동 계측(automatic instrumentation)을 지원하지 않는다.

## 설정 {#setup}

OpenTelemetry C++를 빌드하려면
[시작 가이드](/docs/languages/cpp/getting-started/)의 안내를 따른다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initialize-tracing}

```cpp
auto provider = opentelemetry::trace::Provider::GetTracerProvider();
auto tracer = provider->GetTracer("foo_library", "1.0.0");
```

첫 단계에서 얻은 `TracerProvider`는 보통 OpenTelemetry C++ SDK가 제공하는
싱글턴(singleton) 객체이다. 이는 API 인터페이스에 대한 구체적인 구현을 제공하는
데 사용된다. SDK를 사용하지 않는 경우, API는 `TracerProvider`의 기본 no-op(아무
동작도 하지 않는) 구현을 제공한다.

두 번째 단계에서 얻은 `Tracer`는 스팬(Span)을 생성하고 시작하는 데 필요하다.

### 스팬 시작하기 {#start-a-span}

```cpp
auto span = tracer->StartSpan("HandleRequest");
```

이 코드는 스팬을 생성하고, 이름을 `"HandleRequest"`로 설정하며, 시작 시간을 현재
시간으로 설정한다. 스팬에 추가 데이터를 채워 넣는 데 사용할 수 있는 다른 연산은
API 문서를 참고한다.

### 스팬을 활성 상태로 표시하기 {#mark-a-span-as-active}

```cpp
auto scope = tracer->WithActiveSpan(span);
```

이 코드는 스팬을 활성 상태로 표시하고 `Scope` 객체를 반환한다. 스코프(scope)
객체는 스팬이 얼마나 오래 활성 상태를 유지하는지를 제어한다. 스팬은 스코프
객체의 수명(lifetime) 동안 활성 상태를 유지한다.

활성 스팬이라는 개념은 중요한데, 부모를 명시적으로 지정하지 않고 생성된 스팬은
모두 현재 활성 스팬을 부모로 갖게 되기 때문이다. 부모가 없는 스팬을 루트
스팬이라 한다.

### 중첩된 스팬 생성하기 {#create-nested-spans}

```cpp
auto outer_span = tracer->StartSpan("Outer operation");
auto outer_scope = tracer->WithActiveSpan(outer_span);
{
    auto inner_span = tracer->StartSpan("Inner operation");
    auto inner_scope = tracer->WithActiveSpan(inner_span);
    // ... perform inner operation
    inner_span->End();
}
// ... perform outer operation
outer_span->End();
```

스팬은 중첩될 수 있으며, 다른 스팬과 부모-자식 관계를 가질 수 있다. 특정 스팬이
활성 상태일 때, 새로 생성되는 스팬은 활성 스팬의 트레이스 ID와 그 밖의 컨텍스트
속성을 상속받는다.

### 컨텍스트 전파 {#context-propagation}

```cpp
// set global propagator
opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
    nostd::shared_ptr<opentelemetry::context::propagation::TextMapPropagator>(
        new opentelemetry::trace::propagation::HttpTraceContext()));

// get global propagator
HttpTextMapCarrier<opentelemetry::ext::http::client::Headers> carrier;
auto propagator =
    opentelemetry::context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();

//inject context to headers
auto current_ctx = opentelemetry::context::RuntimeContext::GetCurrent();
propagator->Inject(carrier, current_ctx);

//Extract headers to context
auto current_ctx = opentelemetry::context::RuntimeContext::GetCurrent();
auto new_context = propagator->Extract(carrier, current_ctx);
auto remote_span = opentelemetry::trace::propagation::GetSpan(new_context);
```

`Context`는 스팬 ID, 트레이스 ID, 플래그를 포함해 현재 활성 스팬의 메타데이터를
담고 있다. 컨텍스트 전파(Context Propagation)는 이 컨텍스트를 서비스 경계
너머로, 흔히 HTTP 헤더를 통해 전달하기 위한 분산 트레이싱(distributed tracing)의
중요한 메커니즘이다. 오픈텔레메트리는 W3C Trace Context HTTP 헤더를 사용해 원격
서비스로 컨텍스트를 전파하는 텍스트 기반 접근 방식을 제공한다.

### 추가 자료 {#further-reading}

- [트레이스 API](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__trace.html)
- [트레이스 SDK](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__sdk__trace.html)
- [간단한 메트릭 예제](https://github.com/open-telemetry/opentelemetry-cpp/tree/main/examples/metrics_simple)

## 메트릭 {#metrics}

### 익스포터와 리더 초기화하기 {#initialize-exporter-and-reader}

익스포터와 리더(reader)를 초기화한다. 여기서는 기본적으로 stdout에 출력하는
OStream 익스포터를 초기화한다. 리더는 주기적으로 집계 저장소(Aggregation
Store)에서 메트릭을 수집하여 내보낸다.

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::MetricExporter> exporter{new opentelemetry::exporters::OStreamMetricExporter};
std::unique_ptr<opentelemetry::sdk::metrics::MetricReader> reader{
    new opentelemetry::sdk::metrics::PeriodicExportingMetricReader(std::move(exporter), options)};
```

### 미터 프로바이더 초기화하기 {#initialize-a-meter-provider}

`MeterProvider`를 초기화하고 리더를 추가한다. 이후 `Meter` 객체를 얻을 때 이를
사용한다.

```cpp
auto provider = std::shared_ptr<opentelemetry::metrics::MeterProvider>(new opentelemetry::sdk::metrics::MeterProvider());
auto p = std::static_pointer_cast<opentelemetry::sdk::metrics::MeterProvider>(provider);
p->AddMetricReader(std::move(reader));
```

### 카운터 생성하기 {#create-a-counter}

Meter로부터 Counter(카운터) 계측기(instrument)를 생성하고, 측정값을 기록한다.
`MeterProvider`가 반환하는 모든 Meter 포인터는 동일한 Meter를 가리킨다. 이는
Meter를 라이브러리 곳곳에 계속 전달할 필요 없이, 서로 다른 함수에서 캡처한
메트릭을 결합할 수 있다는 의미이다.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto double_counter = meter->CreateDoubleCounter(counter_name);
// Create a label set which annotates metric values
std::map<std::string, std::string> labels = {{"key", "value"}};
auto labelkv = common::KeyValueIterableView<decltype(labels)>{labels};
double_counter->Add(val, labelkv);
```

### 히스토그램 생성하기 {#create-a-histogram}

Meter로부터 Histogram(히스토그램) 계측기를 생성하고, 측정값을 기록한다.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto histogram_counter = meter->CreateDoubleHistogram("histogram_name");
histogram_counter->Record(val, labelkv);
```

### 관측 가능한 카운터 생성하기 {#create-an-observable-counter}

Meter로부터 Observable Counter(관측 가능한 카운터) 계측기를 생성하고, 콜백을
추가한다. 콜백은 메트릭 수집 시점에 측정값을 기록하는 데 사용된다. 수집이
지속되는 동안 Instrument 객체를 계속 활성 상태로 유지해야 한다.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto counter = meter->CreateDoubleObservableCounter(counter_name);
counter->AddCallback(MeasurementFetcher::Fetcher, nullptr);
```

### 뷰 생성하기 {#create-views}

#### Counter 계측기를 Sum 집계에 매핑하기 {#map-the-counter-instrument-to-sum-aggregation}

Counter 계측기를 Sum 집계(Aggregation)에 매핑하는 뷰(View)를 생성한다. 이 뷰를
프로바이더에 추가한다. 커스텀 집계 구성(configuration)이나 속성 프로세서를
추가하고 싶은 경우가 아니라면 뷰 생성은 선택 사항이다. 메트릭 SDK는 계측기와
집계 사이에 누락된 뷰를 기본 매핑으로 생성한다.

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kCounter, "counter_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> meter_selector{
    new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> sum_view{
    new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kSum}};
p->AddView(std::move(instrument_selector), std::move(meter_selector), std::move(sum_view));
```

#### Histogram 계측기를 Histogram 집계에 매핑하기 {#map-the-histogram-instrument-to-histogram-aggregation}

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> histogram_instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kHistogram, "histogram_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> histogram_meter_selector{
    new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> histogram_view{
    new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kHistogram}};
p->AddView(std::move(histogram_instrument_selector), std::move(histogram_meter_selector),
    std::move(histogram_view));
```

#### 관측 가능한 Counter 계측기를 Sum 집계에 매핑하기 {#map-the-observable-counter-instrument-to-sum-aggregation}

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> observable_instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kObservableCounter,
                                     "observable_counter_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> observable_meter_selector{
  new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> observable_sum_view{
  new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kSum}};
p->AddView(std::move(observable_instrument_selector), std::move(observable_meter_selector),
         std::move(observable_sum_view));
```

### 추가 자료 {#further-reading-1}

- [메트릭 API](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__metrics.html#)
- [메트릭 SDK](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__sdk__metrics.html)
- [간단한 메트릭 예제](https://github.com/open-telemetry/opentelemetry-cpp/tree/main/examples/metrics_simple)

## 로그 {#logs}

### 익스포터와 프로세서 초기화하기 {#initialize-exporter-and-processor}

익스포터와 프로세서를 초기화한다. 여기서는 기본적으로 로그 레코드를 stdout에
출력하는 OStream 익스포터를 초기화한다. 프로세서는 LogRecord를 익스포터에
도달하기 전에 내보낼 수 있는 형태로 변환하는 역할을 담당한다.

```cpp
auto exporter = opentelemetry::exporter::logs::OStreamLogRecordExporterFactory::Create();
auto processor =
    opentelemetry::sdk::logs::SimpleLogRecordProcessorFactory::Create(std::move(exporter));
```

### 로거 프로바이더 등록하기 {#register-a-logger-provider}

`LoggerProviderFactory`를 사용해 `LoggerProvider`를 생성하고 이를 전역
프로바이더로 등록한다. 팩토리는 `unique_ptr`를 반환하지만, `SetLoggerProvider`는
`shared_ptr`를 받는다는 점에 유의한다. 이 프로바이더를 사용해 `Logger` 객체를
얻는다. 로거(Logger)에는 이름을 붙여 방출하는 컴포넌트를 식별할 수 있다.

```cpp
// LoggerProviderFactory::Create returns a unique_ptr;
// wrap it in a shared_ptr for SetLoggerProvider
std::shared_ptr<opentelemetry::logs::LoggerProvider> provider(
    opentelemetry::sdk::logs::LoggerProviderFactory::Create(std::move(processor)));
opentelemetry::logs::Provider::SetLoggerProvider(provider);
auto logger = provider->GetLogger(name, "1.0.0");
```

### 로그 레코드 방출하기 {#emit-a-log-record}

얻은 로거를 사용해 구조화된 로그 레코드를 방출한다. 지원되는 심각도(severity)
값은 다음과 같다: `kTrace`, `kDebug`, `kInfo`, `kWarn`, `kError`, `kFatal`.

선택적으로, **암시적으로(implicitly)** 스팬을 스코프에 추가하고 그 스코프 내에서
로그를 남기거나, **명시적으로(explicitly)** 트레이스 ID, 스팬 ID, 플래그를
전달하여 로그 레코드에 트레이스 컨텍스트를 채워 넣을 수 있다.

#### 활성 스팬 스코프 사용하기 {#with-active-span-scope}

스팬을 스코프에 추가해 현재 활성 런타임 컨텍스트로 설정하면, 그 스코프 내에서
방출되는 모든 로그 레코드는 자동으로 스팬 컨텍스트 메타데이터를 얻는다.

```cpp
{
  auto span = get_tracer()->StartSpan("HandleRequest");
  auto scope = opentelemetry::trace::Scope{span};
  // This log record gets trace_id, span_id, and trace_flags, etc.
  // from the active scope
  logger->Info("Handling request");
}
```

#### 명시적 트레이스 컨텍스트 사용하기 {#with-explicit-trace-context}

트레이스 컨텍스트를 명시적으로 전달할 수도 있다.

```cpp
auto span = get_tracer()->StartSpan("HandleRequest");
auto ctx = span->GetContext();
logger->Info("Handling request", ctx.trace_id(), ctx.span_id(), ctx.trace_flags());
```

### 추가 자료 {#further-reading-2}

- [로그 API](/docs/specs/otel/logs/api/)
- [로그 SDK](/docs/specs/otel/logs/sdk/)
- [로그 예제](https://github.com/open-telemetry/opentelemetry-cpp/tree/main/examples/logs_simple)

## 다음 단계 {#next-steps}

또한 텔레메트리 백엔드 한 곳 이상으로
[텔레메트리 데이터를 내보내기](/docs/languages/cpp/exporters) 위해 적절한
익스포터를 구성해야 할 것이다.
