---
title: OBI Config v2 참조
linkTitle: Config v2 참조
description:
  Config v2로 독립형(standalone) OBI 또는 OBI 컬렉터 리시버를 구성하는 방법을
  알아본다.
weight: 3
# prettier-ignore
cSpell:ignore: Aerospike jsonrpc ollama openai qwen rerank sattributes SIGUSR sqlpp
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

Config v2는 OBI v0.11.0 이상에서 사용할 수 있다. Config v2는
오픈텔레메트리(OpenTelemetry) 선언적 구성(declarative configuration) 구조를
사용한다. 리소스, 샘플링, 익스포터 같은 공통 설정은 문서의 루트에 유지되고, OBI
전용 설정은 `extensions.obi` 아래에 그룹화된다.

이미 Config v1 파일이 있다면, 직접 다시 작성하는 대신
[Config v1에서 v2로 마이그레이션하는 가이드](../migrate-to-config-v2/)를
사용한다.

## 구성 구조 선택 {#choose-a-configuration-structure}

구성을 어떻게 구조화하는지는 OBI를 어떻게 실행하는지에 따라 달라진다.

- **독립형 OBI**: 완전한 오픈텔레메트리 선언적 구성 문서를 사용한다. 공통
  오픈텔레메트리 설정은 문서의 루트에, OBI 설정은 `extensions.obi` 아래에
  정의한다.
- **OBI 컬렉터 리시버**: OBI 캡처 설정을 `receivers.obi` 바로 아래에 정의한다.
  리소스 보강, 처리, 내보내기를 구성하려면 컬렉터 파이프라인을 사용한다.

## 독립형 OBI 구성 {#configure-standalone-obi}

다음 예시는 실행 파일 하나를 계측하고 디버깅을 위해 캡처된 스팬을 표준 출력에
인쇄한다. 이 구성을 프로덕션에서 사용하기 전에, 실행 파일 경로를 바꾸고,
`debug_trace_output`을 제거하며, `tracer_provider` 아래에 OTLP 익스포터를
구성한다.

```yaml
file_format: '1.0'

extensions:
  obi:
    version: '2.0'
    capture:
      policy:
        default_action: exclude
      rules:
        - action: include
          match:
            process:
              exe_path_glob: ['/path/to/your/application']
    daemon:
      logging:
        debug_trace_output: text
```

OBI를 시작하기 전에 구성 파일을 검증한다.

```sh
obi config validate ./obi-v2.yaml
```

### 구성 구조 {#configuration-structure}

```yaml
file_format: '1.0'
log_level: info

resource: {}
tracer_provider: {}
meter_provider: {}

extensions:
  obi:
    version: '2.0'
    capture: {}
    enrich: {}
    correlation: {}
    daemon: {}
```

두 version 필드는 모두 필수이지만, 서로 다른 스키마를 식별한다.

- `file_format: "1.0"`은 오픈텔레메트리 선언적 구성 스키마를 식별한다.
- `extensions.obi.version: "2.0"`은 OBI 구성 스키마를 식별한다. 현재 `"2.0"`이
  유일하게 지원되는 값이다.

두 필드 중 어느 것도 OBI 릴리스 버전으로 설정하지 않는다.

### 지원되는 최상위 필드 {#supported-top-level-fields}

OBI v0.12.1은 다음 오픈텔레메트리 선언적 구성 필드를 지원한다.

| 필드                         | 지원 내용                                                                                                                     |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `file_format`                | 필수. 지원되는 값은 `"1.0"`이다.                                                                                              |
| `log_level`                  | OBI 로깅을 설정한다. trace와 debug 레벨은 `DEBUG`에, info는 `INFO`에, warning은 `WARN`에, error와 fatal은 `ERROR`에 매핑된다. |
| `resource`                   | `host.name`, `host.id`, `service.name`, `service.namespace` 이름의 문자열 속성을 지원한다.                                    |
| `tracer_provider.sampler`    | always-on, always-off, trace-ID-ratio, 그리고 이러한 샘플러의 단순 부모 기반(parent-based) 형식을 지원한다.                   |
| `tracer_provider.processors` | 하나의 OTLP 익스포터를 갖는 하나의 batch 프로세서를 지원한다.                                                                 |
| `meter_provider.readers`     | 최대 하나의 periodic OTLP reader와 하나의 Prometheus development pull reader를 지원한다.                                      |

예를 들어, 문자열 리소스 속성으로 고정된 서비스 아이덴티티를 설정한다.

```yaml
resource:
  attributes:
    - name: service.name
      value: checkout
    - name: service.namespace
      value: shop
```

