---
title: Quarkus 계측
linkTitle: Quarkus
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

[Quarkus](https://quarkus.io/)는 소프트웨어 개발자가 JVM과 Quarkus 네이티브
이미지 애플리케이션 모두에서 효율적인 클라우드 네이티브 애플리케이션을
구축하도록 돕기 위해 설계된 오픈 소스 프레임워크이다.

Quarkus는 익스텐션(extension)을 사용해 다양한 라이브러리에 대한 최적화된 지원을
제공한다.
[Quarkus OpenTelemetry 익스텐션](https://quarkus.io/guides/opentelemetry)은
다음을 제공한다:

- 기본 제공 계측
- 오픈텔레메트리(OpenTelemetry) SDK 자동 구성으로,
  [오픈텔레메트리 SDK](/docs/languages/java/configuration/)에 정의된 거의 모든
  시스템 프로퍼티를 지원한다
- [Vert.x](https://vertx.io/) 기반 OTLP 익스포터
- 오픈텔레메트리 Java 에이전트가 지원하지 않는 네이티브 이미지
  애플리케이션에서도 동일한 계측을 사용할 수 있다

> [!NOTE]
>
> Quarkus OpenTelemetry 계측은 Quarkus에서 유지 관리하고 지원한다. 자세한 내용은
> [Quarkus 커뮤니티 지원](https://quarkus.io/support/)을 참고한다.

네이티브 이미지 애플리케이션을 실행하지 않는다면, Quarkus는
[오픈텔레메트리 Java 에이전트](../agent/)로도 계측할 수 있다.

## 시작하기 {#getting-started}

Quarkus 애플리케이션에서 오픈텔레메트리를 활성화하려면, 프로젝트에
`quarkus-opentelemetry` 익스텐션 의존성을 추가한다.

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
implementation("io.quarkus:quarkus-opentelemetry")
```

{{% /tab %}} {{< /tabpane>}}

기본적으로 **트레이싱** 시그널만 활성화되어 있다. **메트릭**과 **로그**를
활성화하려면 `application.properties` 파일에 다음 구성을 추가한다:

```properties
quarkus.otel.metrics.enabled=true
quarkus.otel.logs.enabled=true
```

오픈텔레메트리 로깅은 Quarkus 3.16.0 이상에서 지원된다.

이 옵션들과 그 밖의 구성 옵션에 대한 자세한 내용은
[오픈텔레메트리 구성 레퍼런스](https://quarkus.io/guides/opentelemetry#configuration-reference)를
참고한다.

## 더 알아보기 {#learn-more}

- [오픈텔레메트리 사용하기](https://quarkus.io/guides/opentelemetry), 모든
  [구성](https://quarkus.io/guides/opentelemetry#configuration-reference) 옵션을
  다루는 일반 레퍼런스
- 시그널별 가이드
  - [트레이싱](https://quarkus.io/guides/opentelemetry-tracing)
  - [메트릭](https://quarkus.io/guides/opentelemetry-metrics)
  - [로그](https://quarkus.io/guides/opentelemetry-logging)
