---
title: 계측
description: 오픈텔레메트리(OpenTelemetry)가 계측을 어떻게 지원하는지 알아본다.
aliases: [instrumenting]
weight: 15
default_lang_commit: 58e684763e8dd50a07ef5fbf428303973025428a
---

시스템이 [관찰 가능][observable]하려면 반드시 **계측(instrumented)** 되어야
한다. 즉, 시스템 구성 요소의 코드가 [트레이스][traces], [메트릭][metrics],
[로그][logs]와 같은 [시그널][signals]을 내보내야(emit) 한다는 의미이다.

오픈텔레메트리(OpenTelemetry)를 사용하면 다음 두 가지 주요 방법으로 코드를
계측할 수 있다.

1. 공식 [대부분의 언어를 위한 API 및 SDK](/docs/languages/)를 통한
   [코드 기반 솔루션](code-based/)
2. [제로 코드 솔루션](zero-code/)

**코드 기반(Code-based)** 솔루션을 사용하면 애플리케이션 자체로부터 더 깊이 있는
인사이트와 풍부한 텔레메트리를 얻을 수 있다. 오픈텔레메트리 API를 사용하여
애플리케이션으로부터 텔레메트리를 생성(generate)할 수 있게 해주며, 이는 제로
코드 솔루션이 생성하는 텔레메트리를 보완하는 필수적인 역할을 한다.

**제로 코드(Zero-code)** 솔루션은 처음 시작할 때나, 텔레메트리를 얻어야 하는
애플리케이션을 수정할 수 없는 경우에 유용하다. 사용 중인 라이브러리 및/또는
애플리케이션이 실행되는 환경으로부터 풍부한 텔레메트리를 제공한다. 다르게
생각하면, 애플리케이션의 _경계(edges)_ 에서 무슨 일이 일어나고 있는지에 대한
정보를 제공한다고도 볼 수 있다.

두 솔루션을 동시에 사용할 수도 있다.

## 추가적인 오픈텔레메트리 이점 {#additional-opentelemetry-benefits}

오픈텔레메트리는 제로 코드 및 코드 기반 텔레메트리 솔루션 이상의 것을 제공한다.
다음 사항들도 오픈텔레메트리의 일부이다.

- 라이브러리는 오픈텔레메트리 API를 의존성(dependency)으로 활용할 수 있으며,
  오픈텔레메트리 SDK를 가져오지(import) 않는 한 해당 라이브러리를 사용하는
  애플리케이션에는 아무런 영향을 미치지 않는다.
- 각 [시그널][signals]에 대해 이를 생성(create), 처리(process),
  내보내기(export)할 수 있는 여러 방법을 사용할 수 있다.
- 구현체에 내장된 [컨텍스트 전파](../context-propagation/) 덕분에, 시그널이
  어디서 생성되었는지에 관계없이 서로 연관 지을 수 있다.
- [리소스](../resources/)와 [계측 스코프](../instrumentation-scope/)를 사용하면
  [호스트](/docs/specs/semconv/resource/host/),
  [운영체제](/docs/specs/semconv/resource/os/),
  [K8s 클러스터](/docs/specs/semconv/resource/k8s/#cluster)와 같은 여러
  엔티티별로 시그널을 그룹화할 수 있다.
- API 및 SDK의 각 언어별 구현체는 [오픈텔레메트리 명세](/docs/specs/otel/)의
  요구사항과 기대사항을 따른다.
- [시맨틱 컨벤션](../semantic-conventions/)은 코드베이스와 플랫폼 전반에서
  표준화를 위해 사용할 수 있는 공통 명명 스키마(naming schema)를 제공한다.

[logs]: ../signals/logs/
[metrics]: ../signals/metrics/
[observable]: ../observability-primer/#what-is-observability
[signals]: ../signals/
[traces]: ../signals/traces/
