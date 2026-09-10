---
title: 익스텐션
aliases: [/docs/instrumentation/java/extensions]
description:
  익스텐션(Extension)은 별도의 배포판을 만들지 않고도 에이전트에 기능을
  추가한다.
weight: 300
cSpell:ignore: Customizer Dotel myextension
default_lang_commit: d30d20ea078abf2b4a9aa270aa01042efa91dc99
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/extensions-minimal"?>

## 소개 {#introduction}

익스텐션(Extension)은 별도의 배포판(전체 에이전트의 커스텀 버전)을 만들지 않고도
오픈텔레메트리(OpenTelemetry) Java 에이전트에 새로운 기능과 능력을 추가한다.
익스텐션은 에이전트의 동작 방식을 커스터마이즈하는 플러그인이라고 생각하면 된다.

익스텐션으로 다음을 할 수 있다.

- 현재 지원되지 않는 라이브러리를 위한 새로운 계측 추가
- 기존 계측 동작 커스터마이즈
- 커스텀 SDK 컴포넌트(샘플러, 익스포터, 전파자) 구현
- 환경 변수나 선언적 구성으로 다루지 못하는 경우에 대해 프로그래밍 방식으로 구성
  커스터마이즈
- 텔레메트리 데이터 수집 및 처리 수정

## 빠른 시작 {#quick-start}

시작을 돕기 위해, 커스텀 스팬 프로세서를 추가하는 최소한의 익스텐션을 소개한다.

Gradle 프로젝트(build.gradle.kts)를 생성한다.

<!-- prettier-ignore-start -->
<?code-excerpt "build.gradle.kts"?>
```kotlin
plugins {
    id("java")
    id("com.gradleup.shadow")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(8))
    }
}

dependencies {
    // Use BOM to manage OpenTelemetry dependency versions
    compileOnly(platform("io.opentelemetry:opentelemetry-bom:1.64.0"))

    // OpenTelemetry SDK autoconfiguration SPI (provided by agent)
    compileOnly("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi")

    // OpenTelemetry SDK (needed for SpanProcessor and trace classes)
    compileOnly("io.opentelemetry:opentelemetry-sdk")

    // Annotation processor for automatic SPI registration
    compileOnly("com.google.auto.service:auto-service:1.1.1")
    annotationProcessor("com.google.auto.service:auto-service:1.1.1")

    // Add any external dependencies with 'implementation' scope
    // implementation("org.apache.commons:commons-lang3:3.19.0")
}

tasks.assemble {
    dependsOn(tasks.shadowJar)
}
```
<!-- prettier-ignore-end -->

`SpanProcessor` 구현을 생성한다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/MySpanProcessor.java" from="public"?>
```java
public class MySpanProcessor implements SpanProcessor {

  @Override
  public void onStart(Context parentContext, ReadWriteSpan span) {
    // Add custom attributes when span starts
    span.setAttribute("custom.processor", "active");
  }

  @Override
  public boolean isStartRequired() {
    return true;
  }

  @Override
  public void onEnd(ReadableSpan span) {
    // Process span when it ends (optional)
  }

  @Override
  public boolean isEndRequired() {
    return false;
  }

  @Override
  public CompletableResultCode shutdown() {
    return CompletableResultCode.ofSuccess();
  }
}
```
<!-- prettier-ignore-end -->

`AutoConfigurationCustomizerProvider` SPI를 사용하는 익스텐션 클래스를 생성한다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/MyExtensionProvider.java" from="@AutoService"?>
```java
@AutoService(AutoConfigurationCustomizerProvider.class)
public class MyExtensionProvider implements AutoConfigurationCustomizerProvider {

  @Override
  public void customize(AutoConfigurationCustomizer config) {
    config.addTracerProviderCustomizer(this::configureTracer);
  }

  private SdkTracerProviderBuilder configureTracer(
      SdkTracerProviderBuilder tracerProvider, ConfigProperties config) {
    return tracerProvider
        .setSpanLimits(SpanLimits.builder().setMaxNumberOfAttributes(1024).build())
        .addSpanProcessor(new MySpanProcessor());
  }
}
```
<!-- prettier-ignore-end -->

익스텐션을 빌드한다.

```bash
./gradlew shadowJar
```

익스텐션을 사용한다.

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=build/libs/my-extension-all.jar \
     -jar myapp.jar
