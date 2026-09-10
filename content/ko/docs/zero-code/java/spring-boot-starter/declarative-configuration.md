---
title: 선언적 구성
weight: 25
cSpell:ignore: Customizer Dotel genai sqlcommenter
default_lang_commit: 8c95bffcf7243a916f79a0d525cf55b6a3d34ad7
---

선언적 구성은 `application.yaml` 안에서
[오픈텔레메트리(OpenTelemetry) 선언적 구성 스키마](/docs/languages/sdk-configuration/declarative-configuration/)를
사용한다.

이 방식은 다음과 같은 경우에 유용하다:

- 설정할 구성 옵션이 많은 경우
- `application.properties`나 `application.yaml`에서 사용할 수 없는 구성 옵션을
  사용하려는 경우
- [Java 에이전트](/docs/zero-code/java/agent/declarative-configuration/)와
  동일한 구성 형식을 사용하려는 경우

> [!WARNING]
>
> 선언적 구성은 실험적 기능이다.

선언적 구성이 Spring Boot 스타터에 어떻게 들어맞는지에 대한 배경은 블로그 게시물
[The Voyage of a Small Environment Variable](/blog/2026/spring-boot-declarative-config/)을
참고한다.

## 지원되는 버전 {#supported-versions}

선언적 구성은 **오픈텔레메트리 Spring Boot 스타터 버전 2.26.0 이상**에서
지원된다.

## 의존성 관리 {#dependency-management}

