---
title: JMX 메트릭
weight: 14
description:
  오픈텔레메트리(OpenTelemetry)를 사용해 JMX MBean으로부터 메트릭을 수집한다
cSpell:ignore: jconsole jmxremote mbean mbeans visualvm wildfly
default_lang_commit: 4ce7276981ee26a4680c9c1387e542a98b029440
---

이 페이지는
[JMX](https://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html)(Java
Management Extensions) MBean으로부터 메트릭을 수집하여 오픈텔레메트리로 내보내는
방법을 설명한다.

## 개요 {#overview}

JMX(Java Management Extensions)는 JMX MBean(Managed Bean)을 통해 애플리케이션을
관리하고 모니터링하는 도구를 제공하는 Java 기술이다. 이러한 MBean은 외부에서
관측할 수 있는 관리 속성과 작업을 노출한다.

오픈텔레메트리 JMX Metric Insight 모듈을 사용하면 JMX 메트릭을 오픈텔레메트리로
연결할 수 있으며, 이를 통해 다음이 가능해진다.

- JVM 메트릭(메모리, 가비지 컬렉션, 스레드 등) 모니터링
- 애플리케이션별 MBean으로부터 메트릭 수집
- 다른 오픈텔레메트리 텔레메트리 시그널과 함께 JMX 데이터 내보내기
- 인기 있는 대상 시스템(Tomcat, Jetty, Wildfly 등)을 위한 사전 정의된 메트릭
  매핑 사용

## 설치 {#installation}

### Java 에이전트 사용하기 {#using-the-java-agent}

JMX 메트릭을 수집하는 가장 쉬운 방법은 JMX 메트릭 익스텐션과 함께 오픈텔레메트리
Java 에이전트를 사용하는 것이다.

1. 오픈텔레메트리 Java 에이전트를 다운로드한다(아직 설치하지 않았다면).

   ```sh
   curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
   ```

2. 에이전트와 함께 애플리케이션을 실행하고 JMX 메트릭을 활성화한다.

   ```sh
   java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.jmx.target.system=tomcat \
     -Dotel.jmx.config=/path/to/custom-metrics.yaml \
     -jar myapp.jar
   ```

JMX 메트릭 수집은 다음 구성 옵션 중 하나(또는 둘 다)를 설정하여 활성화된다.

- 활성화할 사전 정의된 메트릭 집합을 선택하는 `otel.jmx.target.system`
- 커스텀 JMX 규칙 경로를 제공하는 `otel.jmx.config`

Java 에이전트를 사용할 때, JVM 런타임 메트릭(cpu, memory 등)은
`runtime-telemetry` 모듈을 통해 캡처되며 추가 구성 없이 기본으로 활성화된다.

## 구성 {#configuration}

JMX 메트릭은 두 가지 방법으로 수집할 수 있다.

- **JVM 내부에서**, Java 에이전트와 함께 내부 JMX 인터페이스를 사용한다.
- **JVM 외부에서**, JMX 스크레이퍼와 함께 원격 JMX 인터페이스를 사용한다.

### Java 에이전트 구성 {#java-agent-configuration}

오픈텔레메트리 Java 에이전트를 사용할 때는 다음 속성을 사용해 JMX 메트릭을
구성한다.

| 시스템 속성              | 환경 변수                | 설명                                                | 기본값 |
| ------------------------ | ------------------------ | --------------------------------------------------- | ------ |
| `otel.jmx.enabled`       | `OTEL_JMX_ENABLED`       | JMX 메트릭 수집을 활성화한다.                       | `true` |
| `otel.jmx.target.system` | `OTEL_JMX_TARGET_SYSTEM` | 사용할 사전 정의된 메트릭 집합의 쉼표로 구분된 목록 | 없음   |
| `otel.jmx.config`        | `OTEL_JMX_CONFIG`        | 메트릭 매핑을 위한 커스텀 YAML 경로                 | 없음   |

### JMX 스크레이퍼 구성 {#jmx-scraper-configuration}