독립형 구성을 검증할 때, OBI는 지원되지 않는 파이프라인 필드를 무시하지 않고
오류로 보고한다. v0.12.1에서는 `attribute_limits`, `instrumentation/development`
또는 `logger_provider`를 사용하지 않는다. 또한 `disabled: true`, 비어 있지 않은
`distribution`, 비어 있지 않은 `propagator`를 거부한다.

Config v2 OTLP/gRPC 및 OTLP/HTTP 익스포터 예시는
[익스포터 구성](../migrate-to-config-v2/#configure-exporters)을 참고한다. OBI가
텔레메트리를 내보내는 방식에 대한 일반적인 정보는
[데이터 내보내기 구성](../export-data/)을 참고한다.

## 워크로드 선택 {#select-workloads}

OBI가 계측할 워크로드를 지정하려면 `capture.policy`와 `capture.rules`를
사용한다. OBI는 정의한 순서대로 규칙을 평가한다.

```yaml
extensions:
  obi:
    version: '2.0'
    capture:
      policy:
        default_action: exclude
        match_order: first_match_wins
        min_process_age: 5s
      rules:
        - action: exclude
          name: exclude-system-namespaces
          match:
            kubernetes:
              namespace_glob: ['kube-system', 'monitoring']
        - action: include
          name: checkout-service
          match:
            process:
              open_ports: '8080,9090-9091'
              exe_path_glob: ['/srv/checkout-*']
```

`default_action`을 생략하면 OBI는 기본적으로 워크로드를 포함한다. 규칙에
일치하는 워크로드만 계측하려면 `default_action`을 `exclude`로 설정하고 하나
이상의 include 규칙을 추가한다.

`match_order`를 `first_match_wins` 또는 `last_match_wins`로 설정한다.
런타임에서는 제외가 항상 우선한다. `first_match_wins`에서는 exclude 규칙을
include 규칙 앞에 둔다. `last_match_wins`에서는 exclude 규칙을 include 규칙 뒤에
둔다.

`rules`를 설정하면, 이 목록은 OBI 및 컬렉터 바이너리, 일반적인 시스템
네임스페이스, 이미 OTLP를 내보내는 서비스에 대한 OBI의 내장 제외(built-in
exclusion)를 대체한다. 이는 모든 내장 제외를 제거하는 `rules: []`에도 적용된다.
여전히 필요한 제외는 보존한다. 마이그레이션 명령은 이러한 제외를 생성된 목록에
작성하므로, 대체할 의도가 아니라면 유지한다.

### 프로세스 매치 필드 {#process-match-fields}

| 필드                              | 값                                                             |
| --------------------------------- | -------------------------------------------------------------- |
| `open_ports`                      | `"8080,9090-9091"`과 같은 쉼표로 구분된 포트 및 범위           |
| `target_pids`                     | 프로세스 ID 배열                                               |
| `language_glob`, `language_regex` | 프로그래밍 언어 매치                                           |
| `cmd_args_glob`, `cmd_args_regex` | 명령줄 인수 매치                                               |
| `exe_path_glob`, `exe_path_regex` | 실행 파일 경로 매치                                            |
| `containers_only`                 | 컨테이너 워크로드만 매치                                       |
| `exports_otlp`                    | 주어진 `port`와 `protocol`에서 OTLP를 내보내는 프로세스를 매치 |

glob 필드에는 값의 배열을, 정규 표현식 필드에는 하나의 표현식을 제공한다.

### 쿠버네티스 매치 필드 {#kubernetes-match-fields}

| 필드                                       | 값                                     |
| ------------------------------------------ | -------------------------------------- |
| `namespace_glob`, `namespace_regex`        | 쿠버네티스 네임스페이스 매치           |
| `metadata_glob`, `metadata_regex`          | 쿠버네티스 메타데이터 필드와 매치의 맵 |
| `pod_labels`, `pod_labels_regex`           | Pod 레이블과 매치의 맵                 |
| `pod_annotations`, `pod_annotations_regex` | Pod 어노테이션과 매치의 맵             |

지원되는 메타데이터 키에는 pod, deployment, ReplicaSet, DaemonSet, StatefulSet,
Job, CronJob, owner, container 이름이 포함된다.

### 일치한 워크로드 정제 {#refine-a-matched-workload}

일치하는 워크로드에 대해 시그널 내보내기 및 HTTP 라우트 설정을 재정의하려면
include 규칙에서 `refine` 블록을 사용한다.

```yaml
extensions:
  obi:
    version: '2.0'
    capture:
      rules:
        - action: include
          name: staging
          match:
            kubernetes:
              namespace_glob: ['staging-*']
          refine:
            exports:
              traces: false
              metrics: true
        - action: include
          name: orders
          match:
            kubernetes:
              namespace_glob: ['orders']
          refine:
            http:
              routes:
                incoming:
                  patterns: ['/orders/{id}']
                  ignored_patterns: ['/health']
                  unmatched: path
```

v0.12.1에서 `refine`은 `exports`와 `http.routes`를 지원한다. 비어 있지 않은
`http.filters` 필드나 워크로드별 샘플링은 지원하지 않는다. 모든 워크로드에 대한
샘플링은 `tracer_provider.sampler`로 구성한다.

여러 규칙이 하나의 워크로드에 일치하면, 규칙은 앞선 규칙에서 생략한 정제를
상속하지 않는다. 규칙이 겹칠 수 있으면, 각 정제를 명시적으로 지정하고 그 결과
동작을 테스트한다.

## 캡처 구성 {#configure-capture}

OBI가 워크로드를 선택하고 텔레메트리를 캡처하는 방식을 구성하려면
`extensions.obi.capture`를 사용한다. 다음 설정은 독립형 OBI와 OBI 컬렉터
리시버에서 사용할 수 있다.

| 섹션              | 목적                                                                       |
| ----------------- | -------------------------------------------------------------------------- |
| `policy`, `rules` | 워크로드를 선택하고 워크로드별 정제를 적용한다.                            |
| `instrumentation` | 애플리케이션 프로토콜을 활성화하고 튜닝한다.                               |
| `runtimes`        | Go, Node.js, Java 런타임 계측을 제어한다.                                  |
| `network`         | 네트워크 흐름 및 TCP 통계 캡처를 구성한다.                                 |
| `limits`          | 카디널리티 및 메모리 가드레일을 설정한다.                                  |
| `engine`          | 배칭, PID 필터링, 컨텍스트 전파, 트래픽 제어 및 기타 eBPF 동작을 튜닝한다. |
| `safety`          | 필요한 시스템 capabilities를 강제한다.                                     |
| `channels`        | 내부 버퍼링과 백프레셔를 튜닝한다.                                         |
| `telemetry`       | OBI 리포터 캐시와 메트릭 보존을 튜닝한다.                                  |

### 프로토콜 계측 {#protocol-instrumentation}

`instrumentation` 아래에서 HTTP, gRPC, SQL, Redis, Kafka, MongoDB, Couchbase,
DNS, GPU, Aerospike 계측을 구성할 수 있다. 각 프로토콜에 대해 트레이스와
메트릭을 별도로 활성화한다.

```yaml
extensions:
  obi:
    version: '2.0'
    capture:
      instrumentation:
        http:
          enabled:
            traces: true
            metrics: true
        dns:
          enabled:
            traces: false
            metrics: true
```

HTTP 라우트는 수신 요청과 송신 요청에 대해 별도로 구성한다. `incoming` 및
`outgoing` 섹션은 모두 `patterns`, `ignored_patterns`, `ignore_mode`,
`unmatched`, `wildcard_char`, `max_path_segment_cardinality`를 받는다. 이러한
설정의 동작은 [라우트 구성](../routes-decorator/)을 참고한다.

Config v2는 프로토콜과 시그널별로 애플리케이션 필터를 독립적으로 적용한다. 예를
들어, 동일한 필터를 HTTP 메트릭이나 SQL 텔레메트리에 적용하지 않고 HTTP
트레이스를 필터링할 수 있다. 이러한 필터는
`capture.instrumentation.<protocol>.filters.traces`와 `.metrics` 아래에
정의한다.

네트워크 흐름 필터와 TCP 통계 필터는 v0.12.1에서 시그널별로 구분되지 않는다.
이러한 각 그룹에 대해, 트레이스와 메트릭에 동일한 필터 맵을 사용한다. 두 맵이
다르면 검증에서 오류를 보고한다.

HTTP 페이로드 추출을 활성화하려면 `payload_extraction.enabled`에 추출기를
추가한다. 지원되는 값은 `graphql`, `elasticsearch`, `aws`, `sqlpp`, `openai`,
`anthropic`, `gemini`, `qwen`, `bedrock`, `mcp`, `embedding`, `rerank`,
`retrieval`, `ollama`, `openai_compatible`, `jsonrpc`, `enrichment`이다.
활성화된 추출기를 구성하려면 해당하는 중첩 블록을 사용한다. 중첩 블록만으로는
추출기가 활성화되지 않는다.

### 런타임 계측 {#runtime-instrumentation}

Go 프로브, Node.js `SIGUSR1` 주입, Java 에이전트 연결(attachment)을 활성화하거나
비활성화하려면 `capture.runtimes`를 사용한다. Java 디버그 설정과 연결 타임아웃도
구성할 수 있다. OBI v0.12.1은 비어 있지 않은 런타임 `filter` 필드를 지원하지
않는다. 대신 캡처 규칙을 사용하여 워크로드를 선택한다.

### 네트워크 옵저버빌리티 {#network-observability}

네트워크 흐름 텔레메트리를 구성하려면 `capture.network.capture`를, TCP 통계를
구성하려면 `capture.network.stats`를 사용한다. TCP 통계의 `features` 목록은
`tcp_rtt`, `tcp_failed_connections`, `tcp_retransmits`, `tcp_io`를 지원한다.

`tcp_io`는 다른 기능보다 훨씬 많은 이벤트를 생성할 수 있으므로, 전송별 및 수신별
통계가 필요할 때만 활성화한다. 배포 및 메트릭 세부 사항은
[네트워크 옵저버빌리티](../../network/)를 참고한다.

## 독립형 전용 기능 구성 {#configure-standalone-only-features}

OBI를 독립형 프로세스로 실행할 때는 `extensions.obi` 아래에서 다음 섹션도 사용할
수 있다.

- 쿠버네티스 메타데이터, 서비스 이름 지정, 속성 보강을 구성하려면 `enrich`를
  사용한다. 쿠버네티스 모드를 `autodetect`, `enabled` 또는 `disabled`로
  설정한다.
- 애플리케이션 로그의 트레이스 컨텍스트 어노테이션을 구성하려면 `correlation`을
  사용한다. [트레이스와 로그 상관관계](../../trace-log-correlation/)를 참고한다.
- 로깅 출력, 프로파일링, 정상 종료(graceful shutdown), 내부 메트릭, 독립형
  Prometheus 메트릭 셰이핑을 구성하려면 `daemon`을 사용한다. 로깅 상세 수준은
  최상위 `log_level` 필드로 설정한다.

## 컬렉터 리시버 구성 {#collector-receiver-configuration}

컬렉터 리시버 구성에서는, 독립형 구성에서 `extensions.obi.capture` 아래에 있는
필드를 `receivers.obi` 바로 아래 `version` 옆에 배치한다. `capture` 레벨은
포함하지 않는다. 예를 들어, 다음 YAML은 OBI 리시버 구성 요소 본문이다.

```yaml
version: '2.0'
policy:
  default_action: exclude
rules:
  - action: include
    match:
      process:
        open_ports: '8080'
instrumentation:
  http:
    enabled:
      traces: true
      metrics: true
```

리시버 구성 요소 본문을 별도의 파일에 저장하고 검증한다.

```sh
obi config validate --mode=receiver ./obi-receiver-v2.yaml
```

검증이 성공하면, 구성 요소 본문을 컬렉터 구성의 `receivers.obi` 아래에 복사한다.
그런 다음 적절한 트레이스 및 메트릭 파이프라인에 `obi`를 추가한다.

독립형 전용 `enrich`, `correlation` 또는 `daemon` 섹션은 리시버 구성에 추가하지
않는다. 보강에는 `k8sattributes` 같은 컬렉터 프로세서를, 운영 설정에는 컬렉터
서비스 텔레메트리를, 데이터 내보내기에는 컬렉터 익스포터를 사용한다. 전체 설정은
[OBI를 컬렉터 리시버로 실행하기](../collector-receiver/)를 참고한다.

## 환경 변수 {#environment-variables}

OBI는 구성 파일을 읽을 때, YAML을 파싱하기 전에 다음 환경 변수 표현식을
확장한다.

- `${VAR}` 및 `${env:VAR}`
- `${VAR:-fallback}` 및 `${env:VAR:-fallback}`

동등한 `$()` 형식을 사용할 수도 있다. 표현식을 리터럴 텍스트로 유지하려면 앞에
`$`를 하나 더 붙인다.

OBI는 Config v1 환경 변수 이름을 Config v2 필드로 자동 매핑하지 않는다. 환경
재정의를 보존하려면,
[환경 재정의 마이그레이션](../migrate-to-config-v2/#migrate-environment-overrides)에서
설명한 대로 해당하는 Config v2 필드에 치환 표현식을 추가한다.

## 구성 검증 {#validate-a-configuration}

배포에 맞는 검증 모드를 사용한다. 명령은 지원되지 않는 필드와 충돌하는 설정을
보고한다.

```sh
# Standalone document
obi config validate ./obi-v2.yaml

# Receiver component body
obi config validate --mode=receiver ./obi-receiver-v2.yaml
```

검증 명령은 OBI를 시작하거나, eBPF 프로그램을 연결(attach)하거나, 익스포터에
연결하거나, 실행 중인 커널을 확인하지 않는다. 검증이 성공하면, 카나리(canary)
배포에서 구성을 테스트한다.