```

## 익스텐션 사용하기 {#using-extensions}

Java 에이전트에서 익스텐션을 사용하는 방법에는 두 가지가 있다.

- **별도의 JAR 파일로 로드** - 개발과 테스트에 유연하다
- **에이전트에 임베드** - 프로덕션을 위한 단일 JAR 배포

| 방식            | 장점                                              | 단점                         | 적합한 경우    |
| --------------- | ------------------------------------------------- | ---------------------------- | -------------- |
| **런타임 로딩** | 익스텐션을 쉽게 교체할 수 있고 재빌드가 필요 없음 | 추가 명령줄 플래그 필요      | 개발, 테스트   |
| **임베딩**      | 단일 JAR, 더 단순한 배포, 로드를 잊을 수 없음     | 익스텐션 변경 시 재빌드 필요 | 프로덕션, 배포 |

### 런타임에 익스텐션 로드하기 {#loading-extensions-at-runtime}

익스텐션은 `otel.javaagent.extensions` 시스템 속성 또는
`OTEL_JAVAAGENT_EXTENSIONS` 환경 변수를 사용하여 런타임에 로드할 수 있다. 이
구성 옵션은 익스텐션 JAR 파일이나 익스텐션 JAR가 담긴 디렉터리의 경로를 쉼표로
구분한 목록으로 받는다.

#### 단일 익스텐션 {#single-extension}

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=/path/to/my-extension.jar \
     -jar myapp.jar
```

#### 여러 익스텐션 {#multiple-extensions}

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=/path/to/extension1.jar,/path/to/extension2.jar \
     -jar myapp.jar
```

#### 익스텐션 디렉터리 {#extension-directory}

여러 익스텐션 JAR가 담긴 디렉터리를 지정할 수 있으며, 그 디렉터리의 모든 JAR가
로드된다.

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=/path/to/extensions-directory \
     -jar myapp.jar
```

#### 경로 혼합 {#mixed-paths}

개별 JAR 파일과 디렉터리를 조합할 수 있다.

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=/path/to/extension1.jar,/opt/extensions,/tmp/custom.jar \
     -jar myapp.jar
```

#### 익스텐션 로딩 동작 방식 {#how-extension-loading-works}

런타임에 익스텐션을 로드하면 에이전트는 다음을 수행한다.

1. 익스텐션 JAR에 오픈텔레메트리 API를 패키징하지 않아도 익스텐션에서 사용할 수
   있게 한다
2. Java의
   [ServiceLoader](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ServiceLoader.html)
   메커니즘을 사용하여(예를 들어 코드의 `@AutoService` 어노테이션을 통해)
   익스텐션의 컴포넌트를 발견한다

### 에이전트에 익스텐션 임베드하기 {#embedding-extensions-in-the-agent}

또 다른 배포 옵션은 오픈텔레메트리 Java 에이전트와 익스텐션을 모두 포함하는 단일
JAR 파일을 만드는 것이다. 이 방식은 배포를 단순화하고(관리할 JAR 파일이 하나뿐),
`-Dotel.javaagent.extensions` 명령줄 옵션이 필요 없게 하여 실수로 익스텐션
로드를 잊어버리기 어렵게 만든다.

#### 동작 방식 {#how-it-works}

에이전트는 에이전트 JAR 파일 안의 특별한 `extensions/` 디렉터리에서 익스텐션을
자동으로 찾으므로, Gradle 빌드 태스크를 사용하여 다음을 할 수 있다.

1. 오픈텔레메트리 Java 에이전트 JAR 다운로드
2. 내용물 추출
3. `extensions/` 디렉터리에 익스텐션 JAR 추가
4. 전체를 단일 JAR로 다시 패키징

#### `extendedAgent` Gradle 태스크 {#the-extendedagent-gradle-task}

익스텐션 프로젝트의 `build.gradle.kts` 파일에 다음을 추가한다.

```kotlin
plugins {
    id("java")

    // Shadow plugin: Combines all your extension's code and dependencies into one JAR
    // This is required because extensions must be packaged as a single JAR file
    id("com.gradleup.shadow") version "9.2.2"
}

group = "com.example"
version = "1.0"

configurations {
    // Create a temporary configuration to download the agent JAR
    // Think of this as a "download slot" that's separate from your extension's dependencies
    create("otel")
}

