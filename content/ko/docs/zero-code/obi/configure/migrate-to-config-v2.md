---
title: OBI Config v1에서 Config v2로 마이그레이션
linkTitle: Config v2로 마이그레이션
description: OBI Config v1 파일을 Config v2로 안전하게 마이그레이션하는 방법을 알아본다.
weight: 4
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

OBI v0.11.0 이상에는 구성을 마이그레이션하고 검증하는 명령이 포함되어 있다. 한
번에 하나의 배포만 마이그레이션하고, 생성된 Config v2 파일을 검증한 다음,
롤아웃을 계속 진행하기 전에 테스트한다.

이 가이드는 독립형(standalone) OBI와 OBI 컬렉터 리시버(Collector receiver)
구성을 마이그레이션하는 방법을 설명한다. Config v2 구조와 지원되는 필드는
[Config v2 참조](../config-v2/)를 참고한다.

## 마이그레이션하기 전에 {#before-you-migrate}

마이그레이션하기 전에 롤백 계획을 준비한다.

1. 현재 Config v1 파일을 저장한다. 배포에서 사용하는 정확한 OBI 바이너리 버전,
   이미지 태그 또는 이미지 다이제스트를 기록한다.
2. 환경 변수, 명령줄 플래그, Helm 값, 쿠버네티스 매니페스트 또는 비밀 정보
   주입(secret injection)을 통해 제공되는 설정을 모두 나열한다. 마이그레이션
   명령은 제공한 파일만 읽는다.
3. 대상 OBI v0.12.1 이상 바이너리를 설치한다.
4. 카나리(canary) 배포를 위해 대표 인스턴스나 워크로드 하나를 선택한다.

마이그레이션 명령은 소스 파일의 치환 표현식(substitution expression)을
해석한다. 따라서 생성된 파일에는 비밀 값(secret)이 포함될 수 있다. 출력을
비공개 임시 디렉터리에 쓰고, 주의 깊게 검토하며, 비밀 값을 커밋하지 않는다.

## 독립형 구성 마이그레이션 {#migrate-a-standalone-configuration}

`obi config migrate` 명령은 하나의 Config v1 파일을 받는다. 생성된 Config v2
YAML은 표준 출력으로, 마이그레이션 보고서는 표준 오류로 쓴다.

```sh
umask 077
migration_dir="$(mktemp -d)"

obi config migrate ./obi-v1.yaml \
  > "${migration_dir}/obi-v2.yaml" \
  2> "${migration_dir}/migration-report.txt"
```

생성된 파일을 사용하기 전에 종료 상태(exit status)를 확인한다. 종료 상태 0은
성공을 나타낸다. 성공한 보고서는 `migrated v1 config to OBI config v2`로
시작한다. 오류는 `migration failed:`로 시작한다.

그런 다음 생성된 파일을 검증한다.

```sh
obi config validate "${migration_dir}/obi-v2.yaml"
```

마이그레이션 및 검증 명령은 다음 종료 상태를 사용한다.

| 상태  | 의미                                       |
| ----- | ------------------------------------------ |
| `0`   | 명령이 성공했다.                           |
| `1`   | 파싱, 검증 또는 마이그레이션이 실패했다.   |
| `2`   | 명령 구문이나 인수가 잘못되었다.           |

## 생성된 구조 이해하기 {#understand-the-generated-structure}

마이그레이션 명령은 Config v1 설정을 다음 Config v2 섹션으로 옮긴다.

| Config v1 설정                                        | Config v2 위치                                                     |
| ---------------------------------------------------- | ----------------------------------------------------------------- |
| 디스커버리 선택자                                     | `extensions.obi.capture.policy` 및 `extensions.obi.capture.rules` |
| 애플리케이션 프로토콜                                 | `extensions.obi.capture.instrumentation`                          |
| 런타임 계측                                           | `extensions.obi.capture.runtimes`                                 |
| 네트워크 캡처 및 통계                                 | `extensions.obi.capture.network`                                  |
| eBPF 및 배칭 제어                                     | `extensions.obi.capture.engine`                                   |
| 쿠버네티스 및 서비스 이름 보강                        | `extensions.obi.enrich`                                           |
| 트레이스-로그 상관관계                                | `extensions.obi.correlation`                                      |
| 프로세스 로깅, 프로파일링, 종료, 내부 메트릭          | `extensions.obi.daemon`                                           |
| 트레이스 샘플링 및 내보내기                           | 최상위 `tracer_provider`                                          |
| 메트릭 내보내기                                       | 최상위 `meter_provider`                                           |
| 리소스 속성                                           | 최상위 `resource`                                                 |

