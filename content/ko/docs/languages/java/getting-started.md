---
title: 예제로 시작하기
description: 5분 이내에 애플리케이션의 텔레메트리를 확보한다!
weight: 10
default_lang_commit: ffaf53d830ea8a37dc6cf72c624b82fb2058fb10
---

<?code-excerpt path-base="examples/java/getting-started"?>

이 페이지는 Java에서 오픈텔레메트리를 시작하는 방법을 보여준다.

간단한 Java 애플리케이션을 자동으로 계측하여 [트레이스][traces],
[메트릭][metrics], [로그][logs]가 콘솔로 출력되도록 하는 방법을 배운다.

## 사전 요구 사항 {#prerequisites}

다음이 로컬에 설치되어 있는지 확인한다.

- Spring Boot 3을 사용하므로 Java JDK 17 이상이 필요하다. [그 외의 경우 Java 8
  이상][java-vers]
- [Gradle](https://gradle.org/)

## 예제 애플리케이션 {#example-application}

다음 예제는 기본적인 [Spring Boot][] 애플리케이션을 사용한다. Apache Wicket이나
Play와 같은 다른 웹 프레임워크를 사용할 수도 있다. 라이브러리와 지원되는
프레임워크의 전체 목록은
[레지스트리](/ecosystem/registry/?component=instrumentation&language=java)를
참고한다.

더 정교한 예제는 [예제](../examples/)를 참고한다.

### 의존성 {#dependencies}

시작하려면 `java-simple`이라는 새 디렉터리에 환경을 설정한다. 해당 디렉터리 안에
다음 내용으로 `build.gradle.kts` 파일을 생성한다.

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.0.6"
  id("io.spring.dependency-management") version "1.1.0"
}

sourceSets {
  main {
    java.setSrcDirs(setOf("."))
  }
}

repositories {
  mavenCentral()
}

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
}
```

### HTTP 서버 생성 및 실행 {#create-and-launch-an-http-server}

동일한 폴더에 `DiceApplication.java` 파일을 생성하고 다음 코드를 파일에
추가한다.

<?code-excerpt "src/main/java/otel/DiceApplication.java"?>

```java
package otel;

