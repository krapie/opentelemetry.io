---
title: 체크아웃 서비스
linkTitle: 체크아웃
aliases: [checkoutservice]
# prettier-ignore
cSpell:ignore: fatalf otelgrpc otelsarama otlpmetricgrpc otlptracegrpc sarama sdkmetric sdktrace
default_lang_commit: f9a0439ac56dba1515283e1a1cb6d6a90634a20f
---

이 서비스는 사용자의 체크아웃 주문을 처리하는 역할을 담당한다. 체크아웃 서비스는
주문을 처리하기 위해 여러 다른 서비스를 호출한다.

[체크아웃 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/checkout/)

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

오픈텔레메트리 SDK는 `initTracerProvider` 함수를 사용하여 `main`에서 초기화된다.

```go
func initTracerProvider() *sdktrace.TracerProvider {
    ctx := context.Background()

    exporter, err := otlptracegrpc.New(ctx)
    if err != nil {
        log.Fatal(err)
    }
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(initResource()),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
    return tp
}
```

서비스가 종료될 때 모든 스팬이 내보내지도록 보장하려면
`TracerProvider.Shutdown()`을 호출해야 한다. 이 서비스는 main의 지연(deferred)
함수의 일부로 이 호출을 수행한다.

```go
tp := initTracerProvider()
defer func() {
    if err := tp.Shutdown(context.Background()); err != nil {
        log.Printf("Error shutting down tracer provider: %v", err)
    }
}()
```

### gRPC 자동 계측 추가 {#adding-grpc-auto-instrumentation}

이 서비스는 gRPC 요청을 수신하며, 이는 gRPC 서버 생성의 일부로 main 함수에서
계측된다.

```go
var srv = grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

이 서비스는 여러 개의 나가는(outgoing) gRPC 호출을 발생시키며, 이들은 모두 gRPC
클라이언트를 계측으로 감싸는 방식으로 계측된다.

```go
func createClient(ctx context.Context, svcAddr string) (*grpc.ClientConn, error) {
    return grpc.DialContext(ctx, svcAddr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    )
}
```

### Kafka(Sarama) 자동 계측 추가 {#adding-kafka--sarama--auto-instrumentation}

이 서비스는 처리된 결과를 Kafka 토픽에 기록하며, 이는 다시 다른 마이크로서비스에
의해 처리된다. Kafka 클라이언트를 계측하려면 Producer가 생성된 이후에 이를
래핑해야 한다.

```go
saramaConfig := sarama.NewConfig()
producer, err := sarama.NewAsyncProducer(brokers, saramaConfig)
if err != nil {
    return nil, err
}
producer = otelsarama.WrapAsyncProducer(saramaConfig, producer)
```

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬을 가져올 수 있다.

```go
span := trace.SpanFromContext(ctx)
```

스팬에 속성을 추가하려면 스팬 객체의 `SetAttributes`를 사용한다. `PlaceOrder`
함수에서는 스팬에 여러 속성이 추가된다.

```go
span.SetAttributes(
    attribute.String("app.order.id", orderID.String()), shippingTrackingAttribute,
    attribute.Float64("app.shipping.amount", shippingCostFloat),
    attribute.Float64("app.order.amount", totalPriceFloat),
    attribute.Int("app.order.items.count", len(prep.orderItems)),
)
```

### 스팬 이벤트 추가 {#add-span-events}

스팬 이벤트를 추가하려면 스팬 객체의 `AddEvent`를 사용한다. `PlaceOrder`
함수에서는 여러 스팬 이벤트가 추가된다. 일부 이벤트는 추가 속성을 가지고 있고,
그렇지 않은 이벤트도 있다.

속성 없이 스팬 이벤트를 추가하는 경우:

```go
span.AddEvent("prepared")
```

추가 속성과 함께 스팬 이벤트를 추가하는 경우:

```go
span.AddEvent("charged",
    trace.WithAttributes(attribute.String("app.payment.transaction.id", txID)))