생성된 파일에는 Config v1 동작을 보존하는 데 필요한 경우 명시적 기본값이
포함된다. 카나리 배포에서 구성을 테스트하기 전까지는 이 값들을 변경하지 않는다.

## 동작 변경 사항 검토 {#review-behavior-changes}

### 캡처 기본값과 규칙 순서 {#capture-defaults-and-rule-order}

선택 필드가 없는 Config v1 파일은 애플리케이션 캡처를 비활성화한다. 반면 Config
v2 파일은 기본적으로 워크로드를 포함한다. Config v1 동작을 보존하기 위해, 소스
파일이 워크로드를 선택하지 않으면 마이그레이션 명령은 `default_action`을
`exclude`로 설정한다.

또한 마이그레이션 명령은 내장 제외(built-in exclusion)를 작성하며 Config v1
include 선택자의 순서를 뒤집을 수 있다. 생성된 순서는 Config v2 규칙 모델에서
Config v1 우선순위를 보존한다. 겹치는 규칙과 하나의 규칙에만 일치하는
워크로드를 테스트하기 전까지는 규칙 순서를 바꾸지 않는다.

`rules: []`를 명시적으로 설정하면 OBI는 내장 제외를 제거한다.

### 필터 {#filters}

Config v1은 애플리케이션 텔레메트리, 네트워크 텔레메트리, TCP 통계에 대해 각각
하나의 필터를 제공한다. 마이그레이션 명령은 각 Config v1 필터를 해당하는 모든
Config v2 필드로 복사하여, 생성된 구성이 원래 동작을 보존하도록 한다.

그 기준선을 확립한 후에는, Config v2에서 프로토콜과 시그널별로 애플리케이션
필터를 독립적으로 조정할 수 있다. 예를 들어 HTTP 트레이스 필터는 더 이상 HTTP
메트릭 필터나 SQL 필터와 일치할 필요가 없다. 이러한 변경은 카나리 배포에서
수행하고, 롤아웃하기 전에 텔레메트리 볼륨과 카디널리티(cardinality)를 비교한다.

네트워크 흐름 필터와 TCP 통계 필터는 v0.12.1에서 트레이스와 메트릭 간에 공유된
상태로 유지된다. 각 그룹 내에서 두 맵을 동일하게 유지한다. 두 맵이 다르면
검증에서 오류를 보고한다.

### HTTP 라우트 {#http-routes}

전역 Config v1 라우트 설정은 수신 및 송신 트래픽에 적용된다. 마이그레이션 명령은
이를 `capture.instrumentation.http.routes` 아래의 양방향으로 복사한다.

서비스별 라우트 패턴은 `rules[].refine.http.routes`로 이동한다. 특정 방향에
대해, 명시적인 서비스별 목록은 전역 목록을 대체하며, 빈 목록은 이를 지운다. 전역
패턴과 서비스별 패턴의 조합이 Config v1 상속 동작을 보존할 수 없으면
마이그레이션이 실패한다.

### 워크로드 정제 {#workload-refinements}

Config v1 선택자에는 내보내기 정제와 라우트 정제가 포함될 수 있다. Config
v2에서 include 규칙은 앞서 일치한 규칙에서 생략된 정제를 상속하지 않는다. 이
차이가 동작을 바꾸는 경우, 명시적 `exports` 또는 `routes` 필드와 생략된 필드가
섞인 선택자 목록에 대해서는 마이그레이션이 실패한다.

이러한 선택자를 마이그레이션하려면, 적용 가능한 모든 선택자에 각 정제를
명시적으로 지정한다. 조건부 상속에 의존하는 선택자는 각 Config v2 규칙이
독립적이도록 재구성한 다음, 겹치는 규칙을 모두 테스트한다.

## 익스포터 구성 {#configure-exporters}

Config v2는 최상위 오픈텔레메트리(OpenTelemetry) 섹션에서 텔레메트리
파이프라인을 정의한다. 마이그레이션 명령은 OTLP/gRPC 익스포터를 생성할 수 있다.
명령이 의미를 바꾸지 않고서는 엔드포인트나 프로토콜을 결정할 수 없으면
마이그레이션이 실패한다.

OTLP/gRPC 트레이스 익스포터에는 다음 구성을 사용한다.

```yaml
tracer_provider:
  processors:
    - batch:
        exporter:
          otlp_grpc:
            endpoint: http://collector:4317
            tls:
              insecure: true
```

