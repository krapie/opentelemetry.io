---
title: 구성
weight: 10
aliases: [agent-config]
cSpell:ignore: classloaders customizer
default_lang_commit: 5d68eec62bc16a5558678eae6d2f8c5083113823
---

> [!NOTE] 자세한 정보
>
> 이 페이지에서는 Java 에이전트에 구성을 제공하는 다양한 방법을 설명한다. 구성
> 옵션 자체에 대한 정보는 [SDK 구성하기](/docs/languages/java/configuration)를
> 참고한다.

## 에이전트 구성 {#agent-configuration}

에이전트는 다음 소스 중 하나 이상에서 구성을 읽을 수 있다(우선순위가 높은 것부터
낮은 순).

- 시스템 속성
- [환경 변수](#configuring-with-environment-variables)
- [구성 파일](#configuration-file)
- [`AutoConfigurationCustomizerProvider`](https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk-extensions/autoconfigure-spi/src/main/java/io/opentelemetry/sdk/autoconfigure/spi/AutoConfigurationCustomizerProvider.java)
  SPI를 사용하는
  [`AutoConfigurationCustomizer#addPropertiesSupplier()`](https://github.com/open-telemetry/opentelemetry-java/blob/f92e02e4caffab0d964c02a32fe305d6d6ba372e/sdk-extensions/autoconfigure-spi/src/main/java/io/opentelemetry/sdk/autoconfigure/spi/AutoConfigurationCustomizer.java#L73)
  함수가 제공하는 속성

## 환경 변수로 구성하기 {#configuring-with-environment-variables}

특정 환경에서는 환경 변수로 설정을 구성하는 것이 선호되는 경우가 많다. 시스템
속성으로 구성할 수 있는 설정은 환경 변수로도 설정할 수 있다. 아래의 여러 설정은
두 형식에 대한 예시를 모두 제공하지만, 그렇지 않은 경우에는 다음 단계를 사용해
원하는 시스템 속성에 대한 올바른 이름 매핑을 확인한다.

- 시스템 속성 이름을 대문자로 변환한다.
- 모든 `.` 및 `-` 문자를 `_`로 바꾼다.

예를 들어 `otel.instrumentation.common.default-enabled`는
`OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED`로 변환된다.

## 구성 파일 {#configuration-file}

다음 속성을 설정하여 에이전트 구성 파일의 경로를 제공할 수 있다.

{{% config_option name="otel.javaagent.configuration-file" %}} 에이전트 구성이
담긴 유효한 Java 속성 파일의 경로이다. {{% /config_option %}}

## 익스텐션 {#extensions}

다음 속성을 설정하여 [익스텐션][extensions]을 활성화할 수 있다.

{{% config_option name="otel.javaagent.extensions" %}}

익스텐션 jar 파일 또는 jar 파일이 담긴 폴더의 경로이다. 폴더를 가리키는 경우,
해당 폴더의 모든 jar 파일이 각각 별개의 독립적인 익스텐션으로 취급된다.

{{% /config_option %}}

## Java 에이전트 로깅 출력 {#java-agent-logging-output}

다음 속성을 설정하여 에이전트의 로깅 출력을 구성할 수 있다.

{{% config_option name="otel.javaagent.logging" %}}

Java 에이전트 로깅 모드이다. 다음 3가지 모드가 지원된다.

- `simple`: 에이전트가 표준 오류 스트림으로 로그를 출력한다. `INFO` 이상 로그만
  출력된다. 이것이 기본 Java 에이전트 로깅 모드이다.
- `none`: 에이전트가 아무것도 로깅하지 않으며, 자신의 버전조차 출력하지 않는다.
- `application`: 에이전트가 자신의 로그를 계측 대상 애플리케이션의 slf4j 로거로
  리다이렉트하려고 시도한다. 이는 여러 클래스로더를 사용하지 않는 단순한 단일
  jar 애플리케이션에 가장 잘 맞으며, Spring Boot 앱도 지원된다. Java 에이전트
  출력 로그는 계측 대상 애플리케이션의 로깅 구성(예: `logback.xml` 또는
  `log4j2.xml`)을 통해 추가로 구성할 수 있다. **프로덕션 환경에서 실행하기 전에
  이 모드가 애플리케이션에서 동작하는지 반드시 테스트한다.**

{{% /config_option %}}

## SDK 구성 {#sdk-configuration}

에이전트의 기본 구성에는 SDK의 자동 구성(autoconfiguration) 모듈이 사용된다.
익스포트나 샘플링 구성과 같은 설정을 찾으려면
[문서](/docs/languages/java/configuration)를 참고한다.

> [!IMPORTANT]
>
> SDK 자동 구성과 달리, Java 에이전트 및 오픈텔레메트리(OpenTelemetry) Spring
> Boot 스타터 2.0+ 버전은 기본 프로토콜로 `grpc`가 아닌 `http/protobuf`를
> 사용한다.

## 기본적으로 비활성화된 리소스 프로바이더 활성화하기 {#enable-resource-providers-that-are-disabled-by-default}

SDK 자동 구성의 리소스 구성 외에도, 기본적으로 비활성화되어 있는 추가 리소스
프로바이더(resource provider)를 활성화할 수 있다.

{{% config_option
name="otel.resource.providers.aws.enabled"
default=false
%}} [AWS 리소스 프로바이더](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/aws-resources)를
활성화한다. {{% /config_option %}}

{{% config_option
name="otel.resource.providers.gcp.enabled"
default=false
%}} [GCP 리소스 프로바이더](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/gcp-resources)를
활성화한다. {{% /config_option %}}

{{% config_option
name="otel.resource.providers.azure.enabled"
default=false
%}} [Azure 리소스 프로바이더](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/azure-resources)를
활성화한다. {{% /config_option %}}

[extensions]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/examples/extension#readme