dependencies {
    // Download the official OpenTelemetry Java agent into the 'otel' configuration
    "otel"("io.opentelemetry.javaagent:opentelemetry-javaagent:{{% param vers.instrumentation %}}")

    /*
      Interfaces and SPIs that we implement. We use `compileOnly` dependency because during
      runtime all necessary classes are provided by javaagent itself.
     */
    compileOnly("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi:{{% param vers.otel %}}")
    compileOnly("io.opentelemetry:opentelemetry-sdk:{{% param vers.otel %}}")
    compileOnly("io.opentelemetry:opentelemetry-api:{{% param vers.otel %}}")

    // Required for custom instrumentation
    compileOnly("io.opentelemetry.javaagent:opentelemetry-javaagent-extension-api:{{% param vers.instrumentation %}}-alpha")
    compileOnly("io.opentelemetry.instrumentation:opentelemetry-instrumentation-api-incubator:{{% param vers.instrumentation %}}-alpha")
    compileOnly("net.bytebuddy:byte-buddy:1.15.10")

    // Provides @AutoService annotation that makes registration of our SPI implementations much easier
    compileOnly("com.google.auto.service:auto-service:1.1.1")
    annotationProcessor("com.google.auto.service:auto-service:1.1.1")
}

// Task: Create an extended agent JAR (agent + your extension)
val extendedAgent by tasks.registering(Jar::class) {
    dependsOn(configurations["otel"])
    archiveFileName.set("opentelemetry-javaagent.jar")

    // Step 1: Unpack the official agent JAR
    from(zipTree(configurations["otel"].singleFile))

    // Step 2: Add your extension JAR to the "extensions/" directory
    from(tasks.shadowJar.get().archiveFile) {
        into("extensions")
    }

    // Step 3: Preserve the agent's startup configuration (MANIFEST.MF)
    doFirst {
        manifest.from(
            zipTree(configurations["otel"].singleFile).matching {
                include("META-INF/MANIFEST.MF")
            }.singleFile
        )
    }
}

tasks {
    // Make sure the shadow JAR is built during the normal build process
    assemble {
        dependsOn(shadowJar)
    }
}
```

완전한 예제는
[익스텐션 예제](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/examples/extension/build.gradle.kts)의
gradle 파일을 참조한다.

#### 확장된 에이전트 빌드 및 사용하기 {#building-and-using-the-extended-agent}

`build.gradle.kts`에 `extendedAgent` 태스크를 추가했다면:

```bash
# 1. Build your extension and create the extended agent
./gradlew extendedAgent

# 2. Find the output in build/libs/
ls build/libs/opentelemetry-javaagent.jar

