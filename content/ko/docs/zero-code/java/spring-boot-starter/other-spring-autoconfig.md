---
title: 기타 Spring 자동 구성
weight: 70
cSpell:ignore: autoconfigurations
default_lang_commit: f304b22c356b4ad4046b8ce5550ddf503a510d69
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/spring-starter"?>

오픈텔레메트리(OpenTelemetry) Spring 스타터 대신 오픈텔레메트리 Zipkin 스타터를
사용할 수 있다.

## Zipkin 스타터 {#zipkin-starter}

OpenTelemetry Zipkin Exporter Starter는 분산 트레이싱(distributed tracing)을
설정하는 데 필요한 `opentelemetry-api`, `opentelemetry-sdk`,
`opentelemetry-extension-annotations`, `opentelemetry-logging-exporter`,
`opentelemetry-spring-boot-autoconfigurations` 및 spring 프레임워크 스타터를
포함하는 스타터 패키지이다. 또한
[opentelemetry-exporters-zipkin](https://github.com/open-telemetry/opentelemetry-java/tree/v1.64.0/exporters/zipkin)
아티팩트와 그에 대응하는 익스포터 자동 구성을 제공한다.

런타임에 클래스패스에 익스포터가 존재하고 spring 애플리케이션 컨텍스트에 해당
익스포터의 spring 빈(bean)이 없으면, 익스포터 빈이 초기화되어 활성 트레이서
프로바이더의 단순 스팬 프로세서(simple span processor)에 추가된다. 자세한 내용은
[구현 (OpenTelemetryAutoConfiguration.java)](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/spring/spring-boot-autoconfigure/src/main/java/io/opentelemetry/instrumentation/spring/autoconfigure/OpenTelemetryAutoConfiguration.java)을
참고한다.

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<dependencies>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
    <version>{{% param vers.otel %}}</version>
  </dependency>
</dependencies>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
dependencies {
  implementation("io.opentelemetry:opentelemetry-exporter-zipkin:{{% param vers.otel %}}")
}
```

{{% /tab %}} {{< /tabpane>}}

### 구성 {#configurations}

{{< tabpane text=true >}} {{% tab "Properties" %}}

Zipkin 익스포터를 활성화한다(클래스패스에 `ZipkinSpanExporter`가 필요함):

```yaml
otel:
  exporter:
    zipkin:
      enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

[선언적 구성](../declarative-configuration/)에서는 Zipkin 익스포터를 표준
[선언적 구성 스키마](/docs/languages/sdk-configuration/declarative-configuration/)의
일부로 `tracer_provider.processors` 아래에 구성한다:

```yaml
otel:
  tracer_provider:
    processors:
      - batch:
          exporter:
            zipkin:
              endpoint: http://localhost:9411/api/v2/spans
```

{{% /tab %}} {{< /tabpane >}}
