---
title: 계측
aliases:
  - manual
  - manual_instrumentation
weight: 30
description: 오픈텔레메트리(OpenTelemetry) Go를 위한 수동 계측
cSpell:ignore: fatalf logr logrus otlplog otlploghttp sdktrace sighup
default_lang_commit: 3560c03d5cbe845c6189e6e30441434c7760eca0
---

{{% include instrumentation-intro.md %}}

## 설정 {#setup}

## 트레이스 {#traces}

### 트레이서 얻기 {#getting-a-tracer}

스팬을 생성하려면, 먼저 트레이서를 얻거나 초기화해야 한다.

알맞은 패키지가 설치되어 있는지 확인한다.

```sh
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/trace \
  go.opentelemetry.io/otel/sdk \
```

그런 다음 익스포터, 리소스, 트레이서 프로바이더를 초기화하고, 마지막으로
트레이서를 초기화한다.

```go
package app

import (
	"context"
	"fmt"
	"log"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"
	"go.opentelemetry.io/otel/trace"
)

var tracer trace.Tracer

func newExporter(ctx context.Context)  /* (someExporter.Exporter, error) */ {
	// Your preferred exporter: console, jaeger, zipkin, OTLP, etc.
}

func newTracerProvider(exp sdktrace.SpanExporter) *sdktrace.TracerProvider {
	// Ensure default SDK resources and the required service name are set.
	r, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName("ExampleService"),
		),
	)

	if err != nil {
		panic(err)
	}

	return sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(r),
	)
}

func main() {
	ctx := context.Background()

	exp, err := newExporter(ctx)
	if err != nil {
		log.Fatalf("failed to initialize exporter: %v", err)
	}

	// Create a new tracer provider with a batch span processor and the given exporter.
	tp := newTracerProvider(exp)

	// Handle shutdown properly so nothing leaks.
	defer func() { _ = tp.Shutdown(ctx) }()

	otel.SetTracerProvider(tp)

	// Finally, set the tracer that can be used for this package.
	tracer = tp.Tracer("example.io/package/name")
}
```

이제 `tracer`에 접근하여 코드를 수동으로 계측할 수 있다.

> [!WARNING]
>
> [OBI](/docs/zero-code/obi)와 같은 eBPF 기반
> [Go 제로 코드 계측](/docs/zero-code/go)과 함께 수동 스팬을 추가하는 경우, 전역
> 트레이서 프로바이더를 설정하지 않는다. 자세한 내용은
> [Auto SDK](/docs/zero-code/go/autosdk) 문서를 참고한다.

### 스팬 생성하기 {#creating-spans}

스팬은 트레이서에 의해 생성된다. 초기화된 트레이서가 없다면, 먼저 초기화해야
한다.

트레이서로 스팬을 생성하려면, `context.Context` 인스턴스에 대한 핸들도 필요하다.
이는 일반적으로 요청 객체와 같은 것에서 얻게 되며, 이미 [계측
라이브러리][instrumentation library]로부터 부모 스팬을 포함하고 있을 수도 있다.

```go
func httpHandler(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "hello-span")
	defer span.End()

	// do some work to track with hello-span
}
```

Go에서는 `context` 패키지를 사용해 활성 스팬을 저장한다. 스팬을 시작하면, 생성된
스팬뿐 아니라 그 스팬을 포함하는 수정된 컨텍스트에 대한 핸들도 얻게 된다.

스팬이 완료되면 불변(immutable) 상태가 되며 더 이상 수정할 수 없다.

### 현재 스팬 얻기 {#get-the-current-span}

현재 스팬을 얻으려면, 핸들을 가지고 있는 `context.Context`에서 꺼내야 한다.

```go
// This context needs contain the active span you plan to extract.
ctx := context.TODO()
span := trace.SpanFromContext(ctx)

// Do something with the current span, optionally calling `span.End()` if you want it to end
```

이는 특정 시점에 현재 스팬에 정보를 추가하고 싶을 때 유용할 수 있다.

### 중첩된 스팬 생성하기 {#create-nested-spans}

중첩된 작업 내의 작업을 추적하기 위해 중첩된 스팬을 생성할 수 있다.

핸들을 가지고 있는 현재 `context.Context`가 이미 그 안에 스팬을 포함하고 있다면,
새 스팬을 생성하는 것은 중첩된 스팬을 만드는 것이 된다. 예를 들면:

```go
func parentFunction(ctx context.Context) {
	ctx, parentSpan := tracer.Start(ctx, "parent")
	defer parentSpan.End()

	// call the child function and start a nested span in there
	childFunction(ctx)

	// do more work - when this function ends, parentSpan will complete.
}

func childFunction(ctx context.Context) {
	// Create a span to track `childFunction()` - this is a nested span whose parent is `parentSpan`
	ctx, childSpan := tracer.Start(ctx, "child")
	defer childSpan.End()

	// do work here, when this function returns, childSpan will complete.
}
```

