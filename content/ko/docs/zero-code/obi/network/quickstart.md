---
title: OBI 네트워크 메트릭 빠른 시작
linkTitle: 빠른 시작
description:
  오픈텔레메트리(OpenTelemetry) eBPF 계측으로 네트워크 메트릭을 생성하는 빠른
  시작 가이드
weight: 1
default_lang_commit: 348e946b973626a41f9ae316b93516c2c165b664
---

OBI는 모든 환경(물리 호스트, 가상 호스트, 컨테이너)에서 네트워크 메트릭을 생성할
수 있다. OBI가 각 메트릭을 소스 및 대상 쿠버네티스 엔티티(entity)의 메타데이터로
데코레이션(decoration)할 수 있으므로, 쿠버네티스 환경을 사용하는 것이 권장된다.

이 빠른 시작 가이드의 지침은 kubectl 명령줄 유틸리티로 쿠버네티스에 직접
배포하는 데 초점을 맞춘다. 이 튜토리얼은 쿠버네티스에 OBI를 처음부터 배포하는
방법을 설명한다. Helm을 사용하려면
[Helm으로 쿠버네티스에 OBI 배포하기](../../setup/kubernetes-helm/) 문서를
참고한다.

## 네트워크 메트릭과 함께 OBI 배포하기 {#deploy-obi-with-network-metrics}

네트워크 메트릭을 활성화하려면 OBI 구성에서 다음 옵션을 설정한다.

환경 변수:

```bash
export OTEL_EBPF_NETWORK_METRICS=true
```

네트워크 메트릭을 사용하려면 메트릭이 쿠버네티스 메타데이터로 데코레이션되어야
한다. 이 기능을 활성화하려면 OBI 구성에서 다음 옵션을 설정한다.

환경 변수:

```bash
export OTEL_EBPF_KUBE_METADATA_ENABLE=true
```

더 많은 구성 옵션은 [OBI 구성 옵션](../../configure/options/)을 참고한다.

OBI 구성에 대해 자세히 알아보려면 [OBI 구성 문서](../../configure/options/)를
참고한다.

## 간단한 설정 {#simple-setup}

### OBI 배포하기 {#deploy-obi}

다음 YAML 구성은 네트워크 메트릭을 위한 간단한 OBI 배포를 제공한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: obi
  name: obi
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: obi
  name: obi-config
data:
  obi-config.yml: |
    network:
      enable: true
    attributes:
      kubernetes:
        enable: true
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace: obi
  name: obi
spec:
  selector:
    matchLabels:
      instrumentation: obi
  template:
    metadata:
      labels:
        instrumentation: obi
    spec:
      serviceAccountName: obi
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: obi-config
          configMap:
            name: obi-config
        - name: obi
          image: otel/ebpf-instrument:main
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /config
              name: obi-config
          env:
            - name: OTEL_EBPF_CONFIG_PATH
              value: '/config/obi-config.yml'
```

이 구성에 대한 몇 가지 참고 사항:

- 컨테이너 이미지는 최신 개발 중인 `otel/ebpf-instrument:main` 이미지를
  사용한다.
- OBI는 노드당 하나의 OBI 인스턴스만 필요하므로 DaemonSet으로 실행해야 한다
- 호스트에서 네트워크 패킷을 수신 대기하려면 OBI에는 `hostNetwork: true` 권한이
  필요하다

### 네트워크 메트릭 생성 검증하기 {#verify-network-metrics-generation}

모든 것이 예상대로 작동하면, OBI 인스턴스는 네트워크 흐름(flow)을 캡처하고
처리하고 있어야 한다. 이를 테스트하려면 OBI DaemonSet의 로그를 확인하여 일부
디버그 정보가 출력되는지 본다.

```bash
kubectl logs daemonset/obi -n obi | head
```

출력은 다음과 같은 형태이다.

```text
network_flow: obi.ip=172.18.0.2 iface= direction=255 src.address=10.244.0.4 dst.address=10.96.0.1
```

### 오픈텔레메트리 엔드포인트로 메트릭 내보내기 {#export-metrics-to-opentelemetry-endpoint}

네트워크 메트릭이 수집되고 있음을 확인한 후, OBI가 메트릭을 오픈텔레메트리
형식으로 컬렉터 엔드포인트로 내보내도록 구성한다.

오픈텔레메트리 익스포터를 구성하려면
[데이터 내보내기 문서](/docs/zero-code/obi/configure/export-data#opentelemetry-metrics-exporter-component)를
확인한다.

### 허용되는 속성 {#allowed-attributes}

기본적으로, OBI는 `obi.network.flow.bytes` 메트릭에 다음 [속성](./)을 포함한다.

- `k8s.src.owner.name`
- `k8s.src.namespace`
- `k8s.dst.owner.name`
- `k8s.dst.namespace`
- `k8s.cluster.name`

OBI는 카디널리티 폭증을 피하기 위해 사용 가능한 속성의 일부만 포함한다.

예를 들면 다음과 같다.

```yaml
network:
  allowed_attributes:
    - k8s.src.owner.name
    - k8s.src.owner.type
    - k8s.dst.owner.name
    - k8s.dst.owner.type
```

이에 상응하는 Prometheus 메트릭은 다음과 같다.

```text
obi.network.flow.bytes:
  k8s_src_owner_name="frontend"
  k8s_src_owner_type="deployment"
  k8s_dst_owner_name="backend"
  k8s_dst_owner_type="deployment"
```

위 예시는 개별 파드 이름 대신 소스 및 대상 쿠버네티스 소유자 이름과 유형으로
`obi.network.flow.bytes` 값을 집계한다.

## CIDR 구성 {#cidr-configuration}

OBI를 구성하여 CIDR 범위별로도 메트릭을 세분화할 수 있다. 이는 클라우드 제공업체
IP 범위나 내부/외부 트래픽과 같은 특정 네트워크 범위로의 트래픽을 추적하는 데
유용하다.

`network`의 `cidrs` YAML 하위 섹션(또는 `OTEL_EBPF_NETWORK_CIDRS` 환경 변수)은
CIDR 범위 목록과 그에 해당하는 이름을 받는다.

예를 들어, 미리 정의된 네트워크별로 메트릭을 추적하려면 다음과 같이 한다.

```yaml
network:
  cidrs:
    - cidr: 10.0.0.0/8
      name: 'cluster-internal'
    - cidr: 192.168.0.0/16
      name: 'private'
    - cidr: 172.16.0.0/12
      name: 'container-internal'
```

그러면 이에 상응하는 Prometheus 메트릭은 다음과 같다.

```text
obi_network_flow_bytes:
  src_cidr="cluster-internal"
  dst_cidr="private"
```
