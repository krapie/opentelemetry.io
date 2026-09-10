---
title: SDK 구성
weight: 30
cSpell:ignore: distro
default_lang_commit: 2d89b60b2e09d42ba96757b0afdbc31f54a2b0e7
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/spring-starter"?>

이 spring 스타터는
[구성 메타데이터](https://docs.spring.io/spring-boot/docs/current/reference/html/configuration-metadata.html)를
지원하므로, IDE에서 사용 가능한 모든 프로퍼티를 확인하고 자동 완성할 수 있다.

## 일반 구성 {#general-configuration}

오픈텔레메트리(OpenTelemetry) 스타터는 모든
[SDK 자동 구성](/docs/zero-code/java/agent/configuration/#sdk-configuration)을
지원한다(2.2.0부터).

`application.properties`나 `application.yaml` 파일의 프로퍼티로, 또는 환경
변수로 구성을 업데이트할 수 있다.

{{< tabpane text=true >}} {{% tab "Properties" %}}

`application.yaml` 예제:

```yaml
otel:
  propagators:
    - tracecontext
    - b3
  resource:
    attributes:
      deployment.environment: dev
      service:
        name: cart
        namespace: shop
```

환경 변수 예제:

```shell
export OTEL_PROPAGATORS="tracecontext,b3"
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=dev,service.name=cart,service.namespace=shop"
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

SDK 수준 설정(리소스, 전파자, 익스포터)은 표준
[선언적 구성 스키마](/docs/languages/sdk-configuration/declarative-configuration/)를
`application.yaml`에서 직접 사용한다. 시스템 프로퍼티와 환경 변수는 값을
오버라이드하는 데 여전히 사용할 수 있다.
[환경 변수 오버라이드](../declarative-configuration/#environment-variable-overrides)를
참고한다.

```yaml
otel:
  file_format: '1.0'

  resource:
    attributes:
      - name: deployment.environment
        value: dev
      - name: service.name
        value: cart
      - name: service.namespace
        value: shop

  propagator:
    composite:
      - tracecontext:
      - b3:
```

{{% /tab %}} {{< /tabpane >}}

## 리소스 속성 오버라이드 {#overriding-resource-attributes}

Spring Boot에서 늘 그렇듯이, `application.properties` 및 `application.yaml`
파일의 프로퍼티를 환경 변수로 오버라이드할 수 있다.

예를 들어, 표준 `OTEL_RESOURCE_ATTRIBUTES` 환경 변수를 설정하면
(`service.name`이나 `service.namespace`는 바꾸지 않고) `deployment.environment`
리소스 속성을 설정하거나 오버라이드할 수 있다:

```shell
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=prod"
```

또는 `OTEL_RESOURCE_ATTRIBUTES_DEPLOYMENT_ENVIRONMENT` 환경 변수를 사용해 단일
리소스 속성을 설정하거나 오버라이드할 수 있다:

```shell
export OTEL_RESOURCE_ATTRIBUTES_DEPLOYMENT_ENVIRONMENT="prod"
```

두 번째 옵션은
[SpEL](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html)
표현식을 지원한다.

`DEPLOYMENT_ENVIRONMENT`는 Spring Boot의
[완화된 바인딩(Relaxed Binding)](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)에
의해 `deployment.environment`로 변환됨에 유의한다.

## 오픈텔레메트리 스타터 비활성화 {#disable-the-opentelemetry-starter}

{{< tabpane text=true >}} {{% tab "Properties" %}}

예를 들어 테스트 목적으로 스타터를 비활성화하려면 `otel.sdk.disabled`를 `true`로
설정한다:

```yaml
otel:
  sdk:
    disabled: true
```

{{% /tab %}} {{% tab "Declarative Configuration" %}}

예를 들어 테스트 목적으로 스타터를 비활성화하려면 `otel.disabled`를 `true`로
설정한다.

참고: [선언적 구성](../declarative-configuration/)에서는 프로퍼티 이름이
`otel.sdk.disabled`가 아니라 `otel.disabled`이다.

```yaml
otel:
  file_format: '1.0'
  disabled: true
```

{{% /tab %}} {{< /tabpane >}}

## 프로그래밍 방식 구성 {#programmatic-configuration}

[프로그래밍 방식 구성](../programmatic-configuration/)을 참고한다.

## 리소스 프로바이더 {#resource-providers}

{{< tabpane text=true >}} {{% tab "Properties" %}}

오픈텔레메트리 스타터는 Java 에이전트와 동일한 리소스 프로바이더를 포함한다:

- [공통 리소스 프로바이더](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/resources/library)
- [기본적으로 비활성화된 리소스 프로바이더](/docs/zero-code/java/agent/configuration/#enable-resource-providers-that-are-disabled-by-default)

추가로, 오픈텔레메트리 스타터는 다음 Spring Boot 전용 리소스 프로바이더를
포함한다:

### Distribution 리소스 프로바이더 {#distribution-resource-provider}

FQN:
`io.opentelemetry.instrumentation.spring.autoconfigure.resources.DistroVersionResourceProvider`

| 속성                       | 값                                  |
| -------------------------- | ----------------------------------- |
| `telemetry.distro.name`    | `opentelemetry-spring-boot-starter` |
| `telemetry.distro.version` | 스타터의 버전                       |

### Spring 리소스 프로바이더 {#spring-resource-provider}

FQN:
`io.opentelemetry.instrumentation.spring.autoconfigure.resources.SpringResourceProvider`

| 속성              | 값                                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| `service.name`    | `spring.application.name` 또는 `build-info.properties`의 `build.name` ([서비스 이름](#service-name) 참고) |
| `service.version` | `build-info.properties`의 `build.version`                                                                 |

{{% /tab %}} {{% tab "Declarative Configuration" %}}

[선언적 구성](../declarative-configuration/)에서는 리소스 프로바이더를
`resource.detection/development.detectors` 아래에 디텍터(detector)로 명시적으로
구성한다. 나열된 디텍터만 활성화되며, SPI를 통해 자동으로 발견되는 것은 없다.

```yaml
otel:
  resource:
    detection/development:
      detectors:
        - container: # container.id
        - host: # host.name, host.arch
        - host_id: # host.id
        - os: # os.type, os.description
        - process: # process.pid, process.executable.path, process.command_line
        - process_runtime: # process.runtime.name/version/description
        - service: # service.name, service.instance.id
        - spring: # service.name (from spring.application.name), service.version (from build-info)
```

`telemetry.distro.name` 및 `telemetry.distro.version` 속성은 문제 해결 목적으로
스타터가 항상 자동으로 추가한다.

{{% /tab %}} {{< /tabpane >}}

## 서비스 이름 {#service-name}

이러한 리소스 프로바이더를 사용하면, 서비스 이름은 오픈텔레메트리
[명세(specification)](/docs/languages/sdk-configuration/general/#otel_service_name)에
따라 다음 우선순위 규칙으로 결정된다:

{{< tabpane text=true >}} {{% tab "Properties" %}}

1. `otel.service.name` spring 프로퍼티 또는 `OTEL_SERVICE_NAME` 환경 변수(가장
   높은 우선순위)
2. `otel.resource.attributes` 시스템/spring 프로퍼티의 `service.name` 또는
   `OTEL_RESOURCE_ATTRIBUTES` 환경 변수
3. `spring.application.name` spring 프로퍼티
4. `build-info.properties`
5. META-INF/MANIFEST.MF의 `Implementation-Title`
6. 기본값은 `unknown_service:java`(가장 낮은 우선순위)

{{% /tab %}} {{% tab "Declarative Configuration" %}}

서비스 이름은 포함하는 리소스 디텍터에 따라
달라진다([리소스 프로바이더](#resource-providers) 참고):

1. `otel.resource.attributes`의 `service.name`(가장 높은 우선순위):

   ```yaml
   otel:
     resource:
       attributes:
         - name: service.name
           value: my-spring-app
   ```

2. `service` 디텍터 — 포함되면 `OTEL_SERVICE_NAME`에서 자동으로 감지한다:

   ```yaml
   otel:
     resource:
       detection/development:
         detectors:
           - service:
   ```

3. `spring` 디텍터 — 포함되면 `spring.application.name`과
   `build-info.properties`에서 감지한다:

   ```yaml
   otel:
     resource:
       detection/development:
         detectors:
           - spring:
   ```

4. 기본값은 `unknown_service:java`(가장 낮은 우선순위)

{{% /tab %}} {{< /tabpane >}}

`build-info.properties` 파일을 생성하려면 pom.xml 파일에 다음 스니펫을 사용한다:

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<build>
    <finalName>${project.artifactId}</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
springBoot {
  buildInfo {
  }
}
```

{{% /tab %}} {{< /tabpane>}}
