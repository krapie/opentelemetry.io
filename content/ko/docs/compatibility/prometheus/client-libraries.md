---
title: Prometheus 클라이언트 라이브러리 vs. 오픈텔레메트리
linkTitle: 클라이언트 라이브러리
cSpell:ignore: hvac
default_lang_commit: b4b91dc7bdbb03ae1de2c1e276194d98b9d15b94
---

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>

> [!NOTE]
>
> 이 페이지는 Java와 Go를 다룬다. 다른 언어의 예제도 계획되어 있다.

이 가이드는
[Prometheus 클라이언트 라이브러리](https://prometheus.io/docs/instrumenting/clientlibs/)에
익숙하며 오픈텔레메트리(OpenTelemetry) 메트릭 API와 SDK에서 이에 대응하는 패턴을
이해하고자 하는 개발자를 위한 것이다. 가장 흔히 쓰이는 패턴을 다루지만, 모든
것을 다루지는 않는다.

## 개념적 차이 {#conceptual-differences}

코드를 살펴보기 전에, 두 시스템 사이의 구조적인 차이 몇 가지를 이해해 두면
도움이 된다.
[Prometheus와 OpenMetrics 호환성](/docs/specs/otel/compatibility/prometheus_and_openmetrics/)
명세는 두 시스템 사이의 완전한 변환 규칙을 문서화하고 있다. 이 섹션에서는 새로운
계측 코드를 작성할 때 가장 관련이 깊은 차이를 다룬다.

### 레지스트리(MeterProvider) {#registry-meterprovider}

Prometheus에서 메트릭은 레지스트리(registry)에 등록되며, 기본적으로는
전역(global) 레지스트리이다. 코드 어디에서든 메트릭을 선언할 수 있고, 등록되고
나면 스크레이핑할 수 있게 된다. 익스포터(HTTP 서버 또는 OTLP 푸시)는 별도의
독립적인 단계로 레지스트리에 연결된다.

오픈텔레메트리에서 `MeterProvider`와 `Meter`는 메트릭 API의 일부이다.
라이브러리나 컴포넌트로 스코프된 `Meter`를 `MeterProvider`로부터 얻고, 그
`Meter`로부터 계측기를 생성한다. 이러한 측정값이 어떻게 처리되는지 — 어떤
익스포터가 이를 수신하는지, 어떻게 집계되는지, 어떤 일정으로 처리되는지 —는
`MeterProvider`에 바인딩된 SDK와 그 구성(configuration)에 따라 결정되며, 이는
계측 코드 자체와는 분리되어 있다([API와 SDK](#otel-api-and-sdk) 참고).

Prometheus와 마찬가지로, 오픈텔레메트리도 전역 `MeterProvider`(계측 코드에서
명시적인 연결이 필요하지 않음)와, 이를 지원하는 라이브러리에 전달할 수 있는
명시적인 `MeterProvider` 인스턴스를 모두 지원한다.

### 레이블 이름(속성) {#label-names-attributes}

Prometheus는 레이블(label) _이름_ 을 메트릭 생성 시점에 선언해야 한다. 레이블
_값_ 은 `labelValues(...)`를 통해 기록 시점에 바인딩된다.

오픈텔레메트리에는 사전 레이블 선언이 없다. 속성(attribute) 키와 값은
`Attributes`를 통해 측정 시점에 함께 제공된다.

### 명명 규칙 {#naming-conventions}

Prometheus는 `snake_case` 형식의 메트릭 이름을 사용한다. 카운터 이름은
`_total`로 끝난다. 관례적으로 Prometheus 메트릭 이름에는 충돌을 피하기 위해
애플리케이션이나 라이브러리 이름이 접두사로 붙는다(예:
`smart_home_hvac_on_seconds_total`). 이는 모든 메트릭이 하나의 평평한(flat) 전역
네임스페이스를 공유하기 때문이다.

오픈텔레메트리는 관례적으로
[점으로 구분된 이름(dotted name)](/docs/specs/semconv/general/naming/)을
사용한다. 소유권과 네임스페이스는 계측 스코프(instrumentation scope, `Meter`
이름, 예를 들어 `smart.home`)로 표현되므로, 메트릭 이름 자체에는 접두사가
필요하지 않다(예: `hvac.on`). Prometheus로 내보낼 때 익스포터는 이름을 변환한다.
점은 밑줄로 바뀌고, 단위 약어는 전체 단어로 확장되며(예: `s` → `seconds`),
카운터에는 `_total` 접미사가 붙는다. 단위가 `s`인 `hvac.on`이라는 이름의
오픈텔레메트리 카운터는 `hvac_on_seconds_total`로 내보내진다. 이름 변환 규칙
전체는
[호환성 명세](/docs/specs/otel/compatibility/prometheus_and_openmetrics/)를
참고한다. 변환 전략은 구성할 수 있다 — 예를 들어 UTF-8 문자를 보존하거나 단위 및
타입 접미사를 생략할 수 있다. 자세한 내용은
[Prometheus 익스포터](/docs/specs/otel/metrics/sdk_exporters/prometheus/) 구성
참고 문서를 확인한다.

### 상태 유지 계측기와 콜백 계측기 {#stateful-and-callback-instruments}

두 시스템 모두 두 가지 기록 모드를 지원한다.

- **Prometheus** 는 자체적으로 누적값을 유지하는 _상태 유지(stateful)_
  계측기(`Counter`, `Gauge`)와, 스크레이핑 시점에 콜백을 호출해 현재 값을
  반환하는 함수 기반 계측기를 구분한다. 명명 방식은 클라이언트 라이브러리마다
  다르다(Go에서는 `GaugeFunc`/`CounterFunc`, Java에서는
  `GaugeWithCallback`/`CounterWithCallback`).
- **오픈텔레메트리** 는 이를 _동기(synchronous)_ (counter, histogram 등)와
  _비동기(asynchronous)_ (등록된 콜백을 통해 관측)로 부른다. 의미는 동일하다.

또한 Prometheus의 `Gauge`는 서로 다른 두 가지 OTel 계측기 타입을 아우른다는
점에도 유의한다. 온도처럼 비가산적(non-additive)인 값에는 `Gauge`가, 활성 연결
수처럼 증가하거나 감소할 수 있는 가산적(additive)인 값에는 `UpDownCounter`가
대응한다. 자세한 내용은 [Gauge](#gauge)를 참고한다.

### OTel: API와 SDK {#otel-api-and-sdk}

오픈텔레메트리는 계측과 구성(configuration)을 **API** 패키지와 **SDK**
패키지라는 2계층 설계로 분리한다. API는 메트릭을 기록하는 데 사용되는
인터페이스를 정의한다. SDK는 구체적인 프로바이더, 익스포터, 처리 파이프라인 등
구현을 제공한다.

계측 코드는 API에만 의존해야 한다. SDK는 애플리케이션 시작 시점에 한 번
구성되고, 코드베이스의 나머지 부분에 전달되는 API 참조에 연결된다. 이렇게 하면
계측 라이브러리 코드가 특정 SDK 버전과 분리되며, 테스트를 위해 no-op 구현으로
손쉽게 바꿔 끼울 수 있다.

### OTel: 계측 스코프 {#otel-instrumentation-scope}

Prometheus 메트릭은 전역적이다. 프로세스 내의 모든 메트릭은 이름과 레이블만 으로
식별되는 동일한 평평한 네임스페이스를 공유한다.

오픈텔레메트리는 계측기 그룹마다 이름과 선택적인 버전(예: `smart.home`)으로
식별되는 `Meter`로 스코프를 지정한다. Prometheus로 내보낼 때, 스코프 이름과
버전은 모든 메트릭 포인트에 `otel_scope_name`과 `otel_scope_version` 레이블로
추가된다. 추가적인 스코프 속성이 있다면 이 또한 `otel_scope_[속성 이름]`이라는
이름의 레이블로 추가된다. 이러한 레이블은 자동으로 나타나므로, Prometheus에서
넘어온 사용자에게는 낯설 수 있다. 익스포터의 `without_scope_info` 옵션으로 이를
생략할 수 있다 — 자세한 내용은
[Prometheus 익스포터](/docs/specs/otel/metrics/sdk_exporters/prometheus/) 구성
참고 문서를 확인한다. 스코프 정보를 생략하는 것은 각 메트릭 이름이 단일
스코프에서만 생성될 때만 안전하다는 점에 유의한다. 두 스코프가 동일한 이름의
메트릭을 내보내는 경우, 스코프 레이블만이 이 둘을 구분하는 유일한 수단이다. 이
레이블이 없으면 출처를 구분할 방법 없이 중복된 시계열(time series)이 생기며,
이는 Prometheus에서 유효하지 않은 출력을 만들어낸다.

### OTel: 집계 템포럴리티 {#otel-aggregation-temporality}

Prometheus 메트릭은 항상 누적(cumulative)이다. 오픈텔레메트리는 누적과
델타(delta) 템포럴리티(temporality)를 모두 지원하지만, Prometheus 익스포터는
모든 계측기에 대해 누적 방식을 강제한다. Prometheus에서 마이그레이션하는
개발자에게는 이 부분이 투명하게 처리된다 — 이미 의존하고 있던 동작이 그대로
유지된다.

### OTel: 리소스 속성 {#otel-resource-attributes}

Prometheus는 `job`과 `instance` 레이블을 사용해 스크레이핑 대상(target)을
식별하며, 이 레이블은 스크레이핑 시점에 Prometheus 서버가 추가한다.

오픈텔레메트리에는 `Resource`가 있다. 이는 프로세스에서 나오는 모든 텔레메트리에
첨부되는 구조화된 메타데이터로, `service.name`이나 `service.instance.id` 같은
속성을 가진다. Prometheus로 내보낼 때 익스포터는 리소스 속성을 `job`과
`instance` 레이블에 매핑하고, 나머지 속성은 `target_info` 메트릭으로
노출한다(`target_info`는 OpenMetrics 1.0의 관례이다 — 현재 Prometheus에서 이를
수동으로 내보내고 있다면, OTel에서는 리소스 속성을 설정하는 것이 이에 대응한다).
정확한 매핑 규칙은
[호환성 명세](/docs/specs/otel/compatibility/prometheus_and_openmetrics/)를
참고한다. `target_info` 메트릭은 `without_target_info`로 생략할 수 있고, 특정
리소스 속성은 `with_resource_constant_labels`를 통해 메트릭 수준의 레이블로
승격할 수 있다. 자세한 내용은
[Prometheus 익스포터](/docs/specs/otel/metrics/sdk_exporters/prometheus/) 구성
참고 문서를 확인한다.

## 초기화 {#initialization}

아래 예제는 Prometheus 스크레이프 엔드포인트를 노출하는 방식과 OTLP 엔드포인트로
푸시하는 방식, 두 가지 주요 배포 패턴을 다룬다.

### Prometheus 스크레이프 엔드포인트 노출 {#expose-a-prometheus-scrape-endpoint}

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt "src/main/java/otel/PrometheusScrapeInit.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Counter;
import io.prometheus.metrics.exporter.httpserver.HTTPServer;
import java.io.IOException;

public class PrometheusScrapeInit {
  public static void main(String[] args) throws IOException, InterruptedException {
    // Create a counter and register it with the default PrometheusRegistry.
    Counter doorOpens =
        Counter.builder()
            .name("door_opens_total")
            .help("Total number of times a door has been opened")
            .labelNames("door")
            .register();

    // Start the HTTP server; Prometheus scrapes http://localhost:9464/metrics.
    HTTPServer server = HTTPServer.builder().port(9464).buildAndStart();
    Runtime.getRuntime().addShutdownHook(new Thread(server::close));

    doorOpens.labelValues("front").inc();

    Thread.currentThread().join(); // sleep forever
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelScrapeInit.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.exporter.prometheus.PrometheusHttpServer;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.metrics.SdkMeterProvider;

public class OtelScrapeInit {
  // Preallocate attribute keys and, when values are static, entire Attributes objects.
  private static final AttributeKey<String> DOOR = AttributeKey.stringKey("door");
  private static final Attributes FRONT_DOOR = Attributes.of(DOOR, "front");

  public static void main(String[] args) throws InterruptedException {
    // Configure the SDK: register a Prometheus reader that serves /metrics.
    OpenTelemetrySdk sdk =
        OpenTelemetrySdk.builder()
            .setMeterProvider(
                SdkMeterProvider.builder()
                    .registerMetricReader(PrometheusHttpServer.builder().setPort(9464).build())
                    .build())
            .build();
    Runtime.getRuntime().addShutdownHook(new Thread(sdk::close));

    // Instrumentation code uses the OpenTelemetry API type, not the SDK type directly.
    OpenTelemetry openTelemetry = sdk;

    // Metrics are served at http://localhost:9464/metrics.
    Meter meter = openTelemetry.getMeter("smart.home");
    LongCounter doorOpens =
        meter
            .counterBuilder("door.opens")
            .setDescription("Total number of times a door has been opened")
            .build();

    doorOpens.add(1, FRONT_DOOR);

    Thread.currentThread().join(); // sleep forever
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_scrape_init.go"?>

```go
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	// Create a counter and register it with a custom registry.
	reg := prometheus.NewRegistry()
	doorOpens := prometheus.NewCounterVec(prometheus.CounterOpts{
		Name: "door_opens_total",
		Help: "Total number of times a door has been opened",
	}, []string{"door"})
	reg.MustRegister(doorOpens)

	// Prometheus scrapes http://localhost:9464/metrics.
	http.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
	go http.ListenAndServe(":9464", nil) //nolint:errcheck

	doorOpens.WithLabelValues("front").Inc()

	select {} // sleep forever
}
```

OpenTelemetry

<?code-excerpt "otel_scrape_init.go"?>

```go
package main

import (
	"context"
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/prometheus"
	"go.opentelemetry.io/otel/metric"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
)

func main() {
	ctx := context.Background()
	// Configure the SDK: register a Prometheus reader that serves /metrics.
	exporter, err := prometheus.New()
	if err != nil {
		panic(err)
	}
	provider := sdkmetric.NewMeterProvider(sdkmetric.WithReader(exporter))
	defer provider.Shutdown(ctx) //nolint:errcheck

	// Metrics are served at http://localhost:9464/metrics.
	http.Handle("/metrics", promhttp.Handler())
	go http.ListenAndServe(":9464", nil) //nolint:errcheck

	// Instrumentation code uses the API, not the SDK, directly.
	meter := provider.Meter("smart.home")
	doorOpens, err := meter.Int64Counter("door.opens",
		metric.WithDescription("Total number of times a door has been opened"))
	if err != nil {
		panic(err)
	}

	doorOpens.Add(ctx, 1, metric.WithAttributes(attribute.String("door", "front")))

	select {} // sleep forever
}
```

{{% /tab %}} {{< /tabpane >}}

### OTLP 엔드포인트로 메트릭 푸시 {#push-metrics-to-an-otlp-endpoint}

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusOtlpInit.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Counter;
import io.prometheus.metrics.exporter.opentelemetry.OpenTelemetryExporter;

public class PrometheusOtlpInit {
  public static void main(String[] args) throws Exception {
    // Create a counter and register it with the default PrometheusRegistry.
    Counter doorOpens =
        Counter.builder()
            .name("door_opens_total")
            .help("Total number of times a door has been opened")
            .labelNames("door")
            .register();

    // Start the OTLP exporter. It reads from the default PrometheusRegistry and
    // pushes metrics to the configured endpoint on a fixed interval.
    OpenTelemetryExporter exporter =
        OpenTelemetryExporter.builder()
            .protocol("http/protobuf")
            .endpoint("http://localhost:4318")
            .intervalSeconds(60)
            .buildAndStart();
    Runtime.getRuntime().addShutdownHook(new Thread(exporter::close));

    doorOpens.labelValues("front").inc();

    Thread.currentThread().join(); // sleep forever
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelOtlpInit.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.exporter.otlp.http.metrics.OtlpHttpMetricExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.metrics.SdkMeterProvider;
import io.opentelemetry.sdk.metrics.export.PeriodicMetricReader;
import java.time.Duration;

public class OtelOtlpInit {
  public static void main(String[] args) throws InterruptedException {
    // Configure the SDK: export metrics over OTLP/HTTP on a fixed interval.
    OpenTelemetrySdk sdk =
        OpenTelemetrySdk.builder()
            .setMeterProvider(
                SdkMeterProvider.builder()
                    .registerMetricReader(
                        PeriodicMetricReader.builder(
                                OtlpHttpMetricExporter.builder()
                                    .setEndpoint("http://localhost:4318")
                                    .build())
                            .setInterval(Duration.ofSeconds(60))
                            .build())
                    .build())
            .build();
    Runtime.getRuntime().addShutdownHook(new Thread(sdk::close));

    // Instrumentation code uses the OpenTelemetry API type, not the SDK type directly.
    OpenTelemetry openTelemetry = sdk;

    Meter meter = openTelemetry.getMeter("smart.home");
    LongCounter doorOpens =
        meter
            .counterBuilder("door.opens")
            .setDescription("Total number of times a door has been opened")
            .build();

    doorOpens.add(1);

    Thread.currentThread().join(); // sleep forever
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

Prometheus Go 클라이언트 라이브러리에는 OTLP 푸시 익스포터가 포함되어 있지 않다.

OpenTelemetry

<?code-excerpt "otel_otlp_init.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
	"go.opentelemetry.io/otel/metric"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
)

func main() {
	ctx := context.Background()
	// Configure the SDK: export metrics over OTLP/HTTP on a fixed interval.
	// The endpoint defaults to localhost:4318 and can be configured via
	// the OTEL_EXPORTER_OTLP_ENDPOINT environment variable.
	exporter, err := otlpmetrichttp.New(ctx)
	if err != nil {
		panic(err)
	}
	provider := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(sdkmetric.NewPeriodicReader(exporter)),
	)
	defer provider.Shutdown(ctx) //nolint:errcheck

	meter := provider.Meter("smart.home")
	doorOpens, err := meter.Int64Counter("door.opens",
		metric.WithDescription("Total number of times a door has been opened"))
	if err != nil {
		panic(err)
	}

	doorOpens.Add(ctx, 1, metric.WithAttributes(attribute.String("door", "front")))

	select {} // sleep forever
}
```

{{% /tab %}} {{< /tabpane >}}

## 카운터 {#counter}

카운터(counter)는 단조 증가(monotonically increasing)하는 값을 기록한다.
Prometheus의 `Counter`는 오픈텔레메트리의 `Counter` 계측기에 대응한다.

- **단위 인코딩(Unit encoding)**: Prometheus는 단위를 메트릭 이름에
  인코딩한다(`hvac_on_seconds_total`). 오픈텔레메트리는 이름(`hvac.on`)과
  단위(`s`)를 분리하며, Prometheus 익스포터가 단위 접미사를 자동으로 붙인다.

### 카운터 {#counter-1}

Prometheus의 `Counter`에는 오픈텔레메트리에 대응하는 기능이 없는 두 가지 시리즈
관리 기능이 있다.

- **시리즈 사전 초기화(Series pre-initialization)**: Prometheus 클라이언트는
  레이블 값 조합을 미리 초기화해서, 어떤 기록도 발생하기 전에 스크레이핑 출력에
  값 0으로 나타나게 할 수 있다. 오픈텔레메트리에는 이에 대응하는 기능이 없다.
  데이터 포인트는 첫 `add()` 호출 시점에야 처음 나타난다.
- **사전 바인딩된 시리즈(Pre-bound series)**: Prometheus 클라이언트는
  `labelValues()`의 결과를 캐시해 특정 레이블 값 조합에 미리 바인딩할 수 있게
  해준다. 이후 호출은 내부 시리즈 조회를 건너뛰고 데이터 포인트로 바로 이동한다.
  오픈텔레메트리에는 이에 대응하는 기능이 없지만,
  [논의가 진행 중](https://github.com/open-telemetry/opentelemetry-specification/issues/4126)이다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusCounter.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Counter;

public class PrometheusCounter {
  public static void counterUsage() {
    Counter hvacOnTime =
        Counter.builder()
            .name("hvac_on_seconds_total")
            .help("Total time the HVAC system has been running, in seconds")
            .labelNames("zone")
            .register();

    // Pre-bind to label value sets: subsequent calls go directly to the data point,
    // skipping the internal series lookup.
    var upstairs = hvacOnTime.labelValues("upstairs");
    var downstairs = hvacOnTime.labelValues("downstairs");

    upstairs.inc(127.5);
    downstairs.inc(3600.0);

    // Pre-initialize zones so they appear in /metrics with value 0 on startup.
    hvacOnTime.initLabelValues("basement");
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelCounter.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleCounter;
import io.opentelemetry.api.metrics.Meter;

public class OtelCounter {
  // Preallocate attribute keys and, when values are static, entire Attributes objects.
  private static final AttributeKey<String> ZONE = AttributeKey.stringKey("zone");
  private static final Attributes UPSTAIRS = Attributes.of(ZONE, "upstairs");
  private static final Attributes DOWNSTAIRS = Attributes.of(ZONE, "downstairs");

  public static void counterUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // HVAC on-time is fractional — use ofDoubles() to get a DoubleCounter.
    // No upfront label declaration: attributes are provided at record time.
    DoubleCounter hvacOnTime =
        meter
            .counterBuilder("hvac.on")
            .setDescription("Total time the HVAC system has been running")
            .setUnit("s")
            .ofDoubles()
            .build();

    hvacOnTime.add(127.5, UPSTAIRS);
    hvacOnTime.add(3600.0, DOWNSTAIRS);
  }
}
```

주요 차이점:

- `inc(value)` → `add(value)`. Prometheus와 달리 오픈텔레메트리는 명시적인 값을
  요구한다 — 인자 없는 `inc()` 축약형은 없다.
- 오픈텔레메트리는 `LongCounter`(정수, 기본값)와 `DoubleCounter`(소수 값에
  사용하며, `.ofDoubles()`를 통해 얻는다)를 구분한다. Prometheus는 단일
  `Counter` 타입을 사용한다.
- 핫 패스(hot path)에서 호출마다 할당이 발생하지 않도록 `AttributeKey`
  인스턴스는 (항상) 미리 할당하고, `Attributes` 객체는 (값이 정적일 때) 미리
  할당한다.

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_counter.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

var hvacOnTime = prometheus.NewCounterVec(prometheus.CounterOpts{
	Name: "hvac_on_seconds_total",
	Help: "Total time the HVAC system has been running, in seconds",
}, []string{"zone"})

func prometheusCounterUsage(reg *prometheus.Registry) {
	reg.MustRegister(hvacOnTime)

	// Pre-bind to label value sets: subsequent calls avoid the series lookup.
	upstairs := hvacOnTime.WithLabelValues("upstairs")
	downstairs := hvacOnTime.WithLabelValues("downstairs")

	upstairs.Add(127.5)
	downstairs.Add(3600.0)

	// Pre-initialize a series so it appears in /metrics with value 0.
	hvacOnTime.WithLabelValues("basement")
}
```

OpenTelemetry

<?code-excerpt "otel_counter.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Preallocate attribute options when values are static to avoid per-call allocation.
var (
	zoneUpstairsOpts   = []metric.AddOption{metric.WithAttributes(attribute.String("zone", "upstairs"))}
	zoneDownstairsOpts = []metric.AddOption{metric.WithAttributes(attribute.String("zone", "downstairs"))}
)

func otelCounterUsage(ctx context.Context, meter metric.Meter) {
	// No upfront label declaration: attributes are provided at record time.
	hvacOnTime, err := meter.Float64Counter("hvac.on",
		metric.WithDescription("Total time the HVAC system has been running"),
		metric.WithUnit("s"))
	if err != nil {
		panic(err)
	}

	hvacOnTime.Add(ctx, 127.5, zoneUpstairsOpts...)
	hvacOnTime.Add(ctx, 3600.0, zoneDownstairsOpts...)
}
```

주요 차이점:

- `Add(value)` → `Add(ctx, value, metric.WithAttributes(...))`. 모든 계측기
  호출은 첫 번째 인자로 `context.Context`를 요구한다.
- Go에서는 `meter.Float64Counter`와 `meter.Int64Counter`가 별도의 메서드이다.
  Prometheus는 단일 `Counter` 타입을 사용한다.
- 계측기 생성은 `(Instrument, error)`를 반환하며, 오류를 반드시 처리해야 한다.

{{% /tab %}} {{< /tabpane >}}

### 콜백(비동기) 카운터 {#callback-async-counter}

콜백 카운터(오픈텔레메트리에서는 비동기 카운터)는 총합이 디바이스나 런타임 같은
외부 소스에 의해 유지되며, 직접 증가시키는 대신 수집 시점에 관측하고 싶을 때
사용한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusCounterCallback.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.CounterWithCallback;

public class PrometheusCounterCallback {
  public static void counterCallbackUsage() {
    // Each zone has its own smart energy meter tracking cumulative joule totals.
    // Use a callback counter to report those values at scrape time without
    // maintaining separate counters in application code.
    CounterWithCallback.builder()
        .name("energy_consumed_joules_total")
        .help("Total energy consumed in joules")
        .labelNames("zone")
        .callback(
            callback -> {
              callback.call(SmartHomeDevices.totalEnergyJoules("upstairs"), "upstairs");
              callback.call(SmartHomeDevices.totalEnergyJoules("downstairs"), "downstairs");
            })
        .register();
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelCounterCallback.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;

public class OtelCounterCallback {
  private static final AttributeKey<String> ZONE = AttributeKey.stringKey("zone");
  private static final Attributes UPSTAIRS = Attributes.of(ZONE, "upstairs");
  private static final Attributes DOWNSTAIRS = Attributes.of(ZONE, "downstairs");

  public static void counterCallbackUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // Each zone has its own smart energy meter tracking cumulative joule totals.
    // Use an asynchronous counter to report those values when a MetricReader
    // collects metrics, without maintaining separate counters in application code.
    meter
        .counterBuilder("energy.consumed")
        .setDescription("Total energy consumed")
        .setUnit("J")
        .ofDoubles()
        .buildWithCallback(
            measurement -> {
              measurement.record(SmartHomeDevices.totalEnergyJoules("upstairs"), UPSTAIRS);
              measurement.record(SmartHomeDevices.totalEnergyJoules("downstairs"), DOWNSTAIRS);
            });
  }
}
```

주요 차이점:

- 오픈텔레메트리는 정수 카운터와 부동소수점 카운터를 구분한다. `.ofDoubles()`는
  부동소수점 변형을 선택한다. Prometheus의 `CounterWithCallback`은 항상
  부동소수점 값을 사용한다.

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_counter_callback.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

type energyCollector struct{ desc *prometheus.Desc }

func newEnergyCollector() *energyCollector {
	return &energyCollector{desc: prometheus.NewDesc(
		"energy_consumed_joules_total",
		"Total energy consumed in joules",
		[]string{"zone"}, nil,
	)}
}

func (c *energyCollector) Describe(ch chan<- *prometheus.Desc) { ch <- c.desc }
func (c *energyCollector) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.CounterValue, totalEnergyJoules("upstairs"), "upstairs")
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.CounterValue, totalEnergyJoules("downstairs"), "downstairs")
}

func prometheusCounterCallbackUsage(reg *prometheus.Registry) {
	// Each zone has its own smart energy meter tracking cumulative joule totals.
	// Implement prometheus.Collector to report those values at scrape time.
	reg.MustRegister(newEnergyCollector())
}
```

OpenTelemetry

<?code-excerpt "otel_counter_callback.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

var (
	zoneUpstairs   = attribute.String("zone", "upstairs")
	zoneDownstairs = attribute.String("zone", "downstairs")
)

func otelCounterCallbackUsage(meter metric.Meter) {
	// Each zone has its own smart energy meter tracking cumulative joule totals.
	// Use an observable counter to report those values when metrics are collected.
	_, err := meter.Float64ObservableCounter("energy.consumed",
		metric.WithDescription("Total energy consumed"),
		metric.WithUnit("J"),
		metric.WithFloat64Callback(func(_ context.Context, o metric.Float64Observer) error {
			o.Observe(totalEnergyJoules("upstairs"), metric.WithAttributes(zoneUpstairs))
			o.Observe(totalEnergyJoules("downstairs"), metric.WithAttributes(zoneDownstairs))
			return nil
		}))
	if err != nil {
		panic(err)
	}
}
```

주요 차이점:

- Prometheus 예제는 레이블이 달린 카운터 값을 보고하기 위해 `Describe`와
  `Collect` 메서드로 `prometheus.Collector`를 구현한다.
- 오픈텔레메트리는 `Float64ObservableCounter`와 `Int64ObservableCounter`를
  구분한다.

{{% /tab %}} {{< /tabpane >}}

## 게이지 {#gauge}

게이지(gauge)는 증가하거나 감소할 수 있는 순간값(instantaneous value)을
기록한다. Prometheus는 이러한 모든 값에 단일 `Gauge` 타입을 사용하지만,
오픈텔레메트리는 올바른 계측기를 선택할 때 **가산적(additive)** 값과
**비가산적(non-additive)** 값을 구분한다.

- **비가산적** 값은 인스턴스 간에 의미 있게 합산할 수 없다 — 예를 들어 온도의
  경우, 세 개의 방 센서에서 읽은 값을 더해도 유용한 숫자가 나오지 않는다. 이는
  OTel의 `Gauge`와 `ObservableGauge`에 대응한다.
- **가산적** 값은 인스턴스 간에 의미 있게 합산할 수 있다 — 예를 들어 서비스
  인스턴스 전체에서 연결된 디바이스 수를 합산하면 유용한 총합을 얻을 수 있다.
  이는 OTel의 `UpDownCounter`와 `ObservableUpDownCounter`에 대응한다.

이 구분은 abs, inc/dec, 콜백 변형을 포함한 모든 게이지 패턴에 적용된다. 더
자세한 설명은
[계측기 선택 가이드](/docs/specs/otel/metrics/supplementary-guidelines/#instrument-selection)를
참고한다.

### 게이지 — abs {#gauge--abs}

이 패턴은 구성 값이나 디바이스 설정값(setpoint)처럼 절대값으로 기록되는 값에
사용한다. Prometheus의 `Gauge`는 오픈텔레메트리의 `Gauge` 계측기에 대응한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusGauge.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Gauge;

public class PrometheusGauge {
  public static void gaugeUsage() {
    Gauge thermostatSetpoint =
        Gauge.builder()
            .name("thermostat_setpoint_celsius")
            .help("Target temperature set on the thermostat")
            .labelNames("zone")
            .register();

    thermostatSetpoint.labelValues("upstairs").set(22.5);
    thermostatSetpoint.labelValues("downstairs").set(20.0);
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelGauge.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleGauge;
import io.opentelemetry.api.metrics.Meter;

public class OtelGauge {
  // Preallocate attribute keys and, when values are static, entire Attributes objects.
  private static final AttributeKey<String> ZONE = AttributeKey.stringKey("zone");
  private static final Attributes UPSTAIRS = Attributes.of(ZONE, "upstairs");
  private static final Attributes DOWNSTAIRS = Attributes.of(ZONE, "downstairs");

  public static void gaugeUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    DoubleGauge thermostatSetpoint =
        meter
            .gaugeBuilder("thermostat.setpoint")
            .setDescription("Target temperature set on the thermostat")
            .setUnit("Cel")
            .build();

    thermostatSetpoint.set(22.5, UPSTAIRS);
    thermostatSetpoint.set(20.0, DOWNSTAIRS);
  }
}
```

주요 차이점:

- `set(value)` → `set(value, attributes)`. 메서드 이름은 동일하다.
- 오픈텔레메트리는 `LongGauge`(정수, `.ofLongs()`를 통해)와
  `DoubleGauge`(기본값)를 구분한다. Prometheus는 단일 `Gauge` 타입을 사용한다.
- 핫 패스에서 호출마다 할당이 발생하지 않도록 `AttributeKey` 인스턴스는 (항상)
  미리 할당하고, `Attributes` 객체는 (값이 정적일 때) 미리 할당한다.

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_gauge.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

var thermostatSetpoint = prometheus.NewGaugeVec(prometheus.GaugeOpts{
	Name: "thermostat_setpoint_celsius",
	Help: "Target temperature set on the thermostat",
}, []string{"zone"})

func prometheusGaugeUsage(reg *prometheus.Registry) {
	reg.MustRegister(thermostatSetpoint)

	thermostatSetpoint.WithLabelValues("upstairs").Set(22.5)
	thermostatSetpoint.WithLabelValues("downstairs").Set(20.0)
}
```

OpenTelemetry

<?code-excerpt "otel_gauge.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Preallocate attribute options when values are static to avoid per-call allocation.
var (
	zoneUpstairsGaugeOpts   = []metric.RecordOption{metric.WithAttributes(attribute.String("zone", "upstairs"))}
	zoneDownstairsGaugeOpts = []metric.RecordOption{metric.WithAttributes(attribute.String("zone", "downstairs"))}
)

func otelGaugeUsage(ctx context.Context, meter metric.Meter) {
	thermostatSetpoint, err := meter.Float64Gauge("thermostat.setpoint",
		metric.WithDescription("Target temperature set on the thermostat"),
		metric.WithUnit("Cel"))
	if err != nil {
		panic(err)
	}

	thermostatSetpoint.Record(ctx, 22.5, zoneUpstairsGaugeOpts...)
	thermostatSetpoint.Record(ctx, 20.0, zoneDownstairsGaugeOpts...)
}
```

주요 차이점:

- `Set(value)` → `Record(ctx, value, metric.WithAttributes(...))`.
- Go에서는 `meter.Float64Gauge`와 `meter.Int64Gauge`가 별도의 메서드이다.
  Prometheus는 단일 `Gauge` 타입을 사용한다.

{{% /tab %}} {{< /tabpane >}}

### 콜백 게이지 — abs {#callback-gauge--abs}

콜백 게이지(오픈텔레메트리에서는 비동기 게이지)는 센서 값처럼 비가산적인 값이
외부에서 유지되며, 직접 추적하는 대신 수집 시점에 관측하고 싶을 때 사용한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusGaugeCallback.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.GaugeWithCallback;

public class PrometheusGaugeCallback {
  public static void gaugeCallbackUsage() {
    // Temperature sensors maintain their own readings in firmware.
    // Use a callback gauge to report those values at scrape time without
    // maintaining a separate gauge in application code.
    GaugeWithCallback.builder()
        .name("room_temperature_celsius")
        .help("Current temperature in the room")
        .labelNames("room")
        .callback(
            callback -> {
              callback.call(SmartHomeDevices.livingRoomTemperatureCelsius(), "living_room");
              callback.call(SmartHomeDevices.bedroomTemperatureCelsius(), "bedroom");
            })
        .register();
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelGaugeCallback.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;

public class OtelGaugeCallback {
  private static final AttributeKey<String> ROOM = AttributeKey.stringKey("room");
  private static final Attributes LIVING_ROOM = Attributes.of(ROOM, "living_room");
  private static final Attributes BEDROOM = Attributes.of(ROOM, "bedroom");

  public static void gaugeCallbackUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // Temperature sensors maintain their own readings in firmware.
    // Use an asynchronous gauge to report those values when a MetricReader
    // collects metrics, without maintaining separate gauges in application code.
    meter
        .gaugeBuilder("room.temperature")
        .setDescription("Current temperature in the room")
        .setUnit("Cel")
        .buildWithCallback(
            measurement -> {
              measurement.record(SmartHomeDevices.livingRoomTemperatureCelsius(), LIVING_ROOM);
              measurement.record(SmartHomeDevices.bedroomTemperatureCelsius(), BEDROOM);
            });
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_gauge_callback.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

type temperatureCollector struct{ desc *prometheus.Desc }

func newTemperatureCollector() *temperatureCollector {
	return &temperatureCollector{desc: prometheus.NewDesc(
		"room_temperature_celsius",
		"Current temperature in the room",
		[]string{"room"}, nil,
	)}
}

func (c *temperatureCollector) Describe(ch chan<- *prometheus.Desc) { ch <- c.desc }
func (c *temperatureCollector) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.GaugeValue, livingRoomTemperatureCelsius(), "living_room")
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.GaugeValue, bedroomTemperatureCelsius(), "bedroom")
}

func prometheusGaugeCallbackUsage(reg *prometheus.Registry) {
	// Temperature sensors maintain their own readings in firmware.
	// Implement prometheus.Collector to report those values at scrape time.
	reg.MustRegister(newTemperatureCollector())
}
```

OpenTelemetry

<?code-excerpt "otel_gauge_callback.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

var (
	roomLivingRoom = attribute.String("room", "living_room")
	roomBedroom    = attribute.String("room", "bedroom")
)

func otelGaugeCallbackUsage(meter metric.Meter) {
	// Temperature sensors maintain their own readings in firmware.
	// Use an observable gauge to report those values when metrics are collected.
	_, err := meter.Float64ObservableGauge("room.temperature",
		metric.WithDescription("Current temperature in the room"),
		metric.WithUnit("Cel"),
		metric.WithFloat64Callback(func(_ context.Context, o metric.Float64Observer) error {
			o.Observe(livingRoomTemperatureCelsius(), metric.WithAttributes(roomLivingRoom))
			o.Observe(bedroomTemperatureCelsius(), metric.WithAttributes(roomBedroom))
			return nil
		}))
	if err != nil {
		panic(err)
	}
}
```

주요 차이점:

- Prometheus 예제는 레이블이 달린 게이지 값을 보고하기 위해 `Describe`와
  `Collect` 메서드로 `prometheus.Collector`를 구현한다.

{{% /tab %}} {{< /tabpane >}}

### 게이지 — inc와 dec {#gauge--inc-and-dec}

Prometheus의 `Gauge`는 연결된 디바이스 수나 활성 세션 수처럼 점진적으로 변화하는
값에 대해 증가와 감소를 지원한다. 오픈텔레메트리의 `Gauge`는 절대값만 기록하며,
이 패턴은 오픈텔레메트리의 `UpDownCounter` 계측기에 대응한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusUpDownCounter.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Gauge;

public class PrometheusUpDownCounter {
  public static void upDownCounterUsage() {
    // Prometheus uses Gauge for values that can increase or decrease.
    Gauge devicesConnected =
        Gauge.builder()
            .name("devices_connected")
            .help("Number of smart home devices currently connected")
            .labelNames("device_type")
            .register();

    // Increment when a device connects, decrement when it disconnects.
    devicesConnected.labelValues("thermostat").inc();
    devicesConnected.labelValues("thermostat").inc();
    devicesConnected.labelValues("lock").inc();
    devicesConnected.labelValues("lock").dec();
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelUpDownCounter.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongUpDownCounter;
import io.opentelemetry.api.metrics.Meter;

public class OtelUpDownCounter {
  // Preallocate attribute keys and, when values are static, entire Attributes objects.
  private static final AttributeKey<String> DEVICE_TYPE = AttributeKey.stringKey("device_type");
  private static final Attributes THERMOSTAT = Attributes.of(DEVICE_TYPE, "thermostat");
  private static final Attributes LOCK = Attributes.of(DEVICE_TYPE, "lock");

  public static void upDownCounterUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    LongUpDownCounter devicesConnected =
        meter
            .upDownCounterBuilder("devices.connected")
            .setDescription("Number of smart home devices currently connected")
            .build();

    // add() accepts positive and negative values.
    devicesConnected.add(1, THERMOSTAT);
    devicesConnected.add(1, THERMOSTAT);
    devicesConnected.add(1, LOCK);
    devicesConnected.add(-1, LOCK);
  }
}
```

주요 차이점:

- `inc()` / `dec()` → `add(1)` / `add(-1)`. `add()`는 양수와 음수 값을 모두
  받는다.
- Prometheus의 타입은 `Gauge`이고, 오픈텔레메트리의 타입은
  `LongUpDownCounter`(또는 `.ofDoubles()`를 통한 `DoubleUpDownCounter`)이다.

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_up_down_counter.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

// Prometheus uses Gauge for values that can increase or decrease.
var devicesConnected = prometheus.NewGaugeVec(prometheus.GaugeOpts{
	Name: "devices_connected",
	Help: "Number of smart home devices currently connected",
}, []string{"device_type"})

func prometheusUpDownCounterUsage(reg *prometheus.Registry) {
	reg.MustRegister(devicesConnected)

	// Increment when a device connects, decrement when it disconnects.
	devicesConnected.WithLabelValues("thermostat").Inc()
	devicesConnected.WithLabelValues("thermostat").Inc()
	devicesConnected.WithLabelValues("lock").Inc()
	devicesConnected.WithLabelValues("lock").Dec()
}
```

OpenTelemetry

<?code-excerpt "otel_up_down_counter.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Preallocate attribute options when values are static to avoid per-call allocation.
var (
	deviceThermostatAddOpts = []metric.AddOption{metric.WithAttributes(attribute.String("device_type", "thermostat"))}
	deviceLockAddOpts       = []metric.AddOption{metric.WithAttributes(attribute.String("device_type", "lock"))}
)

func otelUpDownCounterUsage(ctx context.Context, meter metric.Meter) {
	devicesConnected, err := meter.Int64UpDownCounter("devices.connected",
		metric.WithDescription("Number of smart home devices currently connected"))
	if err != nil {
		panic(err)
	}

	// Add() accepts positive and negative values.
	devicesConnected.Add(ctx, 1, deviceThermostatAddOpts...)
	devicesConnected.Add(ctx, 1, deviceThermostatAddOpts...)
	devicesConnected.Add(ctx, 1, deviceLockAddOpts...)
	devicesConnected.Add(ctx, -1, deviceLockAddOpts...)
}
```

주요 차이점:

- `Inc()` / `Dec()` → `Add(ctx, 1, ...)` / `Add(ctx, -1, ...)`. `Add()`는 양수와
  음수 값을 모두 받는다.
- Prometheus의 타입은 `Gauge`이고, 오픈텔레메트리의 타입은
  `Int64UpDownCounter`(또는 `meter.Float64UpDownCounter`를 통한
  `Float64UpDownCounter`)이다.

{{% /tab %}} {{< /tabpane >}}

### 콜백 게이지 — inc와 dec {#callback-gauge--inc-and-dec}

콜백 게이지(오픈텔레메트리에서는 비동기 업다운카운터)는, 그렇지 않았다면
`inc()`/`dec()`로 추적했을 가산적인 값이 디바이스 관리자나 커넥션 풀 같은
외부에서 유지되며, 이를 수집 시점에 관측하고 싶을 때 사용한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusUpDownCounterCallback.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.GaugeWithCallback;

public class PrometheusUpDownCounterCallback {
  public static void upDownCounterCallbackUsage() {
    // The device manager maintains the count of connected devices.
    // Use a callback gauge to report that value at scrape time.
    GaugeWithCallback.builder()
        .name("devices_connected")
        .help("Number of smart home devices currently connected")
        .labelNames("device_type")
        .callback(
            callback -> {
              callback.call(SmartHomeDevices.connectedDeviceCount("thermostat"), "thermostat");
              callback.call(SmartHomeDevices.connectedDeviceCount("lock"), "lock");
            })
        .register();
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelUpDownCounterCallback.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;

public class OtelUpDownCounterCallback {
  private static final AttributeKey<String> DEVICE_TYPE = AttributeKey.stringKey("device_type");
  private static final Attributes THERMOSTAT = Attributes.of(DEVICE_TYPE, "thermostat");
  private static final Attributes LOCK = Attributes.of(DEVICE_TYPE, "lock");

  public static void upDownCounterCallbackUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // The device manager maintains the count of connected devices.
    // Use an asynchronous up-down counter to report that value when a MetricReader
    // collects metrics.
    meter
        .upDownCounterBuilder("devices.connected")
        .setDescription("Number of smart home devices currently connected")
        .buildWithCallback(
            measurement -> {
              measurement.record(SmartHomeDevices.connectedDeviceCount("thermostat"), THERMOSTAT);
              measurement.record(SmartHomeDevices.connectedDeviceCount("lock"), LOCK);
            });
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_up_down_counter_callback.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

type deviceCountCollector struct{ desc *prometheus.Desc }

func newDeviceCountCollector() *deviceCountCollector {
	return &deviceCountCollector{desc: prometheus.NewDesc(
		"devices_connected",
		"Number of smart home devices currently connected",
		[]string{"device_type"}, nil,
	)}
}

func (c *deviceCountCollector) Describe(ch chan<- *prometheus.Desc) { ch <- c.desc }
func (c *deviceCountCollector) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.GaugeValue, float64(connectedDeviceCount("thermostat")), "thermostat")
	ch <- prometheus.MustNewConstMetric(c.desc, prometheus.GaugeValue, float64(connectedDeviceCount("lock")), "lock")
}

func prometheusUpDownCounterCallbackUsage(reg *prometheus.Registry) {
	// The device manager maintains the count of connected devices.
	// Implement prometheus.Collector to report those values at scrape time.
	reg.MustRegister(newDeviceCountCollector())
}
```

OpenTelemetry

<?code-excerpt "otel_up_down_counter_callback.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

var (
	deviceThermostat = attribute.String("device_type", "thermostat")
	deviceLock       = attribute.String("device_type", "lock")
)

func otelUpDownCounterCallbackUsage(meter metric.Meter) {
	// The device manager maintains the count of connected devices.
	// Use an observable up-down counter to report that value when metrics are collected.
	_, err := meter.Int64ObservableUpDownCounter("devices.connected",
		metric.WithDescription("Number of smart home devices currently connected"),
		metric.WithInt64Callback(func(_ context.Context, o metric.Int64Observer) error {
			o.Observe(int64(connectedDeviceCount("thermostat")), metric.WithAttributes(deviceThermostat))
			o.Observe(int64(connectedDeviceCount("lock")), metric.WithAttributes(deviceLock))
			return nil
		}))
	if err != nil {
		panic(err)
	}
}
```

주요 차이점:

- Prometheus 예제는 레이블이 달린 게이지 값을 보고하기 위해 `Describe`와
  `Collect` 메서드로 `prometheus.Collector`를 구현한다.
- `Int64ObservableUpDownCounter`는 `metric.WithInt64Callback`을 사용한다.

{{% /tab %}} {{< /tabpane >}}

## 히스토그램 {#histogram}

히스토그램(histogram)은 일련의 측정값의 분포를 기록하며, 관측 횟수, 합계, 그리고
구성 가능한 버킷 경계 안에 속하는 개수를 추적한다.

Prometheus와 오픈텔레메트리 모두 클래식(명시적 버킷) 히스토그램과 네이티브(base2
지수) 히스토그램을 지원한다. Prometheus에는 `Summary` 타입도 있는데, 이는 OTel에
직접 대응하는 것이 없다 — 아래 [Summary](#summary)를 참고한다.

Prometheus의 `Histogram`은 오픈텔레메트리의 `Histogram` 계측기에 대응한다.

### 클래식(명시적) 히스토그램 {#classic-explicit-histogram}

두 시스템 모두 고정된 버킷 경계가 관측값을 이산적인 범위로 나누는 클래식
히스토그램을 지원한다.

- **버킷 구성(Bucket configuration)**: Prometheus는 생성 시점에 계측기 자체에
  버킷 경계를 선언한다. 오픈텔레메트리에서는 버킷 경계가 계측기에 힌트로
  설정되며, SDK 수준에서 구성된 뷰(view)로 재정의(override)되거나 대체될 수
  있다. 이러한 분리 덕분에 계측 코드는 수집 구성과 독립적으로 유지된다. 경계가
  지정되지 않고 뷰도 구성되지 않은 경우, SDK는 밀리초 단위 지연 시간에 맞춰
  설계된 기본
  집합(`[0, 5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000, 7500, 10000]`)을
  사용하는데, 이는 초 단위 측정값에는 적절하지 않을 가능성이 높다. 기존
  히스토그램을 마이그레이션할 때는 항상 경계를 제공하거나 뷰를 구성한다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusHistogram.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Histogram;

public class PrometheusHistogram {
  public static void histogramUsage() {
    Histogram deviceCommandDuration =
        Histogram.builder()
            .name("device_command_duration_seconds")
            .help("Time to receive acknowledgment from a smart home device")
            .labelNames("device_type")
            .classicUpperBounds(0.1, 0.25, 0.5, 1.0, 2.5, 5.0)
            .register();

    deviceCommandDuration.labelValues("thermostat").observe(0.35);
    deviceCommandDuration.labelValues("lock").observe(0.85);
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelHistogram.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;
import java.util.List;

public class OtelHistogram {
  // Preallocate attribute keys and, when values are static, entire Attributes objects.
  private static final AttributeKey<String> DEVICE_TYPE = AttributeKey.stringKey("device_type");
  private static final Attributes THERMOSTAT = Attributes.of(DEVICE_TYPE, "thermostat");
  private static final Attributes LOCK = Attributes.of(DEVICE_TYPE, "lock");

  public static void histogramUsage(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // setExplicitBucketBoundariesAdvice() sets default boundaries as a hint to the SDK.
    // Views configured at the SDK level take precedence over this advice.
    DoubleHistogram deviceCommandDuration =
        meter
            .histogramBuilder("device.command.duration")
            .setDescription("Time to receive acknowledgment from a smart home device")
            .setUnit("s")
            .setExplicitBucketBoundariesAdvice(List.of(0.1, 0.25, 0.5, 1.0, 2.5, 5.0))
            .build();

    deviceCommandDuration.record(0.35, THERMOSTAT);
    deviceCommandDuration.record(0.85, LOCK);
  }
}
```

주요 차이점:

- `observe(value)` → `record(value, attributes)`.
- 오픈텔레메트리는 `LongHistogram`(정수, `.ofLongs()`를 통해)과
  `DoubleHistogram`(기본값)을 구분한다. Prometheus는 단일 `Histogram` 타입을
  사용한다.
- 핫 패스에서 호출마다 할당이 발생하지 않도록 `AttributeKey` 인스턴스는 (항상)
  미리 할당하고, `Attributes` 객체는 (값이 정적일 때) 미리 할당한다.
- SDK 뷰는 `setExplicitBucketBoundariesAdvice()`로 설정한 경계를 재정의할 수
  있으며, 속성 필터링, 최소/최대값 기록, 계측기 이름 변경 등 히스토그램 수집의
  다른 측면도 구성할 수 있다.

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_histogram.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

var deviceCommandDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
	Name:    "device_command_duration_seconds",
	Help:    "Time to receive acknowledgment from a smart home device",
	Buckets: []float64{0.1, 0.25, 0.5, 1.0, 2.5, 5.0},
}, []string{"device_type"})

func prometheusHistogramUsage(reg *prometheus.Registry) {
	reg.MustRegister(deviceCommandDuration)

	deviceCommandDuration.WithLabelValues("thermostat").Observe(0.35)
	deviceCommandDuration.WithLabelValues("lock").Observe(0.85)
}
```

OpenTelemetry

<?code-excerpt "otel_histogram.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Preallocate attribute options when values are static to avoid per-call allocation.
var (
	deviceThermostatOpts = []metric.RecordOption{metric.WithAttributes(attribute.String("device_type", "thermostat"))}
	deviceLockOpts       = []metric.RecordOption{metric.WithAttributes(attribute.String("device_type", "lock"))}
)

func otelHistogramUsage(ctx context.Context, meter metric.Meter) {
	// WithExplicitBucketBoundaries sets default boundaries as a hint to the SDK.
	// Views configured at the SDK level take precedence over this hint.
	deviceCommandDuration, err := meter.Float64Histogram("device.command.duration",
		metric.WithDescription("Time to receive acknowledgment from a smart home device"),
		metric.WithUnit("s"),
		metric.WithExplicitBucketBoundaries(0.1, 0.25, 0.5, 1.0, 2.5, 5.0))
	if err != nil {
		panic(err)
	}

	deviceCommandDuration.Record(ctx, 0.35, deviceThermostatOpts...)
	deviceCommandDuration.Record(ctx, 0.85, deviceLockOpts...)
}
```

주요 차이점:

- `Observe(value)` → `Record(ctx, value, metric.WithAttributes(...))`.
- Go에서 `metric.WithExplicitBucketBoundaries(...)`는 슬라이스가 아니라 가변
  인자(variadic)이다. Prometheus는 `HistogramOpts`의 `Buckets` 필드를 사용한다.
- SDK 뷰는 `WithExplicitBucketBoundaries()`로 설정한 경계를 재정의할 수 있으며,
  속성 필터링, 최소/최대값 기록, 계측기 이름 변경 등 히스토그램 수집의 다른
  측면도 구성할 수 있다.

{{% /tab %}} {{< /tabpane >}}

### 네이티브(base2 지수) 히스토그램 {#native-base2-exponential-histogram}

두 시스템 모두 관측된 범위를 다루기 위해 버킷 경계를 자동으로 조정하는, 수동
구성이 필요 없는 네이티브(base2 지수) 히스토그램을 지원한다.

- **형식 선택(Format selection)**: Prometheus 계측기는 클래식 형식만, 네이티브
  형식만, 또는 둘 다 동시에 내보낼 수 있다 — 이를 통해 계측 코드를 변경하지
  않고도 점진적인 마이그레이션이 가능하다. 오픈텔레메트리 에서는 형식 선택이
  계측 코드 밖에서, 즉 익스포터나 뷰를 통해 구성되므로 계측 코드는 어느 쪽이든
  변경할 필요가 없다.
- **계측 코드(Instrumentation code)**: 오픈텔레메트리 계측 코드는 클래식
  히스토그램과 네이티브 히스토그램에 대해 동일하다. 동일한 `record()` 호출이 SDK
  구성 방식에 따라 둘 중 하나의 형식을 만들어낸다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

Prometheus에서는 히스토그램 형식이 계측기 생성 시점에 결정된다. 아래 예제는
네이티브 형식으로 제한하기 위해 `.nativeOnly()`를 사용한다. 이를 생략하면
클래식과 네이티브 형식이 동시에 내보내진다.

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusHistogramNative.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Histogram;

public class PrometheusHistogramNative {
  public static void nativeHistogramUsage() {
    Histogram deviceCommandDuration =
        Histogram.builder()
            .name("device_command_duration_seconds")
            .help("Time to receive acknowledgment from a smart home device")
            .labelNames("device_type")
            .nativeOnly()
            .register();

    deviceCommandDuration.labelValues("thermostat").observe(0.35);
    deviceCommandDuration.labelValues("lock").observe(0.85);
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

Prometheus에서는 `NativeHistogramBucketFactor`를 설정하면 클래식 버킷 구성과
함께 네이티브 히스토그램이 활성화된다 — 두 형식이 동시에 보고된다.

<?code-excerpt "prometheus_histogram_native.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

var nativeDeviceCommandDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
	Name:                        "device_command_duration_seconds",
	Help:                        "Time to receive acknowledgment from a smart home device",
	NativeHistogramBucketFactor: 1.1,
}, []string{"device_type"})

func nativeHistogramUsage(reg *prometheus.Registry) {
	reg.MustRegister(nativeDeviceCommandDuration)

	nativeDeviceCommandDuration.WithLabelValues("thermostat").Observe(0.35)
	nativeDeviceCommandDuration.WithLabelValues("lock").Observe(0.85)
}
```

주요 차이점:

- Go에서 네이티브 히스토그램을 활성화하려면 `NativeHistogramBucketFactor`를
  1.0보다 큰 값으로 설정해야 한다 — 선택 사항이 아니다. 0(제로 값)으로 설정하면
  네이티브 히스토그램이 완전히 비활성화된다. 이 값은 연속된 버킷 경계 사이의
  최대 비율을 제어한다. 값이 작을수록 버킷 수는 늘어나지만 더 미세한 해상도를
  얻는다. 흔히 쓰이는 값인 `1.1`과 비슷한 버킷 밀도를 얻으려면
  `AggregationBase2ExponentialHistogram`에 `MaxScale: 3`을 설정한다.

{{% /tab %}} {{< /tabpane >}}

오픈텔레메트리에서 계측 코드는 클래식 히스토그램의 경우와 동일하다. base2 지수
형식은 계측 레이어 바깥에서 별도로 구성된다.

선호되는 방법은 메트릭 익스포터에서 이를 구성하는 것이다. 이렇게 하면 계측
코드를 건드리지 않고도 해당 익스포터를 통해 내보내지는 모든 히스토그램에
적용된다.

{{< tabpane text=true >}} {{% tab Java %}}

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/OtelHistogramExponentialExporter.java"?>

```java
package otel;

import io.opentelemetry.exporter.otlp.http.metrics.OtlpHttpMetricExporter;
import io.opentelemetry.sdk.metrics.Aggregation;
import io.opentelemetry.sdk.metrics.InstrumentType;
import io.opentelemetry.sdk.metrics.export.DefaultAggregationSelector;

public class OtelHistogramExponentialExporter {
  static OtlpHttpMetricExporter createExporter() {
    // Configure the exporter to use exponential histograms for all histogram instruments.
    // This is the preferred approach — it applies globally without modifying instrumentation code.
    return OtlpHttpMetricExporter.builder()
        .setEndpoint("http://localhost:4318")
        .setDefaultAggregationSelector(
            DefaultAggregationSelector.getDefault()
                .with(InstrumentType.HISTOGRAM, Aggregation.base2ExponentialBucketHistogram()))
        .build();
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>
<?code-excerpt "otel_histogram_exponential_exporter.go" region="createExponentialExporter"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
)

func createExponentialExporter(ctx context.Context) (*otlpmetrichttp.Exporter, error) {
	// Configure the exporter to use exponential histograms for all histogram instruments.
	// This is the preferred approach — it applies globally without modifying instrumentation code.
	return otlpmetrichttp.New(ctx,
		otlpmetrichttp.WithAggregationSelector(func(ik sdkmetric.InstrumentKind) sdkmetric.Aggregation {
			if ik == sdkmetric.InstrumentKindHistogram {
				return sdkmetric.AggregationBase2ExponentialHistogram{}
			}
			return sdkmetric.DefaultAggregationSelector(ik)
		}),
	)
}
```

{{% /tab %}} {{< /tabpane >}}

예를 들어 일부 계측기에는 base2 지수 히스토그램을 사용하고 다른 계측기에는
명시적 버킷을 유지하는 것처럼 더 세밀한 제어가 필요하다면, 대신 뷰를 구성한다.

{{< tabpane text=true >}} {{% tab Java %}}

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/OtelHistogramExponentialView.java"?>

```java
package otel;

import io.opentelemetry.sdk.metrics.Aggregation;
import io.opentelemetry.sdk.metrics.InstrumentSelector;
import io.opentelemetry.sdk.metrics.SdkMeterProvider;
import io.opentelemetry.sdk.metrics.View;

public class OtelHistogramExponentialView {
  static SdkMeterProvider createMeterProvider() {
    // Use a view for per-instrument control — select a specific instrument by name
    // to use exponential histograms while keeping explicit buckets for others.
    return SdkMeterProvider.builder()
        .registerView(
            InstrumentSelector.builder().setName("device.command.duration").build(),
            View.builder().setAggregation(Aggregation.base2ExponentialBucketHistogram()).build())
        .build();
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>
<?code-excerpt "otel_histogram_exponential.go" region="createExponentialView"?>

```go
func createExponentialView() sdkmetric.View {
	// Use a view for per-instrument control — select a specific instrument by name
	// to use exponential histograms while keeping explicit buckets for others.
	return sdkmetric.NewView(
		sdkmetric.Instrument{Name: "device.command.duration"},
		sdkmetric.Stream{Aggregation: sdkmetric.AggregationBase2ExponentialHistogram{}!},
	)
}
```

{{% /tab %}} {{< /tabpane >}}

### Summary {#summary}

Prometheus의 `Summary`는 스크레이핑 시점에 클라이언트 측에서 분위수(quantile)를
계산하고, 이를 레이블이 달린 시계열(예: `{quantile="0.95"}`)로 노출한다.
오픈텔레메트리에는 이에 직접 대응하는 것이 없다.

분위수 추정에는 **base2 지수 히스토그램** 이 권장되는 대안이다. 이는 관측된
범위를 다루기 위해 버킷 경계를 자동으로 조정하며, PromQL의
`histogram_quantile()`이 쿼리 시점에 오차 범위가 정해진 분위수를 계산할 수 있다.
`Summary`와 달리 그 결과는 인스턴스 전반에 걸쳐 집계할 수 있다.
[네이티브(base2 지수) 히스토그램](#native-base2-exponential-histogram)을
참고한다.

분위수가 아니라 개수와 합계만 필요하다면, 명시적 버킷 경계가 없는 히스토그램으로
최소한의 오버헤드로 그 통계값을 얻을 수 있다. 아래 예제는 이 더 단순한 방식을
보여준다.

{{< tabpane text=true >}} {{% tab Java %}}

Prometheus

<?code-excerpt path-base="examples/java/prometheus-compatibility"?>
<?code-excerpt "src/main/java/otel/PrometheusSummary.java"?>

```java
package otel;

import io.prometheus.metrics.core.metrics.Summary;

public class PrometheusSummary {
  public static void summaryUsage() {
    Summary deviceCommandDuration =
        Summary.builder()
            .name("device_command_duration_seconds")
            .help("Time to receive acknowledgment from a smart home device")
            .labelNames("device_type")
            .quantile(0.5, 0.05)
            .quantile(0.95, 0.01)
            .quantile(0.99, 0.001)
            .register();

    deviceCommandDuration.labelValues("thermostat").observe(0.35);
    deviceCommandDuration.labelValues("lock").observe(0.85);
  }
}
```

OpenTelemetry

<?code-excerpt "src/main/java/otel/OtelHistogramAsSummary.java"?>

```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;
import java.util.List;

public class OtelHistogramAsSummary {
  private static final AttributeKey<String> DEVICE_TYPE = AttributeKey.stringKey("device_type");
  private static final Attributes THERMOSTAT = Attributes.of(DEVICE_TYPE, "thermostat");
  private static final Attributes LOCK = Attributes.of(DEVICE_TYPE, "lock");

  public static void summaryReplacement(OpenTelemetry openTelemetry) {
    Meter meter = openTelemetry.getMeter("smart.home");
    // No explicit bucket boundaries: captures count and sum, a good stand-in for most
    // Summary use cases. For quantile estimation, add boundaries that bracket your thresholds.
    DoubleHistogram deviceCommandDuration =
        meter
            .histogramBuilder("device.command.duration")
            .setDescription("Time to receive acknowledgment from a smart home device")
            .setUnit("s")
            .setExplicitBucketBoundariesAdvice(List.of())
            .build();

    deviceCommandDuration.record(0.35, THERMOSTAT);
    deviceCommandDuration.record(0.85, LOCK);
  }
}
```

{{% /tab %}} {{% tab Go %}}

<?code-excerpt path-base="examples/go/prometheus-compatibility"?>

Prometheus

<?code-excerpt "prometheus_summary.go"?>

```go
package main

import "github.com/prometheus/client_golang/prometheus"

var summaryDeviceCommandDuration = prometheus.NewSummaryVec(prometheus.SummaryOpts{
	Name:       "device_command_duration_seconds",
	Help:       "Time to receive acknowledgment from a smart home device",
	Objectives: map[float64]float64{0.5: 0.05, 0.95: 0.01, 0.99: 0.001},
}, []string{"device_type"})

func summaryUsage(reg *prometheus.Registry) {
	reg.MustRegister(summaryDeviceCommandDuration)

	summaryDeviceCommandDuration.WithLabelValues("thermostat").Observe(0.35)
	summaryDeviceCommandDuration.WithLabelValues("lock").Observe(0.85)
}
```

OpenTelemetry

<?code-excerpt "otel_histogram_as_summary.go"?>

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Preallocate attribute options when values are static to avoid per-call allocation.
var (
	summaryThermostatOpts = []metric.RecordOption{metric.WithAttributes(attribute.String("device_type", "thermostat"))}
	summaryLockOpts       = []metric.RecordOption{metric.WithAttributes(attribute.String("device_type", "lock"))}
)

func summaryReplacement(ctx context.Context, meter metric.Meter) {
	// No explicit bucket boundaries: captures count and sum only.
	// For quantile estimation, prefer a base2 exponential histogram instead.
	deviceCommandDuration, err := meter.Float64Histogram("device.command.duration",
		metric.WithDescription("Time to receive acknowledgment from a smart home device"),
		metric.WithUnit("s"),
		metric.WithExplicitBucketBoundaries()) // no boundaries
	if err != nil {
		panic(err)
	}

	deviceCommandDuration.Record(ctx, 0.35, summaryThermostatOpts...)
	deviceCommandDuration.Record(ctx, 0.85, summaryLockOpts...)
}
```

{{% /tab %}} {{< /tabpane >}}
