---
title: 통화 서비스
linkTitle: 통화
aliases: [currencyservice]
cSpell:ignore: decltype labelkv noexcept nostd
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

이 서비스는 서로 다른 통화 간 금액을 변환하는 기능을 제공한다.

[통화 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/currency/)

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

오픈텔레메트리(OpenTelemetry) SDK는 `tracer_common.h`에 정의된 `initTracer`
함수를 사용해 `main`에서 초기화된다.

```cpp
void initTracer()
{
  auto exporter = opentelemetry::exporter::otlp::OtlpGrpcExporterFactory::Create();
  auto processor =
      opentelemetry::sdk::trace::SimpleSpanProcessorFactory::Create(std::move(exporter));
  std::vector<std::unique_ptr<opentelemetry::sdk::trace::SpanProcessor>> processors;
  processors.push_back(std::move(processor));
  std::shared_ptr<opentelemetry::sdk::trace::TracerContext> context =
      opentelemetry::sdk::trace::TracerContextFactory::Create(std::move(processors));
  std::shared_ptr<opentelemetry::trace::TracerProvider> provider =
      opentelemetry::sdk::trace::TracerProviderFactory::Create(context);
 // Set the global trace provider
  opentelemetry::trace::Provider::SetTracerProvider(provider);

 // set global propagator
  opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
      opentelemetry::nostd::shared_ptr<opentelemetry::context::propagation::TextMapPropagator>(
          new opentelemetry::trace::propagation::HttpTraceContext()));
}
```

### 새 스팬 생성 {#create-new-spans}

새로운 스팬은 `Tracer->StartSpan("spanName", attributes, options)`를 사용해
생성하고 시작할 수 있다. 스팬을 생성한 후에는 `Tracer->WithActiveSpan(span)`을
사용해 시작하고 활성 컨텍스트에 넣어야 한다. `Convert` 함수에서 이에 대한 예시를
확인할 수 있다.

```cpp
std::string span_name = "CurrencyService/Convert";
auto span =
    get_tracer("currency")->StartSpan(span_name,
                                  {{SemanticConventions::kRpcSystem, "grpc"},
                                   {SemanticConventions::kRpcService, "oteldemo.CurrencyService"},
                                   {SemanticConventions::kRpcMethod, "Convert"},
                                   {SemanticConventions::kRpcGrpcStatusCode, 0}},
                                  options);
auto scope = get_tracer("currency")->WithActiveSpan(span);
```

### 스팬에 속성 추가 {#adding-attributes-to-spans}

`Span->SetAttribute(key, value)`를 사용해 스팬에 속성을 추가할 수 있다.

```cpp
span->SetAttribute("app.currency.conversion.from", from_code);
span->SetAttribute("app.currency.conversion.to", to_code);
```

### 스팬 이벤트 추가 {#add-span-events}

스팬 이벤트를 추가하려면 `Span->AddEvent(name)`을 사용한다.

```cpp
span->AddEvent("Conversion successful, response sent back");
```

### 스팬 상태 설정 {#set-span-status}

스팬 상태를 그에 맞게 `Ok` 또는 `Error`로 설정해야 한다.
`Span->SetStatus(status)`를 사용해 설정할 수 있다.

```cpp
span->SetStatus(StatusCode::kOk);
```

### 트레이싱 컨텍스트 전파 {#tracing-context-propagation}

C++에서 전파(propagation)는 자동으로 처리되지 않는다. 호출자로부터 전파
컨텍스트를 추출하여 이후의 스팬에 주입해야 한다. `GrpcServerCarrier` 클래스는
인바운드 gRPC 요청에서 컨텍스트를 추출하는 메서드를 정의하며, 이는 서비스 호출
구현에서 활용된다.

`GrpcServerCarrier` 클래스는 `tracer_common.h`에 다음과 같이 정의되어 있다.

```cpp
class GrpcServerCarrier : public opentelemetry::context::propagation::TextMapCarrier
{
public:
  GrpcServerCarrier(ServerContext *context) : context_(context) {}
  GrpcServerCarrier() = default;
  virtual opentelemetry::nostd::string_view Get(
      opentelemetry::nostd::string_view key) const noexcept override
  {
    auto it = context_->client_metadata().find(key.data());
    if (it != context_->client_metadata().end())
    {
      return it->second.data();
    }
    return "";
  }

  virtual void Set(opentelemetry::nostd::string_view key,
                   opentelemetry::nostd::string_view value) noexcept override
  {
   // Not required for server
  }

  ServerContext *context_;
};
```

