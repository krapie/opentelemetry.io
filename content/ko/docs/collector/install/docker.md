---
title: Docker로 컬렉터 설치하기
linkTitle: Docker
weight: 100
default_lang_commit: db3275691334b83b3fb6dd95a5dddb3b08b4a1c3
---

다음 명령은 Docker 이미지를 가져와 컨테이너에서 컬렉터를 실행한다.
`{{% param vers %}}`를 실행하려는 컬렉터 버전으로 바꾼다.

{{< tabpane text=true >}} {{% tab DockerHub %}}

```sh
docker pull otel/opentelemetry-collector:{{% param vers %}}
docker run otel/opentelemetry-collector:{{% param vers %}}
```

{{% /tab %}} {{% tab ghcr.io %}}

```sh
docker pull ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:{{% param vers %}}
docker run ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:{{% param vers %}}
```

{{% /tab %}} {{< /tabpane >}}

작업 디렉터리에서 커스텀 구성 파일을 불러오려면, 해당 파일을 볼륨(volume)으로
마운트한다.

{{< tabpane text=true >}} {{% tab DockerHub %}}

```sh
docker run -v $(pwd)/config.yaml:/etc/otelcol/config.yaml otel/opentelemetry-collector:{{% param vers %}}
```

{{% /tab %}} {{% tab ghcr.io %}}

```sh
docker run -v $(pwd)/config.yaml:/etc/otelcol/config.yaml ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:{{% param vers %}}
```

{{% /tab %}} {{< /tabpane >}}

## Docker Compose {#docker-compose}

기존 `docker-compose.yaml` 파일에 오픈텔레메트리(OpenTelemetry) 컬렉터를 추가할
수도 있다.

```yaml
otel-collector:
  image: otel/opentelemetry-collector
  volumes:
    - ./otel-collector-config.yaml:/etc/otelcol/config.yaml
  ports:
    - 1888:1888 # pprof extension
    - 8888:8888 # Prometheus metrics exposed by the Collector
    - 8889:8889 # Prometheus exporter metrics
    - 13133:13133 # health_check extension
    - 4317:4317 # OTLP gRPC receiver
    - 4318:4318 # OTLP http receiver
    - 55679:55679 # zpages extension
```

`otel-collector-config.yaml` 파일은 컬렉터가 시작하는 데 필요하다. 자세한 내용은
[컬렉터 구성](/docs/collector/configuration/)을 참고한다.

다음은 수신된 모든 텔레메트리를 로그로 남기는 최소한의 컬렉터 구성이다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
    logs:
      receivers: [otlp]
      exporters: [debug]
```
