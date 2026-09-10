---
title: 시작하기
weight: 20
cSpell:ignore: springboot
default_lang_commit: f304b22c356b4ad4046b8ce5550ddf503a510d69
---

> [!NOTE]
>
> Spring Boot 애플리케이션을 계측하는 데는 [Java 에이전트](../../agent)를 사용할
> 수도 있다. 장단점은 [Java 제로 코드 계측(Zero-code instrumentation)](..)을
> 참고한다.

## Compatibility {#compatibility}

오픈텔레메트리 Spring Boot 스타터는 Spring Boot 2.6+ 및 3.1+, 그리고 Spring Boot
Native 이미지 애플리케이션에서 동작한다.
[opentelemetry-java-examples/spring-native](https://github.com/open-telemetry/opentelemetry-java-examples/tree/main/spring-native)
저장소에는 오픈텔레메트리 Spring Boot 스타터로 계측한 Spring Boot Native 이미지
애플리케이션 예제가 있다.

## Dependency management {#dependency-management}

Bill of Material([BOM](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#bill-of-materials-bom-poms))은
(전이 의존성(dependency)을 포함해) 의존성 버전이 서로 정렬되도록 보장한다.

모든 오픈텔레메트리 의존성 간 버전을 맞추려면, 오픈텔레메트리 스타터를 사용할 때
`opentelemetry-instrumentation-bom` BOM을 반드시 임포트해야 한다.

> [!NOTE]
>
> Maven을 사용할 때는 프로젝트에서 다른 BOM보다 먼저 오픈텔레메트리 BOM을
> 임포트한다. 예를 들어 `spring-boot-dependencies` BOM을 임포트한다면, 오픈텔레메
> 트리 BOM 다음에 선언해야 한다.
>
> Gradle은 여러 BOM이 있을 때 의존성의
> [최신 버전](https://docs.gradle.org/current/userguide/dependency_constraints_conflicts.html#sub:resolving-version-conflicts)을
> 선택하므로, BOM의 순서는 중요하지 않다.

다음 예제는 Maven으로 오픈텔레메트리 BOM을 임포트하는 방법을 보여준다:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.opentelemetry.instrumentation</groupId>
            <artifactId>opentelemetry-instrumentation-bom</artifactId>
            <version>{{% param vers.instrumentation %}}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Gradle과 Spring Boot를 함께 사용할 때, BOM을 임포트하는 방법에는 두 가지가 있다.

Gradle의 네이티브 BOM 지원 기능을 사용해 `dependencies`를 추가할 수 있다:

```kotlin
import org.springframework.boot.gradle.plugin.SpringBootPlugin

plugins {
  id("java")
  id("org.springframework.boot") version "3.2.O"
}

dependencies {
  implementation(platform(SpringBootPlugin.BOM_COORDINATES))
  implementation(platform("io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:{{% param vers.instrumentation %}}"))
}
```

Gradle에서 사용할 수 있는 다른 방법은 `io.spring.dependency-management` 플러그인을
사용해 `dependencyManagement`에 BOM을 임포트하는 것이다:

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.O"
  id("io.spring.dependency-management") version "1.1.0"
}

dependencyManagement {
  imports {
    mavenBom("io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:{{% param vers.instrumentation %}}")
  }
}
```

> [!NOTE]
>
> Gradle로 설정하는 여러 방식을 섞어 쓰지 않도록 주의한다. 예를 들어
> `io.spring.dependency-management` 플러그인과 함께
> `implementation(platform("io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:{{% param vers.instrumentation %}}"))`을
> 사용하지 않는다.

### OpenTelemetry Starter dependency {#opentelemetry-starter-dependency}

오픈텔레메트리 스타터(starter)를 활성화하려면 아래 의존성을 추가한다.

오픈텔레메트리 스타터는 OpenTelemetry Spring Boot
[자동 구성(autoconfiguration)](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)을
사용한다.

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter")
```

{{% /tab %}} {{< /tabpane>}}
