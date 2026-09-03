---
title: 옵저버빌리티 개론
description: 핵심 옵저버빌리티(observability) 개념.
weight: 9
default_lang_commit: c4c8aa767969ed84ebb8fcd5a892a5e5f36d47a4
---

## 옵저버빌리티란 무엇인가? {#what-is-observability}

옵저버빌리티(observability)는 시스템의 내부 동작 방식을 알지 못해도 그 시스템에
대한 질문을 던짐으로써 외부에서 시스템을 이해할 수 있게 해준다. 또한 새로운
문제, 즉 '알려지지 않은 미지의 문제(unknown unknowns)'를 쉽게 해결하고 처리할 수
있게 해준다. 그리고 '왜 이런 일이 일어나는가?'라는 질문에 답하는 데도 도움을
준다.

시스템에 대해 이러한 질문을 하려면 애플리케이션이 적절하게
계측(instrument)되어야 한다. 즉 애플리케이션 코드가
[트레이스(traces)](/docs/concepts/signals/traces/),
[메트릭(metrics)](/docs/concepts/signals/metrics/),
[로그(logs)](/docs/concepts/signals/logs/)와 같은
[시그널(signals)](/docs/concepts/signals/)을 내보내야 한다. 애플리케이션이
적절하게 계측되었다는 것은 개발자가 문제를 해결하는 데 필요한 모든 정보를 이미
갖추고 있어서, 문제 해결을 위해 계측을 추가할 필요가 없는 상태를 의미한다.

[오픈텔레메트리(OpenTelemetry)](/docs/what-is-opentelemetry/)는 시스템을 관찰
가능하게 만드는 데 도움이 되도록 애플리케이션 코드를 계측하는 메커니즘이다.

## 신뢰성과 메트릭 {#reliability-and-metrics}

**텔레메트리(telemetry)** 는 시스템과 그 동작에서 발생하는 데이터를 의미한다. 이
데이터는 [트레이스](/docs/concepts/signals/traces/),
[메트릭](/docs/concepts/signals/metrics/), [로그](/docs/concepts/signals/logs/)
형태로 나타날 수 있다.

**신뢰성(reliability)** 은 '서비스가 사용자가 기대하는 대로 동작하고 있는가?'
라는 질문에 답한다. 시스템이 100% 가동 중이더라도, 사용자가 검은색 신발 한
켤레를 장바구니에 담기 위해 '장바구니에 담기'를 클릭했을 때 시스템이 항상 검은색
신발을 담지 못한다면, 그 시스템은 신뢰할 수 **없다**고 할 수 있다.

**메트릭(metrics)** 은 인프라 또는 애플리케이션에 대한 수치 데이터를 일정 기간
동안 집계한 것이다. 예를 들어 시스템 오류율, CPU 사용률, 특정 서비스의 요청률
등이 있다. 메트릭과 이것이 오픈텔레메트리와 어떻게 관련되는지 더 알아보려면
[메트릭](/docs/concepts/signals/metrics/)을 참고한다.

**SLI**, 즉 서비스 수준 지표(Service Level Indicator)는 서비스의 동작에 대한
측정값을 나타낸다. 좋은 SLI는 사용자의 관점에서 서비스를 측정한다. SLI의 예로는
웹 페이지가 로드되는 속도를 들 수 있다.

**SLO**, 즉 서비스 수준 목표(Service Level Objective)는 신뢰성을 조직이나 다른
팀에 전달하는 수단을 나타낸다. 이는 하나 이상의 SLI를 비즈니스 가치와
연결함으로써 달성된다.

## 분산 트레이싱 이해하기 {#understanding-distributed-tracing}

분산 트레이싱(distributed tracing)을 사용하면 복잡한 분산 시스템을 통과하며
전파되는 요청을 관찰할 수 있다. 분산 트레이싱은 애플리케이션이나 시스템 상태의
가시성을 높이고, 로컬 환경에서 재현하기 어려운 동작을 디버깅할 수 있게 해준다.
이는 일반적으로 비결정적(nondeterministic) 문제를 가지고 있거나 로컬에서
재현하기에는 너무 복잡한 분산 시스템에 필수적이다.

분산 트레이싱을 이해하려면 그 구성 요소인 로그, 스팬, 트레이스 각각의 역할을
이해해야 한다.

### 로그 {#logs}

