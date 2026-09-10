---
title: 기본 제공 계측
weight: 40
cSpell:ignore: autoconfigurations autoconfigures webflux webmvc
default_lang_commit: 2d89b60b2e09d42ba96757b0afdbc31f54a2b0e7
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/spring-starter"?>

여러 프레임워크에 대해 기본 제공 계측(out-of-the-box instrumentation)을 사용할
수 있다:

{{< tabpane text=true >}} {{% tab "Properties" %}}

| 기능                  | 프로퍼티                                        | 기본값 |
| --------------------- | ----------------------------------------------- | ------ |
| JDBC                  | `otel.instrumentation.jdbc.enabled`             | true   |
| Logback               | `otel.instrumentation.logback-appender.enabled` | true   |
| Logback MDC           | `otel.instrumentation.logback-mdc.enabled`      | true   |
| Spring Web            | `otel.instrumentation.spring-web.enabled`       | true   |
| Spring Web MVC        | `otel.instrumentation.spring-webmvc.enabled`    | true   |
| Spring WebFlux        | `otel.instrumentation.spring-webflux.enabled`   | true   |
| Kafka                 | `otel.instrumentation.kafka.enabled`            | true   |
| MongoDB               | `otel.instrumentation.mongo.enabled`            | true   |
| Micrometer            | `otel.instrumentation.micrometer.enabled`       | false  |
| R2DBC (reactive JDBC) | `otel.instrumentation.r2dbc.enabled`            | true   |

특정 계측을 비활성화하려면:

```yaml
otel:
  instrumentation:
    logback-appender:
      enabled: false
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

[선언적 구성(declarative configuration)](../declarative-configuration/)에서는
계측 활성화/비활성화에 `otel.distribution.spring_starter.instrumentation` 아래의
중앙 집중식 목록을 사용한다. 계측 이름은 `-`(kebab-case)가 아니라
`_`(snake_case)를 사용한다.

| 기능                  | 이름               | 기본값   |
| --------------------- | ------------------ | -------- |
| JDBC                  | `jdbc`             | enabled  |
| Logback               | `logback_appender` | enabled  |
| Logback MDC           | `logback_mdc`      | enabled  |
| Spring Web            | `spring_web`       | enabled  |
| Spring Web MVC        | `spring_webmvc`    | enabled  |
| Spring WebFlux        | `spring_webflux`   | enabled  |
| Kafka                 | `kafka`            | enabled  |
| MongoDB               | `mongo`            | enabled  |
| Micrometer            | `micrometer`       | disabled |
| R2DBC (reactive JDBC) | `r2dbc`            | enabled  |

특정 계측을 비활성화하려면:

```yaml
otel:
  distribution:
    spring_starter:
      instrumentation:
        disabled:
          - logback_appender
```

{{% /tab %}} {{< /tabpane >}}

## 계측을 선택적으로 켜기 {#turn-on-instrumentations-selectively}

{{< tabpane text=true >}} {{% tab "Properties" %}}

특정 계측만 사용하려면, 먼저 모든 계측을 끈 다음 필요한 계측을 하나씩 켠다:

```yaml
otel:
  instrumentation:
    common:
      default-enabled: false
    jdbc:
      enabled: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

[선언적 구성](../declarative-configuration/)에서는 `default_enabled`를 `false`로
설정하고, 원하는 계측을 `enabled`에 나열한다:

```yaml
otel:
  distribution:
    spring_starter:
      instrumentation:
        default_enabled: false
        enabled:
          - jdbc
```

{{% /tab %}} {{< /tabpane >}}

## 공통 계측 구성 {#common-instrumentation-configuration}

모든 데이터베이스 계측에 공통으로 적용되는 프로퍼티:

{{< tabpane text=true >}} {{% tab "Properties" %}}

모든 데이터베이스 계측에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation:
    common:
      db-statement-sanitizer:
        enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

모든 데이터베이스 계측에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation/development:
    java:
      common:
        database:
          statement_sanitizer:
            enabled: true