스팬이 완료되면 불변 상태가 되며 더 이상 수정할 수 없다.

### 스팬 속성 {#span-attributes}

속성(Attribute)은 스팬에 메타데이터로 적용되는 키와 값이며, 트레이스를 집계하고
필터링하고 그룹화하는 데 유용하다. 속성은 스팬 생성 시점에 추가할 수도 있고,
스팬이 완료되기 전이라면 생애주기 중 다른 어느 시점에든 추가할 수도 있다.

```go
// setting attributes at creation...
ctx, span = tracer.Start(ctx, "attributesAtCreation", trace.WithAttributes(attribute.String("hello", "world")))
// ... and after creation
span.SetAttributes(attribute.Bool("isTrue", true), attribute.String("stringAttr", "hi!"))
```

속성 키는 미리 계산해 둘 수도 있다.

```go
var myKey = attribute.Key("myCoolAttribute")
span.SetAttributes(myKey.String("a value"))
```

#### 시맨틱 속성 {#semantic-attributes}

시맨틱 속성(Semantic Attributes)은 HTTP 메서드, 상태 코드, 사용자 에이전트 등과
같은 공통 개념에 대해 여러 언어, 프레임워크, 런타임에 걸쳐 공유되는 속성 키
집합을 제공하기 위해 [오픈텔레메트리 명세][OpenTelemetry Specification]가
정의하는 속성이다. 이러한 속성은 `go.opentelemetry.io/otel/semconv/v1.43.0`
패키지에서 사용할 수 있다.

자세한 내용은 [트레이스 시맨틱 컨벤션][Trace semantic conventions]을 참고한다.

### 이벤트 {#events}

이벤트(event)는 스팬의 생애주기 동안 "무언가 일어났음"을 나타내는, 사람이 읽을
수 있는 메시지이다. 예를 들어 뮤텍스(mutex)로 보호되는 리소스에 대한 독점적
접근이 필요한 함수가 있다고 하자. 이벤트는 두 시점에 생성될 수 있다. 하나는
리소스에 대한 접근을 시도할 때이고, 다른 하나는 뮤텍스를 획득할 때이다.

```go
span.AddEvent("Acquiring lock")
mutex.Lock()
span.AddEvent("Got lock, doing work...")
// do stuff
span.AddEvent("Unlocking")
mutex.Unlock()
```

이벤트의 유용한 특성 중 하나는 타임스탬프가 스팬 시작 시점으로부터의 오프셋으로
표시되어, 이벤트 사이에 얼마나 시간이 경과했는지 쉽게 확인할 수 있다는 점이다.

이벤트도 자체적인 속성을 가질 수 있다.

```go
span.AddEvent("Cancelled wait due to external signal", trace.WithAttributes(attribute.Int("pid", 4328), attribute.String("signal", "SIGHUP")))
```

### 스팬 상태 설정하기 {#set-span-status}

{{% include "span-status-preamble.md" %}}

```go
import (
	// ...
	"go.opentelemetry.io/otel/codes"
	// ...
)

// ...

result, err := operationThatCouldFail()
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
}
```

### 에러 기록하기 {#record-errors}

실패한 작업이 있고 그로 인해 발생한 에러를 캡처하고 싶다면, 그 에러를 기록할 수
있다.

```go
import (
	// ...
	"go.opentelemetry.io/otel/codes"
	// ...
)

// ...

result, err := operationThatCouldFail()
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
	span.RecordError(err)
}
```

`RecordError`를 사용할 때는, 실패한 작업을 추적하는 스팬을 에러 스팬으로
간주하고 싶지 않은 경우가 아니라면 스팬의 상태도 `Error`로 설정할 것을 강력히
권장한다. `RecordError` 함수는 호출될 때 스팬 상태를 자동으로 설정하지
**않는다**.

### 전파자와 컨텍스트 {#propagators-and-context}

트레이스는 단일 프로세스를 넘어 확장될 수 있다. 이때 필요한 것이 _컨텍스트
전파(context propagation)_, 즉 트레이스에 대한 식별자를 원격 프로세스로 전송하는
메커니즘이다.

네트워크를 통해 트레이스 컨텍스트를 전파하려면, 오픈텔레메트리 API에
전파자(propagator)를 등록해야 한다.

```go
import (
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/propagation"
)
...
otel.SetTextMapPropagator(propagation.TraceContext{})
```

> 오픈텔레메트리는 W3C TraceContext 표준을 지원하지 않는 기존 트레이싱
> 시스템(`go.opentelemetry.io/contrib/propagators/b3`)과의 호환성을 위해 B3 헤더
> 형식도 지원한다.

컨텍스트 전파를 구성한 후에는, 컨텍스트를 실제로 직렬화하는 뒷단의 작업을
처리하기 위해 자동 계측을 사용하고 싶을 것이다.