**로그(log)** 는 서비스나 다른 구성 요소에서 내보내는 타임스탬프가 찍힌
메시지이다. [트레이스](#distributed-traces)와 달리 로그는 특정 사용자 요청이나
트랜잭션과 반드시 연관되어 있지는 않다. 로그는 소프트웨어 어디에서나 거의 찾아볼
수 있다. 과거에는 개발자와 운영자 모두 시스템 동작을 이해하는 데 로그에 크게
의존해 왔다.

샘플 로그:

```text
I, [2021-02-23T13:26:23.505892 #22473]  INFO -- : [6459ffe1-ea53-4044-aaa3-bf902868f730] Started GET "/" for ::1 at 2021-02-23 13:26:23 -0800
```

로그는 일반적으로 어디에서 호출되었는지와 같은 맥락 정보(contextual
information)가 부족하기 때문에 코드 실행을 추적하기에는 충분하지 않다.

[스팬](#spans)의 일부로 포함되거나 트레이스 및 스팬과 상호 연관될 때, 로그는
훨씬 더 유용해진다.

로그와 이것이 오픈텔레메트리와 어떻게 관련되는지 더 알아보려면
[로그](/docs/concepts/signals/logs/)를 참고한다.

### 스팬 {#spans}

**스팬(span)** 은 단일 작업 단위 또는 연산(operation)을 나타낸다. 스팬은 요청이
수행하는 특정 연산을 추적하여, 해당 연산이 실행되는 동안 무슨 일이 일어났는지를
보여준다.

스팬은 이름, 시간 관련 데이터,
[구조화된 로그 메시지](/docs/concepts/signals/traces/#span-events), 그리고
[기타 메타데이터(즉 속성)](/docs/concepts/signals/traces/#attributes)를 포함하여
자신이 추적하는 연산에 대한 정보를 제공한다.

#### 스팬 속성 {#span-attributes}

스팬 속성(span attributes)은 스팬에 첨부된 메타데이터이다.

다음 표는 스팬 속성의 예시를 보여준다.

| 키                          | 값                                                                                 |
| :-------------------------- | :--------------------------------------------------------------------------------- |
| `http.request.method`       | `"GET"`                                                                            |
| `network.protocol.version`  | `"1.1"`                                                                            |
| `url.path`                  | `"/webshop/articles/4"`                                                            |
| `url.query`                 | `"?s=1"`                                                                           |
| `server.address`            | `"example.com"`                                                                    |
| `server.port`               | `8080`                                                                             |
| `url.scheme`                | `"https"`                                                                          |
| `http.route`                | `"/webshop/articles/:article_id"`                                                  |
| `http.response.status_code` | `200`                                                                              |
| `client.address`            | `"192.0.2.4"`                                                                      |
| `client.socket.address`     | `"192.0.2.5"` (클라이언트가 프록시를 경유한다)                                     |
| `user_agent.original`       | `"Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/72.0"` |

스팬과 이것이 오픈텔레메트리와 어떻게 관련되는지 더 알아보려면
[스팬](/docs/concepts/signals/traces/#spans)을 참고한다.

### 분산 트레이스 {#distributed-traces}

**분산 트레이스(distributed trace)** 는 흔히 **트레이스(trace)** 라고도 불리며,
마이크로서비스(microservice)나 서버리스(serverless) 애플리케이션과 같은
아키텍처에서 하나의 요청(애플리케이션 또는 최종 사용자가 만든 요청)이 여러
서비스를 통과하며 전파되는 경로를 기록한다.

트레이스는 하나 이상의 스팬으로 구성된다. 첫 번째 스팬은 루트 스팬(root span)을
나타낸다. 각 루트 스팬은 시작부터 끝까지 하나의 요청을 나타낸다. 부모 아래에
있는 스팬들은 요청 중에 발생하는 일(또는 요청을 구성하는 단계)에 대해 더
심층적인 맥락을 제공한다.

예를 들어 사용자가 웹 페이지를 로드할 때, 최초의 HTTP 요청은 API 게이트웨이,
백엔드 서비스, 데이터베이스를 차례로 거칠 수 있다. 이러한 각 단계는 하나의
스팬으로 표현되며, 이 스팬들이 모여 요청의 전체 여정(end-to-end journey)을
보여주는 하나의 트레이스를 이룬다.

트레이싱이 없으면 분산 시스템에서 성능 문제의 근본 원인(root cause)을 찾기가
어려울 수 있다. 트레이싱은 요청이 분산 시스템을 통과하며 흐르는 동안 그 안에서
발생하는 일을 세분화하여 보여줌으로써, 분산 시스템을 디버깅하고 이해하는 부담을
덜어준다.

많은 옵저버빌리티 백엔드는 다음과 같은 워터폴 다이어그램(waterfall diagram)
형태로 트레이스를 시각화한다.

![샘플 트레이스](/img/waterfall-trace.svg '트레이스 워터폴 다이어그램')

워터폴 다이어그램은 루트 스팬과 그 자식 스팬(child span) 간의 부모-자식 관계를
보여준다. 한 스팬이 다른 스팬을 감싸고 있을 때, 이는 중첩 관계(nested
relationship)도 함께 나타낸다.

트레이스와 이것이 오픈텔레메트리와 어떻게 관련되는지 더 알아보려면
[트레이스](/docs/concepts/signals/traces/)를 참고한다.