import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DiceApplication {
  public static void main(String[] args) {
    SpringApplication app = new SpringApplication(DiceApplication.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
  }
}
```

`RollController.java`라는 또 다른 파일을 생성하고 다음 코드를 파일에 추가한다.

<?code-excerpt "src/main/java/otel/RollController.java"?>

```java
package otel;

import java.util.Optional;
import java.util.concurrent.ThreadLocalRandom;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RollController {
  private static final Logger logger = LoggerFactory.getLogger(RollController.class);

  @GetMapping("/rolldice")
  public String index(@RequestParam("player") Optional<String> player) {
    int result = this.getRandomNumber(1, 6);
    if (player.isPresent()) {
      logger.info("{} is rolling the dice: {}", player.get(), result);
    } else {
      logger.info("Anonymous player is rolling the dice: {}", result);
    }
    return Integer.toString(result);
  }

  public int getRandomNumber(int min, int max) {
    return ThreadLocalRandom.current().nextInt(min, max + 1);
  }
}
```

다음 명령으로 애플리케이션을 빌드하고 실행한 다음, 웹 브라우저에서
<http://localhost:8080/rolldice>를 열어 정상적으로 동작하는지 확인한다.

```sh
gradle assemble
java -jar ./build/libs/java-simple.jar
```

## 계측 {#instrumentation}

다음으로, 실행 시점에 애플리케이션을 자동으로 계측하기 위해
[Java 에이전트](/docs/zero-code/java/agent/)를 사용한다. [Java 에이전트를
구성하는][configure the Java agent] 방법은 여러 가지가 있지만, 아래 단계에서는
환경 변수를 사용한다.

1. `opentelemetry-java-instrumentation` 저장소의 [릴리스][releases]에서
   [opentelemetry-javaagent.jar][]를 다운로드한다. 이 JAR 파일은 에이전트와 모든
   자동 계측 패키지를 포함한다.

   ```sh
   curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
   ```

   > [!IMPORTANT] <i class="fas fa-edit"></i> JAR 파일의 경로를 기록해 둔다.

2. Java 에이전트 JAR과 [콘솔 익스포터][console exporter]를 지정하는 변수를
   셸/터미널 환경에 적합한 표기법으로 설정하고 내보낸다(export). 여기서는 bash
   계열 셸에 대한 표기법을 예시로 든다.

   ```sh
   export JAVA_TOOL_OPTIONS="-javaagent:PATH/TO/opentelemetry-javaagent.jar" \
     OTEL_TRACES_EXPORTER=console \
     OTEL_METRICS_EXPORTER=console \
     OTEL_LOGS_EXPORTER=console \
     OTEL_METRIC_EXPORT_INTERVAL=15000
   ```

   <!-- markdownlint-disable no-blanks-blockquote -->

   > [!NOTE]
   >
   > 위 `PATH/TO`를 실제 JAR 경로로 바꾼다.

   > [!WARNING]
   >
   > 메트릭이 제대로 생성되는지 더 빠르게 확인할 수 있도록, 위에서 보여준 것처럼
   > **테스트 중에만** `OTEL_METRIC_EXPORT_INTERVAL`을 기본값보다 훨씬 낮은
   > 값으로 설정한다.

3. **애플리케이션**을 다시 한번 실행한다.

   ```console
   $ java -jar ./build/libs/java-simple.jar
   ...
   ```

   `otel.javaagent`의 출력을 확인한다.

4. _다른_ 터미널에서 `curl`을 사용해 요청을 보낸다.

   ```sh
   curl localhost:8080/rolldice
   ```

5. 서버 프로세스를 중지한다.

4단계에서 서버와 클라이언트로부터 다음과 비슷한 트레이스 및 로그 출력을 확인했을
것이다(편의를 위해 트레이스 출력은 줄바꿈되어 있다).

```sh
[otel.javaagent 2023-04-24 17:33:54:567 +0200] [http-nio-8080-exec-1] INFO
io.opentelemetry.exporter.logging.LoggingSpanExporter - 'RollController.index' :
 70c2f04ec863a956e9af975ba0d983ee 7fd145f5cda13625 INTERNAL [tracer:
 io.opentelemetry.spring-webmvc-6.0:1.25.0-alpha] AttributesMap{data=
 {thread.id=39, thread.name=http-nio-8080-exec-1}, capacity=128,
 totalAddedValues=2}
[otel.javaagent 2023-04-24 17:33:54:568 +0200] [http-nio-8080-exec-1] INFO
io.opentelemetry.exporter.logging.LoggingSpanExporter - 'GET /rolldice' :
70c2f04ec863a956e9af975ba0d983ee 647ad186ad53eccf SERVER [tracer:
io.opentelemetry.tomcat-10.0:1.25.0-alpha] AttributesMap{
  data={user_agent.original=curl/7.87.0, net.host.name=localhost,
  net.transport=ip_tcp, http.target=/rolldice, net.sock.peer.addr=127.0.0.1,
  thread.name=http-nio-8080-exec-1, net.sock.peer.port=53422,
  http.route=/rolldice, net.sock.host.addr=127.0.0.1, thread.id=39,
  net.protocol.name=http, http.status_code=200, http.scheme=http,
  net.protocol.version=1.1, http.response_content_length=1,
  net.host.port=8080, http.method=GET}, capacity=128, totalAddedValues=17}
```

5단계에서 서버를 중지할 때, 수집된 모든 메트릭의 출력을 확인할 수 있을
것이다(편의를 위해 메트릭 출력은 줄바꿈되고 축약되어 있다).

```sh
[otel.javaagent 2023-04-24 17:34:25:347 +0200] [PeriodicMetricReader-1] INFO
io.opentelemetry.exporter.logging.LoggingMetricExporter - Received a collection
 of 19 metrics for export.
[otel.javaagent 2023-04-24 17:34:25:347 +0200] [PeriodicMetricReader-1] INFO
io.opentelemetry.exporter.logging.LoggingMetricExporter - metric:
ImmutableMetricData{resource=Resource{schemaUrl=
https://opentelemetry.io/schemas/1.19.0, attributes={host.arch="aarch64",
host.name="OPENTELEMETRY", os.description="Mac OS X 13.3.1", os.type="darwin",
process.command_args=[/bin/java, -jar, java-simple.jar],
process.executable.path="/bin/java", process.pid=64497,
process.runtime.description="Homebrew OpenJDK 64-Bit Server VM 20",
process.runtime.name="OpenJDK Runtime Environment",
process.runtime.version="20", service.name="java-simple",
telemetry.auto.version="1.25.0", telemetry.sdk.language="java",
telemetry.sdk.name="opentelemetry", telemetry.sdk.version="1.25.0"}},
instrumentationScopeInfo=InstrumentationScopeInfo{name=io.opentelemetry.runtime-metrics,
version=1.25.0, schemaUrl=null, attributes={}},
name=process.runtime.jvm.buffer.limit, description=Total capacity of the buffers
in this pool, unit=By, type=LONG_SUM, data=ImmutableSumData{points=
[ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes=
{pool="mapped - 'non-volatile memory'"}, value=0, exemplars=[]},
ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes={pool="mapped"},
value=0, exemplars=[]},
ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes={pool="direct"},
value=8192, exemplars=[]}], monotonic=false, aggregationTemporality=CUMULATIVE}}
...
```

## 다음은? {#what-next}

더 알아보려면 다음을 참고한다.

- 텔레메트리 데이터를 위한 다른 [익스포터][exporter]로 이 예제를 실행해본다.
- 자신의 애플리케이션 중 하나에 [제로 코드 계측](/docs/zero-code/java/agent/)을
  시도해본다.
- 가벼운 커스터마이즈 텔레메트리를 원한다면 [어노테이션][annotations]을
  시도해본다.
- [수동 계측][manual instrumentation]에 대해 배우고 더 많은
  [예제](../examples/)를 사용해본다.
- Java 기반 [Ad Service](/docs/demo/services/ad/)와 Kotlin 기반
  [Fraud Detection Service](/docs/demo/services/fraud-detection/)를 포함하는
  [오픈텔레메트리 데모](/docs/demo/)를 살펴본다.

[traces]: /docs/concepts/signals/traces/
[metrics]: /docs/concepts/signals/metrics/
[logs]: /docs/concepts/signals/logs/
[annotations]: /docs/zero-code/java/agent/annotations/
[configure the java agent]: /docs/zero-code/java/agent/configuration/
[console exporter]: /docs/languages/java/configuration/#properties-exporters
[exporter]: /docs/languages/java/configuration/#properties-exporters
[java-vers]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/VERSIONING.md#language-version-compatibility
[manual instrumentation]: ../instrumentation
[opentelemetry-javaagent.jar]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
[releases]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases
[Spring Boot]: https://spring.io/guides/gs/spring-boot/