이 클래스는 `Convert` 메서드에서 컨텍스트를 추출하고, 새 스팬을 생성할 때 사용할
올바른 컨텍스트를 담기 위한 `StartSpanOptions` 객체를 생성하는 데 활용된다.

```cpp
StartSpanOptions options;
options.kind = SpanKind::kServer;
GrpcServerCarrier carrier(context);

auto prop        = context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();
auto current_ctx = context::RuntimeContext::GetCurrent();
auto new_context = prop->Extract(carrier, current_ctx);
options.parent   = GetSpan(new_context)->GetContext();
```

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

오픈텔레메트리 `MeterProvider`는 `meter_common.h`에 정의된 `initMeter()` 함수를
사용해 `main()`에서 초기화된다.

```cpp
void initMeter()
{
  // Build MetricExporter
  otlp_exporter::OtlpGrpcMetricExporterOptions otlpOptions;
  auto exporter = otlp_exporter::OtlpGrpcMetricExporterFactory::Create(otlpOptions);

  // Build MeterProvider and Reader
  metric_sdk::PeriodicExportingMetricReaderOptions options;
  std::unique_ptr<metric_sdk::MetricReader> reader{
      new metric_sdk::PeriodicExportingMetricReader(std::move(exporter), options) };
  auto provider = std::shared_ptr<metrics_api::MeterProvider>(new metric_sdk::MeterProvider());
  auto p = std::static_pointer_cast<metric_sdk::MeterProvider>(provider);
  p->AddMetricReader(std::move(reader));
  metrics_api::Provider::SetMeterProvider(provider);
}
```

### IntCounter 시작하기 {#starting-intcounter}

전역 `currency_counter` 변수는 `meter_common.h`에 정의된 `initIntCounter()`
함수를 호출하는 `main()`에서 생성된다.

```cpp
nostd::unique_ptr<metrics_api::Counter<uint64_t>> initIntCounter()
{
  std::string counter_name = name + "_counter";
  auto provider = metrics_api::Provider::GetMeterProvider();
  nostd::shared_ptr<metrics_api::Meter> meter = provider->GetMeter(name, version);
  auto int_counter = meter->CreateUInt64Counter(counter_name);
  return int_counter;
}
```

### 통화 변환 요청 카운트하기 {#counting-currency-conversion-requests}

`CurrencyCounter()` 메서드는 다음과 같이 구현되어 있다.

```cpp
void CurrencyCounter(const std::string& currency_code)
{
    std::map<std::string, std::string> labels = { {"currency_code", currency_code} };
    auto labelkv = common::KeyValueIterableView<decltype(labels)>{ labels };
    currency_counter->Add(1, labelkv);
}
```

`Convert()` 함수가 호출될 때마다, `to_code`로 받은 통화 코드를 사용해 변환
횟수를 센다.

```cpp
CurrencyCounter(to_code);
```

## 로그 {#logs}

오픈텔레메트리 `LoggerProvider`는 `logger_common.h`에 정의된 `initLogger()`
함수를 사용해 `main()`에서 초기화된다.

```cpp
void initLogger() {
  otlp::OtlpGrpcLogRecordExporterOptions loggerOptions;
  auto exporter  = otlp::OtlpGrpcLogRecordExporterFactory::Create(loggerOptions);
  auto processor = logs_sdk::SimpleLogRecordProcessorFactory::Create(std::move(exporter));
  std::vector<std::unique_ptr<logs_sdk::LogRecordProcessor>> processors;
  processors.push_back(std::move(processor));
  auto context = logs_sdk::LoggerContextFactory::Create(std::move(processors));
  std::shared_ptr<logs::LoggerProvider> provider = logs_sdk::LoggerProviderFactory::Create(std::move(context));
  opentelemetry::logs::Provider::SetLoggerProvider(provider);
}
```

### LoggerProvider 사용하기 {#using-the-loggerprovider}

초기화된 로거 프로바이더는 `server.cpp`의 `main`에서 호출된다.

```cpp
logger = getLogger(name);
```

이는 `logger`라는 지역 변수에 로거를 할당한다.

```cpp
nostd::shared_ptr<opentelemetry::logs::Logger> logger;
```

이후 코드 전반에서 로그를 남겨야 할 때마다 이 변수가 사용된다.

```cpp
logger->Info(std::string(__func__) + " conversion successful");
```
