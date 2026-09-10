---
title: 추가 계측
weight: 60
default_lang_commit: 2d89b60b2e09d42ba96757b0afdbc31f54a2b0e7
---

오픈텔레메트리(OpenTelemetry) Spring Boot 스타터는 추가 계측으로 보강할 수 있는
[기본 제공 계측](../out-of-the-box-instrumentation)을 제공한다.

## Log4j2 계측 {#log4j2-instrumentation}

`log4j2.xml` 파일에 오픈텔레메트리 어펜더(appender)를 추가해야 한다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" packages="io.opentelemetry.instrumentation.log4j.appender.v2_17">
    <Appenders>
        <OpenTelemetry name="OpenTelemetryAppender"/>
    </Appenders>
    <Loggers>
        <Root>
            <AppenderRef ref="OpenTelemetryAppender" level="All"/>
        </Root>
    </Loggers>
</Configuration>
```

오픈텔레메트리 어펜더에 대한 더 많은 구성 옵션은
[Log4j](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/log4j/log4j-appender-2.17/library/README.md)
계측 라이브러리에서 확인할 수 있다.

{{< tabpane text=true >}} {{% tab "Properties" %}}

`OpenTelemetry` 인스턴스로 Log4j 오픈텔레메트리 어펜더의 구성을 활성화한다:

```yaml
otel:
  instrumentation:
    log4j-appender:
      enabled: true # default: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

[선언적 구성](../declarative-configuration/)에서는 중앙 집중식 계측 목록을
사용해 Log4j를 활성화하거나 비활성화한다:

```yaml
otel:
  distribution:
    spring_starter:
      instrumentation:
        disabled:
          - log4j_appender
```

{{% /tab %}} {{< /tabpane >}}

## 계측 라이브러리 {#instrumentation-libraries}

[오픈텔레메트리 계측 라이브러리](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md#libraries--frameworks)를
사용해 다른 계측을 구성할 수 있다.
