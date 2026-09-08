---
title: 네트워크 메트릭
linkTitle: 네트워크
description:
  지점 간(point-to-point) 네트워크 메트릭을 관찰하도록 OBI를 구성한다.
weight: 8
cSpell:ignore: OpenShift replicaset statefulset
default_lang_commit: cae278629a885922cffa958f97745e25c32902aa
---

오픈텔레메트리 eBPF 계측(OpenTelemetry eBPF Instrumentation)은 서로 다른
엔드포인트 간의 네트워크 메트릭을 제공하도록 구성할 수 있다. 예를 들어 물리
노드, 컨테이너, 쿠버네티스 파드, 서비스 등의 사이가 해당된다.

## 시작하기 {#get-started}

OBI 네트워킹 메트릭 사용을 시작하려면 [빠른 시작 설정 문서](quickstart/)를
참고하고, 고급 구성을 위해서는 [구성 문서](config/)를 참고한다.

## 네트워크 메트릭 {#network-metrics}

OBI는 바이트 및 패킷 흐름(flow) 메트릭과 함께 존 간(inter-zone) 바이트 메트릭을
제공한다.

**흐름(flow) 메트릭**: 애플리케이션 관점에서 서로 다른 엔드포인트 간에 송수신된
바이트를 캡처한다.

- 오픈텔레메트리를 통해 내보내는(export) 경우 `obi.network.flow.bytes`
- Prometheus 엔드포인트로 내보내는 경우 `obi_network_flow_bytes_total`
- 활성화하려면 [OTEL_EBPF_METRICS_FEATURES](../configure/export-data/) 구성
  옵션에 `network` 옵션을 추가한다.

**흐름 패킷 메트릭**: 엔드포인트 간에 송수신된 패킷 수를 센다.

- 오픈텔레메트리를 통해 내보내는 경우 `obi.network.flow.packets`
- Prometheus 엔드포인트로 내보내는 경우 `obi_network_flow_packets_total`
- 활성화하려면 [OTEL_EBPF_METRICS_FEATURES](../configure/export-data/)에
  `network_flow_packets` 옵션을 추가한다.

**존 간(inter-zone) 메트릭**: 애플리케이션 관점에서 서로 다른 가용
영역(availability zone) 간에 송수신된 바이트를 캡처한다.

- 오픈텔레메트리를 통해 내보내는 경우 `obi.network.inter.zone.bytes`
- Prometheus 엔드포인트로 내보내는 경우 `obi_network_inter_zone_bytes_total`
- 활성화하려면 [OTEL_EBPF_METRICS_FEATURES](../configure/export-data/) 구성
  옵션에 `network_inter_zone` 옵션을 추가한다.

> [!NOTE]
>
> 이 메트릭은 호스트 관점에서 캡처되므로, 네트워크 스택의 오버헤드(프로토콜 헤더
> 등)를 포함한다.

기본적으로 네트워크 흐름 바이트에 대해서는 다음 속성만 보고된다.
`k8s.src.owner.name`, `k8s.src.namespace`, `k8s.dst.owner.name`,
`k8s.dst.namespace`, `k8s.cluster.name`.

존 간 바이트 메트릭의 경우, 기본 속성은 `k8s.cluster.name`, `src.zone`,
`dst.zone`이다.

## 메트릭 속성 {#metric-attributes}

네트워크 메트릭에는 다음 속성으로 레이블이 지정된다.

