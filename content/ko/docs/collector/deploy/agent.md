---
title: 에이전트 배포 패턴
linkTitle: 에이전트 패턴
description: 시그널을 컬렉터로 전송한 후 백엔드로 내보낸다
aliases: [/docs/collector/deployment/agent]
weight: 200
cSpell:ignore: prometheusremotewrite
default_lang_commit: 6cebc46de450dd44481a8a6f17c9b3d6f04aa0f2
---

에이전트 배포 패턴에서 텔레메트리 시그널은 다음으로부터 발생할 수 있다.

- [오픈텔레메트리 프로토콜(OpenTelemetry Protocol, OTLP)][otlp]을 사용하여
  오픈텔레메트리(OpenTelemetry) SDK로 [계측된][instrumentation] 애플리케이션
- OTLP 익스포터를 사용하는 컬렉터

시그널은 애플리케이션과 함께 실행되거나 동일한 호스트에서 실행되는
[컬렉터][collector] 인스턴스로 전송되며, 사이드카(sidecar)나 DaemonSet 형태로
실행될 수 있다.

각 클라이언트 사이드 SDK 또는 다운스트림 컬렉터는 컬렉터 인스턴스의 주소로
구성된다.

![탈중앙화된 컬렉터 배포 개념도](../../img/otel-agent-sdk.svg)

1. 애플리케이션에서는 SDK가 OTLP 데이터를 컬렉터로 전송하도록 구성된다.
1. 컬렉터는 텔레메트리 데이터를 하나 이상의 백엔드로 전송하도록 구성된다.

## 예제 {#example}

에이전트 배포 패턴의 이 예제에서는 먼저 오픈텔레메트리 Java SDK를 사용하여
[메트릭을 내보내는(export) Java 애플리케이션][instrument-java-metrics]을
수동으로 계측하며, 기본 `OTEL_METRICS_EXPORTER` 값인 `otlp`도 포함한다.
다음으로, 컬렉터의 주소로 [OTLP 익스포터][otlp-exporter]를 구성한다. 예를 들면
다음과 같다.

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=http://collector.example.com:4318
```

다음으로, `collector.example.com:4318`에서 실행 중인 컬렉터를 다음과 같이
구성한다.

{{< tabpane text=true >}} {{% tab Traces %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending traces to
    protocols:
      http:
        endpoint: 0.0.0.0:4318

exporters:
  otlp/jaeger: # Jaeger supports OTLP directly
    endpoint: https://jaeger.example.com:4317
    sending_queue:
      batch:

service:
  pipelines:
    traces/dev:
      receivers: [otlp]
      exporters: [otlp/jaeger]
```

{{% /tab %}} {{% tab Metrics %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending metrics to
    protocols:
      http:
        endpoint: 0.0.0.0:4318

exporters:
  prometheusremotewrite: # the PRW exporter, to ingest metrics to backend
    endpoint: https://prw.example.com/v1/api/remote_write
    sending_queue:
      batch:

service:
  pipelines:
    metrics/prod:
      receivers: [otlp]
      exporters: [prometheusremotewrite]
```

{{% /tab %}} {{% tab Logs %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending logs to
    protocols:
      http:
        endpoint: 0.0.0.0:4318

exporters:
  file: # the File Exporter, to ingest logs to local file
    path: ./app42_example.log
    rotation:

service:
  pipelines:
    logs/dev:
      receivers: [otlp]
      exporters: [file]
```

{{% /tab %}} {{< /tabpane >}}

이 패턴을 엔드투엔드로 살펴보려면 [Java][java-otlp-example] 또는
[Python][py-otlp-example] 예제를 참고한다.

## 트레이드오프 {#trade-offs}

에이전트 컬렉터를 사용할 때의 주요 장단점은 다음과 같다.

장점:

- 시작하기 간단하다.
- 애플리케이션과 컬렉터 간에 명확한 일대일 매핑이 존재한다.

단점:

- 팀과 인프라 리소스에 대한 확장성(scalability)이 제한적이다.
- 복잡하거나 계속 진화하는 배포 환경에는 유연하지 않다.

[instrumentation]: /docs/languages/
[otlp]: /docs/specs/otel/protocol/
[collector]: /docs/collector/
[instrument-java-metrics]: /docs/languages/java/api/#meterprovider
[otlp-exporter]: /docs/specs/otel/protocol/exporter/
[java-otlp-example]:
  https://github.com/open-telemetry/opentelemetry-java-docs/tree/main/otlp
[py-otlp-example]:
  https://opentelemetry-python.readthedocs.io/en/stable/examples/metrics/instruments/README.html