[시작하기](../getting-started/#dependency-management) 페이지에서 설명한 대로
`dependencyManagement`에 `opentelemetry-instrumentation-bom`을 임포트해야 한다.

> [!IMPORTANT]
>
> Spring Boot 3.5 이상에서는 `opentelemetry-instrumentation-bom` 임포트가
> **필수**이다. Spring Boot 3.5 이상은 자체 오픈텔레메트리 의존성 관리를
> 포함하며, 이는 `opentelemetry-api`를 선언적 구성에 필요한
> `io.opentelemetry.common.ComponentLoader`가 포함되지 않은 버전으로 고정한다.
> BOM이 없으면 애플리케이션이
> `NoClassDefFoundError: io/opentelemetry/common/ComponentLoader` 오류와 함께
> 시작에 실패한다.
>
> Maven을 사용할 때는 오픈텔레메트리 BOM의 버전이 우선하도록 Spring Boot 부모 /
> `spring-boot-dependencies` BOM보다 **먼저** 임포트해야 한다.

## 시작하기 {#getting-started}

선언적 구성을 사용하려면 `application.yaml`에 `otel.file_format: "1.0"`(또는
현재 버전이나 원하는 버전)을 추가한다:

```yaml
otel:
  file_format: '1.0'

  resource:
    detection/development:
      detectors:
        - service:
    attributes:
      - name: service.name
        value: my-spring-app

  propagator:
    composite:
      - tracecontext:
      - baggage:

  tracer_provider:
    processors:
      - batch:
          exporter:
            otlp_http:
              endpoint: ${OTEL_EXPORTER_OTLP_TRACES_ENDPOINT:http://localhost:4318/v1/traces}

  meter_provider:
    readers:
      - periodic:
          exporter:
            otlp_http:
              endpoint: ${OTEL_EXPORTER_OTLP_METRICS_ENDPOINT:http://localhost:4318/v1/metrics}

  logger_provider:
    processors:
      - batch:
          exporter:
            otlp_http:
              endpoint: ${OTEL_EXPORTER_OTLP_LOGS_ENDPOINT:http://localhost:4318/v1/logs}
```

`${VAR:default}`는 에이전트의 독립 실행형 YAML 파일에서 사용하는
`${VAR:-default}` 구문이 아니라 단일 콜론(Spring 구문)을 사용함에 유의한다.

## 기존 구성 변환하기 {#convert-your-existing-configuration}

{{< dc-converter source="spring" >}}

## 구성 옵션 매핑 {#mapping-of-configuration-options}

다음 규칙은 `application.properties` / `application.yaml` 구성 옵션이 이에
상응하는 선언적 구성으로 매핑되는 방식을 설명한다:

### 계측 활성화/비활성화 {#instrumentation-enabledisable}

선언적 구성에서는 계측 활성화/비활성화에 개별 프로퍼티 대신 중앙 집중식 목록을
사용한다. 계측 이름은 `-`(kebab-case)가 아니라 `_`(snake_case)를 사용한다.

| 프로퍼티                                              | 선언적 구성                                                                     |
| ----------------------------------------------------- | ------------------------------------------------------------------------------- |
| `otel.instrumentation.jdbc.enabled=true`              | `otel.distribution.spring_starter.instrumentation.enabled: [jdbc]`              |
| `otel.instrumentation.logback-appender.enabled=false` | `otel.distribution.spring_starter.instrumentation.disabled: [logback_appender]` |
| `otel.instrumentation.common.default-enabled=false`   | `otel.distribution.spring_starter.instrumentation.default_enabled: false`       |

예시:

```yaml
otel:
  distribution:
    spring_starter:
      instrumentation:
        default_enabled: false
        enabled:
          - jdbc
          - spring_web
        disabled:
          - logback_appender
```

### 계측 구성 {#instrumentation-configuration}

`otel.instrumentation.*` 아래의 구성 옵션(활성화/비활성화 제외)은
`otel.instrumentation/development.java.*`로 매핑된다:

1. `otel.instrumentation.` 접두사를 제거한다
2. 각 세그먼트별로 `-`를 `_`로 바꾼다
3. `otel.instrumentation/development.java.` 아래에 배치한다
4. 키에 붙은 `/development` 접미사는 실험적 기능을 나타낸다(역방향 매핑은
   [`ConfigPropertiesBackedDeclarativeConfigProperties`](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/declarative-config-bridge/src/main/java/io/opentelemetry/instrumentation/config/bridge/ConfigPropertiesBackedDeclarativeConfigProperties.java)의
   `translateName` 메서드를 참고한다)

예를 들면:

| 프로퍼티                                                            | 선언적 구성                                                                                      |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `otel.instrumentation.logback-appender.experimental-log-attributes` | `otel.instrumentation/development.java.logback_appender.experimental_log_attributes/development` |

일부 옵션은 기본 알고리즘을 따르지 않는 특수한 매핑을 가진다:

| 프로퍼티                                                                        | 선언적 구성                                                                                        |
| ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `otel.instrumentation.common.db.query-sanitization.enabled`                     | `otel.instrumentation/development.java.common.db.query_sanitization.enabled`                       |
| `otel.instrumentation.common.db-statement-sanitizer.enabled` (deprecated)       | `otel.instrumentation/development.java.common.db_statement_sanitizer.enabled`                      |
| `otel.instrumentation.common.db.experimental.sqlcommenter.enabled`              | `otel.instrumentation/development.java.common.db.sqlcommenter/development.enabled`                 |
| `otel.instrumentation.http.client.capture-request-headers`                      | `otel.instrumentation/development.general.http.client.request_captured_headers`                    |
| `otel.instrumentation.http.client.capture-response-headers`                     | `otel.instrumentation/development.general.http.client.response_captured_headers`                   |
| `otel.instrumentation.http.server.capture-request-headers`                      | `otel.instrumentation/development.general.http.server.request_captured_headers`                    |
| `otel.instrumentation.http.server.capture-response-headers`                     | `otel.instrumentation/development.general.http.server.response_captured_headers`                   |
| `otel.instrumentation.http.client.emit-experimental-telemetry`                  | `otel.instrumentation/development.java.common.http.client.emit_experimental_telemetry/development` |
| `otel.instrumentation.http.server.emit-experimental-telemetry`                  | `otel.instrumentation/development.java.common.http.server.emit_experimental_telemetry/development` |
| `otel.instrumentation.http.known-methods`                                       | `otel.instrumentation/development.java.common.http.known_methods`                                  |
| `otel.instrumentation.messaging.experimental.receive-telemetry.enabled`         | `otel.instrumentation/development.java.common.messaging.receive_telemetry/development.enabled`     |
| `otel.instrumentation.messaging.experimental.capture-headers`                   | `otel.instrumentation/development.java.common.messaging.capture_headers/development`               |
| `otel.instrumentation.genai.capture-message-content`                            | `otel.instrumentation/development.java.common.gen_ai.capture_message_content`                      |
| `otel.instrumentation.sanitization.url.experimental.sensitive-query-parameters` | `otel.instrumentation/development.general.sanitization.url.sensitive_query_parameters/development` |
| `otel.semconv-stability.opt-in`                                                 | `otel.instrumentation/development.general.semconv_stability.opt_in`                                |
| `otel.semconv.exception.signal.preview`                                         | `otel.instrumentation/development.general.semconv_exception.signal.preview`                        |
| `otel.instrumentation.experimental.span-suppression-strategy`                   | `otel.instrumentation/development.java.common.span_suppression_strategy/development`               |
| `otel.instrumentation.opentelemetry-annotations.exclude-methods`                | `otel.instrumentation/development.java.opentelemetry_extension_annotations.exclude_methods`        |
| `otel.experimental.javascript-snippet`                                          | `otel.instrumentation/development.java.servlet.javascript_snippet/development`                     |
| `otel.jmx.enabled`                                                              | `otel.instrumentation/development.java.jmx.enabled`                                                |
| `otel.jmx.config`                                                               | `otel.instrumentation/development.java.jmx.config`                                                 |
| `otel.jmx.discovery.delay`                                                      | `otel.instrumentation/development.java.jmx.discovery.delay`                                        |
| `otel.jmx.target.system`                                                        | `otel.instrumentation/development.java.jmx.target.system`                                          |

`instrumentation/development` 섹션에는 두 개의 최상위 그룹이 있다:

- `general.*` — 언어 간 공통 구성(HTTP 헤더, 시맨틱 컨벤션 안정성)
- `java.*` — Java 전용 계측 구성

### SDK 비활성화 {#disable-the-sdk}

| 프로퍼티                 | 선언적 구성           |
| ------------------------ | --------------------- |
| `otel.sdk.disabled=true` | `otel.disabled: true` |

### SDK 구성 {#sdk-configuration}

SDK 수준 구성(익스포터, 전파자, 리소스)은 [시작하기](#getting-started)
예제에서처럼 표준
[선언적 구성 스키마](/docs/languages/sdk-configuration/declarative-configuration/)를
`otel:` 바로 아래에서 따른다.

## 에이전트 선언적 구성과의 차이점 {#differences-from-agent-declarative-configuration}

| 항목            | 에이전트                                                 | Spring Boot 스타터                                            |
| --------------- | -------------------------------------------------------- | ------------------------------------------------------------- |
| 구성 위치       | 별도 파일 (`-Dotel.config.file=...`)                     | `application.yaml` 내부                                       |
| 변수 구문       | `${VAR:-default}` (이중 콜론)                            | `${VAR:default}` (단일 콜론, Spring)                          |
| 프로파일        | 지원 안 함                                               | Spring 프로파일이 정상 동작                                   |
| 활성화/비활성화 | `distribution.javaagent.instrumentation.*`               | `distribution.spring_starter.instrumentation.*`               |
| 기본 활성화     | `distribution.javaagent.instrumentation.default_enabled` | `distribution.spring_starter.instrumentation.default_enabled` |

## 환경 변수 오버라이드 {#environment-variable-overrides}

Spring의 완화된 바인딩(relaxed binding)을 사용하면 환경 변수로 선언적 구성
YAML의 어느 부분이든 오버라이드할 수 있다:

```shell
# instrumentation/development 아래의 스칼라 값을 오버라이드
OTEL_INSTRUMENTATION/DEVELOPMENT_JAVA_FOO_STRING_KEY=new_value

# 인덱스가 있는 목록 요소를 오버라이드 (예: 익스포터 엔드포인트)
OTEL_TRACER_PROVIDER_PROCESSORS_0_BATCH_EXPORTER_OTLP_HTTP_ENDPOINT=http://custom:4318/v1/traces
```

규칙: 대문자로 바꾸고, `.`를 `_`로 바꾸며, `/`는 그대로 유지한다(예:
`INSTRUMENTATION/DEVELOPMENT`). 목록 인덱스에는 `_0_`, `_1_`을 사용한다.

이것은 Spring의 표준 기능이며, `application.yaml`의 모든 키에 대해 동작한다.

## 기간 형식 {#duration-format}

선언적 구성은 **기간을 밀리초 단위로만 지원한다**(예: 5초는 `5000`). `5s`와 같은
기간 문자열을 사용하면 오류가 발생한다.

## 프로그래밍 방식 구성 {#programmatic-configuration}

선언적 구성에서는
`AutoConfigurationCustomizerProvider`([프로그래밍 방식 구성](../programmatic-configuration/)
참고)가 `DeclarativeConfigurationCustomizerProvider`로 대체된다. 스팬 익스포터와
같은 컴포넌트는 `ComponentProvider` API를 사용한다. 자세한 내용과 예제는
[에이전트 확장 API 섹션](/docs/zero-code/java/agent/declarative-configuration/)을
참고한다. 동일한 API가 Spring Boot 스타터에도 적용된다.