| 속성                                              | 설명                                                                                                                                                                                                          |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `direction`                                       | 수신 트래픽은 `ingress`, 송신 트래픽은 `egress`                                                                                                                                                               |
| `dst.address`                                     | 대상 IP 주소(egress의 경우 원격, ingress의 경우 로컬)                                                                                                                                                         |
| `dst.cidr`                                        | 대상 CIDR(구성된 경우)                                                                                                                                                                                        |
| `dst.name`                                        | 대상 서비스 이름(서비스 디스커버리로부터 확인됨)                                                                                                                                                              |
| `dst.port`                                        | 대상 포트(egress의 경우 원격, ingress의 경우 로컬)                                                                                                                                                            |
| `dst.zone` / `dst_zone`                           | 대상 클라우드 가용 영역의 이름                                                                                                                                                                                |
| `iface`                                           | 네트워크 인터페이스 이름                                                                                                                                                                                      |
| `k8s.cluster.name` / `k8s_cluster_name`           | 쿠버네티스 클러스터의 이름. OBI는 노드 레이블, OpenShift 인프라 메타데이터, Google Cloud, Microsoft Azure, Amazon Web Services를 확인한다. 감지 결과를 재정의하려면 `OTEL_EBPF_KUBE_CLUSTER_NAME`을 설정한다. |
| `k8s.dst.name` / `k8s_dst_name`                   | 대상 파드 이름                                                                                                                                                                                                |
| `k8s.dst.namespace` / `k8s_dst_namespace`         | 대상 네임스페이스 이름                                                                                                                                                                                        |
| `k8s.dst.node.ip` / `k8s_dst_node_ip`             | 대상 노드 IP 주소                                                                                                                                                                                             |
| `k8s.dst.node.name` / `k8s_dst_node_name`         | 대상 노드 이름                                                                                                                                                                                                |
| `k8s.dst.owner.name` / `k8s_dst_owner_name`       | 대상 워크로드 소유자 이름                                                                                                                                                                                     |
| `k8s.dst.owner.type` / `k8s_dst_owner_type`       | 대상 워크로드 소유자 유형: `replicaset`, `deployment`, `statefulset`, `daemonset`, `job`, `cronjob`, `node`                                                                                                   |
| `k8s.dst.type` / `k8s_dst_type`                   | 대상 워크로드 유형: `pod`, `replicaset`, `deployment`, `statefulset`, `daemonset`, `job`, `cronjob`, `node`                                                                                                   |
| `k8s.src.name` / `k8s_src_name`                   | 소스 파드 이름                                                                                                                                                                                                |
| `k8s.src.namespace` / `k8s_src_namespace`         | 소스 네임스페이스 이름                                                                                                                                                                                        |
| `k8s.src.node.ip` / `k8s_src_node_ip`             | 소스 노드 IP 주소                                                                                                                                                                                             |
| `k8s.src.node.name` / `k8s_src_node_name`         | 소스 노드 이름                                                                                                                                                                                                |
| `k8s.src.owner.name` / `k8s_src_owner_name`       | 소스 워크로드 소유자 이름                                                                                                                                                                                     |
| `k8s.src.owner.type` / `k8s_src_owner_type`       | 소스 워크로드 소유자 유형: `replicaset`, `deployment`, `statefulset`, `daemonset`, `job`, `cronjob`, `node`                                                                                                   |
| `k8s.src.type` / `k8s_src_type`                   | 소스 워크로드 유형: `pod`, `replicaset`, `deployment`, `statefulset`, `daemonset`, `job`, `cronjob`, `node`                                                                                                   |
| `network.protocol.name` / `network_protocol_name` | 네트워크 프로토콜 이름(예: `http` 또는 `https`)                                                                                                                                                               |
| `network.type` / `network_type`                   | 네트워크 유형(예: `ipv4` 또는 `ipv6`)                                                                                                                                                                         |
| `obi.ip` / `obi_ip`                               | 메트릭을 내보낸 OBI 인스턴스의 로컬 IP 주소                                                                                                                                                                   |
| `service.name`                                    | 계측된 엔드포인트와 연관된 로컬 서비스 이름                                                                                                                                                                   |
| `service.namespace`                               | 계측된 엔드포인트와 연관된 로컬 서비스 네임스페이스                                                                                                                                                           |
| `service.peer.name`                               | 대상 엔드포인트와 연관된 원격 피어 서비스 이름                                                                                                                                                                |
| `service.peer.namespace`                          | 대상 엔드포인트와 연관된 원격 피어 서비스 네임스페이스                                                                                                                                                        |
| `src.address`                                     | 소스 IP 주소(egress의 경우 로컬, ingress의 경우 원격)                                                                                                                                                         |
| `src.cidr`                                        | 소스 CIDR(구성된 경우)                                                                                                                                                                                        |
| `src.name`                                        | 소스 서비스 이름(서비스 디스커버리로부터 확인됨)                                                                                                                                                              |
| `src.port`                                        | 소스 포트(egress의 경우 로컬, ingress의 경우 원격)                                                                                                                                                            |
| `src.zone` / `src_zone`                           | 소스 클라우드 가용 영역의 이름                                                                                                                                                                                |
| `transport`                                       | 전송 프로토콜: `tcp`, `udp`                                                                                                                                                                                   |

## 메트릭 축소(reduction) {#metric-reduction}

카디널리티가 높은 경우를 줄이기 위해, 네트워크 메트릭은 프로세스 수준에서 사전
집계(pre-aggregate)되어 메트릭 백엔드로 전송되는 메트릭 수를 줄인다.

기본적으로 모든 메트릭은 다음 속성으로 집계된다.

- `direction`
- `transport`
- `src.address`
- `dst.address`
- `src.port`
- `dst.port`

OBI 구성에서 메트릭을 집계할 때 허용할 속성을 지정할 수 있다.

예를 들어 (기본값인 개별 파드 이름 대신) 소스 및 대상 쿠버네티스 소유자로
네트워크 메트릭을 집계하려면 다음 구성을 사용할 수 있다.

```yaml
network:
  allowed_attributes:
    - k8s.src.owner.name
    - k8s.dst.owner.name
    - k8s.src.owner.type
    - k8s.dst.owner.type
```

그러면 이에 상응하는 Prometheus 메트릭은 다음과 같다.

```text
obi_network_flow_bytes:
  k8s_src_owner_name="frontend"
  k8s_src_owner_type="deployment"
  k8s_dst_owner_name="backend"
  k8s_dst_owner_type="deployment"
```

위 예시는 개별 파드 이름 대신 소스 및 대상 쿠버네티스 소유자 이름과 유형으로
`obi.network.flow.bytes` 값을 집계한다.

## CIDR 기반 메트릭 {#cidr-based-metrics}

OBI를 구성하여 CIDR 범위별로도 메트릭을 세분화할 수 있다. 이는 클라우드 제공업체
IP 범위나 내부/외부 트래픽과 같은 특정 네트워크 범위로의 트래픽을 추적하는 데
유용하다.

`network`의 `cidrs` YAML 하위 섹션(또는 `OTEL_EBPF_NETWORK_CIDRS` 환경 변수)은
CIDR 범위 목록과 그에 해당하는 이름을 받는다. 예를 들면 다음과 같다.

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
