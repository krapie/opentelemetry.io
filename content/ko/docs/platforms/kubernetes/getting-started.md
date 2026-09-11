---
title: 시작하기
weight: 1
# prettier-ignore
cSpell:ignore: filelog filelogreceiver kubelet kubeletstats kubeletstatsreceiver sattributes sattributesprocessor sclusterreceiver sobjectsreceiver
default_lang_commit: 4cb7e22f1e45d17854b309efc730499880aa7197
---

이 페이지는 오픈텔레메트리(OpenTelemetry)를 사용해 쿠버네티스 클러스터를
모니터링하는 가장 빠른 방법을 안내한다. 쿠버네티스 클러스터, 노드, 파드,
컨테이너의 메트릭과 로그를 수집하는 데 초점을 맞추며, OTLP 데이터를 방출하는
서비스를 클러스터가 지원하도록 활성화하는 방법도 다룬다.

쿠버네티스에서 오픈텔레메트리가 실제로 동작하는 모습을 보고 싶다면
[오픈텔레메트리 데모](/docs/demo/kubernetes-deployment/)부터 시작하는 것이 가장
좋다. 이 데모는 오픈텔레메트리의 구현을 보여주기 위한 것이며, 쿠버네티스 자체를
모니터링하는 방법의 예시로 의도된 것은 아니다. 이 안내를 마친 후에는 데모를
설치하여 실제 워크로드에 모니터링이 어떻게 반응하는지 살펴보는 재미있는 실험을
해볼 수 있다.