독립형 JMX 스크레이퍼를 사용해 원격 JVM으로부터 메트릭을 수집할 때는 다음 속성을
사용해 구성한다(참고: `otel.jmx.enabled`는 필요하지 않다).

| 시스템 속성              | 환경 변수                | 설명                                                | 기본값 |
| ------------------------ | ------------------------ | --------------------------------------------------- | ------ |
| `otel.jmx.service.url`   | `OTEL_JMX_SERVICE_URL`   | 원격 JVM 연결을 위한 JMX 서비스 URL                 | (필수) |
| `otel.jmx.target.system` | `OTEL_JMX_TARGET_SYSTEM` | 사용할 사전 정의된 메트릭 집합의 쉼표로 구분된 목록 | 없음   |
| `otel.jmx.config`        | `OTEL_JMX_CONFIG`        | 메트릭 매핑을 위한 커스텀 YAML 경로                 | 없음   |

전체 구성 참조는
[JMX 스크레이퍼 문서](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper#configuration-reference)를
참고한다.

원격 JVM은 원격 JMX 연결을 허용하도록 구성되어야 한다는 점에 유의한다.
`jconsole`이나 `visualvm` 도구로 연결할 수 있는지 확인하는 것이, 구성과 선택적
인증이 예상대로 동작하는지 확인하는 첫 단계로 권장된다.

### 사전 정의된 대상 시스템 {#predefined-target-systems}

오픈텔레메트리는 인기 있는 Java 프레임워크와 애플리케이션 서버를 위한 사전
정의된 메트릭 매핑을 제공한다. 이를 활성화하려면 `otel.jmx.target.system` 속성을
사용한다(Java 에이전트와 JMX 스크레이퍼 모두에서 사용 가능).

**예제 - Tomcat 모니터링(Java 에이전트):**

```sh
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.jmx.target.system=tomcat \
  -jar myapp.jar
```

사용 가능한 대상 시스템의 전체 목록은 다음을 참고한다.

- [Java 에이전트 사전 정의된 대상 시스템](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/README.md#predefined-metrics)
- [JMX 스크레이퍼 사전 정의된 대상 시스템](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper#configuration-reference)

쉼표로 구분하여 여러 대상 시스템을 지정할 수 있다.

### 원격 JMX 연결 {#remote-jmx-connections}

원격 JVM으로부터 메트릭을 수집하려면 JMX 스크레이퍼를 사용해야 한다. 이는 두
개의 개별 JVM을 필요로 한다.

1. **대상 JVM** - 모니터링 대상 애플리케이션
2. **스크레이퍼 JVM** - JMX 메트릭 스크레이퍼

#### Step 1: 대상 JVM 구성하기 {#step-1-configure-the-target-jvm}

먼저, JMX 원격 기능을 활성화하여 대상 애플리케이션을 시작한다.

```sh
java -Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  -jar myapp.jar
```

> [!WARNING] 위 예제는 단순화를 위해 인증과 SSL을 비활성화했다. 프로덕션
> 환경에서는 JMX 연결에 항상 인증과 SSL을 활성화한다.

#### Step 2: JMX 스크레이퍼 실행하기 {#step-2-run-the-jmx-scraper}

[오픈텔레메트리 Java Contrib 릴리스](https://github.com/open-telemetry/opentelemetry-java-contrib/releases)
페이지에서 JMX 스크레이퍼를
다운로드한다(`opentelemetry-jmx-scraper-<version>-all.jar`를 찾는다).

그런 다음 대상 JVM을 가리키도록 스크레이퍼를 실행한다.

```sh
java -Dotel.jmx.service.url=service:jmx:rmi:///jndi/rmi://tomcat.example.com:9999/jmxrmi \
  -Dotel.jmx.target.system=tomcat \
  -jar opentelemetry-jmx-scraper.jar
```

Java 에이전트와 동일한 속성(대상 시스템, 수집 간격 등)을 사용해 스크레이퍼를
구성할 수 있다.

자세한 내용은
[JMX 스크레이퍼 문서](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper)를
참고한다.

> [!NOTE] 지원 중단된 JMX Metric Gatherer에서 마이그레이션하는 경우
> [마이그레이션 가이드](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper#migration-from-jmx-gatherer)를
> 참고한다.

## 커스텀 메트릭 매핑 {#custom-metric-mappings}

애플리케이션별 MBean이나 커스텀 모니터링 요구 사항이 있는 경우, YAML 구성 파일을
사용해 커스텀 메트릭 매핑을 생성할 수 있다.

### 커스텀 YAML 구성 생성하기 {#creating-a-custom-yaml-configuration}

JMX 속성을 오픈텔레메트리 메트릭에 매핑하는 방법을 정의하는 YAML 파일을
생성한다.

**예제 - `custom-jmx-metrics.yaml`:**

```yaml
rules:
  - bean: com.myapp:type=CustomMetrics
    mapping:
      RequestCount:
        metric: myapp.requests.count
        type: counter
        description: Total request count
        unit: '1'
      ResponseTime:
        metric: myapp.response.time
        type: gauge
        description: Average response time
        unit: ms
      ActiveSessions:
        metric: myapp.sessions.active
        type: updowncounter
        description: Active sessions
        unit: '1'
```

애플리케이션에서 이 파일을 사용한다.

```sh
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.jmx.config=/path/to/custom-jmx-metrics.yaml \
  -jar myapp.jar
```

YAML 구문에 대한 전체 참조는
[JMX 메트릭 구성 문서](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/jmx-metrics)를
참고한다.

## 검증 {#verification}

JMX 메트릭이 수집되고 있는지 검증하려면 다음을 수행한다.

1. **로그 확인** - JMX 메트릭 수집이 시작되었음을 나타내는 메시지를 찾는다
2. **로깅 익스포터 사용** - 백엔드 없이 콘솔에서 메트릭을 확인하도록 로깅
   익스포터를 구성한다
3. **메트릭 백엔드 사용** - OTLP 익스포터를 구성하고 옵저버빌리티 플랫폼에서
   메트릭을 확인한다
4. **JConsole 사용** - JConsole로 애플리케이션에 연결하여 MBean에 접근할 수
   있는지 확인한다

**로깅 익스포터를 사용하는 예제(Java 에이전트):**

```sh
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.metrics.exporter=logging \
  -jar myapp.jar
```

**OTLP 익스포터를 사용하는 예제(Java 에이전트):**

```sh
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.metrics.exporter=otlp \
  -Dotel.exporter.otlp.endpoint=http://localhost:4318 \
  -jar myapp.jar
```

**OTLP 익스포터를 사용하는 예제(JMX 스크레이퍼):**

```sh
java -Dotel.jmx.service.url=service:jmx:rmi:///jndi/rmi://myapp.example.com:9999/jmxrmi \
  -Dotel.jmx.target.system=tomcat \
  -Dotel.metrics.exporter=otlp \
  -Dotel.exporter.otlp.endpoint=http://localhost:4318 \
  -jar opentelemetry-jmx-scraper.jar
```

## 추가 리소스 {#additional-resources}

- [JMX 스크레이퍼 문서](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper) -
  전체 구성 참조 및 예제
- [JMX 스크레이퍼 마이그레이션 가이드](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper#migration-from-jmx-gatherer) -
  지원 중단된 JMX Metric Gatherer로부터 마이그레이션
- [JMX 메트릭(Java 에이전트)](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/README.md) -
  Java 에이전트 JMX 메트릭 문서
- [사전 정의된 대상 시스템](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/jmx-scraper#configuration-reference) -
  인기 있는 프레임워크를 위한 내장 메트릭 집합
- [Java 에이전트 문서](/docs/zero-code/java/agent/) - 일반적인 Java 에이전트
  구성
- [구성 가이드](../configuration/) - 오픈텔레메트리 SDK 구성 옵션

## 관련 주제 {#related-topics}

- [계측 생태계](../instrumentation/) - 다른 계측 옵션
- [심(Shim)](../instrumentation/#shims) - 다른 옵저버빌리티 라이브러리 연결
- [메트릭 API](../api/#meterprovider) - 커스텀 메트릭 생성
