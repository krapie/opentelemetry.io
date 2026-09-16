---
title: 계측 생태계
aliases:
  - /docs/java/getting_started
  - /docs/java/manual_instrumentation
  - manual
  - manual_instrumentation
  - libraries
weight: 10
description: 오픈텔레메트리(OpenTelemetry) Java의 계측 생태계
default_lang_commit: b51a1db58883aa963c461d34356aa86ac18d94b7
---

<!-- markdownlint-disable no-duplicate-heading -->

계측은 [API](../api/)를 사용해 텔레메트리를 기록한다. [SDK](../sdk/)는 API의
내장 참조 구현체로, 계측 API 호출로 생성된 텔레메트리를 처리하고 내보내도록
[구성](../configuration/)된다. 이 페이지는 최종 사용자를 위한 리소스와 범용적인
계측 주제를 포함하여, 오픈텔레메트리 Java에서의 오픈텔레메트리 생태계를 다룬다.

- [계측 범주](#instrumentation-categories)는 서로 다른 사용 사례와 설치 패턴을
  다룬다.
- [컨텍스트 전파](#context-propagation)는 트레이스, 메트릭, 로그 간의 상관관계를
  제공하여 시그널들이 서로를 보완하게 해준다.
- [시맨틱 컨벤션](#semantic-conventions)은 표준 작업에 대한 텔레메트리를
  생성하는 방법을 정의한다.
- [로그 계측](#log-instrumentation)은 기존 Java 로깅 프레임워크의 로그를
  오픈텔레메트리로 가져오는 데 사용된다.
- [JMX 메트릭](../jmx/)은 JMX MBean을 통해 JVM 및 애플리케이션 메트릭을
  모니터링한다.

> [!NOTE]
>
> [계측 범주](#instrumentation-categories)는 애플리케이션을 계측하는 여러 옵션을
> 열거하지만, 사용자는 [Java 에이전트](#zero-code-java-agent)로 시작할 것을
> 권장한다. Java 에이전트는 설치 과정이 간단하며, 방대한 라이브러리로부터 계측을
> 자동으로 감지하고 설치한다.

## 계측 범주 {#instrumentation-categories}

계측에는 여러 범주가 있다.

- [제로 코드: Java 에이전트](#zero-code-java-agent)는 애플리케이션 바이트코드를
  동적으로 조작하는 제로 코드 계측(zero-code instrumentation) **[1]** 형태이다.
- [제로 코드: Spring Boot 스타터](#zero-code-spring-boot-starter)는 Spring
  자동구성(autoconfigure)을 활용해 [라이브러리 계측](#library-instrumentation)을
  설치하는 제로 코드 계측 **[1]** 형태이다.
- [라이브러리 계측](#library-instrumentation)은 라이브러리를 계측하기 위해
  익스텐션 포인트를 감싸거나 사용하며, 사용자가 라이브러리 사용법을 설치 및/또는
  조정해야 한다.
- [네이티브 계측](#native-instrumentation)은 라이브러리와 프레임워크에 직접
  내장된다.
- [수동 계측](#manual-instrumentation)은 애플리케이션 작성자가 작성하며,
  일반적으로 애플리케이션 도메인에 특화되어 있다.
- [심(Shim)](#shims)은 한 옵저버빌리티 라이브러리에서 다른 라이브러리로 데이터를
  연결하며, 일반적으로 특정 라이브러리에서 오픈텔레메트리로 연결하는 방향이다.

**[1]**: 제로 코드 계측은 감지된 라이브러리/프레임워크에 기반해 자동으로
설치된다.

[opentelemetry-java-instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
프로젝트는 Java 에이전트, Spring Boot 스타터, 라이브러리 계측의 소스 코드를 담고
있다.

### 제로 코드: Java 에이전트 {#zero-code-java-agent}

Java 에이전트는 애플리케이션 바이트코드를 동적으로 조작하는 제로 코드
[자동 계측](/docs/specs/otel/glossary/#automatic-instrumentation) 형태이다.

Java 에이전트가 계측하는 라이브러리 목록은
[지원되는 라이브러리](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)의
"Auto-instrumented versions" 열을 참고한다.

자세한 내용은 [Java 에이전트](/docs/zero-code/java/agent/)를 참고한다.

### 제로 코드: Spring Boot 스타터 {#zero-code-spring-boot-starter}

Spring Boot 스타터는 Spring 자동구성(autoconfigure)을 활용해
[라이브러리 계측](#library-instrumentation)을 설치하는 제로 코드
[자동 계측](/docs/specs/otel/glossary/#automatic-instrumentation) 형태이다.

자세한 내용은 [Spring Boot 스타터](/docs/zero-code/java/spring-boot-starter/)를
참고한다.

### 라이브러리 계측 {#library-instrumentation}

[라이브러리 계측](/docs/specs/otel/glossary/#instrumentation-library)은
라이브러리를 계측하기 위해 익스텐션 포인트를 감싸거나 사용하며, 사용자가
라이브러리 사용법을 설치 및/또는 조정해야 한다.

계측 라이브러리 목록은
[지원되는 라이브러리](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)의
"Standalone Library Instrumentation" 열을 참고한다.

### 네이티브 계측 {#native-instrumentation}

[네이티브 계측](/docs/specs/otel/glossary/#natively-instrumented)은 라이브러리나
프레임워크에 직접 내장된다. 오픈텔레메트리는 라이브러리 작성자가
[API](../api/)를 사용해 네이티브 계측을 추가할 것을 권장한다. 장기적으로
네이티브 계측이 표준이 되기를 바라며,
[opentelemetry-java-instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)에서
오픈텔레메트리가 유지 관리하는 계측은 그 간극을 메우기 위한 임시 수단으로 본다.

네이티브 계측은 다음과 같이 오픈텔레메트리 Java 에이전트와 상호작용해야 한다.
시작 시 Java 에이전트는 [OpenTelemetry](../api/#opentelemetry) 인스턴스를
초기화하고 [제로 코드](#zero-code-java-agent) 계측을 설치한다. 네이티브 계측을
추가하는 라이브러리는 사용할 `OpenTelemetry` 인스턴스를 사용자가 커스터마이즈할
수 있게 해야 하지만, (존재한다면) Java 에이전트가 초기화한 인스턴스를 자동으로
사용해야 한다. 이를 달성하는 방법에 대한 안내는
[GlobalOpenTelemetry](../api/#globalopentelemetry)를 참고한다.

{{% docs/languages/native-libraries %}}

### 수동 계측 {#manual-instrumentation}

[수동 계측](/docs/specs/otel/glossary/#manual-instrumentation)은 애플리케이션
작성자가 작성하며, 일반적으로 애플리케이션 도메인에 특화되어 있다.

수동 계측은 다음과 같이 오픈텔레메트리 Java 에이전트와 상호작용해야 한다. 시작
시 Java 에이전트는 [OpenTelemetry](../api/#opentelemetry) 인스턴스를 초기화하고,
`GlobalOpenTelemetry`를 통해 애플리케이션의 수동 계측이 이를 사용할 수 있게
한다. 하지만 애플리케이션 소유자는 Java 에이전트가 항상 설치되어 있다고 가정할
수 없다. 예를 들어 Java 에이전트는 로컬 개발이나 테스트 환경에는 설치되지 않을
수 있으며, 디버깅 목적으로 Java 에이전트를 제거하는 특수한 경우도 있다. 수동
계측은 (존재한다면) Java 에이전트가 초기화한
[OpenTelemetry](../api/#opentelemetry) 인스턴스를 사용해야 하지만, Java
에이전트가 없다면 이를 감지하고 대체 `OpenTelemetry` 인스턴스를 설정할 수 있어야
한다. 이를 달성하는 방법에 대한 안내는
[GlobalOpenTelemetry](../api/#globalopentelemetry)를 참고한다.

### 심(Shim) {#shims}

심(shim)은 한 옵저버빌리티 라이브러리에서 다른 라이브러리로 데이터를 연결하는
계측이며, 일반적으로 특정 라이브러리에서 오픈텔레메트리로 연결하는
방향(_from_)이다.

오픈텔레메트리 Java 생태계에서 유지 관리되는 심:

| 설명                                                                                                                | 문서                                                                                                                                                                                    | 시그널           | 아티팩트                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| [OpenTracing](https://opentracing.io/)를 오픈텔레메트리로 연결                                                      | [README](https://github.com/open-telemetry/opentelemetry-java/tree/main/opentracing-shim)                                                                                               | 트레이스         | `io.opentelemetry:opentelemetry-opentracing-shim:{{% param vers.otel %}}`                                                       |
| [Opencensus](https://opencensus.io/)를 오픈텔레메트리로 연결                                                        | [README](https://github.com/open-telemetry/opentelemetry-java/tree/main/opencensus-shim)                                                                                                | 트레이스, 메트릭 | `io.opentelemetry:opentelemetry-opencensus-shim:{{% param vers.otel %}}-alpha`                                                  |
| [Micrometer](https://micrometer.io/)를 오픈텔레메트리로 연결                                                        | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/micrometer/micrometer-1.5/library)                                              | 메트릭           | `io.opentelemetry.instrumentation:opentelemetry-micrometer-1.5:{{% param vers.instrumentation %}}-alpha`                        |
| [JMX](https://docs.oracle.com/javase/7/docs/technotes/guides/management/agent.html)를 오픈텔레메트리로 연결         | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/README.md), [JMX 메트릭 가이드](../jmx/)                            | 메트릭           | `io.opentelemetry.instrumentation:opentelemetry-jmx-metrics:{{% param vers.instrumentation %}}-alpha`                           |
| 오픈텔레메트리를 [Prometheus Java SimpleClient](https://github.com/prometheus/client_java/tree/simpleclient)로 연결 | [README](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/prometheus-client-bridge)                                                                               | 메트릭           | `io.opentelemetry.contrib:opentelemetry-prometheus-client-bridge:{{% param vers.contrib %}}-alpha`                              |
| 오픈텔레메트리를 [Prometheus Java PrometheusRegistry](https://github.com/prometheus/client_java)로 연결             | [PrometheusMetricReader Javadoc](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-exporter-prometheus/latest/io/opentelemetry/exporter/prometheus/PrometheusMetricReader.html) | 메트릭           | `io.opentelemetry:opentelemetry-exporter-prometheus:{{% param vers.otel %}}-alpha`                                              |
| 오픈텔레메트리를 [Micrometer](https://micrometer.io/)로 연결                                                        | [README](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/micrometer-meter-provider)                                                                              | 메트릭           | `io.opentelemetry.contrib:opentelemetry-micrometer-meter-provider:{{% param vers.contrib %}}-alpha`                             |
| [Log4j](https://logging.apache.org/log4j/2.x/index.html)를 오픈텔레메트리로 연결                                    | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/log4j/log4j-appender-2.17/library)                                              | 로그             | `io.opentelemetry.instrumentation:opentelemetry-log4j-appender-2.17:{{% param vers.instrumentation %}}-alpha`                   |
| [Logback](https://logback.qos.ch/)을 오픈텔레메트리로 연결                                                          | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/logback/logback-appender-1.0/library)                                           | 로그             | `io.opentelemetry.instrumentation:opentelemetry-logback-appender-1.0:{{% param vers.instrumentation %}}-alpha`                  |
| 오픈텔레메트리 컨텍스트를 [Log4j](https://logging.apache.org/log4j/2.x/index.html)로 연결                           | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/log4j/log4j-context-data/log4j-context-data-2.17/library-autoconfigure)         | 컨텍스트         | `io.opentelemetry.instrumentation:opentelemetry-log4j-context-data-2.17-autoconfigure:{{% param vers.instrumentation %}}-alpha` |
| 오픈텔레메트리 컨텍스트를 [Logback](https://logback.qos.ch/)으로 연결                                               | [README](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/logback/logback-mdc-1.0/library)                                                | 컨텍스트         | `io.opentelemetry.instrumentation:opentelemetry-logback-mdc-1.0:{{% param vers.instrumentation %}}-alpha`                       |

## 컨텍스트 전파 {#context-propagation}

오픈텔레메트리 API는 상호 보완적으로 설계되어, 전체가 부분의 합보다 크다. 각
시그널은 고유한 강점을 가지며, 이들이 모여 설득력 있는 옵저버빌리티 스토리를
엮어낸다.

중요한 점은, 다양한 시그널의 데이터가 트레이스 컨텍스트를 통해 서로 연결된다는
것이다.

- 스팬은 스팬 부모와 링크를 통해 다른 스팬과 관계를 맺으며, 이들 각각은 관련
  스팬의 트레이스 컨텍스트를 기록한다.
- 메트릭은 [예시값(exemplar)](/docs/specs/otel/metrics/data-model/#exemplars)을
  통해 스팬과 관계를 맺으며, 이는 특정 측정값의 트레이스 컨텍스트를 기록한다.
- 로그는 로그 레코드에 트레이스 컨텍스트를 기록함으로써 스팬과 관계를 맺는다.

이러한 상관관계가 동작하려면, 트레이스 컨텍스트가 애플리케이션 전체(함수 호출과
스레드를 가로질러)와 애플리케이션 경계를 넘어 전파되어야 한다.
[컨텍스트 API](../api/#context-api)가 이를 돕는다. 계측은 컨텍스트를 인식하는
방식으로 작성되어야 한다.

- 애플리케이션의 진입점을 나타내는 라이브러리(즉, HTTP 서버, 메시지 컨슈머 등)는
  수신 메시지로부터 [컨텍스트를 추출](../api/#contextpropagators)해야 한다.
- 애플리케이션의 출구점을 나타내는 라이브러리(즉, HTTP 클라이언트, 메시지
  프로듀서 등)는 발신 메시지에 [컨텍스트를 주입](../api/#contextpropagators)해야
  한다.
- 라이브러리는 콜스택을 통해 그리고 모든 스레드에 걸쳐 암묵적으로 또는
  명시적으로 [Context](../api/#context)를 전달해야 한다.

## 시맨틱 컨벤션 {#semantic-conventions}

[시맨틱 컨벤션](/docs/specs/semconv/)은 표준 작업에 대한 텔레메트리를 생성하는
방법을 정의한다. 특히 시맨틱 컨벤션은 스팬 이름, 스팬 종류, 메트릭 계측기,
메트릭 단위, 메트릭 타입, 속성 키/값/요구 수준을 명시한다.

계측을 작성할 때는 시맨틱 컨벤션을 참고하고, 해당 도메인에 적용 가능한 컨벤션을
준수한다.

오픈텔레메트리 Java는 속성 키와 값에 대해 생성된 상수를 포함하여, 시맨틱 컨벤션
준수를 돕는 [아티팩트를 게시한다](../api/#semantic-attributes).

## 로그 계측 {#log-instrumentation}

[LoggerProvider](../api/#loggerprovider)/[Logger](../api/#logger) API는
구조적으로 대응하는 [트레이스](../api/#tracerprovider) 및
[메트릭](../api/#meterprovider) API와 유사하지만, 사용 사례는 다르다. 현재
`LoggerProvider`/`Logger` 및 관련 클래스는
[로그 브리지(Log Bridge) API](/docs/specs/otel/logs/api/)를 나타내며, 이는 다른
로그 API/프레임워크로 기록된 로그를 오픈텔레메트리로 연결하는 로그
어펜더(appender)를 작성하기 위해 존재한다. 이들은 Log4j/SLF4J/Logback 등을
대체하는 최종 사용자용 API로 사용하기 위한 것이 아니다.

오픈텔레메트리에서 로그 계측을 사용하는 데는 서로 다른 애플리케이션 요구 사항에
맞는 두 가지 일반적인 워크플로가 있다.

### 컬렉터로 직접 전송 {#direct-to-collector}

컬렉터로 직접 전송하는 워크플로에서는, 네트워크 프로토콜(예: OTLP)을 사용해
애플리케이션에서 컬렉터로 로그가 직접 방출된다. 이 워크플로는 추가적인 로그
포워딩 구성 요소가 필요하지 않아 설정이 간단하며, 애플리케이션이
[로그 데이터 모델](/docs/specs/otel/logs/data-model/)을 준수하는 구조화된 로그를
쉽게 방출할 수 있게 한다. 하지만 애플리케이션이 로그를 큐에 넣고 네트워크 위치로
내보내는 데 필요한 오버헤드는 모든 애플리케이션에 적합하지 않을 수 있다.

이 워크플로를 사용하려면 다음을 수행한다.

- 적절한 로그 어펜더를 설치한다. **[1]**
- 로그 레코드를 원하는 대상
  목적지([컬렉터](https://github.com/open-telemetry/opentelemetry-collector)
  등)로 내보내도록 오픈텔레메트리 [로그 SDK](../sdk/#sdkloggerprovider)를
  구성한다.

**[1]**: 로그 어펜더는 로그 프레임워크의 로그를 오픈텔레메트리 로그 SDK로
연결하는 [심](#shims)의 한 종류이다. "Log4j를 오픈텔레메트리로 연결", "Logback을
오픈텔레메트리로 연결" 항목을 참고한다. 다양한 시나리오에 대한 시연은
[로그 어펜더 예제](https://github.com/open-telemetry/opentelemetry-java-docs/tree/main/log-appender)를
참고한다.

### 파일 또는 표준 출력을 통한 방식 {#via-file-or-stdout}

파일 또는 표준 출력 워크플로에서는 로그가 파일이나 표준 출력에 기록된다. 다른
구성 요소(예: FluentBit)가 로그를 읽거나 테일링하고, 더 구조화된 형식으로
파싱하여 컬렉터와 같은 대상으로 전달하는 역할을 담당한다. 이 워크플로는
애플리케이션 요구 사항이 [컬렉터로 직접 전송](#direct-to-collector)에 따른 추가
오버헤드를 허용하지 않는 상황에서 선호될 수 있다. 하지만 다운스트림에서 필요한
모든 로그 필드가 로그에 인코딩되어 있어야 하고, 로그를 읽는 구성 요소가 데이터를
[로그 데이터 모델](/docs/specs/otel/logs/data-model)로 파싱해야 한다. 로그
포워딩 구성 요소의 설치 및 구성은 이 문서의 범위를 벗어난다.

트레이스와의 로그 상관관계는 오픈텔레메트리 컨텍스트를 로그 프레임워크로
연결하는 [심](#shims)을 설치하여 사용할 수 있다. "오픈텔레메트리 컨텍스트를
Log4j로 연결", "오픈텔레메트리 컨텍스트를 Logback으로 연결" 항목을 참고한다.

> [!NOTE]
>
> 표준 출력을 사용하는 로그 계측의 엔드투엔드 예제는
> [Java 예제 저장소](https://github.com/open-telemetry/opentelemetry-java-examples/blob/main/logging-k8s-stdout-otlp-json/README.md)에서
> 확인할 수 있다.
