---
title: OBI 메트릭 카디널리티
linkTitle: 메트릭 카디널리티
description:
  계측된 환경의 크기와 복잡도를 고려하여, 기본 OBI 설치가 생성하는 메트릭의
  카디널리티를 계산하는 방법에 대한 개요.
weight: 24
cSpell:ignore: kube-system spanmetrics
default_lang_commit: f4cc67cd44fa9d9f23de8f5a121f15d7eea9b043
---

[OBI 메트릭](../metrics/)의 카디널리티(cardinality)는 계측된 환경의 크기와
복잡도에 크게 의존하므로, 간단하고 정확한 공식을 제공할 방법이 없다.

이 문서는 기본 OBI 설치가 생성할 수 있는 메트릭 카디널리티의 근사치를 제공하려고
한다. 각 메트릭 패밀리는 선택적으로 활성화하거나 비활성화할 수 있으므로, OBI가
생성할 수 있는 각 메트릭 유형에 대해 여러 섹션으로 나뉘어 있다.

단순화를 위해, 아래 공식은 단일 클러스터를 가정한다. 클러스터마다 카디널리티를
곱해야 한다.

## 용어 {#terminology}

계속하기 전에, 모호하거나 해석의 여지가 있는 몇 가지 용어를 명확히 해야 한다.

- **인스턴스(Instance)**: 각 계측 대상이다. 애플리케이션 수준 메트릭에서는
  서비스 또는 클라이언트 인스턴스이다. 쿠버네티스에서는 Pod이다. 애플리케이션
  인스턴스는 여러 프로세스에서 실행될 수 있다. 네트워크 수준 메트릭에서 각
  인스턴스는 주어진 호스트의 모든 네트워크 흐름을 계측하는 OBI 인스턴스이다.
- **인스턴스 소유자(Instance Owner)**: 쿠버네티스에서 대부분의 인스턴스(Pod)에는
  소유자 리소스가 있다. 카디널리티를 제어하기 위해 인스턴스 대신 소유자에 대한
  데이터를 보고하는 것이 나을 때가 있다. 인스턴스 소유자의 예로는 Deployment,
  DaemonSet, ReplicaSet, StatefulSet이 있지만, Pod에 소유자가 없으면(독립형 Pod)
  Pod 자체가 소유자로 보고된다.
- **URL 경로(URL Path)**: 클라이언트가 보내고 서버가 수신하는 URL 요청의
  원시(raw) 경로이다. 예: `/clients/348579843/command/833`.
- **URL 라우트(URL Route)**: 카디널리티를 제어하기 위해 의미론적으로 그룹화한,
  URL 요청의 집계된 경로이다. 일반적으로 일부 웹 프레임워크가 코드에서 HTTP
  요청을 정의하는 방식을 모방한다. 예:
  `/clients/{clientId}/command/{command_num}`.
- **작업(Operation)**: 어떤 기능이 요청되었는지를 설명한다.
  - HTTP: 모든 HTTP 동사(예: `GET`) 뒤에 URL 라우트가 붙는다
  - gRPC: 서비스의 경로
  - SQL: SQL 명령(예: `SELECT`, `UPDATE` 또는 기타 명령) 뒤에 대상 테이블이
    붙는다
  - Kafka: Produce/Fetch
- **서버(Server)**: HTTP 또는 gRPC 요청을 수신하고 처리하는 모든 인스턴스이다.
  서버는 클라이언트가 될 수도 있다.
- **클라이언트(Client)**: HTTP, gRPC, 데이터베이스 또는 MQ 요청을 제출하는 모든
  인스턴스이다. 클라이언트는 서버가 될 수도 있다.
- **서비스(Service)**: 쿠버네티스 맥락에서, 공통 호스트 이름과 포트를 통해
  접근하는 서버 그룹이 제공하는 기능이다.
- **엔드포인트(Endpoint)**: 서비스, 서버 또는 클라이언트를 식별하는 IP 또는
  호스트 이름과 포트이다.
- **반환 코드(Return code):** 각 서비스 호출이 반환하며, 실행 결과에 대한 일부
  메타 정보를 설명한다. HTTP의 경우 HTTP 상태 코드이고, 다른 프로토콜의 경우
  일반적으로 0(성공) 또는 1(오류)이다.

## 애플리케이션 수준 메트릭 {#application-level-metrics}

애플리케이션 수준 메트릭의 경우, 카디널리티에 영향을 주는 여러 요인이 있지만
이들이 선형적으로 관련되어 있지 않으므로 간단한 곱셈 공식을 따를 수 없다.

