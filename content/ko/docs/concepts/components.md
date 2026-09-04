---
title: 구성 요소
description: 오픈텔레메트리(OpenTelemetry)를 구성하는 주요 구성 요소
aliases: [data-collection]
weight: 20
default_lang_commit: 99a39c5e4e51daba968bfbb3eb078be4a14ad363
---

오픈텔레메트리(OpenTelemetry)는 현재 여러 주요 구성 요소로 이루어져 있다.

- [명세](#specification)
- [컬렉터](#collector)
- [언어별 API 및 SDK 구현체](#language-specific-api--sdk-implementations)
  - [계측 라이브러리](#instrumentation-libraries)
  - [익스포터](#exporters)
  - [제로 코드 계측](#zero-code-instrumentation)
  - [리소스 감지기](#resource-detectors)
  - [서비스 간 전파자](#cross-service-propagators)
  - [샘플러](#samplers)
- [쿠버네티스 오퍼레이터](#kubernetes-operator)
- [서비스형 함수 자산](#function-as-a-service-assets)

오픈텔레메트리를 사용하면 텔레메트리 데이터를 생성하고 내보내기 위한 벤더별
SDK와 도구가 필요하지 않게 된다.

## 명세(Specification) {#specification}

모든 구현체가 따라야 하는 언어 간 공통 요구사항과 기대사항을 설명한다. 용어의
정의를 넘어, 명세는 다음 사항을 정의한다.

- **API:** 트레이스, 메트릭, 로그 데이터를 생성하고 서로 연관 짓기 위한 데이터
  타입과 오퍼레이션(operation)을 정의한다.
- **SDK:** API의 언어별 구현체에 대한 요구사항을 정의한다. 설정, 데이터 처리,
  내보내기(exporting) 개념도 여기서 정의된다.
- **데이터:** 텔레메트리 백엔드가 지원할 수 있는 오픈텔레메트리
  프로토콜(OpenTelemetry Protocol, OTLP)과 벤더 중립적인 시맨틱 컨벤션을
  정의한다.

자세한 내용은 [명세](/docs/specs/)를 참고한다.

## 컬렉터(Collector) {#collector}

오픈텔레메트리 컬렉터는 텔레메트리 데이터를 수신(receive), 처리(process),
내보내기(export)할 수 있는 벤더 중립적인(vendor-agnostic) 프록시(proxy)이다.
OTLP, Jaeger, Prometheus를 비롯해 다양한 상용/독점(commercial/proprietary) 도구
등 여러 포맷으로 텔레메트리 데이터를 수신하는 것을 지원하며, 하나 이상의
백엔드로 데이터를 전송하는 것도 지원한다. 또한 내보내기 전에 텔레메트리 데이터를
처리하고 필터링하는 것도 지원한다.

자세한 내용은 [컬렉터](/docs/collector/)를 참고한다.

## 언어별 API 및 SDK 구현체 {#language-specific-api--sdk-implementations}

오픈텔레메트리는 오픈텔레메트리 API를 사용하여 원하는 언어로 텔레메트리 데이터를
생성하고, 이를 원하는 백엔드로 내보낼 수 있게 해주는 언어별 SDK도 제공한다.
이러한 SDK를 사용하면 일반적인 라이브러리와 프레임워크를 위한 계측 라이브러리를
통합할 수 있으며, 이를 애플리케이션의 수동 계측(manual instrumentation)과
연결하는 데 사용할 수 있다.

자세한 내용은 [계측](/docs/concepts/instrumentation/)을 참고한다.

### 계측 라이브러리 {#instrumentation-libraries}

오픈텔레메트리는 지원되는 언어에서 널리 사용되는 라이브러리와 프레임워크로부터
관련 텔레메트리 데이터를 생성하는 다양한 구성 요소를 지원한다. 예를 들어 HTTP
라이브러리에서 발생하는 인바운드 및 아웃바운드 HTTP 요청은 해당 요청에 대한
데이터를 생성한다.

오픈텔레메트리가 지향하는 목표(aspirational goal) 중 하나는 널리 사용되는 모든
라이브러리가 기본적으로 관찰 가능(observable)하도록 만들어져, 별도의 의존성이
필요하지 않도록 하는 것이다.

자세한 내용은 [라이브러리 계측](/docs/concepts/instrumentation/libraries/)을
참고한다.

### 익스포터(Exporter) {#exporters}

{{% docs/languages/exporters/intro %}}

### 제로 코드(Zero-code) 계측 {#zero-code-instrumentation}

해당되는 경우, 오픈텔레메트리의 언어별 구현체는 소스 코드를 건드리지 않고도
애플리케이션을 계측할 수 있는 방법을 제공한다. 기본 메커니즘은 언어에 따라
다르지만, 제로 코드 계측은 애플리케이션에 오픈텔레메트리 API 및 SDK 기능을
추가한다. 또한 일련의 계측 라이브러리와 익스포터 의존성을 추가할 수도 있다.

자세한 내용은 [제로 코드 계측](/docs/concepts/instrumentation/zero-code/)을
참고한다.

### 리소스 감지기(Resource detectors) {#resource-detectors}

[리소스](/docs/concepts/resources/)는 텔레메트리를 생성하는 엔터티를 리소스
속성으로 나타낸다. 예를 들어, 쿠버네티스의 컨테이너에서 실행되며 텔레메트리를
생성하는 프로세스에는 Pod 이름과 네임스페이스가 있으며, 배포(deployment) 이름도
있을 수 있다. 이러한 속성을 모두 리소스에 포함할 수 있다.

오픈텔레메트리의 언어별 구현체는 `OTEL_RESOURCE_ATTRIBUTES` 환경 변수를 통해,
그리고 프로세스 런타임, 서비스, 호스트, 운영체제와 같은 다양한 일반적인 엔터티에
대해 리소스 감지(resource detection) 기능을 제공한다.

자세한 내용은 [리소스](/docs/concepts/resources/)를 참고한다.

### 서비스 간 전파자 {#cross-service-propagators}

전파(propagation)는 서비스와 프로세스 간에 데이터를 이동시키는 메커니즘이다.
트레이싱에만 국한되지는 않지만, 전파를 통해 트레이스는 프로세스와 네트워크
경계를 가로질러 임의로 분산된 서비스 전반에서 시스템에 대한 인과 관계
정보(causal information)를 구축할 수 있다.

대부분의 사용 사례에서 컨텍스트 전파는 계측 라이브러리를 통해 이루어진다. 필요한
경우, 스팬의 컨텍스트나 [배기지](/docs/concepts/signals/baggage/)와 같은 횡단
관심사(cross-cutting concern)를 직렬화하고 역직렬화하기 위해
전파자(propagator)를 직접 사용할 수도 있다.

### 샘플러 {#samplers}

샘플링은 시스템에서 생성되는 트레이스의 양을 제한하는 프로세스이다.
오픈텔레메트리의 각 언어별 구현체는 여러
[헤드 샘플러](/docs/concepts/sampling/#head-sampling)를 제공한다.

자세한 내용은 [샘플링](/docs/concepts/sampling)을 참고한다.

## 쿠버네티스 오퍼레이터 {#kubernetes-operator}

오픈텔레메트리 오퍼레이터(OpenTelemetry Operator)는 쿠버네티스
오퍼레이터(Kubernetes Operator)의 구현체이다. 이 오퍼레이터는 오픈텔레메트리
컬렉터와, 오픈텔레메트리를 사용하는 워크로드의 자동 계측을 관리한다.

자세한 내용은 [K8s 오퍼레이터](/docs/platforms/kubernetes/operator/)를 참고한다.

## 서비스형 함수(Function as a Service) 자산 {#function-as-a-service-assets}

오픈텔레메트리는 여러 클라우드 벤더가 제공하는 서비스형
함수(Function-as-a-Service)를 모니터링하는 다양한 방법을 지원한다.
오픈텔레메트리 커뮤니티는 현재 애플리케이션을 자동 계측할 수 있는 사전 빌드된
Lambda 레이어와, 애플리케이션을 수동 또는 자동으로 계측할 때 사용할 수 있는
독립형(standalone) 컬렉터 Lambda 레이어 옵션을 제공한다.

자세한 내용은 [서비스형 함수](/docs/platforms/faas/)를 참고한다.
