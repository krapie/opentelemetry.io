---
title: 트레이스
weight: 1
description: 애플리케이션을 통과하는 요청의 경로이다.
cSpell:ignore: Guten
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

**트레이스(trace)** 는 요청이 애플리케이션에 전달될 때 발생하는 일에 대한 큰
그림을 보여준다. 애플리케이션이 단일 데이터베이스를 가진 모놀리스(monolith)이든
정교한 서비스 메시(mesh of services)이든 관계없이, 트레이스는 애플리케이션에서
요청이 거치는 전체 "경로"를 이해하는 데 필수적이다.

이를 [스팬](#spans)으로 표현되는 세 가지 작업 단위(unit of work)로 살펴보자.

> [!NOTE]
>
> 다음 JSON 예시들은 특정 포맷을 나타내지 않으며, 특히 더 장황한(verbose)
> [OTLP/JSON](/docs/specs/otlp/#json-protobuf-encoding)은 나타내지 않는다.

`hello` 스팬:

```json
{
  "name": "hello",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "051581bf3cb55c13"
  },
  "parent_id": null,
  "start_time": "2022-04-29T18:52:58.114201Z",
  "end_time": "2022-04-29T18:52:58.114687Z",
  "attributes": {
    "http.route": "some_route1"
  },
  "events": [
    {
      "name": "Guten Tag!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이것은 전체 작업의 시작과 끝을 나타내는 루트 스팬(root span)이다. 트레이스를
나타내는 `trace_id` 필드는 있지만 `parent_id`는 없다는 점에 유의한다. 이를 통해
이것이 루트 스팬임을 알 수 있다.

`hello-greetings` 스팬:

```json
{
  "name": "hello-greetings",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "5fb397be34d26b51"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114304Z",
  "end_time": "2022-04-29T22:52:58.114561Z",
  "attributes": {
    "http.route": "some_route2"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    },
    {
      "name": "bye now!",
      "timestamp": "2022-04-29T18:52:58.114585Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이 스팬은 인사말을 전하는 것과 같은 특정 작업을 캡슐화(encapsulate)하며, 그
부모는 `hello` 스팬이다. 루트 스팬과 동일한 `trace_id`를 공유한다는 점에
유의하는데, 이는 동일한 트레이스의 일부임을 나타낸다. 또한 `hello` 스팬의
`span_id`와 일치하는 `parent_id`를 가진다.

`hello-salutations` 스팬:

```json
{
  "name": "hello-salutations",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "93564f51e1abe1c2"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114492Z",
  "end_time": "2022-04-29T18:52:58.114631Z",
  "attributes": {
    "http.route": "some_route3"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이 스팬은 이 트레이스에서 세 번째 작업을 나타내며, 이전 스팬과 마찬가지로
`hello` 스팬의 자식이다. 이는 또한 `hello-greetings` 스팬의 형제(sibling)임을
의미한다.

이 세 개의 JSON 블록은 모두 동일한 `trace_id`를 공유하며, `parent_id` 필드는
계층 구조(hierarchy)를 나타낸다. 이것이 바로 트레이스인 것이다!

또 한 가지 주목할 점은 각 스팬이 구조화된 로그처럼 보인다는 것이다. 실제로 어느
정도는 그렇기 때문이다! 트레이스를 생각하는 한 가지 방법은, 컨텍스트,
상관관계(correlation), 계층 구조 등이 내장된 구조화된 로그의 모음이라고 보는
것이다. 다만 이러한 "구조화된 로그"는 서로 다른 프로세스, 서비스, VM, 데이터
센터 등에서 발생할 수 있다. 바로 이것이 트레이싱이 모든 시스템에 대한
엔드투엔드(end-to-end) 뷰를 나타낼 수 있게 해주는 이유이다.

오픈텔레메트리(OpenTelemetry)에서 트레이싱이 어떻게 동작하는지 이해하기 위해,
코드를 계측하는 데 관여하는 구성 요소 목록을 살펴보자.

## 트레이서 프로바이더(Tracer Provider) {#tracer-provider}

트레이서 프로바이더(때로는 `TracerProvider`라고도 불린다)는 `Tracer`를 생성하는
팩토리(factory)이다. 대부분의 애플리케이션에서 트레이서 프로바이더는 한 번만
초기화되며, 그 수명 주기(lifecycle)는 애플리케이션의 수명 주기와 일치한다.
트레이서 프로바이더 초기화에는 리소스와 익스포터 초기화도 포함된다. 트레이서
프로바이더는 일반적으로 오픈텔레메트리를 사용한 트레이싱의 첫 단계이다. 일부
언어 SDK에서는 이미 전역(global) 트레이서 프로바이더가 초기화되어 있다.

## 트레이서(Tracer) {#tracer}

트레이서는 서비스에 대한 요청과 같이 특정 작업에서 발생하는 일에 대한 더 많은
정보를 담은 스팬을 생성한다. 트레이서는 트레이서 프로바이더로부터 생성된다.

## 트레이스 익스포터(Trace Exporters) {#trace-exporters}

트레이스 익스포터는 트레이스를 소비자(consumer)에게 전송한다. 이 소비자는 디버깅
및 개발 시점(development-time)을 위한 표준 출력일 수도 있고, 오픈텔레메트리
컬렉터, 또는 원하는 오픈소스나 벤더 백엔드일 수도 있다.

## 컨텍스트 전파(Context Propagation) {#context-propagation}

컨텍스트 전파는 분산 트레이싱을 가능하게 하는 핵심 개념이다. 컨텍스트 전파를
통해 스팬이 생성된 위치와 관계없이 스팬들을 서로 연관 지어(correlate) 하나의
트레이스로 조립할 수 있다. 이 주제에 대해 자세히 알아보려면
[컨텍스트 전파](../../context-propagation) 개념 페이지를 참고한다.

## 스팬(Spans) {#spans}

**스팬(span)** 은 작업(work) 또는 오퍼레이션(operation)의 단위를 나타낸다.
스팬은 트레이스를 구성하는 기본 단위(building block)이다. 오픈텔레메트리에서
스팬은 다음 정보를 포함한다.

- 이름
- 부모 스팬 ID(루트 스팬의 경우 비어 있음)
- 시작 및 종료 타임스탬프
- [스팬 컨텍스트](#span-context)
- [속성](#attributes)
- [스팬 이벤트](#span-events)
- [스팬 링크](#span-links)
- [스팬 상태](#span-status)

스팬 예시:

```json
{
  "name": "/v1/sys/health",
  "context": {
    "trace_id": "7bba9f33312b3dbb8b2c2c62bb7abe2d",
    "span_id": "086e83747d0e381e"
  },
  "parent_id": "",
  "start_time": "2021-10-22 16:04:01.209458162 +0000 UTC",
  "end_time": "2021-10-22 16:04:01.209514132 +0000 UTC",
  "status_code": "STATUS_CODE_OK",
  "status_message": "",
  "attributes": {
    "net.transport": "IP.TCP",
    "net.peer.ip": "172.17.0.1",
    "net.peer.port": "51820",
    "net.host.ip": "10.177.2.152",
    "net.host.port": "26040",
    "http.method": "GET",
    "http.target": "/v1/sys/health",
    "http.server_name": "mortar-gateway",
    "http.route": "/v1/sys/health",
    "http.user_agent": "Consul Health Check",
    "http.scheme": "http",
    "http.host": "10.177.2.152:26040",
    "http.flavor": "1.1"
  },
  "events": [
    {
      "name": "",
      "message": "OK",
      "timestamp": "2021-10-22 16:04:01.209512872 +0000 UTC"
    }
  ]
}
```

스팬은 부모 스팬 ID의 존재가 암시하듯 중첩될 수 있다. 자식 스팬(child span)은
하위 작업(sub-operation)을 나타낸다. 이를 통해 스팬은 애플리케이션에서 수행된
작업을 더 정확하게 포착할 수 있다.

### 스팬 컨텍스트(Span Context) {#span-context}

스팬 컨텍스트는 모든 스팬에 존재하는 불변(immutable) 객체이며 다음 정보를
포함한다.

- 스팬이 속한 트레이스를 나타내는 트레이스 ID
- 스팬의 스팬 ID
- 트레이스에 대한 정보를 담은 바이너리 인코딩인 트레이스 플래그(Trace Flags)
- 벤더별 트레이스 정보를 전달할 수 있는 키-값 쌍 목록인 트레이스 스테이트(Trace
  State)

스팬 컨텍스트는 [분산 컨텍스트](#context-propagation) 및 [배기지](../baggage)와
함께 직렬화되어 전파되는 스팬의 일부이다.

스팬 컨텍스트는 트레이스 ID를 포함하고 있기 때문에 [스팬 링크](#span-links)를
생성할 때 사용된다.

### 속성(Attributes) {#attributes}

속성은 스팬에 주석을 달아(annotate) 해당 스팬이 추적하는 작업에 대한 정보를
전달하는 데 사용할 수 있는 메타데이터를 담은 키-값 쌍이다.

예를 들어 이커머스(eCommerce) 시스템에서 사용자의 장바구니에 항목을 추가하는
작업을 스팬이 추적한다면, 사용자 ID, 장바구니에 추가할 항목의 ID, 장바구니 ID를
캡처할 수 있다.

속성은 스팬 생성 중 또는 생성 후에 추가할 수 있다. SDK 샘플링에서 속성을 사용할
수 있도록 스팬 생성 시점에 속성을 추가하는 것이 좋다. 스팬 생성 후에 값을
추가해야 하는 경우에는 해당 값으로 스팬을 업데이트한다.

속성에는 각 언어 SDK가 구현하는 다음과 같은 규칙이 있다.

- 키는 null이 아닌 문자열 값이어야 한다
- 값은 null이 아닌 문자열, 불리언(boolean), 부동소수점 값(floating point value),
  정수(integer) 또는 이러한 값들의 배열이어야 한다

또한 일반적인 작업에 흔히 존재하는 메타데이터에 대한 잘 알려진 명명 규칙(naming
convention)인
[시맨틱 속성(Semantic Attributes)](/docs/specs/semconv/general/trace/)도 있다.
시스템 전반에서 일반적인 종류의 메타데이터를 표준화할 수 있도록 가능하면 시맨틱
속성 명명 방식을 사용하는 것이 도움이 된다.

### 스팬 이벤트(Span Events) {#span-events}

스팬 이벤트는 스팬에 대한 구조화된 로그 메시지(또는 주석)라고 생각할 수 있으며,
일반적으로 스팬의 지속 시간(duration) 중 의미 있는 단일 시점(singular point in
time)을 나타내는 데 사용된다.

예를 들어 웹 브라우저에서 다음 두 가지 시나리오를 생각해보자.

1. 페이지 로드 추적
2. 페이지가 상호작용 가능(interactive) 상태가 되는 시점 표시

첫 번째 시나리오는 시작과 끝이 있는 작업이므로 이를 추적하는 데는 스팬을
사용하는 것이 가장 적합하다.

두 번째 시나리오는 의미 있는 단일 시점을 나타내므로 이를 추적하는 데는 스팬
이벤트를 사용하는 것이 가장 적합하다.

#### 스팬 이벤트와 스팬 속성 중 언제 무엇을 사용할지 {#when-to-use-span-events-versus-span-attributes}

스팬 이벤트에도 속성이 포함되므로 속성 대신 이벤트를 언제 사용해야 하는지에 대한
질문이 항상 명확한 답을 갖는 것은 아니다. 결정을 내릴 때는 특정 타임스탬프가
의미 있는지를 고려한다.

예를 들어 스팬으로 작업을 추적하는 중 해당 작업이 완료되면, 그 작업에서 얻은
데이터를 텔레메트리에 추가하고 싶을 수 있다.

- 작업이 완료되는 시점의 타임스탬프가 의미 있거나 관련이 있다면, 해당 데이터를
  스팬 이벤트에 첨부한다.
- 타임스탬프가 의미가 없다면, 해당 데이터를 스팬 속성으로 첨부한다.

### 스팬 링크(Span Links) {#span-links}

링크는 하나의 스팬을 하나 이상의 다른 스팬과 연관 지어 인과 관계(causal
relationship)를 암시할 수 있도록 존재한다. 예를 들어 일부 작업이 트레이스로
추적되는 분산 시스템이 있다고 가정해보자.

이러한 작업 중 일부에 대한 응답으로 추가 작업이 실행을 위해 큐에 들어가지만, 그
실행은 비동기(asynchronous)적으로 이루어진다. 이 후속 작업 또한 트레이스로
추적할 수 있다.

후속 작업에 대한 트레이스를 첫 번째 트레이스와 연관 짓고 싶지만, 후속 작업이
언제 시작될지 예측할 수 없다. 이 두 트레이스를 서로 연관 지어야 하므로 스팬
링크를 사용한다.

첫 번째 트레이스의 마지막 스팬을 두 번째 트레이스의 첫 번째 스팬에 링크할 수
있다. 이제 두 스팬은 서로 인과적으로 연관된다.

링크는 선택 사항이지만 트레이스 스팬들을 서로 연관 짓는 좋은 방법이다.

자세한 내용은 [스팬 링크](/docs/specs/otel/trace/api/#link)를 참고한다.

### 스팬 상태(Span Status) {#span-status}

모든 스팬에는 상태(status)가 있다. 가능한 값은 다음 세 가지이다.

- `Unset`
- `Error`
- `Ok`

기본값은 `Unset`이다. 스팬 상태가 `Unset`이라는 것은 해당 스팬이 추적한 작업이
오류 없이 성공적으로 완료되었음을 의미한다.

스팬 상태가 `Error`이면, 해당 스팬이 추적하는 작업에서 어떤 오류가 발생했음을
의미한다. 예를 들어 요청을 처리하는 서버에서 발생한 HTTP 500 오류가 원인일 수
있다.

스팬 상태가 `Ok`이면, 해당 스팬이 애플리케이션 개발자에 의해 명시적으로 오류가
없다고 표시되었음을 의미한다. 직관적이지 않을 수 있지만, 스팬이 오류 없이 완료된
것으로 알려진 경우에도 스팬 상태를 굳이 `Ok`로 설정할 필요는 없다. 이는
`Unset`으로 이미 다루어지기 때문이다. `Ok`가 하는 역할은 사용자가 명시적으로
설정한 스팬 상태에 대해 명확한 "최종 판단(final call)"을 나타내는 것이다. 이는
개발자가 스팬이 "성공(successful)" 외에 다른 방식으로 해석되지 않기를 원하는
모든 상황에서 유용하다.

다시 정리하면, `Unset`은 오류 없이 완료된 스팬을 나타낸다. `Ok`는 개발자가
스팬을 명시적으로 성공으로 표시했을 때를 나타낸다. 대부분의 경우 스팬을
명시적으로 `Ok`로 표시할 필요는 없다.

### 스팬 종류(Span Kind) {#span-kind}

스팬이 생성될 때, 그 종류는 `Client`, `Server`, `Internal`, `Producer`,
`Consumer` 중 하나이다. 이 스팬 종류는 트레이스를 어떻게 조립해야 하는지에 대한
힌트를 트레이싱 백엔드에 제공한다. 오픈텔레메트리 명세에 따르면 서버 스팬의
부모는 흔히 원격 클라이언트 스팬이며, 클라이언트 스팬의 자식은 대개 서버
스팬이다. 마찬가지로 컨슈머 스팬의 부모는 항상 프로듀서이고, 프로듀서 스팬의
자식은 항상 컨슈머이다. 제공되지 않으면 스팬 종류는 내부(internal)로 간주된다.

SpanKind에 대한 자세한 내용은 [SpanKind](/docs/specs/otel/trace/api/#spankind)를
참고한다.

#### 클라이언트(Client) {#client}

클라이언트 스팬은 나가는 HTTP 요청이나 데이터베이스 호출과 같은
동기(synchronous) 아웃바운드 원격 호출을 나타낸다. 여기서 "동기"란
`async/await`를 가리키는 것이 아니라, 나중에 처리하기 위해 큐에 들어가지
않는다는 사실을 의미한다는 점에 유의한다.

#### 서버(Server) {#server}

서버 스팬은 들어오는 HTTP 요청이나 원격 프로시저 호출(remote procedure call)과
같은 동기 인바운드 원격 호출을 나타낸다.

#### 내부(Internal) {#internal}

내부 스팬은 프로세스 경계를 넘지 않는 작업을 나타낸다. 함수 호출이나 Express
미들웨어를 계측하는 것과 같은 작업이 내부 스팬을 사용할 수 있다.

#### 프로듀서(Producer) {#producer}

프로듀서 스팬은 나중에 비동기적으로 처리될 수 있는 작업(job)의 생성을 나타낸다.
이는 작업 큐에 삽입되는 원격 작업일 수도 있고, 이벤트 리스너가 처리하는 로컬
작업일 수도 있다.

#### 컨슈머(Consumer) {#consumer}

컨슈머 스팬은 프로듀서가 생성한 작업의 처리를 나타내며, 프로듀서 스팬이 이미
종료된 이후 한참 지나서 시작될 수도 있다.

## 명세(Specification) {#specification}

자세한 내용은 [트레이스 명세](/docs/specs/otel/overview/#tracing-signal)를
참고한다.
