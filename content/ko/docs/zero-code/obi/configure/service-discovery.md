---
title: OBI 서비스 디스커버리 구성
linkTitle: 서비스 디스커버리
description:
  OBI 서비스 디스커버리(service discovery) 구성 요소가 계측할 프로세스를
  검색하는 방식을 구성하는 방법을 알아본다.
weight: 20
# prettier-ignore
cSpell:ignore: filestorecsi kube-node-lease kube-system rdns replicaset statefulset testserver volumepopulator
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2에서는
> `capture.policy`와 `capture.rules`를 사용한다. 자세한 내용은
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

`OTEL_EBPF_AUTO_TARGET_EXE`, `OTEL_EBPF_OPEN_PORT`,
`OTEL_EBPF_AUTO_TARGET_LANGUAGE`, `OTEL_EBPF_TARGET_PID` 환경 변수를 사용하면
단일 서비스나 관련 서비스 그룹을 계측하도록 OBI를 더 쉽게 구성할 수 있다.

일부 시나리오에서는 OBI가 여러 서비스를 계측한다. 예를 들어 노드의 모든
서비스를 계측하는 [쿠버네티스 DaemonSet](../../setup/kubernetes/)으로 실행되는
경우다. `discovery` YAML 섹션을 사용하면 OBI가 계측할 수 있는 서비스에 대해 더
세밀한 선택 기준(selection criteria)을 지정할 수 있다.

