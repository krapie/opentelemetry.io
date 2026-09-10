---
title: Helm으로 쿠버네티스에 OBI 배포하기
linkTitle: Helm 차트
description: Helm 차트로 쿠버네티스에 OBI를 배포하는 방법을 알아본다.
weight: 2
default_lang_commit: eaf1126a8a051205de71dcad0aab717da06b4fe9
---

> [!NOTE]
>
> 다양한 Helm 구성 옵션에 대한 자세한 내용은
> [OBI Helm 차트 문서](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-ebpf-instrumentation)를
> 확인하거나
> [Artifact Hub](https://artifacthub.io/packages/helm/opentelemetry-helm/opentelemetry-ebpf-instrumentation)에서
> 차트를 살펴본다. 자세한 구성 파라미터는
> [values.yaml](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-ebpf-instrumentation/values.yaml)
> 파일을 참고한다.

목차:

<!-- TOC -->

- [Helm으로 OBI 배포하기](#deploying-obi-from-helm)
- [OBI 구성하기](#configuring-obi)
- [OBI 메타데이터 구성하기](#configuring-obi-metadata)
- [k8s-cache로 쿠버네티스 메타데이터 중앙 집중화하기](#centralizing-kubernetes-metadata-with-k8s-cache)
- [Helm 구성에 시크릿 제공하기](#providing-secrets-to-the-helm-configuration)

<!-- TOC -->

## Helm으로 OBI 배포하기 {#deploying-obi-from-helm}

먼저, 오픈텔레메트리(OpenTelemetry) Helm 리포지터리를 Helm에 추가해야 한다.

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

다음 명령은 `obi` 네임스페이스에 기본 구성으로 OBI DaemonSet을 배포한다.

```sh
helm install obi -n obi --create-namespace open-telemetry/opentelemetry-ebpf-instrumentation
```

기본 OBI 구성은 다음과 같다.

- Pod HTTP 포트 `9090`의 `/metrics` 경로에서 메트릭을 Prometheus 메트릭으로
  내보낸다.
- 클러스터 내 모든 애플리케이션을 계측하려고 시도한다.
- 기본적으로 애플리케이션 수준 메트릭만 제공하고
  [네트워크 수준 메트릭](../../network/)은 제외한다.
- 예를 들어 `k8s.namespace.name`이나 `k8s.pod.name`과 같은 쿠버네티스 메타데이터
  레이블로 메트릭을 데코레이션(decoration)하도록 OBI를 구성한다.

## OBI 구성하기 {#configuring-obi}

OBI의 기본 구성을 재정의하고 싶을 수 있다. 예를 들어 메트릭 또는 스팬을
Prometheus 대신 오픈텔레메트리로 내보내거나, 계측할 서비스 수를 제한하는
경우이다.

기본 [OBI 구성 옵션](../../configure/)을 직접 지정한 값으로 재정의할 수 있다.

예를 들어, 커스텀 구성이 담긴 `helm-obi.yml` 파일을 생성한다.

```yaml
config:
  data:
    # Contents of the actual OBI configuration file
    discovery:
      instrument:
        - k8s_namespace: demo
        - k8s_namespace: blog
    routes:
      unmatched: heuristic
```

`config.data` 섹션에는 [OBI 구성 옵션 문서](../../configure/options/)에 설명된
OBI 구성 파일이 담긴다.

그런 다음 `-f` 플래그로 재정의한 구성을 `helm` 명령에 전달한다. 예를 들면 다음과
같다.

```sh
helm install obi open-telemetry/opentelemetry-ebpf-instrumentation -f helm-obi.yml
```

또는, OBI 차트가 이전에 배포된 적이 있다면 다음과 같이 한다.

```sh
helm upgrade obi open-telemetry/opentelemetry-ebpf-instrumentation -f helm-obi.yml
```

## OBI 메타데이터 구성하기 {#configuring-obi-metadata}

OBI가 Prometheus 익스포터를 사용하여 데이터를 내보내는 경우, Prometheus
스크레이퍼가 OBI Pod를 발견할 수 있도록 OBI Pod 어노테이션을 재정의해야 할 수
있다. 예시 `helm-obi.yml` 파일에 다음 섹션을 추가할 수 있다.

```yaml
podAnnotations:
  prometheus.io/scrape: 'true'
  prometheus.io/path: '/metrics'
  prometheus.io/port: '9090'
```

마찬가지로, Helm 차트는 서비스 어카운트(ServiceAccount), 클러스터
롤(ClusterRole), 시큐리티 컨텍스트(securityContext) 등 OBI 배포에 관여하는 여러
리소스의 이름, 레이블, 어노테이션을 재정의할 수 있게 해준다.
[OBI Helm 차트 문서](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-ebpf-instrumentation)에서
다양한 구성 옵션을 설명한다.

## k8s-cache로 쿠버네티스 메타데이터 중앙 집중화하기 {#centralizing-kubernetes-metadata-with-k8s-cache}

기본적으로 각 OBI Pod는 로컬 노드뿐 아니라 K8s 클러스터 전체의 Pod, Node,
Service 메타데이터를 관찰(watch)하기 위해 쿠버네티스 API 서버에 각자의 커넥션을
연다. 이는 요청의 소스뿐 아니라 대상 정보까지 보강하기 위한 것이다(예:
[피어(peer)](/docs/specs/semconv/registry/attributes/service/#service-attributes-for-peer-services)
속성을 추가하기 위해 아웃바운드 HTTP 요청의 서비스 이름을 가져오거나,
[서비스 그래프(service graph)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/servicegraphconnector)
메트릭의 대상을 확인하기 위함). 대규모 클러스터에서 각 OBI Pod가 K8s 클러스터
전체 메타데이터를 조회하면 API 서버에 과부하가 걸리고 클러스터 전체에 영향을 줄
수 있다.

이를 방지하기 위해 OBI Helm 차트는 `k8s-cache`라는 작은 보조 서비스(companion
service)를 배포할 수 있다. 이 캐시는 모든 OBI Pod를 대신하여 쿠버네티스 API를 한
번만 관찰(watch)하고 gRPC를 통해 메타데이터를 스트리밍하며, 이를 통해 OBI의
Pod별 인포머 트래픽이 API 서버로 가는 것을 제거하고 API 부하를 크게 줄인다.

이를 활성화하려면 `helm-obi.yml`에서 `k8sCache.replicas`를 0이 아닌 값으로
설정한다.

```yaml
k8sCache:
  replicas: 1
```

단일 레플리카로도 대개 충분하다. 고가용성이나 매우 큰 클러스터의 경우 레플리카
수를 늘린다. OBI Pod는 캐시 `Service`를 통해 이들 사이에서 로드 밸런싱하며, 장애
발생 시 정상 레플리카로 재연결한다.

`k8sCache.replicas`가 `0`(기본값)이면 캐시가 배포되지 않으며, 각 OBI Pod는 자체
로컬 인포머를 사용한다.

## Helm 구성에 시크릿 제공하기 {#providing-secrets-to-the-helm-configuration}

오픈텔레메트리 엔드포인트(OpenTelemetry Endpoint)를 통해 메트릭과 트레이스를
옵저버빌리티 백엔드로 직접 전송하는 경우, `OTEL_EXPORTER_OTLP_HEADERS` 환경
변수로 자격 증명을 제공해야 할 수 있다.

권장 방법은 이러한 값을 쿠버네티스 Secret에 저장한 뒤, Helm 구성에서 이를
참조하는 환경 변수를 지정하는 것이다.

예를 들어, 다음 시크릿을 배포한다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: obi-secret
type: Opaque
stringData:
  otlp-headers: 'Authorization=Basic ....'
```

그런 다음 `helm-config.yml` 파일에서 `envValueFrom` 섹션을 통해 이를 참조한다.

```yaml
env:
  OTEL_EXPORTER_OTLP_ENDPOINT: '<...your OTLP endpoint URL...>'
envValueFrom:
  OTEL_EXPORTER_OTLP_HEADERS:
    secretKeyRef:
      key: otlp-headers
      name: obi-secret
```
