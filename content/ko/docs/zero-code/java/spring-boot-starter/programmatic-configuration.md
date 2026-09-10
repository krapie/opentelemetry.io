---
title: 프로그래밍 방식 구성
weight: 35
vers:
  contrib: 1.54.0
cSpell:ignore: customizer fileconfig
default_lang_commit: d30d20ea078abf2b4a9aa270aa01042efa91dc99
---

<?code-excerpt path-base="examples/java/spring-starter"?>

프로그래밍 방식 구성에는 `AutoConfigurationCustomizerProvider`를 사용할 수 있다.
프로그래밍 방식 구성은 프로퍼티로 구성할 수 없는 고급 사용 사례에 권장된다.

> [!WARNING]
>
> `AutoConfigurationCustomizerProvider`는
> [선언적 구성](../declarative-configuration/)에서는 동작하지 않는다. 선언적
> 구성에서는 대신 `DeclarativeConfigurationCustomizerProvider`를 사용한다.
> 자세한 내용과 예제는
> [에이전트 확장 API 섹션](/docs/zero-code/java/agent/declarative-configuration/)을
> 참고한다.

## 트레이싱에서 actuator 엔드포인트 제외 {#exclude-actuator-endpoints-from-tracing}

예를 들어, 헬스 체크 엔드포인트를 트레이싱에서 제외하도록 샘플러(sampler)를
커스터마이징할 수 있다:

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<dependencies>
  <dependency>
    <groupId>io.opentelemetry.contrib</groupId>
    <artifactId>opentelemetry-samplers</artifactId>
    <version>{{% param vers.contrib %}}-alpha</version>
  </dependency>
</dependencies>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
dependencies {
  implementation("io.opentelemetry.contrib:opentelemetry-samplers:{{% param vers.contrib %}}-alpha")
}
```

{{% /tab %}} {{< /tabpane>}}

<?code-excerpt "src/main/java/otel/FilterPaths.java"?>

```java
package otel;

import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.contrib.sampler.RuleBasedRoutingSampler;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;
import io.opentelemetry.semconv.UrlAttributes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FilterPaths {

  @Bean
  public AutoConfigurationCustomizerProvider otelCustomizer() {
    return p ->
        p.addSamplerCustomizer(
            (fallback, config) ->
                RuleBasedRoutingSampler.builder(SpanKind.SERVER, fallback)
                    .drop(UrlAttributes.URL_PATH, "^/actuator")
                    .build());
  }
}
```

## 프로그래밍 방식으로 익스포터 구성 {#configure-the-exporter-programmatically}

OTLP 익스포터를 프로그래밍 방식으로 구성할 수도 있다. 이 구성은 기본 OTLP
익스포터를 대체하고 요청에 사용자 정의 헤더를 추가한다.

<?code-excerpt "src/main/java/otel/CustomAuth.java"?>

```java
package otel;

import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;
import java.util.Collections;
import java.util.Map;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CustomAuth {
  @Bean
  public AutoConfigurationCustomizerProvider otelCustomizer() {
    return p ->
        p.addSpanExporterCustomizer(
            (exporter, config) -> {
              if (exporter instanceof OtlpHttpSpanExporter) {
                return ((OtlpHttpSpanExporter) exporter)
                    .toBuilder().setHeaders(this::headers).build();
              }
              return exporter;
            });
  }

  private Map<String, String> headers() {
    return Collections.singletonMap("Authorization", "Bearer " + refreshToken());
  }

  private String refreshToken() {
    // e.g. read the token from a kubernetes secret
    return "token";
  }
}
```

## 프로그래밍 방식으로 계측 구성 읽기 {#read-instrumentation-configuration-programmatically}

> [!NOTE]
>
> 오픈텔레메트리(OpenTelemetry) Spring Boot 스타터 버전 2.30.0 이상이 필요하다.

계측 모듈은 `application.properties` / `application.yaml`로 구성했든
[선언적 구성](../declarative-configuration/)으로 구성했든, `ConfigProvider`
빈(bean)을 통해 구성을 읽는다. 직접 작성한 코드에서 계측 구성 값을 읽어야 한다면
`ConfigProvider` 빈을 바로 오토와이어링(autowire)한다:

선언적 구성이 활성화되면 `otelProperties`(`ConfigProperties`) 빈은 호환성
브리지로만 제공된다. 이는 지원 중단되었으며 3.0에서 제거될 예정이다. 대신
`ConfigProvider`를 사용한다.

<?code-excerpt path-base="content-modules/opentelemetry-java-examples/spring-declarative-configuration"?>

<?code-excerpt "src/main/java/io/opentelemetry/examples/fileconfig/ReadInstrumentationConfig.java" from="package"?>

```java
package io.opentelemetry.examples.fileconfig;

import io.opentelemetry.api.incubator.config.ConfigProvider;
import io.opentelemetry.api.incubator.config.DeclarativeConfigProperties;
import org.springframework.stereotype.Component;

/** Example of reading instrumentation configuration from application code. */
@Component
public class ReadInstrumentationConfig {

  private final ConfigProvider configProvider;

  public ReadInstrumentationConfig(ConfigProvider configProvider) {
    this.configProvider = configProvider;
  }

  public boolean isDbQuerySanitizationEnabled() {
    DeclarativeConfigProperties dbConfig =
        configProvider
            .getInstrumentationConfig()
            .get("java")
            .get("common")
            .get("db")
            .get("query_sanitization");
    return dbConfig.getBoolean("enabled", true);
  }
}
```

`getInstrumentationConfig()` 아래의 키는 값을 `otel.instrumentation.*`
프로퍼티로 설정했든 선언적 YAML로 설정했든,
[선언적 구성](../declarative-configuration/#instrumentation-configuration)에서
사용하는 것과 동일한 `instrumentation/development.java.*` 구조를 따른다.
프로퍼티 이름이 이 구조로 어떻게 변환되는지는
[매핑 표](../declarative-configuration/#instrumentation-configuration)를
참고한다.
