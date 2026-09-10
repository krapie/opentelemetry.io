---
title: OBI 메트릭 및 트레이스 속성 구성
linkTitle: 메트릭 속성
description:
  보고되는 속성을 제어하는 메트릭 및 트레이스 속성 구성 요소를 구성한다.
  여기에는 인스턴스 ID 데코레이션과 계측된 쿠버네티스 Pod의 메타데이터가
  포함된다.
weight: 30
cSpell:ignore: kube kubecache kubeconfig OpenShift replicaset statefulset
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

OBI가 메트릭과 트레이스의 속성을 데코레이션(decoration)하는 방식을 구성할 수
있다. `attributes` 최상위 YAML 섹션을 사용하여 속성이 설정되는 방식을 활성화하고
구성한다.

[OBI 내보낸 메트릭](../../metrics/) 문서는 각 메트릭으로 보고할 수 있는 속성을
나열한다. OBI는 일부 속성을 기본적으로 보고하고, 카디널리티(cardinality)를
제어하기 위해 나머지는 숨긴다.

`select` 하위 섹션으로 메트릭 속성, 선택적 트레이스 속성, 리소스 속성을
제어한다. 각 항목에는 두 개의 하위 속성 `include`와 `exclude`가 있다.

- `include`는 보고할 속성의 목록이다. 각 속성은 이름이거나 와일드카드일 수 있다.
  예를 들어 `k8s.dst.*`는 `k8s.dst`로 시작하는 모든 속성을 포함한다. `include`
  목록을 제공하지 않으면 OBI는 기본 속성 집합을 보고한다. 주어진 메트릭의 기본
  속성에 대한 자세한 내용은 [OBI 내보낸 메트릭](../../metrics/)을 참고한다
- `exclude`는 `include` 목록 또는 기본 속성 집합에서 제거할 속성 이름이나
  와일드카드의 목록이다

예시:

```yaml
attributes:
  select:
    obi_network_flow_bytes:
      # limit the OTEL_EBPF_network_flow_bytes attributes to only the three attributes
      include:
        - obi.ip
        - src.name
        - dst.port
    sql_client_duration:
      # report all the possible attributes but db_statement
      include: ['*']
      exclude: ['db_statement']
    http_client_request_duration:
      # report the default attribute set but exclude the Kubernetes Pod information
      exclude: ['k8s.pod.*']
    resource:
      # keep the default resource set, except for these high-cardinality fields
      exclude: ['k8s.pod.name', 'service.instance.id']
```

또한 와일드카드를 메트릭 이름으로 사용하여 이름이 같은 메트릭 그룹에 대해 속성을
추가하고 제외할 수 있다. 예를 들면 다음과 같다.

```yaml
attributes:
  select:
    http_*:
      include: ['*']
      exclude: ['http_path', 'http_route']
    http_client_*:
      # override http_* exclusion
      include: ['http_path']
    http_server_*:
      # override http_* exclusion
      include: ['http_route']
```

앞의 예시에서, 이름이 `http_` 또는 `http.`로 시작하는 모든 메트릭은
`http_path`와 `http_route`(또는 `http.path`/`http.route`)를 제외한 가능한 모든
속성을 포함한다. `http_client_*`와 `http_server_*` 섹션은 기본 구성을
재정의하여, HTTP 클라이언트 메트릭에는 `http_path` 속성을, HTTP 서버 메트릭에는
`http_route` 속성을 활성화한다.

메트릭 이름이 와일드카드를 사용하는 여러 정의와 일치하면, 정확한 일치가
와일드카드 일치보다 우선한다.

## 리소스 선택 {#resource-selection}

특수 `resource` 키를 사용하여 OTLP 리소스와 Prometheus `target_info` 및
`traces_target_info` 메트릭의 리소스 속성을 필터링한다. `include`를 생략하면
OBI는 기본적으로 감지된 리소스 속성을 보존하고 일치하는 `exclude` 항목만
제거한다. 쿠버네티스 및 `OTEL_RESOURCE_ATTRIBUTES` 값을 포함하여 런타임에 발견된
리소스 속성도 동일한 선택을 사용한다. 선택은 내보내기 대상이 될 수 있는 속성만
필터링한다. 시작 시점에 알 수 없는 Prometheus 속성에는 계속
`prometheus_export.extra_resource_attributes`를 사용한다.

