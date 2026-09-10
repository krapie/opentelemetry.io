---
title: API로 계측 확장하기
linkTitle: API로 확장하기
description:
  오픈텔레메트리(OpenTelemetry) API를 Java 에이전트와 함께 사용하여 자동으로
  생성된 텔레메트리를 커스텀 스팬 및 메트릭으로 확장한다
weight: 21
default_lang_commit: 4b5381a2e9f129651ab8658357ab846bd4c965f2
---

## 소개 {#introduction}

기본 제공 계측 외에도, 오픈텔레메트리 API를 사용한 커스텀 수동 계측으로 Java
에이전트를 확장할 수 있다. 이를 통해 코드를 크게 변경하지 않고도 직접 작성한
코드에 [스팬](/docs/concepts/signals/traces/#spans)과
[메트릭](/docs/concepts/signals/metrics)을 생성할 수 있다.

## 의존성 {#dependencies}

`opentelemetry-api` 라이브러리에 대한 의존성을 추가한다.

### Maven {#maven}

```xml
<dependencies>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>{{% param vers.otel %}}</version>
  </dependency>
</dependencies>
```

### Gradle {#gradle}

```groovy
dependencies {
    implementation('io.opentelemetry:opentelemetry-api:{{% param vers.otel %}}')
}
```

## OpenTelemetry {#opentelemetry}

Java 에이전트는 `GlobalOpenTelemetry`가 에이전트에 의해 설정되는 특별한
경우이다. `GlobalOpenTelemetry.getOrNoop()`을 호출하기만 하면 `OpenTelemetry`
인스턴스에 접근할 수 있다.

## Span {#span}

> [!NOTE]
>
> 가장 흔한 사용 사례에서는 수동 계측 대신 `@WithSpan` 어노테이션을 사용한다.
> 자세한 내용은 [어노테이션](../annotations)을 참고한다.

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = GlobalOpenTelemetry.getTracer("application");
```

[Span](/docs/languages/java/api/#span) 섹션에서 설명하는 대로 `Tracer`를 사용해
스팬을 생성한다.

전체 예제는 [예제 저장소][example repository]에서 확인할 수 있다.

## Meter {#meter}

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.metrics.Meter;

Meter meter = GlobalOpenTelemetry.getMeter("application");
```

[Meter](/docs/languages/java/api/#meter) 섹션에서 설명하는 대로 `Meter`를 사용해
카운터, 게이지 또는 히스토그램을 생성한다.

전체 예제는 [예제 저장소][example repository]에서 확인할 수 있다.

[example repository]:
  https://github.com/open-telemetry/opentelemetry-java-examples/tree/main/javaagent