| YAML<br>환경 변수                                                                                                 | 설명                                                                                                                                                                                                                                                                                                                   | 유형             | 기본값                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| `instrument`                                                                                                        | 서비스마다 서로 다른 선택 기준을 지정하고, 보고되는 이름이나 네임스페이스를 재정의(override)한다. 자세한 내용은 [디스커버리 서비스](#discovery-services) 섹션을 참고한다.                                                                                                                                            | list of objects   | (unset)                                                                                                       |
| `exclude_instrument`                                                                                                | 계측에서 제외할 서비스에 대한 선택 기준을 지정한다. 옵저버빌리티 환경에서 흔히 볼 수 있는 서비스의 계측을 피하는 데 유용하다. 자세한 내용은 [계측에서 서비스 제외](#exclude-services-from-instrumentation) 섹션을 참고한다.                                                                                            | list of objects   | (unset)                                                                                                       |
| `default_exclude_instrument`                                                                                        | OBI 자신, 오픈텔레메트리 컬렉터, 특정 쿠버네티스 시스템 네임스페이스에서 실행되는 서비스의 계측을 비활성화한다. OBI가 자신과 컬렉터, 이 네임스페이스의 서비스를 계측하도록 허용하려면 빈 값으로 설정한다. 자세한 내용은 [계측에서 기본 제외 서비스](#default-exclude-services-from-instrumentation) 섹션을 참고한다. | list of objects   | Path: `{*/obi,obi,*otelcol,*otelcol-contrib,*otelcol-contrib[!/]*}` and certain Kubernetes system namespaces  |
| `skip_go_specific_tracers`<br>`OTEL_EBPF_SKIP_GO_SPECIFIC_TRACERS`                                                  | **eBPF** 트레이서가 계측할 실행 파일을 검사할 때 Go 관련 세부 사항의 감지를 비활성화한다. 트레이서는 일반적으로 덜 효율적인 범용 계측 방식으로 대체(fallback)한다. 자세한 내용은 [Go 특화 트레이서 건너뛰기](#skip-go-specific-tracers) 섹션을 참고한다.                                                              | boolean           | false                                                                                                          |
| `exclude_otel_instrumented_services`<br>`OTEL_EBPF_EXCLUDE_OTEL_INSTRUMENTED_SERVICES`                              | 이미 오픈텔레메트리로 계측된 서비스에 대한 OBI 계측을 비활성화한다. 자세한 내용은 [OTel로 계측된 서비스 제외](#exclude-otel-instrumented-services) 섹션을 참고한다.                                                                                                                                                    | boolean           | true                                                                                                           |
| `exclude_otel_instrumented_services_span_metrics`<br>`OTEL_EBPF_EXCLUDE_OTEL_INSTRUMENTED_SERVICES_SPAN_METRICS`    | 이미 오픈텔레메트리로 계측된 서비스에 대해 OBI 스팬 메트릭/서비스 그래프 메트릭 생성을 비활성화한다. 자세한 내용은 [OTel로 계측된 서비스 제외](#exclude-otel-instrumented-services) 섹션을 참고한다.                                                                                                                   | boolean           | false                                                                                                          |
| `process_context_poll_interval`<br>`OTEL_EBPF_PROCESS_CONTEXT_POLL_INTERVAL`                                       | 실험적인 `OTEL_CTX` 프로세스 컨텍스트 매핑을 통해 게시된 업데이트된 리소스 속성과 메타데이터를 OBI가 계측된 프로세스에서 얼마나 자주 확인하는지 설정한다. OBI가 프로세스를 처음 발견했을 때만 확인하려면 `0`으로 설정한다.                                                                                             | duration          | `1s`                                                                                                           |

## 디스커버리 서비스 {#discovery-services}

서비스 유형별로 서비스 이름, 네임스페이스, 기타 설정을 재정의할 수 있다.

| YAML                     | 설명                                                                                                    | 유형                       | 기본값  |
| ------------------------ | --------------------------------------------------------------------------------------------------------- | -------------------------- | ------- |
| `open_ports`             | 열려 있는(수신 대기 중인) 포트를 기준으로 계측할 프로세스를 선택한다. 자세한 내용은 [열린 포트](#open-ports)를 참고한다.                            | string                     | (unset) |
| `exe_path`               | 실행 파일 이름 경로를 기준으로 계측할 프로세스를 선택한다. 자세한 내용은 [실행 파일 경로](#executable-path)를 참고한다.                            | string (glob)               | (unset) |
| `languages`              | 감지된 프로그래밍 언어를 기준으로 프로세스를 선택한다. 자세한 내용은 [언어](#languages)를 참고한다.                                                | string (glob)               | (unset) |
| `cmd_args`               | 명령줄 인수를 기준으로 프로세스를 선택한다. 자세한 내용은 [명령줄 인수](#command-line-arguments)를 참고한다.                                       | string (glob)               | (unset) |
| `target_pids`            | PID를 기준으로 프로세스를 선택한다. 자세한 내용은 [대상 PID](#target-pids)를 참고한다.                                                             | list of integers            | (unset) |
| `containers_only`        | OCI 컨테이너에서 실행 중인, 계측할 프로세스를 선택한다. 자세한 내용은 [컨테이너 전용](#containers-only)을 참고한다.                                     | boolean                     | false   |
| `container_name`         | OCI 컨테이너 이름을 기준으로 서비스를 필터링한다. 자세한 내용은 [컨테이너 이름](#container-name)을 참고한다.                                          | string (glob)               | (unset) |
| `k8s_namespace`          | 쿠버네티스 네임스페이스를 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s 네임스페이스](#k8s-namespace)를 참고한다.                                    | string (glob)               | (unset) |
| `k8s_pod_name`           | 쿠버네티스 Pod를 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s Pod 이름](#k8s-pod-name)을 참고한다.                                              | string (glob)               | (unset) |
| `k8s_deployment_name`    | 쿠버네티스 Deployment를 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s Deployment 이름](#k8s-deployment-name)을 참고한다.                          | string (glob)               | (unset) |
| `k8s_replicaset_name`    | 쿠버네티스 ReplicaSet을 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s ReplicaSet 이름](#k8s-replicaset-name)을 참고한다.                          | string (glob)               | (unset) |
| `k8s_statefulset_name`   | 쿠버네티스 StatefulSet을 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s StatefulSet 이름](#k8s-statefulset-name)을 참고한다.                       | string (glob)               | (unset) |
| `k8s_daemonset_name`     | 쿠버네티스 DaemonSet을 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s DaemonSet 이름](#k8s-daemonset-name)을 참고한다.                             | string (glob)               | (unset) |
| `k8s_owner_name`         | 쿠버네티스 Pod 소유자(Deployment, ReplicaSet, DaemonSet, StatefulSet)를 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s 소유자 이름](#k8s-owner-name)을 참고한다. | string (glob)               | (unset) |
| `k8s_pod_labels`         | 쿠버네티스 Pod 레이블을 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s Pod 레이블](#k8s-pod-labels)을 참고한다.                                       | map[string]string (glob)    | (unset) |
| `k8s_pod_annotations`    | 쿠버네티스 Pod 어노테이션(annotation)을 기준으로 서비스를 필터링한다. 자세한 내용은 [K8s Pod 어노테이션](#k8s-pod-annotations)을 참고한다.                | map[string]string (glob)    | (unset) |

### 열린 포트 {#open-ports}

열려 있는(수신 대기 중인) 포트를 기준으로 계측할 프로세스를 선택한다. 이
속성은 쉼표로 구분된 포트 목록(예: `80`)과 포트 범위(예: `8000-8999`)를 받는다.
실행 파일이 목록의 포트 중 하나만 일치해도 OBI는 이를 일치하는 것으로 간주한다.

예를 들어 다음 속성을 지정하면:

```yaml
discovery:
  instrument:
    - open_ports: 80,443,8000-8999
```

OBI는 포트 80, 443, 또는 8000에서 8999 사이의 포트를 여는 모든 실행 파일을
선택한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

실행 파일이 여러 포트를 열면, OBI가 모든 애플리케이션 포트에서 HTTP/S 및 gRPC
요청을 모두 계측하도록 하려면 그 포트 중 하나만 지정하면 된다. 현재는 계측을
특정 포트로 노출된 메서드로만 제한할 수 없다.

### 실행 파일 경로 {#executable-path}

실행 파일 이름 경로를 기준으로 계측할 프로세스를 선택한다. 이 속성은 실행
파일이 파일 시스템에 위치한 디렉터리를 포함한 전체 실행 명령줄과 일치시킬
glob을 받는다.

OBI는 이 속성과 일치하는 실행 파일 경로를 가진 모든 프로세스를 계측하려고
시도한다. 예를 들어 `exe_path: *`로 설정하면 OBI는 호스트의 모든 실행 파일을
계측하려고 시도한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### 언어 {#languages}

감지된 프로그래밍 언어를 기준으로 프로세스를 선택한다. 이 속성은 정규화된 언어
이름(예: `go`, `java`, `python`, `nodejs`)에 대한 glob 매처를 받는다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - languages: go
```

이를 `exe_path`나 `open_ports` 같은 다른 선택자와 조합할 수 있으며, 이 경우
구성된 모든 선택자가 일치해야 한다.

### 명령줄 인수 {#command-line-arguments}

명령줄 인수를 기준으로 프로세스를 선택한다. 이 속성은 프로세스의 전체 명령줄
인수와 일치시킬 glob을 받는다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - cmd_args: '*--profile=prod*'
```

이 선택자는 동일한 `instrument` 항목의 다른 선택자와 조합할 수 있다.

### 대상 PID {#target-pids}

PID를 기준으로 프로세스를 선택한다. 계측해야 할 프로세스 ID를 이미 알고 있는
경우 이 선택자를 사용한다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - target_pids: [1234, 5678]
```

루트 레벨에서 `target_pids`를 사용하거나 `OTEL_EBPF_TARGET_PID=1234,5678`을
통해 PID 타겟팅을 전역으로 구성할 수도 있다.

### 컨테이너 전용 {#containers-only}

OCI 컨테이너에서 실행 중인, 계측할 프로세스를 선택한다. 이 확인을 수행하기
위해 OBI는 프로세스의 네트워크 네임스페이스를 검사하여 자신의 네트워크
네임스페이스와 비교한다. OBI에 네트워크 네임스페이스 검사를 수행할 충분한
권한이 없으면 이 옵션을 무시한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### 컨테이너 이름 {#container-name}

이 선택자 속성은 지정된 glob 패턴과 이름이 일치하는 OCI 컨테이너(예: Docker)에서
실행 중인 애플리케이션으로 계측을 제한한다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - container_name: '*testserver*'
    - container_name: 'my-app-*'
```

이 예시는 이름에 `testserver`가 포함되거나 `my-app-`로 시작하는 컨테이너에서
실행 중인 모든 프로세스를 검색(discover)한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s 네임스페이스 {#k8s-namespace}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 Namespace에서 실행
중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s Pod 이름 {#k8s-pod-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 Pod에서 실행 중인
애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s Deployment 이름 {#k8s-deployment-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 Deployment에서 실행
중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s ReplicaSet 이름 {#k8s-replicaset-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 ReplicaSet에서 실행
중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s StatefulSet 이름 {#k8s-statefulset-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 StatefulSet에서 실행
중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s DaemonSet 이름 {#k8s-daemonset-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 쿠버네티스 DaemonSet에서 실행
중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s 소유자 이름 {#k8s-owner-name}

이 선택자 속성은 지정된 glob과 이름이 일치하는 `Deployment`, `ReplicaSet`,
`DaemonSet`, `StatefulSet`이 소유한 Pod에서 실행 중인 애플리케이션으로 계측을
제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

### K8s Pod 레이블 {#k8s-pod-labels}

이 선택자 속성은 제공된 값을 glob으로 사용하여 일치하는 레이블을 가진 Pod에서
실행 중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - k8s_namespace: frontend
      k8s_pod_labels:
        instrument: obi
```

앞의 예시는 `frontend` 네임스페이스에 있는 Pod 중 값이 glob `obi`와 일치하는
`instrument` 레이블을 가진 모든 Pod를 검색한다.

### K8s Pod 어노테이션 {#k8s-pod-annotations}

이 선택자 속성은 제공된 값을 glob으로 사용하여 일치하는 어노테이션을 가진
Pod에서 실행 중인 애플리케이션으로 계측을 제한한다.

동일한 `instrument` 항목에 다른 선택자를 지정하면, 프로세스는 모든 선택자
속성과 일치해야 한다.

예를 들면 다음과 같다.

```yaml
discovery:
  instrument:
    - k8s_namespace: backend
      k8s_pod_annotations:
        obi.instrument: 'true'
```

앞의 예시는 `backend` 네임스페이스에 있는 Pod 중 값이 glob `true`와 일치하는
`obi.instrument` 어노테이션을 가진 모든 Pod를 검색한다.

## 계측에서 서비스 제외 {#exclude-services-from-instrumentation}

`exclude_instrument` 섹션을 사용하면 계측에서 제외할 서비스에 대한 선택
기준을 지정할 수 있다. 이 섹션은 이 문서의
[디스커버리 서비스](#discovery-services) 섹션에서 설명한 것과 동일한 정의
형식을 따른다.

이 옵션은 옵저버빌리티 환경에서 흔히 볼 수 있는 서비스의 계측을 피하는 데
도움이 된다. 예를 들어 이 옵션을 사용하여 Prometheus 계측을 제외할 수 있다.

### 예시: 특정 네임스페이스 제외 {#example-exclude-specific-namespaces}

```yaml
discovery:
  instrument:
    - k8s_namespace: '*' # Instrument all namespaces
  exclude_instrument:
    - k8s_namespace: development # Except development namespace
    - k8s_namespace: staging # And staging namespace
```

### 예시: 레이블로 서비스 제외 {#example-exclude-services-by-labels}

```yaml
discovery:
  rules:
    - match:
        k8s_namespace: production
      exclude:
        k8s_pod_labels:
          skip-instrumentation: 'true'
```

이 예시에서 `skip-instrumentation`은 사용자 정의 쿠버네티스 Pod 레이블이다.
조직의 레이블링 규칙에 따라 일치나 제외를 위한 커스텀 레이블 키와 값을 자유롭게
사용할 수 있다.

### 예시: 특정 실행 파일 제외 {#example-exclude-specific-executables}

```yaml
discovery:
  rules:
    - match:
        open_ports: 80,443,8080
      exclude:
        exe_path: '*prometheus*'
        exe_path: '*grafana*'
```

## 계측에서 기본 제외 서비스 {#default-exclude-services-from-instrumentation}

`default_exclude_instrument` 섹션은 OBI 자신(셀프 계측), 오픈텔레메트리
컬렉터, 기타 옵저버빌리티 구성 요소의 계측을 비활성화한다. 또한 메트릭 생성의
전체 비용을 줄이기 위해 다양한 쿠버네티스 시스템 네임스페이스의 계측을
비활성화한다. 다음 섹션은 제외된 모든 구성 요소를 나열한다.

- `exe_path` 기준으로 제외되는 서비스: `*/obi`, `obi`, `*otelcol`,
  `*otelcol-contrib`, `*otelcol-contrib[!/]*`.
- `k8s_namespace` 기준으로 제외되는 서비스: `kube-system`, `kube-node-lease`,
  `local-path-storage`, `cert-manager`, `monitoring`, `gke-connect`,
  `gke-gmp-system`, `gke-managed-cim`, `gke-managed-filestorecsi`,
  `gke-managed-metrics-server`, `gke-managed-system`, `gke-system`,
  `gke-managed-volumepopulator`, `gatekeeper-system`.

OBI가 자신이나 다른 제외된 구성 요소 일부를 계측하도록 허용하려면 이 옵션을
변경한다.

참고: 이러한 셀프 계측(self-instrumentation)을 활성화하려면 여전히
`instrument` 섹션에 해당 구성 요소를 포함하거나, 이 구성 요소가 더 포괄적인
포함 기준의 일부여야 한다.

### 예시: 기본적으로 제외된 네임스페이스에 대한 계측 활성화 {#example-enable-instrumentation-for-a-default-excluded-namespace}

기본적으로 제외된 네임스페이스(예: `monitoring`)에 대해 계측을 활성화하려면
기본 제외 목록에서 해당 네임스페이스를 제거하고 `instrument` 섹션에 추가해야
한다.

다음 예시는 `monitoring` 네임스페이스에 대한 계측을 활성화한다.

```yaml
discovery:
  # Include the monitoring namespace in instrumentation
  instrument:
    - k8s_namespace: monitoring

  # Override default exclusions to remove monitoring namespace
  # This list keeps other default exclusions but removes monitoring
  default_exclude_instrument:
    - exe_path: '{*/obi,obi,*otelcol,*otelcol-contrib,*otelcol-contrib[!/]*}'
    - k8s_namespace: kube-system
    - k8s_namespace: kube-node-lease
    - k8s_namespace: local-path-storage
    - k8s_namespace: cert-manager
    # monitoring namespace removed from this list
    - k8s_namespace: gke-connect
    - k8s_namespace: gke-gmp-system
    - k8s_namespace: gke-managed-cim
    - k8s_namespace: gke-managed-filestorecsi
    - k8s_namespace: gke-managed-metrics-server
    - k8s_namespace: gke-managed-system
    - k8s_namespace: gke-system
    - k8s_namespace: gke-managed-volumepopulator
    - k8s_namespace: gatekeeper-system
```

### 예시: 모든 기본 제외 비활성화 {#example-disable-all-default-exclusions}

모든 기본 제외를 비활성화하고 OBI가 일치하는 모든 서비스(자신과 기타
옵저버빌리티 구성 요소 포함)를 계측하도록 하려면 `default_exclude_instrument`를
빈 목록으로 설정한다.

```yaml
discovery:
  instrument:
    - k8s_namespace: '*' # or specific namespaces/selectors

  # Empty list disables all default exclusions
  default_exclude_instrument: []
```

> [!WARNING]
>
> 모든 기본 제외를 비활성화하면 OBI가 자신이나 다른 텔레메트리 컬렉터를
> 계측할 경우 리소스 사용량이 증가하고 피드백 루프가 발생할 수 있다.
> 프로덕션 환경에서는 이 구성을 신중하게 사용한다.

## Go 특화 트레이서 건너뛰기 {#skip-go-specific-tracers}

`skip_go_specific_tracers` 옵션은 **eBPF** 트레이서가 계측할 실행 파일을
검사할 때 Go 관련 세부 사항의 감지를 비활성화한다. 트레이서는 일반적으로 덜
효율적인 범용 계측 방식으로 대체된다.

## OTel로 계측된 서비스 제외 {#exclude-otel-instrumented-services}

`exclude_otel_instrumented_services` 옵션은 이미 오픈텔레메트리로 계측된
서비스에 대한 OBI 계측을 비활성화한다. OBI는 대개 쿠버네티스 클러스터의 모든
서비스를 모니터링하도록 배포되므로, 계측 선택(또는 제외) 기준을 신중하게 만들지
않으면 이미 계측된 서비스를 모니터링할 때 중복 텔레메트리 데이터가 발생할 수
있다. 불필요한 구성 오버헤드를 피하기 위해 OBI는 메트릭과 트레이스를 게시하는
오픈텔레메트리 SDK 호출을 감시하며, 자체 텔레메트리 데이터를 게시하는 서비스의
계측을 자동으로 끈다. 애플리케이션이 생성한 텔레메트리 데이터가 OBI가 생성한
메트릭 및 트레이스와 충돌하지 않는다면 이 옵션을 끈다.

## 프로세스 컨텍스트 보강 {#process-context-enrichment}

OBI v0.12.1부터 OBI는 실험적인 오픈텔레메트리 `OTEL_CTX` 프로세스 컨텍스트
매핑을 게시하는 프로세스로부터 문자열 리소스 속성과 메타데이터를 읽는다. OBI는
프로세스를 처음 발견했을 때 확인하고, 이후 초당 한 번씩 업데이트를 확인한다.
명시적으로 구성된 서비스 이름과 네임스페이스가 계속 우선한다.

Config v1에서는 `discovery.process_context_poll_interval` 또는
`OTEL_EBPF_PROCESS_CONTEXT_POLL_INTERVAL`을 사용하여 폴링 간격을 변경한다.
초기 확인만 유지하고 이후 폴링을 비활성화하려면 간격을 `0`으로 설정한다.
Config v2는 v0.12.1의 기본 간격을 사용하며 이 설정을 노출하지 않는다.

## 서비스 이름과 네임스페이스 재정의 {#override-service-name-and-namespace}

오픈텔레메트리나 Prometheus를 통해 계측 데이터를 내보내는 경우, OBI는 다른
계측 솔루션과의 상호 운용성을 높이기 위해
[오픈텔레메트리 오퍼레이터의 서비스 이름 규칙](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/auto-instrumentation/resource-attributes.md#how-resource-attributes-are-calculated-from-the-pods-metadata)을
따른다.

OBI는 서비스 이름과 네임스페이스를 자동으로 설정하기 위해 다음 순서로 기준을
사용한다.

1. 계측된 프로세스나 컨테이너의 `OTEL_RESOURCE_ATTRIBUTES` 및
   `OTEL_SERVICE_NAME` 환경 변수로 설정된 리소스 속성.
2. 쿠버네티스에서는 다음 Pod 어노테이션으로 설정된 리소스 속성:
   - `resource.opentelemetry.io/service.name`
   - `resource.opentelemetry.io/service.namespace`
3. 쿠버네티스에서는 다음 Pod 레이블로 설정된 리소스 속성:
   - `app.kubernetes.io/name`은 서비스 이름을 설정한다
   - `app.kubernetes.io/part-of`는 서비스 네임스페이스를 설정한다
4. 쿠버네티스에서는 Pod 소유자의 메타데이터로부터 계산된 리소스 속성을 다음
   순서로(사용 가능 여부에 따라) 사용한다:
   - `k8s.deployment.name`
   - `k8s.replicaset.name`
   - `k8s.statefulset.name`
   - `k8s.daemonset.name`
   - `k8s.cronjob.name`
   - `k8s.job.name`
   - `k8s.pod.name`
   - `k8s.container.name`
5. Java 애플리케이션의 경우, Spring Boot 애플리케이션 이름, JAR 매니페스트
   제목, JAR 기본 이름 중 먼저 사용 가능한 값.
6. 계측된 프로세스의 실행 파일 이름.

앞의 3번 항목에 있는 쿠버네티스 레이블은 구성을 통해 재정의할 수 있다.

YAML 예시:

```yaml
attributes:
  kubernetes:
    resource_labels:
      service.name:
        # gets service name from the first existing Pod label
        - override-svc-name
        - app.kubernetes.io/name
      service.namespace:
        # gets service namespace from the first existing Pod label
        - override-svc-ns
        - app.kubernetes.io/part-of
```

이들은 쉼표로 구분된 어노테이션 및 레이블 이름 목록을 받는다.

### 향상된 서비스 이름 조회(v0.5.0+) {#enhanced-service-name-lookup-v050}

v0.5.0부터 OBI는 특히 동적이고 분산된 환경에서 더 정확한 서비스 식별을
제공하는 향상된 서비스 이름 확인(resolution) 기능을 포함한다.

**개선 사항**:

1. **DNS 기반 확인**: OBI는 DNS 쿼리를 사용하여 서비스 이름을 확인할 수
   있으며, 애플리케이션이 사용하는 실제 서비스 디스커버리 메커니즘과 더 잘
   정렬된다
2. **메타데이터 보강**: 컨테이너 런타임과 오케스트레이션 플랫폼을 포함한 여러
   메타데이터 소스로부터의 향상된 조회

3. **연결 추적**: 클라이언트와 서버 양쪽의 아이덴티티를 확인하여 서비스 간
   통신을 더 잘 추적

**동작 방식**:

OBI는 서비스 간 네트워크 통신을 감지하면, 다음 우선순위 순서로 구성된 사용
가능한 소스를 사용하여 서비스 이름을 확인하려고 시도한다.

1. **쿠버네티스 메타데이터 조회**: 쿠버네티스 API에서 Pod와 소유자 메타데이터를
   확인한다(기본적으로 활성화됨)
2. **eBPF 기반 리버스 DNS**: 커널 레벨에서 캡처한 DNS 응답을 사용한다(선택
   사항, `OTEL_EBPF_NAME_RESOLVER_SOURCES=rdns` 필요)
3. **표준 리버스 DNS 조회**: IP 주소에 대해 리버스 DNS 쿼리를 수행한다(선택
   사항, `OTEL_EBPF_NAME_RESOLVER_SOURCES=dns` 필요)

로컬 서비스 이름은 다음 우선순위 계층을 따른다.

1. `OTEL_SERVICE_NAME` 환경 변수(최우선순위)
2. `OTEL_RESOURCE_ATTRIBUTES` 환경 변수(service.name 키)
3. 서비스 이름 어노테이션(resource.opentelemetry.io/service.name)
4. 서비스 이름 레이블(예: app.kubernetes.io/name)
5. 쿠버네티스 Pod 소유자 이름(예: Deployment 이름)(최하위 우선순위)

이 개선 사항은 특히 다음과 같은 경우에 유용하다.

- **서비스 메시 환경**: DNS가 서비스 라우팅에 사용되는 경우
- **쿠버네티스 클러스터**: Service 리소스와의 상관관계(correlation) 개선
- **마이크로서비스 아키텍처**: 정확한 서비스 이름을 통한 더 나은 서비스 그래프
  시각화

**구성**:

쿠버네티스 메타데이터 기반 서비스 조회는 기본적으로 활성화되어 있다. DNS 기반
확인 방식을 활성화하려면 이름 리졸버(name resolver) 소스를 구성한다.

```bash
# Enable both Kubernetes metadata and standard DNS reverse lookups
export OTEL_EBPF_NAME_RESOLVER_SOURCES=k8s,dns

# Or enable eBPF-captured DNS lookups (requires DNS event capture)
export OTEL_EBPF_NAME_RESOLVER_SOURCES=k8s,rdns

# Enable all resolver sources
export OTEL_EBPF_NAME_RESOLVER_SOURCES=k8s,dns,rdns

# Optional: Adjust cache size (default: 1024)
export OTEL_EBPF_NAME_RESOLVER_CACHE_LEN=2048

# Optional: Adjust cache time-to-live (default: 5 minutes)
export OTEL_EBPF_NAME_RESOLVER_CACHE_TTL=10m
```

쿠버네티스 환경에서는 다음을 확인한다.

- 네트워크 정책이 DNS 쿼리를 허용하는지(DNS 기반 확인을 사용하는 경우)
- CoreDNS나 이에 상응하는 DNS 서비스가 실행 중인지
- RDNS를 사용하는 경우 eBPF 프로그램이 DNS 이벤트를 캡처할 수 있는지

**이점**:

- **더 정확한 서비스 그래프**: 서비스 간 통신에서 IP 대신 실제 서비스 이름을
  표시한다
- **더 나은 트레이스 상관관계**: 트레이스가 서비스 카탈로그와 일치하는 서비스
  이름을 표시한다
- **더 쉬운 문제 해결**: 수동 IP 조회 없이 어떤 서비스가 통신하고 있는지
  식별한다

**예시**:

향상된 조회가 없으면 다음과 같이 표시될 수 있다.

```console
service.name: "10.0.1.42"
peer.service: "10.0.2.15"
```

향상된 조회를 활성화하면:

```console
service.name: "frontend"
peer.service: "backend-api"
```