## 트레이스 선택 {#trace-selection}

내보내는 오픈텔레메트리(OpenTelemetry) 트레이스에는 `attributes.select` 아래에서
(메트릭 이름이 아닌) `traces` 키를 사용한다. 이 키는 `db.query.text`,
`graphql.document`, `url.query`, GenAI 페이로드 속성, `db.response.error` 같은
트레이스 데코레이션을 제어한다.

```yaml
attributes:
  select:
    traces:
      include:
        - db.query.text
        - db.response.error
```

`url.query`는 HTTP 요청에 쿼리 문자열이 포함되어 있을 때 기본적으로 활성화된다.
이를 생략하려면 `attributes.select.traces.exclude`에 `url.query`를 추가한다.
OBI는 클라이언트 스팬의 `url.full`에도 쿼리 문자열을 보존하며, 알려진 민감한
키의 값은 자동으로 `REDACTED`로 대체한다.

대소문자를 구분하는 내장 삭제(redaction) 목록을 확장하거나 좁히려면
`attributes.sensitive_query_params`를 사용한다.

```yaml
attributes:
  sensitive_query_params:
    add: [tenant_secret]
    remove: [session]
```

이에 상응하는 환경 변수는 `OTEL_EBPF_SENSITIVE_QUERY_PARAMS_ADD`와
`OTEL_EBPF_SENSITIVE_QUERY_PARAMS_REMOVE`이며, 쉼표로 구분된 값을 사용한다. 내장
목록은 일반적인 자격 증명, 인증 토큰, 비밀번호, 결제 식별자, 서명된 AWS 및
Google Cloud URL 매개변수를 포함한다.

MCP 도구 호출 인수와 결과는 선택적 `gen_ai.tool.call.arguments` 및
`gen_ai.tool.call.result` 트레이스 속성으로 사용할 수 있다. 이 값에는 민감한
페이로드가 포함될 수 있으므로, `attributes.select.traces.include`에 정확한
항목이 필요하다. `gen_ai.*` 같은 와일드카드 항목으로는 활성화되지 않는다.

### `db.response.error` {#db-response-error}

`db.response.error`는 오픈텔레메트리 시맨틱 컨벤션의 일부가 아니다. OBI는 이
문자열을 `attributes.select.traces` 아래의 구성 플래그로만 사용한다.

| 조건                  | 동작                                                                                                                                                                                                                                           |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 포함되지 않음(기본값) | 실패한 데이터베이스 관련 스팬(SQL, Redis, MongoDB, Couchbase, Memcached, HTTP를 통한 SQL++)에서 `span.status.message`는 비어 있는 상태로 유지된다. 이는 다른 선택적 속성(예: `db.query.text`)의 동작과 일관된다. 즉, 선택되지 않으면 생략된다. |
| 포함됨                | 동일한 스팬에서 `span.status.message`는 프로토콜 응답에서 파싱한 실제 오류 설명으로 설정된다.                                                                                                                                                  |

`db.response.error`는 OTLP 트레이스에서 스팬 속성으로 첨부되는 경우가 절대 없다.
내보내기 중에 OBI는 게이팅된 값을 데이터베이스 스팬의 `span.status.message`를
구성하는 데에만 사용한 다음, 내보낸 스팬에서 해당 속성을 제거한다. 이 옵션을
활성화하면 스팬의 별도 `db.response.error` 필드가 아니라 상태 설명이 변경된다.

이 옵트인이 존재하는 이유는 오류 문자열에 민감하거나 고카디널리티인 세부
정보(스키마 이름, 쿼리 조각, 데이터 값)가 포함될 수 있기 때문이다.

## 분산 트레이스와 컨텍스트 전파 {#distributed-traces-and-context-propagation}

YAML 섹션: `ebpf`

YAML 구성의 `ebpf` 섹션이나 환경 변수로 이 구성 요소를 구성할 수 있다.

