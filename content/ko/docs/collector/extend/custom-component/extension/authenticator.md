---
title: 인증자 익스텐션 빌드하기
linkTitle: 인증자
weight: 100
aliases:
  - /docs/collector/custom-auth
  - /docs/collector/building/authenticator-extension
cSpell:ignore: configauth oidc
default_lang_commit: f304b22c356b4ad4046b8ce5550ddf503a510d69
---

오픈텔레메트리(OpenTelemetry) 컬렉터를 사용하면 리시버와 익스포터를
인증자(authenticator)에 연결하여, 리시버 측에서는 들어오는 연결을 인증하고
익스포터 측에서는 나가는 요청에 인증 데이터를 추가할 수 있다.

인증자는 [익스텐션][extensions]을 통해 구현된다. 이 문서에서는 자체 인증자를
구현하는 방법을 안내한다. 기존 인증자를 사용하는 방법을 알아보려면 해당 인증자의
문서를 참고한다. 기존 인증자 목록은 이 웹사이트의
[레지스트리](/ecosystem/registry/)에서 확인할 수 있다.

이 가이드를 참고하여 커스텀 인증자를 빌드하는 방법에 대한 일반적인 방향을
확인하고, 각 타입과 함수의 시맨틱(semantics)에 대해서는
[API 참조 가이드](https://pkg.go.dev/go.opentelemetry.io/collector/config/configauth)를
참고한다.

도움이 필요하다면 [CNCF Slack 워크스페이스](https://slack.cncf.io)의
[#opentelemetry-collector-dev](https://cloud-native.slack.com/archives/C07CCCMRXBK)
채널에 참여한다.

## 아키텍처 {#architecture}

오픈텔레메트리의 [인증자][Authenticators]는 다른 익스텐션과 마찬가지이지만,
인증이 수행되는 방식(예: HTTP 또는 gRPC 요청 인증)을 정의하는 하나 이상의 특정
인터페이스도 구현해야 한다. HTTP 및 gRPC 요청을 가로채려면 리시버와 함께 [서버
인증자][sa]를 사용한다. HTTP 및 gRPC 요청에 인증 데이터를 추가하려면 익스포터와
함께 클라이언트 인증자를 사용한다. 인증자는 두 인터페이스를 동시에 구현할 수도
있으며, 이 경우 익스텐션의 단일 인스턴스가 들어오는 요청과 나가는 요청을 모두
처리할 수 있다.

컬렉터 배포판에서 인증자 익스텐션을 사용할 수 있게 되면, 다른 익스텐션과 동일한
방식으로 구성 파일에서 참조할 수 있다. 다만, 인증자는 이를 사용하는 구성
요소에서 참조될 때만 효과가 있다. 다음 구성은 `oidc` 인증자 익스텐션을 사용하는
`otlp/auth`라는 이름의 리시버를 보여준다.

```yaml
extensions:
  oidc:

receivers:
  otlp/auth:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        auth:
          authenticator: oidc

processors:
exporters:

service:
  extensions:
    - oidc
  pipelines:
    traces:
      receivers:
        - otlp/auth
      processors: []
      exporters: []
```

인증자의 여러 인스턴스가 필요하다면, 서로 다른 이름을 지정한다.

```yaml
extensions:
  oidc/some-provider:
  oidc/another-provider:

receivers:
  otlp/auth:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        auth:
          authenticator: oidc/some-provider

processors:
exporters:

service:
  extensions:
    - oidc/some-provider
    - oidc/another-provider
  pipelines:
    traces:
      receivers:
        - otlp/auth
      processors: []
      exporters: []
```

### 서버 인증자 {#server-authenticators}

[서버 인증자][sa]는 `Authenticate` 메서드를 가진 익스텐션이다. 이 함수는 요청이
들어올 때마다 호출되며, 요청을 인증하기 위해 요청의 헤더를 확인한다. 인증자가
요청이 유효하다고 판단하면 `nil` 오류를 반환한다. 요청이 유효하지 않다면, 그
이유를 설명하는 오류를 반환한다.

익스텐션이기 때문에, 인증자는 필요한 리소스(키, 클라이언트, 캐시 등)를
[`Start`](https://pkg.go.dev/go.opentelemetry.io/collector/component#Component)에서
설정하고, `Shutdown`에서 모든 것을 정리해야 한다.

`Authenticate` 함수는 들어오는 모든 요청에 대해 실행되며, 이 함수가 완료될
때까지 파이프라인은 진행될 수 없다. 따라서 인증자는 느리거나 불필요한 차단
작업을 피해야 한다. `context`에 데드라인(deadline)이 설정되어 있다면,
파이프라인이 지연되거나 멈추지 않도록 코드가 이를 준수하도록 해야 한다.

또한 인증자에 좋은 옵저버빌리티(observability), 특히 메트릭과 트레이스를
추가해야 한다. 이는 사용자가 오류가 증가하기 시작할 때 경고(alert)를 설정할 수
있도록 돕고, 인증 문제를 더 쉽게 문제 해결할 수 있게 해준다.

### 클라이언트 인증자 {#client-authenticators}

[클라이언트 인증자][Client authenticators]는 정의된 인터페이스 중 하나 이상을
구현하는 추가 함수를 가진 익스텐션이다. 각 인증자는 인증 데이터를 주입할 수 있게
해주는 객체를 받는다. 예를 들어, HTTP 클라이언트 인증자는
[`http.RoundTripper`](https://pkg.go.dev/net/http#RoundTripper)를 제공하며, gRPC
클라이언트 인증자는
[`credentials.PerRPCCredentials`](https://pkg.go.dev/google.golang.org/grpc/credentials#PerRPCCredentials)를
생성할 수 있다.

## 커스텀 인증자를 배포판에 추가하기 {#add-your-custom-authenticator-to-a-distribution}

커스텀 인증자는 컬렉터 자체와 동일한 바이너리의 일부여야 한다. 자체 인증자를
빌드할 때는 두 가지 옵션이 있다.

- [오픈텔레메트리 컬렉터 빌더][builder]를 사용하여 커스텀 컬렉터 배포판을 빌드할
  수 있다.
- Go 모듈을 게시하는 등의 방법을 제공하여 사용자가 자신의 배포판에 익스텐션을
  추가할 수 있도록 할 수 있다.

[authenticators]:
  https://pkg.go.dev/go.opentelemetry.io/collector/config/configauth
[builder]:
  https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder
[client authenticators]:
  https://pkg.go.dev/go.opentelemetry.io/collector/config/configauth#readme-client-authenticators
[extensions]: /docs/collector/configuration/#extensions
[sa]:
  https://pkg.go.dev/go.opentelemetry.io/collector/config/configauth#readme-server-authenticators
