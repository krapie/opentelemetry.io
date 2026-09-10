---
title: OBI 내부 메트릭 리포터 구성
linkTitle: 내부 메트릭 리포터
description:
  선택적 내부 메트릭 리포터 구성 요소가 자동 계측 도구의 내부 동작에 대한
  메트릭을 Prometheus 형식으로 보고하는 방식을 구성한다.
weight: 80
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

YAML 섹션: `internal_metrics`

이 구성 요소는 자동 계측 도구의 동작에 대한 내부 메트릭을 보고한다. 이 메트릭은
[Prometheus](https://prometheus.io/)나 [오픈텔레메트리](/)를 사용하여 내보낼 수
있다.

Prometheus로 메트릭을 내보내려면 `internal_metrics` 섹션에서 `exporter`를
`prometheus`로 설정한다. 그런 다음 `prometheus` 하위 섹션에서 `port`를 설정한다.

오픈텔레메트리로 메트릭을 내보내려면 `internal_metrics` 섹션에서 `exporter`를
`otel`로 설정한다. 그런 다음 `otel_metrics_export`에서 엔드포인트를 설정한다.

예시:

```yaml
internal_metrics:
  exporter: prometheus
  prometheus:
    port: 6060
    path: /internal/metrics
```

## 구성 요약 {#configuration-summary}

| YAML                        | 환경 변수                                              | 유형    | 기본값              | 요약                                                              |
| --------------------------- | ------------------------------------------------------ | ------- | ------------------- | ----------------------------------------------------------------- |
| `exporter`                  | `OTEL_EBPF_INTERNAL_METRICS_EXPORTER`                  | string  | `disabled`          | [내부 메트릭 익스포터를 선택한다.](#internal-metrics-exporter)    |
| `prometheus.port`           | `OTEL_EBPF_INTERNAL_METRICS_PROMETHEUS_PORT`           | int     | (unset)             | [Prometheus 스크레이프 엔드포인트의 HTTP 포트.](#prometheus-port) |
| `prometheus.path`           | `OTEL_EBPF_INTERNAL_METRICS_PROMETHEUS_PATH`           | string  | `/internal/metrics` | [Prometheus 메트릭의 HTTP 쿼리 경로.](#prometheus-path)           |
| `avoided_services.disabled` | `OTEL_EBPF_INTERNAL_METRICS_AVOIDED_SERVICES_DISABLED` | boolean | `false`             | 회피된 서비스 메트릭을 비활성화한다.                              |
| `avoided_services.limit`    | `OTEL_EBPF_INTERNAL_METRICS_AVOIDED_SERVICES_LIMIT`    | int     | `2000`              | 오버플로 시리즈를 포함하여 회피된 서비스 시리즈를 제한한다.       |

---

## 내부 메트릭 익스포터 {#internal-metrics-exporter}

내부 메트릭 익스포터를 설정한다. `disabled`, `prometheus`, `otel`을 사용할 수
있다.

---

## Prometheus 포트 {#prometheus-port}

Prometheus 스크레이프 엔드포인트의 HTTP 포트를 설정한다. 설정하지 않거나 0으로
설정하면 OBI는 Prometheus 엔드포인트를 열지 않고 메트릭도 보고하지 않는다.

[`prometheus_export.port`](../export-data/#prometheus-exporter-component)와
동일한 값을 사용하거나(두 메트릭 계열이 동일한 HTTP 서버를 공유하지만 서로 다른
경로를 사용함), 다른 값을 사용할 수 있다(OBI는 서로 다른 메트릭 계열을 위해 두
개의 HTTP 서버를 연다).

---

## Prometheus 경로 {#prometheus-path}

Prometheus 메트릭을 가져올 HTTP 쿼리 경로를 설정한다.

[`prometheus_export.port`](../export-data/#prometheus-exporter-component)와
`internal_metrics.prometheus.port`가 동일한 값을 사용하는 경우,
`internal_metrics.prometheus.path`를 `prometheus_export.path`와 다른 값으로
설정하여 메트릭 계열을 분리하거나, 동일한 값을 사용하여 두 메트릭 계열을 동일한
스크레이프 엔드포인트에 나열할 수 있다.

## 회피된 서비스 카디널리티 {#avoided-services-cardinality}

`obi.avoided.services` OTLP 메트릭(Prometheus에서는 `obi_avoided_services`)은
서비스가 오픈텔레메트리 데이터를 직접 내보낸다는 것을 감지한 후 OBI가 중복
텔레메트리를 회피한 서비스를 보고한다. 시리즈에는 서비스 이름, 서비스
네임스페이스, 회피된 시그널(`metrics` 또는 `traces`)이 포함되지만, 고카디널리티
서비스 인스턴스 ID는 포함되지 않는다.

`avoided_services.limit`은 시리즈의 개수를 제한한다. 추가 서비스는
`otel.metric.overflow=true`(Prometheus: `otel_metric_overflow="true"`)를 가진
시리즈로 결합된다. 오픈텔레메트리 SDK의 기본 메트릭 카디널리티 제한을 사용하려면
제한을 `0`으로 설정하고, 이 메트릭을 생략하려면 `disabled: true`로 설정한다.