# 3. Use it with your application (no -Dotel.javaagent.extensions needed)
java -javaagent:build/libs/opentelemetry-javaagent.jar -jar myapp.jar
```

#### 여러 익스텐션 임베드하기 {#embedding-multiple-extensions}

여러 익스텐션을 임베드하려면, 여러 익스텐션 JAR를 포함하도록 `extendedAgent`
태스크를 수정한다.

```kotlin
val extendedAgent by tasks.registering(Jar::class) {
  dependsOn(configurations["otel"])
  archiveFileName.set("opentelemetry-javaagent.jar")

  from(zipTree(configurations["otel"].singleFile))

  // Add multiple extensions
  from(tasks.shadowJar.get().archiveFile) {
    into("extensions")
  }
  from(file("../other-extension/build/libs/other-extension-all.jar")) {
    into("extensions")
  }

  doFirst {
    manifest.from(
      zipTree(configurations["otel"].singleFile).matching {
        include("META-INF/MANIFEST.MF")
      }.singleFile
    )
  }
}
```

## 익스텐션 작성하기 {#writing-extensions}

익스텐션을 만드는 일은 하나 이상의 서비스 프로바이더 인터페이스(Service Provider
Interface, SPI) 클래스를 구현하고, 이를 JAR 파일로 패키징한 뒤, 애플리케이션을
실행할 때 에이전트가 그 JAR를 가리키게 하는 것으로
이루어진다([익스텐션 사용하기](#using-extensions) 참고).

> [!TIP]
>
> 아래에서 설명하는 각 SPI를 다루는 완전하고 실행 가능한 레퍼런스는 Java 계측
> 저장소의
> [익스텐션 예제 프로젝트](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/examples/extension)를
> 참고한다.

### 프로젝트 설정 및 의존성 {#project-setup-and-dependencies}

익스텐션은 에이전트 및 애플리케이션과의 충돌을 피하기 위해 의존성을 신중하게
관리해야 한다. 에이전트가 클래스 로더 전반에서 익스텐션을 어떻게 격리하는지에
대한 배경 지식은
[Java 에이전트 구조](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/contributing/javaagent-structure.md)를
참고한다.

#### 에이전트가 제공하는 의존성(`compileOnly` 사용) {#dependencies-provided-by-agent-use-compileonly}

이러한 API는 런타임에 에이전트에서 사용할 수 있다.

```kotlin
compileOnly("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi")
compileOnly("io.opentelemetry.instrumentation:opentelemetry-instrumentation-api")
compileOnly("io.opentelemetry.instrumentation:opentelemetry-instrumentation-api-incubator")
compileOnly("io.opentelemetry.javaagent:opentelemetry-javaagent-extension-api")
```

#### 애플리케이션 클래스패스의 의존성(`compileOnly` 사용) {#dependencies-from-application-classpath-use-compileonly}

계측을 만들 때는 대상 애플리케이션의 클래스를 참조해야 한다. 이 역시
`compileOnly`여야 한다.

```kotlin
// Only accessible in Advice classes during instrumentation
compileOnly("javax.servlet:javax.servlet-api:3.0.1")
```

#### 외부 런타임 의존성(`implementation` 사용) {#external-runtime-dependencies-use-implementation}

익스텐션이 런타임에 필요로 하는 외부 라이브러리는 `implementation` 스코프를
사용해야 하며 shadow JAR에 패키징된다.

```kotlin
implementation("org.apache.commons:commons-lang3:3.19.0")
implementation("com.google.guava:guava:33.0.0-jre")
```

> [!IMPORTANT]
>
> 익스텐션은 별도의 JAR 파일에서 의존성을 로드할 수 없다. 모든 의존성은 단일
> shadow JAR로 병합되어야 한다.

### 익스텐션 지점 개요 {#extension-points-overview}

오픈텔레메트리 Java 에이전트는 SPI 인터페이스를 통해 여러 익스텐션 지점을
제공한다. 가장 흔히 사용되는 것들은 다음과 같다.

> [!NOTE]
>
> 아래의 구성 관련 SPI(예: `AutoConfigurationCustomizerProvider`)는 SDK가 환경
> 변수나 시스템 속성으로 구성될 때 적용된다.
> [선언적 구성](../declarative-configuration)이 사용될 때는 다르게 동작하거나
> 적용되지 않는다. 자세한 내용은 아래 각 익스텐션 지점의 레퍼런스를 참고한다.

| 익스텐션 지점                         | 패키지                                                        | 목적                           |
| ------------------------------------- | ------------------------------------------------------------- | ------------------------------ |
| `AutoConfigurationCustomizerProvider` | `io.opentelemetry.sdk.autoconfigure.spi`                      | SDK 커스터마이즈의 주요 진입점 |
| `ConfigurablePropagatorProvider`      | `io.opentelemetry.sdk.autoconfigure.spi`                      | 커스텀 전파자 등록             |
| `ConfigurableSamplerProvider`         | `io.opentelemetry.sdk.autoconfigure.spi.traces`               | 커스텀 샘플러 등록             |
| `ResourceProvider`                    | `io.opentelemetry.sdk.autoconfigure.spi`                      | 커스텀 리소스 속성 추가        |
| `InstrumenterCustomizerProvider`      | `io.opentelemetry.instrumentation.api.incubator.instrumenter` | 기존 계측 커스터마이즈         |
| `InstrumentationModule`               | `io.opentelemetry.javaagent.extension.instrumentation`        | 새로운 계측 생성               |

내장 및 커뮤니티 구현을 포함한 자동 구성 SPI의 전체 레퍼런스는
[SPI(서비스 프로바이더 인터페이스)](/docs/languages/java/configuration/#spi-service-provider-interface)를
참고한다.

### 익스텐션에서의 구성 {#configuration-in-extensions}

익스텐션은 자신의 동작을 커스터마이즈하기 위해 구성을 읽고 제공할 수 있다.

#### 익스텐션에서 구성에 접근하기 {#accessing-configuration-in-extensions}

많은 SPI 메서드는 구성을 읽을 수 있는 `ConfigProperties` 매개변수를 받는다.

```java
@Override
public Sampler createSampler(ConfigProperties config) {
  // Read configuration with defaults
  String endpoint = config.getString("otel.exporter.otlp.endpoint", "http://localhost:4317");
  int threshold = config.getInt("otel.instrumentation.myext.threshold", 100);
  boolean enabled = config.getBoolean("otel.instrumentation.myext.enabled", true);
  return new MySampler(endpoint, threshold, enabled);
}
```

#### 기본 구성 제공하기 {#providing-default-configuration}

익스텐션은 재정의되지 않은 경우 사용될 기본 구성 값을 제공할 수 있다.

```java
@Override
public void customize(AutoConfigurationCustomizer config) {
  config.addPropertiesSupplier(() -> {
    Map<String, String> props = new HashMap<>();
    props.put("otel.exporter.otlp.endpoint", "http://my-backend:8080");
    props.put("otel.service.name", "my-service");
    props.put("otel.instrumentation.myext.enabled", "true");
    return props;
  });
}
```

#### 구성 이름 규칙 {#configuration-naming-conventions}

구성 매개변수 이름에는 다음 규칙을 따른다.

표준 오픈텔레메트리 속성은 `otel.*` 접두사를 사용한다.

- `otel.service.name`
- `otel.traces.sampler`
- `otel.exporter.otlp.endpoint`

계측별 속성은 `otel.instrumentation.<name>.*`를 사용한다.

- `otel.instrumentation.cassandra.enabled`
- `otel.instrumentation.jdbc.statement-sanitizer.enabled`

익스텐션별 속성도 같은 패턴을 따른다.

- `otel.instrumentation.myextension.enabled`
- `otel.instrumentation.myextension.threshold`
- `otel.instrumentation.myextension.custom-value`

### @AutoService 사용하기 {#using-autoservice}

`@AutoService` 어노테이션은 SPI 등록에 필요한 `META-INF/services/` 파일을
자동으로 생성한다. 사용하려면:

의존성을 추가한다.

```kotlin
compileOnly("com.google.auto.service:auto-service:1.1.1")
annotationProcessor("com.google.auto.service:auto-service:1.1.1")
```

그런 다음 SPI 구현에 다음과 같이 어노테이션을 붙인다.

```java
import com.google.auto.service.AutoService;

