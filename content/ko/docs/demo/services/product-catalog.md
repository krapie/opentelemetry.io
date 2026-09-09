---
title: 상품 카탈로그 서비스
linkTitle: 상품 카탈로그
aliases: [productcatalogservice]
# prettier-ignore
cSpell:ignore: fatalf otelcodes otelgrpc otlpmetricgrpc otlptracegrpc sdkmetric sdktrace sprintf
default_lang_commit: 6588437286e916c2eb44a721161ce46c21f1706b
---

이 서비스는 상품에 대한 정보를 반환하는 역할을 담당한다. 이 서비스는 모든 상품을
조회하거나, 특정 상품을 검색하거나, 단일 상품에 대한 세부 정보를 반환하는 데
사용할 수 있다.

[상품 카탈로그 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/product-catalog/)

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

오픈텔레메트리 SDK는 `initTracerProvider` 함수를 사용하여 `main`에서 초기화된다.

```go
func initTracerProvider() *sdktrace.TracerProvider {
    ctx := context.Background()

    exporter, err := otlptracegrpc.New(ctx)
    if err != nil {
        log.Fatalf("OTLP Trace gRPC Creation: %v", err)
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
tp := InitTracerProvider()
defer func() {
    if err := tp.Shutdown(context.Background()); err != nil {
        log.Fatalf("Tracer Provider Shutdown: %v", err)
    }
}()
```

### gRPC 자동 계측 추가 {#adding-grpc-auto-instrumentation}

이 서비스는 gRPC 요청을 수신하며, 이는 gRPC 서버 생성의 일부로 main 함수에서
계측된다.

```go
srv := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

이 서비스는 나가는(outgoing) gRPC 호출을 발생시키며, 이들은 모두 gRPC
클라이언트를 계측으로 감싸는 방식으로 계측된다.

```go
func createClient(ctx context.Context, svcAddr string) (*grpc.ClientConn, error) {
    return grpc.DialContext(ctx, svcAddr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    )
}
```

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬을 가져올 수 있다.

```go
span := trace.SpanFromContext(ctx)
```

스팬에 속성을 추가하려면 스팬 객체의 `SetAttributes`를 사용한다. `GetProduct`
함수에서는 상품 ID에 대한 속성이 스팬에 추가된다.

```go
span.SetAttributes(
    attribute.String("app.product.id", req.Id),
)
```

### 스팬 상태 설정 {#setting-span-status}

이 서비스는 기능 플래그를 기반으로 오류 상태를 포착하고 처리할 수 있다. 오류
상태가 발생하면 스팬 객체의 `SetStatus`를 사용하여 그에 맞게 스팬 상태가
설정된다. 이는 `GetProduct` 함수에서 확인할 수 있다.

```go
msg := fmt.Sprintf("Error: ProductCatalogService Fail Feature Flag Enabled")
span.SetStatus(otelcodes.Error, msg)
```

### 스팬 이벤트 추가 {#add-span-events}

스팬 이벤트를 추가하려면 스팬 객체의 `AddEvent`를 사용한다. `GetProduct`
함수에서는 오류 상태가 처리되었을 때, 또는 상품을 성공적으로 찾았을 때 스팬
이벤트가 추가된다.

```go
span.AddEvent(msg)
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
`initMeterProvider.Shutdown()`을 호출해야 한다. 이 서비스는 main의 지연 함수의
일부로 이 호출을 수행한다.

```go
mp := initMeterProvider()
defer func() {
    if err := mp.Shutdown(context.Background()); err != nil {
        log.Fatalf("Error shutting down meter provider: %v", err)
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

상품 카탈로그 서비스는 로그를 컬렉터로 직접 전송하며, 로그 브릿지(log bridge)를
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
logger = otelslog.NewLogger("product-catalog")
```

로거로 전송되기 전에 출력을 포맷하기 위해 `fmt.Sprintf`를 사용하는 것에
유의한다.

```go
logger.Info("Loading Product Catalog...")
logger.Info(fmt.Sprintf("Product Catalog reload interval: %d", interval))
logger.Error(fmt.Sprintf("Error shutting down meter provider: %v", err))
```

`slog`를 사용하는 것의 장점은 출력에 추가 속성을 첨부할 수 있다는 점이다. 다음
예시는 `product.name` 및 `product.id` 속성을 첨부한다. 이를 통해 로그 출력의
일부로 이러한 값을 확인하고 파싱할 수 있으며, Grafana에서 이를 별도의
열(column)로 더 쉽게 확인할 수 있다.

```go
logger.LogAttrs(
	ctx,
	slog.LevelInfo, "Product Found",
	slog.String("app.product.name", found.Name),
	slog.String("app.product.id", req.Id),
)
```
