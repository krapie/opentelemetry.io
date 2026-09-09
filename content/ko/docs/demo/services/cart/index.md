---
title: 장바구니 서비스
linkTitle: 장바구니
aliases: [cartservice]
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

이 서비스는 사용자가 장바구니에 담은 상품을 관리한다. 장바구니 데이터에 빠르게
접근하기 위해 Valkey 캐싱 서비스와 상호작용한다.

[장바구니 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/cart/)

> **참고**: .NET용 오픈텔레메트리는 트레이스와 메트릭에 대해 표준 오픈텔레메트리
> API 대신 `System.Diagnostic.DiagnosticSource` 라이브러리를 API로 사용한다.
> 로그에는 `Microsoft.Extensions.Logging.Abstractions` 라이브러리가 사용된다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

오픈텔레메트리는 .NET 의존성 주입(dependency injection) 컨테이너에서 구성된다.
`AddOpenTelemetry()` 빌더 메서드는 원하는 계측 라이브러리를 구성하고, 익스포터를
추가하고, 기타 옵션을 설정하는 데 사용된다. 익스포터와 리소스 속성의 구성은 환경
변수를 통해 수행된다.

```cs
Action<ResourceBuilder> appResourceBuilder =
    resource => resource
        .AddContainerDetector()
        .AddHostDetector();

builder.Services.AddOpenTelemetry()
    .ConfigureResource(appResourceBuilder)
    .WithTracing(tracerBuilder => tracerBuilder
        .AddSource("OpenTelemetry.Demo.Cart")
        .AddRedisInstrumentation(
            options => options.SetVerboseDatabaseStatements = true)
        .AddAspNetCoreInstrumentation()
        .AddGrpcClientInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬(activity)을 가져올
수 있다.

```cs
var activity = Activity.Current;
```

스팬(activity)에 속성(.NET에서는 태그(tag))을 추가하려면 activity 객체의
`SetTag`를 사용한다. `services/CartService.cs`의 `AddItem` 함수에서는 자동
계측된 스팬에 여러 속성이 추가된다.

```cs
activity?.SetTag("app.user.id", request.UserId);
activity?.SetTag("app.product.quantity", request.Item.Quantity);
activity?.SetTag("app.product.id", request.Item.ProductId);
```

### 스팬 이벤트 추가 {#add-span-events}

스팬(activity) 이벤트를 추가하려면 activity 객체의 `AddEvent`를 사용한다.
`services/CartService.cs`의 `GetCart` 함수에서는 스팬 이벤트가 추가된다.

```cs
activity?.AddEvent(new("Fetch cart"));
```

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

오픈텔레메트리 트레이스를 구성하는 것과 마찬가지로, .NET 의존성 주입
컨테이너에서도 `AddOpenTelemetry()` 호출이 필요하다. 이 빌더는 원하는 계측
라이브러리, 익스포터 등을 구성한다.

```cs
Action<ResourceBuilder> appResourceBuilder =
    resource => resource
        .AddContainerDetector()
        .AddHostDetector();

builder.Services.AddOpenTelemetry()
    .ConfigureResource(appResourceBuilder)
    .WithMetrics(meterBuilder => meterBuilder
        .AddMeter("OpenTelemetry.Demo.Cart")
        .AddProcessInstrumentation()
        .AddRuntimeInstrumentation()
        .AddAspNetCoreInstrumentation()
        .SetExemplarFilter(ExemplarFilterType.TraceBased)
        .AddOtlpExporter());
```

### 이그젬플러(Exemplars) {#exemplars}

[이그젬플러(Exemplar)](/docs/specs/otel/metrics/data-model/#exemplars)는
장바구니 서비스에서 트레이스 기반 이그젬플러 필터로 구성되어 있으며, 이를 통해
오픈텔레메트리 SDK가 메트릭에 이그젬플러를 첨부할 수 있다.

먼저 `CartActivitySource`, `Meter`, 그리고 두 개의 `Histogram`을 생성한다. 이
히스토그램은 `AddItem`과 `GetCart` 메서드의 지연 시간을 추적하는데, 이 두
메서드는 장바구니 서비스에서 중요한 메서드이기 때문이다.

이 두 메서드는 장바구니 서비스에 매우 중요한데, 사용자가 장바구니에 상품을
추가하거나 체크아웃 프로세스로 넘어가기 전에 장바구니를 확인할 때 너무 오래
기다려서는 안 되기 때문이다.

```cs
private static readonly ActivitySource CartActivitySource = new("OpenTelemetry.Demo.Cart");
private static readonly Meter CartMeter = new Meter("OpenTelemetry.Demo.Cart");
private static readonly Histogram<long> addItemHistogram = CartMeter.CreateHistogram<long>(
    "app.cart.add_item.latency",
    advice: new InstrumentAdvice<long>
    {
        HistogramBucketBoundaries = [ 500000, 600000, 700000, 800000, 900000, 1000000, 1100000 ]
    });
private static readonly Histogram<long> getCartHistogram = CartMeter.CreateHistogram<long>(
    "app.cart.get_cart.latency",
    advice: new InstrumentAdvice<long>
    {
        HistogramBucketBoundaries = [ 300000, 400000, 500000, 600000, 700000, 800000, 900000 ]
    });
```

장바구니 서비스의 결과가 마이크로초 단위이기 때문에 기본값이 맞지 않아 커스텀
버킷 경계(bucket boundary)도 정의되어 있다는 점에 유의한다.

변수가 정의되면, 각 메서드 실행의 지연 시간은 다음과 같이 `StopWatch`로
추적된다.

```cs
var stopwatch = Stopwatch.StartNew();

(method logic)

addItemHistogram.Record(stopwatch.ElapsedTicks);
```

이 모든 것을 연결하려면, 트레이스 파이프라인에 생성된 소스를 추가해야 한다. (위
스니펫에 이미 있지만 참고를 위해 여기에도 추가한다.)

```cs
.AddSource("OpenTelemetry.Demo.Cart")
```

그리고 메트릭 파이프라인에는 `Meter`와 `ExemplarFilter`를 추가한다.

```cs
.AddMeter("OpenTelemetry.Demo.Cart")
.SetExemplarFilter(ExemplarFilterType.TraceBased)
```

이그젬플러를 시각화하려면 Grafana <http://localhost:8080/grafana>로 이동한 다음
Dashboards > Demo > Cart Service Exemplars로 이동한다.

이그젬플러는 95번째 백분위수 차트에서는 특별한 "다이아몬드 모양의 점"으로,
히트맵 차트에서는 작은 사각형으로 나타난다. 이그젬플러를 선택하면 측정값의
타임스탬프, 원시 값, 그리고 기록 시점의 트레이스 컨텍스트를 포함한 데이터를
확인할 수 있다. `trace_id`를 통해 트레이싱 백엔드(이 경우 Jaeger)로 이동할 수
있다.

![장바구니 서비스 이그젬플러](exemplars.png)

## 로그 {#logs}

로그는 .NET 의존성 주입 컨테이너에서 `LoggingBuilder` 수준에서
`AddOpenTelemetry()`를 호출하여 구성된다. 이 빌더는 원하는 옵션, 익스포터 등을
구성한다.

```cs
builder.Logging
    .AddOpenTelemetry(options => options.AddOtlpExporter());
```
