---
title: OBI Prometheus 및 오픈텔레메트리 메트릭 히스토그램 구성
linkTitle: 메트릭 히스토그램
description:
  Prometheus 및 오픈텔레메트리(OpenTelemetry) 메트릭 히스토그램을 구성하고,
  네이티브 히스토그램과 지수 히스토그램을 사용할지 여부를 설정한다.
weight: 60
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

OBI Prometheus 및 오픈텔레메트리(OpenTelemetry) 메트릭 히스토그램을 구성할 수
있다. 네이티브 히스토그램과 지수 히스토그램(exponential histogram)을 사용하도록
선택할 수도 있다.

## 히스토그램 버킷 재정의 {#override-histogram-buckets}

`buckets` YAML 구성 옵션을 설정하여 오픈텔레메트리 및 Prometheus 메트릭
익스포터의 히스토그램 버킷 경계를 재정의할 수 있다.

YAML 섹션: `otel_metrics_export.buckets`

예를 들면 다음과 같다.

```yaml
otel_metrics_export:
  buckets:
    duration_histogram: [0, 1, 2]
```

| YAML                 | 유형        |
| -------------------- | ----------- |
| `duration_histogram` | `[]float64` |

요청 지속 시간(duration)과 관련된 메트릭의 버킷 경계를 설정한다. 구체적으로는
다음과 같다.

- `http.server.request.duration`(OTel) / `http_server_request_duration_seconds`
  (Prometheus)
- `http.client.request.duration`(OTel) / `http_client_request_duration_seconds`
  (Prometheus)
- `rpc.server.call.duration`(OTel) / `rpc_server_call_duration_seconds`
  (Prometheus)
- `rpc.client.call.duration`(OTel) / `rpc_client_call_duration_seconds`
  (Prometheus)

값을 설정하지 않으면 OBI는
[오픈텔레메트리 시맨틱 컨벤션](/docs/specs/semconv/http/http-metrics/)의 기본
버킷 경계를 사용한다.

```text
0, 0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10
```

YAML 섹션: `prometheus_export.buckets`

```yaml
prometheus_export:
  buckets:
    request_size_histogram: [0, 10, 20, 22]
    response_size_histogram: [0, 10, 20, 22]
```

| YAML                      | 유형        |
| ------------------------- | ----------- |
| `request_size_histogram`  | `[]float64` |
| `response_size_histogram` | `[]float64` |

요청 및 응답 크기와 관련된 메트릭의 버킷 경계를 설정한다.

- `http.server.request.body.size`(OTel) / `http_server_request_body_size_bytes`
  (Prometheus)
- `http.client.request.body.size`(OTel) / `http_client_request_body_size_bytes`
  (Prometheus)
- `http.server.response.body.size`(OTel) /
  `http_server_response_body_size_bytes`(Prometheus)
- `http.client.response.body.size`(OTel) /
  `http_client_response_body_size_bytes`(Prometheus)

값을 설정하지 않으면 OBI는 다음 기본 버킷 경계를 사용한다.

```text
0, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192
```

이 기본값은 불안정(UNSTABLE)하며, Prometheus나 오픈텔레메트리 시맨틱 컨벤션이
다른 버킷 경계를 권장하면 변경될 수 있다.

### TCP RTT 히스토그램 {#tcp-rtt-histogram}

두 익스포터 중 하나의 `buckets` 섹션 아래에서 `stat_tcp_rtt_histogram`을
사용하여 `obi.stat.tcp.rtt` / `obi_stat_tcp_rtt_seconds`의 명시적 경계를
설정한다.

```yaml
otel_metrics_export:
  buckets:
    stat_tcp_rtt_histogram: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
```

기본 경계는 초 단위로 다음과 같다.

```text
0.0005, 0.001, 0.002, 0.005, 0.010, 0.025, 0.050, 0.100, 0.250, 0.500, 1.0
```

`histogram_aggregation`이 `base2_exponential_bucket_histogram`이면
오픈텔레메트리 익스포터는 명시적 경계를 무시한다.

## 네이티브 히스토그램과 지수 히스토그램 사용 {#use-native-histograms-and-exponential-histograms}

Prometheus의 경우
[Prometheus 컬렉터에서 `native-histograms` 기능을 활성화](https://web.archive.org/web/20250102120553/https://prometheus.io/docs/prometheus/latest/feature_flags/#native-histograms)하여
[네이티브 히스토그램](https://prometheus.io/docs/concepts/metric_types/#histogram)을
활성화한다.

오픈텔레메트리의 경우, 버킷을 수동으로 정의하는 대신 미리 정의된 히스토그램에
[지수 히스토그램](/docs/specs/otel/metrics/data-model/#exponentialhistogram)을
사용할 수 있다. 표준
[OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION](/docs/specs/otel/metrics/sdk_exporters/otlp/#additional-environment-variable-configuration)
환경 변수를 설정한다. 자세한 내용은 [OTel 메트릭 익스포터](../export-data/)
섹션의 `histogram_aggregation` 섹션을 참고한다.