Prometheus에서 오픈텔레메트리로 마이그레이션을 시작하려 하거나, 오픈텔레메트리
컬렉터를 사용해 Prometheus 메트릭을 수집하는 데 관심이 있다면
[Prometheus 리시버](/docs/platforms/kubernetes/collector/components/#prometheus-receiver)를
참고한다.

## 개요 {#overview}

쿠버네티스는 다양한 방식으로 많은 중요한 텔레메트리를 노출한다. 로그, 이벤트,
여러 다른 오브젝트에 대한 메트릭, 워크로드가 생성하는 데이터를 가지고 있다.

이 모든 데이터를 수집하기 위해 [오픈텔레메트리 컬렉터](/docs/collector/)를
사용한다. 컬렉터는 이 모든 데이터를 효율적으로 수집하고 의미 있는 방식으로
보강할 수 있게 해주는 다양한 도구를 갖추고 있다.

모든 데이터를 수집하려면 컬렉터를 두 가지 방식으로 설치해야 한다. 하나는
[Daemonset](/docs/collector/deploy/agent/)으로, 다른 하나는
[Deployment](/docs/collector/deploy/gateway/)로 설치한다. Daemonset으로 설치한
컬렉터는 서비스가 방출하는 텔레메트리와 노드, 파드, 컨테이너의 로그 및 메트릭을
수집하는 데 사용된다. Deployment로 설치한 컬렉터는 클러스터의 메트릭과 이벤트를
수집하는 데 사용된다.

컬렉터를 설치하기 위해
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용하며, 이 차트는 컬렉터 구성을 더 쉽게 만들어주는 몇 가지 구성 옵션을
제공한다. Helm이 익숙하지 않다면 [Helm 프로젝트 사이트](https://helm.sh/)를
확인한다. 쿠버네티스 오퍼레이터 사용에 관심이 있다면
[오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator/)를 참고하되, 이
가이드에서는 Helm 차트에 초점을 맞춘다.

## 준비 {#preparation}

이 가이드는 [Kind 클러스터](https://kind.sigs.k8s.io/) 사용을 가정하지만,
적합하다고 판단하는 어떤 쿠버네티스 클러스터든 자유롭게 사용해도 된다.

[Kind가 이미 설치되어 있다고 가정하고](https://kind.sigs.k8s.io/#installation-and-usage),
새 kind 클러스터를 생성한다.

```sh
kind create cluster
```

[Helm이 이미 설치되어 있다고 가정하고](https://helm.sh/docs/intro/install/),
나중에 설치할 수 있도록 오픈텔레메트리 컬렉터 Helm 차트를 추가한다.

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

## Daemonset 컬렉터 {#daemonset-collector}

쿠버네티스 텔레메트리를 수집하는 첫 단계는 노드와 해당 노드에서 실행 중인
워크로드에 관련된 텔레메트리를 수집하기 위해 오픈텔레메트리 컬렉터의 daemonset
인스턴스를 배포하는 것이다. daemonset을 사용하면 이 컬렉터 인스턴스가 모든
노드에 설치되는 것을 보장할 수 있다. daemonset의 각 컬렉터 인스턴스는 자신이
실행 중인 노드로부터만 데이터를 수집한다.

이 컬렉터 인스턴스는 다음 컴포넌트를 사용한다.

- [OTLP 리시버](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver):
  애플리케이션 트레이스, 메트릭, 로그를 수집한다.
- [쿠버네티스 속성 프로세서](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor):
  수신되는 애플리케이션 텔레메트리에 쿠버네티스 메타데이터를 추가한다.
- [Kubeletstats 리시버](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver):
  kubelet의 API 서버에서 노드, 파드, 컨테이너 메트릭을 가져온다.
- [Filelog 리시버](/docs/platforms/kubernetes/collector/components/#filelog-receiver):
  stdout/stderr에 기록된 쿠버네티스 로그와 애플리케이션 로그를 수집한다.

각 컴포넌트를 하나씩 살펴본다.

### OTLP 리시버 {#otlp-receiver}

[OTLP 리시버](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)는
[OTLP 형식](/docs/specs/otel/protocol/)의 트레이스, 메트릭, 로그를 수집하기 위한
최선의 솔루션이다. 애플리케이션 텔레메트리를 다른 형식으로 방출하고 있다면
[컬렉터에 그에 맞는 리시버가 있을 가능성이 높지만](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver),
이 튜토리얼에서는 텔레메트리가 OTLP 형식이라고 가정한다.

필수 사항은 아니지만, 노드에서 실행 중인 애플리케이션이 동일한 노드에서 실행
중인 컬렉터로 트레이스, 메트릭, 로그를 방출하는 것이 일반적인 관행이다. 이렇게
하면 네트워크 상호작용이 단순해지고 `k8sattributes` 프로세서를 사용해 쿠버네티스
메타데이터를 쉽게 연관 지을 수 있다.

### 쿠버네티스 속성 프로세서 {#kubernetes-attributes-processor}

[쿠버네티스 속성 프로세서](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor)는
쿠버네티스 파드로부터 텔레메트리를 수신하는 모든 컬렉터에서 사용하는 것이 적극
권장되는 컴포넌트이다. 이 프로세서는 쿠버네티스 파드를 자동으로 검색하고, 파드
이름이나 노드 이름과 같은 메타데이터를 추출하여, 추출한 메타데이터를 리소스
속성으로 스팬, 메트릭, 로그에 추가한다. 이 프로세서는 텔레메트리에 쿠버네티스
컨텍스트를 추가하므로, 애플리케이션의 트레이스, 메트릭, 로그 시그널을 파드
메트릭 및 트레이스와 같은 쿠버네티스 텔레메트리와 연관 지을 수 있게 해준다.

### Kubeletstats 리시버 {#kubeletstats-receiver}

[Kubeletstats 리시버](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver)는
노드에 대한 메트릭을 수집하는 리시버이다. 컨테이너 메모리 사용량, 파드 CPU
사용량, 노드 네트워크 오류와 같은 메트릭을 수집한다. 모든 텔레메트리에는 파드
이름이나 노드 이름과 같은 쿠버네티스 메타데이터가 포함된다. 쿠버네티스 속성
프로세서를 사용하고 있으므로, 애플리케이션의 트레이스, 메트릭, 로그를
Kubeletstats 리시버가 생성하는 메트릭과 연관 지을 수 있다.

### Filelog 리시버 {#filelog-receiver}

[Filelog 리시버](/docs/platforms/kubernetes/collector/components/#filelog-receiver)는
쿠버네티스가 `/var/log/pods/*/*/*.log`에 기록하는 로그를 테일링하여
stdout/stderr에 기록된 로그를 수집한다. 대부분의 로그 테일러(tailer)와
마찬가지로, filelog 리시버는 필요에 따라 파일을 파싱할 수 있는 다양한
액션(action)을 제공한다.

언젠가는 Filelog 리시버를 직접 구성해야 할 수도 있지만, 이 안내에서는
오픈텔레메트리 Helm 차트가 복잡한 구성을 모두 대신 처리해준다. 또한 파일 이름을
기반으로 유용한 쿠버네티스 메타데이터를 추출한다. 쿠버네티스 속성 프로세서를
사용하고 있으므로, 애플리케이션의 트레이스, 메트릭, 로그를 Filelog 리시버가
생성하는 로그와 연관 지을 수 있다.

---

오픈텔레메트리 컬렉터 Helm 차트는 daemonset으로 설치한 컬렉터에서 이 모든
컴포넌트를 구성하는 작업을 쉽게 만들어준다. 또한 RBAC, 마운트, 호스트 포트와
같은 쿠버네티스 관련 세부 사항도 모두 처리해준다.

한 가지 주의할 점은, 이 차트는 기본적으로 데이터를 어떤 백엔드로도 전송하지
않는다는 것이다. 선호하는 백엔드에서 실제로 데이터를 사용하려면 익스포터를 직접
구성해야 한다.

다음은 우리가 사용할 `values.yaml`이다.

```yaml
mode: daemonset

image:
  repository: otel/opentelemetry-collector-k8s

presets:
  # enables the k8sattributesprocessor and adds it to the traces, metrics, and logs pipelines
  kubernetesAttributes:
    enabled: true
  # enables the kubeletstatsreceiver and adds it to the metrics pipelines
  kubeletMetrics:
    enabled: true
  # Enables the filelogreceiver and adds it to the logs pipelines
  logsCollection:
    enabled: true
## The chart only includes the debugexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlp exporter
# config:
#   exporters:
#     otlp:
#       endpoint: "<SOME BACKEND>"
#   service:
#     pipelines:
#       traces:
#         exporters: [ otlp ]
#       metrics:
#         exporters: [ otlp ]
#       logs:
#         exporters: [ otlp ]
```

이 `values.yaml`을 차트와 함께 사용하려면 원하는 파일 위치에 저장한 다음, 다음
명령을 실행해 차트를 설치한다.

```sh
helm install otel-collector open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

이제 클러스터에서 daemonset으로 설치된 오픈텔레메트리 컬렉터가 실행되며 각
노드로부터 텔레메트리를 수집하고 있을 것이다!

## Deployment 컬렉터 {#deployment-collector}

쿠버네티스 텔레메트리를 수집하는 다음 단계는 클러스터 전체에 관련된 텔레메트리를
수집하기 위해 컬렉터의 deployment 인스턴스를 배포하는 것이다. 정확히 하나의
레플리카를 가진 deployment를 사용하면 중복 데이터가 생성되지 않도록 보장할 수
있다.

이 컬렉터 인스턴스는 다음 컴포넌트를 사용한다.

- [쿠버네티스 클러스터 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver):
  클러스터 수준의 메트릭과 엔티티 이벤트를 수집한다.
- [쿠버네티스 오브젝트 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver):
  쿠버네티스 API 서버로부터 이벤트와 같은 오브젝트를 수집한다.

각 컴포넌트를 하나씩 살펴본다.

### 쿠버네티스 클러스터 리시버 {#kubernetes-cluster-receiver}

[쿠버네티스 클러스터 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver)는
클러스터 전체 상태에 대한 메트릭을 수집하기 위한 컬렉터의 솔루션이다. 이
리시버는 노드 상태, 파드 단계, 컨테이너 재시작 횟수, 사용 가능 및 원하는
(desired) deployment 수 등에 대한 메트릭을 수집할 수 있다.

### 쿠버네티스 오브젝트 리시버 {#kubernetes-objects-receiver}

[쿠버네티스 오브젝트 리시버](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver)는
쿠버네티스 오브젝트를 로그로 수집하기 위한 컬렉터의 솔루션이다. 어떤 오브젝트든
수집할 수 있지만, 흔하고 중요한 사용 사례는 쿠버네티스 이벤트를 수집하는 것이다.

---

오픈텔레메트리 컬렉터 Helm 차트는 deployment로 설치한 컬렉터에서 이 모든
컴포넌트에 대한 구성을 간소화해준다. 또한 RBAC와 마운트 같은 쿠버네티스 관련
세부 사항도 모두 처리해준다.

한 가지 주의할 점은, 이 차트는 기본적으로 데이터를 어떤 백엔드로도 전송하지
않는다는 것이다. 선호하는 백엔드에서 실제로 데이터를 사용하려면 익스포터를 직접
구성해야 한다.

다음은 우리가 사용할 `values.yaml`이다.

```yaml
mode: deployment

image:
  repository: otel/opentelemetry-collector-k8s

# We only want one of these collectors - any more and we'd produce duplicate data
replicaCount: 1

presets:
  # enables the k8sclusterreceiver and adds it to the metrics pipelines
  clusterMetrics:
    enabled: true
  # enables the k8sobjectsreceiver to collect events only and adds it to the logs pipelines
  kubernetesEvents:
    enabled: true
## The chart only includes the debugexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlp exporter
# config:
# exporters:
#   otlp:
#     endpoint: "<SOME BACKEND>"
# service:
#   pipelines:
#     traces:
#       exporters: [ otlp ]
#     metrics:
#       exporters: [ otlp ]
#     logs:
#       exporters: [ otlp ]
```

이 `values.yaml`을 차트와 함께 사용하려면 원하는 파일 위치에 저장한 다음, 다음
명령을 실행해 차트를 설치한다.

```sh
helm install otel-collector-cluster open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

이제 클러스터에서 deployment로 설치된 컬렉터가 실행되며 클러스터 메트릭과
이벤트를 수집하고 있을 것이다!
