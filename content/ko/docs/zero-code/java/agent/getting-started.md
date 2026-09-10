---
title: 시작하기
weight: 1
cSpell:ignore: Dotel
default_lang_commit: 5d68eec62bc16a5558678eae6d2f8c5083113823
---

## 설정 {#setup}

1.  `opentelemetry-java-instrumentation` 저장소의 [Releases][]에서
    [opentelemetry-javaagent.jar][]를 다운로드하여 원하는 디렉터리에 둔다. 이
    JAR 파일에는 에이전트와 계측 라이브러리가 들어 있다.
2.  JVM 시작 인자에 `-javaagent:path/to/opentelemetry-javaagent.jar`와 기타
    구성을 추가하고 애플리케이션을 실행한다.
    - 시작 명령에 직접 지정:

      ```shell
      java -javaagent:path/to/opentelemetry-javaagent.jar -Dotel.service.name=your-service-name -jar myapp.jar
      ```

    - `JAVA_TOOL_OPTIONS`와 기타 환경 변수를 통해 지정:

      ```shell
      export JAVA_TOOL_OPTIONS="-javaagent:path/to/opentelemetry-javaagent.jar"
      export OTEL_SERVICE_NAME="your-service-name"
      java -jar myapp.jar
      ```

## 선언적 구성(Declarative configuration) {#declarative-configuration}

선언적 구성은 환경 변수나 시스템 속성 대신 YAML 파일을 사용한다. 설정할 구성
옵션이 많거나, 환경 변수나 시스템 속성으로는 사용할 수 없는 구성 옵션을 쓰고
싶을 때 유용하다.

자세한 내용은 [선언적 구성](../declarative-configuration) 페이지를 참고한다.

## 에이전트 구성하기 {#configuring-the-agent}

에이전트는 매우 다양하게 구성할 수 있다.

한 가지 방법은 `-D` 플래그로 구성 속성을 전달하는 것이다. 다음 예시에서는 서비스
이름과 트레이스용 Zipkin 익스포터를 구성한다.

```sh
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.service.name=your-service-name \
     -Dotel.traces.exporter=zipkin \
     -jar myapp.jar
```

환경 변수를 사용하여 에이전트를 구성할 수도 있다.

```sh
OTEL_SERVICE_NAME=your-service-name \
OTEL_TRACES_EXPORTER=zipkin \
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -jar myapp.jar
```

Java 속성 파일을 제공하고 그곳에서 구성 값을 불러올 수도 있다.

```sh
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.configuration-file=path/to/properties/file.properties \
     -jar myapp.jar
```

또는

```sh
OTEL_JAVAAGENT_CONFIGURATION_FILE=path/to/properties/file.properties \
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -jar myapp.jar
```

전체 구성 옵션을 보려면 [에이전트 구성](../configuration)을 참고한다.

## 지원되는 라이브러리, 프레임워크, 애플리케이션 서비스 및 JVM {#supported-libraries-frameworks-application-services-and-jvms}

Java 에이전트에는 많은 인기 컴포넌트를 위한 계측 라이브러리가 함께 제공된다.
전체 목록은 [지원되는 라이브러리, 프레임워크, 애플리케이션 서비스 및
JVM][support]을 참고한다.

## 문제 해결 {#troubleshooting}

{{% config_option name="otel.javaagent.debug" %}}

디버그 로그를 보려면 `true`로 설정한다. 로그가 상당히 장황하다는 점에 유의한다.

{{% /config_option %}}

## 다음 단계 {#next-steps}

애플리케이션이나 서비스에 자동 계측을 구성했다면, 선택한 메서드에
[어노테이션](../annotations)을 추가하거나
[수동 계측](/docs/languages/java/instrumentation/)을 추가하여 커스텀 텔레메트리
데이터를 수집할 수 있다.

[opentelemetry-javaagent.jar]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
[releases]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases
[support]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md