@AutoService(AutoConfigurationCustomizerProvider.class)
public class MyExtension implements AutoConfigurationCustomizerProvider {
  // Implementation
}
```

이는 클래스 이름이 담긴
`META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider`를
수동으로 만드는 것과 동일하다.

## 익스텐션 지점 레퍼런스 {#extension-point-reference}

### AutoConfigurationCustomizerProvider {#autoconfigurationcustomizerprovider}

> [!NOTE]
>
> 이것은 [선언적 구성](../declarative-configuration)이 사용되는 상황에서는
> 동작하지 않는다.

이것은 SDK 구성을 커스터마이즈하는 주요 진입점이다. 다음을 할 수 있다.

- 트레이서 프로바이더 커스터마이즈
- 스팬 프로세서 및 익스포터 추가
- 기본 구성 속성 제공
- 기타 SDK 컴포넌트 커스터마이즈

**예시:**

<!-- prettier-ignore-start -->
<?code-excerpt path-base="examples/java-instrumentation/extension"?>
<?code-excerpt "src/main/java/com/example/javaagent/DemoAutoConfigurationCustomizerProvider.java" from="@AutoService"?>
```java
@AutoService(AutoConfigurationCustomizerProvider.class)
public class DemoAutoConfigurationCustomizerProvider
    implements AutoConfigurationCustomizerProvider {

  @Override
  public void customize(AutoConfigurationCustomizer autoConfiguration) {
    autoConfiguration
        .addTracerProviderCustomizer(this::configureSdkTracerProvider)
        .addPropertiesSupplier(this::getDefaultProperties);
  }

  private SdkTracerProviderBuilder configureSdkTracerProvider(
      SdkTracerProviderBuilder tracerProvider, ConfigProperties config) {

    return tracerProvider
        .setIdGenerator(new DemoIdGenerator())
        .setSpanLimits(SpanLimits.builder().setMaxNumberOfAttributes(1024).build())
        .addSpanProcessor(new DemoSpanProcessor())
        .addSpanProcessor(SimpleSpanProcessor.create(new DemoSpanExporter()));
  }

  private Map<String, String> getDefaultProperties() {
    Map<String, String> properties = new HashMap<>();
    properties.put("otel.exporter.otlp.endpoint", "http://backend:8080");
    properties.put("otel.exporter.otlp.insecure", "true");
    properties.put("otel.config.max.attrs", "16");
    properties.put("otel.traces.sampler", "demo");
    return properties;
  }
}
```
<!-- prettier-ignore-end -->

### InstrumenterCustomizerProvider {#instrumentercustomizerprovider}

코드를 수정하지 않고 기존 계측을 커스터마이즈한다. 내장 계측에 속성이나 메트릭을
추가하거나 동작을 수정하는 권장 방법이다.

**예시:**

<!-- prettier-ignore-start -->
<?code-excerpt path-base="examples/java-instrumentation/extension"?>
<?code-excerpt "src/main/java/com/example/javaagent/DemoInstrumenterCustomizerProvider.java" from="/**"?>
```java
/**
 * This example demonstrates how to use the InstrumenterCustomizerProvider SPI to customize
 * instrumentation behavior without modifying the core instrumentation code.
 *
 * <p>This customizer adds:
 *
 * <ul>
 *   <li>Custom attributes to HTTP server spans (based on instrumentation name)
 *   <li>Custom attributes to HTTP client spans (based on instrumentation type)
 *   <li>Custom metrics for HTTP operations
 *   <li>Request correlation IDs via context customization
 *   <li>Custom span name transformation
 * </ul>
 *
 * <p>The customizer will be automatically applied to instrumenters that match the specified
 * instrumentation name or type.
 *
 * @see InstrumenterCustomizerProvider
 * @see InstrumenterCustomizer
 */
