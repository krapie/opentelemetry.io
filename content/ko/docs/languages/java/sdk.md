---
title: SDK로 텔레메트리 관리하기
weight: 12
cSpell:ignore: autoconfigured data_point FQCNs inflight Interceptable okhttp
default_lang_commit: 8d6b626b3dd798de9065335d8c4cc0912959c484
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/configuration"?>

SDK는 [API](../api/)의 내장 참조 구현체로, 계측 API 호출로 생성된 텔레메트리를
처리하고 내보낸다. 이 페이지는 설명, 관련 Javadoc 링크, 아티팩트 좌표,
프로그래밍 방식 구성 예시 등을 포함하는 SDK에 대한 개념적 개요이다.
[제로 코드 SDK 자동구성](../configuration/#zero-code-sdk-autoconfigure)을 포함한
SDK 구성에 대한 자세한 내용은 **[SDK 구성하기](../configuration/)** 를 참고한다.

SDK는 다음과 같은 최상위 구성 요소로 이루어진다.

- [SdkTracerProvider](#sdktracerprovider): 샘플링, 처리, 스팬 내보내기를 위한
  도구를 포함한 `TracerProvider`의 SDK 구현체이다.
- [SdkMeterProvider](#sdkmeterprovider): 메트릭 스트림 구성 및 메트릭 읽기/
  내보내기를 위한 도구를 포함한 `MeterProvider`의 SDK 구현체이다.
- [SdkLoggerProvider](#sdkloggerprovider): 로그 처리 및 내보내기를 위한 도구를
  포함한 `LoggerProvider`의 SDK 구현체이다.
- [TextMapPropagator](#textmappropagator): 프로세스 경계를 넘어 컨텍스트를
  전파한다.

이들은 [OpenTelemetrySdk](#opentelemetrysdk)로 결합되며, 이는 완전히 구성된
[SDK 구성 요소](#sdk-components)를 계측에 전달하기 편리하게 해주는 캐리어
객체이다.

SDK는 많은 사용 사례에 충분한 다양한 내장 구성 요소와 함께 제공되며, 확장성을
위한 [플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)를 지원한다.

## SDK 플러그인 확장 인터페이스 {#sdk-plugin-extension-interfaces}

내장 구성 요소로 충분하지 않을 때, 다양한 플러그인 확장 인터페이스를 구현하여
SDK를 확장할 수 있다.

- [Sampler](#sampler): 어떤 스팬이 기록되고 샘플링되는지 구성한다.
- [SpanProcessor](#spanprocessor): 스팬이 시작되고 끝날 때 이를 처리한다.
- [SpanExporter](#spanexporter): 스팬을 프로세스 밖으로 내보낸다.
- [MetricReader](#metricreader): 집계된 메트릭을 읽는다.
- [MetricExporter](#metricexporter): 메트릭을 프로세스 밖으로 내보낸다.
- [LogRecordProcessor](#logrecordprocessor): 로그 레코드가 내보내질 때 이를
  처리한다.
- [LogRecordExporter](#logrecordexporter): 로그 레코드를 프로세스 밖으로
  내보낸다.
- [TextMapPropagator](#textmappropagator): 프로세스 경계를 넘어 컨텍스트를
  전파한다.

## SDK 구성 요소 {#sdk-components}

`io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}` 아티팩트는
오픈텔레메트리 SDK를 담고 있다.

다음 절에서는 SDK의 핵심 사용자 대상 구성 요소에 대해 설명한다. 각 구성 요소
절은 다음을 포함한다.

- Javadoc 타입 참조 링크를 포함한 간단한 설명.
- 해당 구성 요소가
  [플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)인 경우, 사용
  가능한 내장 구현체 및 `opentelemetry-java-contrib` 구현체의 표.
- [프로그래밍 방식 구성](../configuration/#programmatic-configuration)에 대한
  간단한 시연.
- 해당 구성 요소가
  [플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)인 경우, 커스텀
  구현체에 대한 간단한 시연.

### OpenTelemetrySdk {#opentelemetrysdk}

[OpenTelemetrySdk](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk/latest/io/opentelemetry/sdk/OpenTelemetrySdk.html)는
[OpenTelemetry](../api/#opentelemetry)의 SDK 구현체이다. 이는 최상위 SDK 구성
요소를 담는 홀더로, 완전히 구성된 SDK 구성 요소를 계측에 전달하기 편리하게
해준다.

`OpenTelemetrySdk`는 애플리케이션 소유자가 구성하며, 다음으로 이루어진다.

- [SdkTracerProvider](#sdktracerprovider): `TracerProvider`의 SDK 구현체이다.
- [SdkMeterProvider](#sdkmeterprovider): `MeterProvider`의 SDK 구현체이다.
- [SdkLoggerProvider](#sdkloggerprovider): `LoggerProvider`의 SDK 구현체이다.
- [ContextPropagators](#textmappropagator): 프로세스 경계를 넘어 컨텍스트를
  전파한다.

다음 코드 스니펫은 `OpenTelemetrySdk`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/OpenTelemetrySdkConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;

public class OpenTelemetrySdkConfig {
  public static OpenTelemetrySdk create() {
    Resource resource = ResourceConfig.create();
    return OpenTelemetrySdk.builder()
        .setTracerProvider(SdkTracerProviderConfig.create(resource))
        .setMeterProvider(SdkMeterProviderConfig.create(resource))
        .setLoggerProvider(SdkLoggerProviderConfig.create(resource))
        .setPropagators(ContextPropagatorsConfig.create())
        .build();
  }
}
```
<!-- prettier-ignore-end -->

### 리소스(Resource) {#resource}

[Resource](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-common/latest/io/opentelemetry/sdk/resources/Resource.html)는
텔레메트리 소스를 정의하는 속성의 집합이다. 애플리케이션은 동일한 리소스를
[SdkTracerProvider](#sdktracerprovider), [SdkMeterProvider](#sdkmeterprovider),
[SdkLoggerProvider](#sdkloggerprovider)와 연관지어야 한다.

> [!NOTE]
>
> [ResourceProvider](../configuration/#resourceprovider)는 환경에 기반하여
> [자동구성된](../configuration/#zero-code-sdk-autoconfigure) 리소스에 컨텍스트
> 정보를 기여한다. 사용 가능한 `ResourceProvider`의 목록은 문서를 참고한다.

다음 코드 스니펫은 `Resource`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ResourceConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.ServiceAttributes;

public class ResourceConfig {
  public static Resource create() {
    return Resource.getDefault().toBuilder()
        .put(ServiceAttributes.SERVICE_NAME, "my-service")
        .build();
  }
}
```
<!-- prettier-ignore-end -->

### SDK 트레이서 프로바이더(SdkTracerProvider) {#sdktracerprovider}

[SdkTracerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-trace/latest/io/opentelemetry/sdk/trace/SdkTracerProvider.html)는
[TracerProvider](../api/#tracerprovider)의 SDK 구현체이며, API가 생성한 트레이스
텔레메트리를 처리하는 역할을 한다.

`SdkTracerProvider`는 애플리케이션 소유자가 구성하며, 다음으로 이루어진다.

- [Resource](#resource): 스팬이 연관되는 리소스이다.
- [Sampler](#sampler): 어떤 스팬이 기록되고 샘플링되는지 구성한다.
- [SpanProcessor](#spanprocessor): 스팬이 시작되고 끝날 때 이를 처리한다.
- [SpanExporter](#spanexporter): (연관된 `SpanProcessor`와 함께) 스팬을 프로세스
  밖으로 내보낸다.
- [SpanLimits](#spanlimits): 스팬과 연관된 데이터의 제한을 제어한다.

다음 코드 스니펫은 `SdkTracerProvider`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SdkTracerProviderConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;

public class SdkTracerProviderConfig {
  public static SdkTracerProvider create(Resource resource) {
    return SdkTracerProvider.builder()
        .setResource(resource)
        .addSpanProcessor(
            SpanProcessorConfig.batchSpanProcessor(
                SpanExporterConfig.otlpHttpSpanExporter("http://localhost:4318/v1/spans")))
        .setSampler(SamplerConfig.parentBasedSampler(SamplerConfig.traceIdRatioBased(.25)))
        .setSpanLimits(SpanLimitsConfig::spanLimits)
        .build();
  }
}
```
<!-- prettier-ignore-end -->

#### 샘플러(Sampler) {#sampler}

[Sampler](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-trace/latest/io/opentelemetry/sdk/trace/samplers/Sampler.html)는
어떤 스팬이 기록되고 샘플링되는지 결정하는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다.

> [!NOTE]
>
> 기본적으로 `SdkTracerProvider`는 `ParentBased(root=AlwaysOn)` 샘플러로
> 구성된다. 이로 인해 호출하는 애플리케이션이 샘플링을 수행하지 않는 한 100%의
> 스팬이 샘플링된다. 이것이 너무 시끄럽거나 비용이 많이 든다면, 샘플러를
> 변경한다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
샘플러:

| 클래스                    | 아티팩트                                                                                      | 설명                                                                                                                                 |
| ------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `ParentBased`             | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                  | 스팬 부모의 샘플링 상태에 기반하여 스팬을 샘플링한다.                                                                                |
| `AlwaysOn`                | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                  | 모든 스팬을 샘플링한다.                                                                                                              |
| `AlwaysOff`               | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                  | 모든 스팬을 드롭한다.                                                                                                                |
| `TraceIdRatioBased`       | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                  | 구성 가능한 비율에 기반하여 스팬을 샘플링한다.                                                                                       |
| `JaegerRemoteSampler`     | `io.opentelemetry:opentelemetry-sdk-extension-jaeger-remote-sampler:{{% param vers.otel %}}`  | 원격 서버의 구성에 기반하여 스팬을 샘플링한다.                                                                                       |
| `LinksBasedSampler`       | `io.opentelemetry.contrib:opentelemetry-samplers:{{% param vers.contrib %}}-alpha`            | 스팬 링크의 샘플링 상태에 기반하여 스팬을 샘플링한다.                                                                                |
| `RuleBasedRoutingSampler` | `io.opentelemetry.contrib:opentelemetry-samplers:{{% param vers.contrib %}}-alpha`            | 구성 가능한 규칙에 기반하여 스팬을 샘플링한다.                                                                                       |
| `ConsistentSamplers`      | `io.opentelemetry.contrib:opentelemetry-consistent-sampling:{{% param vers.contrib %}}-alpha` | [확률 샘플링](/docs/specs/otel/trace/tracestate-probability-sampling/)에 정의된 다양한 일관성 샘플러(consistent sampler) 구현체이다. |

다음 코드 스니펫은 `Sampler`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SamplerConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.extension.trace.jaeger.sampler.JaegerRemoteSampler;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import java.time.Duration;

public class SamplerConfig {
  public static Sampler parentBasedSampler(Sampler root) {
    return Sampler.parentBasedBuilder(root)
        .setLocalParentNotSampled(Sampler.alwaysOff())
        .setLocalParentSampled(Sampler.alwaysOn())
        .setRemoteParentNotSampled(Sampler.alwaysOff())
        .setRemoteParentSampled(Sampler.alwaysOn())
        .build();
  }

  public static Sampler alwaysOn() {
    return Sampler.alwaysOn();
  }

  public static Sampler alwaysOff() {
    return Sampler.alwaysOff();
  }

  public static Sampler traceIdRatioBased(double ratio) {
    return Sampler.traceIdRatioBased(ratio);
  }

  public static Sampler jaegerRemoteSampler() {
    return JaegerRemoteSampler.builder()
        .setInitialSampler(Sampler.alwaysOn())
        .setEndpoint("http://endpoint")
        .setPollingInterval(Duration.ofSeconds(60))
        .setServiceName("my-service-name")
        .build();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 샘플링 로직을 제공하려면 `Sampler` 인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomSampler.java"?>
```java
package otel;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.context.Context;
import io.opentelemetry.sdk.trace.data.LinkData;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.sdk.trace.samplers.SamplingResult;
import java.util.List;

public class CustomSampler implements Sampler {
  @Override
  public SamplingResult shouldSample(
      Context parentContext,
      String traceId,
      String name,
      SpanKind spanKind,
      Attributes attributes,
      List<LinkData> parentLinks) {
    // Callback invoked when span is started, before any SpanProcessor is called.
    // If the SamplingDecision is:
    // - DROP: the span is dropped. A valid span context is created and SpanProcessor#onStart is
    // still called, but no data is recorded and SpanProcessor#onEnd is not called.
    // - RECORD_ONLY: the span is recorded but not sampled. Data is recorded to the span,
    // SpanProcessor#onStart and SpanProcessor#onEnd are called, but the span's sampled status
    // indicates it should not be exported out of process.
    // - RECORD_AND_SAMPLE: the span is recorded and sampled. Data is recorded to the span,
    // SpanProcessor#onStart and SpanProcessor#onEnd are called, and the span's sampled status
    // indicates it should be exported out of process.
    return SpanKind.SERVER == spanKind ? SamplingResult.recordAndSample() : SamplingResult.drop();
  }

  @Override
  public String getDescription() {
    // Return a description of the sampler.
    return this.getClass().getSimpleName();
  }
}
```
<!-- prettier-ignore-end -->

#### 스팬 프로세서(SpanProcessor) {#spanprocessor}

[SpanProcessor](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-trace/latest/io/opentelemetry/sdk/trace/SpanProcessor.html)는
스팬이 시작되고 끝날 때 호출되는 콜백을 가진
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다. 이는 스팬을
프로세스 밖으로 내보내기 위해 [SpanExporter](#spanexporter)와 함께 사용되는
경우가 많지만, 데이터 보강과 같은 다른 용도로도 사용된다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
스팬 프로세서:

| 클래스                    | 아티팩트                                                                                    | 설명                                                                    |
| ------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `BatchSpanProcessor`      | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                | 샘플링된 스팬을 배치로 묶어 구성 가능한 `SpanExporter`를 통해 내보낸다. |
| `SimpleSpanProcessor`     | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                                | 각 샘플링된 스팬을 구성 가능한 `SpanExporter`를 통해 내보낸다.          |
| `BaggageSpanProcessor`    | `io.opentelemetry.contrib:opentelemetry-baggage-processor:{{% param vers.contrib %}}-alpha` | 배기지로 스팬을 보강한다.                                               |
| `JfrSpanProcessor`        | `io.opentelemetry.contrib:opentelemetry-jfr-events:{{% param vers.contrib %}}-alpha`        | 스팬으로부터 JFR 이벤트를 생성한다.                                     |
| `StackTraceSpanProcessor` | `io.opentelemetry.contrib:opentelemetry-span-stacktrace:{{% param vers.contrib %}}-alpha`   | 선택된 스팬을 스택 트레이스 데이터로 보강한다.                          |
| `InferredSpansProcessor`  | `io.opentelemetry.contrib:opentelemetry-inferred-spans:{{% param vers.contrib %}}-alpha`    | 계측 대신 비동기 프로파일러(async profiler)로부터 스팬을 생성한다.      |

다음 코드 스니펫은 `SpanProcessor`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SpanProcessorConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.trace.SpanProcessor;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import java.time.Duration;

public class SpanProcessorConfig {
  public static SpanProcessor batchSpanProcessor(SpanExporter spanExporter) {
    return BatchSpanProcessor.builder(spanExporter)
        .setMaxQueueSize(2048)
        .setExporterTimeout(Duration.ofSeconds(30))
        .setScheduleDelay(Duration.ofSeconds(5))
        .build();
  }

  public static SpanProcessor simpleSpanProcessor(SpanExporter spanExporter) {
    return SimpleSpanProcessor.builder(spanExporter).build();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 스팬 처리 로직을 제공하려면 `SpanProcessor` 인터페이스를 구현한다. 예를
들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomSpanProcessor.java"?>
```java
package otel;

import io.opentelemetry.context.Context;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.ReadWriteSpan;
import io.opentelemetry.sdk.trace.ReadableSpan;
import io.opentelemetry.sdk.trace.SpanProcessor;

public class CustomSpanProcessor implements SpanProcessor {

  @Override
  public void onStart(Context parentContext, ReadWriteSpan span) {
    // Callback invoked when span is started.
    // Enrich the record with a custom attribute.
    span.setAttribute("my.custom.attribute", "hello world");
  }

  @Override
  public boolean isStartRequired() {
    // Indicate if onStart should be called.
    return true;
  }

  @Override
  public void onEnd(ReadableSpan span) {
    // Callback invoked when span is ended.
  }

  @Override
  public boolean isEndRequired() {
    // Indicate if onEnd should be called.
    return false;
  }

  @Override
  public CompletableResultCode shutdown() {
    // Optionally shutdown the processor and cleanup any resources.
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode forceFlush() {
    // Optionally process any records which have been queued up but not yet processed.
    return CompletableResultCode.ofSuccess();
  }
}
```
<!-- prettier-ignore-end -->

#### 스팬 익스포터(SpanExporter) {#spanexporter}

[SpanExporter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-trace/latest/io/opentelemetry/sdk/trace/export/SpanExporter.html)는
스팬을 프로세스 밖으로 내보내는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다.
`SdkTracerProvider`에 직접 등록되는 대신, [SpanProcessor](#spanprocessor)(주로
`BatchSpanProcessor`)와 짝을 이룬다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
스팬 익스포터:

| 클래스                         | 아티팩트                                                                                 | 설명                                                                                    |
| ------------------------------ | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `OtlpHttpSpanExporter` **[1]** | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`                   | OTLP `http/protobuf`를 통해 스팬을 내보낸다.                                            |
| `OtlpGrpcSpanExporter` **[1]** | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`                   | OTLP `grpc`를 통해 스팬을 내보낸다.                                                     |
| `LoggingSpanExporter`          | `io.opentelemetry:opentelemetry-exporter-logging:{{% param vers.otel %}}`                | 디버깅 형식으로 JUL에 스팬을 로깅한다.                                                  |
| `OtlpJsonLoggingSpanExporter`  | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`           | OTLP JSON 인코딩으로 JUL에 스팬을 로깅한다.                                             |
| `OtlpStdoutSpanExporter`       | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`           | OTLP [JSON 파일 인코딩][JSON file encoding](실험적)으로 `System.out`에 스팬을 로깅한다. |
| `ZipkinSpanExporter`           | `io.opentelemetry:opentelemetry-exporter-zipkin:{{% param vers.otel %}}`                 | Zipkin으로 스팬을 내보낸다.                                                             |
| `InterceptableSpanExporter`    | `io.opentelemetry.contrib:opentelemetry-processors:{{% param vers.contrib %}}-alpha`     | 내보내기 전에 유연한 인터셉터로 스팬을 전달한다.                                        |
| `KafkaSpanExporter`            | `io.opentelemetry.contrib:opentelemetry-kafka-exporter:{{% param vers.contrib %}}-alpha` | Kafka 토픽에 기록하여 스팬을 내보낸다.                                                  |

**[1]**: 구현 세부 사항은 [OTLP 익스포터](#otlp-exporters)를 참고한다.

다음 코드 스니펫은 `SpanExporter`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SpanExporterConfig.java"?>
```java
package otel;

import io.opentelemetry.exporter.logging.LoggingSpanExporter;
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingSpanExporter;
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import java.time.Duration;

public class SpanExporterConfig {
  public static SpanExporter otlpHttpSpanExporter(String endpoint) {
    return OtlpHttpSpanExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static SpanExporter otlpGrpcSpanExporter(String endpoint) {
    return OtlpGrpcSpanExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static SpanExporter logginSpanExporter() {
    return LoggingSpanExporter.create();
  }

  public static SpanExporter otlpJsonLoggingSpanExporter() {
    return OtlpJsonLoggingSpanExporter.create();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 스팬 내보내기 로직을 제공하려면 `SpanExporter` 인터페이스를 구현한다.
예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomSpanExporter.java"?>
```java
package otel;

import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import java.util.Collection;
import java.util.logging.Level;
import java.util.logging.Logger;

public class CustomSpanExporter implements SpanExporter {

  private static final Logger logger = Logger.getLogger(CustomSpanExporter.class.getName());

  @Override
  public CompletableResultCode export(Collection<SpanData> spans) {
    // Export the records. Typically, records are sent out of process via some network protocol, but
    // we simply log for illustrative purposes.
    logger.log(Level.INFO, "Exporting spans");
    spans.forEach(span -> logger.log(Level.INFO, "Span: " + span));
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode flush() {
    // Export any records which have been queued up but not yet exported.
    logger.log(Level.INFO, "flushing");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode shutdown() {
    // Shutdown the exporter and cleanup any resources.
    logger.log(Level.INFO, "shutting down");
    return CompletableResultCode.ofSuccess();
  }
}
```
<!-- prettier-ignore-end -->

#### 스팬 제한(SpanLimits) {#spanlimits}

[SpanLimits](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-trace/latest/io/opentelemetry/sdk/trace/SpanLimits.html)는
최대 속성 길이, 최대 속성 개수 등 스팬이 캡처하는 데이터에 대한 제약을 정의한다.

다음 코드 스니펫은 `SpanLimits`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SpanLimitsConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.trace.SpanLimits;

public class SpanLimitsConfig {
  public static SpanLimits spanLimits() {
    return SpanLimits.builder()
        .setMaxNumberOfAttributes(128)
        .setMaxAttributeValueLength(1024)
        .setMaxNumberOfLinks(128)
        .setMaxNumberOfAttributesPerLink(128)
        .setMaxNumberOfEvents(128)
        .setMaxNumberOfAttributesPerEvent(128)
        .build();
  }
}
```
<!-- prettier-ignore-end -->

### SDK 미터 프로바이더(SdkMeterProvider) {#sdkmeterprovider}

[SdkMeterProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-metrics/latest/io/opentelemetry/sdk/metrics/SdkMeterProvider.html)는
[MeterProvider](../api/#meterprovider)의 SDK 구현체이며, API가 생성한 메트릭
텔레메트리를 처리하는 역할을 한다.

`SdkMeterProvider`는 애플리케이션 소유자가 구성하며, 다음으로 이루어진다.

- [Resource](#resource): 메트릭이 연관되는 리소스이다.
- [MetricReader](#metricreader): 메트릭의 집계된 상태를 읽는다.
  - 선택적으로,
    [CardinalityLimitSelector](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-metrics/latest/io/opentelemetry/sdk/metrics/export/CardinalityLimitSelector.html)로
    계측기 종류별 카디널리티 제한을 재정의할 수 있다. 설정하지 않으면, 각
    계측기는 수집 주기당 속성 조합 2000개로 제한된다. 카디널리티 제한은
    [뷰](#views)를 통해 개별 계측기에 대해서도 구성할 수 있다. 자세한 내용은
    [카디널리티 제한](/docs/specs/otel/metrics/sdk/#cardinality-limits)을
    참고한다.
- [MetricExporter](#metricexporter): (연관된 `MetricReader`와 함께) 메트릭을
  프로세스 밖으로 내보낸다.
- [Views](#views): 사용하지 않는 메트릭을 드롭하는 것을 포함하여 메트릭 스트림을
  구성한다.

다음 코드 스니펫은 `SdkMeterProvider`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SdkMeterProviderConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.metrics.SdkMeterProvider;
import io.opentelemetry.sdk.metrics.SdkMeterProviderBuilder;
import io.opentelemetry.sdk.resources.Resource;
import java.util.List;
import java.util.Set;

public class SdkMeterProviderConfig {
  public static SdkMeterProvider create(Resource resource) {
    SdkMeterProviderBuilder builder =
        SdkMeterProvider.builder()
            .setResource(resource)
            .registerMetricReader(
                MetricReaderConfig.periodicMetricReader(
                    MetricExporterConfig.otlpHttpMetricExporter(
                        "http://localhost:4318/v1/metrics")));
    // Uncomment to optionally register metric reader with cardinality limits
    // builder.registerMetricReader(
    //     MetricReaderConfig.periodicMetricReader(
    //         MetricExporterConfig.otlpHttpMetricExporter("http://localhost:4318/v1/metrics")),
    //     instrumentType -> 100);

    ViewConfig.dropMetricView(builder, "some.custom.metric");
    ViewConfig.histogramBucketBoundariesView(
        builder, "http.server.request.duration", List.of(1.0, 5.0, 10.0));
    ViewConfig.attributeFilterView(
        builder, "http.client.request.duration", Set.of("http.request.method"));
    ViewConfig.cardinalityLimitsView(builder, "http.server.active_requests", 100);
    return builder.build();
  }
}
```
<!-- prettier-ignore-end -->

#### 메트릭 리더(MetricReader) {#metricreader}

[MetricReader](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-metrics/latest/io/opentelemetry/sdk/metrics/export/MetricReader.html)는
집계된 메트릭을 읽는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다. 메트릭을
프로세스 밖으로 내보내기 위해 [MetricExporter](#metricexporter)와 함께 사용되는
경우가 많지만, 풀 기반(pull-based) 프로토콜로 외부 스크레이퍼에 메트릭을
제공하는 데 사용되기도 한다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
메트릭 리더:

| 클래스                 | 아티팩트                                                                           | 설명                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `PeriodicMetricReader` | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                       | 주기적으로 메트릭을 읽어 구성 가능한 `MetricExporter`를 통해 내보낸다. |
| `PrometheusHttpServer` | `io.opentelemetry:opentelemetry-exporter-prometheus:{{% param vers.otel %}}-alpha` | 다양한 Prometheus 형식으로 HTTP 서버에 메트릭을 제공한다.              |

다음 코드 스니펫은 `MetricReader`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/MetricReaderConfig.java"?>
```java
package otel;

import io.opentelemetry.exporter.prometheus.PrometheusHttpServer;
import io.opentelemetry.sdk.metrics.export.MetricExporter;
import io.opentelemetry.sdk.metrics.export.MetricReader;
import io.opentelemetry.sdk.metrics.export.PeriodicMetricReader;
import java.time.Duration;

public class MetricReaderConfig {
  public static MetricReader periodicMetricReader(MetricExporter metricExporter) {
    return PeriodicMetricReader.builder(metricExporter).setInterval(Duration.ofSeconds(60)).build();
  }

  public static MetricReader prometheusMetricReader() {
    return PrometheusHttpServer.builder().setHost("localhost").setPort(9464).build();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 메트릭 리더 로직을 제공하려면 `MetricReader` 인터페이스를 구현한다. 예를
들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomMetricReader.java"?>
```java
package otel;

import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.common.export.MemoryMode;
import io.opentelemetry.sdk.metrics.Aggregation;
import io.opentelemetry.sdk.metrics.InstrumentType;
import io.opentelemetry.sdk.metrics.data.AggregationTemporality;
import io.opentelemetry.sdk.metrics.export.AggregationTemporalitySelector;
import io.opentelemetry.sdk.metrics.export.CollectionRegistration;
import io.opentelemetry.sdk.metrics.export.MetricReader;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.logging.Level;
import java.util.logging.Logger;

public class CustomMetricReader implements MetricReader {

  private static final Logger logger = Logger.getLogger(CustomMetricExporter.class.getName());

  private final ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
  private final AtomicReference<CollectionRegistration> collectionRef =
      new AtomicReference<>(CollectionRegistration.noop());

  @Override
  public void register(CollectionRegistration collectionRegistration) {
    // Callback invoked when SdkMeterProvider is initialized, providing a handle to collect metrics.
    collectionRef.set(collectionRegistration);
    executorService.scheduleWithFixedDelay(this::collectMetrics, 0, 60, TimeUnit.SECONDS);
  }

  private void collectMetrics() {
    // Collect metrics. Typically, records are sent out of process via some network protocol, but we
    // simply log for illustrative purposes.
    logger.log(Level.INFO, "Collecting metrics");
    collectionRef
        .get()
        .collectAllMetrics()
        .forEach(metric -> logger.log(Level.INFO, "Metric: " + metric));
  }

  @Override
  public CompletableResultCode forceFlush() {
    // Export any records which have been queued up but not yet exported.
    logger.log(Level.INFO, "flushing");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode shutdown() {
    // Shutdown the exporter and cleanup any resources.
    logger.log(Level.INFO, "shutting down");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public AggregationTemporality getAggregationTemporality(InstrumentType instrumentType) {
    // Specify the required aggregation temporality as a function of instrument type
    return AggregationTemporalitySelector.deltaPreferred()
        .getAggregationTemporality(instrumentType);
  }

  @Override
  public MemoryMode getMemoryMode() {
    // Optionally specify the memory mode, indicating whether metric records can be reused or must
    // be immutable
    return MemoryMode.REUSABLE_DATA;
  }

  @Override
  public Aggregation getDefaultAggregation(InstrumentType instrumentType) {
    // Optionally specify the default aggregation as a function of instrument kind
    return Aggregation.defaultAggregation();
  }
}
```
<!-- prettier-ignore-end -->

#### 메트릭 익스포터(MetricExporter) {#metricexporter}

[MetricExporter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-metrics/latest/io/opentelemetry/sdk/metrics/export/MetricExporter.html)는
메트릭을 프로세스 밖으로 내보내는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다.
`SdkMeterProvider`에 직접 등록되는 대신, [PeriodicMetricReader](#metricreader)와
짝을 이룬다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
메트릭 익스포터:

| 클래스                           | 아티팩트                                                                             | 설명                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| `OtlpHttpMetricExporter` **[1]** | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`               | OTLP `http/protobuf`를 통해 메트릭을 내보낸다.                                            |
| `OtlpGrpcMetricExporter` **[1]** | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`               | OTLP `grpc`를 통해 메트릭을 내보낸다.                                                     |
| `LoggingMetricExporter`          | `io.opentelemetry:opentelemetry-exporter-logging:{{% param vers.otel %}}`            | 디버깅 형식으로 JUL에 메트릭을 로깅한다.                                                  |
| `OtlpJsonLoggingMetricExporter`  | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`       | OTLP JSON 인코딩으로 JUL에 메트릭을 로깅한다.                                             |
| `OtlpStdoutMetricExporter`       | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`       | OTLP [JSON 파일 인코딩][JSON file encoding](실험적)으로 `System.out`에 메트릭을 로깅한다. |
| `InterceptableMetricExporter`    | `io.opentelemetry.contrib:opentelemetry-processors:{{% param vers.contrib %}}-alpha` | 내보내기 전에 유연한 인터셉터로 메트릭을 전달한다.                                        |

**[1]**: 구현 세부 사항은 [OTLP 익스포터](#otlp-exporters)를 참고한다.

다음 코드 스니펫은 `MetricExporter`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/MetricExporterConfig.java"?>
```java
package otel;

import io.opentelemetry.exporter.logging.LoggingMetricExporter;
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingMetricExporter;
import io.opentelemetry.exporter.otlp.http.metrics.OtlpHttpMetricExporter;
import io.opentelemetry.exporter.otlp.metrics.OtlpGrpcMetricExporter;
import io.opentelemetry.sdk.metrics.export.MetricExporter;
import java.time.Duration;

public class MetricExporterConfig {
  public static MetricExporter otlpHttpMetricExporter(String endpoint) {
    return OtlpHttpMetricExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static MetricExporter otlpGrpcMetricExporter(String endpoint) {
    return OtlpGrpcMetricExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static MetricExporter logginMetricExporter() {
    return LoggingMetricExporter.create();
  }

  public static MetricExporter otlpJsonLoggingMetricExporter() {
    return OtlpJsonLoggingMetricExporter.create();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 메트릭 내보내기 로직을 제공하려면 `MetricExporter` 인터페이스를 구현한다.
예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomMetricExporter.java"?>
```java
package otel;

import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.common.export.MemoryMode;
import io.opentelemetry.sdk.metrics.Aggregation;
import io.opentelemetry.sdk.metrics.InstrumentType;
import io.opentelemetry.sdk.metrics.data.AggregationTemporality;
import io.opentelemetry.sdk.metrics.data.MetricData;
import io.opentelemetry.sdk.metrics.export.AggregationTemporalitySelector;
import io.opentelemetry.sdk.metrics.export.MetricExporter;
import java.util.Collection;
import java.util.logging.Level;
import java.util.logging.Logger;

public class CustomMetricExporter implements MetricExporter {

  private static final Logger logger = Logger.getLogger(CustomMetricExporter.class.getName());

  @Override
  public CompletableResultCode export(Collection<MetricData> metrics) {
    // Export the records. Typically, records are sent out of process via some network protocol, but
    // we simply log for illustrative purposes.
    logger.log(Level.INFO, "Exporting metrics");
    metrics.forEach(metric -> logger.log(Level.INFO, "Metric: " + metric));
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode flush() {
    // Export any records which have been queued up but not yet exported.
    logger.log(Level.INFO, "flushing");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode shutdown() {
    // Shutdown the exporter and cleanup any resources.
    logger.log(Level.INFO, "shutting down");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public AggregationTemporality getAggregationTemporality(InstrumentType instrumentType) {
    // Specify the required aggregation temporality as a function of instrument type
    return AggregationTemporalitySelector.deltaPreferred()
        .getAggregationTemporality(instrumentType);
  }

  @Override
  public MemoryMode getMemoryMode() {
    // Optionally specify the memory mode, indicating whether metric records can be reused or must
    // be immutable
    return MemoryMode.REUSABLE_DATA;
  }

  @Override
  public Aggregation getDefaultAggregation(InstrumentType instrumentType) {
    // Optionally specify the default aggregation as a function of instrument kind
    return Aggregation.defaultAggregation();
  }
}
```
<!-- prettier-ignore-end -->

#### 뷰(Views) {#views}

[Views](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-metrics/latest/io/opentelemetry/sdk/metrics/View.html)를
사용하면 메트릭 이름, 메트릭 설명, 메트릭 집계(즉, 히스토그램 버킷 경계), 유지할
속성 키 집합, 카디널리티 제한 등을 변경하여 메트릭 스트림을 커스터마이징할 수
있다.

> [!NOTE]
>
> 특정 계측기에 여러 뷰가 일치할 때 뷰는 다소 직관적이지 않은 동작을 보인다.
> 하나의 일치하는 뷰가 메트릭 이름을 바꾸고 다른 뷰가 메트릭 집계를 바꾸는 경우,
> 이름과 집계가 모두 바뀔 것이라고 예상할 수 있지만 실제로는 그렇지 않다. 대신,
> 두 개의 메트릭 스트림이 생성된다. 하나는 구성된 메트릭 이름과 기본 집계를
> 가지고, 다른 하나는 원래 메트릭 이름과 구성된 집계를 가진다. 다시 말해,
> 일치하는 뷰는 _병합되지 않는다_. 최선의 결과를 위해서는 좁은 선택 기준으로
> 뷰를 구성한다(즉, 단일한 특정 계측기를 선택한다).

다음 코드 스니펫은 `View`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ViewConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.metrics.Aggregation;
import io.opentelemetry.sdk.metrics.InstrumentSelector;
import io.opentelemetry.sdk.metrics.SdkMeterProviderBuilder;
import io.opentelemetry.sdk.metrics.View;
import java.util.List;
import java.util.Set;

public class ViewConfig {
  public static SdkMeterProviderBuilder dropMetricView(
      SdkMeterProviderBuilder builder, String metricName) {
    return builder.registerView(
        InstrumentSelector.builder().setName(metricName).build(),
        View.builder().setAggregation(Aggregation.drop()).build());
  }

  public static SdkMeterProviderBuilder histogramBucketBoundariesView(
      SdkMeterProviderBuilder builder, String metricName, List<Double> bucketBoundaries) {
    return builder.registerView(
        InstrumentSelector.builder().setName(metricName).build(),
        View.builder()
            .setAggregation(Aggregation.explicitBucketHistogram(bucketBoundaries))
            .build());
  }

  public static SdkMeterProviderBuilder attributeFilterView(
      SdkMeterProviderBuilder builder, String metricName, Set<String> keysToRetain) {
    return builder.registerView(
        InstrumentSelector.builder().setName(metricName).build(),
        View.builder().setAttributeFilter(keysToRetain).build());
  }

  public static SdkMeterProviderBuilder cardinalityLimitsView(
      SdkMeterProviderBuilder builder, String metricName, int cardinalityLimit) {
    return builder.registerView(
        InstrumentSelector.builder().setName(metricName).build(),
        View.builder().setCardinalityLimit(cardinalityLimit).build());
  }
}
```
<!-- prettier-ignore-end -->

### SDK 로거 프로바이더(SdkLoggerProvider) {#sdkloggerprovider}

[SdkLoggerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-logs/latest/io/opentelemetry/sdk/logs/SdkLoggerProvider.html)는
[LoggerProvider](../api/#loggerprovider)의 SDK 구현체이며, 로그 브리지 API가
생성한 로그 텔레메트리를 처리하는 역할을 한다.

`SdkLoggerProvider`는 애플리케이션 소유자가 구성하며, 다음으로 이루어진다.

- [Resource](#resource): 로그가 연관되는 리소스이다.
- [LogRecordProcessor](#logrecordprocessor): 로그가 내보내질 때 이를 처리한다.
- [LogRecordExporter](#logrecordexporter): (연관된 `LogRecordProcessor`와 함께)
  로그를 프로세스 밖으로 내보낸다.
- [LogLimits](#loglimits): 로그와 연관된 데이터의 제한을 제어한다.

다음 코드 스니펫은 `SdkLoggerProvider`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SdkLoggerProviderConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.logs.SdkLoggerProvider;
import io.opentelemetry.sdk.resources.Resource;

public class SdkLoggerProviderConfig {
  public static SdkLoggerProvider create(Resource resource) {
    return SdkLoggerProvider.builder()
        .setResource(resource)
        .addLogRecordProcessor(
            LogRecordProcessorConfig.batchLogRecordProcessor(
                LogRecordExporterConfig.otlpHttpLogRecordExporter("http://localhost:4318/v1/logs")))
        .setLogLimits(LogLimitsConfig::logLimits)
        .build();
  }
}
```
<!-- prettier-ignore-end -->

#### 로그 레코드 프로세서(LogRecordProcessor) {#logrecordprocessor}

[LogRecordProcessor](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-logs/latest/io/opentelemetry/sdk/logs/LogRecordProcessor.html)는
로그가 내보내질 때 호출되는 콜백을 가진
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다. 이는 로그를
프로세스 밖으로 내보내기 위해 [LogRecordExporter](#logrecordexporter)와 함께
사용되는 경우가 많지만, 데이터 보강과 같은 다른 용도로도 사용된다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
로그 레코드 프로세서:

| 클래스                     | 아티팩트                                                                             | 설명                                                                       |
| -------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| `BatchLogRecordProcessor`  | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                         | 로그 레코드를 배치로 묶어 구성 가능한 `LogRecordExporter`를 통해 내보낸다. |
| `SimpleLogRecordProcessor` | `io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}`                         | 각 로그 레코드를 구성 가능한 `LogRecordExporter`를 통해 내보낸다.          |
| `EventToSpanEventBridge`   | `io.opentelemetry.contrib:opentelemetry-processors:{{% param vers.contrib %}}-alpha` | 이벤트 로그 레코드를 현재 스팬의 스팬 이벤트로 기록한다.                   |

다음 코드 스니펫은 `LogRecordProcessor`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/LogRecordProcessorConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.logs.LogRecordProcessor;
import io.opentelemetry.sdk.logs.export.BatchLogRecordProcessor;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;
import io.opentelemetry.sdk.logs.export.SimpleLogRecordProcessor;
import java.time.Duration;

public class LogRecordProcessorConfig {
  public static LogRecordProcessor batchLogRecordProcessor(LogRecordExporter logRecordExporter) {
    return BatchLogRecordProcessor.builder(logRecordExporter)
        .setMaxQueueSize(2048)
        .setExporterTimeout(Duration.ofSeconds(30))
        .setScheduleDelay(Duration.ofSeconds(1))
        .build();
  }

  public static LogRecordProcessor simpleLogRecordProcessor(LogRecordExporter logRecordExporter) {
    return SimpleLogRecordProcessor.create(logRecordExporter);
  }
}
```
<!-- prettier-ignore-end -->

커스텀 로그 처리 로직을 제공하려면 `LogRecordProcessor` 인터페이스를 구현한다.
예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomLogRecordProcessor.java"?>
```java
package otel;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.context.Context;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.logs.LogRecordProcessor;
import io.opentelemetry.sdk.logs.ReadWriteLogRecord;

public class CustomLogRecordProcessor implements LogRecordProcessor {

  @Override
  public void onEmit(Context context, ReadWriteLogRecord logRecord) {
    // Callback invoked when log record is emitted.
    // Enrich the record with a custom attribute.
    logRecord.setAttribute(AttributeKey.stringKey("my.custom.attribute"), "hello world");
  }

  @Override
  public CompletableResultCode shutdown() {
    // Optionally shutdown the processor and cleanup any resources.
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode forceFlush() {
    // Optionally process any records which have been queued up but not yet processed.
    return CompletableResultCode.ofSuccess();
  }
}
```
<!-- prettier-ignore-end -->

#### 로그 레코드 익스포터(LogRecordExporter) {#logrecordexporter}

[LogRecordExporter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-logs/latest/io/opentelemetry/sdk/logs/export/LogRecordExporter.html)는
로그 레코드를 프로세스 밖으로 내보내는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다.
`SdkLoggerProvider`에 직접 등록되는 대신,
[LogRecordProcessor](#logrecordprocessor)(주로 `BatchLogRecordProcessor`)와 짝을
이룬다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
로그 레코드 익스포터:

| 클래스                                     | 아티팩트                                                                             | 설명                                                                                           |
| ------------------------------------------ | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| `OtlpHttpLogRecordExporter` **[1]**        | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`               | OTLP `http/protobuf`를 통해 로그 레코드를 내보낸다.                                            |
| `OtlpGrpcLogRecordExporter` **[1]**        | `io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`               | OTLP `grpc`를 통해 로그 레코드를 내보낸다.                                                     |
| `SystemOutLogRecordExporter`               | `io.opentelemetry:opentelemetry-exporter-logging:{{% param vers.otel %}}`            | 디버깅 형식으로 시스템 출력(system out)에 로그 레코드를 로깅한다.                              |
| `OtlpJsonLoggingLogRecordExporter` **[2]** | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`       | OTLP JSON 인코딩으로 JUL에 로그 레코드를 로깅한다.                                             |
| `OtlpStdoutLogRecordExporter`              | `io.opentelemetry:opentelemetry-exporter-logging-otlp:{{% param vers.otel %}}`       | OTLP [JSON 파일 인코딩][JSON file encoding](실험적)으로 `System.out`에 로그 레코드를 로깅한다. |
| `InterceptableLogRecordExporter`           | `io.opentelemetry.contrib:opentelemetry-processors:{{% param vers.contrib %}}-alpha` | 내보내기 전에 유연한 인터셉터로 로그 레코드를 전달한다.                                        |

**[1]**: 구현 세부 사항은 [OTLP 익스포터](#otlp-exporters)를 참고한다.

**[2]**: `OtlpJsonLoggingLogRecordExporter`는 JUL에 로깅하며, 신중하게 구성하지
않으면 무한 루프(즉, JUL -> SLF4J -> Logback -> OpenTelemetry Appender ->
OpenTelemetry Log SDK -> JUL)를 일으킬 수 있다.

다음 코드 스니펫은 `LogRecordExporter`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/LogRecordExporterConfig.java"?>
```java
package otel;

import io.opentelemetry.exporter.logging.SystemOutLogRecordExporter;
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingLogRecordExporter;
import io.opentelemetry.exporter.otlp.http.logs.OtlpHttpLogRecordExporter;
import io.opentelemetry.exporter.otlp.logs.OtlpGrpcLogRecordExporter;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;
import java.time.Duration;

public class LogRecordExporterConfig {
  public static LogRecordExporter otlpHttpLogRecordExporter(String endpoint) {
    return OtlpHttpLogRecordExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static LogRecordExporter otlpGrpcLogRecordExporter(String endpoint) {
    return OtlpGrpcLogRecordExporter.builder()
        .setEndpoint(endpoint)
        .addHeader("api-key", "value")
        .setTimeout(Duration.ofSeconds(10))
        .build();
  }

  public static LogRecordExporter systemOutLogRecordExporter() {
    return SystemOutLogRecordExporter.create();
  }

  public static LogRecordExporter otlpJsonLoggingLogRecordExporter() {
    return OtlpJsonLoggingLogRecordExporter.create();
  }
}
```
<!-- prettier-ignore-end -->

커스텀 로그 레코드 내보내기 로직을 제공하려면 `LogRecordExporter` 인터페이스를
구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomLogRecordExporter.java"?>
```java
package otel;

import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.logs.data.LogRecordData;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;
import java.util.Collection;
import java.util.logging.Level;
import java.util.logging.Logger;

public class CustomLogRecordExporter implements LogRecordExporter {

  private static final Logger logger = Logger.getLogger(CustomLogRecordExporter.class.getName());

  @Override
  public CompletableResultCode export(Collection<LogRecordData> logs) {
    // Export the records. Typically, records are sent out of process via some network protocol, but
    // we simply log for illustrative purposes.
    System.out.println("Exporting logs");
    logs.forEach(log -> System.out.println("log record: " + log));
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode flush() {
    // Export any records which have been queued up but not yet exported.
    logger.log(Level.INFO, "flushing");
    return CompletableResultCode.ofSuccess();
  }

  @Override
  public CompletableResultCode shutdown() {
    // Shutdown the exporter and cleanup any resources.
    logger.log(Level.INFO, "shutting down");
    return CompletableResultCode.ofSuccess();
  }
}
```
<!-- prettier-ignore-end -->

#### 로그 제한(LogLimits) {#loglimits}

[LogLimits](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-logs/latest/io/opentelemetry/sdk/logs/LogLimits.html)는
최대 속성 길이 및 최대 속성 개수 등 로그 레코드가 캡처하는 데이터에 대한 제약을
정의한다.

다음 코드 스니펫은 `LogLimits`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/LogLimitsConfig.java"?>
```java
package otel;

import io.opentelemetry.sdk.logs.LogLimits;

public class LogLimitsConfig {
  public static LogLimits logLimits() {
    return LogLimits.builder()
        .setMaxNumberOfAttributes(128)
        .setMaxAttributeValueLength(1024)
        .build();
  }
}
```
<!-- prettier-ignore-end -->

### 텍스트 맵 전파자(TextMapPropagator) {#textmappropagator}

[TextMapPropagator](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-context/latest/io/opentelemetry/context/propagation/TextMapPropagator.html)는
텍스트 형식으로 프로세스 경계를 넘어 컨텍스트를 전파하는 역할을 하는
[플러그인 확장 인터페이스](#sdk-plugin-extension-interfaces)이다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
TextMapPropagator:

| 클래스                      | 아티팩트                                                                                      | 설명                                                                       |
| --------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `W3CTraceContextPropagator` | `io.opentelemetry:opentelemetry-api:{{% param vers.otel %}}`                                  | W3C 트레이스 컨텍스트 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다. |
| `W3CBaggagePropagator`      | `io.opentelemetry:opentelemetry-api:{{% param vers.otel %}}`                                  | W3C 배기지 전파 프로토콜을 사용해 배기지를 전파한다.                       |
| `MultiTextMapPropagator`    | `io.opentelemetry:opentelemetry-context:{{% param vers.otel %}}`                              | 여러 전파자를 조합한다.                                                    |
| `JaegerPropagator`          | `io.opentelemetry:opentelemetry-extension-trace-propagators:{{% param vers.otel %}}`          | Jaeger 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다.                |
| `B3Propagator`              | `io.opentelemetry:opentelemetry-extension-trace-propagators:{{% param vers.otel %}}`          | B3 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다.                    |
| `OtTracePropagator`         | `io.opentelemetry:opentelemetry-extension-trace-propagators:{{% param vers.otel %}}`          | OpenTracing 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다.           |
| `PassThroughPropagator`     | `io.opentelemetry:opentelemetry-api-incubator:{{% param vers.otel %}}-alpha`                  | 텔레메트리에 참여하지 않고 구성 가능한 필드 집합을 전파한다.               |
| `AwsXrayPropagator`         | `io.opentelemetry.contrib:opentelemetry-aws-xray-propagator:{{% param vers.contrib %}}-alpha` | AWS X-Ray 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다.             |
| `AwsXrayLambdaPropagator`   | `io.opentelemetry.contrib:opentelemetry-aws-xray-propagator:{{% param vers.contrib %}}-alpha` | 환경 변수와 AWS X-Ray 전파 프로토콜을 사용해 트레이스 컨텍스트를 전파한다. |

다음 코드 스니펫은 `TextMapPropagator`의 프로그래밍 방식 구성을 보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ContextPropagatorsConfig.java"?>
```java
package otel;

import io.opentelemetry.api.baggage.propagation.W3CBaggagePropagator;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.context.propagation.TextMapPropagator;

public class ContextPropagatorsConfig {
  public static ContextPropagators create() {
    return ContextPropagators.create(
        TextMapPropagator.composite(
            W3CTraceContextPropagator.getInstance(), W3CBaggagePropagator.getInstance()));
  }
}
```
<!-- prettier-ignore-end -->

커스텀 전파자 로직을 제공하려면 `TextMapPropagator` 인터페이스를 구현한다. 예를
들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomTextMapPropagator.java"?>
```java
package otel;

import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.TextMapGetter;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import java.util.Collection;
import java.util.Collections;

public class CustomTextMapPropagator implements TextMapPropagator {

  @Override
  public Collection<String> fields() {
    // Return fields used for propagation. See W3CTraceContextPropagator for reference
    // implementation.
    return Collections.emptyList();
  }

  @Override
  public <C> void inject(Context context, C carrier, TextMapSetter<C> setter) {
    // Inject context. See W3CTraceContextPropagator for reference implementation.
  }

  @Override
  public <C> Context extract(Context context, C carrier, TextMapGetter<C> getter) {
    // Extract context. See W3CTraceContextPropagator for reference implementation.
    return context;
  }
}
```
<!-- prettier-ignore-end -->

## 부록 {#appendix}

### 내부 로깅 {#internal-logging}

SDK 구성 요소는 다양한 정보를
[java.util.logging](https://docs.oracle.com/javase/7/docs/api/java/util/logging/package-summary.html)에
서로 다른 로그 레벨로, 그리고 해당 구성 요소의 정규화된 클래스 이름에 기반한
로거 이름을 사용해 로깅한다.

기본적으로 로그 메시지는 애플리케이션의 루트 핸들러가 처리한다. 애플리케이션에
커스텀 루트 핸들러를 설치하지 않았다면, `INFO` 레벨 이상의 로그는 기본적으로
콘솔로 전송된다.

오픈텔레메트리를 위한 로거의 동작을 바꾸고 싶을 수 있다. 예를 들어, 디버깅 시
추가 정보를 출력하도록 로깅 레벨을 낮추거나, 특정 클래스에서 오는 오류를
무시하도록 레벨을 높이거나, 오픈텔레메트리가 특정 메시지를 로깅할 때마다 커스텀
코드를 실행하도록 커스텀 핸들러나 필터를 설치할 수 있다. 로거 이름과 로그 정보에
대한 상세한 목록은 유지되지 않는다. 다만 모든 오픈텔레메트리 API, SDK, contrib,
계측 구성 요소는 동일한 `io.opentelemetry.*` 패키지 접두사를 공유한다. 모든
`io.opentelemetry.*`에 대해 더 세밀한 로그를 활성화하고 출력을 살펴본 다음, 관심
있는 패키지나 FQCN(정규화된 클래스 이름)으로 좁혀가는 것이 유용할 수 있다.

예를 들면:

```properties
## Turn off all OpenTelemetry logging
io.opentelemetry.level = OFF
```

```properties
## Turn off logging for just the BatchSpanProcessor
io.opentelemetry.sdk.trace.export.BatchSpanProcessor.level = OFF
```

```properties
## Log "FINE" messages for help in debugging
io.opentelemetry.level = FINE

## Sets the default ConsoleHandler's logger's level
## Note this impacts the logging outside of OpenTelemetry as well
java.util.logging.ConsoleHandler.level = FINE
```

더 세밀한 제어와 특수한 경우의 처리를 위해, 커스텀 핸들러와 필터를 코드로 지정할
수 있다.

```java
// Custom filter which does not log errors which come from the export
public class IgnoreExportErrorsFilter implements java.util.logging.Filter {

 public boolean isLoggable(LogRecord record) {
    return !record.getMessage().contains("Exception thrown by the export");
 }
}
```

```properties
## Registering the custom filter on the BatchSpanProcessor
io.opentelemetry.sdk.trace.export.BatchSpanProcessor = io.opentelemetry.extension.logging.IgnoreExportErrorsFilter
```

### OTLP 익스포터 {#otlp-exporters}

[스팬 익스포터](#spanexporter), [메트릭 익스포터](#metricexporter),
[로그 익스포터](#logrecordexporter) 절에서는 다음과 같은 형태의 OTLP 익스포터를
설명한다.

- `OtlpHttp{Signal}Exporter`: OTLP `http/protobuf`를 통해 데이터를 내보낸다.
- `OtlpGrpc{Signal}Exporter`: OTLP `grpc`를 통해 데이터를 내보낸다.

모든 시그널의 익스포터는
`io.opentelemetry:opentelemetry-exporter-otlp:{{% param vers.otel %}}`를 통해
사용할 수 있으며, OTLP 프로토콜의 `grpc` 버전과 `http/protobuf` 버전 사이,
그리고 시그널 사이에 상당히 겹치는 부분이 있다. 다음 절에서는 이러한 핵심 개념을
자세히 설명한다.

- [센더(Sender)](#senders): 서로 다른 HTTP/gRPC 클라이언트 라이브러리를 위한
  추상화이다.
- OTLP 익스포터를 위한 [인증](#authentication) 옵션.
- 익스포터 및 기타 SDK 구성 요소가 내보내는
  [SDK 자체 모니터링 메트릭](#sdk-self-monitoring-metrics).

#### 센더(Senders) {#senders}

OTLP 익스포터는 HTTP 및 gRPC 요청을 실행하기 위해 다양한 클라이언트 라이브러리에
의존한다. Java 생태계의 모든 사용 사례를 만족하는 단일 HTTP/gRPC 클라이언트
라이브러리는 없다.

- Java 11 이상에는 내장 `java.net.http.HttpClient`가 있지만,
  `opentelemetry-java`는 Java 8 이상 사용자를 지원해야 하며, 트레일러
  헤더(trailer header)를 지원하지 않아 `gRPC`를 통한 내보내기에는 사용할 수
  없다.
- [OkHttp](https://lysine.dev/okhttp/)는 트레일러 헤더를 지원하는 강력한 HTTP
  클라이언트를 제공하지만, kotlin 표준 라이브러리에 의존한다.
- [grpc-java](https://github.com/grpc/grpc-java)는 다양한
  [트랜스포트 구현체](https://github.com/grpc/grpc-java#transport)를 가진 자체
  `ManagedChannel` 추상화를 제공하지만, `http/protobuf`에는 적합하지 않다.

다양한 사용 사례를 수용하기 위해, `opentelemetry-exporter-otlp`는 애플리케이션
제약에 맞는 다양한 구현체를 가진 내부 "센더(sender)" 추상화를 사용한다. 다른
구현체를 선택하려면, 기본 의존성인
`io.opentelemetry:opentelemetry-exporter-sender-okhttp`를 제외하고 대안에 대한
의존성을 추가한다.

| 아티팩트                                                                                              | 설명                                                     | OTLP 프로토콜           | 기본값 |
| ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------- | ------ |
| `io.opentelemetry:opentelemetry-exporter-sender-okhttp:{{% param vers.otel %}}`                       | OkHttp 기반 구현체이다.                                  | `grpc`, `http/protobuf` | 예     |
| `io.opentelemetry:opentelemetry-exporter-sender-jdk:{{% param vers.otel %}}`                          | Java 11 이상 `java.net.http.HttpClient` 기반 구현체이다. | `http/protobuf`         | 아니오 |
| `io.opentelemetry:opentelemetry-exporter-sender-grpc-managed-channel:{{% param vers.otel %}}` **[1]** | `grpc-java`의 `ManagedChannel` 기반 구현체이다.          | `grpc`                  | 아니오 |

**[1]**: `opentelemetry-exporter-sender-grpc-managed-channel`을 사용하려면,
[gRPC 트랜스포트 구현체](https://github.com/grpc/grpc-java#transport)에 대한
의존성도 추가해야 한다.

#### 인증 {#authentication}

OTLP 익스포터는 정적 및 동적 헤더 기반 인증과 mTLS를 위한 메커니즘을 제공한다.

환경 변수와 시스템 프로퍼티를 사용해
[제로 코드 SDK 자동구성](../configuration/#zero-code-sdk-autoconfigure)을
사용하는 경우, [관련 시스템 프로퍼티](../configuration/#properties-exporters)를
참고한다.

- 정적 헤더 기반 인증을 위한 `otel.exporter.otlp.headers`.
- mTLS 인증을 위한 `otel.exporter.otlp.client.key`,
  `otel.exporter.otlp.client.certificate`.

다음 코드 스니펫은 정적 및 동적 헤더 기반 인증의 프로그래밍 방식 구성을
보여준다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/OtlpAuthenticationConfig.java"?>
```java
package otel;

import io.opentelemetry.exporter.otlp.http.logs.OtlpHttpLogRecordExporter;
import io.opentelemetry.exporter.otlp.http.metrics.OtlpHttpMetricExporter;
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import java.time.Duration;
import java.time.Instant;
import java.util.Collections;
import java.util.Map;
import java.util.function.Supplier;

public class OtlpAuthenticationConfig {
  public static void staticAuthenticationHeader(String endpoint) {
    // If the OTLP destination accepts a static, long-lived authentication header like an API key,
    // set it as a header.
    // This reads the API key from the OTLP_API_KEY env var to avoid hard coding the secret in
    // source code.
    String apiKeyHeaderName = "api-key";
    String apiKeyHeaderValue = System.getenv("OTLP_API_KEY");

    // Initialize OTLP Span, Metric, and LogRecord exporters using a similar pattern
    OtlpHttpSpanExporter spanExporter =
        OtlpHttpSpanExporter.builder()
            .setEndpoint(endpoint)
            .addHeader(apiKeyHeaderName, apiKeyHeaderValue)
            .build();
    OtlpHttpMetricExporter metricExporter =
        OtlpHttpMetricExporter.builder()
            .setEndpoint(endpoint)
            .addHeader(apiKeyHeaderName, apiKeyHeaderValue)
            .build();
    OtlpHttpLogRecordExporter logRecordExporter =
        OtlpHttpLogRecordExporter.builder()
            .setEndpoint(endpoint)
            .addHeader(apiKeyHeaderName, apiKeyHeaderValue)
            .build();
  }

  public static void dynamicAuthenticationHeader(String endpoint) {
    // If the OTLP destination requires a dynamic authentication header, such as a JWT which needs
    // to be periodically refreshed, use a header supplier.
    // Here we implement a simple supplier which adds a header of the form "Authorization: Bearer
    // <token>", where <token> is fetched from refreshBearerToken every 10 minutes.
    String username = System.getenv("OTLP_USERNAME");
    String password = System.getenv("OTLP_PASSWORD");
    Supplier<Map<String, String>> supplier =
        new AuthHeaderSupplier(() -> refreshToken(username, password), Duration.ofMinutes(10));

    // Initialize OTLP Span, Metric, and LogRecord exporters using a similar pattern
    OtlpHttpSpanExporter spanExporter =
        OtlpHttpSpanExporter.builder().setEndpoint(endpoint).setHeaders(supplier).build();
    OtlpHttpMetricExporter metricExporter =
        OtlpHttpMetricExporter.builder().setEndpoint(endpoint).setHeaders(supplier).build();
    OtlpHttpLogRecordExporter logRecordExporter =
        OtlpHttpLogRecordExporter.builder().setEndpoint(endpoint).setHeaders(supplier).build();
  }

  private static class AuthHeaderSupplier implements Supplier<Map<String, String>> {
    private final Supplier<String> tokenRefresher;
    private final Duration tokenRefreshInterval;
    private Instant refreshedAt = Instant.ofEpochMilli(0);
    private String currentTokenValue;

    private AuthHeaderSupplier(Supplier<String> tokenRefresher, Duration tokenRefreshInterval) {
      this.tokenRefresher = tokenRefresher;
      this.tokenRefreshInterval = tokenRefreshInterval;
    }

    @Override
    public Map<String, String> get() {
      return Collections.singletonMap("Authorization", "Bearer " + getToken());
    }

    private synchronized String getToken() {
      Instant now = Instant.now();
      if (currentTokenValue == null || now.isAfter(refreshedAt.plus(tokenRefreshInterval))) {
        currentTokenValue = tokenRefresher.get();
        refreshedAt = now;
      }
      return currentTokenValue;
    }
  }

  private static String refreshToken(String username, String password) {
    // For a production scenario, this would be replaced with an out-of-band request to exchange
    // username / password for bearer token.
    return "abc123";
  }
}
```
<!-- prettier-ignore-end -->

### SDK 자체 모니터링 메트릭 {#sdk-self-monitoring-metrics}

Java SDK는 익스포터, 스팬 및 로그 레코드 프로세서, 트레이서 및 로거 프로바이더,
그리고 주기적 메트릭 리더(periodic metric reader)에 대한 자체 모니터링 메트릭을
내보낼 수 있다. 스키마 선택은 OTLP 익스포터와 배치 스팬 및 로그 레코드
프로세서에 적용된다. 다른 구성 요소의 경우, 스키마 선택은 어떤 이름이
사용되는지가 아니라 자체 모니터링이 활성화되는지 여부를 제어한다.

OTLP 익스포터 빌더는 기본적으로 자체 모니터링에
`GlobalOpenTelemetry.getMeterProvider()`를 사용한다. 다른 프로바이더를
사용하려면 빌더에서 `setMeterProvider(...)`를 호출한다.
[제로 코드 SDK 자동구성](../configuration/#zero-code-sdk-autoconfigure)은 구성된
SDK `MeterProvider`를 자동으로 제공한다.

프로그래밍 방식으로 구성된 익스포터의 경우, 메트릭 스키마를 선택하려면
`InternalTelemetryVersion.LEGACY` 또는 `InternalTelemetryVersion.LATEST`로
`setInternalTelemetryVersion(...)`을 호출한다. 제로 코드 SDK 자동구성에서는
`otel.experimental.sdk.telemetry.version`을 `legacy` 또는 `latest`로 설정하며,
기본값은 `legacy`이다.

[선언적 구성](../configuration/#declarative-configuration)에서는 SDK 자체
모니터링 텔레메트리가 기본적으로 비활성화되어 있다. 이를 활성화하려면
`instrumentation/development.java.otel_sdk.internal_telemetry_version`을
`legacy` 또는 `latest`로 설정한다.

```yaml
instrumentation/development:
  java:
    otel_sdk:
      internal_telemetry_version: latest
```

다음 표는 각 구성 요소가 내보내는 메트릭 이름을 요약한 것이다. 대시(dash)는 해당
스키마가 그 구성 요소에 대한 메트릭을 정의하지 않는다는 것을 나타낸다.

| 구성 요소                 | `legacy`                                       | `latest`                                                                                                                                                                                                                                                                         |
| ------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OTLP 익스포터             | `otlp.exporter.seen`, `otlp.exporter.exported` | `otel.sdk.exporter.span.inflight`, `otel.sdk.exporter.span.exported`, `otel.sdk.exporter.metric_data_point.inflight`, `otel.sdk.exporter.metric_data_point.exported`, `otel.sdk.exporter.log.inflight`, `otel.sdk.exporter.log.exported`, `otel.sdk.exporter.operation.duration` |
| `BatchSpanProcessor`      | `queueSize`, `processedSpans`                  | `otel.sdk.processor.span.queue.capacity`, `otel.sdk.processor.span.queue.size`, `otel.sdk.processor.span.processed`                                                                                                                                                              |
| `BatchLogRecordProcessor` | `queueSize`, `processedLogs`                   | `otel.sdk.processor.log.queue.capacity`, `otel.sdk.processor.log.queue.size`, `otel.sdk.processor.log.processed`                                                                                                                                                                 |
| `SdkTracerProvider`       | —                                              | `otel.sdk.span.started`, `otel.sdk.span.live`                                                                                                                                                                                                                                    |
| `SdkLoggerProvider`       | —                                              | `otel.sdk.log.created`                                                                                                                                                                                                                                                           |
| `PeriodicMetricReader`    | —                                              | `otel.sdk.metric_reader.collection.duration`                                                                                                                                                                                                                                     |

`SimpleSpanProcessor`와 `SimpleLogRecordProcessor`는 항상 시맨틱 컨벤션 스키마를
사용하며, 각각 `otel.sdk.processor.span.processed`와
`otel.sdk.processor.log.processed`를 기록한다.

레거시 익스포터 메트릭은 둘 다 값이 `span`, `metric`, `log` 중 하나인 `type`
속성을 포함한다. `otlp.exporter.exported`에는 `success` 속성도 포함된다.

`latest` 이름은
[SDK 메트릭 시맨틱 컨벤션](/docs/specs/semconv/otel/sdk-metrics/)을 따른다.
익스포터 메트릭에는 `otel.component.type`, `otel.component.name`,
`server.address`, `server.port`가 포함된다. 내보내기에 실패하면 `.exported`와
`.duration` 메트릭에는 `error.type`이 추가되지만, `.inflight` 메트릭에는
추가되지 않는다. `otel.sdk.exporter.operation.duration` 메트릭에는 HTTP의 경우
`http.response.status_code`, gRPC의 경우 `rpc.grpc.status_code`도 포함된다.

레거시 메트릭은 SDK 메트릭 시맨틱 컨벤션보다 먼저 만들어졌으며, 기존 사용자에게
미치는 영향을 피하기 위해 SDK 자동구성의 기본값으로 남아 있다. Java SDK는 현재
이를 제거할 일정을 정의하지 않았다.

### 벤치마크 {#benchmarks}

SDK는 [JMH](https://github.com/openjdk/jmh) 벤치마크 결과를
[open-telemetry.github.io/opentelemetry-java/benchmarks/](https://open-telemetry.github.io/opentelemetry-java/benchmarks/)에
게시한다. 벤치마크는 노이즈를 최소화하기 위해 전용 베어메탈 러너(bare-metal
runner)를 사용해 `main`에 대한 모든 커밋마다 실행된다. 결과에는 날짜 필터링 및
시리즈 선택을 위한 도구와, 무엇을 왜 벤치마킹하는지 Javadoc이 설명하는 벤치마크
소스 코드에 대한 링크가 포함된다.

현재 벤치마크는 세 시그널 모두의 **기록 경로(record path)** — 애플리케이션
스레드가 스팬 시작/종료, 메트릭 측정, 로그 발행마다 거치는 핫 패스(hot path) —
를 다룬다.

| 벤치마크                                     | 차원                                                                  |
| -------------------------------------------- | --------------------------------------------------------------------- |
| [`SpanRecordBenchmark`][span-record-src]     | 스팬 크기, 동시 스레드                                                |
| [`MetricRecordBenchmark`][metric-record-src] | 계측기 종류 + 집계, 집계 시간성(temporality), 카디널리티, 동시 스레드 |
| [`LogRecordBenchmark`][log-record-src]       | 로그 레코드 크기, 동시 스레드                                         |

> [!NOTE]
>
> **내보내기 경로(export path)**(배치 프로세서 플러시, 익스포터 I/O 등)에 대한
> 벤치마크는 계획되어 있지만, 내보내기는 핫 패스 밖에서 일어나므로 우선순위가
> 낮다.

[JSON file encoding]:
  /docs/specs/otel/protocol/file-exporter/#json-file-serialization
[span-record-src]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk/all/src/jmh/java/io/opentelemetry/sdk/SpanRecordBenchmark.java
[metric-record-src]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk/all/src/jmh/java/io/opentelemetry/sdk/MetricRecordBenchmark.java
[log-record-src]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk/all/src/jmh/java/io/opentelemetry/sdk/LogRecordBenchmark.java

### 테스트 {#testing}

`io.opentelemetry:opentelemetry-sdk-testing` 아티팩트는 어떤 백엔드로도 데이터를
내보내지 않고, 코드가 생성한 텔레메트리를 검증하기 위한 유틸리티를 제공한다.

다음 구성 요소를 사용할 수 있다.

| 클래스                                                                                                                                                                                  | 설명                                                                                                                                                                                                          |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [OpenTelemetryExtension](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/junit5/OpenTelemetryExtension.html)                  | 인메모리 익스포터와 W3C 트레이스 컨텍스트 전파를 갖춘 `OpenTelemetrySdk`를 설정하고, 이를 `GlobalOpenTelemetry`로 등록하며, 각 테스트 전에 캡처된 텔레메트리를 모두 초기화하는 JUnit 5 익스텐션이다.          |
| [OpenTelemetryRule](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/junit4/OpenTelemetryRule.html)                            | `OpenTelemetryExtension`의 JUnit 4 버전이다. `@ClassRule`로는 사용할 수 없다.                                                                                                                                 |
| [OpenTelemetryAssertions](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/assertj/OpenTelemetryAssertions.html)               | `SpanData`, `MetricData`, `LogRecordData`, `Attributes`, `EventData`에 대해 OTel을 인식하는 `assertThat()` 오버로드로 AssertJ를 확장한다. `import static ...OpenTelemetryAssertions.assertThat`으로 사용한다. |
| [InMemorySpanExporter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/exporter/InMemorySpanExporter.html)                    | 내보낸 스팬을 메모리에 캡처한다.                                                                                                                                                                              |
| [InMemoryMetricReader](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/exporter/InMemoryMetricReader.html)                    | 집계된 메트릭을 메모리에서 읽는다.                                                                                                                                                                            |
| [InMemoryLogRecordExporter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/exporter/InMemoryLogRecordExporter.html)          | 내보낸 로그 레코드를 메모리에 캡처한다.                                                                                                                                                                       |
| [TestClock](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/time/TestClock.html)                                              | 테스트에서 시간을 제어하기 위한 변경 가능한 `Clock`이다. `SdkTracerProvider.builder().setClock(...)`에 전달한다.                                                                                              |
| [TestSpanData](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/trace/TestSpanData.html)                                       | 실제 계측을 실행하지 않고 테스트에서 `SpanData` 인스턴스를 구성하기 위한 변경 불가능한 빌더이다.                                                                                                              |
| [TestLogRecordData](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/logs/TestLogRecordData.html)                              | 테스트에서 `LogRecordData` 인스턴스를 구성하기 위한 변경 불가능한 빌더이다.                                                                                                                                   |
| [SettableContextStorageProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-testing/latest/io/opentelemetry/sdk/testing/context/SettableContextStorageProvider.html) | 런타임에 `ContextStorage`를 교체할 수 있게 해주는 `ContextStorageProvider`이다. 컨텍스트 전파 동작을 테스트하는 데 유용하다.                                                                                  |

#### JUnit 5 {#junit-5}

`OpenTelemetryExtension`은 JUnit 5에서 시작하기에 권장되는 지점이다.

```java
class CoolTest {
  @RegisterExtension
  static final OpenTelemetryExtension otelTesting = OpenTelemetryExtension.create();

  private final Tracer tracer = otelTesting.getOpenTelemetry().getTracer("test");

  @Test
  void test() {
    tracer.spanBuilder("name").startSpan().end();
    assertThat(otelTesting.getSpans())
        .satisfiesExactly(span -> assertThat(span).hasName("name"));
  }
}
```

`getSpans()`, `getMetrics()`, `getLogRecords()`로 원시 텔레메트리에 접근한다.
유창한(fluent) 트레이스 수준 단언(assertion)을 위해서는 `TracesAssert`를 통한
`assertTraces()`를 사용한다. 텔레메트리는 각 테스트 전에 자동으로 초기화되며,
테스트 도중에 초기화하려면 `clearSpans()`, `clearMetrics()`,
`clearLogRecords()`를 사용할 수 있다.

#### JUnit 4 {#junit-4}

`OpenTelemetryRule`은 JUnit 4에서 동일한 API를 제공한다.

```java
public class CoolTest {
  @Rule public OpenTelemetryRule otelTesting = OpenTelemetryRule.create();

  private Tracer tracer;

  @Before
  public void setUp() {
    tracer = otelTesting.getOpenTelemetry().getTracer("test");
  }

  @Test
  public void test() {
    tracer.spanBuilder("name").startSpan().end();
    assertThat(otelTesting.getSpans())
        .satisfiesExactly(span -> assertThat(span).hasName("name"));
  }
}
```
