---
title: Java 에이전트 선언적 구성
linkTitle: 선언적 구성
weight: 11
cSpell:ignore: Customizer Dotel
default_lang_commit: dc7446bf09ad87c04654d71faf195857922e8eff
---

선언적 구성은 환경 변수나 시스템 속성 대신 YAML 파일을 사용한다.

이 접근 방식은 다음과 같은 경우에 유용하다.

- 설정할 구성 옵션이 많은 경우
- 환경 변수나 시스템 속성으로는 사용할 수 없는 구성 옵션을 쓰고 싶은 경우

환경 변수와 마찬가지로, 이 구성 문법은 언어 중립적이며
오픈텔레메트리(OpenTelemetry) Java 에이전트를 포함해 선언적 구성을 지원하는 모든
오픈텔레메트리 Java SDK에서 동작한다.

> [!WARNING]
>
> 선언적 구성 스키마는 안정적이다. 아직 실험적인 부분에는 `/development`
> 접미사가 붙어 있다. 선언적 구성에 대한 Java 에이전트 지원은 아직 실험적이다.

## 지원 버전 {#supported-versions}

선언적 구성은 **오픈텔레메트리 Java 에이전트 버전 2.26.0 이상**에서 지원된다.

## 시작하기 {#getting-started}

1. 아래 구성 파일을 `otel-config.yaml`로 저장한다.
2. JVM 시작 인자에 다음을 추가한다.<br>
   `-Dotel.config.file=/path/to/otel-config.yaml`

```yaml
file_format: '1.0'

resource:
  attributes_list: ${OTEL_RESOURCE_ATTRIBUTES}
  detection/development:
    detectors:
      - service: # will add "service.instance.id" and "service.name" from OTEL_SERVICE_NAME

propagator:
  composite:
    - tracecontext:
    - baggage:

tracer_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_TRACES_ENDPOINT:-http://localhost:4318/v1/traces}

meter_provider:
  readers:
    - periodic:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_METRICS_ENDPOINT:-http://localhost:4318/v1/metrics}

logger_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_LOGS_ENDPOINT:-http://localhost:4318/v1/logs}
```

선언적 구성에 대한 더 일반적인 시작 가이드는 [SDK 선언적
구성][SDK Declarative configuration] 문서를 참조한다.