```

{{% /tab %}} {{< /tabpane >}}

## JDBC 계측 {#jdbc-instrumentation}

{{< tabpane text=true >}} {{% tab "Properties" %}}

JDBC에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation:
    jdbc:
      statement-sanitizer:
        enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

JDBC에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation/development:
    java:
      jdbc:
        statement_sanitizer:
          enabled: true
```

{{% /tab %}} {{< /tabpane >}}

## Logback {#logback}

시스템 프로퍼티로 실험적 기능을 활성화해 속성을 캡처할 수 있다:

{{< tabpane text=true >}} {{% tab "Properties" %}}

| 프로퍼티                                         | 타입    | 기본값 | 설명                                                                                                                                             |
| ------------------------------------------------ | ------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `experimental-log-attributes`                    | Boolean | false  | 실험적 로그 속성 `thread.name` 및 `thread.id`의 캡처를 활성화한다.                                                                               |
| `experimental.capture-code-attributes`           | Boolean | false  | [소스 코드 속성][source code attributes]의 캡처를 활성화한다. 로깅 지점에서 소스 코드 속성을 캡처하면 성능 오버헤드가 발생할 수 있음에 유의한다. |
| `experimental.capture-marker-attribute`          | Boolean | false  | Logback 마커를 속성으로 캡처하는 기능을 활성화한다.                                                                                              |
| `experimental.capture-key-value-pair-attributes` | Boolean | false  | Logback 키-값 쌍을 속성으로 캡처하는 기능을 활성화한다.                                                                                          |
| `experimental.capture-logger-context-attributes` | Boolean | false  | Logback 로거 컨텍스트 프로퍼티를 속성으로 캡처하는 기능을 활성화한다.                                                                            |
| `experimental.capture-mdc-attributes`            | String  |        | 캡처할 MDC 속성의 쉼표로 구분된 목록이다. 모든 속성을 캡처하려면 와일드카드 문자 `*`를 사용한다.                                                 |

```yaml
otel:
  instrumentation:
    logback-appender:
      experimental-log-attributes: false
      experimental:
        capture-code-attributes: false
        capture-marker-attribute: false
        capture-key-value-pair-attributes: false
        capture-logger-context-attributes: false
        capture-mdc-attributes: '*'
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

| 프로퍼티                                        | 타입    | 기본값 | 설명                                                                                                                                             |
| ----------------------------------------------- | ------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `experimental_log_attributes/development`       | Boolean | false  | 실험적 로그 속성 `thread.name` 및 `thread.id`의 캡처를 활성화한다.                                                                               |
| `capture_code_attributes/development`           | Boolean | false  | [소스 코드 속성][source code attributes]의 캡처를 활성화한다. 로깅 지점에서 소스 코드 속성을 캡처하면 성능 오버헤드가 발생할 수 있음에 유의한다. |
| `capture_marker_attribute/development`          | Boolean | false  | Logback 마커를 속성으로 캡처하는 기능을 활성화한다.                                                                                              |
| `capture_key_value_pair_attributes/development` | Boolean | false  | Logback 키-값 쌍을 속성으로 캡처하는 기능을 활성화한다.                                                                                          |
| `capture_logger_context_attributes/development` | Boolean | false  | Logback 로거 컨텍스트 프로퍼티를 속성으로 캡처하는 기능을 활성화한다.                                                                            |
| `capture_mdc_attributes/development`            | String  |        | 캡처할 MDC 속성의 쉼표로 구분된 목록이다. 모든 속성을 캡처하려면 와일드카드 문자 `*`를 사용한다.                                                 |

```yaml
otel:
  instrumentation/development:
    java:
      logback_appender:
        experimental_log_attributes/development: false
        capture_code_attributes/development: false
        capture_marker_attribute/development: false
        capture_key_value_pair_attributes/development: false
        capture_logger_context_attributes/development: false
        capture_mdc_attributes/development: '*'
