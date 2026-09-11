---
title: 오픈텔레메트리 컬렉터 차트
linkTitle: 컬렉터 차트
# prettier-ignore
cSpell:ignore: filelog filelogreceiver hostmetricsreceiver kubelet kubeletstats kubeletstatsreceiver otlp_http sattributesprocessor sclusterreceiver sobjectsreceiver statefulset
default_lang_commit: 638af010dde026663ffa75f96c0d9310a09d6714
---

## 소개 {#introduction}

[오픈텔레메트리 컬렉터](/docs/collector)는 쿠버네티스 클러스터와 그 안에서
동작하는 모든 서비스를 모니터링하기 위한 중요한 도구이다. 쿠버네티스
클러스터에서 컬렉터 배포를 설치하고 관리하기 쉽게 만들기 위해 오픈텔레메트리
커뮤니티는
[오픈텔레메트리 컬렉터 Helm 차트](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector)를
만들었다. 이 Helm 차트를 사용하면 컬렉터를 Deployment, Daemonset, Statefulset
형태로 설치할 수 있다.

### 차트 설치 {#installing-the-chart}

릴리스 이름 `my-opentelemetry-collector`로 차트를 설치하려면 다음 명령을
실행한다.

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install my-opentelemetry-collector open-telemetry/opentelemetry-collector \
   --set image.repository="otel/opentelemetry-collector-k8s" \
   --set mode=<daemonset|deployment|statefulset>
```

### 구성 {#configuration}

오픈텔레메트리 컬렉터 차트는 `mode`가 설정되어 있어야 한다. `mode`는 사용 사례에
필요한 쿠버네티스 배포 종류에 따라 `daemonset`, `deployment`, `statefulset` 중
하나로 설정할 수 있다.

설치하면 차트는 시작할 수 있도록 몇 가지 기본 컬렉터 컴포넌트를 제공한다.
기본적으로 컬렉터의 구성은 다음과 같다.

```yaml
exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug: {}
extensions:
  health_check: {}
processors:
  batch: {}
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
receivers:
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: opentelemetry-collector
          scrape_interval: 10s
          static_configs:
            - targets:
                - ${env:MY_POD_IP}:8888
  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
service:
  extensions:
    - health_check
  pipelines:
    logs:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
    metrics:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - prometheus
    traces:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - jaeger
        - zipkin
  telemetry:
    metrics:
      address: ${env:MY_POD_IP}:8888
```

차트는 기본 리시버를 기준으로 포트도 활성화한다. `values.yaml`에서 값을 `null`로
설정하면 기본 구성을 제거할 수 있다. 포트도 `values.yaml`에서 비활성화할 수
있다.

`values.yaml`의 `config` 섹션을 사용해 구성의 어떤 부분이든 추가하거나 수정할 수
있다. 파이프라인을 변경할 때는 기본 컴포넌트를 포함해 파이프라인에 있는 모든
컴포넌트를 명시적으로 나열해야 한다.

예를 들어 메트릭 및 로깅 파이프라인과 otlp가 아닌 리시버를 비활성화하려면 다음과
같이 한다.

```yaml
config:
  receivers:
    jaeger: null
    prometheus: null
    zipkin: null
  service:
    pipelines:
      traces:
        receivers:
          - otlp
      metrics: null
      logs: null