이 페이지는
[오픈텔레메트리 Java 에이전트](https://github.com/open-telemetry/opentelemetry-java-instrumentation)의
세부 사항에 초점을 맞춘다. Spring Boot 스타터에 대해서는
[Spring Boot 스타터 선언적 구성](/docs/zero-code/java/spring-boot-starter/declarative-configuration/)을
참고한다.

## 기존 구성 변환하기 {#convert-your-existing-configuration}

{{< dc-converter source="agent" >}}

## 구성 옵션 매핑 {#mapping-of-configuration-options}

기존 환경 변수나 시스템 속성 구성을 선언적 구성으로 매핑하고 싶을 때는 다음
규칙을 사용한다.

1. 구성 옵션이 `otel.javaagent.`로 시작한다면(예: `otel.javaagent.logging`),
   이는 대부분 환경 변수나 시스템 속성으로만 설정할 수 있는 속성이다(자세한
   내용은 아래
   [환경 변수 및 시스템 속성 전용 옵션](#environment-variables-and-system-properties-only-options)
   섹션 참고). 그렇지 않으면 `otel.javaagent.` 접두사를 제거하고 아래 `agent`
   섹션 아래에 둔다.
2. 구성 옵션이 `otel.instrumentation.`로 시작한다면(예:
   `otel.instrumentation.spring-batch.experimental.chunk.new-trace`),
   `otel.instrumentation.` 접두사를 제거하고 아래 `instrumentation` 섹션 아래에
   둔다.
3. 그 밖의 경우, 그 옵션은 대부분 SDK 구성에 속한다.
   [마이그레이션 구성](https://github.com/open-telemetry/opentelemetry-configuration/blob/main/examples/otel-sdk-migration-config.yaml)에서
   올바른 섹션을 찾는다. `otel.bsp.schedule.delay` 같은 시스템 속성이 있다면,
   마이그레이션 구성에서 대응하는 환경 변수 `OTEL_BSP_SCHEDULE_DELAY`를 찾는다.
4. `.`를 사용해 들여쓰기 수준을 만든다.
5. `-`를 `_`로 변환한다.
6. 적절한 곳에 YAML 불리언 및 정수 타입을 사용한다(예: `"true"` 대신 `true`,
   `"5000"` 대신 `5000`).
7. 특수 매핑이 있는 옵션은 아래에서 따로 설명한다.

```yaml
instrumentation/development:
  general:
    http:
      client:
        request_captured_headers: # was otel.instrumentation.http.client.capture-request-headers
          - Content-Type
          - Accept
        response_captured_headers: # was otel.instrumentation.http.client.capture-response-headers
          - Content-Type
          - Content-Encoding
      server:
        request_captured_headers: # was otel.instrumentation.http.server.capture-request-headers
          - Content-Type
          - Accept
        response_captured_headers: # was otel.instrumentation.http.server.capture-response-headers
          - Content-Type
          - Content-Encoding

  java:
    common:
      service_mapping: # was "otel.instrumentation.common.peer-service-mapping"
        - peer: 1.2.3.4
          service_name: FooService
        - peer: 2.3.4.5
          service_name: BarService

    agent:
      # was otel.instrumentation.common.default-enabled
      # instrumentation_mode: none  # was false
      instrumentation_mode: default # was true
    spring_batch:
      experimental:
        chunk:
          new_trace: true
```

에이전트별 옵션(`otel.javaagent.`로 시작)은 `distribution` 섹션 아래에 둔다.

```yaml
distribution:
  javaagent:
    instrumentation:
      default_enabled: false # was otel.instrumentation.common.default-enabled
      enabled:
        - tomcat
        - spring_webmvc
      disabled:
        - armeria_grpc
    exclude_classes: # was otel.javaagent.exclude-classes
      - com.example.excluded.Class1
    exclude_class_loaders: # was otel.javaagent.exclude-class-loaders
      - com.example.ExcludedClassLoader
```

## 환경 변수 및 시스템 속성 전용 옵션 {#environment-variables-and-system-properties-only-options}

다음 구성 옵션은 선언적 구성이 지원하지만, 환경 변수나 시스템 속성으로만 사용할
수 있다.

- `otel.javaagent.configuration-file`(단, 선언적 구성에서는 필요하지 않아야 함)
- `otel.javaagent.debug`
- `otel.javaagent.enabled`
- `otel.javaagent.experimental.field-injection.enabled`
- `otel.javaagent.experimental.security-manager-support.enabled`
- `otel.javaagent.extensions`
- `otel.javaagent.logging.application.logs-buffer-max-records`
- `otel.javaagent.logging`

이 옵션들은 선언적 구성 파일을 읽기 전, 에이전트 시작 시점에 필요하다.

## 지속 시간 형식 {#duration-format}

- 선언적 구성은 **밀리초 단위의 지속 시간만 지원한다**(예: 5초는 `5000`).
- `OTEL_BSP_SCHEDULE_DELAY=5s`를 사용하면 오류가 발생한다(환경 변수에는
  유효하지만 선언적 구성에는 유효하지 않음).

예시:

```yaml
tracer_provider:
  processors:
    - batch:
        schedule_delay: ${OTEL_BSP_SCHEDULE_DELAY:-5000}
```

## 동작 차이 {#behavior-differences}

- (Java 에이전트가 기본적으로 추가하는) 리소스 속성 `telemetry.distro.name`은
  `opentelemetry-java-instrumentation`이 아니라 `opentelemetry-javaagent` 값을
  가진다(3.0 릴리스에서 다시 정렬될 예정).

## 아직 지원되지 않는 기능 {#not-yet-supported-features}

### 아직 환경 변수나 시스템 속성이 필요한 기능 {#features-that-still-need-environment-variables-or-system-properties}

환경 변수와 시스템 속성으로는 지원되지만 아직 선언적 구성으로는 지원되지 않는
기능이 일부 있다.

다음 설정은 여전히 환경 변수나 시스템 속성으로 설정해야 한다.

- `otel.javaagent.experimental.thread-propagation-debugger.enabled`

### 아직 전혀 지원되지 않는 기능 {#features-not-yet-supported-at-all}

아직 선언적 구성으로 지원되지 않는 Java 에이전트 기능:

- `otel.javaagent.add-thread-details`

아직 선언적 구성으로 지원되지 않는 Contrib 기능:

- [AWS X-Ray](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/aws-xray)
- [GCP 인증](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/gcp-auth-extension)
- [Inferred Spans](https://github.com/open-telemetry/opentelemetry-java-contrib/blob/main/inferred-spans)

## 익스텐션 API {#extension-api}

익스텐션은 새로운 선언적 구성 API를 사용한다.

- `AutoConfigurationCustomizerProvider`를 사용하는 익스텐션은 새로운
  `DeclarativeConfigurationCustomizerProvider` API로 마이그레이션해야 한다. 기존
  [AgentTracerProviderConfigurer](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/javaagent-tooling/src/main/java/io/opentelemetry/javaagent/tooling/AgentTracerProviderConfigurer.java)가
  새로운
  [SpanLoggingCustomizerProvider](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/javaagent-tooling/src/main/java/io/opentelemetry/javaagent/tooling/SpanLoggingCustomizerProvider.java)에
  어떻게 매핑되는지 확인한다.
- 스팬 익스포터 같은 컴포넌트는 이제 `ComponentProvider` API를 사용해야 한다.
  예시로, 기존 API와 새 API를 모두 지원하는
  [Baggage Processor](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/baggage-processor)를
  확인한다.

[SDK Declarative configuration]:
  /docs/languages/sdk-configuration/declarative-configuration