```

{{% /tab %}} {{< /tabpane >}}

[source code attributes]:
  /docs/specs/semconv/general/attributes/#source-code-attributes

또는 `logback.xml`이나 `logback-spring.xml` 파일에 오픈텔레메트리(OpenTelemetry)
Logback 어펜더(appender)를 추가해 이러한 기능을 활성화할 수도 있다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>
    <appender name="OpenTelemetry"
        class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender">
        <captureExperimentalAttributes>false</captureExperimentalAttributes>
        <captureCodeAttributes>true</captureCodeAttributes>
        <captureMarkerAttribute>true</captureMarkerAttribute>
        <captureKeyValuePairAttributes>true</captureKeyValuePairAttributes>
        <captureLoggerContext>true</captureLoggerContext>
        <captureMdcAttributes>*</captureMdcAttributes>
    </appender>
    <root level="INFO">
        <appender-ref ref="console"/>
        <appender-ref ref="OpenTelemetry"/>
    </root>
</configuration>
```

## Spring Web 자동 구성 {#spring-web-autoconfiguration}

[opentelemetry-spring-web-3.1](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-web/spring-web-3.1/library)에
정의된 `RestTemplate` 트레이스 인터셉터에 대한 자동 구성을 제공한다. 이 자동
구성은 `RestTemplate` 빈 후처리기(bean post processor)를 적용하여 Spring
`RestTemplate` 빈을 사용해 전송되는 모든 요청을 계측한다. 이 기능은 spring web
버전 3.1 이상에서 지원된다. 오픈텔레메트리 `RestTemplate` 인터셉터에 대해 더
자세히 알아보려면
[opentelemetry-spring-web-3.1](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-web/spring-web-3.1/library)을
참고한다.

`RestTemplate`을 생성하는 다음 방식들이 지원된다:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/RestTemplateConfig.java"?>
```java
package otel;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

  @Bean
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }
}
```

<?code-excerpt "src/main/java/otel/RestTemplateController.java"?>
```java
package otel;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
public class RestTemplateController {

  private final RestTemplate restTemplate;

  public RestTemplateController(RestTemplateBuilder restTemplateBuilder) {
    restTemplate = restTemplateBuilder.rootUri("http://localhost:8080").build();
  }
}
```
<!-- prettier-ignore-end -->

`RestClient`를 생성하는 다음 방식들이 지원된다:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/RestClientConfig.java"?>
```java
package otel;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

@Configuration
public class RestClientConfig {

  @Bean
  public RestClient restClient() {
    return RestClient.create();
  }
}
```

<?code-excerpt "src/main/java/otel/RestClientController.java"?>
```java
package otel;

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestClient;

@RestController
public class RestClientController {

  private final RestClient restClient;

  public RestClientController(RestClient.Builder restClientBuilder) {
    restClient = restClientBuilder.baseUrl("http://localhost:8080").build();
  }
}
```
<!-- prettier-ignore-end -->

Java 에이전트에서 가능한 것처럼, 다음 엔티티(entity)의 캡처를 구성할 수 있다:

