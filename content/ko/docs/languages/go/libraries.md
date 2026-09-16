---
title: 계측 라이브러리 사용하기
linkTitle: 라이브러리
aliases:
  - /docs/languages/go/using_instrumentation_libraries
  - /docs/languages/go/automatic_instrumentation
weight: 40
default_lang_commit: 825010e3cfece195ae4dfd019eff080ef8eb6365
---

{{% docs/languages/libraries-intro "go" %}}

## 계측 라이브러리 사용하기 {#use-instrumentation-libraries}

라이브러리가 기본적으로 오픈텔레메트리(OpenTelemetry)를 지원하지 않는다면,
[계측 라이브러리](/docs/specs/otel/glossary/#instrumentation-library)를 사용해
해당 라이브러리나 프레임워크의 텔레메트리 데이터를 생성할 수 있다.

예를 들어
[`net/http`용 계측 라이브러리](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp)는
HTTP 요청을 기반으로 [스팬](/docs/concepts/signals/traces/#spans)과
[메트릭](/docs/concepts/signals/metrics/)을 자동으로 생성한다.

## 설정 {#setup}

각 계측 라이브러리는 패키지이다. 일반적으로 이는 알맞은 패키지를 `go get`해야
함을 의미한다. 예를 들어
[Contrib 저장소](https://github.com/open-telemetry/opentelemetry-go-contrib)에서
관리되는 계측 라이브러리를 받으려면 다음을 실행한다.

```sh
go get go.opentelemetry.io/contrib/instrumentation/{import-path}/otel{package-name}
```

그런 다음 라이브러리가 활성화를 위해 요구하는 사항에 따라 코드에서 이를
구성한다.

[시작하기](../getting-started/)에서 `net/http` 서버에 대한 계측을 설정하는
방법을 보여주는 예제를 제공한다.

## 사용 가능한 패키지 {#available-packages}

사용 가능한 계측 라이브러리의 전체 목록은
[오픈텔레메트리 레지스트리](/ecosystem/registry/?language=go&component=instrumentation)에서
찾을 수 있다.

## 다음 단계 {#next-steps}

계측 라이브러리는 인바운드 및 아웃바운드 HTTP 요청에 대한 텔레메트리 데이터를
생성하는 등의 작업을 할 수 있지만, 실제 애플리케이션을 계측하지는 않는다.

코드에 [커스텀 계측](../instrumentation/)을 통합하여 텔레메트리 데이터를
보강한다. 이는 표준 라이브러리의 텔레메트리를 보완하며, 실행 중인 애플리케이션에
대한 더 깊은 통찰을 제공할 수 있다.
