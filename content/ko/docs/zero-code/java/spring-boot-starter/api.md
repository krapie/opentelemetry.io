---
title: API로 계측 확장하기
linkTitle: API로 확장하기
description:
  오픈텔레메트리(OpenTelemetry) API를 Spring Boot 스타터와 함께 사용하여
  자동으로 생성된 텔레메트리를 사용자 정의 스팬과 메트릭으로 확장한다
weight: 21
default_lang_commit: 4b5381a2e9f129651ab8658357ab846bd4c965f2
---

## 소개 {#introduction}

기본 제공 계측(out-of-the-box instrumentation) 외에도,
오픈텔레메트리(OpenTelemetry) API를 사용해 사용자 정의 수동 계측으로 Spring
스타터를 확장할 수 있다. 이를 통해 코드를 크게 변경하지 않고도 직접 작성한
코드에 대한 [스팬](/docs/concepts/signals/traces/#spans)과
[메트릭](/docs/concepts/signals/metrics)을 생성할 수 있다.

필요한 의존성은 Spring Boot 스타터에 이미 포함되어 있다.

## OpenTelemetry {#opentelemetry}

Spring Boot 스타터는 `OpenTelemetry`를 Spring 빈(bean)으로 사용할 수 있는 특별한
경우이다. `OpenTelemetry`를 Spring 컴포넌트에 주입하기만 하면 된다.

## Span {#span}

> [!NOTE]
>
> 가장 일반적인 사용 사례에서는 수동 계측 대신 `@WithSpan`
> 어노테이션(annotation)을 사용한다. 자세한 내용은
> [어노테이션](../annotations)을 참고한다.

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;

@Controller
public class MyController {
  private final Tracer tracer;

  public MyController(OpenTelemetry openTelemetry) {
    this.tracer = openTelemetry.getTracer("application");
  }
}
```

[Span](/docs/languages/java/api/#span) 섹션에서 설명하는 대로 `Tracer`를 사용해
스팬을 생성한다.

전체 예제는 [예제 저장소][example repository]에서 확인할 수 있다.

## Meter {#meter}

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.metrics.Meter;

@Controller
public class MyController {
  private final Meter meter;

  public MyController(OpenTelemetry openTelemetry) {
    this.meter = openTelemetry.getMeter("application");
  }
}
```

[Meter](/docs/languages/java/api/#meter) 섹션에서 설명하는 대로 `Meter`를 사용해
카운터(counter), 게이지(gauge), 히스토그램(histogram)을 생성한다.

전체 예제는 [예제 저장소][example repository]에서 확인할 수 있다.

[example repository]:
  https://github.com/open-telemetry/opentelemetry-java-examples/tree/main/spring-native
