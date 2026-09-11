---
title: Lambda 자동 계측
weight: 11
description: 오픈텔레메트리(OpenTelemetry)로 Lambda를 자동으로 계측한다.
cSpell:ignore: Corretto regionalized
default_lang_commit: 2930608f29463e76d08a496239c05ed75b20120e
---

오픈텔레메트리 커뮤니티는 다음 언어를 위한 독립 실행형(standalone) 계측 Lambda
레이어를 제공한다.

- Java
- JavaScript
- Python
- Ruby

이 레이어는 AWS 포털을 사용해 Lambda에 추가하여 애플리케이션을 자동으로 계측할
수 있다. 이 레이어에는 컬렉터가 포함되어 있지 않으므로, 데이터를 전송할 외부
컬렉터 인스턴스를 구성하지 않는 한 컬렉터를 별도로 추가해야 한다.

## OTel 컬렉터 Lambda 레이어의 ARN 추가 {#add-the-arn-of-the-otel-collector-lambda-layer}

애플리케이션에 레이어를 추가하고 컬렉터를 구성하려면
[컬렉터 Lambda 레이어 가이드](../lambda-collector/)를 참고한다. 이 작업을 먼저
진행하는 것을 권장한다.

## 언어 요구 사항 {#language-requirements}

{{< tabpane text=true >}} {{% tab Java %}}

이 Lambda 레이어는 Java 8, 11, 17(Corretto) Lambda 런타임을 지원한다. 지원되는
Java 버전에 대한 자세한 정보는
[오픈텔레메트리 Java 문서](/docs/languages/java/)를 참고한다.

**참고:** Java 자동 계측 에이전트는 이 Lambda 레이어에 포함되어 있다. 자동
계측은 AWS Lambda의 시작 시간에 상당한 영향을 미치므로, 초기화 중 첫 요청에서
타임아웃이 발생하지 않고 프로덕션 요청을 처리하려면 일반적으로 프로비저닝된
동시성(provisioned concurrency)과 워밍업(warmup) 요청을 함께 사용해야 한다.

기본적으로 레이어의 OTel Java 에이전트는 애플리케이션의 모든 코드를 자동으로
계측하려고 시도한다. 이는 Lambda의 콜드 스타트(cold startup) 시간에 부정적인
영향을 미칠 수 있다.

애플리케이션에서 실제로 사용하는 라이브러리/프레임워크에 대해서만 자동 계측을
활성화하는 것을 권장한다.

특정 계측만 활성화하려면 다음 환경 변수를 사용할 수 있다.

- `OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED`: false로 설정하면 레이어의 자동
  계측이 비활성화되며, 각 계측을 개별적으로 활성화해야 한다.
- `OTEL_INSTRUMENTATION_<NAME>_ENABLED`: true로 설정하면 특정 라이브러리나
  프레임워크에 대한 자동 계측이 활성화된다. `<NAME>`을 활성화하려는 계측
  이름으로 바꾼다. 사용 가능한 계측 목록은 [특정 에이전트 계측 억제하기][1]를
  참고한다.

  [1]:
    /docs/zero-code/java/agent/disable/#suppressing-specific-agent-instrumentation

예를 들어, Lambda와 AWS SDK에 대해서만 자동 계측을 활성화하려면 다음 환경 변수를
설정한다.

```sh
OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false
OTEL_INSTRUMENTATION_AWS_LAMBDA_ENABLED=true
OTEL_INSTRUMENTATION_AWS_SDK_ENABLED=true
```

{{% /tab %}} {{% tab JavaScript %}}

이 Lambda 레이어는 Node.js v18 이상 Lambda 런타임을 지원한다. 지원되는
JavaScript 및 Node.js 버전에 대한 자세한 정보는
[오픈텔레메트리 JavaScript 문서](https://github.com/open-telemetry/opentelemetry-js)를
참고한다.

{{% /tab %}} {{% tab Python %}}

이 Lambda 레이어는 Python 3.9 이상 Lambda 런타임을 지원한다. 지원되는 Python
버전에 대한 자세한 정보는
[오픈텔레메트리 Python 문서](https://github.com/open-telemetry/opentelemetry-python/blob/main/README.md#python-version-support)와
[PyPi](https://pypi.org/project/opentelemetry-api/)의 패키지를 참고한다.

{{% /tab %}} {{% tab Ruby %}}

이 Lambda 레이어는 Ruby 3.2 및 3.3 Lambda 런타임을 지원한다. 지원되는
오픈텔레메트리 Ruby SDK 및 API 버전에 대한 자세한 정보는
[오픈텔레메트리 Ruby 문서](https://github.com/open-telemetry/opentelemetry-ruby/blob/main/README.md#compatibility)와
[RubyGem](https://rubygems.org/search?query=opentelemetry)의 패키지를 참고한다.

{{% /tab %}} {{< /tabpane >}}

## `AWS_LAMBDA_EXEC_WRAPPER` 구성 {#configure-aws_lambda_exec_wrapper}

Node.js, Java, Ruby, Python의 경우 `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-handler`를
설정해 애플리케이션의 진입점(entry point)을 변경한다. 이 래퍼 스크립트는 자동
계측이 적용된 상태로 Lambda 애플리케이션을 호출한다.

## 계측 Lambda 레이어의 ARN 추가 {#add-the-arn-of-instrumentation-lambda-layer}

Lambda 함수에서 OTel 자동 계측을 활성화하려면 계측 레이어와 컬렉터 레이어를 추가
및 구성한 다음 트레이싱을 활성화해야 한다.

1. AWS 콘솔에서 계측하려는 Lambda 함수를 연다.
2. Designer의 Layers 섹션에서 Add a layer를 선택한다.
3. Specify an ARN에서 레이어 ARN을 붙여넣은 다음 Add를 선택한다.

사용하는 언어에 맞는
[최신 계측 레이어 릴리스](https://github.com/open-telemetry/opentelemetry-lambda/releases)를
찾아 `<region>` 태그를 Lambda가 위치한 리전으로 변경한 후 해당 ARN을 사용한다.

참고: Lambda 레이어는 리전화된(regionalized) 리소스이므로, 게시된 리전에서만
사용할 수 있다. Lambda 함수와 동일한 리전의 레이어를 사용해야 한다. 커뮤니티는
사용 가능한 모든 리전에 레이어를 게시한다.

## SDK 익스포터 구성 {#configure-your-sdk-exporters}

Lambda 레이어가 사용하는 기본 익스포터는 gRPC/HTTP 리시버가 내장된 컬렉터가
있다면 변경 없이 동작한다. 환경 변수를 업데이트할 필요는 없다. 다만 언어별로
프로토콜 지원 수준과 기본값이 다르며, 아래에 문서화되어 있다.

{{< tabpane text=true >}} {{% tab Java %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=grpc`는 `grpc`, `http/protobuf`, `http/json`을
지원한다. `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`

{{% /tab %}} {{% tab JavaScript %}}

`OTEL_EXPORTER_OTLP_PROTOCOL` 환경 변수는 지원되지 않는다. 하드코딩된 익스포터는
`http/protobuf` 프로토콜을 사용한다.
`OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{% tab Python %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`는 `http/protobuf`와 `http/json`을
지원한다. `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{% tab Ruby %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`는 `http/protobuf`를 지원한다.
`OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{< /tabpane >}}

## Lambda 게시 {#publish-your-lambda}

새로운 변경 사항과 계측을 배포하려면 Lambda의 새 버전을 게시한다.