예를 들어 HTTP 라우트 수와 서버 주소 수 모두 카디널리티를 증가시키지만, 모든
서버 인스턴스가 동일한 HTTP 라우트를 받아들이는 것은 아니므로 단순히 곱할 수는
없다.

다음 공식은 매우 대략적인 최대 한계를 제공할 수 있지만,
[우리의 측정](#case-study-cardinality-of-opentelemetry-demo)에서 실제
카디널리티는 계산값보다 2자릿수(2 orders of magnitude) 낮았다. 이러한 이유로,
카디널리티를 미리 계산하려고 하기보다는 측정 중심의 접근 방식을 권장한다.

그래도 전체 카디널리티에 영향을 줄 수 있는 요인 목록은 다음과 같다.

- **인스턴스(Instances)**: 계측된 엔티티의 수이다. 서비스와 클라이언트 모두일 수
  있다.
- **메트릭 이름(MetricNames)**: 애플리케이션 수준 메트릭 이름의 수이다. OBI가
  계측하는 애플리케이션 유형에 따라 다르다. 보고될 각 메트릭마다 하나씩 센다.
- OBI가 다른 애플리케이션에 요청을 수행하는 애플리케이션을 계측할 때의
  클라이언트 측 메트릭:
  - `http.client.request.duration`
  - `http.client.request.body.size`
  - `rpc.client.call.duration`
  - `db.client.operation.duration`
  - `messaging.client.operation.duration`
  - `messaging.process.duration`
- OBI가 다른 애플리케이션의 요청을 처리하는 애플리케이션을 계측할 때의 서버 측
  메트릭:
  - `http.server.request.duration`
  - `http.server.request.body.size`
  - `rpc.server.call.duration`
- **히스토그램 버킷(HistogramBuckets)**: 모든 애플리케이션 수준 메트릭이
  히스토그램이므로, 이를 계산에 넣어 각 메트릭에 곱해야 한다. 버킷은 OBI에서
  구성할 수 있지만, 기본 개수는 지속 시간 메트릭의 경우 15개, 본문 크기 메트릭의
  경우 11개이며, 여기에 2개의 메트릭(히스토그램 sum과 count)이 추가된다.
- **작업(Operations)**: 호출되는 기능에 해당한다. HTTP 서비스에서는 HTTP
  메서드와 HTTP 라우트를 그룹화하고, RPC에서는 RPC 메서드 이름이다.
- **엔드포인트(Endpoints)**: 서버 주소와 포트의 수이다.
- **반환 코드(ReturnCodes)**: 작업의 가능한 결과 수이다. 일반적으로 gRPC에서는
  Ok/Err, 또는 HTTP 상태 코드이다.

### 예시 계산 {#example-calculation}

제시된 카디널리티 공식의 피연산자는 겹칠 수 있다. 예를 들어 계측된 클라이언트
애플리케이션이 `/foo`와 `/bar` HTTP 요청을 보내고 서비스 A와 B 모두에 연결할 수
있으므로:

- 작업: 2
- 엔드포인트: 2

`Operations * Endpoints` 값은 카디널리티를 4배로 만든다. 하지만 `/foo` 라우트가
서비스 A 전용이고 `/bar` 라우트가 서비스 B 전용이라면, 실제 카디널리티 배수는
2에 불과하다.

카디널리티를 계산할 때는 계산에 대한 낙관적 한계와 비관적 한계를 설정한다.

다음 예시는 예시 시스템의 카디널리티를 계산하는 방법을 보여준다. 클라이언트와
백엔드 모두 OBI로 계측된다. 다른 구성 요소는 외부에 있다.

![예시 아키텍처](./cardinality-example.png)

비관적 계산은 다음과 같다.

```text
#Instances * #MetricNames * #HistoBuckets * #Operations * #Endpoints * #ReturnCodes =
= 2 * 5 * 177/3 * 37/3 =2771
```

참조로 사용한 수치:

- 인스턴스 2개, 클라이언트와 백엔드
- 역할과 프로토콜에 따른 메트릭 유형 5개:
  - 클라이언트
    - `rpc.client.call.duration`
  - RPC 서버로서의 백엔드
    - `rpc.server.call.duration`
  - SQL 및 HTTP 클라이언트로서의 백엔드
    - `http.client.request.duration`
    - `http.client.request.body.size`
    - `db.client.operation.duration`
- 대부분의 메트릭이 지속 시간 기반이므로 히스토그램 메트릭 17개
- 작업 7개: RPC Add/List/Delete, HTTP PUT, SQL Insert/Select/Delete
- 엔드포인트 3개: 백엔드, ID 제공자(Identity provider), DB
- 반환 코드 7개: RPC OK/Err, HTTP 200/401/500, SQL OK/Err

카디널리티가 163을 넘지 않을 것처럼 보일 수 있다. 하지만 일부 배수는 전체
시스템에 적용되지 않을 수 있으므로 이 수치는 현실적이지도 정확하지도 않다. 예를
들어 SQL 메서드는 RPC 및 HTTP 메트릭에 곱해지지 않아야 한다.

이 간단한 시나리오에서는, 최대 카디널리티를 수동으로 396으로 셀 수 있으며, 이는
초기 계산값인 2771보다 훨씬 적다.

| #   | 인스턴스   | 메트릭                          | 엔드포인트 | 작업       | 코드 |
| --- | ---------- | ------------------------------- | ---------- | ---------- | ---- |
| 1   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | Add        | OK   |
| 2   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | Add        | Err  |
| 3   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | List       | OK   |
| 4   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | List       | Err  |
| 5   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | Delete     | OK   |
| 6   | 클라이언트 | `rpc.client.call.duration`      | 백엔드     | Delete     | Err  |
| 7   | 백엔드     | `rpc.server.call.duration`      |            | Add        | OK   |
| 8   | 백엔드     | `rpc.server.call.duration`      |            | Add        | Err  |
| 9   | 백엔드     | `rpc.server.call.duration`      |            | List       | OK   |
| 10  | 백엔드     | `rpc.server.call.duration`      |            | List       | Err  |
| 11  | 백엔드     | `rpc.server.call.duration`      |            | Delete     | OK   |
| 12  | 백엔드     | `rpc.server.call.duration`      |            | Delete     | Err  |
| 13  | 백엔드     | `http.client.request.duration`  | ID 제공자  | PUT /login | 200  |
| 14  | 백엔드     | `http.client.request.duration`  | ID 제공자  | PUT /login | 401  |
| 15  | 백엔드     | `http.client.request.duration`  | ID 제공자  | PUT /login | 500  |
| 16  | 백엔드     | `http.client.request.body.size` | ID 제공자  | PUT /login | 200  |
| 17  | 백엔드     | `http.client.request.body.size` | ID 제공자  | PUT /login | 401  |
| 18  | 백엔드     | `http.client.request.body.size` | ID 제공자  | PUT /login | 500  |
| 19  | 백엔드     | `db.client.operation.duration`  | DB         | Insert     | OK   |
| 20  | 백엔드     | `db.client.operation.duration`  | DB         | Insert     | Err  |
| 21  | 백엔드     | `db.client.operation.duration`  | DB         | Select     | OK   |
| 22  | 백엔드     | `db.client.operation.duration`  | DB         | Select     | Err  |
| 23  | 백엔드     | `db.client.operation.duration`  | DB         | Delete     | OK   |
| 24  | 백엔드     | `db.client.operation.duration`  | DB         | Delete     | Err  |

간결성을 위해 히스토그램 버킷은 세지 않았다. 다음으로 메트릭 인스턴스에
히스토그램 버킷을 곱하고, 히스토그램 `_count`와 `_sum`을 더한다.

- 본문 크기 메트릭 인스턴스 3개 x 13 = 39
- 지속 시간 메트릭 인스턴스 21개 x 17 = 357

계산된 총 카디널리티: **396**

위 예시는 카디널리티 영향을 계산하는 하나의 공식을 제공하기가 어렵다는 것을
보여준다. 모든 정보를 알 수 있는 매우 간단한 예시에서는 정확한 카디널리티를 셀
수 있었다. 애플리케이션과 그 상호 연결에 대한 정보가 거의 또는 전혀 없는 대규모
쿠버네티스 클러스터에서는 이런 작업이 불가능할 것이다.

## 네트워크 수준 메트릭 {#network-level-metrics}

OBI는 단일 Counter인 `obi.network.flow.bytes` 하나만 제공하므로, 네트워크 수준
메트릭을 계산하는 것이 애플리케이션 수준 메트릭보다 간단하다. 하지만
카디널리티는 애플리케이션이 얼마나 상호 연결되어 있는지에도 의존한다.

`obi.network.flow.bytes`의 기본 속성은 다음과 같다.

- 방향(request/response)
- 쿠버네티스의 소스 및 대상 엔드포인트 소유자: `k8s_src_owner_name`,
  `k8s_dst_owner_name`, `k8s_src_owner_type`, `k8s_dst_owner_type`,
  `k8s_src_namespace`, `k8s_dst_namespace`
- `k8s_cluster_name`: 클러스터마다 고유하다. 나머지 메트릭과 마찬가지로 단일
  클러스터를 가정한다.

단순화된 비관적 공식은 다음과 같다.

```text
#Directions * #SourceOwners * #DestinationOwners
```

모든 소스 소유자가 모든 대상 소유자에 연결되어 있다고 가정했다. 연결 계수를
적용하는 것이 더 현실적이다. 예를 들어 Deployment/DaemonSet/StatefulSet이
100개이고 각 소유자가 평균 2개의 다른 소유자에 연결된 클러스터는 다음과 같은
카디널리티를 갖는다.

방향 2개 x 소스 소유자 100개 x 대상 소유자 2개 = **400**

## 서비스 그래프 메트릭 {#service-graph-metrics}

서비스 그래프 메트릭은 HTTP, RPC, SQL, Redis, Kafka 등 애플리케이션 메트릭으로
계측할 수 있는 인스턴스에 대해 생성된다. 네트워크 메트릭은 프로토콜에 관계없이
네트워크 트래픽이 있는 모든 인스턴스에 대해 생성된다.

서비스 그래프 메트릭은 다음 메트릭을 생성한다.

- `traces_service_graph_request_client`: 버킷 15개인 히스토그램
- `traces_service_graph_request_server`: 버킷 15개인 히스토그램
- `traces_service_graph_request_failed_total`: 카운터
- `traces_service_graph_request_total`: 카운터

각 메트릭에는 다음 속성도 있다.

- `source`: obi
- `client`와 `client_namespace`
- `server`와 `server_namespace`

계산은 네트워크 메트릭과 유사하지만 카디널리티가 더 높다.

- 단일 카운터 메트릭 대신, 전체 카디널리티가 36인 메트릭/히스토그램 집합을
  보고한다. 즉, 15+2 히스토그램 2개 + 카운터 2개이다.
- 인스턴스의 소유자(예: Deployment)로 집계하는 대신, 클라이언트는 요청을
  제출하는 인스턴스이고, 서버는 일반적으로 단일 서비스 인스턴스를 통해
  접근되므로 소유자일 수 있다.

## 스팬 메트릭 {#span-metrics}

- `traces_spanmetrics_latency`: 버킷 15 + 2개인 히스토그램
- `traces_spanmetrics_calls_total`: 카운터
- `traces_spanmetrics_size_total`: 카운터
- `traces_spanmetrics_response_size_total`: 카운터

각 메트릭에 카디널리티를 추가할 수 있는 속성은 다음과 같다.

- Service/ServiceNamespace/Instance ID
- 스팬 종류(Span Kind): Client/Server/Internal
- 스팬 이름(Span Name): 일반적으로 작업의 이름이며 카디널리티가 높을 수 있다
- 반환 코드

최대 카디널리티는 대략 다음과 같이 계산할 수 있다.

```text
19 metric buckets * 3 span kinds * #Instances * #Operations * #ReturnCodes
```

[앞의 애플리케이션 메트릭 계산 예시](#example-calculation)에서 설명한 대로, 많은
수의 HTTP 반환 코드가 HTTP 서비스에만 곱해지거나, 일부 인스턴스 그룹이 전체
라우트의 일부만 갖는다고 가정했다.

## 사례 연구: 오픈텔레메트리 데모의 카디널리티 {#case-study-cardinality-of-opentelemetry-demo}

이 섹션에서는 3개 노드의 로컬 클러스터에 배포된
[오픈텔레메트리 데모](/docs/demo/architecture/)의 카디널리티를 계산한다. 예시
애플리케이션에 번들된 모든 오픈텔레메트리 계측을 비활성화하고, 계측을 수행하기
위해 OBI를 배포했다.

### 애플리케이션 수준 메트릭 측정 {#measure-application-level-metrics}

대부분의 계측된 인스턴스가 클라이언트이자 서비스이므로, 더 정확하게 하기 위해
공식에서 `#instances` 인자를 무시한다.

```text
#MetricNames * (#HistoBuckets+2) * #Operations * #Endpoints * #ReturnCodes
```

최종 카디널리티에 비선형적으로 영향을 주는 속성의 효과를 최소화하기 위해, 모든
메트릭 유형(HTTP, gRPC, Kafka)에 대해 카디널리티 수치를 별도로 계산한다.

**HTTP 메트릭:**

- 메트릭 4개: 클라이언트, 서버, 요청 크기, 시간
- 평균 15개의 히스토그램 버킷
- 알려진 작업: 75개. 실행 중인 OTel 데모에서 PromQL 쿼리
  `group by (http_request_method, http_route)({__name__=~"http_.*"})`로 측정
- 엔드포인트 26개. 실행 중인 OTel 데모에서 PromQL 쿼리
  `group by (server_address, server_port)({__name__=~"http_.*"})`로 측정
- 응답 상태 코드 6개: 200, 301, 308, 403, 408, 504. 실행 중인 OTel 데모에서 추출

HTTP 메트릭에 대해 계산된 총 최대 한계는 다음과 같다.

```text
4 x 15 x 75 x 26 x 6 =~ 702,000
```

이는 애플리케이션 수준 메트릭에 대해 이 공식이 얼마나 비효과적인지를 보여준다.
알려진 모든 애플리케이션 메트릭 유형을 포함하더라도 측정된 수치가 훨씬 낮기
때문이다.

`count({__name__=~"http_.*|rpc_.*|sql_.*|redis_.*|messaging_.*"})` **→ 9,600**

### 네트워크 수준 메트릭 측정 {#measure-network-level-metrics}

네트워크 수준 메트릭의 경우, 방향 2개(request/response)와 21개
배포(deployment)가 모든 21개 배포에 정보를 요청한다고 가정하면 다음과 같은
카디널리티 수치가 나온다.

2×21×21 = 882

아키텍처를 알고 있다면, 아키텍처 다이어그램의 화살표만 세고 그것이 양방향이라고
가정하여 더 낮은 추정치를 얻을 수 있다.

2x29 = 58

네트워크 메트릭은 오픈텔레메트리 데모 연결, 기타 내부 클러스터 연결, 계측
트래픽을 측정하므로 실제 카디널리티는 더 높다.

`count(obi_network_flow_bytes_total)` **→ 330**

다음 쿼리로 네임스페이스 간 트래픽을 그룹화하면 어느 부분이 오픈텔레메트리
데모에 속하는지 더 잘 파악할 수 있다.

```text
count(obi_network_flow_bytes_total) by (k8s_src_namespace, k8s_dst_namespace)
```

이 쿼리는 다음 정보를 반환한다.

| k8s_src_namespace | k8s_dst_namespace | 개수 |
| ----------------- | ----------------- | ---- |
| default           | default           | 156  |
| kube-system       | default           | 47   |
| default           | kube-system       | 47   |
|                   | default           | 14   |
| default           |                   | 14   |
|                   | kube-system       | 13   |
| kube-system       |                   | 13   |
|                   | gmp-system        | 3    |
| gmp-system        |                   | 3    |
| default           | gmp-system        | 1    |
| gmp-system        | default           | 1    |

데모 구성 요소 간 트래픽에 대해 오픈텔레메트리 데모가 생성하는 네트워크 메트릭
수는 156개이다. `default` 네임스페이스가 소스이자 대상이다. `kube-system`,
`gmp-system`, 또는 네임스페이스가 전혀 없는 다른 트래픽도 있으며, 이는 외부
연결, 텔레메트리, 또는 쿠버네티스 관리에 속한다.

### 서비스 그래프 메트릭 측정 {#measure-service-graph-metrics}

네트워크 메트릭은 서비스 그래프를 만드는 데 자주 사용되지만, 실제 서비스 그래프
메트릭은 다른 형태를 갖는다.

- 단일 카운터 메트릭 대신, 카운터 메트릭 2개와 16+2 버킷을 가진 히스토그램
  메트릭 2개가 더 있다.
- 서비스 그래프 메트릭은 일반적으로 내부 쿠버네티스 트래픽이나 애플리케이션
  수준에서 계측되지 않은 인스턴스의 트래픽을 무시한다.

측정된 수치는 다음과 같다.

`count({__name__=~".*service_graph.*"})` **→ 2300**

### 스팬 메트릭 측정 {#measure-span-metrics}

애플리케이션 수준 메트릭 계산에서, 관련된 매개변수가 많아 분석적 수치를 얻으려는
시도가 어렵다는 것을 보여줬다. PromQL로 측정하면 카디널리티의 정확한 측정값을
얻을 수 있다.

`count({__name__=~".*spanmetrics.*"})` **→ 3900**
