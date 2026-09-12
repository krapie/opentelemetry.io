---
title: 타겟 얼로케이터
description:
  배포된 모든 컬렉터 인스턴스에 PrometheusReceiver의 대상을 분배하는 도구이다.
cSpell:ignore: labeldrop labelmap statefulset
default_lang_commit: fe623719bc24346e9dcd77e9769026cf1c720cc5
---

오픈텔레메트리(OpenTelemetry) 오퍼레이터는 선택적 컴포넌트인
[타겟 얼로케이터(Target Allocator, TA)](https://github.com/open-telemetry/opentelemetry-operator/tree/main/cmd/otel-allocator)를
함께 제공한다. 한마디로, TA는 Prometheus의 서비스 디스커버리(service discovery)
기능과 메트릭 수집 기능을 분리해 서로 독립적으로 스케일링할 수 있게 해주는
메커니즘이다. 컬렉터는 Prometheus를 설치하지 않고도 Prometheus 메트릭을
관리한다. TA는 컬렉터의 Prometheus Receiver 구성을 관리한다.

TA는 다음 두 가지 기능을 수행한다.

1. Prometheus 대상을 여러 컬렉터에 고르게 분배
2. Prometheus 커스텀 리소스 검색(discovery)

## 시작하기 {#getting-started}

OpenTelemetryCollector 커스텀 리소스(Custom Resource, CR)를 생성하면서 TA를
활성화로 설정하면, 오퍼레이터는 해당 CR의 일부로 각 컬렉터 파드에 특정
`http_sd_config` 지시문을 제공하기 위한 새 디플로이먼트와 서비스를 생성한다.
또한 CR의 Prometheus 리시버 구성을 변경하여 TA의
[http_sd_config](https://prometheus.io/docs/prometheus/latest/http_sd/)를
사용하도록 만든다. 다음 예제는 타겟 얼로케이터를 시작하는 방법을 보여준다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: collector-with-ta
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs:
          - job_name: 'otel-collector'
            scrape_interval: 10s
            static_configs:
            - targets: [ '0.0.0.0:8888' ]
            metric_relabel_configs:
            - action: labeldrop
              regex: (id|name)
              replacement: $$1
            - action: labelmap
              regex: label_(.+)
              replacement: $$1

    exporters:
      # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
      debug:

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: []
          exporters: [debug]
```

오픈텔레메트리 오퍼레이터는 뒤에서 조정(reconciliation)을 거친 뒤 컬렉터의
구성을 다음과 같이 변환한다.

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: otel-collector
          scrape_interval: 10s
          http_sd_configs:
            - url: http://collector-with-ta-targetallocator:80/jobs/otel-collector/targets?collector_id=$POD_NAME
          metric_relabel_configs:
            - action: labeldrop
              regex: (id|name)
              replacement: $$1
            - action: labelmap
              regex: label_(.+)
              replacement: $$1

exporters:
  debug:

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: []
      exporters: [debug]
```

오퍼레이터가 `scrape_configs` 섹션에서 기존 서비스 디스커버리 구성(예:
`static_configs`, `file_sd_configs` 등)을 제거하고, 오퍼레이터가 프로비저닝한
타겟 얼로케이터 인스턴스를 가리키는 `http_sd_configs` 구성을 추가하는 것에
주목한다.

타겟 얼로케이터(TargetAllocator)에 대한 더 자세한 정보는
[TargetAllocator](https://github.com/open-telemetry/opentelemetry-operator/tree/main/cmd/otel-allocator)를
참고한다.
