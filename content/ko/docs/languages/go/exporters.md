---
title: 익스포터
aliases: [exporting_data]
weight: 50
# prettier-ignore
cSpell:ignore: autoexport otlplog otlploggrpc otlploghttp otlpmetric otlpmetricgrpc otlpmetrichttp otlptrace otlptracegrpc otlptracehttp sdkmetric sdktrace stdoutlog stdouttrace
default_lang_commit: 6894a64b96cdd29faea3b89ea3821fe77a5c299a
---

{{% docs/languages/exporters/intro %}}

## 환경 변수를 사용한 자동 익스포터 구성 {#automatic-exporter-configuration-with-environment-variables}

[`go.opentelemetry.io/contrib/exporters/autoexport`](https://pkg.go.dev/go.opentelemetry.io/contrib/exporters/autoexport)
패키지를 사용해
[표준 오픈텔레메트리(OpenTelemetry) 환경 변수](/docs/specs/otel/configuration/sdk-environment-variables/)로
익스포터를 자동으로 구성할 수 있다.

이 패키지는 런타임에 적절한 익스포터를 선택하고 초기화하기 위해 **익스포터
선택자(exporter selector)** 환경 변수를 읽는 팩토리 함수를 제공한다.

| 함수                                                                                                     | 환경 변수               | 설명                         |
| -------------------------------------------------------------------------------------------------------- | ----------------------- | ---------------------------- |
| [`NewSpanExporter`](https://pkg.go.dev/go.opentelemetry.io/contrib/exporters/autoexport#NewSpanExporter) | `OTEL_TRACES_EXPORTER`  | 트레이스 익스포터를 생성한다 |
| [`NewMetricReader`](https://pkg.go.dev/go.opentelemetry.io/contrib/exporters/autoexport#NewMetricReader) | `OTEL_METRICS_EXPORTER` | 메트릭 리더를 생성한다       |
| [`NewLogExporter`](https://pkg.go.dev/go.opentelemetry.io/contrib/exporters/autoexport#NewLogExporter)   | `OTEL_LOGS_EXPORTER`    | 로그 익스포터를 생성한다     |

선택자 변수에서 지원하는 값은 `otlp`(기본값)와 `none`이다.
`OTEL_METRICS_EXPORTER`의 경우 `prometheus`도 지원된다. 익스포터가 선택되면, 그
구성(엔드포인트, 헤더, 타임아웃, 프로토콜 등)은 기반이 되는 OTLP 익스포터
패키지가 표준
[OTLP 익스포터 환경 변수](/docs/languages/sdk-configuration/otlp-exporter/)에서
읽어들인다.

사용 예시:

```go
import (
	"context"

	"go.opentelemetry.io/contrib/exporters/autoexport"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

func main() {
	ctx := context.Background()

	// Create trace exporter using environment variables
	spanExporter, err := autoexport.NewSpanExporter(ctx)
	if err != nil {
		// handle error
	}

	// Create trace provider with the exporter
	tracerProvider := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(spanExporter),
	)

	// Create metric reader using environment variables
	metricReader, err := autoexport.NewMetricReader(ctx)
	if err != nil {
		// handle error
	}

	// Create meter provider with the reader
	meterProvider := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(metricReader),
	)
}
```

> [!NOTE]
>
> 표준 OTLP 익스포터 패키지(`otlptracegrpc`, `otlptracehttp` 등)는 이미
> `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`,
> `OTEL_EXPORTER_OTLP_TIMEOUT`, `OTEL_EXPORTER_OTLP_COMPRESSION`과 같은 대부분의
> OTLP 환경 변수를 읽어들인다.
>
> `autoexport` 패키지는 어떤 익스포터 구현체를 사용할지 선택하는 **익스포터
> 선택자 변수**(`OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`,
> `OTEL_LOGS_EXPORTER`)에 대한 지원을 추가한다. 이러한 분리 덕분에 명시적으로
> 임포트하지 않는 한 (gRPC와 같은) 익스포터 의존성을 번들로 포함하지 않아
> 바이너리 크기를 더 작게 유지할 수 있다.
>
> 또한 `OTEL_SDK_DISABLED`는 현재 Go SDK에서 지원되지 않는다는 점에 유의한다.

Go SDK와 contrib 패키지에서 어떤 환경 변수를 지원하는지에 대한 전체 개요는
[오픈텔레메트리 명세 준수 매트릭스](https://github.com/open-telemetry/opentelemetry-specification/blob/main/spec-compliance-matrix.md)를
참고한다.

## 콘솔 {#console}

콘솔 익스포터는 개발 및 디버깅 작업에 유용하며, 설정하기 가장 간단하다.

### 콘솔 트레이스 {#console-traces}

[`go.opentelemetry.io/otel/exporters/stdout/stdouttrace`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/stdout/stdouttrace)
패키지에는 콘솔 트레이스 익스포터의 구현체가 들어 있다.

### 콘솔 메트릭 {#console-metrics}

[`go.opentelemetry.io/otel/exporters/stdout/stdoutmetric`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/stdout/stdoutmetric)
패키지에는 콘솔 메트릭 익스포터의 구현체가 들어 있다.

### 콘솔 로그(실험적) {#console-logs}

[`go.opentelemetry.io/otel/exporters/stdout/stdoutlog`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/stdout/stdoutlog)
패키지에는 콘솔 로그 익스포터의 구현체가 들어 있다.

## OTLP {#otlp}

트레이스 데이터를 OTLP 엔드포인트([컬렉터](/docs/collector) 또는 Jaeger v1.35.0
이상 등)로 보내려면, 해당 엔드포인트로 전송하는 OTLP 익스포터를 구성해야 한다.

### HTTP를 통한 OTLP 트레이스 {#otlp-traces-over-http}

[`go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp)에는
바이너리 protobuf 페이로드와 함께 HTTP를 사용하는 OTLP 트레이스 익스포터의
구현체가 들어 있다.

### gRPC를 통한 OTLP 트레이스 {#otlp-traces-over-grpc}

[`go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc)에는
gRPC를 사용하는 OTLP 트레이스 익스포터의 구현체가 들어 있다.

### Jaeger {#jaeger}

OTLP 익스포터를 사용해 보려면, v1.35.0부터
[Jaeger](https://www.jaegertracing.io/)를 Docker 컨테이너에서 OTLP
엔드포인트이자 트레이스 시각화 도구로 실행할 수 있다.

```shell
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/jaeger:latest
```

### HTTP를 통한 OTLP 메트릭 {#otlp-metrics-over-http}

[`go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp)에는
바이너리 protobuf 페이로드와 함께 HTTP를 사용하는 OTLP 메트릭 익스포터의
구현체가 들어 있다.

### gRPC를 통한 OTLP 메트릭 {#otlp-metrics-over-grpc}

[`go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc)에는
gRPC를 사용하는 OTLP 메트릭 익스포터의 구현체가 들어 있다.

### HTTP를 통한 OTLP 로그(실험적) {#otlp-logs-over-http-experimental}

[`go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp)에는
바이너리 protobuf 페이로드와 함께 HTTP를 사용하는 OTLP 로그 익스포터의 구현체가
들어 있다.

### gRPC를 통한 OTLP 로그(실험적) {#otlp-logs-over-grpc-experimental}

[`go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploggrpc`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploggrpc)에는
gRPC를 사용하는 OTLP 로그 익스포터의 구현체가 들어 있다.

## Prometheus(실험적) {#prometheus-experimental}

Prometheus 익스포터는 Prometheus 스크레이프(scrape) HTTP 엔드포인트를 통해
메트릭을 보고하는 데 사용된다.

[`go.opentelemetry.io/otel/exporters/prometheus`](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/prometheus)에는
Prometheus 메트릭 익스포터의 구현체가 들어 있다.

Prometheus 익스포터 사용 방법을 더 자세히 알아보려면
[prometheus 예제](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/examples/prometheus)를
시도해 본다.
