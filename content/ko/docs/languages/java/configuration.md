---
title: SDK 구성하기
linkTitle: SDK 구성하기
weight: 13
aliases: [config]
# prettier-ignore
cSpell:ignore: autoconfigured blrp Customizer Dotel ignore LOWMEMORY ottrace PKCS retryable
default_lang_commit: 8d6b626b3dd798de9065335d8c4cc0912959c484
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/configuration"?>

[SDK](../sdk/)는 [API](../api/)의 내장 참조 구현체로, 계측 API 호출로 생성된
텔레메트리를 처리하고 내보낸다. 적절하게 처리하고 내보내도록 SDK를 구성하는 것은
오픈텔레메트리를 애플리케이션에 통합하는 데 필수적인 단계이다.

모든 SDK 구성 요소는 [프로그래밍 방식 구성 API](#programmatic-configuration)를
가진다. 이는 SDK를 구성하는 가장 유연하고 표현력 있는 방법이다. 하지만 구성을
변경하려면 코드를 수정하고 애플리케이션을 다시 컴파일해야 하며, API가 Java로
작성되어 있어 언어 상호운용성이 없다.

[제로 코드 SDK 자동구성](#zero-code-sdk-autoconfigure) 모듈은 시스템 속성이나
환경 변수를 통해 SDK 구성 요소를 구성하며, 속성만으로 충분하지 않은 경우를 위한
다양한 확장 지점을 제공한다.

> [!NOTE] **참고**
>
> - [제로 코드 SDK 자동구성](#zero-code-sdk-autoconfigure) 모듈은 상용구 코드를
>   줄이고, 코드를 다시 작성하거나 애플리케이션을 다시 컴파일하지 않고도
>   재구성할 수 있게 하며, 언어 상호운용성을 갖추고 있으므로 사용을 권장한다.
> - [Java 에이전트](/docs/zero-code/java/agent/)와
>   [Spring 스타터](/docs/zero-code/java/spring-boot-starter/)는 제로 코드 SDK
>   자동구성 모듈을 사용해 SDK를 자동으로 구성하고, 이를 통해 계측을 설치한다.
>   모든 자동구성 관련 내용은 Java 에이전트와 Spring 스타터 사용자에게 적용된다.

## 프로그래밍 방식 구성 {#programmatic-configuration}

프로그래밍 방식 구성 인터페이스는 [SDK](../sdk/) 구성 요소를 생성하기 위한 API의
집합이다. 모든 SDK 구성 요소는 프로그래밍 방식 구성 API를 가지며, 다른 모든 구성
메커니즘은 이 API 위에 구축된다. 예를 들어,
[자동구성 환경 변수 및 시스템 속성](#environment-variables-and-system-properties)
구성 인터페이스는 잘 알려진 환경 변수와 시스템 속성을 해석하여 프로그래밍 방식
구성 API에 대한 일련의 호출로 변환한다.

다른 구성 메커니즘이 더 편리하지만, 필요한 정확한 구성을 표현하는 코드를
작성하는 것만큼 유연한 방법은 없다. 특정 기능이 상위 구성 메커니즘에서 지원되지
않는 경우, 프로그래밍 방식 구성을 사용할 수밖에 없을 수 있다.

[SDK 구성 요소](../sdk/#sdk-components) 절에서는 SDK의 핵심 사용자 대상 영역에
대한 간단한 프로그래밍 방식 구성 API를 보여준다. 전체 API 참조는 코드를
참고한다.

## 제로 코드 SDK 자동구성 {#zero-code-sdk-autoconfigure}

자동구성 모듈(아티팩트
`io.opentelemetry:opentelemetry-sdk-extension-autoconfigure:{{% param vers.otel %}}`)은
[프로그래밍 방식 구성 인터페이스](#programmatic-configuration) 위에 구축된 구성
인터페이스로, 코드 작성 없이 [SDK 구성 요소](../sdk/#sdk-components)를 구성한다.
서로 다른 두 가지 자동구성 워크플로가 있다.

- [환경 변수와 시스템 속성](#environment-variables-and-system-properties)은 환경
  변수와 시스템 속성을 해석하여 SDK 구성 요소를 생성하며, 프로그래밍 방식 구성을
  오버레이하기 위한 다양한 커스터마이징 지점을 포함한다.
- [선언적 구성](#declarative-configuration)(**현재 개발 중**)은 구성 모델을
  해석하여 SDK 구성 요소를 생성하며, 이는 일반적으로 YAML 구성 파일로
  인코딩된다.

다음과 같이 자동구성을 사용해 SDK 구성 요소를 자동으로 구성한다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/AutoConfiguredSdk.java"?>
```java
package otel;

import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;

public class AutoConfiguredSdk {
  public static OpenTelemetrySdk autoconfiguredSdk() {
    return AutoConfiguredOpenTelemetrySdk.initialize().getOpenTelemetrySdk();
  }
}
```
<!-- prettier-ignore-end -->

> [!NOTE] **참고**
>
> - [Java 에이전트](/docs/zero-code/java/agent/)와
>   [Spring 스타터](/docs/zero-code/java/spring-boot-starter/)는 제로 코드 SDK
>   자동구성 모듈을 사용해 SDK를 자동으로 구성하고, 이를 통해 계측을 설치한다.
>   모든 자동구성 관련 내용은 Java 에이전트와 Spring 스타터 사용자에게 적용된다.
> - 자동구성 모듈은 적절한 시점에 SDK를 종료하기 위해 Java 셧다운 훅(shutdown
>   hook)을 등록한다. 오픈텔레메트리 Java는
>   [내부 로깅에 `java.util.logging`을 사용](../sdk/#internal-logging)하므로,
>   셧다운 훅 도중 일부 로그가 억제될 수 있다. 이는 JDK 자체의 버그이며
>   오픈텔레메트리 Java가 제어할 수 있는 부분이 아니다. 셧다운 훅 동안 로깅이
>   필요하다면, 셧다운 훅에서 스스로 종료되어 로그 메시지를 억제할 수 있는 로깅
>   프레임워크 대신 `System.out`을 사용하는 것을 고려한다. 자세한 내용은 이
>   [JDK 버그](https://bugs.openjdk.java.net/browse/JDK-8161253)를 참고한다.

### 환경 변수와 시스템 속성 {#environment-variables-and-system-properties}

자동구성 모듈은
[환경 변수 구성 명세](/docs/specs/otel/configuration/sdk-environment-variables/)에
나열된 속성을 지원하며, 이따금 실험적이거나 Java 특화된 항목이 추가된다.

다음 속성들은 시스템 속성으로 나열되어 있지만, 환경 변수로도 설정할 수 있다.
시스템 속성을 환경 변수로 변환하려면 다음 단계를 적용한다.

- 이름을 대문자로 변환한다.
- 모든 `.`과 `-` 문자를 `_`로 바꾼다.

예를 들어 `otel.sdk.disabled` 시스템 속성은 `OTEL_SDK_DISABLED` 환경 변수와
동일하다.

속성이 시스템 속성과 환경 변수로 모두 정의된 경우, 시스템 속성이 우선한다.

#### 속성: 일반 {#properties-general}

[SDK](../sdk/#opentelemetrysdk)를 비활성화하기 위한 속성:

| 시스템 속성         | 설명                                                  | 기본값  |
| ------------------- | ----------------------------------------------------- | ------- |
| `otel.sdk.disabled` | `true`이면 오픈텔레메트리 SDK를 비활성화한다. **[1]** | `false` |

**[1]**: 비활성화된 경우
`AutoConfiguredOpenTelemetrySdk#getOpenTelemetrySdk()`는 최소한으로 구성된
인스턴스를 반환한다(예: `OpenTelemetrySdk.builder().build()`).

SDK 자체 모니터링 텔레메트리를 위한 속성:

| 시스템 속성                               | 설명                                                                                          | 기본값   |
| ----------------------------------------- | --------------------------------------------------------------------------------------------- | -------- |
| `otel.experimental.sdk.telemetry.version` | 자체 모니터링 텔레메트리 스키마를 선택한다. 유효한 값은 `legacy`와 `latest`이다. **[1]** 참고 | `legacy` |

**[1]**: 익스포터뿐 아니라 모든 SDK 자체 모니터링 텔레메트리의 스키마를
선택한다. 배치 스팬 및 로그 레코드 프로세서의 경우 메트릭 이름을 선택하며,
트레이서 및 로거 프로바이더와 주기적 메트릭 리더의 경우 자체 모니터링 메트릭이
기록되는지 여부를 제어한다. 구성 세부 정보와 각 구성 요소가 방출하는 메트릭
이름은 [SDK 자체 모니터링 메트릭](../sdk/#sdk-self-monitoring-metrics)을
참고한다.

속성 제한을 위한 속성([스팬 제한](../sdk/#spanlimits),
[로그 제한](../sdk/#loglimits) 참고):

| 시스템 속성                         | 설명                                                                                                                                            | 기본값    |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| `otel.attribute.value.length.limit` | 속성 값의 최대 길이이다. 스팬과 로그에 적용된다. `otel.span.attribute.value.length.limit`, `otel.span.attribute.count.limit`에 의해 재정의된다. | 제한 없음 |
| `otel.attribute.count.limit`        | 속성의 최대 개수이다. 스팬, 스팬 이벤트, 스팬 링크, 로그에 적용된다.                                                                            | `128`     |

[컨텍스트 전파](../sdk/#textmappropagator)를 위한 속성:

| 시스템 속성        | 설명                                                                                                                                                | 기본값                       |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `otel.propagators` | 쉼표로 구분된 전파자 목록이다. 알려진 값에는 `tracecontext`, `baggage`, `b3`, `b3multi`, `jaeger`, `ottrace`, `xray`, `xray-lambda`가 있다. **[1]** | `tracecontext,baggage` (W3C) |

**[1]**: 알려진 전파자와 아티팩트(아티팩트 좌표는
[텍스트 맵 전파자](../sdk/#textmappropagator) 참고):

- `tracecontext`는 `W3CTraceContextPropagator`를 구성한다.
- `baggage`는 `W3CBaggagePropagator`를 구성한다.
- `b3`, `b3multi`는 `B3Propagator`를 구성한다.
- `jaeger`는 `JaegerPropagator`를 구성한다.
- `ottrace`는 `OtTracePropagator`를 구성한다.
- `xray`는 `AwsXrayPropagator`를 구성한다.
- `xray-lambda`는 `AwsXrayLambdaPropagator`를 구성한다.

#### 속성: 리소스 {#properties-resource}

[리소스](../sdk/#resource)를 구성하기 위한 속성:

| 시스템 속성                             | 설명                                                                                                                                      | 기본값                 |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `otel.service.name`                     | 논리적 서비스 이름을 지정한다. `otel.resource.attributes`로 정의된 `service.name`보다 우선한다.                                           | `unknown_service:java` |
| `otel.resource.attributes`              | 다음 형식으로 리소스 속성을 지정한다: `key1=val1,key2=val2,key3=val3`.                                                                    |                        |
| `otel.resource.disabled.keys`           | 필터링할 리소스 속성 키를 지정한다.                                                                                                       |                        |
| `otel.java.enabled.resource.providers`  | 활성화할 `ResourceProvider`의 정규화된 클래스 이름을 쉼표로 구분한 목록이다. **[1]** 설정하지 않으면 모든 리소스 프로바이더가 활성화된다. |                        |
| `otel.java.disabled.resource.providers` | 비활성화할 `ResourceProvider`의 정규화된 클래스 이름을 쉼표로 구분한 목록이다. **[1]**                                                    |                        |

**[1]**: 예를 들어
[OS 리소스 프로바이더](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/resources/library/src/main/java/io/opentelemetry/instrumentation/resources/OsResourceProvider.java)를
비활성화하려면
`-Dotel.java.disabled.resource.providers=io.opentelemetry.instrumentation.resources.OsResourceProvider`를
설정한다.

**참고**: `otel.service.name`과 `otel.resource.attributes` 시스템 속성/환경
변수는 `io.opentelemetry.sdk.autoconfigure.EnvironmentResourceProvider` 리소스
프로바이더에서 해석된다. `otel.java.enabled.resource-providers`를 통해 리소스
프로바이더를 명시적으로 지정하기로 했다면, 예상치 못한 문제를 피하기 위해 이를
포함하는 것이 좋다. 리소스 프로바이더 아티팩트 좌표는
[ResourceProvider](#resourceprovider)를 참고한다.

#### 속성: 트레이스 {#properties-traces}

`otel.traces.exporter`로 지정된 익스포터와 짝을 이루는
[배치 스팬 프로세서](../sdk/#spanprocessor)를 위한 속성:

| 시스템 속성                      | 설명                                                 | 기본값  |
| -------------------------------- | ---------------------------------------------------- | ------- |
| `otel.bsp.schedule.delay`        | 연속된 두 번의 내보내기 사이의 간격(밀리초)이다.     | `5000`  |
| `otel.bsp.max.queue.size`        | 배치 처리 전 큐에 쌓일 수 있는 스팬의 최대 개수이다. | `2048`  |
| `otel.bsp.max.export.batch.size` | 한 배치에서 내보낼 스팬의 최대 개수이다.             | `512`   |
| `otel.bsp.export.timeout`        | 데이터를 내보내는 데 허용되는 최대 시간(밀리초)이다. | `30000` |

[샘플러](../sdk/#sampler)를 위한 속성:

| 시스템 속성               | 설명                                                                                                                                                                                       | 기본값                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------- |
| `otel.traces.sampler`     | 사용할 샘플러이다. 알려진 값에는 `always_on`, `always_off`, `traceidratio`, `parentbased_always_on`, `parentbased_always_off`, `parentbased_traceidratio`, `jaeger_remote`가 있다. **[1]** | `parentbased_always_on` |
| `otel.traces.sampler.arg` | 지원되는 경우 구성된 트레이서에 대한 인자이다(예: 비율).                                                                                                                                   |                         |

**[1]**: 알려진 샘플러와 아티팩트(아티팩트 좌표는 [샘플러](../sdk/#sampler)
참고):

- `always_on`은 `AlwaysOnSampler`를 구성한다.
- `always_off`는 `AlwaysOffSampler`를 구성한다.
- `traceidratio`는 `TraceIdRatioBased`를 구성한다. `otel.traces.sampler.arg`가
  비율을 설정한다.
- `parentbased_always_on`은 `ParentBased(root=AlwaysOnSampler)`를 구성한다.
- `parentbased_always_off`는 `ParentBased(root=AlwaysOffSampler)`를 구성한다.
- `parentbased_traceidratio`는 `ParentBased(root=TraceIdRatioBased)`를 구성한다.
  `otel.traces.sampler.arg`가 비율을 설정한다.
- `jaeger_remote`는 `JaegerRemoteSampler`를 구성한다.
  `otel.traces.sampler.arg`는
  [명세](/docs/specs/otel/configuration/sdk-environment-variables/#general-sdk-configuration)에
  설명된 대로 쉼표로 구분된 인자 목록이다.

[스팬 제한](../sdk/#spanlimits)을 위한 속성:

| 시스템 속성                              | 설명                                                                            | 기본값    |
| ---------------------------------------- | ------------------------------------------------------------------------------- | --------- |
| `otel.span.attribute.value.length.limit` | 스팬 속성 값의 최대 길이이다. `otel.attribute.value.length.limit`보다 우선한다. | 제한 없음 |
| `otel.span.attribute.count.limit`        | 스팬당 속성의 최대 개수이다. `otel.attribute.count.limit`보다 우선한다.         | `128`     |
| `otel.span.event.count.limit`            | 스팬당 이벤트의 최대 개수이다.                                                  | `128`     |
| `otel.span.link.count.limit`             | 스팬당 링크의 최대 개수이다.                                                    | `128`     |

#### 속성: 메트릭 {#properties-metrics}

[주기적 메트릭 리더](../sdk/#metricreader)를 위한 속성:

| 시스템 속성                   | 설명                                                | 기본값  |
| ----------------------------- | --------------------------------------------------- | ------- |
| `otel.metric.export.interval` | 두 번의 내보내기 시도 시작 사이의 간격(밀리초)이다. | `60000` |

예시값(exemplar)을 위한 속성:

| 시스템 속성                    | 설명                                                                                       | 기본값        |
| ------------------------------ | ------------------------------------------------------------------------------------------ | ------------- |
| `otel.metrics.exemplar.filter` | 예시값 샘플링을 위한 필터이다. `ALWAYS_OFF`, `ALWAYS_ON`, `TRACE_BASED` 중 하나일 수 있다. | `TRACE_BASED` |

카디널리티 제한을 위한 속성:

| 시스템 속성                           | 설명                                                                                                  | 기본값 |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------ |
| `otel.java.metrics.cardinality.limit` | 설정된 경우 카디널리티 제한을 구성한다. 이 값은 메트릭당 고유한 데이터 포인트의 최대 개수를 결정한다. | `2000` |

#### 속성: 로그 {#properties-logs}

`otel.logs.exporter`를 통해 익스포터와 짝을 이루는
[로그 레코드 프로세서](../sdk/#logrecordprocessor)를 위한 속성:

| 시스템 속성                       | 설명                                                        | 기본값  |
| --------------------------------- | ----------------------------------------------------------- | ------- |
| `otel.blrp.schedule.delay`        | 연속된 두 번의 내보내기 사이의 간격(밀리초)이다.            | `1000`  |
| `otel.blrp.max.queue.size`        | 배치 처리 전 큐에 쌓일 수 있는 로그 레코드의 최대 개수이다. | `2048`  |
| `otel.blrp.max.export.batch.size` | 한 배치에서 내보낼 로그 레코드의 최대 개수이다.             | `512`   |
| `otel.blrp.export.timeout`        | 데이터를 내보내는 데 허용되는 최대 시간(밀리초)이다.        | `30000` |

#### 속성: 익스포터 {#properties-exporters}

익스포터를 설정하기 위한 속성:

| 시스템 속성                      | 목적                                                                                                                                                                 | 기본값          |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `otel.traces.exporter`           | 스팬 익스포터의 쉼표로 구분된 목록이다. 알려진 값에는 `otlp`, `zipkin`, `console`, `logging-otlp`, `none`이 있다. **[1]**                                            | `otlp`          |
| `otel.metrics.exporter`          | 메트릭 익스포터의 쉼표로 구분된 목록이다. 알려진 값에는 `otlp`, `prometheus`, `console`, `none`이 있다. **[1]**                                                      | `otlp`          |
| `otel.logs.exporter`             | 로그 레코드 익스포터의 쉼표로 구분된 목록이다. 알려진 값에는 `otlp`, `console`, `logging-otlp`, `none`이 있다. **[1]**                                               | `otlp`          |
| `otel.java.exporter.memory_mode` | `reusable_data`이면 할당을 줄이기 위해 (지원하는 익스포터에서) 재사용 가능 메모리 모드를 활성화한다. 알려진 값에는 `reusable_data`, `immutable_data`가 있다. **[2]** | `reusable_data` |

**[1]**: 알려진 익스포터와 아티팩트(익스포터 아티팩트 좌표는
[스팬 익스포터](../sdk/#spanexporter),
[메트릭 익스포터](../sdk/#metricexporter),
[로그 익스포터](../sdk/#logrecordexporter) 참고):

- `otlp`는 `OtlpHttp{Signal}Exporter`/`OtlpGrpc{Signal}Exporter`를 구성한다.
- `zipkin`은 `ZipkinSpanExporter`를 구성한다.
- `console`은 `LoggingSpanExporter`, `LoggingMetricExporter`,
  `SystemOutLogRecordExporter`를 구성한다.
- `logging-otlp`는 `OtlpJsonLogging{Signal}Exporter`를 구성한다.
- `experimental-otlp/stdout`은 `OtlpStdout{Signal}Exporter`를 구성한다(이 옵션은
  실험적이며 변경되거나 제거될 수 있다).

**[2]**: `otel.java.exporter.memory_mode=reusable_data`를 준수하는 익스포터는
`OtlpGrpc{Signal}Exporter`, `OtlpHttp{Signal}Exporter`,
`OtlpStdout{Signal}Exporter`, `PrometheusHttpServer`이다.

`otlp` 스팬, 메트릭, 로그 익스포터를 위한 속성:

| 시스템 속성                                                | 설명                                                                                                                                                                                                                                                                                                                                                                                | 기본값                                                                                                  |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `otel.{signal}.exporter=otlp`                              | {signal}에 대해 오픈텔레메트리 익스포터를 선택한다.                                                                                                                                                                                                                                                                                                                                 |                                                                                                         |
| `otel.exporter.otlp.protocol`                              | OTLP 트레이스, 메트릭, 로그 요청에 사용할 전송 프로토콜이다. 옵션에는 `grpc`와 `http/protobuf`가 있다.                                                                                                                                                                                                                                                                              | `grpc` **[1]**                                                                                          |
| `otel.exporter.otlp.{signal}.protocol`                     | OTLP {signal} 요청에 사용할 전송 프로토콜이다. 옵션에는 `grpc`와 `http/protobuf`가 있다.                                                                                                                                                                                                                                                                                            | `grpc` **[1]**                                                                                          |
| `otel.exporter.otlp.endpoint`                              | 모든 OTLP 트레이스, 메트릭, 로그를 전송할 엔드포인트이다. 대개 오픈텔레메트리 컬렉터의 주소이다. TLS 사용 여부에 따라 `http` 또는 `https` 스킴을 가진 URL이어야 한다.                                                                                                                                                                                                               | 프로토콜이 `grpc`이면 `http://localhost:4317`, `http/protobuf`이면 `http://localhost:4318`.             |
| `otel.exporter.otlp.{signal}.endpoint`                     | OTLP {signal}을 전송할 엔드포인트이다. 대개 오픈텔레메트리 컬렉터의 주소이다. TLS 사용 여부에 따라 `http` 또는 `https` 스킴을 가진 URL이어야 한다. 프로토콜이 `http/protobuf`이면 경로에 버전과 시그널을 추가해야 한다(예: `v1/traces`, `v1/metrics`, `v1/logs`).                                                                                                                   | 프로토콜이 `grpc`이면 `http://localhost:4317`, `http/protobuf`이면 `http://localhost:4318/v1/{signal}`. |
| `otel.exporter.otlp.certificate`                           | OTLP 트레이스, 메트릭, 로그 서버의 TLS 자격 증명을 검증할 때 사용할 신뢰할 수 있는 인증서가 담긴 파일의 경로이다. 파일은 PEM 형식의 X.509 인증서를 하나 이상 포함해야 한다.                                                                                                                                                                                                         | 호스트 플랫폼의 신뢰할 수 있는 루트 인증서가 사용된다.                                                  |
| `otel.exporter.otlp.{signal}.certificate`                  | OTLP {signal} 서버의 TLS 자격 증명을 검증할 때 사용할 신뢰할 수 있는 인증서가 담긴 파일의 경로이다. 파일은 PEM 형식의 X.509 인증서를 하나 이상 포함해야 한다.                                                                                                                                                                                                                       | 호스트 플랫폼의 신뢰할 수 있는 루트 인증서가 사용된다                                                   |
| `otel.exporter.otlp.client.key`                            | OTLP 트레이스, 메트릭, 로그 클라이언트의 TLS 자격 증명을 검증할 때 사용할 개인 클라이언트 키가 담긴 파일의 경로이다. 파일은 PKCS8 PEM 형식의 개인 키 하나를 포함해야 한다.                                                                                                                                                                                                          | 클라이언트 키 파일이 사용되지 않는다.                                                                   |
| `otel.exporter.otlp.{signal}.client.key`                   | OTLP {signal} 클라이언트의 TLS 자격 증명을 검증할 때 사용할 개인 클라이언트 키가 담긴 파일의 경로이다. 파일은 PKCS8 PEM 형식의 개인 키 하나를 포함해야 한다.                                                                                                                                                                                                                        | 클라이언트 키 파일이 사용되지 않는다.                                                                   |
| `otel.exporter.otlp.client.certificate`                    | OTLP 트레이스, 메트릭, 로그 클라이언트의 TLS 자격 증명을 검증할 때 사용할 신뢰할 수 있는 인증서가 담긴 파일의 경로이다. 파일은 PEM 형식의 X.509 인증서를 하나 이상 포함해야 한다.                                                                                                                                                                                                   | 체인 파일이 사용되지 않는다.                                                                            |
| `otel.exporter.otlp.{signal}.client.certificate`           | OTLP {signal} 서버의 TLS 자격 증명을 검증할 때 사용할 신뢰할 수 있는 인증서가 담긴 파일의 경로이다. 파일은 PEM 형식의 X.509 인증서를 하나 이상 포함해야 한다.                                                                                                                                                                                                                       | 체인 파일이 사용되지 않는다.                                                                            |
| `otel.exporter.otlp.headers`                               | OTLP 트레이스, 메트릭, 로그 요청에 헤더로 전달할, 쉼표로 구분된 키-값 쌍이다.                                                                                                                                                                                                                                                                                                       |                                                                                                         |
| `otel.exporter.otlp.{signal}.headers`                      | OTLP {signal} 요청에 헤더로 전달할, 쉼표로 구분된 키-값 쌍이다.                                                                                                                                                                                                                                                                                                                     |                                                                                                         |
| `otel.exporter.otlp.compression`                           | OTLP 트레이스, 메트릭, 로그 요청에 사용할 압축 유형이다. 옵션에는 `gzip`이 있다.                                                                                                                                                                                                                                                                                                    | 압축이 사용되지 않는다.                                                                                 |
| `otel.exporter.otlp.{signal}.compression`                  | OTLP {signal} 요청에 사용할 압축 유형이다. 옵션에는 `gzip`이 있다.                                                                                                                                                                                                                                                                                                                  | 압축이 사용되지 않는다.                                                                                 |
| `otel.exporter.otlp.timeout`                               | 각 OTLP 트레이스, 메트릭, 로그 배치를 전송하는 데 허용되는 최대 대기 시간(밀리초)이다.                                                                                                                                                                                                                                                                                              | `10000`                                                                                                 |
| `otel.exporter.otlp.{signal}.timeout`                      | 각 OTLP {signal} 배치를 전송하는 데 허용되는 최대 대기 시간(밀리초)이다.                                                                                                                                                                                                                                                                                                            | `10000`                                                                                                 |
| `otel.exporter.otlp.metrics.temporality.preference`        | 선호하는 출력 집계 시간성(temporality)이다. 옵션에는 `DELTA`, `LOWMEMORY`, `CUMULATIVE`가 있다. `CUMULATIVE`이면 모든 계측기가 누적 시간성을 가진다. `DELTA`이면 카운터(동기 및 비동기)와 히스토그램은 델타가 되고, 업다운카운터(동기 및 비동기)는 누적이 된다. `LOWMEMORY`이면 동기 카운터와 히스토그램은 델타가 되고, 비동기 카운터와 업다운카운터(동기 및 비동기)는 누적이 된다. | `CUMULATIVE`                                                                                            |
| `otel.exporter.otlp.metrics.default.histogram.aggregation` | 선호하는 기본 히스토그램 집계이다. 옵션에는 `BASE2_EXPONENTIAL_BUCKET_HISTOGRAM`과 `EXPLICIT_BUCKET_HISTOGRAM`이 있다.                                                                                                                                                                                                                                                              | `EXPLICIT_BUCKET_HISTOGRAM`                                                                             |
| `otel.java.exporter.otlp.retry.disabled`                   | `false`이면 일시적인 오류가 발생할 때 재시도한다. **[2]**                                                                                                                                                                                                                                                                                                                           | `false`                                                                                                 |

**참고:** 텍스트 플레이스홀더 `{signal}`은 지원되는
[오픈텔레메트리 시그널](/docs/concepts/signals/)을 가리킨다. 유효한 값에는
`traces`, `metrics`, `logs`가 있다. 시그널별 구성은 일반 버전보다 우선한다. 예를
들어 `otel.exporter.otlp.endpoint`와 `otel.exporter.otlp.traces.endpoint`를 모두
설정하면, 후자가 우선한다.

**[1]**: 오픈텔레메트리 Java 에이전트 2.x와 오픈텔레메트리 Spring Boot 스타터는
기본으로 `http/protobuf`를 사용한다.

**[2]**: [OTLP](/docs/specs/otlp/#otlpgrpc-response)는
[일시적인](/docs/specs/otel/protocol/exporter/#retry) 오류를 재시도 전략으로
처리하도록 요구한다. 재시도가 활성화되면, 재시도 가능한 gRPC 상태 코드는 지터를
적용한 지수 백오프 알고리즘을 사용해 재시도된다. `RetryPolicy`의 구체적인 옵션은
[프로그래밍 방식 커스터마이징](#programmatic-customization)을 통해서만
커스터마이즈할 수 있다.

`zipkin` 스팬 익스포터를 위한 속성:

| 시스템 속성                     | 설명                                           | 기본값                               |
| ------------------------------- | ---------------------------------------------- | ------------------------------------ |
| `otel.traces.exporter=zipkin`   | Zipkin 익스포터를 선택한다                     |                                      |
| `otel.exporter.zipkin.endpoint` | 연결할 Zipkin 엔드포인트이다. HTTP만 지원된다. | `http://localhost:9411/api/v2/spans` |

`prometheus` 메트릭 익스포터를 위한 속성.

| 시스템 속성                        | 설명                                                           | 기본값    |
| ---------------------------------- | -------------------------------------------------------------- | --------- |
| `otel.metrics.exporter=prometheus` | Prometheus 익스포터를 선택한다                                 |           |
| `otel.exporter.prometheus.port`    | prometheus 메트릭 서버를 바인딩하는 데 사용되는 로컬 포트이다. | `9464`    |
| `otel.exporter.prometheus.host`    | prometheus 메트릭 서버를 바인딩하는 데 사용되는 로컬 주소이다. | `0.0.0.0` |

#### 프로그래밍 방식 커스터마이징 {#programmatic-customization}

프로그래밍 방식 커스터마이징은
[지원되는 속성](#environment-variables-and-system-properties)을
[프로그래밍 방식 구성](#programmatic-configuration)으로 보완할 수 있는 후크를
제공한다.

[Spring 스타터](/docs/zero-code/java/spring-boot-starter/)를 사용하는 경우
[Spring 스타터 프로그래밍 방식 구성](/docs/zero-code/java/spring-boot-starter/sdk-configuration/#programmatic-configuration)도
참고한다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomizedAutoConfiguredSdk.java"?>
```java
package otel;

import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;
import java.util.Collections;

public class CustomizedAutoConfiguredSdk {
  public static OpenTelemetrySdk autoconfiguredSdk() {
    return AutoConfiguredOpenTelemetrySdk.builder()
        // Optionally customize TextMapPropagator.
        .addPropagatorCustomizer((textMapPropagator, configProperties) -> textMapPropagator)
        // Optionally customize Resource.
        .addResourceCustomizer((resource, configProperties) -> resource)
        // Optionally customize Sampler.
        .addSamplerCustomizer((sampler, configProperties) -> sampler)
        // Optionally customize SpanExporter.
        .addSpanExporterCustomizer((spanExporter, configProperties) -> spanExporter)
        // Optionally customize SpanProcessor.
        .addSpanProcessorCustomizer((spanProcessor, configProperties) -> spanProcessor)
        // Optionally supply additional properties.
        .addPropertiesSupplier(Collections::emptyMap)
        // Optionally customize ConfigProperties.
        .addPropertiesCustomizer(configProperties -> Collections.emptyMap())
        // Optionally customize SdkTracerProviderBuilder.
        .addTracerProviderCustomizer((builder, configProperties) -> builder)
        // Optionally customize SdkMeterProviderBuilder.
        .addMeterProviderCustomizer((builder, configProperties) -> builder)
        // Optionally customize MetricExporter.
        .addMetricExporterCustomizer((metricExporter, configProperties) -> metricExporter)
        // Optionally customize MetricReader.
        .addMetricReaderCustomizer((metricReader, configProperties) -> metricReader)
        // Optionally customize SdkLoggerProviderBuilder.
        .addLoggerProviderCustomizer((builder, configProperties) -> builder)
        // Optionally customize LogRecordExporter.
        .addLogRecordExporterCustomizer((logRecordExporter, configProperties) -> logRecordExporter)
        // Optionally customize LogRecordProcessor.
        .addLogRecordProcessorCustomizer((processor, configProperties) -> processor)
        .build()
        .getOpenTelemetrySdk();
  }
}
```
<!-- prettier-ignore-end -->

#### SPI(서비스 프로바이더 인터페이스) {#spi-service-provider-interface}

[SPI](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html)(아티팩트
`io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi:{{% param vers.otel %}}`)는
SDK에 내장된 구성 요소를 넘어 SDK 자동구성을 확장한다.

다음 절에서는 사용 가능한 SPI에 대해 설명한다. 각 SPI 절은 다음을 포함한다.

- Javadoc 타입 참조 링크를 포함한 간단한 설명.
- 사용 가능한 내장 구현체와 `opentelemetry-java-contrib` 구현체의 표.
- 커스텀 구현체에 대한 간단한 시연.

##### ResourceProvider {#resourceprovider}

[ResourceProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/ResourceProvider.html)는
자동구성된 [리소스](../sdk/#resource)에 기여한다.

SDK에 내장되어 있고 커뮤니티가 `opentelemetry-java-contrib`에서 유지 관리하는
`ResourceProvider`:

| 클래스                                                                      | 아티팩트                                                                                            | 설명                                                                                        |
| --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `io.opentelemetry.sdk.autoconfigure.internal.EnvironmentResourceProvider`   | `io.opentelemetry:opentelemetry-sdk-extension-autoconfigure:{{% param vers.otel %}}`                | `OTEL_SERVICE_NAME`과 `OTEL_RESOURCE_ATTRIBUTES` 환경 변수에 기반해 리소스 속성을 제공한다. |
| `io.opentelemetry.instrumentation.resources.ContainerResourceProvider`      | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | 컨테이너 리소스 속성을 제공한다.                                                            |
| `io.opentelemetry.instrumentation.resources.HostResourceProvider`           | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | 호스트 리소스 속성을 제공한다.                                                              |
| `io.opentelemetry.instrumentation.resources.HostIdResourceProvider`         | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | 호스트 ID 리소스 속성을 제공한다.                                                           |
| `io.opentelemetry.instrumentation.resources.ManifestResourceProvider`       | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | jar 매니페스트에 기반해 서비스 리소스 속성을 제공한다.                                      |
| `io.opentelemetry.instrumentation.resources.OsResourceProvider`             | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | OS 리소스 속성을 제공한다.                                                                  |
| `io.opentelemetry.instrumentation.resources.ProcessResourceProvider`        | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | 프로세스 리소스 속성을 제공한다.                                                            |
| `io.opentelemetry.instrumentation.resources.ProcessRuntimeResourceProvider` | `io.opentelemetry.instrumentation:opentelemetry-resources:{{% param vers.instrumentation %}}-alpha` | 프로세스 런타임 리소스 속성을 제공한다.                                                     |
| `io.opentelemetry.contrib.gcp.resource.GCPResourceProvider`                 | `io.opentelemetry.contrib:opentelemetry-gcp-resources:{{% param vers.contrib %}}-alpha`             | GCP 런타임 환경 리소스 속성을 제공한다.                                                     |
| `io.opentelemetry.contrib.aws.resource.BeanstalkResourceProvider`           | `io.opentelemetry.contrib:opentelemetry-aws-resources:{{% param vers.contrib %}}-alpha`             | AWS beanstalk 런타임 환경 리소스 속성을 제공한다.                                           |
| `io.opentelemetry.contrib.aws.resource.Ec2ResourceProvider`                 | `io.opentelemetry.contrib:opentelemetry-aws-resources:{{% param vers.contrib %}}-alpha`             | AWS ec2 런타임 환경 리소스 속성을 제공한다.                                                 |
| `io.opentelemetry.contrib.aws.resource.EcsResourceProvider`                 | `io.opentelemetry.contrib:opentelemetry-aws-resources:{{% param vers.contrib %}}-alpha`             | AWS ecs 런타임 환경 리소스 속성을 제공한다.                                                 |
| `io.opentelemetry.contrib.aws.resource.EksResourceProvider`                 | `io.opentelemetry.contrib:opentelemetry-aws-resources:{{% param vers.contrib %}}-alpha`             | AWS eks 런타임 환경 리소스 속성을 제공한다.                                                 |
| `io.opentelemetry.contrib.aws.resource.LambdaResourceProvider`              | `io.opentelemetry.contrib:opentelemetry-aws-resources:{{% param vers.contrib %}}-alpha`             | AWS lambda 런타임 환경 리소스 속성을 제공한다.                                              |

리소스 자동구성에 참여하려면 `ResourceProvider` 인터페이스를 구현한다. 예를
들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomResourceProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider;
import io.opentelemetry.sdk.resources.Resource;

public class CustomResourceProvider implements ResourceProvider {

  @Override
  public Resource createResource(ConfigProperties config) {
    // Callback invoked to contribute to the resource.
    return Resource.builder().put("my.custom.resource.attribute", "abc123").build();
  }

  @Override
  public int order() {
    // Optionally influence the order of invocation.
    return 0;
  }
}
```
<!-- prettier-ignore-end -->

##### AutoConfigurationCustomizerProvider {#autoconfigurationcustomizerprovider}

다양한 자동구성된 SDK 구성 요소를 커스터마이즈하려면
[AutoConfigurationCustomizerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/AutoConfigurationCustomizerProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomizerProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizer;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;
import java.util.Collections;

public class CustomizerProvider implements AutoConfigurationCustomizerProvider {

  @Override
  public void customize(AutoConfigurationCustomizer customizer) {
    // Optionally customize TextMapPropagator.
    customizer.addPropagatorCustomizer((textMapPropagator, configProperties) -> textMapPropagator);
    // Optionally customize Resource.
    customizer.addResourceCustomizer((resource, configProperties) -> resource);
    // Optionally customize Sampler.
    customizer.addSamplerCustomizer((sampler, configProperties) -> sampler);
    // Optionally customize SpanExporter.
    customizer.addSpanExporterCustomizer((spanExporter, configProperties) -> spanExporter);
    // Optionally customize SpanProcessor.
    customizer.addSpanProcessorCustomizer((spanProcessor, configProperties) -> spanProcessor);
    // Optionally supply additional properties.
    customizer.addPropertiesSupplier(Collections::emptyMap);
    // Optionally customize ConfigProperties.
    customizer.addPropertiesCustomizer(configProperties -> Collections.emptyMap());
    // Optionally customize SdkTracerProviderBuilder.
    customizer.addTracerProviderCustomizer((builder, configProperties) -> builder);
    // Optionally customize SdkMeterProviderBuilder.
    customizer.addMeterProviderCustomizer((builder, configProperties) -> builder);
    // Optionally customize MetricExporter.
    customizer.addMetricExporterCustomizer((metricExporter, configProperties) -> metricExporter);
    // Optionally customize MetricReader.
    customizer.addMetricReaderCustomizer((metricReader, configProperties) -> metricReader);
    // Optionally customize SdkLoggerProviderBuilder.
    customizer.addLoggerProviderCustomizer((builder, configProperties) -> builder);
    // Optionally customize LogRecordExporter.
    customizer.addLogRecordExporterCustomizer((exporter, configProperties) -> exporter);
    // Optionally customize LogRecordProcessor.
    customizer.addLogRecordProcessorCustomizer((processor, configProperties) -> processor);
  }

  @Override
  public int order() {
    // Optionally influence the order of invocation.
    return 0;
  }
}
```
<!-- prettier-ignore-end -->

##### ConfigurableSpanExporterProvider {#configurablespanexporterprovider}

커스텀 스팬 익스포터가 자동구성에 참여할 수 있게 하려면
[ConfigurableSpanExporterProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/traces/ConfigurableSpanExporterProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomSpanExporterProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider;
import io.opentelemetry.sdk.trace.export.SpanExporter;

public class CustomSpanExporterProvider implements ConfigurableSpanExporterProvider {

  @Override
  public SpanExporter createExporter(ConfigProperties config) {
    // Callback invoked when OTEL_TRACES_EXPORTER includes the value from getName().
    return new CustomSpanExporter();
  }

  @Override
  public String getName() {
    return "custom-exporter";
  }
}
```
<!-- prettier-ignore-end -->

##### ConfigurableMetricExporterProvider {#configurablemetricexporterprovider}

커스텀 메트릭 익스포터가 자동구성에 참여할 수 있게 하려면
[ConfigurableMetricExporterProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/metrics/ConfigurableMetricExporterProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomMetricExporterProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.metrics.ConfigurableMetricExporterProvider;
import io.opentelemetry.sdk.metrics.export.MetricExporter;

public class CustomMetricExporterProvider implements ConfigurableMetricExporterProvider {

  @Override
  public MetricExporter createExporter(ConfigProperties config) {
    // Callback invoked when OTEL_METRICS_EXPORTER includes the value from getName().
    return new CustomMetricExporter();
  }

  @Override
  public String getName() {
    return "custom-exporter";
  }
}
```
<!-- prettier-ignore-end -->

##### ConfigurableLogRecordExporterProvider {#configurablelogrecordexporterprovider}

커스텀 로그 레코드 익스포터가 자동구성에 참여할 수 있게 하려면
[ConfigurableLogRecordExporterProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/logs/ConfigurableLogRecordExporterProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomLogRecordExporterProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.logs.ConfigurableLogRecordExporterProvider;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;

public class CustomLogRecordExporterProvider implements ConfigurableLogRecordExporterProvider {

  @Override
  public LogRecordExporter createExporter(ConfigProperties config) {
    // Callback invoked when OTEL_LOGS_EXPORTER includes the value from getName().
    return new CustomLogRecordExporter();
  }

  @Override
  public String getName() {
    return "custom-exporter";
  }
}
```
<!-- prettier-ignore-end -->

##### ConfigurableSamplerProvider {#configurablesamplerprovider}

커스텀 샘플러가 자동구성에 참여할 수 있게 하려면
[ConfigurableSamplerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/traces/ConfigurableSamplerProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomSamplerProvider.java"?>
```java
package otel;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider;
import io.opentelemetry.sdk.trace.samplers.Sampler;

public class CustomSamplerProvider implements ConfigurableSamplerProvider {

  @Override
  public Sampler createSampler(ConfigProperties config) {
    // Callback invoked when OTEL_TRACES_SAMPLER is set to the value from getName().
    return new CustomSampler();
  }

  @Override
  public String getName() {
    return "custom-sampler";
  }
}
```
<!-- prettier-ignore-end -->

##### ConfigurablePropagatorProvider {#configurablepropagatorprovider}

커스텀 전파자가 자동구성에 참여할 수 있게 하려면
[ConfigurablePropagatorProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/latest/io/opentelemetry/sdk/autoconfigure/spi/ConfigurablePropagatorProvider.html)
인터페이스를 구현한다. 예를 들면:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CustomTextMapPropagatorProvider.java"?>
```java
package otel;

import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigurablePropagatorProvider;

public class CustomTextMapPropagatorProvider implements ConfigurablePropagatorProvider {
  @Override
  public TextMapPropagator getPropagator(ConfigProperties config) {
    // Callback invoked when OTEL_PROPAGATORS includes the value from getName().
    return new CustomTextMapPropagator();
  }

  @Override
  public String getName() {
    return "custom-propagator";
  }
}
```
<!-- prettier-ignore-end -->

### 선언적 구성 {#declarative-configuration}

선언적 구성은 현재 개발 중이다. 이는
[opentelemetry-configuration](https://github.com/open-telemetry/opentelemetry-configuration)과
[선언적 구성](/docs/specs/otel/configuration/#declarative-configuration)에
설명된 대로 YAML 파일 기반 구성을 가능하게 한다.

사용하려면
`io.opentelemetry:opentelemetry-sdk-extension-incubator:{{% param vers.otel %}}-alpha`를
포함시키고, 아래 표에 설명된 대로 구성 파일의 경로를 지정한다.

| 시스템 속성                     | 목적                      | 기본값 |
| ------------------------------- | ------------------------- | ------ |
| `otel.experimental.config.file` | SDK 구성 파일의 경로이다. | 미설정 |

> [!WARNING]
>
> 구성 파일이 지정되면
> [환경 변수와 시스템 속성](#environment-variables-and-system-properties)은
> 무시되며, [프로그래밍 방식 커스터마이징](#programmatic-customization)과
> [SPI](#spi-service-provider-interface)는 건너뛴다. SDK 구성은 오직 파일의
> 내용만으로 결정된다.

추가적인 세부 정보는 다음 리소스를 참고한다.

- [사용 문서](https://github.com/open-telemetry/opentelemetry-java/tree/main/sdk-extensions/declarative-config)
- [Java 에이전트를 사용하는 예제](https://github.com/open-telemetry/opentelemetry-java-examples/tree/main/javaagent-declarative-configuration)
- [Java 에이전트를 사용하지 않는 예제](https://github.com/open-telemetry/opentelemetry-java-examples/tree/main/declarative-configuration)