OTLP/gRPC 메트릭 익스포터에는 다음 구성을 사용한다.

```yaml
meter_provider:
  readers:
    - periodic:
        exporter:
          otlp_grpc:
            endpoint: http://collector:4317
            tls:
              insecure: true
        interval: 60000
```

Config v1 파일이 OTLP over HTTP를 사용하는 경우, 먼저 다른 설정을
마이그레이션한다. 그런 다음 HTTP 익스포터를 수동으로 추가한다.

```yaml
tracer_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: http://collector:4318/v1/traces
            encoding: protobuf

meter_provider:
  readers:
    - periodic:
        exporter:
          otlp_http:
            endpoint: http://collector:4318/v1/metrics
            encoding: protobuf
```

OTLP over HTTP의 경우 `encoding`을 `json`으로 설정할 수도 있다. Config v2는
선언적 익스포터 헤더도 지원한다. 익스포터 인증 환경 변수는 런타임 입력으로
유지되며, 마이그레이션 명령은 그 값을 생성된 파일로 복사하지 않는다.

## 수동 변경이 필요한 설정 처리 {#handle-settings-that-need-manual-changes}

마이그레이션 명령이 설정을 보존할 수 없으면 실패하고 오류 메시지에 Config v1
필드를 식별한다. 명령을 다시 실행하기 전에 지원되지 않는 동작을 대체하거나
폐기한다. 다음 설정은 일반적으로 수동 변경이 필요하다.

| Config v1 설정                                                              | 해야 할 일                                                                                                                                                        |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `service_name`, `service_namespace`                                       | 독립형 OBI에서는 `service.name` 또는 `service.namespace`를 최상위 `resource` 속성으로 설정한다. OBI 컬렉터 리시버에서는 컬렉터 resource 프로세서를 사용한다.  |
| `prometheus_export.path`                                                  | 선택한 컬렉터 또는 Prometheus 익스포터가 지원하는 경로를 사용한다. Config v2에는 이에 대한 이식 가능한 필드가 없다.                                           |
| 커스텀이거나 빈 `discovery.excluded_linux_system_paths`                   | 경로 기반 제외를 `capture.rules`의 워크로드 제외로 대체한다. 해당하는 워크로드 제외가 필요 없으면 설정을 제거한다.                                            |
| `health_check.*`                                                          | 배포 헬스 체크나 컬렉터 헬스 체크 기능을 사용한다.                                                                                                            |
| `jvm_runtime_metrics.sampling_interval`                                   | 커스텀 간격을 제거하고 Config v2 JVM 샘플링 동작을 테스트한다.                                                                                                |
| `attributes.instance_id.dns`                                              | `service.instance.id`를 최상위 리소스 속성으로 설정하거나 인스턴스 아이덴티티 보강을 컬렉터 프로세서로 옮긴다.                                                |
| `ebpf.stats_wakeup_data_bytes`                                            | 커스텀 웨이크업 임계값을 제거하고 Config v2 기본값으로 성능을 테스트한다.                                                                                     |
| 선택자 `name`, `namespace`, 선택자별 메트릭 또는 선택자별 샘플러          | 지원되는 match 및 `refine` 필드로 선택자를 다시 설계한다. 정제가 다를 경우 명시적 규칙으로 분리한다.                                                          |
| 선택자 `exports.logs`                                                     | 선택자별 로그 내보내기 정제를 제거한다. 로그 상관관계 대상 워크로드를 선택하려면 캡처 규칙을 사용한다.                                                        |
| `ebpf.log_enricher.services`                                              | 별도의 로그 어노테이션 선택자를 대상 워크로드를 선택하는 캡처 규칙으로 대체한다.                                                                              |
| `sensitive_query_params`                                                  | 마이그레이션 전에 개인정보 보호 정책을 다시 설계한다. Config v2에는 이에 상응하는 필드가 없다.                                                                |
| Debug 익스포터 또는 지원되지 않는 샘플러                                  | 지원되는 OTLP 익스포터나 샘플러를 구성한다.                                                                                                                   |
| 활성화된 OTLP 계측 목록과 Prometheus 계측 목록이 다른 경우               | 마이그레이션하기 전에 프로토콜 메트릭 활성화를 일관되게 만든다.                                                                                               |

명령은 지원되지 않는 다른 메트릭 기능과 히스토그램, 익스포터 또는 Prometheus
설정을 개별적으로 보고한다. 마이그레이션을 다시 실행하기 전에 보고된 모든 필드를
검토하고 해결한다. 필드에 직접적인 대응 항목이 없으면, 지원되는 Config v2 또는
컬렉터 동작을 선택하고 카나리 배포에서 변경 사항을 검증한다.