- [HTTP 요청 및 응답 헤더](/docs/zero-code/java/agent/instrumentation/http/#capturing-http-request-and-response-headers)
- [알려진 HTTP 메서드](/docs/zero-code/java/agent/instrumentation/http/#configuring-known-http-methods)
- [실험적 HTTP 텔레메트리](/docs/zero-code/java/agent/instrumentation/http/#enabling-experimental-http-telemetry)

## Spring Web MVC 자동 구성 {#spring-web-mvc-autoconfiguration}

이 기능은 애플리케이션 컨텍스트에
[텔레메트리를 생성하는 서블릿 `Filter`](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/spring/spring-webmvc/spring-webmvc-5.3/library/src/main/java/io/opentelemetry/instrumentation/spring/webmvc/v5_3/WebMvcTelemetryProducingFilter.java)
빈을 추가하여 Spring WebMVC 컨트롤러에 대한 계측을 자동으로 구성한다. 이 필터는
요청 실행을 서버 스팬으로 감싸며, HTTP 요청에 수신된 추적 컨텍스트가 있으면 이를
전파한다. 오픈텔레메트리 Spring WebMVC 계측에 대해 더 자세히 알아보려면
[opentelemetry-spring-webmvc-5.3 계측 라이브러리](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-webmvc/spring-webmvc-5.3/library)를
참고한다.

Java 에이전트에서 가능한 것처럼, 다음 엔티티의 캡처를 구성할 수 있다:

- [HTTP 요청 및 응답 헤더](/docs/zero-code/java/agent/instrumentation/http/#capturing-http-request-and-response-headers)
- [알려진 HTTP 메서드](/docs/zero-code/java/agent/instrumentation/http/#configuring-known-http-methods)
- [실험적 HTTP 텔레메트리](/docs/zero-code/java/agent/instrumentation/http/#enabling-experimental-http-telemetry)

## Spring WebFlux 자동 구성 {#spring-webflux-autoconfiguration}

[opentelemetry-spring-webflux-5.3](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-webflux/spring-webflux-5.3/library)에
정의된 오픈텔레메트리 WebClient ExchangeFilter에 대한 자동 구성을 제공한다. 이
자동 구성은 빈 후처리기를 적용하여 Spring의 WebClient 및 WebClient Builder 빈을
사용해 전송되는 모든 아웃고잉 HTTP 요청을 계측한다. 이 기능은 spring webflux
버전 5.0 이상에서 지원된다. 자세한 내용은
[opentelemetry-spring-webflux-5.3](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-webflux/spring-webflux-5.3/library)을
참고한다.

`WebClient`를 생성하는 다음 방식들이 지원된다:

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/WebClientConfig.java"?>
```java
package otel;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

  @Bean
  public WebClient webClient() {
    return WebClient.create();
  }
}
```

<?code-excerpt "src/main/java/otel/WebClientController.java"?>
```java
package otel;

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;

@RestController
public class WebClientController {

  private final WebClient webClient;

  public WebClientController(WebClient.Builder webClientBuilder) {
    webClient = webClientBuilder.baseUrl("http://localhost:8080").build();
  }
}
```
<!-- prettier-ignore-end -->

## Kafka 계측 {#kafka-instrumentation}

Kafka 클라이언트 계측에 대한 자동 구성을 제공한다.

{{< tabpane text=true >}} {{% tab "Properties" %}}

Kafka에 대한 실험적 스팬 속성의 캡처를 활성화한다:

```yaml
otel:
  instrumentation:
    kafka:
      experimental-span-attributes: false # default: false
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

Kafka에 대한 실험적 스팬 속성의 캡처를 활성화한다:

```yaml
otel:
  instrumentation/development:
    java:
      kafka:
        experimental_span_attributes/development: false
```

{{% /tab %}} {{< /tabpane >}}

## Micrometer 계측 {#micrometer-instrumentation}

Micrometer에서 오픈텔레메트리로의 브리지에 대한 자동 구성을 제공한다.

## MongoDB 계측 {#mongodb-instrumentation}

MongoDB 클라이언트 계측에 대한 자동 구성을 제공한다.

{{< tabpane text=true >}} {{% tab "Properties" %}}

MongoDB에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation:
    mongo:
      statement-sanitizer:
        enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

MongoDB에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation/development:
    java:
      mongo:
        statement_sanitizer:
          enabled: true
```

{{% /tab %}} {{< /tabpane >}}

## R2DBC 계측 {#r2dbc-instrumentation}

오픈텔레메트리 R2DBC 계측에 대한 자동 구성을 제공한다.

{{< tabpane text=true >}} {{% tab "Properties" %}}

R2DBC에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation:
    r2dbc:
      statement-sanitizer:
        enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

R2DBC에 대해 DB 문(statement) 정제를 활성화한다:

```yaml
otel:
  instrumentation/development:
    java:
      r2dbc:
        statement_sanitizer:
          enabled: true
```

{{% /tab %}} {{< /tabpane >}}
