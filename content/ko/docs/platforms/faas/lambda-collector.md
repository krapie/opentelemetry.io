---
title: Lambda 컬렉터 구성
linkTitle: Lambda 컬렉터 구성
weight: 11
description: Lambda에 컬렉터 Lambda 레이어를 추가하고 구성한다.
cSpell:ignore: ADOT awsxray configmap confmap regionalized
default_lang_commit: f49ec57e5a0ec766b07c7c8e8974c83531620af3
---

오픈텔레메트리 커뮤니티는 사용자에게 최대한의 유연성을 제공하기 위해 컬렉터를
계측 레이어와는 별도의 Lambda 레이어로 제공한다. 이는 계측과 컬렉터를 함께
번들로 제공하는 현재의 AWS Distribution of OpenTelemetry(ADOT) 구현과는 다른
방식이다.

## OTel 컬렉터 Lambda 레이어의 ARN 추가 {#add-the-arn-of-the-otel-collector-lambda-layer}

애플리케이션을 계측했다면, 데이터를 수집해 선택한 백엔드로 제출하기 위해 컬렉터
Lambda 레이어를 추가해야 한다.

[최신 컬렉터 레이어 릴리스](https://github.com/open-telemetry/opentelemetry-lambda/releases)를
찾아 `<region>` 태그를 Lambda가 위치한 리전으로 변경한 후 해당 ARN을 사용한다.

참고: Lambda 레이어는 리전화된(regionalized) 리소스이므로, 게시된 리전에서만
사용할 수 있다. Lambda 함수와 동일한 리전의 레이어를 사용해야 한다. 커뮤니티는
사용 가능한 모든 리전에 레이어를 게시한다.

## OTel 컬렉터 구성 {#configure-the-otel-collector}

OTel 컬렉터 Lambda 레이어의 구성은 오픈텔레메트리 표준을 따른다.

기본적으로 OTel 컬렉터 Lambda 레이어는 config.yaml을 사용한다.

### 선호하는 백엔드에 대한 환경 변수 설정 {#set-the-environment-variable-for-your-preferred-backend}

Lambda 환경 변수 설정에서 인증 토큰을 담을 새 변수를 생성한다.

### 기본 익스포터 업데이트 {#update-the-default-exporters}

`config.yaml` 파일에 선호하는 익스포터가 아직 없다면 추가한다. 이전 단계에서
액세스 토큰을 위해 설정한 환경 변수를 사용해 익스포터를 구성한다.

**익스포터에 대한 환경 변수가 설정되지 않은 경우, 기본 구성은 debug 익스포터를
사용한 데이터 방출만 지원한다.** 다음은 기본 구성이다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: '0.0.0.0:4317'
      http:
        endpoint: '0.0.0.0:4318'

exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
  telemetry:
    metrics:
      address: localhost:8888
```

## Lambda 게시 {#publish-your-lambda}

변경 사항을 활성화하려면 Lambda의 새 버전을 게시한다.

## 고급 OTel 컬렉터 구성 {#advanced-otel-collector-configuration}

커스텀 구성에 사용할 수 있는 컴포넌트 목록은 여기에서 확인할 수 있다. 디버깅을
활성화하려면 구성 파일을 사용해 로그 레벨을 debug로 설정하면 된다. 아래 예제를
참고한다.

### 선호하는 Confmap 프로바이더 선택 {#choose-your-preferred-confmap-provider}

OTel Lambda 레이어는 `file`, `env`, `yaml`, `http`, `https`, `s3` 유형의 confmap
프로바이더를 지원한다. 서로 다른 Confmap 프로바이더를 사용해 OTel 컬렉터 구성을
커스터마이징하려면 자세한 내용은
[Amazon Distribution of OpenTelemetry Confmap 프로바이더 문서](https://aws-otel.github.io/docs/components/confmap-providers#confmap-providers-supported-by-the-adot-collector)를
참고한다.

### 커스텀 구성 파일 생성 {#create-a-custom-configuration-file}

다음은 루트 디렉터리에 있는 `collector.yaml` 구성 파일의 예제이다.

```yaml
#collector.yaml in the root directory
#Set an environment variable 'OPENTELEMETRY_COLLECTOR_CONFIG_URI' to '/var/task/collector.yaml'

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 'localhost:4317'
      http:
        endpoint: 'localhost:4318'

exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug:
  awsxray:

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      exporters: [debug]
  telemetry:
    metrics:
      address: localhost:8888
```

### 환경 변수를 사용해 커스텀 구성 파일 매핑 {#map-your-custom-configuration-file-using-environment-variables}

Confmap 프로바이더를 통해 컬렉터 구성을 설정했다면, Lambda 함수에
`OPENTELEMETRY_COLLECTOR_CONFIG_URI` 환경 변수를 생성하고 해당 Confmap
프로바이더 기준의 구성 경로를 값으로 설정한다. 예를 들어 file confmap
프로바이더를 사용한다면 값을 `/var/task/<path>/<to>/<filename>`으로 설정한다.
이렇게 하면 익스텐션이 컬렉터 구성을 찾을 위치를 알 수 있다.

#### CLI를 사용한 커스텀 컬렉터 구성 {#custom-collector-configuration-using-the-cli}

Lambda 콘솔이나 AWS CLI를 통해 이를 설정할 수 있다.

```bash
aws lambda update-function-configuration --function-name Function --environment Variables={OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/collector.yaml}
```

#### CloudFormation에서 구성 환경 변수 설정 {#set-configuration-environment-variables-from-cloudformation}

**CloudFormation** 템플릿을 통해서도 환경 변수를 구성할 수 있다.

```yaml
Function:
  Type: AWS::Serverless::Function
  Properties:
    ...
    Environment:
      Variables:
        OPENTELEMETRY_COLLECTOR_CONFIG_URI: /var/task/collector.yaml
```

#### S3 오브젝트에서 구성 로드 {#load-configuration-from-an-s3-object}

S3에서 구성을 로드하려면 함수에 연결된 IAM 역할에 해당 버킷에 대한 읽기 권한이
포함되어 있어야 한다.

```yaml
Function:
  Type: AWS::Serverless::Function
  Properties:
    ...
    Environment:
      Variables:
        OPENTELEMETRY_COLLECTOR_CONFIG_URI: s3://<bucket_name>.s3.<region>.amazonaws.com/collector_config.yaml
```
