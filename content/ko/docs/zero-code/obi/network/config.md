---
title: OBI 네트워크 메트릭 구성 옵션
linkTitle: 구성
description: OBI 네트워크 메트릭에 사용할 수 있는 구성 옵션을 알아본다
weight: 3
cSpell:ignore: BEETPH UDPLITE
default_lang_commit: c7a8125502fd5bba72c796ba76afd3c08a62ac39
---

네트워크 메트릭은 [OBI 구성 YAML 파일](../../configure/options/)의 `network`
속성 아래에서 구성하거나, `OTEL_EBPF_NETWORK_` 접두사가 붙은 환경 변수 집합으로
구성한다.

YAML 예시:

```yaml
network:
  enable: true
  cidrs:
    - 10.10.0.0/24
    - 10.0.0.0/8
    - 10.30.0.0/16
attributes:
  kubernetes:
    enable: true
  select:
    obi_network_flow_bytes:
      include:
        - k8s.src.owner.name
        - k8s.src.namespace
        - k8s.dst.owner.name
        - k8s.dst.namespace
        - src.cidr
        - dst.cidr
otel_metrics_export:
  endpoint: http://localhost:4318
```

`network` YAML 섹션 외에도, OBI 구성에는 네트워크 메트릭을 내보낼 엔드포인트가
필요하다(앞의 예시에서는 `otel_metrics_export`이지만
[Prometheus 엔드포인트](../../configure/options/)도 허용된다).

## 네트워크 메트릭 구성 속성 {#network-metrics-configuration-properties}