@AutoService(InstrumenterCustomizerProvider.class)
public class DemoInstrumenterCustomizerProvider implements InstrumenterCustomizerProvider {

  @Override
  public void customize(InstrumenterCustomizer customizer) {
    String instrumentationName = customizer.getInstrumentationName();
    if (isHttpServerInstrumentation(instrumentationName)) {
      customizeHttpServer(customizer);
    }

    if (customizer.hasType(InstrumenterCustomizer.InstrumentationType.HTTP_CLIENT)) {
      customizeHttpClient(customizer);
    }
  }

  private boolean isHttpServerInstrumentation(String instrumentationName) {
    return instrumentationName.contains("servlet")
        || instrumentationName.contains("jetty")
        || instrumentationName.contains("tomcat")
        || instrumentationName.contains("undertow")
        || instrumentationName.contains("spring-webmvc");
  }

  private void customizeHttpServer(InstrumenterCustomizer customizer) {
    customizer.addAttributesExtractor(new DemoAttributesExtractor());
    customizer.addOperationMetrics(new DemoMetrics());
    customizer.addContextCustomizer(new DemoContextCustomizer());
    customizer.setSpanNameExtractorCustomizer(
        unused -> (SpanNameExtractor<Object>) object -> "CustomHTTP/" + object.toString());
  }

  private void customizeHttpClient(InstrumenterCustomizer customizer) {
    // Simple customization for HTTP client instrumentations
    customizer.addAttributesExtractor(new DemoHttpClientAttributesExtractor());
  }

  /** Custom attributes extractor for HTTP client instrumentations. */
  private static class DemoHttpClientAttributesExtractor
      implements AttributesExtractor<Object, Object> {
    private static final AttributeKey<String> CLIENT_ATTR =
        AttributeKey.stringKey("demo.client.type");

    @Override
    public void onStart(AttributesBuilder attributes, Context context, Object request) {
      attributes.put(CLIENT_ATTR, "demo-http-client");
    }

    @Override
    public void onEnd(
        AttributesBuilder attributes,
        Context context,
        Object request,
        Object response,
        Throwable error) {}
  }

  /** Custom attributes extractor that adds demo-specific attributes. */
  private static class DemoAttributesExtractor implements AttributesExtractor<Object, Object> {
    private static final AttributeKey<String> CUSTOM_ATTR = AttributeKey.stringKey("demo.custom");
    private static final AttributeKey<String> ERROR_ATTR = AttributeKey.stringKey("demo.error");

