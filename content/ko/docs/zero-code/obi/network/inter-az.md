---
title: 클라우드 가용 영역 간 트래픽 측정하기
linkTitle: 클라우드 가용 영역 간 트래픽 측정하기
description:
  서로 다른 클라우드 가용 영역(availability zone) 간의 네트워크 트래픽을
  측정하는 방법
weight: 1
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

> [!NOTE]
>
> 이 기능은 현재 쿠버네티스 클러스터에서만 사용할 수 있다.

클라우드 가용 영역(availability zone) 간 트래픽은 추가 비용이 발생할 수 있다.
OBI는 일반 네트워크 메트릭에 `src.zone` 및 `dst.zone` 속성을 추가하거나, 별도의
`obi.network.inter.zone.bytes`(OTel) /
`obi_network_inter_zone_bytes_total`(Prometheus) 메트릭을 제공하여 이를 측정할
수 있다.

## 일반 네트워크 메트릭에 `src.zone` 및 `dst.zone` 속성 추가하기 {#add-srczone-and-dstzone-attributes-to-regular-network-metrics}

소스 및 대상 가용 영역 속성은 OBI에서 기본적으로 비활성화되어 있다. 이를
활성화하려면 OBI YAML 구성의 포함된 네트워크 속성 목록에 명시적으로 추가한다.

```yaml
attributes:
  select:
    obi_network_flow_bytes:
      include:
        - k8s.src.owner.name
        - k8s.src.namespace
        - k8s.dst.owner.name
        - k8s.dst.namespace
        - k8s.cluster.name
        - src.zone
        - dst.zone
```

이 구성은 서로 다른 `src_zone` 및 `dst_zone` 속성을 가진 각
`obi_network_flow_bytes_total` 메트릭에 대해 존 간(inter-zone) 트래픽을 볼 수
있게 한다.

존 간 트래픽 측정에서 더 높은 세분성(예: 소스/대상 파드 또는 노드)이 필요한
경우, 존 속성을 추가하면 동일한 가용 영역 내의 트래픽에 대해서도 메트릭의
카디널리티에 영향을 준다.

## `obi.network.inter.zone` 메트릭 사용하기 {#use-the-obinetworkinterzone-metric}

존 간 트래픽에 별도의 메트릭을 사용하면 `src.zone` 및 `dst.zone` 속성이 일반
네트워크 메트릭에 추가되지 않으므로, 이 데이터를 수집하는 데 따른 메트릭
카디널리티 영향이 줄어든다.

`obi.network.inter.zone` 메트릭을 활성화하려면
[OTEL_EBPF_METRICS_FEATURES](../../configure/export-data/) 구성 옵션 또는 그에
상응하는 YAML 옵션에 `network_inter_zone` 옵션을 추가한다. 예를 들어 OBI가
오픈텔레메트리(OpenTelemetry)를 통해 메트릭을 내보내도록 구성된 경우 다음과
같다.

```yaml
metrics:
  features:
    - network
    - network_inter_zone
```

## 존 간 트래픽을 측정하는 PromQL 쿼리 {#promql-queries-to-measure-inter-zone-traffic}

`network`와 `network_inter_zone` 메트릭 패밀리가 모두 활성화되어 있다고
가정하면, 다음 PromQL 쿼리를 사용하여 존 간 트래픽을 측정할 수 있다.

전체 존 간 트래픽 처리량:

```promql
sum(rate(obi_network_inter_zone_bytes_total[$__rate_interval]))
```

소스 및 대상 존으로 요약한 존 간 트래픽 처리량:

```promql
sum(rate(obi_network_inter_zone_bytes_total[$__rate_interval])) by(src_zone,dst_zone)
```

전체 동일 존 트래픽 처리량:

```promql
sum(rate(obi_network_flow_bytes_total[$__rate_interval]))
  - sum(rate(obi_network_inter_zone_bytes_total[$__rate_interval]))
```

전체 대비 존 간 트래픽 비율:

```promql
100 * sum(rate(obi_network_inter_zone_bytes_total[$__rate_interval]))
  / sum(rate(obi_network_flow_bytes_total[$__rate_interval]))
```