## 메트릭 {#metrics}

[메트릭](/docs/concepts/signals/metrics)을 생성하기 시작하려면, `Meter`를 생성할
수 있게 해주는 초기화된 `MeterProvider`가 있어야 한다. 미터(Meter)를 사용하면
다양한 종류의 메트릭을 만드는 데 사용할 수 있는 계측기를 생성할 수 있다.
오픈텔레메트리 Go는 현재 다음 계측기를 지원한다.

- Counter(카운터): 음수가 아닌 증가를 지원하는 동기 계측기이다
- Asynchronous Counter(비동기 카운터): 음수가 아닌 증가를 지원하는 비동기
  계측기이다
- Histogram(히스토그램): 히스토그램, 요약(summary), 백분위수(percentile)와 같이
  통계적으로 의미 있는 임의의 값을 지원하는 동기 계측기이다
- Synchronous Gauge(동기 게이지): 실내 온도와 같이 가산적이지 않은 값을 지원하는
  동기 계측기이다
- Asynchronous Gauge(비동기 게이지): 실내 온도와 같이 가산적이지 않은 값을
  지원하는 비동기 계측기이다
- UpDownCounter(업다운카운터): 활성 요청 수와 같이 증가와 감소를 지원하는 동기
  계측기이다
- Asynchronous UpDownCounter(비동기 업다운카운터): 증가와 감소를 지원하는 비동기
  계측기이다

동기 및 비동기 계측기에 대해 더 알아보고, 어떤 종류가 사용 사례에 가장 적합한지
알아보려면
[보충 가이드라인](/docs/specs/otel/metrics/supplementary-guidelines/)을
참고한다.

계측 라이브러리나 수동으로 `MeterProvider`가 생성되지 않으면, 오픈텔레메트리
메트릭 API는 무동작(no-op) 구현체를 사용하며 데이터 생성에 실패한다.

다음에서 더 자세한 패키지 문서를 확인할 수 있다.

- 메트릭 API: [`go.opentelemetry.io/otel/metric`][]
- 메트릭 SDK: [`go.opentelemetry.io/otel/sdk/metric`][]

### 메트릭 초기화하기 {#initialize-metrics}

> [!NB] 라이브러리를 계측하는 경우, **이 단계를 건너뛴다**.