    @Override
    public void onStart(AttributesBuilder attributes, Context context, Object request) {
      attributes.put(CUSTOM_ATTR, "demo-extension");
    }

    @Override
    public void onEnd(
        AttributesBuilder attributes,
        Context context,
        Object request,
        Object response,
        Throwable error) {
      if (error != null) {
        attributes.put(ERROR_ATTR, error.getClass().getSimpleName());
      }
    }
  }

  /** Custom metrics that track request counts. */
  private static class DemoMetrics implements OperationMetrics {
    @Override
    public OperationListener create(Meter meter) {
      LongCounter requestCounter =
          meter
              .counterBuilder("demo.requests")
              .setDescription("Number of requests")
              .setUnit("requests")
              .build();

      return new OperationListener() {
        @Override
        public Context onStart(Context context, Attributes attributes, long startNanos) {
          requestCounter.add(1, attributes);
          return context;
        }

        @Override
        public void onEnd(Context context, Attributes attributes, long endNanos) {
          // Could add duration metrics here if needed
        }
      };
    }
  }

  /** Context customizer that adds request correlation IDs and custom context data. */
  private static class DemoContextCustomizer implements ContextCustomizer<Object> {
    private static final AtomicLong requestIdCounter = new AtomicLong(1);
    private static final ContextKey<String> REQUEST_ID_KEY = ContextKey.named("demo.request.id");

    @Override
    public Context onStart(Context context, Object request, Attributes startAttributes) {
      // Generate a unique request ID for correlation
      String requestId = "req-" + requestIdCounter.getAndIncrement();

      // Add custom context data that can be accessed throughout the request lifecycle
      context = context.with(REQUEST_ID_KEY, requestId);
      return context;
    }
  }
}
```
<!-- prettier-ignore-end -->

### ConfigurablePropagatorProvider {#configurablepropagatorprovider}

`otel.propagators` 구성에서 이름으로 참조할 수 있는 커스텀 전파자를 등록한다.

**예시:**

<!-- prettier-ignore-start -->
<?code-excerpt path-base="examples/java-instrumentation/extension"?>
<?code-excerpt "src/main/java/com/example/javaagent/DemoPropagatorProvider.java" from="@AutoService"?>
```java
@AutoService(ConfigurablePropagatorProvider.class)
public class DemoPropagatorProvider implements ConfigurablePropagatorProvider {
  @Override
  public TextMapPropagator getPropagator(ConfigProperties config) {
    return new DemoPropagator();
  }

  @Override
  public String getName() {
    return "demo";
  }
}
```
<!-- prettier-ignore-end -->

### ConfigurableSamplerProvider {#configurablesamplerprovider}

`otel.traces.sampler` 구성에서 참조할 수 있는 커스텀 샘플러를 등록한다.

**예시(`otel.traces.sampler=demo`):**

<!-- prettier-ignore-start -->
<?code-excerpt path-base="examples/java-instrumentation/extension"?>
<?code-excerpt "src/main/java/com/example/javaagent/DemoConfigurableSamplerProvider.java" from="@AutoService"?>
```java
@AutoService(ConfigurableSamplerProvider.class)
public class DemoConfigurableSamplerProvider implements ConfigurableSamplerProvider {

  @Override
  public Sampler createSampler(ConfigProperties config) {
    return new DemoSampler();
  }

  @Override
  public String getName() {
    return "demo";
  }
}
```
<!-- prettier-ignore-end -->

### ResourceProvider {#resourceprovider}

다른 리소스 프로바이더와 자동으로 병합될 커스텀 리소스 속성을 추가한다.

**예시:**

<!-- prettier-ignore-start -->
<?code-excerpt path-base="examples/java-instrumentation/extension"?>
<?code-excerpt "src/main/java/com/example/javaagent/DemoResourceProvider.java" from="@AutoService"?>
```java
@AutoService(ResourceProvider.class)
public class DemoResourceProvider implements ResourceProvider {
  @Override
  public Resource createResource(ConfigProperties config) {
    Attributes attributes = Attributes.builder().put("custom.resource", "demo").build();
    return Resource.create(attributes);
  }
}
```
<!-- prettier-ignore-end -->

## 익스텐션 예제 {#extension-examples}

더 많은 익스텐션 예제는 Java 계측 저장소의
[익스텐션 프로젝트](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/examples/extension)를
참고한다.