| YAML<br>환경 변수                                                | 설명                                                                                                                                                                      | 유형    | 기본값   |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------- |
| `context_propagation`<br>`OTEL_EBPF_BPF_CONTEXT_PROPAGATION`     | 트레이스 컨텍스트 전파 방식을 제어한다. 허용 값: `all`, `headers`, `tcp`, `headers,tcp`, `disabled`. 자세한 내용은 [컨텍스트 전파 섹션](#context-propagation)을 참고한다. | string  | disabled |
| `track_request_headers`<br>`OTEL_EBPF_BPF_TRACK_REQUEST_HEADERS` | 트레이스 스팬을 위해 수신되는 `Traceparent` 헤더를 추적한다. 자세한 내용은 [요청 헤더 추적 섹션](#track-request-headers)을 참고한다.                                      | boolean | false    |

### 컨텍스트 전파 {#context-propagation}

OBI는 나가는 HTTP 요청에 `Traceparent` 헤더 값을 주입하여, 수신된 컨텍스트를
다운스트림 서비스로 전파할 수 있다. 이 컨텍스트 전파는 모든 프로그래밍 언어에서
동작한다.

TLS로 암호화된 HTTP 요청(HTTPS)의 경우, OBI는 `Traceparent` 헤더 값을 TCP/IP
패킷 수준에서 인코딩한다. 통신의 양쪽 모두에 OBI가 존재해야 한다.

TCP/IP 패킷 수준 인코딩은 Linux 트래픽 제어(Traffic Control, TC)를 사용한다.
TC를 함께 사용하는 eBPF 프로그램은 OBI와 올바르게 체이닝되어야 한다. 프로그램
체이닝에 대한 자세한 내용은 [Cilium 호환성 문서](../../cilium-compatibility/)를
참고한다.

`context_propagation="headers"`로 설정하면 TCP 수준 전파와 Linux 트래픽 제어
프로그램을 비활성화할 수 있다. 이 모드는 모든 오픈텔레메트리 분산 트레이싱
라이브러리와 완전히 호환된다.

컨텍스트 전파 값:

- `all`: HTTP 헤더와 TCP 컨텍스트 전파를 모두 활성화한다
- `headers`: HTTP 헤더를 통한 컨텍스트 전파만 활성화한다
- `tcp`: TCP 패킷 경로를 통한 컨텍스트 전파만 활성화한다
- `headers,tcp`: 두 방식을 모두 명시적으로 활성화한다
- `disabled`: 트레이스 컨텍스트 전파를 비활성화한다

이전의 `http` 별칭은 제거되었다. 더 이상 사용되지 않는 `ip` 값은 효과가 없다.
TCP 옵션 전파에는 `tcp`를 사용한다.

컨테이너화된 환경(쿠버네티스 및 Docker)에서 이 옵션을 사용하려면 다음이
필요하다.

- 호스트 네트워크 액세스 `hostNetwork: true`와 함께 OBI를 `DaemonSet`으로
  배포한다
- 호스트의 `/sys/fs/cgroup` 경로를 로컬 `/sys/fs/cgroup` 경로로 볼륨 마운트한다
- OBI 컨테이너에 `CAP_NET_ADMIN` capability를 부여한다

네트워크 수준 전파는 스트림별 `traceparent` HPACK 헤더를 주입하고 파싱하여
gRPC를 지원한다. HTTP/2는 하나의 연결에서 여러 트레이스 컨텍스트를 다중화하므로,
gRPC에는 TCP 옵션 전파가 사용되지 않는다. gRPC가 아닌 일반 HTTP/2 컨텍스트
전파는 여전히 Go 라이브러리 계측으로 제한된다. Go가 아닌 gRPC 서비스의 경우,
OBI가 시작되기 전에 설정된 지속 연결은 전파를 위해 인식되지 않을 수 있다.

쿠버네티스에서 분산 트레이스를 구성하는 방법의 예시는
[OBI를 사용한 분산 트레이스](../../distributed-traces/) 가이드를 참고한다.

### 요청 헤더 추적 {#track-request-headers}

이 옵션은 OBI가 수신되는 모든 `Traceparent` 헤더 값을 처리하도록 한다. 활성화된
경우, OBI가 `Traceparent` 헤더 값을 가진 수신 서버 요청을 보면, 제공된 '트레이스
ID'를 사용하여 자체 트레이스 스팬을 생성한다.

이 옵션은 Go 애플리케이션에는 영향을 주지 않으며, Go 애플리케이션에서는
`Traceparent` 필드가 항상 처리된다.

이 옵션을 활성화하면 요청량이 많은 시나리오에서 성능 오버헤드가 증가할 수 있다.
이 옵션은 OBI 트레이스를 생성할 때만 유용하며, 메트릭에는 영향을 주지 않는다.

### 기타 속성 {#other-attributes}

| YAML 옵션<br>환경 변수                                     | 설명                                                                      | 유형    | 기본값  |
| ---------------------------------------------------------- | ------------------------------------------------------------------------- | ------- | ------- |
| `heuristic_sql_detect`<br>`OTEL_EBPF_HEURISTIC_SQL_DETECT` | 휴리스틱 SQL 클라이언트 감지를 활성화한다. 자세한 내용은 아래를 참고한다. | boolean | (false) |

`heuristic sql detect` 옵션은 프로토콜이 직접 지원되지 않더라도 OBI가 쿼리
구문을 검사하여 SQL 클라이언트 요청을 감지하도록 한다. 기본적으로 OBI는 SQL
클라이언트 요청을 바이너리 프로토콜 형식으로 감지한다. OBI가 직접 지원하지 않는
데이터베이스 기술을 사용하는 경우, 이 옵션을 활성화하여 데이터베이스 클라이언트
텔레메트리를 얻을 수 있다. 이 옵션은 오탐(false positive)을 만들 수 있으므로,
예를 들어 애플리케이션이 TCP 연결을 통해 로깅용 SQL 텍스트를 보내는 경우 오탐이
발생할 수 있어 기본적으로 활성화되어 있지 않다. 현재 OBI는 PostgreSQL, MySQL,
MSSQL 바이너리 프로토콜을 네이티브로 지원한다.

### 스팬에 대한 HTTP 헤더 및 본문 보강 {#http-header-enrichment-for-spans}

OBI는 `ebpf.payload_extraction.http.enrichment` 구성 섹션을 통해 선택한 HTTP
헤더와 선택한 HTTP 본문 필드를 스팬에 첨부할 수 있다. 이는 애플리케이션을
수동으로 계측하지 않고도 비즈니스 또는 라우팅 헤더를 트레이스로 전달하고 싶을 때
유용하다.

보강 엔진은 규칙 기반이다.

- HTTP 헤더 및 본문 보강을 활성화하려면 `enabled: true`로 설정한다.
- YAML에서 `policy.default_action.headers`와 `policy.default_action.body`를
  사용하여 일치하지 않는 헤더나 본문 콘텐츠를 포함할지 제외할지 정의한다. 둘 다
  기본값은 `exclude`이다.
- `obfuscate` 규칙을 사용하여 민감한 헤더 값이나 JSON 본문 필드가 스팬에
  첨부되기 전에 가린다.
- 규칙은 순서대로 평가된다.

예를 들면 다음과 같다.

```yaml
ebpf:
  buffer_sizes:
    http: 8192
  payload_extraction:
    http:
      enrichment:
        enabled: true
        policy:
          default_action:
            headers: exclude
            body: exclude
          obfuscation_string: '***'
        rules:
          - action: obfuscate
            type: headers
            scope: all
            match:
              patterns:
                - Authorization
              case_sensitive: false
          - action: include
            type: headers
            scope: all
            match:
              patterns:
                - Content-Type
                - X-Custom-*
                - X-Dice-Roll
              case_sensitive: false
          - action: include
            type: body
            scope: request
            match:
              methods: [POST]
              url_path_patterns:
                - /v1/chat/completions
          - action: obfuscate
            type: body
            scope: request
            match:
              methods: [POST]
              url_path_patterns:
                - /v1/chat/completions
              obfuscation_json_paths:
                - $.messages[*].content
```

다음 환경 변수는 전역 보강 동작을 제어한다.

- `OTEL_EBPF_HTTP_ENRICHMENT_ENABLED`
- `OTEL_EBPF_HTTP_ENRICHMENT_OBFUSCATION_STRING`

`policy.default_action.headers`와 `policy.default_action.body` 설정은 YAML에서만
구성되며, 이 기본값에 대한 환경 변수는 없다.

규칙 자체는 YAML에서 구성된다. 헤더 규칙은 `match.patterns`와 선택적
`case_sensitive`를 사용한다. 본문 규칙은 `match.url_path_patterns`,
`match.methods`, `match.obfuscation_json_paths`를 사용한다.

본문 추출에는 HTTP 페이로드 캡처가 필요하다. OBI가 보강하려는 요청 또는 응답
바이트를 캡처할 수 있도록 `ebpf.buffer_sizes.http`를 늘린다. 이 제한은 요청과
응답에 독립적으로 적용된다.

## 인스턴스 ID 데코레이션 {#instance-id-decoration}

YAML 섹션: `attributes.instance_id`

OBI는 각 계측된 애플리케이션을 식별하는 고유한 인스턴스 ID 문자열로 메트릭과
트레이스를 데코레이션한다. 기본적으로 OBI는 OBI를 실행하는 호스트 이름(컨테이너
또는 Pod 이름일 수 있음) 뒤에 계측된 프로세스의 PID를 붙여 사용한다.
`attributes` 최상위 섹션 아래의 `instance_id` YAML 하위 섹션에서 인스턴스 ID가
구성되는 방식을 재정의할 수 있다.

예를 들면 다음과 같다.

```yaml
attributes:
  instance_id:
    dns: false
```

| YAML<br>환경 변수                            | 설명                                                                                                                                                                  | 유형    | 기본값  |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | ------- |
| `dns`<br>`OTEL_EBPF_HOSTNAME_DNS_RESOLUTION` | `true`이면 OBI는 네트워크 DNS에 대해 로컬 호스트 이름을 확인하려고 시도한다. `false`이면 로컬 이름을 사용한다. 자세한 내용은 [dns 섹션](#dns)을 참고한다.             | boolean | true    |
| `override_hostname`<br>`OTEL_EBPF_HOSTNAME`  | 설정하면 OBI는 제공된 문자열을 인스턴스 ID의 호스트 부분으로 사용한다. DNS 확인을 재정의한다. 자세한 내용은 [호스트 이름 재정의 섹션](#override-hostname)을 참고한다. | string  | (unset) |

### DNS {#dns}

`true`이면 OBI는 네트워크 DNS에 대해 로컬 호스트 이름을 확인하려고 시도한다.
`false`이면 로컬 호스트 이름을 사용한다.

### 호스트 이름 재정의 {#override-hostname}

설정하면 OBI는 호스트 이름을 확인하려고 시도하는 대신 제공된 문자열을 인스턴스
ID의 호스트 부분으로 사용한다. 이 옵션은 `dns`보다 우선한다.

## 쿠버네티스 데코레이터 {#kubernetes-decorator}

YAML 섹션: `attributes.kubernetes`

YAML 구성의 `attributes.kubernetes` 섹션이나 환경 변수로 이 구성 요소를 구성할
수 있다.

이 기능을 활성화하려면 OBI Pod에 추가 권한을 제공해야 한다.
["쿠버네티스에서 OBI 실행" 페이지의 "쿠버네티스 메타데이터 데코레이션 구성 섹션"](../../setup/kubernetes/)을
참고한다.

이 옵션을 `true`로 설정하면 OBI는 쿠버네티스 메타데이터로 메트릭과 트레이스를
데코레이션한다. `false`로 설정하면 OBI는 쿠버네티스 메타데이터 데코레이터를
비활성화한다. `autodetect`로 설정하면 OBI는 자신이 쿠버네티스 내부에서 실행
중인지 감지하려고 시도하고, 그렇다면 메타데이터 데코레이션을 활성화한다.

예를 들면 다음과 같다.

```yaml
attributes:
  kubernetes:
    enable: true
```

| YAML<br>환경 변수                                                           | 설명                                                                                                                                                                                                  | 유형           | 기본값         |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | -------------- |
| `enable`<br>`OTEL_EBPF_KUBE_METADATA_ENABLE`                                | 쿠버네티스 메타데이터 데코레이션을 활성화하거나 비활성화한다. 쿠버네티스에서 실행 중이면 활성화하도록 `autodetect`로 설정한다. 자세한 내용은 [쿠버네티스 활성화 섹션](#enable-kubernetes)을 참고한다. | boolean/string | `autodetect`   |
| `cluster_name`<br>`OTEL_EBPF_KUBE_CLUSTER_NAME`                             | 감지된 쿠버네티스 클러스터 이름을 재정의한다.                                                                                                                                                         | string         | (empty)        |
| `kubeconfig_path`<br>`KUBECONFIG`                                           | 쿠버네티스 구성 파일의 경로. 자세한 내용은 [쿠버네티스 구성 경로 섹션](#kubernetes-configuration-path)을 참고한다.                                                                                    | string         | ~/.kube/config |
| `disable_informers`<br>`OTEL_EBPF_KUBE_DISABLE_INFORMERS`                   | 비활성화할 인포머 목록(`node`, `service`). 자세한 내용은 [인포머 비활성화 섹션](#disable-informers)을 참고한다.                                                                                       | string         | (empty)        |
| `meta_restrict_local_node`<br>`OTEL_EBPF_KUBE_META_RESTRICT_LOCAL_NODE`     | 메타데이터를 로컬 노드로만 제한한다. 자세한 내용은 [로컬 노드로 메타데이터 제한 섹션](#meta-restrict-local-node)을 참고한다.                                                                          | boolean        | false          |
| `informers_sync_timeout`<br>`OTEL_EBPF_KUBE_INFORMERS_SYNC_TIMEOUT`         | 시작하기 전에 쿠버네티스 메타데이터를 기다리는 최대 시간. 자세한 내용은 [인포머 동기화 타임아웃 섹션](#informers-sync-timeout)을 참고한다.                                                            | Duration       | 30s            |
| `reconnect_initial_interval`<br>`OTEL_EBPF_KUBE_RECONNECT_INITIAL_INTERVAL` | 연결이 끊긴 후 쿠버네티스 API에 재연결하기 전의 초기 지연. 자세한 내용은 [재연결 초기 간격 섹션](#reconnect-initial-interval)을 참고한다.                                                             | Duration       | 5s             |
| `informers_resync_period`<br>`OTEL_EBPF_KUBE_INFORMERS_RESYNC_PERIOD`       | 모든 쿠버네티스 메타데이터를 주기적으로 재동기화한다. 자세한 내용은 [인포머 재동기화 주기 섹션](#informers-resynchronization-period)을 참고한다.                                                      | Duration       | 30m            |
| `meta_cache_address`<br>`OTEL_EBPF_KUBE_META_CACHE_ADDRESS`                 | 쿠버네티스 메타데이터를 가져올 외부 `k8s-cache` 서비스의 주소. 자세한 내용은 [메타 캐시 주소 섹션](#meta-cache-address)을 참고한다.                                                                   | string         | (empty)        |
| `service_name_template`<br>`OTEL_EBPF_SERVICE_NAME_TEMPLATE`                | 서비스 이름을 위한 Go 템플릿. 자세한 내용은 [서비스 이름 템플릿 섹션](#service-name-template)을 참고한다.                                                                                             | string         | (empty)        |

### 쿠버네티스 활성화 {#enable-kubernetes}

쿠버네티스 환경에서 OBI를 실행하는 경우, 표준 오픈텔레메트리 레이블로 트레이스와
메트릭을 데코레이션하도록 구성할 수 있다.

- `k8s.namespace.name`
- `k8s.deployment.name`
- `k8s.statefulset.name`
- `k8s.replicaset.name`
- `k8s.daemonset.name`
- `k8s.node.name`
- `k8s.pod.name`
- `k8s.container.name`
- `k8s.pod.uid`
- `k8s.pod.start_time`
- `k8s.cluster.name`
- `k8s.owner.name`

`cluster_name`이 비어 있으면 OBI는 쿠버네티스 노드 레이블, OpenShift
Infrastructure 커스텀 리소스, 그리고 Amazon EC2, Google Cloud, Microsoft Azure의
클라우드 제공자 메타데이터를 이 순서대로 시도한다. 이 소스 중 어느 것도 이름을
제공하지 않으면 속성은 비어 있는 상태로 유지된다.

### 쿠버네티스 구성 경로 {#kubernetes-configuration-path}

이는 표준 쿠버네티스 구성 환경 변수이다. 쿠버네티스 클러스터와 통신하기 위한
쿠버네티스 구성을 OBI가 어디에서 찾을지 알려주는 데 사용한다. 일반적으로 이 값을
변경할 필요는 없다.

### 인포머 비활성화 {#disable-informers}

허용되는 값은 `node`와 `service`를 포함할 수 있는 목록이다.

이 옵션을 사용하면 네트워크 메트릭이나 애플리케이션 메트릭 및 트레이스를
데코레이션하는 데 필요한 메타데이터를 얻기 위해 쿠버네티스 API를 지속적으로
수신하는 일부 쿠버네티스 인포머를 선택적으로 비활성화할 수 있다.

매우 큰 클러스터에서 OBI를 DaemonSet으로 배포하면, 여러 인포머를 생성하는 모든
OBI 인스턴스가 쿠버네티스 API에 과부하를 줄 수 있다.

일부 인포머를 비활성화하면 보고되는 메타데이터가 불완전해지지만 쿠버네티스 API의
부하는 줄어든다.

Pods 인포머는 비활성화할 수 없다. 이를 비활성화하려면 전체 쿠버네티스 메타데이터
데코레이션을 비활성화한다.

### 로컬 노드로 메타데이터 제한 {#meta-restrict-local-node}

true이면 OBI는 OBI 인스턴스가 실행되는 노드의 Pod 및 Node 메타데이터만 저장한다.

이 옵션은 메타데이터를 저장하는 데 사용되는 메모리를 줄이지만, 네트워크 바이트나
서비스 그래프 메트릭 같은 일부 메트릭에는 다른 노드에 있는 대상 Pod의
메타데이터가 포함되지 않는다.

### 인포머 동기화 타임아웃 {#informers-sync-timeout}

이는 OBI가 메트릭과 트레이스를 데코레이션하기 시작하기 전에 모든 쿠버네티스
메타데이터를 얻기 위해 기다리는 최대 시간이다. 이 타임아웃에 도달하면 OBI는
정상적으로 시작되지만, 모든 쿠버네티스 메타데이터가 백그라운드에서 업데이트될
때까지 메타데이터 속성이 불완전할 수 있다.

### 재연결 초기 간격 {#reconnect-initial-interval}

OBI가 쿠버네티스 API와의 연결을 잃으면, 이 값은 연결을 재시도하기 전의 초기
지연을 제어한다.

불안정하거나 과부하된 API 서버에 대한 재연결 압력을 줄이려면 이 값을 늘린다.
일시적인 API 중단 후 더 빠른 복구가 필요할 때는 값을 줄인다.

### 인포머 재동기화 주기 {#informers-resynchronization-period}

OBI는 리소스 메타데이터에 대한 모든 업데이트를 즉시 받는다. 또한 OBI는 이
속성으로 지정한 빈도로 모든 쿠버네티스 메타데이터를 주기적으로 재동기화한다.
값이 높을수록 쿠버네티스 API 서비스의 부하가 줄어든다.

### 메타 캐시 주소 {#meta-cache-address}

설정하면 OBI는 쿠버네티스 API 서버에 대해 자체 인포머를 실행하는 대신 외부
`k8s-cache` 서비스에서 gRPC를 통해 쿠버네티스 메타데이터를 가져온다. 이는
쿠버네티스 API 과부하를 방지하기 위해 큰 클러스터와 DaemonSet 배포에서 권장된다.

### 서비스 이름 템플릿 {#service-name-template}

Go 템플릿을 사용하여 서비스 이름을 템플릿화할 수 있다. 이를 통해 조건부이거나
확장된 서비스 이름을 만들 수 있다.

템플릿에서 사용할 수 있는 컨텍스트는 다음과 같다.

```text
Meta: (*informer.ObjectMeta)
  Name: (string)
  Namespace: (string)
  Labels:
    label1: lv1
    label2: lv2
  Annotations:
    Anno1: av1
    Anno2: av2
  Pod: (*PodInfo)
  ...

ContainerName: (string)
```

전체 객체와 구조는 `kubecache informer.pb.go` 소스 파일에서 찾을 수 있다.

서비스 이름 템플릿 예시:

```go
{{- .Meta.Namespace }}/{{ index .Meta.Labels "app.kubernetes.io/name" }}/{{ index .Meta.Labels "app.kubernetes.io/component" -}}{{ if .ContainerName }}/{{ .ContainerName -}}{{ end -}}
```

또는

```go
{{- .Meta.Namespace }}/{{ index .Meta.Labels "app.kubernetes.io/name" }}/{{ index .Meta.Labels "app.kubernetes.io/component" -}}
```

이 예시에서는 첫 번째 줄만 사용되며, 서비스 이름에 공백이 생기지 않도록
잘라낸다.

## 추가 그룹 속성 {#extra-group-attributes}

OBI는 `extra_group_attributes` 구성을 사용하여 커스텀 속성으로 메트릭을 향상할
수 있게 한다. 이를 통해 표준 집합을 넘어서는 추가 메타데이터를 메트릭에 포함할
수 있는 유연성이 생긴다.

이 기능을 사용하려면 그룹 이름과 해당 그룹에 포함할 속성 목록을 지정한다.

현재는 `k8s_app_meta` 그룹만 지원된다. 이 그룹에는 Pod 이름, 네임스페이스,
컨테이너 이름, Pod UID 등 쿠버네티스 관련 메타데이터가 포함된다.

예시 구성:

```yaml
attributes:
  kubernetes:
    enable: true
  extra_group_attributes:
    k8s_app_meta: ['k8s.app.version']
```

이 예시에서:

- `extra_group_attributes > k8s_app_meta` 블록에 `k8s.app.version`을 추가하면
  `k8s.app.version` 레이블이 메트릭에 나타난다.
- 쿠버네티스 매니페스트에서 접두사 `resource.opentelemetry.io/`와 접미사
  `k8s.app.version`을 가진 어노테이션을 정의할 수도 있으며, 이 어노테이션은
  메트릭에 자동으로 포함된다.

다음 표는 기본 그룹 속성을 설명한다.

| 그룹           | 레이블                 |
| -------------- | ---------------------- |
| `k8s_app_meta` | `k8s.namespace.name`   |
| `k8s_app_meta` | `k8s.pod.name`         |
| `k8s_app_meta` | `k8s.container.name`   |
| `k8s_app_meta` | `k8s.deployment.name`  |
| `k8s_app_meta` | `k8s.replicaset.name`  |
| `k8s_app_meta` | `k8s.daemonset.name`   |
| `k8s_app_meta` | `k8s.statefulset.name` |
| `k8s_app_meta` | `k8s.node.name`        |
| `k8s_app_meta` | `k8s.pod.uid`          |
| `k8s_app_meta` | `k8s.pod.start_time`   |
| `k8s_app_meta` | `k8s.cluster.name`     |
| `k8s_app_meta` | `k8s.owner.name`       |

그리고 다음 표는 메트릭과 그에 연관된 그룹을 설명한다.

| 그룹           | OTel 메트릭                           | Prom 메트릭                                   |
| -------------- | ------------------------------------- | --------------------------------------------- |
| `k8s_app_meta` | `process.cpu.utilization`             | `process_cpu_utilization_ratio`               |
| `k8s_app_meta` | `process.cpu.time`                    | `process_cpu_time_seconds_total`              |
| `k8s_app_meta` | `process.memory.usage`                | `process_memory_usage_bytes`                  |
| `k8s_app_meta` | `process.memory.virtual`              | `process_memory_virtual_bytes`                |
| `k8s_app_meta` | `process.disk.io`                     | `process_disk_io_bytes_total`                 |
| `k8s_app_meta` | `messaging.client.operation.duration` | `messaging_client_operation_duration_seconds` |
| `k8s_app_meta` | `messaging.process.duration`          | `messaging_process_duration_seconds`          |
| `k8s_app_meta` | `http.server.request.duration`        | `http_server_request_duration_seconds`        |
| `k8s_app_meta` | `http.server.request.body.size`       | `http_server_request_body_size_bytes`         |
| `k8s_app_meta` | `http.server.response.body.size`      | `http_server_response_body_size_bytes`        |
| `k8s_app_meta` | `http.client.request.duration`        | `http_client_request_duration_seconds`        |
| `k8s_app_meta` | `http.client.request.body.size`       | `http_client_request_body_size_bytes`         |
| `k8s_app_meta` | `http.client.response.body.size`      | `http_client_response_body_size_bytes`        |
| `k8s_app_meta` | `rpc.client.call.duration`            | `rpc_client_call_duration_seconds`            |
| `k8s_app_meta` | `rpc.server.call.duration`            | `rpc_server_call_duration_seconds`            |
| `k8s_app_meta` | `db.client.operation.duration`        | `db_client_operation_duration_seconds`        |
| `k8s_app_meta` | `gpu.kernel.launch.calls`             | `gpu_kernel_launch_calls_total`               |
| `k8s_app_meta` | `gpu.kernel.grid.size`                | `gpu_kernel_grid_size_total`                  |
| `k8s_app_meta` | `gpu.kernel.block.size`               | `gpu_kernel_block_size_total`                 |
| `k8s_app_meta` | `gpu.memory.allocations`              | `gpu_memory_allocations_bytes_total`          |