앱에서 [메트릭](/docs/concepts/signals/metrics/)을 활성화하려면,
[`Meter`](/docs/concepts/signals/metrics/#meter)를 생성할 수 있게 해주는
초기화된 [`MeterProvider`](/docs/concepts/signals/metrics/#meter-provider)가
있어야 한다.

`MeterProvider`가 생성되지 않으면, 메트릭을 위한 오픈텔레메트리 API는 무동작
구현체를 사용하며 데이터 생성에 실패한다. 따라서 다음 패키지를 사용해 SDK 초기화
코드를 포함하도록 소스 코드를 수정해야 한다.

- [`go.opentelemetry.io/otel`][]
- [`go.opentelemetry.io/otel/sdk/metric`][]
- [`go.opentelemetry.io/otel/sdk/resource`][]
- [`go.opentelemetry.io/otel/exporters/stdout/stdoutmetric`][]

알맞은 Go 모듈이 설치되어 있는지 확인한다.

```sh
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/exporters/stdout/stdoutmetric \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/sdk/metric
```

그런 다음 리소스, 메트릭 익스포터, 메트릭 프로바이더를 초기화한다.

```go
package main

import (
	"context"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutmetric"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"
)

func main() {
	// Create resource.
	res, err := newResource()
	if err != nil {
		panic(err)
	}

	// Create a meter provider.
	// You can pass this instance directly to your instrumented code if it
	// accepts a MeterProvider instance.
	meterProvider, err := newMeterProvider(res)
	if err != nil {
		panic(err)
	}

	// Handle shutdown properly so nothing leaks.
	defer func() {
		if err := meterProvider.Shutdown(context.Background()); err != nil {
			log.Println(err)
		}
	}()

	// Register as global meter provider so that it can be used via otel.Meter
	// and accessed using otel.GetMeterProvider.
	// Most instrumentation libraries use the global meter provider as default.
	// If the global meter provider is not set then a no-op implementation
	// is used, which fails to generate data.
	otel.SetMeterProvider(meterProvider)
}

func newResource() (*resource.Resource, error) {
	return resource.Merge(
    resource.Default(),
		resource.NewWithAttributes(
      semconv.SchemaURL,
			semconv.ServiceName("my-service"),
			semconv.ServiceVersion("0.1.0"),
		),
  )
}

func newMeterProvider(res *resource.Resource) (*metric.MeterProvider, error) {
	metricExporter, err := stdoutmetric.New()
	if err != nil {
		return nil, err
	}

	meterProvider := metric.NewMeterProvider(
		metric.WithResource(res),
		metric.WithReader(metric.NewPeriodicReader(metricExporter,
			// Default is 1m. Set to 3s for demonstrative purposes.
			metric.WithInterval(3*time.Second))),
	)
	return meterProvider, nil
}
```

`MeterProvider`가 구성되었으니, 이제 `Meter`를 얻을 수 있다.

### 미터 얻기 {#acquiring-a-meter}

애플리케이션에서 수동으로 계측한 코드가 있는 곳이라면 어디서든
[`otel.Meter`](https://pkg.go.dev/go.opentelemetry.io/otel#Meter)를 호출하여
미터를 얻을 수 있다. 예를 들면:

```go
import "go.opentelemetry.io/otel"

var meter = otel.Meter("example.io/package/name")
```

### 동기 계측기와 비동기 계측기 {#synchronous-and-asynchronous-instruments}

오픈텔레메트리 계측기는 동기이거나 비동기(관측 가능한, observable)이다.

동기 계측기는 호출될 때 측정값을 얻는다. 이 측정은 다른 함수 호출과 마찬가지로,
프로그램 실행 중 하나의 호출로 이루어진다. 이러한 측정값의 집계는 구성된
익스포터에 의해 주기적으로 내보내진다. 측정은 값을 내보내는 것과 분리되어 있기
때문에, 하나의 내보내기 주기에는 집계된 측정값이 0개 또는 여러 개 포함될 수
있다.

반면 비동기 계측기는 SDK의 요청에 따라 측정값을 제공한다. SDK가 내보낼 때,
계측기 생성 시 제공된 콜백이 호출된다. 이 콜백은 즉시 내보내지는 측정값을 SDK에
제공한다. 비동기 계측기에 대한 모든 측정은 내보내기 주기당 한 번씩 수행된다.

비동기 계측기는 다음과 같은 여러 상황에서 유용하다.

- 카운터를 갱신하는 것이 계산적으로 저렴하지 않아서, 현재 실행 중인 스레드가
  측정을 기다리게 하고 싶지 않을 때
- 관측이 프로그램 실행과 무관한 주기로 이루어져야 할 때(즉, 요청 생애주기에
  묶여서는 정확하게 측정할 수 없을 때)
- 측정값에 대해 알려진 타임스탬프가 없을 때

이런 경우에는, 후처리에서 일련의 델타값을 집계하는 방식(동기 방식의 예)보다
누적값을 직접 관측하는 편이 더 나은 경우가 많다.

### 동기 계측기의 성능 최적화 {#performance-optimization-for-synchronous-instruments}

동기 계측기의 경우, 속성이나 값을 계산하는 비용이 큰 연산을 수행하기 전에
`Enabled` 메서드를 사용해 계측기가 활성화되어 있는지 확인할 수 있다.

```go
if apiCounter.Enabled(ctx) {
    // compute attributes or values
    apiCounter.Add(ctx, 1, metric.WithAttributes(attributes...))
}
```

이렇게 하면 `MeterProvider`가 구성되지 않았거나 메트릭을 드롭하도록 뷰가 구성된
경우에 계측이 성능상의 불이익을 겪지 않도록 보장할 수 있다.

### 카운터 사용하기 {#using-counters}

카운터는 음수가 아닌 증가하는 값을 측정하는 데 사용할 수 있다.

예를 들어 HTTP 핸들러에 대한 호출 수를 다음과 같이 보고할 수 있다.

```go
import (
	"net/http"

	"go.opentelemetry.io/otel/metric"
)

func init() {
	apiCounter, err := meter.Int64Counter(
		"api.counter",
		metric.WithDescription("Number of API calls."),
		metric.WithUnit("{call}"),
	)
	if err != nil {
		panic(err)
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if apiCounter.Enabled(r.Context()) {
			apiCounter.Add(r.Context(), 1)
		}

		// do some work in an API call
	})
}
```

### UpDown 카운터 사용하기 {#using-updown-counters}

UpDown 카운터는 증가와 감소가 가능하여, 오르내리는 누적값을 관측할 수 있게
해준다.

예를 들어 어떤 컬렉션의 항목 수를 다음과 같이 보고할 수 있다.

```go
import (
	"context"

	"go.opentelemetry.io/otel/metric"
)

var itemsCounter metric.Int64UpDownCounter

func init() {
	var err error
	itemsCounter, err = meter.Int64UpDownCounter(
		"items.counter",
		metric.WithDescription("Number of items."),
		metric.WithUnit("{item}"),
	)
	if err != nil {
		panic(err)
	}
}

func addItem() {
	// code that adds an item to the collection

	ctx := context.Background()
	if itemsCounter.Enabled(ctx) {
		itemsCounter.Add(ctx, 1)
	}
}

func removeItem() {
	// code that removes an item from the collection

	ctx := context.Background()
	if itemsCounter.Enabled(ctx) {
		itemsCounter.Add(ctx, -1)
	}
}
```

### 게이지 사용하기 {#using-gauges}

게이지는 변경이 발생할 때 가산적이지 않은 값을 측정하는 데 사용된다.

예를 들어 CPU 팬의 현재 속도를 다음과 같이 보고할 수 있다.

```go
import (
	"net/http"

	"go.opentelemetry.io/otel/metric"
)

var (
  fanSpeedSubscription chan int64
  speedGauge metric.Int64Gauge
)

func init() {
	var err error
	speedGauge, err = meter.Int64Gauge(
		"cpu.fan.speed",
		metric.WithDescription("Speed of CPU fan"),
		metric.WithUnit("RPM"),
	)
	if err != nil {
		panic(err)
	}

	getCPUFanSpeed := func() int64 {
		// Generates a random fan speed for demonstration purpose.
		// In real world applications, replace this to get the actual fan speed.
		return int64(1500 + rand.Intn(1000))
	}

	fanSpeedSubscription = make(chan int64, 1)
	go func() {
		defer close(fanSpeedSubscription)

		for idx := 0; idx < 5; idx++ {
			// Synchronous gauges are used when the measurement cycle is
			// synchronous to an external change.
			time.Sleep(time.Duration(rand.Intn(3)) * time.Second)
			fanSpeed := getCPUFanSpeed()
			fanSpeedSubscription <- fanSpeed
		}
	}()
}

func recordFanSpeed() {
	ctx := context.Background()
	for fanSpeed := range fanSpeedSubscription {
		if speedGauge.Enabled(ctx) {
			speedGauge.Record(ctx, fanSpeed)
		}
	}
}
```

### 히스토그램 사용하기 {#using-histograms}

히스토그램은 시간에 따른 값의 분포를 측정하는 데 사용된다.

예를 들어 HTTP 핸들러의 응답 시간 분포를 다음과 같이 보고할 수 있다.

```go
import (
	"net/http"
	"time"

	"go.opentelemetry.io/otel/metric"
)

func init() {
	histogram, err := meter.Float64Histogram(
		"task.duration",
		metric.WithDescription("The duration of task execution."),
		metric.WithUnit("s"),
	)
	if err != nil {
		panic(err)
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// do some work in an API call

		duration := time.Since(start)
		if histogram.Enabled(r.Context()) {
			histogram.Record(r.Context(), duration.Seconds())
		}
	})
}
```

### 관측 가능한(비동기) 카운터 사용하기 {#using-observable-async-counters}

관측 가능한 카운터는 가산적이고 음수가 아니며 단조 증가하는 값을 측정하는 데
사용할 수 있다.

예를 들어 애플리케이션이 시작된 이후 경과한 시간을 다음과 같이 보고할 수 있다.

```go
import (
	"context"
	"time"

	"go.opentelemetry.io/otel/metric"
)

func init() {
	start := time.Now()
	if _, err := meter.Float64ObservableCounter(
		"uptime",
		metric.WithDescription("The duration since the application started."),
		metric.WithUnit("s"),
		metric.WithFloat64Callback(func(_ context.Context, o metric.Float64Observer) error {
			o.Observe(float64(time.Since(start).Seconds()))
			return nil
		}),
	); err != nil {
		panic(err)
	}
}
```

### 관측 가능한(비동기) UpDown 카운터 사용하기 {#using-observable-async-updown-counters}

관측 가능한 UpDown 카운터는 증가와 감소가 가능하여, 가산적이고 음수가 아니며
비단조적으로 증가하는 누적값을 측정할 수 있게 해준다.

예를 들어 일부 데이터베이스 메트릭을 다음과 같이 보고할 수 있다.

```go
import (
	"context"
	"database/sql"

	"go.opentelemetry.io/otel/metric"
)

// registerDBMetrics registers asynchronous metrics for the provided db.
// Make sure to unregister metric.Registration before closing the provided db.
func registerDBMetrics(db *sql.DB, meter metric.Meter, poolName string) (metric.Registration, error) {
	max, err := meter.Int64ObservableUpDownCounter(
		"db.client.connections.max",
		metric.WithDescription("The maximum number of open connections allowed."),
		metric.WithUnit("{connection}"),
	)
	if err != nil {
		return nil, err
	}

	waitTime, err := meter.Int64ObservableUpDownCounter(
		"db.client.connections.wait_time",
		metric.WithDescription("The time it took to obtain an open connection from the pool."),
		metric.WithUnit("ms"),
	)
	if err != nil {
		return nil, err
	}

	reg, err := meter.RegisterCallback(
		func(_ context.Context, o metric.Observer) error {
			stats := db.Stats()
			o.ObserveInt64(max, int64(stats.MaxOpenConnections))
			o.ObserveInt64(waitTime, int64(stats.WaitDuration))
			return nil
		},
		max,
		waitTime,
	)
	if err != nil {
		return nil, err
	}
	return reg, nil
}
```

### 관측 가능한(비동기) 게이지 사용하기 {#using-observable-async-gauges}

관측 가능한 게이지는 가산적이지 않은 값을 측정하는 데 사용해야 한다.

예를 들어 애플리케이션에서 사용하는 힙 객체의 메모리 사용량을 다음과 같이 보고할
수 있다.

```go
import (
	"context"
	"runtime"

	"go.opentelemetry.io/otel/metric"
)

func init() {
	if _, err := meter.Int64ObservableGauge(
		"memory.heap",
		metric.WithDescription(
			"Memory usage of the allocated heap objects.",
		),
		metric.WithUnit("By"),
		metric.WithInt64Callback(func(_ context.Context, o metric.Int64Observer) error {
			var m runtime.MemStats
			runtime.ReadMemStats(&m)
			o.Observe(int64(m.HeapAlloc))
			return nil
		}),
	); err != nil {
		panic(err)
	}
}
```

### 속성 추가하기 {#adding-attributes}

[`WithAttributeSet`](https://pkg.go.dev/go.opentelemetry.io/otel/metric#WithAttributeSet)
또는
[`WithAttributes`](https://pkg.go.dev/go.opentelemetry.io/otel/metric#WithAttributes)
옵션을 사용해 속성을 추가할 수 있다.

```go
import (
	"net/http"

	"go.opentelemetry.io/otel/metric"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"
)

func init() {
	apiCounter, err := meter.Int64UpDownCounter(
		"api.finished.counter",
		metric.WithDescription("Number of finished API calls."),
		metric.WithUnit("{call}"),
	)
	if err != nil {
		panic(err)
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// do some work in an API call and set the response HTTP status code

		apiCounter.Add(r.Context(), 1,
			metric.WithAttributes(semconv.HTTPResponseStatusCode(statusCode)))
	})
}
```

### 뷰 등록하기 {#registering-views}

뷰(view)는 SDK 사용자에게 SDK가 출력하는 메트릭을 커스터마이징할 수 있는
유연성을 제공한다. 어떤 메트릭 계측기를 처리하거나 무시할지 커스터마이징할 수
있다. 또한 집계 방식과 메트릭에 보고하고 싶은 속성도 커스터마이징할 수 있다.

모든 계측기에는 원래 이름, 설명, 속성을 유지하며 계측기 종류에 기반한 기본
집계를 갖는 기본 뷰가 있다. 등록된 뷰가 계측기와 일치하면, 기본 뷰는 등록된 뷰로
대체된다. 계측기와 일치하는 추가로 등록된 뷰는 누적되며, 그 계측기에 대해 여러
개의 내보내진 메트릭이 생성된다.

[`NewView`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric#NewView)
함수를 사용해 뷰를 생성하고,
[`WithView`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric#WithView)
옵션을 사용해 등록할 수 있다.

예를 들어 `http` 계측 라이브러리의 `v0.34.0` 버전에 있는 `latency` 계측기를
`request.latency`로 이름을 바꾸는 뷰는 다음과 같이 생성한다.

```go
view := metric.NewView(metric.Instrument{
	Name: "latency",
	Scope: instrumentation.Scope{
		Name:    "http",
		Version: "0.34.0",
	},
}, metric.Stream{Name: "request.latency"})

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

예를 들어 `http` 계측 라이브러리의 `latency` 계측기가 지수 히스토그램으로
보고되도록 만드는 뷰는 다음과 같이 생성한다.

```go
view := metric.NewView(
	metric.Instrument{
		Name:  "latency",
		Scope: instrumentation.Scope{Name: "http"},
	},
	metric.Stream{
		Aggregation: metric.AggregationBase2ExponentialHistogram{
			MaxSize:  160,
			MaxScale: 20,
		},
	},
)

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

SDK는 메트릭을 내보내기 전에 메트릭과 속성을 필터링한다. 예를 들어 뷰를 사용해
카디널리티가 높은 메트릭의 메모리 사용량을 줄이거나, 민감한 데이터를 포함할 수
있는 속성을 드롭할 수 있다.

`http` 계측 라이브러리의 `latency` 계측기를 드롭하는 뷰는 다음과 같이 생성한다.

```go
view := metric.NewView(
  metric.Instrument{
    Name:  "latency",
    Scope: instrumentation.Scope{Name: "http"},
  },
  metric.Stream{Aggregation: metric.AggregationDrop{}},
)

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

`http` 계측 라이브러리의 `latency` 계측기가 기록하는 `http.request.method`
속성을 제거하는 뷰는 다음과 같이 생성한다.

```go
view := metric.NewView(
  metric.Instrument{
    Name:  "latency",
    Scope: instrumentation.Scope{Name: "http"},
  },
  metric.Stream{AttributeFilter: attribute.NewDenyKeysFilter("http.request.method")},
)

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

기준의 `Name` 필드는 와일드카드 패턴 매칭을 지원한다. `*` 와일드카드는 0개
이상의 문자와 일치하는 것으로 인식되고, `?`는 정확히 한 문자와 일치하는 것으로
인식된다. 예를 들어 `*` 패턴은 모든 계측기 이름과 일치한다.

다음 예제는 이름이 `.ms`로 끝나는 모든 계측기에 대해 단위를 밀리초로 설정하는
뷰를 생성하는 방법을 보여준다.

```go
view := metric.NewView(
  metric.Instrument{Name: "*.ms"},
  metric.Stream{Unit: "ms"},
)

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

`NewView` 함수는 뷰를 생성하는 편리한 방법을 제공한다. `NewView`가 필요한 기능을
제공하지 못한다면, 커스텀
[`View`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric#View)를 직접
생성할 수 있다.

예를 들어 모든 데이터 스트림 이름이 사용하는 단위의 접미사를 갖도록 정규 표현식
매칭을 사용하는 뷰는 다음과 같이 생성한다.

```go
re := regexp.MustCompile(`[._](ms|byte)$`)
var view metric.View = func(i metric.Instrument) (metric.Stream, bool) {
	// In a custom View function, you need to explicitly copy
	// the name, description, and unit.
	s := metric.Stream{Name: i.Name, Description: i.Description, Unit: i.Unit}
	// Any instrument that does not have a unit suffix defined, but has a
	// dimensional unit defined, update the name with a unit suffix.
	if re.MatchString(i.Name) {
		return s, false
	}
	switch i.Unit {
	case "ms":
		s.Name += ".ms"
	case "By":
		s.Name += ".byte"
	default:
		return s, false
	}
	return s, true
}

meterProvider := metric.NewMeterProvider(
	metric.WithView(view),
)
```

## 로그 {#logs}

로그는 **사용자 대상 오픈텔레메트리 로그 API가 존재하지 않는다**는 점에서 메트릭
및 트레이스와 다르다. 대신, 기존의 인기 있는 로그 패키지(slog, logrus, zap, logr
등)에서 로그를 오픈텔레메트리 생태계로 연결하는 도구가 존재한다. 이러한 설계
결정의 근거는 [로깅 명세](/docs/specs/otel/logs/)를 참고한다.

아래에서 다루는 두 가지 일반적인 워크플로우는 각각 서로 다른 애플리케이션 요구
사항을 충족한다.

### 컬렉터로 직접 전송하기 {#direct-to-collector}

**상태**: [실험적](/docs/specs/otel/document-status/)

컬렉터로 직접 전송하는(direct-to-Collector) 워크플로우에서는, 로그가
애플리케이션에서 네트워크 프로토콜(예: OTLP)을 사용해 컬렉터로 직접 방출된다. 이
워크플로우는 추가적인 로그 전달(forwarding) 구성 요소가 필요 없어 설정이
간단하며, 애플리케이션이 [로그 데이터 모델][log data model]을 준수하는 구조화된
로그를 쉽게 방출할 수 있게 해준다. 다만, 애플리케이션이 로그를 큐에 넣고
네트워크 위치로 내보내는 데 필요한 오버헤드는 모든 애플리케이션에 적합하지는
않을 수 있다.

이 워크플로우를 사용하려면:

- 오픈텔레메트리 [로그 SDK](#logs-sdk)를 구성하여 로그 레코드를 원하는
  대상([컬렉터][opentelemetry collector] 또는 기타)으로 내보낸다.
- 적절한 [로그 브리지](#log-bridge)를 사용한다.

#### 로그 SDK {#logs-sdk}

로그 SDK는 [컬렉터로 직접 전송하는](#direct-to-collector) 워크플로우를 사용할 때
로그가 어떻게 처리되는지를 결정한다. [로그 전달](#via-file-or-stdout)
워크플로우를 사용할 때는 로그 SDK가 필요 없다.

일반적인 로그 SDK 구성은 배치 처리하는 로그 레코드 프로세서와 OTLP 익스포터를
설치한다.

앱에서 [로그](/docs/concepts/signals/logs/)를 활성화하려면,
[로그 브리지](#log-bridge)를 사용할 수 있게 해주는 초기화된
[`LoggerProvider`](/docs/concepts/signals/logs/#logger-provider)가 있어야 한다.

`LoggerProvider`가 생성되지 않으면, 로그를 위한 오픈텔레메트리 API는 무동작
구현체를 사용하며 데이터 생성에 실패한다. 따라서 다음 패키지를 사용해 SDK 초기화
코드를 포함하도록 소스 코드를 수정해야 한다.

- [`go.opentelemetry.io/otel`][]
- [`go.opentelemetry.io/otel/sdk/log`][]
- [`go.opentelemetry.io/otel/sdk/resource`][]
- [`go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp`][]

알맞은 Go 모듈이 설치되어 있는지 확인한다.

```sh
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/sdk/log
```

그런 다음 로거 프로바이더를 초기화한다.

```go
package main

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp"
	"go.opentelemetry.io/otel/log/global"
	"go.opentelemetry.io/otel/sdk/log"
	"go.opentelemetry.io/otel/sdk/resource"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"
)

func main() {
	ctx := context.Background()

	// Create resource.
	res, err := newResource()
	if err != nil {
		panic(err)
	}

	// Create a logger provider.
	// You can pass this instance directly when creating bridges.
	loggerProvider, err := newLoggerProvider(ctx, res)
	if err != nil {
		panic(err)
	}

	// Handle shutdown properly so nothing leaks.
	defer func() {
		if err := loggerProvider.Shutdown(ctx); err != nil {
			fmt.Println(err)
		}
	}()

	// Register as global logger provider so that it can be accessed global.LoggerProvider.
	// Most log bridges use the global logger provider as default.
	// If the global logger provider is not set then a no-op implementation
	// is used, which fails to generate data.
	global.SetLoggerProvider(loggerProvider)
}

func newResource() (*resource.Resource, error) {
	return resource.Merge(
    resource.Default(),
		resource.NewWithAttributes(
      semconv.SchemaURL,
			semconv.ServiceName("my-service"),
			semconv.ServiceVersion("0.1.0"),
		),
  )
}

func newLoggerProvider(ctx context.Context, res *resource.Resource) (*log.LoggerProvider, error) {
	exporter, err := otlploghttp.New(ctx)
	if err != nil {
		return nil, err
	}
	processor := log.NewBatchProcessor(exporter)
	provider := log.NewLoggerProvider(
		log.WithResource(res),
		log.WithProcessor(processor),
	)
	return provider, nil
}
```

`LoggerProvider`가 구성되었으니, 이를 사용해 [로그 브리지](#log-bridge)를 설정할
수 있다.

#### 로그 브리지 {#log-bridge}

로그 브리지는 [로그 브리지 API][logs bridge API]를 사용해 기존 로그 패키지의
로그를 오픈텔레메트리 [로그 SDK](#logs-sdk)로 연결하는 구성 요소이다.

사용 가능한 로그 브리지의 전체 목록은
[오픈텔레메트리 레지스트리](/ecosystem/registry/?language=go&component=log-bridge)에서
찾을 수 있다.

각 로그 브리지 패키지 문서에는 사용 예시가 있을 것이다.

### 파일 또는 표준 출력을 통하기 {#via-file-or-stdout}

파일 또는 표준 출력(stdout) 워크플로우에서는, 로그가 파일이나 표준 출력으로
기록된다. 다른 구성 요소(예: FluentBit)가 로그를 읽거나 테일링(tailing)하고, 더
구조화된 형식으로 파싱하며, 컬렉터와 같은 대상으로 전달하는 역할을 담당한다. 이
워크플로우는 애플리케이션 요구 사항이
[컬렉터로 직접 전송하는](#direct-to-collector) 방식의 추가 오버헤드를 허용하지
않는 상황에서 선호될 수 있다. 다만, 다운스트림에서 필요한 모든 로그 필드가
로그에 인코딩되어 있어야 하고, 로그를 읽는 구성 요소가 그 데이터를 [로그 데이터
모델][log data model]로 파싱해야 한다. 로그 전달 구성 요소의 설치와 구성은 이
문서의 범위를 벗어난다.

## 다음 단계 {#next-steps}

또한 텔레메트리 데이터를 하나 이상의 텔레메트리 백엔드로
[내보내기](/docs/languages/go/exporters) 위해 적절한 익스포터를 구성해야 할
것이다.

[opentelemetry specification]: /docs/specs/otel/
[trace semantic conventions]: /docs/specs/semconv/general/trace/
[instrumentation library]: ../libraries/
[opentelemetry collector]:
  https://github.com/open-telemetry/opentelemetry-collector
[logs bridge API]: /docs/specs/otel/logs/api/
[log data model]: /docs/specs/otel/logs/data-model
[`go.opentelemetry.io/otel`]: https://pkg.go.dev/go.opentelemetry.io/otel
[`go.opentelemetry.io/otel/exporters/stdout/stdoutmetric`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/exporters/stdout/stdoutmetric
[`go.opentelemetry.io/otel/metric`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/metric
[`go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp
[`go.opentelemetry.io/otel/sdk/log`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/sdk/log
[`go.opentelemetry.io/otel/sdk/metric`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric
[`go.opentelemetry.io/otel/sdk/resource`]:
  https://pkg.go.dev/go.opentelemetry.io/otel/sdk/resource