네트워크 메트릭을 활성화하려면 최상위
[메트릭 섹션](../../configure/export-data/#metrics-export-features)에 다음
`features` 중 하나를 추가한다.

- `network`는 `obi_network_flow_bytes` 메트릭을 활성화한다. 이는 클러스터 내 두
  엔드포인트 간의 바이트 수이다
- `network_inter_zone`은 `obi_network_inter_zone_bytes` 메트릭을 활성화한다.
  이는 클라우드 클러스터 내 서로 다른 가용 영역(availability zone) 간의 바이트
  수이다

> [!CAUTION]
>
> `obi_network_inter_zone_bytes` 명세는 현재 실험적(experimental)이며 쿠버네티스
> 클러스터에서만 사용할 수 있다. 이 명세는 확정되지 않았으며, 향후 OBI 버전에서
> 호환성이 깨지는 변경(breaking change)이 도입될 수 있다.

| YAML     | 환경 변수                  | 유형   | 기본값          |
| -------- | -------------------------- | ------ | --------------- |
| `source` | `OTEL_EBPF_NETWORK_SOURCE` | string | `socket_filter` |

OBI가 보고하는 네트워크 이벤트를 소싱하는 데 사용되는 Linux 커널 기능을
지정한다.

사용 가능한 옵션은 `tc`와 `socket_filter`이다.

`tc`가 이벤트 소스로 사용되면, OBI는 다이렉트 액션 모드(direct action mode)로
Linux Traffic Control의 ingress 및 egress 필터를 사용하여 네트워크 이벤트를
캡처한다. 이 이벤트 소스 모드는 다른 어떤 eBPF 프로그램도 다이렉트 액션 모드로
동일한 Linux Traffic Control 인터페이스에 연결하지 않는다고 가정한다. 예를 들어
Cilium 쿠버네티스 CNI가 동일한 방식을 사용하므로, 쿠버네티스 클러스터에 Cilium
CNI가 설치되어 있다면 OBI가 `socket_filter` 모드로 네트워크 이벤트를 캡처하도록
구성한다.

`socket_filter`가 이벤트 소스로 사용되면, OBI는 네트워크 이벤트를 캡처하기 위해
eBPF Linux 소켓 필터를 설치한다. 이 모드는 Linux Traffic Control의 egress 및
ingress 필터를 사용하는 Cilium CNI나 다른 eBPF 프로그램과 충돌하지 않는다.

| YAML    | 환경 변수                 | 유형                 | 기본값  |
| ------- | ------------------------- | -------------------- | ------- |
| `cidrs` | `OTEL_EBPF_NETWORK_CIDRS` | list of CIDR strings | (empty) |

CIDR 목록으로, 각각 `src.address`와 `dst.address`에 일치하는 항목으로 `src.cidr`
및 `dst.cidr` 속성에 설정된다.

이 속성은 소스 및 대상 IP 주소의 함수이다. IP 주소가 여기 있는 어떤 주소와도
일치하지 않으면 속성이 설정되지 않는다. IP 주소가 여러 CIDR 정의와 일치하면,
흐름(flow)은 가장 좁은 범위의 CIDR로 데코레이션(decoration)된다. 따라서 다른
어떤 CIDR과도 일치하지 않는 모든 트래픽을 그룹화하기 위해 `0.0.0.0/0` 항목을
안전하게 추가할 수 있다.

YAML에서 각 항목은 일반 CIDR 문자열이거나 `cidr` 및 `name` 필드를 가진 객체일 수
있다. `OTEL_EBPF_NETWORK_CIDRS` 환경 변수는 쉼표로 구분된 CIDR 문자열 목록만
허용한다. 예를 들면 다음과 같다.

```yaml
network:
  cidrs:
    - cidr: 10.0.0.0/8
      name: cluster-internal
    - 192.168.0.0/16
```

```sh
OTEL_EBPF_NETWORK_CIDRS=10.0.0.0/8,192.168.0.0/16
```

| YAML       | 환경 변수                    | 유형   | 기본값    |
| ---------- | ---------------------------- | ------ | --------- |
| `agent_ip` | `OTEL_EBPF_NETWORK_AGENT_IP` | string | (not set) |

각 메트릭에 보고되는 `obi.ip` 속성을 재정의할 수 있다. 설정하지 않으면, OBI는
지정된 네트워크 인터페이스에서 자신의 IP 주소를 자동으로 감지한다(다음 속성
참고).

| YAML             | 환경 변수                          | 유형   | 기본값     |
| ---------------- | ---------------------------------- | ------ | ---------- |
| `agent_ip_iface` | `OTEL_EBPF_NETWORK_AGENT_IP_IFACE` | string | `external` |

`obi.ip` 속성 값을 설정하기 위해 OBI가 자신의 IP 주소를 선택하는 데 사용할
인터페이스를 지정한다. 허용되는 값은 `external`(기본값), `local`, 또는
`name:<interface name>`(예: `name:eth0`)이다.

`agent_ip` 구성 속성이 설정되어 있으면 이 속성은 효과가 없다.

| YAML            | 환경 변수   | 유형   | 기본값 |
| --------------- | ----------- | ------ | ------ |
| `agent_ip_type` | `OTEL_EBPF` | string | `any`  |

OBI가 각 흐름의 `obi.ip` 필드에 보고할 IP 주소 유형(IPv4, IPv6 또는 둘 다)을
지정한다. 허용되는 값은 `any`(기본값), `ipv4`, `ipv6`이다. `agent_ip` 구성
속성이 설정되어 있으면 이 속성은 효과가 없다.

| YAML         | 환경 변수                      | 유형     | 기본값  |
| ------------ | ------------------------------ | -------- | ------- |
| `interfaces` | `OTEL_EBPF_NETWORK_INTERFACES` | []string | (empty) |

흐름을 수집할 인터페이스 이름이다. 비어 있으면, OBI는 `excluded_interfaces`(아래
참고)에 나열된 것을 제외하고 시스템의 모든 인터페이스를 가져온다. 항목이
슬래시로 묶여 있으면(예: `/br-/`) 정규 표현식으로 매칭되고, 그렇지 않으면
대소문자를 구분하는 문자열로 매칭된다.

이 속성을 환경 변수로 설정하는 경우 각 항목은 쉼표로 구분해야 한다. 예를 들면
다음과 같다.

```sh
OTEL_EBPF_NETWORK_INTERFACES=eth0,eth1,/^veth/
```

| YAML                 | 환경 변수                              | 유형     | 기본값 |
| -------------------- | -------------------------------------- | -------- | ------ |
| `exclude_interfaces` | `OTEL_EBPF_NETWORK_EXCLUDE_INTERFACES` | []string | `lo`   |

네트워크 흐름 트레이싱에서 제외할 인터페이스 이름이다. 기본값: `lo`(루프백).
항목이 슬래시로 묶여 있으면(예: `/br-/`) 정규 표현식으로 매칭되고, 그렇지 않으면
대소문자를 구분하는 문자열로 매칭된다.

이 속성을 환경 변수로 설정하는 경우 각 항목은 쉼표로 구분해야 한다. 예를 들면
다음과 같다.

```sh
OTEL_BPF_NETWORK_EXCLUDE_INTERFACES=lo,/^veth/
```

| YAML        | 환경 변수                     | 유형     | 기본값  |
| ----------- | ----------------------------- | -------- | ------- |
| `protocols` | `OTEL_EBPF_NETWORK_PROTOCOLS` | []string | (empty) |

설정하면, OBI는 보고된 인터넷 프로토콜(Internet Protocol)이 이 목록에 없는 모든
네트워크 흐름을 버린다.

허용되는 값은 Linux의
[표준으로 잘 정의된 IP 프로토콜](https://elixir.bootlin.com/linux/v6.8.7/source/include/uapi/linux/in.h#L28)
열거형에 정의되어 있으며, 다음이 될 수 있다: `TCP`, `UDP`, `IP`, `ICMP`, `IGMP`,
`IPIP`, `EGP`, `PUP`, `IDP`, `TP`, `DCCP`, `IPV6`, `RSVP`, `GRE`, `ESP`, `AH`,
`MTP`, `BEETPH`, `ENCAP`, `PIM`, `COMP`, `L2TP`, `SCTP`, `UDPLITE`, `MPLS`,
`ETHERNET`, `RAW`

| YAML                | 환경 변수                             | 유형     | 기본값  |
| ------------------- | ------------------------------------- | -------- | ------- |
| `exclude_protocols` | `OTEL_EBPF_NETWORK_EXCLUDE_PROTOCOLS` | []string | (empty) |

설정하면, OBI는 보고된 인터넷 프로토콜이 이 목록에 있는 모든 네트워크 흐름을
버린다.

`protocols`/`OTEL_EBPF_NETWORK_PROTOCOLS` 목록이 이미 설정되어 있으면 이 속성은
무시된다.

허용되는 값은 Linux의
[표준으로 잘 정의된 IP 프로토콜](https://elixir.bootlin.com/linux/v6.8.7/source/include/uapi/linux/in.h#L28)
열거형에 정의되어 있으며, 다음이 될 수 있다: `TCP`, `UDP`, `IP`, `ICMP`, `IGMP`,
`IPIP`, `EGP`, `PUP`, `IDP`, `TP`, `DCCP`, `IPV6`, `RSVP`, `GRE`, `ESP`, `AH`,
`MTP`, `BEETPH`, `ENCAP`, `PIM`, `COMP`, `L2TP`, `SCTP`, `UDPLITE`, `MPLS`,
`ETHERNET`, `RAW`

| YAML              | 환경 변수                           | 유형    | 기본값 |
| ----------------- | ----------------------------------- | ------- | ------ |
| `cache_max_flows` | `OTEL_EBPF_NETWORK_CACHE_MAX_FLOWS` | integer | `5000` |

이후 내보내기를 위해 플러시되기 전에 어카운팅 캐시(accounting cache)에 누적될 수
있는 흐름 수를 지정한다. 기본값은 5000이다. OBI 로그에서 "received message
larger than max" 오류가 보이면 이 값을 줄인다.

| YAML                   | 환경 변수                                | 유형     | 기본값 |
| ---------------------- | ---------------------------------------- | -------- | ------ |
| `cache_active_timeout` | `OTEL_EBPF_NETWORK_CACHE_ACTIVE_TIMEOUT` | duration | `5s`   |

이후 내보내기를 위해 플러시되기 전에 흐름이 어카운팅 캐시에 유지되는 최대 시간을
지정한다.

| YAML        | 환경 변수                     | 유형   | 기본값 |
| ----------- | ----------------------------- | ------ | ------ |
| `direction` | `OTEL_EBPF_NETWORK_DIRECTION` | string | `both` |

흐름이 캡처되는 인터페이스에서의 방향에 따라 트레이싱할 흐름을 선택할 수 있다.
허용되는 값은 `ingress`, `egress`, 또는 `both`(기본값)이다.

> [!NOTE]
>
> 이 맥락에서 _ingress_ 또는 _egress_는 노드나 클러스터 외부에서 들어오거나
> 나가는 트래픽과 관련된 것이 아니라, 네트워크 인터페이스와 관련된 것이다. 즉,
> 동일한 네트워크 패킷이 가상 네트워크 장치에서는 "ingress"로, 그 기반이 되는
> 물리 네트워크 인터페이스에서는 "egress"로 보일 수 있다.

| YAML       | 환경 변수                    | 유형    | 기본값         |
| ---------- | ---------------------------- | ------- | -------------- |
| `sampling` | `OTEL_EBPF_NETWORK_SAMPLING` | integer | `0` (disabled) |

패킷을 샘플링하여 대상 컬렉터로 전송하는 비율이다. 예를 들어 100으로 설정하면,
평균적으로 100개 패킷 중 1개가 대상 컬렉터로 전송된다.

| YAML          | 환경 변수                       | 유형    | 기본값  |
| ------------- | ------------------------------- | ------- | ------- |
| `print_flows` | `OTEL_EBPF_NETWORK_PRINT_FLOWS` | boolean | `false` |

`true`로 설정하면, OBI는 각 네트워크 흐름을 표준 출력에 출력한다. 참고로, 이로
인해 많은 양의 출력이 생성될 수 있다.

| YAML          | 환경 변수                       | 유형   | 기본값    |
| ------------- | ------------------------------- | ------ | --------- |
| `guess_ports` | `OTEL_EBPF_NETWORK_GUESS_PORTS` | string | `disable` |

> [!IMPORTANT]
>
> v0.7.0에서 네트워크 포트 추측은 이제 **기본적으로 비활성화**된다. 이는 v0.6.0
> 및 이전 버전과 비교하여 호환성이 깨지는 변경(breaking change)이다. OBI가
> 개시자(initiator)를 판별할 수 없는 흐름에 대해 추론된 클라이언트/서버 포트에
> 의존하고 있다면, 서수 추측(ordinal guessing)을 명시적으로 다시 활성화하지 않는
> 한 `client.port`와 `server.port`가 이제 비어 있을 수 있다.

흐름 메타데이터에서 개시자를 판별할 수 없을 때 OBI가 서수 휴리스틱(ordinal
heuristics)을 기반으로 클라이언트 및 서버 포트를 추측하려고 시도해야 하는지를
지정한다. 이는 포트만으로 서비스를 식별하는 데 도움이 될 수 있는, 알려지지 않은
서비스로의 커넥션을 추적하는 데 유용하다.

허용되는 값은 `disable`(기본값), `ordinal`이다.

서수 휴리스틱 기반 포트 추측을 다시 활성화하려면 다음을 사용한다.

```yaml
network:
  guess_ports: ordinal
```

또는 환경 변수를 통해:

```sh
OTEL_EBPF_NETWORK_GUESS_PORTS=ordinal
```