또한 명령은 알 수 없는 Config v1 필드, 이미 Config v2를 사용하는 파일, 여러 YAML
문서를 포함하는 파일을 거부한다.

예를 들어, 의도적인 독립형 서비스 아이덴티티를 리소스 속성으로 매핑한다.

```yaml
resource:
  attributes:
    - name: service.name
      value: checkout
    - name: service.namespace
      value: shop
```

이 조각은 독립형 구성의 루트에만 추가한다. OBI 컬렉터 리시버에서는 대신 컬렉터
resource 프로세서를 사용한다.

## 환경 재정의 마이그레이션 {#migrate-environment-overrides}

마이그레이션 명령은 소스 파일을 읽지만 OBI 런타임 환경 재정의는 적용하지 않는다.
또한 OBI는 Config v1 환경 변수 이름을 Config v2 필드로 자동 매핑하지 않는다.

예를 들어 배포에서 `OTEL_EBPF_BPF_WAKEUP_LEN=999`를 설정하는 경우, 해당하는
Config v2 필드에 명시적 치환 표현식을 추가한다.

```yaml
extensions:
  obi:
    version: '2.0'
    capture:
      engine:
        batching:
          wakeup_len: ${OTEL_EBPF_BPF_WAKEUP_LEN:-500}
```

구성 파일에서 `${VAR}`, `${env:VAR}` 또는 이들의 `:-fallback` 형식을 사용할 수
있다. 동등한 `$()` 형식을 사용할 수도 있다. 표현식을 리터럴 텍스트로 유지하려면
앞에 `$`를 하나 더 붙인다.

명령줄 및 배포 수준 재정의도 같은 방식으로 검토한다. 각 값을 해당하는 Config v2
필드나 컬렉터 파이프라인 설정으로 옮긴다.

## 컬렉터 리시버 마이그레이션 {#migrate-a-collector-receiver}

전체 컬렉터 구성이 아니라 OBI 리시버 구성 요소 본문을 마이그레이션 명령에
전달한다.

```sh
obi config migrate --mode=receiver ./obi-receiver-v1.yaml \
  > ./obi-receiver-v2.yaml \
  2> ./migration-report.txt

obi config validate --mode=receiver ./obi-receiver-v2.yaml
```

명령은 Config v2 리시버 구성 요소 본문을 생성한다. 이 형식에서는 캡처 필드가
`capture` 레벨 없이 `version` 옆에 나타난다.

```yaml
version: '2.0'
policy:
  default_action: exclude
rules:
  - action: include
    match:
      process:
        open_ports: '8080'
```

검증이 성공하면, 구성 요소 본문을 컬렉터 구성의 `receivers.obi` 아래에
복사한다.

리시버 마이그레이션은 독립형 익스포터, 보강, 상관관계, 데몬 또는 내부 텔레메트리
필드를 받지 않는다. 동등한 동작은 컬렉터 익스포터, 프로세서, 익스텐션, 서비스
텔레메트리로 구성한다. 전체 파이프라인 예시는
[OBI를 컬렉터 리시버로 실행하기](../collector-receiver/)를 참고한다.

## 마이그레이션 검증 및 테스트 {#validate-and-test-the-migration}

`obi config validate`는 YAML 구조를 확인하고 OBI가 지정된 Config v2 필드를
지원하는지 검사한다. 이 명령은 OBI를 시작하거나, 익스포터에 연결하거나, eBPF
프로그램을 연결(attach)하거나, 커널 capabilities를 확인하지 않는다.

검증이 성공하면, Config v2를 한 인스턴스에 배포하고 Config v1 배포와 비교한다.

- 동일한 워크로드가 포함되고 제외되는지 확인한다.
- 트레이스, 메트릭, 라우트, 속성, 샘플링 동작을 확인한다.
- 익스포터 연결성과 인증을 확인한다.
- 텔레메트리 볼륨과 카디널리티를 비교한다.
- OBI 로그와 내부 메트릭에서 오류나 백프레셔(backpressure)를 확인한다.
- 겹치는 선택 규칙에 일치하는 워크로드를 실행해 본다.

카나리 테스트 동안 Config v1 파일과 이전 OBI 바이너리 또는 이미지를 사용할 수
있도록 유지한다. 동작이 다르면 구성과 OBI 버전을 롤백한다. 롤아웃을 계속
진행하기 전에 불일치를 해결한다.