ports:
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
```

차트에서 사용 가능한 모든 구성 옵션(주석 포함)은
[values.yaml 파일](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-collector/values.yaml)에서
확인할 수 있다.

### 프리셋 {#presets}

오픈텔레메트리 컬렉터가 쿠버네티스를 모니터링하는 데 사용하는 중요한 컴포넌트 중
다수는 컬렉터 자체의 쿠버네티스 배포에서 특별한 설정이 필요하다. 이러한
컴포넌트를 더 쉽게 사용할 수 있도록, 오픈텔레메트리 컬렉터 차트는 활성화하면
이러한 중요한 컴포넌트에 필요한 복잡한 설정을 대신 처리해주는 몇 가지 프리셋을
제공한다.

프리셋은 시작점으로 사용해야 한다. 관련 컴포넌트에 대해 기본적이지만 풍부한
기능을 구성해준다. 사용 사례에서 이러한 컴포넌트의 추가 구성이 필요하다면,
프리셋을 사용하지 않고 컴포넌트와 그것이 필요로 하는 것(볼륨, RBAC 등)을 직접
구성하는 것을 권장한다.

#### 로그 수집 프리셋 {#logs-collection-preset}

오픈텔레메트리 컬렉터는 쿠버네티스 컨테이너가 표준 출력으로 보내는 로그를
수집하는 데 사용할 수 있다.

이 기능은 기본적으로 비활성화되어 있다. 안전하게 활성화하려면 다음 요구 사항을
충족해야 한다.

- [Filelog 리시버](/docs/platforms/kubernetes/collector/components/#filelog-receiver)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.
- 엄격한 요구 사항은 아니지만, 이 프리셋은 `mode=daemonset`과 함께 사용하는 것이
  권장된다. `filelogreceiver`는 컬렉터가 실행 중인 노드의 로그만 수집할 수
  있으며, 동일한 노드에 여러 컬렉터가 구성되어 있으면 중복 데이터가 발생한다.

이 기능을 활성화하려면 `presets.logsCollection.enabled` 속성을 `true`로
설정한다. 활성화하면 차트는 `logs` 파이프라인에 `filelogreceiver`를 추가한다. 이
리시버는 쿠버네티스 컨테이너 런타임이 모든 컨테이너의 콘솔 출력을 기록하는
파일(`/var/log/pods/*/*/*.log`)을 읽도록 구성된다.

다음은 `values.yaml` 예제이다.

```yaml
mode: daemonset
presets:
  logsCollection:
    enabled: true
```

차트의 기본 로그 파이프라인은 `debug` 익스포터를 사용한다. `logsCollection`
프리셋의 `filelogreceiver`와 함께 사용하면, 내보낸 로그가 실수로 다시 컬렉터에
유입되어 이른바 로그 폭발(log explosion) 현상을 일으키기 쉽다.

이러한 순환을 방지하기 위해, 리시버의 기본 구성은 컬렉터 자신의 로그를 제외한다.
컬렉터의 로그를 포함하고 싶다면, 컬렉터의 표준 출력으로 로그를 보내지 않는
익스포터로 `debug` 익스포터를 반드시 교체한다.

다음은 `logs` 파이프라인의 기본 `debug` 익스포터를 컨테이너 로그를
`https://example.com:55681` 엔드포인트로 전송하는 `otlp_http` 익스포터로
교체하는 `values.yaml` 예제이다. 또한
`presets.logsCollection.includeCollectorLogs`를 사용해 프리셋이 컬렉터 로그
수집도 활성화하도록 지시한다.

```yaml
mode: daemonset

presets:
  logsCollection:
    enabled: true
    includeCollectorLogs: true

config:
  exporters:
    otlp_http:
      endpoint: https://example.com:55681
  service:
    pipelines:
      logs:
        exporters:
          - otlp_http
```

#### 쿠버네티스 속성 프리셋 {#kubernetes-attributes-preset}

오픈텔레메트리 컬렉터는 `k8s.pod.name`, `k8s.namespace.name`, `k8s.node.name`과
같은 쿠버네티스 메타데이터를 로그, 메트릭, 트레이스에 추가하도록 구성할 수 있다.
이 프리셋을 사용하거나 `k8sattributesprocessor`를 수동으로 활성화하는 것을 적극
권장한다.

RBAC 관련 고려 사항으로 인해, 이 기능은 기본적으로 비활성화되어 있다. 다음 요구
사항이 있다.

