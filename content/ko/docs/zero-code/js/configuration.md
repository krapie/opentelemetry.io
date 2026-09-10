---
title: 제로 코드 계측 구성
linkTitle: 구성
description: Node.js를 위한 제로 코드 계측을 구성하는 방법을 알아본다.
aliases:
  - /docs/languages/js/automatic/configuration
  - /docs/languages/js/automatic/module-config
weight: 10
cSpell:ignore: serviceinstance
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

이 모듈은
[환경 변수](/docs/specs/otel/configuration/sdk-environment-variables/)를
설정하여 매우 유연하게 구성할 수 있다. 리소스 감지기(resource detectors),
익스포터(exporter), 트레이스 컨텍스트 전파 헤더 등 자동 계측 동작의 여러 측면을
필요에 맞게 구성할 수 있다.

## SDK 및 익스포터 구성 {#sdk-and-exporter-configuration}

[SDK 및 익스포터 구성](/docs/languages/sdk-configuration/)은 환경 변수로 설정할
수 있다.

## SDK 리소스 감지기 구성 {#sdk-resource-detector-configuration}

기본적으로 이 모듈은 모든 SDK 리소스 감지기를 활성화한다.
`OTEL_NODE_RESOURCE_DETECTORS` 환경 변수를 사용하여 특정 감지기만 활성화하거나
전체를 비활성화할 수 있다.

- `env`
- `host`
- `os`
- `process`
- `serviceinstance`
- `container`
- `alibaba`
- `aws`
- `azure`
- `gcp`
- `all` - 모든 리소스 감지기를 활성화한다
- `none` - 리소스 감지를 비활성화한다

예를 들어, `env`와 `host` 감지기만 활성화하려면 다음과 같이 설정한다.

```shell
OTEL_NODE_RESOURCE_DETECTORS=env,host
```

## 계측 라이브러리 제외 {#excluding-instrumentation-libraries}

기본적으로 모든
[지원되는 계측 라이브러리](https://github.com/open-telemetry/opentelemetry-js-contrib/blob/main/packages/auto-instrumentations-node/README.md#supported-instrumentations)가
활성화되어 있지만, 환경 변수를 사용하여 특정 계측을 활성화하거나 비활성화할 수
있다.

### 특정 계측 활성화 {#enable-specific-instrumentations}

`OTEL_NODE_ENABLED_INSTRUMENTATIONS` 환경 변수를 사용하여,
`@opentelemetry/instrumentation-` 접두사를 뺀 계측 라이브러리 이름을 쉼표로
구분한 목록으로 제공함으로써 특정 계측만 활성화한다.

예를 들어,
[@opentelemetry/instrumentation-http](https://github.com/open-telemetry/opentelemetry-js/tree/main/experimental/packages/opentelemetry-instrumentation-http)와
[@opentelemetry/instrumentation-express](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/instrumentation-express)
계측만 활성화하려면 다음과 같이 한다.

```shell
OTEL_NODE_ENABLED_INSTRUMENTATIONS="http,express"
```

### 특정 계측 비활성화 {#disable-specific-instrumentations}

`OTEL_NODE_DISABLED_INSTRUMENTATIONS` 환경 변수를 사용하여,
`@opentelemetry/instrumentation-` 접두사를 뺀 계측 라이브러리 이름을 쉼표로
구분한 목록으로 제공함으로써 완전히 활성화된 목록을 유지하면서 특정 계측만
비활성화한다.

예를 들어,
[@opentelemetry/instrumentation-fs](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/instrumentation-fs)와
[@opentelemetry/instrumentation-grpc](https://github.com/open-telemetry/opentelemetry-js/tree/main/experimental/packages/opentelemetry-instrumentation-grpc)
계측만 비활성화하려면 다음과 같이 한다.

```shell
OTEL_NODE_DISABLED_INSTRUMENTATIONS="fs,grpc"
```

> [!NOTE]
>
> 두 환경 변수가 모두 설정된 경우, `OTEL_NODE_ENABLED_INSTRUMENTATIONS`가 먼저
> 적용된 다음, 그 목록에 `OTEL_NODE_DISABLED_INSTRUMENTATIONS`가 적용된다.
> 따라서 동일한 계측이 두 목록에 모두 포함되어 있으면, 그 계측은 비활성화된다.
