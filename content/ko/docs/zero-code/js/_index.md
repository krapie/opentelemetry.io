---
title: JavaScript 제로 코드 계측
linkTitle: JavaScript
description:
  소스 코드를 전혀 수정하지 않고 애플리케이션에서 텔레메트리를 캡처하는 방법을
  알아본다.
aliases: [/docs/languages/js/automatic]
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

JavaScript를 위한 제로 코드 계측(Zero-code instrumentation)은 코드 변경 없이
모든 Node.js 애플리케이션을 계측하고 다양한 인기 라이브러리 및 프레임워크로부터
텔레메트리 데이터를 수집(collect)할 수 있는 방법을 제공한다.

## 설정 {#setup}

적절한 패키지를 설치하려면 다음 명령을 실행한다.

```shell
npm install --save @opentelemetry/api
npm install --save @opentelemetry/auto-instrumentations-node
```

`@opentelemetry/api` 및 `@opentelemetry/auto-instrumentations-node` 패키지는
API, SDK, 계측 도구를 설치한다.

## 모듈 구성하기 {#configuring-the-module}

이 모듈은 매우 유연하게 구성할 수 있다.

한 가지 방법은 CLI에서 `env`를 사용하여 환경 변수를 설정함으로써 모듈을 구성하는
것이다.

```shell
env OTEL_TRACES_EXPORTER=otlp OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=your-endpoint \
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

또는 `export`를 사용하여 환경 변수를 설정할 수도 있다.

```shell
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_EXPORTER_OTLP_ENDPOINT="your-endpoint"
export OTEL_NODE_RESOURCE_DETECTORS="env,host,os"
export OTEL_SERVICE_NAME="your-service-name"
export NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"
node app.js
```

기본적으로 모든 SDK
[리소스 감지기(resource detectors)](/docs/languages/js/resources/)가 사용된다.
특정 감지기만 활성화하거나 전체를 비활성화하려면 환경 변수
`OTEL_NODE_RESOURCE_DETECTORS`를 사용할 수 있다.

전체 구성 옵션을 확인하려면 [모듈 구성](configuration)을 참고한다.

## 지원되는 라이브러리 및 프레임워크 {#supported-libraries-and-frameworks}

여러 인기 Node.js 라이브러리가 자동으로 계측된다. 전체 목록은
[지원되는 계측](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/auto-instrumentations-node#supported-instrumentations)을
참고한다.

## 문제 해결 {#troubleshooting}

다음 값 중 하나로 `OTEL_LOG_LEVEL` 환경 변수를 설정하여 로그 레벨을 지정할 수
있다.

- `none`
- `error`
- `warn`
- `info`
- `debug`
- `verbose`
- `all`

기본 레벨은 `info`이다.

> [!NOTE]
>
> - 프로덕션 환경에서는 `OTEL_LOG_LEVEL`을 `info`로 설정하는 것을 권장한다.
> - 로그는 환경이나 디버그 레벨과 관계없이 항상 `console`로 전송된다.
> - 디버그 로그는 매우 상세하며 애플리케이션 성능에 부정적인 영향을 미칠 수
>   있다. 필요할 때만 디버그 로깅을 활성화한다.