- [쿠버네티스 속성 프로세서](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.

이 기능을 활성화하려면 `presets.kubernetesAttributes.enabled` 속성을 `true`로
설정한다. 활성화하면 차트는 필요한 RBAC 역할을 ClusterRole에 추가하고, 활성화된
각 파이프라인에 `k8sattributesprocessor`를 추가한다.

다음은 `values.yaml` 예제이다.

```yaml
mode: daemonset
presets:
  kubernetesAttributes:
    enabled: true
```

#### Kubelet 메트릭 프리셋 {#kubelet-metrics-preset}

오픈텔레메트리 컬렉터는 kubelet의 API 서버에서 노드, 파드, 컨테이너 메트릭을
수집하도록 구성할 수 있다.

이 기능은 기본적으로 비활성화되어 있다. 다음 요구 사항이 있다.

- [Kubeletstats 리시버](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.
- 엄격한 요구 사항은 아니지만, 이 프리셋은 `mode=daemonset`과 함께 사용하는 것이
  권장된다. `kubeletstatsreceiver`는 컬렉터가 실행 중인 노드의 메트릭만 수집할
  수 있으며, 동일한 노드에 여러 컬렉터가 구성되어 있으면 중복 데이터가 발생한다.

이 기능을 활성화하려면 `presets.kubeletMetrics.enabled` 속성을 `true`로
설정한다. 활성화하면 차트는 필요한 RBAC 역할을 ClusterRole에 추가하고, 메트릭
파이프라인에 `kubeletstatsreceiver`를 추가한다.

다음은 `values.yaml` 예제이다.

```yaml
mode: daemonset
presets:
  kubeletMetrics:
    enabled: true
```

#### 클러스터 메트릭 프리셋 {#cluster-metrics-preset}

오픈텔레메트리 컬렉터는 쿠버네티스 API 서버에서 클러스터 수준의 메트릭을
수집하도록 구성할 수 있다. 이 메트릭에는 Kube State Metrics가 수집하는 메트릭
다수가 포함된다.

이 기능은 기본적으로 비활성화되어 있다. 다음 요구 사항이 있다.

- [쿠버네티스 클러스터 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.
- 엄격한 요구 사항은 아니지만, 이 프리셋은 레플리카가 하나인 `mode=deployment`
  또는 `mode=statefulset`과 함께 사용하는 것이 권장된다. 여러 컬렉터에서
  `k8sclusterreceiver`를 실행하면 중복 데이터가 발생한다.

이 기능을 활성화하려면 `presets.clusterMetrics.enabled` 속성을 `true`로
설정한다. 활성화하면 차트는 필요한 RBAC 역할을 ClusterRole에 추가하고, 메트릭
파이프라인에 `k8sclusterreceiver`를 추가한다.

다음은 `values.yaml` 예제이다.

```yaml
mode: deployment
replicaCount: 1
presets:
  clusterMetrics:
    enabled: true
```

#### 쿠버네티스 이벤트 프리셋 {#kubernetes-events-preset}

오픈텔레메트리 컬렉터는 쿠버네티스 이벤트를 수집하도록 구성할 수 있다.

이 기능은 기본적으로 비활성화되어 있다. 다음 요구 사항이 있다.

- [쿠버네티스 오브젝트 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.
- 엄격한 요구 사항은 아니지만, 이 프리셋은 레플리카가 하나인 `mode=deployment`
  또는 `mode=statefulset`과 함께 사용하는 것이 권장된다. 여러 컬렉터에서
  `k8sclusterreceiver`를 실행하면 중복 데이터가 발생한다.

이 기능을 활성화하려면 `presets.kubernetesEvents.enabled` 속성을 `true`로
설정한다. 활성화하면 차트는 필요한 RBAC 역할을 ClusterRole에 추가하고, 이벤트만
수집하도록 구성된 `k8sobjectsreceiver`를 로그 파이프라인에 추가한다.

다음은 `values.yaml` 예제이다.

```yaml
mode: deployment
replicaCount: 1
presets:
  kubernetesEvents:
    enabled: true
```

#### 호스트 메트릭 프리셋 {#host-metrics-preset}

오픈텔레메트리 컬렉터는 쿠버네티스 노드에서 호스트 메트릭을 수집하도록 구성할 수
있다.

이 기능은 기본적으로 비활성화되어 있다. 다음 요구 사항이 있다.

- [호스트 메트릭 리시버](/docs/platforms/kubernetes/collector/components/#host-metrics-receiver)가
  [Contrib 배포판](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib)과
  같은 컬렉터 이미지에 포함되어 있어야 한다.
- 엄격한 요구 사항은 아니지만, 이 프리셋은 `mode=daemonset`과 함께 사용하는 것이
  권장된다. `hostmetricsreceiver`는 컬렉터가 실행 중인 노드의 메트릭만 수집할 수
  있으며, 동일한 노드에 여러 컬렉터가 구성되어 있으면 중복 데이터가 발생한다.

이 기능을 활성화하려면 `presets.hostMetrics.enabled` 속성을 `true`로 설정한다.
활성화하면 차트는 필요한 볼륨과 volumeMounts를 추가하고, 메트릭 파이프라인에
`hostmetricsreceiver`를 추가한다. 기본적으로 메트릭은 10초마다 스크레이핑되며
다음 스크레이퍼가 활성화된다.

- cpu
- load
- memory
- disk
- filesystem[^1]
- network

다음은 `values.yaml` 예제이다.

```yaml
mode: daemonset
presets:
  hostMetrics:
    enabled: true
```

[^1]:
    `kubeletMetrics` 프리셋과 일부 겹치는 부분이 있어, 일부 파일 시스템 유형과
    마운트 지점은 기본적으로 제외된다.
