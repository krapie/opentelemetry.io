---
title: 컨텍스트 전파
weight: 10
description: 분산 트레이싱을 가능하게 하는 개념에 대해 알아본다.
default_lang_commit: 89a269b12093690dd47ccc32d7f8b22ff7f946d8
---

**컨텍스트 전파(context propagation)** 를 사용하면
[시그널](../signals/)([트레이스](../signals/traces/),
[메트릭](../signals/metrics/), [로그](../signals/logs/))이 어디서 생성되었는지에
관계없이 서로 연관 지을(correlate) 수 있다. 트레이싱에만 국한되는 것은 아니지만,
컨텍스트 전파를 통해 [트레이스](../signals/traces/)는 프로세스 및 네트워크
경계를 가로질러 임의로 분산된 서비스 전반에서 시스템에 대한 인과 관계(causal
relationship) 정보를 구축할 수 있다.

컨텍스트 전파를 이해하려면 컨텍스트와 전파라는 두 가지 별개의 개념을 이해해야
한다.

## 컨텍스트 {#context}

컨텍스트는 전송 서비스와 수신 서비스, 또는
[실행 단위](/docs/specs/otel/glossary/#execution-unit)가 하나의 시그널을 다른
시그널과 연관 지을 수 있도록 관련 정보를 담은 객체이다.

Service A가 Service B를 호출할 때, Service A는 트레이스 ID와 스팬 ID를
컨텍스트의 일부로 포함시킨다. Service B는 이 값을 사용하여 동일한 트레이스에
속하는 새 스팬을 생성하며, Service A의 스팬을 그 부모로 설정한다. 이를 통해
서비스 경계를 넘나드는 요청의 전체 흐름을 추적할 수 있다.

## 전파 {#propagation}

전파는 서비스와 프로세스 간에 컨텍스트를 이동시키는 메커니즘이다. 컨텍스트
객체를 직렬화(serialize)하거나 역직렬화(deserialize)하며, 한 서비스에서 다른
서비스로 전파될 관련 정보를 제공한다.

전파는 일반적으로 계측 라이브러리가 처리하며, 사용자에게는 투명하게 이루어진다.
컨텍스트를 수동으로 전파해야 하는 경우에는
[전파자 API](/docs/specs/otel/context/api-propagators/)를 사용할 수 있다.

오픈텔레메트리(OpenTelemetry)는 여러 공식 전파자를 유지 관리한다. 기본 전파자는
[W3C TraceContext](https://www.w3.org/TR/trace-context/) 명세에 지정된 헤더를
사용한다.

## 예시 {#example}

`POST /cart/add`, `GET /checkout/`와 같은 다양한 HTTP 엔드포인트를 제공하는
`Frontend`라는 서비스가, 사용자가 장바구니에 추가하려는 상품 또는 체크아웃에
포함된 상품에 대한 세부 정보를 받기 위해 HTTP 엔드포인트 `GET /product`를 통해
다운스트림(downstream) 서비스인 `Product Catalog`에 접근한다. `Frontend`로부터
들어오는 요청의 컨텍스트 내에서 `Product Catalog` 서비스의 활동을 파악하기 위해,
컨텍스트(여기서는 트레이스 ID와 "Parent ID"로서의 스팬 ID)가 W3C TraceContext
명세에 정의된 대로 `traceparent` 헤더를 사용하여 전파된다. 이는 해당 ID들이
헤더의 필드에 내장된다는 것을 의미한다.

```text
<version>-<trace-id>-<parent-id>-<trace-flags>
```

예를 들면:

```text
00-a0892f3577b34da6a3ce929d0e0e4736-f03067aa0ba902b7-01
```

### 트레이스 {#traces}

앞서 언급했듯이, 컨텍스트 전파를 통해 트레이스는 서비스 전반에 걸쳐 인과 관계
정보를 구축할 수 있다. 이 예시에서 `Product Catalog` 서비스의 HTTP 엔드포인트
`GET /product`에 대한 두 번의 호출은, `traceparent` 헤더에서 원격 컨텍스트를
추출하여 로컬 컨텍스트에 주입함으로써 트레이스 ID와 Parent ID를 설정하여
`Frontend` 서비스의 업스트림(upstream) 호출과 연관 지을 수 있다. 이를 통해
[Jaeger](https://jaegertracing.io)와 같은 [백엔드](/ecosystem/vendors)에서 두
요청을 하나의 트레이스에 속한 스팬으로 볼 수 있다.

![서비스 간 트레이스 연관 관계를 보여주는 컨텍스트 전파
예시](context-propagation-example.svg)

### 로그 {#logs}

오픈텔레메트리 SDK는 로그를 트레이스와 자동으로 연관 지을 수 있다. 즉
컨텍스트(트레이스 ID, 스팬 ID)를 로그 레코드에 주입할 수 있다는 의미이다. 이를
통해 로그가 속한 트레이스와 스팬의 컨텍스트 내에서 로그를 볼 수 있을 뿐만
아니라, 서비스 또는 실행 단위 경계를 넘어 서로 연관된 로그들도 확인할 수 있다.

### 메트릭 {#metrics}

메트릭의 경우, 컨텍스트 전파를 통해 해당 컨텍스트 내에서 측정값을 집계할 수
있다. 예를 들어 모든 `GET /product` 요청의 응답 시간만 살펴보는 대신,
`POST /cart/add > GET /product`와 `GET /checkout > GET /product`의 조합에 대한
메트릭도 얻을 수 있다.

| 이름                            | 초당 호출 수 | 평균 응답 시간 |
| ------------------------------- | ------------ | -------------- |
| `* > GET /product`              | 370          | 300ms          |
| `POST /cart/add > GET /product` | 330          | 130ms          |
| `GET /checkout > GET /product`  | 40           | 1703ms         |

## 커스텀 컨텍스트 전파 {#custom-context-propagation}

대부분의 사용 사례에서는
[계측 라이브러리 또는 네이티브 라이브러리 계측](/docs/concepts/instrumentation/libraries/)이
컨텍스트 전파를 대신 처리해준다. 하지만 이러한 지원이 제공되지 않아 직접
만들어야 하는 경우도 있다. 이를 위해서는 앞서 언급한 전파자 API를 활용해야 한다.

- 발신자(sender) 측에서는 컨텍스트가 캐리어(carrier)에
  [주입(inject)](/docs/specs/otel/context/api-propagators/#inject)된다. 예를
  들면 HTTP 요청의 헤더에 주입되는 식이다. 다른 경우에는 요청에 대한
  메타데이터를 저장할 수 있는 위치를 직접 찾아야 한다.
- 수신 측에서는 컨텍스트가 캐리어에서
  [추출(extract)](/docs/specs/otel/context/api-propagators/#extract)된다.
  마찬가지로 HTTP의 경우 이는 헤더에서 가져온다. 다른 경우에는 발신 측에서
  컨텍스트를 저장하기 위해 선택한 위치를 그대로 선택한다.

메타데이터를 위한 전용 필드가 없는 프로토콜에서도 컨텍스트를 전파하는 것이
가능하다는 점에 유의한다. 다만 수신 측에서는 데이터가 처리되기 전에 반드시
컨텍스트를 추출하고 제거해야 하며, 그렇지 않으면 정의되지 않은 동작(undefined
behavior)이 발생할 수 있다.

다음 언어들에 대해서는 커스텀 컨텍스트 전파를 위한 단계별 튜토리얼이 존재한다.

- [Erlang](/docs/languages/erlang/propagation/#manual-context-propagation)
- [JavaScript](/docs/languages/js/propagation/#manual-context-propagation)
- [PHP](/docs/languages/php/propagation/#manual-context-propagation)
- [Python](/docs/languages/python/propagation/#manual-context-propagation)

## 보안 모범 사례 {#security-best-practices}

전파는 서비스 경계를 넘어 데이터를 주고받는 것을 포함하며, 이는 보안에 영향을
미칠 수 있다.

### 외부 서비스 {#external-services}

서비스가 외부 서비스(직접 소유하거나 신뢰하지 않는 서비스)와 상호작용할 때는
다음 사항을 고려한다.

- **수신 컨텍스트(Incoming context)**: 외부 소스로부터 컨텍스트를 수락할 때는
  주의를 기울인다. 악의적인 행위자(malicious actor)가 위조된 트레이스 헤더를
  전송하여 트레이싱 데이터를 조작하거나 컨텍스트 파싱의 취약점을 악용할 수도
  있다. 신뢰할 수 없는 소스로부터 들어오는 컨텍스트는 무시하거나 살균(sanitize)
  처리하는 것이 좋다.
- **발신 컨텍스트(Outgoing context)**: 외부 서비스로 전파하는 내용에 주의를
  기울인다. 내부 트레이스 ID, 스팬 ID, 또는 배기지 항목이 내부 아키텍처나
  비즈니스 로직에 대한 민감한 정보를 노출할 수도 있다. 외부 또는 공개
  엔드포인트로는 컨텍스트를 전송하지 않도록 전파자를 구성하는 것이 좋다.

### 배기지 {#baggage}

[배기지](../signals/baggage/)를 사용하면 임의의 키-값 쌍을 전파할 수 있다. 이
데이터는 서비스 경계를 넘어 전파되므로, 민감한 정보(사용자 자격 증명, API 키,
개인 식별 정보(PII) 등)를 배기지에 담지 않도록 주의한다. 그렇지 않으면 로그에
기록되거나 신뢰할 수 없는 다운스트림 서비스로 전송될 수 있다.

## 언어별 SDK 지원 {#support-in-language-sdks}

오픈텔레메트리 API 및 SDK의 개별 언어별 구현에 대해서는, 각 문서 페이지에서
컨텍스트 전파 지원에 대한 세부 정보를 확인할 수 있다.

- [C++](/docs/languages/cpp/instrumentation/#context-propagation)
- .NET
- [Erlang](/docs/languages/erlang/propagation/)
- [Go](/docs/languages/go/instrumentation/#propagators-and-context)
- [Java](/docs/languages/java/api/#context-api)
- [JavaScript](/docs/languages/js/propagation/)
- [PHP](/docs/languages/php/propagation/)
- [Python](/docs/languages/python/propagation/)
- [Ruby](/docs/languages/ruby/instrumentation/#context-propagation)
- Rust
- Swift

> [!IMPORTANT] 도움 요청
>
> .NET, Rust, Swift 언어의 경우, 컨텍스트 전파에 대한 언어별 문서가 아직 없다.
> 이 언어들 중 하나라도 잘 알고 있고 도움을 주고 싶다면,
> [기여하는 방법을 알아본다](/docs/contributing/)!

## 명세(Specification) {#specification}

컨텍스트 전파에 대해 더 알아보려면 [컨텍스트 명세](/docs/specs/otel/context/)를
참고한다.