```

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

오픈텔레메트리 SDK는 `initMeterProvider` 함수를 사용하여 `main`에서 초기화된다.

```go
func initMeterProvider() *sdkmetric.MeterProvider {
    ctx := context.Background()

    exporter, err := otlpmetricgrpc.New(ctx)
    if err != nil {
        log.Fatalf("new otlp metric grpc exporter failed: %v", err)
    }

    mp := sdkmetric.NewMeterProvider(sdkmetric.WithReader(sdkmetric.NewPeriodicReader(exporter)))
    global.SetMeterProvider(mp)
    return mp
}
```

서비스가 종료될 때 모든 레코드가 내보내지도록 보장하려면
`MeterProvider.Shutdown()`을 호출해야 한다. 이 서비스는 main의 지연 함수의
일부로 이 호출을 수행한다.

```go
mp := initMeterProvider()
defer func() {
    if err := mp.Shutdown(context.Background()); err != nil {
        log.Printf("Error shutting down meter provider: %v", err)
    }
}()
```

### Golang 런타임 자동 계측 추가 {#adding-golang-runtime-auto-instrumentation}

Golang 런타임은 main 함수에서 계측된다.

```go
err := runtime.Start(runtime.WithMinimumReadMemStatsInterval(time.Second))
if err != nil {
    log.Fatal(err)
}
```

## 로그 {#logs}

로그를 오픈텔레메트리 컬렉터로 전송하는 방법은 두 가지가 있다.

- 컬렉터로 직접 전송
- 파일 또는 `stdout`을 통해 전송

이 두 가지 방식을 모두 사용하는 방법에 대한 문서는
[수동 계측](/docs/languages/go/instrumentation/) 문서의
[로그](/docs/languages/go/instrumentation/#logs) 섹션에서 확인할 수 있다.

체크아웃 서비스는 로그를 컬렉터로 직접 전송하며, 로그 브릿지(log bridge)를
사용하여 구조화된 로그를 출력하는 `slog` 로깅 패키지로 브릿징(bridging)하여
로그를 전송한다.

## LoggerProvider 초기화 {#loggerprovider-initialization}

오픈텔레메트리 SDK는 `initLoggerProvider` 함수를 사용하여 `main`에서 초기화된다.

```go
ctx := context.Background()

logExporter, err := otlploggrpc.New(ctx)
if err != nil {
	return nil
}

loggerProvider := sdklog.NewLoggerProvider(
	sdklog.WithProcessor(sdklog.NewBatchProcessor(logExporter)),
)
global.SetLoggerProvider(loggerProvider)

return loggerProvider
```

서비스가 종료될 때 모든 로그가 내보내지도록 보장하려면
`LoggerProvider.Shutdown()`을 호출한다. 이 서비스는 `main`의 지연 함수의 일부로
이 호출을 수행한다.

```go
lp := initLoggerProvider()
defer func() {
	if err := lp.Shutdown(context.Background()); err != nil {
		logger.Error(fmt.Sprintf("Logger Provider Shutdown: %v", err))
	}
	logger.Info("Shutdown logger provider")
}()
```

### 로깅 기능 {#logging-functionality}

이 서비스는 gRPC 호출을 사용하여 컬렉터로 로그를 전송한다. 로그는 `slog`
패키지를 사용하여 구조화된 형식으로 출력된다.

먼저 로거를 초기화한다.

```go
logger   *slog.Logger
logger = otelslog.NewLogger("checkout")
```

로거로 전송되기 전에 출력을 포맷하기 위해 `fmt.Sprintf`를 사용하는 것에
유의한다.

```go
logger.Info(fmt.Sprintf("order confirmation email sent to %q", req.Email))
logger.Warn(fmt.Sprintf("failed to send order confirmation to %q: %+v", req.Email, err))
logger.Error(fmt.Sprintf("Error shutting down logger provider: %v", err))
```

`slog`를 사용하는 것의 장점은 출력에 추가 속성을 첨부할 수 있다는 점이다. 다음
예시는 `orderID`, `shippingCost`, `totalPrice`와 같은 몇 가지 속성을 첨부한다.
이를 통해 로그 출력의 일부로 이러한 값을 확인하고 파싱할 수 있으며, Grafana에서
이를 별도의 열(column)로 더 쉽게 확인할 수 있다.

```go
logger.LogAttrs(
    ctx,
    slog.LevelInfo, "order placed",
    slog.String("app.order.id", orderID.String()),
    slog.Float64("app.shipping.amount", shippingCostFloat),
    slog.Float64("app.order.amount", totalPriceFloat),
    slog.Int("app.order.items.count", len(prep.orderItems)),
    slog.String("app.shipping.tracking.id", shippingTrackingID),
)
```
