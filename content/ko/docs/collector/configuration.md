---
title: 구성
weight: 20
description: 필요에 맞게 컬렉터를 구성하는 방법을 알아본다
# prettier-ignore
cSpell:ignore: cfssl cfssljson configtls fluentforward gencert genkey initca oidc pprof prodevent prometheusremotewrite spanevents unredacted upsert zpages
default_lang_commit: 30b7dbbdd94cec0b2a0c99317272b103315518bf
---

<!-- markdownlint-disable link-fragments -->

오픈텔레메트리(OpenTelemetry) 컬렉터는 옵저버빌리티(observability) 요구 사항에
맞게 설정할 수 있다. 컬렉터 설정이 어떻게 동작하는지 알아보기 전에 다음 내용을
먼저 살펴본다.

- [데이터 수집 개념][dcc]을 참고해 오픈텔레메트리 컬렉터에 적용되는
  저장소(repository)를 이해한다.
- [최종 사용자를 위한 보안 가이드](/docs/security/config-best-practices/)
- [구성 요소 개발자를 위한 보안 가이드](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md)

## 위치 {#location}

기본적으로 컬렉터 설정은 `/etc/<otel-directory>/config.yaml`에 위치하며,
`<otel-directory>`는 사용 중인 컬렉터 버전이나 컬렉터 배포판(distribution)에
따라 `otelcol`, `otelcol-contrib`, 또는 다른 값이 될 수 있다.

`--config` 옵션을 사용하면 하나 이상의 설정을 지정할 수 있다. 예를 들면 다음과
같다.

```shell
otelcol --config=customconfig.yaml
```

`--config` 플래그는 파일 경로 또는 `"<scheme>:<opaque_data>"` 형태의 설정 URI
값을 받을 수 있다. 현재 오픈텔레메트리 컬렉터는 `scheme`에 대해 다음
제공자(provider)를 지원한다.

- **file** - 파일에서 설정을 읽는다. 예: `file:path/to/config.yaml`.
- **env** - 환경 변수에서 설정을 읽는다. 예: `env:MY_CONFIG_IN_AN_ENVVAR`.
- **yaml** - YAML 문자열에서 설정을 읽으며, `::`로 하위 경로를 구분한다. 예:
  `yaml:exporters::debug::verbosity: detailed`.

<!-- prettier-ignore-start -->
- **http** - HTTP URI에서 설정을 읽는다. 예: `http://www.example.com`
- **https** - HTTPS URI에서 설정을 읽는다. 예:
`https://www.example.com`
<!-- prettier-ignore-end -->

여러 경로에 있는 여러 파일을 사용하여 다중 설정을 제공할 수도 있다. 각 파일은
전체 설정이거나 부분 설정일 수 있으며, 파일들은 서로의 구성 요소를 참조할 수
있다. 파일을 병합한 결과가 완전한 설정을 구성하지 못하면, 필수 구성 요소가
기본적으로 추가되지 않으므로 사용자는 오류를 받게 된다. 명령줄에서 다음과 같이
여러 파일 경로를 전달한다.

```shell
otelcol --config=file:/path/to/first/file --config=file:/path/to/second/file
```

환경 변수, HTTP URI, 또는 YAML 경로를 사용하여 설정을 제공할 수도 있다. 예를
들면 다음과 같다.

```shell
otelcol --config=env:MY_CONFIG_IN_AN_ENVVAR --config=https://server/config.yaml
otelcol --config="yaml:exporters::debug::verbosity: normal"
```

> [!TIP]
>
> YAML 경로에서 중첩된 키를 참조할 때는, 점(dot)을 포함하는 네임스페이스와
> 혼동하지 않도록 반드시 이중 콜론(`::`)을 사용한다. 예:
> `receivers::docker_stats::metrics::container.cpu.utilization::enabled: false`.

설정 파일을 검증하려면 `validate` 명령어를 사용한다. 예를 들면 다음과 같다.

```shell
otelcol validate --config=customconfig.yaml
```

## 설정 구조 {#basics}

모든 컬렉터 설정 파일의 구조는 텔레메트리 데이터에 접근하는 네 가지 종류의
파이프라인 구성 요소로 이루어진다.

- [리시버(Receivers)](#receivers)
  <img width="32" alt="" class="img-initial otel-icon" src="/img/logos/32x32/Receivers.svg">
- [프로세서(Processors)](#processors)
  <img width="32" alt="" class="img-initial otel-icon" src="/img/logos/32x32/Processors.svg">
- [익스포터(Exporters)](#exporters)
  <img width="32" alt="" class="img-initial otel-icon" src="/img/logos/32x32/Exporters.svg">
- [커넥터(Connectors)](#connectors)
  <img width="32" alt="" class="img-initial otel-icon" src="/img/logos/32x32/Load_Balancer.svg">

각 파이프라인 구성 요소를 설정한 후에는, 설정 파일의 [service](#service) 섹션 내
파이프라인을 통해 이를 활성화해야 한다.

파이프라인 구성 요소 외에도, 진단 도구와 같이 컬렉터에 추가할 수 있는 기능을
제공하는 [익스텐션(Extensions)](#extensions)을 설정할 수 있다. 익스텐션은
텔레메트리 데이터에 직접 접근할 필요가 없으며, [service](#service) 섹션을 통해
활성화된다.

<a id="endpoint-0.0.0.0-warning"></a> 다음은 리시버 하나, 프로세서 하나,
익스포터 하나, 그리고 익스텐션 세 개로 구성된 컬렉터 설정 예시이다.

> [!WARNING]
>
> 모든 클라이언트가 로컬에 있는 경우 일반적으로 엔드포인트를 `localhost`에
> 바인딩하는 것이 좋지만, 이 문서의 예시 설정에서는 편의를 위해 "지정되지
> 않음(unspecified)" 주소인 `0.0.0.0`을 사용한다. 컬렉터의 기본값은
> `localhost`이다. 엔드포인트 설정 값으로 이 두 가지 선택지 중 하나를 사용하는
> 것에 대한 자세한 내용은 [Safeguards against denial of service attacks][]를
> 참고한다.

[Safeguards against denial of service attacks]:
  https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md#safeguards-against-denial-of-service-attacks

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  otlp_grpc:
    endpoint: otelcol:4317
    sending_queue:
      batch:

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp_grpc]
    metrics:
      receivers: [otlp]
      exporters: [otlp_grpc]
    logs:
      receivers: [otlp]
      exporters: [otlp_grpc]
```

리시버, 프로세서, 익스포터, 파이프라인은 `type[/name]` 형식을 따르는 구성 요소
식별자를 통해 정의된다는 점에 유의한다. 예를 들면 `otlp` 또는 `otlp/2`와 같다.
식별자가 고유하기만 하면 동일한 타입의 구성 요소를 여러 번 정의할 수 있다. 예를
들면 다음과 같다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  otlp/2:
    protocols:
      grpc:
        endpoint: 0.0.0.0:55690

exporters:
  otlp_grpc:
    endpoint: otelcol:4317
    sending_queue:
      batch:
  otlp_grpc/2:
    endpoint: otelcol2:4317
    sending_queue:
      batch:

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp_grpc]
    traces/2:
      receivers: [otlp/2]
      exporters: [otlp_grpc/2]
    metrics:
      receivers: [otlp]
      exporters: [otlp_grpc]
    logs:
      receivers: [otlp]
      exporters: [otlp_grpc]
```

설정에는 다른 파일을 포함시킬 수도 있으며, 이 경우 컬렉터는 이를 단일 인메모리
YAML 설정 표현으로 병합한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters: ${file:exporters.yaml}

service:
  extensions: []
  pipelines:
    traces:
      receivers: [otlp]
      processors: []
      exporters: [otlp_grpc]
```

`exporters.yaml` 파일의 내용이 다음과 같다면:

```yaml
otlp_grpc:
  endpoint: otelcol.observability.svc.cluster.local:443
```

메모리 상의 최종 결과는 다음과 같다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  otlp_grpc:
    endpoint: otelcol.observability.svc.cluster.local:443

service:
  extensions: []
  pipelines:
    traces:
      receivers: [otlp]
      processors: []
      exporters: [otlp_grpc]
```

## 리시버 <img width="35" class="img-initial otel-icon" alt="" src="/img/logos/32x32/Receivers.svg"> {#receivers}

리시버는 하나 이상의 소스로부터 텔레메트리를 수집(collect)한다. 풀(pull) 방식
또는 푸시(push) 방식일 수 있으며, 하나 이상의
[데이터 소스](/docs/concepts/signals/)를 지원할 수 있다.

리시버는 `receivers` 섹션에서 설정된다. 많은 리시버가 기본 설정을 제공하므로,
리시버 이름만 지정해도 설정할 수 있다. 리시버를 설정하거나 기본 설정을 변경하고
싶다면 이 섹션에서 할 수 있다. 지정한 설정은 존재하는 경우 기본값을 재정의한다.

> 리시버를 설정한다고 해서 자동으로 활성화되는 것은 아니다. 리시버는
> [service](#service) 섹션 내 적절한 파이프라인에 추가함으로써 활성화된다.

컬렉터는 하나 이상의 리시버를 필요로 한다. 다음 예시는 동일한 설정 파일 내
다양한 리시버를 보여준다.

```yaml
receivers:
  # Data sources: logs
  fluentforward:
    endpoint: 0.0.0.0:8006

  # Data sources: metrics
  host_metrics:
    scrapers:
      cpu:
      disk:
      filesystem:
      load:
      memory:
      network:
      process:
      processes:
      paging:

  # Data sources: traces
  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      thrift_binary:
      thrift_compact:
      thrift_http:

  # Data sources: traces, metrics, logs
  kafka:
    protocol_version: 2.0.0

  # Data sources: traces, metrics
  opencensus:

  # Data sources: traces, metrics, logs
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: cert.pem
          key_file: cert-key.pem
      http:
        endpoint: 0.0.0.0:4318

  # Data sources: metrics
  prometheus:
    config:
      scrape_configs:
        - job_name: otel-collector
          scrape_interval: 5s
          static_configs:
            - targets: [localhost:8888]

  # Data sources: traces
  zipkin:
```

> 자세한 리시버 설정은
> [리시버 README](https://github.com/open-telemetry/opentelemetry-collector/blob/main/receiver/README.md)를
> 참고한다.

## 프로세서 <img width="35" class="img-initial otel-icon" alt="" src="/img/logos/32x32/Processors.svg"> {#processors}

프로세서는 리시버가 수집한 데이터를 익스포터로 보내기 전에 수정하거나 변환한다.
데이터 처리는 각 프로세서에 정의된 규칙이나 설정에 따라 이루어지며, 여기에는
필터링, 삭제, 이름 변경, 텔레메트리 재계산 등의 작업이 포함될 수 있다.
파이프라인 내 프로세서의 순서는 컬렉터가 시그널에 적용하는 처리 작업의 순서를
결정한다.

프로세서는 선택 사항이지만, 일부는
[권장된다](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor#recommended-processors).

컬렉터 설정 파일의 `processors` 섹션을 사용하여 프로세서를 설정할 수 있다.
지정한 설정은 존재하는 경우 기본값을 재정의한다.

> 프로세서를 설정한다고 해서 자동으로 활성화되는 것은 아니다. 프로세서는
> [service](#service) 섹션 내 적절한 파이프라인에 추가함으로써 활성화된다.

다음 예시는 동일한 설정 파일 내 여러 기본 프로세서를 보여준다. 전체 프로세서
목록은
[opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor)
목록과
[opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor)
목록을 결합하여 확인할 수 있다.

```yaml
processors:
  # Data sources: traces
  attributes:
    actions:
      - key: environment
        value: production
        action: insert
      - key: db.statement
        action: delete
      - key: email
        action: hash

  # Data sources: traces, metrics, logs
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["container.name"] == "app_container_1"'
        - 'resource.attributes["host.name"] == "localhost"'
        - 'name == "app_3"'
      spanevent:
        - 'attributes["grpc"] == true'
        - 'IsMatch(name, ".*grpc.*")'
    metrics:
      metric:
        - 'name == "my.metric" and resource.attributes["my_label"] == "abc123"'
        - 'type == METRIC_DATA_TYPE_HISTOGRAM'
      datapoint:
        - 'metric.type == METRIC_DATA_TYPE_SUMMARY'
        - 'resource.attributes["service.name"] == "my_service_name"'
    logs:
      log_record:
        - 'IsMatch(body, ".*password.*")'
        - 'severity_number < SEVERITY_NUMBER_WARN'

  # Data sources: traces, metrics, logs
  memory_limiter:
    check_interval: 5s
    limit_mib: 4000
    spike_limit_mib: 500

  # Data sources: traces
  resource:
    attributes:
      - key: cloud.zone
        value: zone-1
        action: upsert
      - key: k8s.cluster.name
        from_attribute: k8s-cluster
        action: insert
      - key: redundant-attribute
        action: delete

  # Data sources: traces
  probabilistic_sampler:
    hash_seed: 22
    sampling_percentage: 15

  # Data sources: traces
  span:
    name:
      to_attributes:
        rules:
          - ^\/api\/v1\/document\/(?P<documentId>.*)\/update$
      from_attributes: [db.svc, operation]
      separator: '::'
```

> 자세한 프로세서 설정은
> [프로세서 README](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/README.md)를
> 참고한다.

## 익스포터 <img width="35" class="img-initial otel-icon" alt="" src="/img/logos/32x32/Exporters.svg"> {#exporters}

익스포터는 하나 이상의 백엔드 또는 목적지로 데이터를 전송한다. 익스포터는 풀
방식 또는 푸시 방식일 수 있으며, 하나 이상의
[데이터 소스](/docs/concepts/signals/)를 지원할 수 있다.

`exporters` 섹션 내 각 키는 익스포터 인스턴스를 정의한다. 이 키는 `type/name`
형식을 따르며, `type`은 익스포터 타입(예: `otlp`, `kafka`, `prometheus`)을
지정하고, `name`(선택 사항)을 추가해 동일한 타입의 여러 인스턴스에 고유한 이름을
부여할 수 있다.

대부분의 익스포터는 최소한 목적지를 지정하는 설정과, 인증 토큰이나 TLS 인증서와
같은 보안 설정을 필요로 한다. 지정한 설정은 존재하는 경우 기본값을 재정의한다.

> 익스포터를 설정한다고 해서 자동으로 활성화되는 것은 아니다. 익스포터는
> [service](#service) 섹션 내 적절한 파이프라인에 추가함으로써 활성화된다.

컬렉터는 하나 이상의 익스포터를 필요로 한다. 다음 예시는 동일한 설정 파일 내
다양한 익스포터를 보여준다.

```yaml
exporters:
  # Data sources: traces, metrics, logs
  file:
    path: ./filename.json

  # Data sources: traces
  otlp_grpc/jaeger:
    endpoint: jaeger-server:4317
    tls:
      cert_file: cert.pem
      key_file: cert-key.pem

  # Data sources: traces, metrics, logs
  kafka:
    protocol_version: 2.0.0

  # Data sources: traces, metrics, logs
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`
  debug:
    verbosity: detailed

  # Data sources: traces, metrics
  opencensus:
    endpoint: otelcol2:55678

  # Data sources: traces, metrics, logs
  otlp_grpc:
    endpoint: otelcol2:4317
    tls:
      cert_file: cert.pem
      key_file: cert-key.pem

  # Data sources: traces, metrics
  otlp_http:
    endpoint: https://otlp.example.com:4318

  # Data sources: metrics
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: default

  # Data sources: metrics
  prometheusremotewrite:
    endpoint: http://prometheus.example.com:9411/api/prom/push
    # When using the official Prometheus (running via Docker)
    # endpoint: 'http://prometheus:9090/api/v1/write', add:
    # tls:
    #   insecure: true

  # Data sources: traces
  zipkin:
    endpoint: http://zipkin.example.com:9411/api/v2/spans
```

일부 익스포터는 [인증서 설정](#setting-up-certificates)에서 설명하는 것처럼,
보안 연결을 수립하기 위해 x.509 인증서를 필요로 한다는 점에 유의한다.

> 익스포터 설정에 대한 자세한 내용은
> [익스포터 README.md](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/README.md)를
> 참고한다.

## 커넥터 <img width="32" class="img-initial otel-icon" alt="" src="/img/logos/32x32/Load_Balancer.svg"> {#connectors}

커넥터는 익스포터이자 리시버 역할을 동시에 수행하며 두 파이프라인을 연결한다.
커넥터는 한 파이프라인의 끝에서 익스포터로서 데이터를 소비하고, 다른
파이프라인의 시작 지점에서 리시버로서 데이터를 내보낸다. 소비되고 내보내지는
데이터는 동일한 타입일 수도, 서로 다른 데이터 타입일 수도 있다. 커넥터를
사용하여 소비한 데이터를 요약하거나, 복제하거나, 라우팅할 수 있다.

컬렉터 설정 파일의 `connectors` 섹션을 사용하여 하나 이상의 커넥터를 설정할 수
있다. 기본적으로는 어떤 커넥터도 설정되어 있지 않다. 각 커넥터 타입은 하나
이상의 데이터 타입 쌍과 함께 동작하도록 설계되며, 그에 맞는 파이프라인을
연결하는 데만 사용할 수 있다.

> 커넥터를 설정한다고 해서 자동으로 활성화되는 것은 아니다. 커넥터는
> [service](#service) 섹션 내 파이프라인을 통해 활성화된다.

다음 예시는 `count` 커넥터와 이를 `pipelines` 섹션에서 설정하는 방법을 보여준다.
이 커넥터는 트레이스에 대해서는 익스포터로, 메트릭에 대해서는 리시버로 동작하여
두 파이프라인을 연결한다는 점에 유의한다.

```yaml
receivers:
  foo:

exporters:
  bar:

connectors:
  count:
    spanevents:
      my.prod.event.count:
        description: The number of span events from my prod environment.
        conditions:
          - 'attributes["env"] == "prod"'
          - 'name == "prodevent"'

service:
  pipelines:
    traces:
      receivers: [foo]
      exporters: [count]
    metrics:
      receivers: [count]
      exporters: [bar]
```

> 자세한 커넥터 설정은
> [커넥터 README](https://github.com/open-telemetry/opentelemetry-collector/blob/main/connector/README.md)를
> 참고한다.

## 익스텐션 <img width="32" class="img-initial otel-icon" alt="" src="/img/logos/32x32/Extensions.svg"> {#extensions}

익스텐션은 텔레메트리 데이터 처리와 직접적인 관련이 없는 작업을 수행하기 위해
컬렉터의 기능을 확장하는 선택적 구성 요소이다. 예를 들어, 컬렉터 상태 모니터링,
서비스 디스커버리(service discovery), 데이터 전달 등을 위한 익스텐션을 추가할 수
있다.

컬렉터 설정 파일의 `extensions` 섹션을 통해 익스텐션을 설정할 수 있다. 대부분의
익스텐션은 기본 설정을 제공하므로, 익스텐션 이름만 지정해도 설정할 수 있다.
지정한 설정은 존재하는 경우 기본값을 재정의한다.

> 익스텐션을 설정한다고 해서 자동으로 활성화되는 것은 아니다. 익스텐션은
> [service](#service) 섹션 내에서 활성화된다.

기본적으로는 어떤 익스텐션도 설정되어 있지 않다. 다음 예시는 동일한 파일에
설정된 여러 익스텐션을 보여준다.

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679
```

> 자세한 익스텐션 설정은
> [익스텐션 README](https://github.com/open-telemetry/opentelemetry-collector/blob/main/extension/README.md)를
> 참고한다.

## Service 섹션 {#service}

`service` 섹션은 리시버, 프로세서, 익스포터, 익스텐션 섹션에서 찾은 설정을
바탕으로 컬렉터에서 어떤 구성 요소가 활성화되는지를 설정하는 데 사용된다. 구성
요소가 설정되어 있더라도 `service` 섹션 내에 정의되어 있지 않으면 활성화되지
않는다.

service 섹션은 세 개의 하위 섹션으로 구성된다.

- 익스텐션
- 파이프라인
- 텔레메트리

### 익스텐션 {#service-extensions}

`extensions` 하위 섹션은 활성화할 익스텐션 목록으로 구성된다. 예를 들면 다음과
같다.

```yaml
service:
  extensions: [health_check, pprof, zpages]
```

### 파이프라인 {#pipelines}

`pipelines` 하위 섹션은 파이프라인을 설정하는 곳으로, 다음과 같은 타입을 가질 수
있다.

- `traces`는 트레이스 데이터를 수집(collect)하고 처리(process)한다.
- `metrics`는 메트릭 데이터를 수집하고 처리한다.
- `logs`는 로그 데이터를 수집하고 처리한다.

파이프라인은 일련의 리시버, 프로세서, 익스포터로 구성된다. 파이프라인에 리시버,
프로세서, 또는 익스포터를 포함시키기 전에, 해당 설정을 적절한 섹션에 정의했는지
확인한다.

동일한 리시버, 프로세서, 또는 익스포터를 둘 이상의 파이프라인에서 사용할 수
있다. 프로세서가 여러 파이프라인에서 참조되는 경우, 각 파이프라인은 해당
프로세서의 별도 인스턴스를 갖는다.

다음은 파이프라인 설정의 예시이다. 프로세서의 순서가 데이터 처리 순서를
결정한다는 점에 유의한다.

```yaml
service:
  pipelines:
    metrics:
      receivers: [opencensus, prometheus]
      exporters: [opencensus, prometheus]
    traces:
      receivers: [opencensus, jaeger]
      processors: [memory_limiter]
      exporters: [opencensus, zipkin]
```

구성 요소와 마찬가지로, `type[/name]` 문법을 사용하여 특정 타입에 대한 추가
파이프라인을 생성할 수 있다. 다음은 이전 설정을 확장한 예시이다.

```yaml
service:
  pipelines:
    # ...
    traces:
      # ...
    traces/2:
      receivers: [opencensus]
      exporters: [zipkin]
```

### 텔레메트리 {#telemetry}

`telemetry` 설정 섹션은 컬렉터 자체에 대한 옵저버빌리티를 설정하는 곳이다.
`logs`와 `metrics`라는 두 개의 하위 섹션으로 구성된다. 이러한 시그널을 설정하는
방법을 알아보려면
[컬렉터에서 내부 텔레메트리 활성화하기](/docs/collector/internal-telemetry#activate-internal-telemetry-in-the-collector)를
참고한다.

## 기타 정보 {#other-information}

### 환경 변수 {#environment-variables}

컬렉터 설정에서는 환경 변수의 사용과 확장(expansion)을 지원한다. 예를 들어
`DB_KEY`와 `OPERATION` 환경 변수에 저장된 값을 사용하려면 다음과 같이 작성할 수
있다.

```yaml
processors:
  attributes/example:
    actions:
      - key: ${env:DB_KEY}
        action: ${env:OPERATION}
```

다음과 같은 bash 문법을 사용해 환경 변수에 기본값을 전달할 수 있다:
`${env:DB_KEY:-some-default-var}`

```yaml
processors:
  attributes/example:
    actions:
      - key: ${env:DB_KEY:-mydefault}
        action: ${env:OPERATION:-}
```

리터럴 `$`를 나타내려면 `$$`를 사용한다. 예를 들어 `$DataVisualization`을
표현하면 다음과 같다.

```yaml
exporters:
  prometheus:
    endpoint: prometheus:8889
    namespace: $$DataVisualization
```

### 프록시 지원 {#proxy-support}

[`net/http`](https://pkg.go.dev/net/http) 패키지를 사용하는 익스포터는 다음
프록시 환경 변수를 따른다.

- `HTTP_PROXY`: HTTP 프록시의 주소
- `HTTPS_PROXY`: HTTPS 프록시의 주소
- `NO_PROXY`: 프록시를 사용하지 않아야 하는 주소

컬렉터 시작 시점에 설정되어 있으면, 프로토콜에 관계없이 익스포터는 이러한 환경
변수에 정의된 대로 트래픽을 프록시하거나 프록시를 우회한다.

### 인증 {#authentication}

HTTP 또는 gRPC 포트를 노출하는 대부분의 리시버는 컬렉터의 인증 메커니즘을
사용하여 보호할 수 있다. 마찬가지로, HTTP 또는 gRPC 클라이언트를 사용하는
대부분의 익스포터는 나가는 요청에 인증을 추가할 수 있다.

컬렉터의 인증 메커니즘은 익스텐션 메커니즘을 사용하며, 이를 통해 커스텀
인증자(authenticator)를 컬렉터 배포판에 연결할 수 있다. 각 인증 익스텐션은 두
가지 방식으로 사용될 수 있다.

- 익스포터를 위한 클라이언트 인증자로서, 나가는 요청에 인증 데이터를 추가한다.
- 리시버를 위한 서버 인증자로서, 들어오는 연결을 인증한다.

알려진 인증자 목록은
[레지스트리](/ecosystem/registry/?s=authenticator&component=extension)를
참고한다. 커스텀 인증자를 개발하는 데 관심이 있다면
[인증자 익스텐션 만들기](/docs/collector/extend/custom-component/extension/authenticator)를
참고한다.

컬렉터의 리시버에 서버 인증자를 추가하려면 다음 단계를 따른다.

1. `.extensions` 아래에 인증자 익스텐션과 그 설정을 추가한다.
2. 컬렉터에 의해 로드되도록, `.services.extensions`에 해당 인증자에 대한 참조를
   추가한다.
3. `.receivers.<your-receiver>.<http-or-grpc-config>.auth` 아래에 해당 인증자에
   대한 참조를 추가한다.

다음 예시는 리시버 측에서 OIDC 인증자를 사용하며, 에이전트 역할을 하는
오픈텔레메트리 컬렉터로부터 데이터를 수신하는 원격 컬렉터에 적합하다.

```yaml
extensions:
  oidc:
    issuer_url: http://localhost:8080/auth/realms/opentelemetry
    audience: collector

receivers:
  otlp/auth:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        auth:
          authenticator: oidc

processors:

exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug:

service:
  extensions:
    - oidc
  pipelines:
    traces:
      receivers:
        - otlp/auth
      processors: []
      exporters:
        - debug
```

에이전트 측에서는, OTLP 익스포터가 OIDC 토큰을 획득하여 원격 컬렉터로 보내는
모든 RPC에 해당 토큰을 추가하도록 하는 예시이다.

```yaml
extensions:
  oauth2client:
    client_id: agent
    client_secret: some-secret
    token_url: http://localhost:8080/auth/realms/opentelemetry/protocol/openid-connect/token

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:

exporters:
  otlp_grpc/auth:
    endpoint: remote-collector:4317
    auth:
      authenticator: oauth2client

service:
  extensions:
    - oauth2client
  pipelines:
    traces:
      receivers:
        - otlp
      processors: []
      exporters:
        - otlp_grpc/auth
```

### 인증서 설정하기 {#setting-up-certificates}

프로덕션 환경에서는 보안 통신을 위해 TLS 인증서를 사용하거나 상호 인증을 위해
mTLS를 사용한다. 이 예시와 같이 자체 서명 인증서(self-signed certificate)를
생성하려면 다음 단계를 따른다. 프로덕션 용도로 인증서를 확보할 때는 현재 사용
중인 인증서 발급 절차를 사용하고 싶을 수도 있다.

[`cfssl`](https://github.com/cloudflare/cfssl)을 설치하고 다음과 같이 `csr.json`
파일을 생성한다.

```json
{
  "hosts": ["localhost", "127.0.0.1"],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "OpenTelemetry Example"
    }
  ]
}
```

그런 다음 다음 명령어를 실행한다.

```sh
cfssl genkey -initca csr.json | cfssljson -bare ca
cfssl gencert -ca ca.pem -ca-key ca-key.pem csr.json | cfssljson -bare cert
```

이렇게 하면 두 개의 인증서가 생성된다.

- `ca.pem`에 있는 "OpenTelemetry Example" 인증 기관(Certificate Authority,
  CA)과, `ca-key.pem`에 있는 관련 키
- OpenTelemetry Example CA가 서명한 `cert.pem`의 클라이언트 인증서와,
  `cert-key.pem`에 있는 관련 키.

#### 컬렉터에서 인증서 사용하기 {#using-certificates-in-the-collector}

인증서를 준비했다면, 이를 사용하도록 컬렉터를 설정한다.

##### 리시버용 TLS 설정(서버 측) {#tls-configuration-for-receivers-server-side}

들어오는 연결을 암호화하려면 리시버에 TLS를 설정한다. 서버 인증서를 지정하려면
`cert_file`과 `key_file`을 사용한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: /path/to/cert.pem
          key_file: /path/to/cert-key.pem
      http:
        endpoint: 0.0.0.0:4318
        tls:
          cert_file: /path/to/cert.pem
          key_file: /path/to/cert-key.pem
```

##### 익스포터용 TLS 설정(클라이언트 측) {#tls-configuration-for-exporters-client-side}

나가는 연결을 암호화하려면 익스포터에 TLS를 설정한다. 서버의 인증서를 검증하려면
`ca_file`을 사용한다.

```yaml
exporters:
  otlp_grpc:
    endpoint: otelcol2:4317
    tls:
      ca_file: /path/to/ca.pem
```

서버에 클라이언트 인증서도 제시해야 하는 경우:

```yaml
exporters:
  otlp_grpc:
    endpoint: otelcol2:4317
    tls:
      ca_file: /path/to/ca.pem
      cert_file: /path/to/cert.pem
      key_file: /path/to/cert-key.pem
```

##### mTLS 설정(상호 TLS) {#mtls-configuration-mutual-tls}

mTLS에서는 리시버와 익스포터가 서로의 인증서를 검증한다. 리시버에서는 클라이언트
인증서를 검증하기 위해 `client_ca_file`을 추가한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: /path/to/server-cert.pem
          key_file: /path/to/server-key.pem
          client_ca_file: /path/to/ca.pem
```

익스포터에서는 서버를 검증할 CA와 클라이언트 인증서를 모두 제공한다.

```yaml
exporters:
  otlp_grpc:
    endpoint: remote-collector:4317
    tls:
      ca_file: /path/to/ca.pem
      cert_file: /path/to/client-cert.pem
      key_file: /path/to/client-key.pem
```

##### 공통 TLS 설정 {#common-tls-settings}

다음 설정을 TLS 설정에 사용할 수 있다.

| 설정                   | 설명                                               |
| ---------------------- | -------------------------------------------------- |
| `ca_file`              | 피어 인증서를 검증하기 위한 CA 인증서 경로         |
| `cert_file`            | TLS 인증서 경로                                    |
| `key_file`             | TLS 개인 키 경로                                   |
| `client_ca_file`       | 클라이언트 인증서를 검증하기 위한 CA 인증서 경로   |
| `insecure`             | TLS 검증 비활성화(프로덕션 환경에는 권장하지 않음) |
| `insecure_skip_verify` | 서버 인증서 검증 건너뛰기(권장하지 않음)           |
| `min_version`          | 최소 TLS 버전(예: `1.2` 또는 `1.3`)                |
| `max_version`          | 최대 TLS 버전                                      |
| `reload_interval`      | 인증서가 재로드되는 주기(duration)                 |

<!-- prettier-ignore-start -->
<!-- markdownlint-disable MD034 -->
> TLS 설정 옵션에 대한 자세한 내용은
> [configtls 문서](https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/config/configtls/README.md)를
> 참고한다.
<!-- markdownlint-enable MD034 -->
<!-- prettier-ignore-end -->

[dcc]: /docs/concepts/components/#collector

## 설정 재정의 {#override-settings}

`--set` 옵션을 사용하여 컬렉터 설정을 재정의할 수 있다. 이 방법으로 정의한
설정은 모든 `--config` 소스가 해석되고 병합된 후 최종 설정에 병합된다.

다음 예시는 중첩된 섹션 내에서 설정을 재정의하는 방법을 보여준다.

### 단순 속성 {#simple-property}

`--set` 옵션은 항상 하나의 키/값 쌍을 받으며, `--set key=value`와 같이 사용한다.
이에 대응하는 YAML은 다음과 같다.

```yaml
key: value
```

### 복잡한 중첩 키 {#complex-nested-keys}

중첩된 맵 값을 참조하려면 쌍의 이름에서 키 구분자로 이중 콜론(`::`)을 사용한다.
예를 들어 `--set outer::inner=value`는 다음과 같이 변환된다.

```yaml
outer:
  inner: value
```

### 여러 값 {#multiple-values}

여러 값을 설정하려면 `--set` 플래그를 여러 번 지정한다. 따라서
`--set a=b --set c=d`는 다음과 같이 된다.

```yaml
a: b
c: d
```

### 배열 값 {#array-values}

배열은 값을 `[]`로 감싸서 표현할 수 있다. 예를 들어 `--set "key=[a, b, c]"`는
다음과 같이 변환된다.

```yaml
key:
  - a
  - b
  - c
```

더 복잡한 데이터 구조를 표현해야 한다면, YAML을 사용하는 것을 강력히 권장한다.

> [!CAUTION]
>
> `--set` 옵션에는 다음과 같은 제약이 있다.
>
> 1. 점(`.`)을 포함하는 키는 설정할 수 없다.
> 2. 등호(`=`)를 포함하는 키는 설정할 수 없다.
> 3. 속성의 값 부분 내에서 설정 키 구분자는 "::"이다. 예를 들어
>    `--set "name={a::b: c}"`는 `--set name::a::b=c`와 동일하다.

## 다른 설정 제공자 포함하기 {#embedding-other-configuration-providers}

하나의 설정 제공자는 다음과 같이 다른 설정 제공자를 참조할 수 있다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:

exporters: ${file:otlp-exporter.yaml}

service:
  extensions: []
  pipelines:
    traces:
      receivers: [otlp]
      processors: []
      exporters: [otlp_grpc]
```

## 배포판에서 사용 가능한 구성 요소 확인하는 방법 {#how-to-check-components-available-in-a-distribution}

하위 명령어인 build-info를 사용한다. 아래는 예시이다.

```bash
otelcol components
```

출력 예시:

```yaml
buildinfo:
  command: otelcol
  description: OpenTelemetry Collector
  version: 0.143.0
receivers:
  - otlp
processors:
  - memory_limiter
exporters:
  - otlp_grpc
  - otlp_http
  - debug
extensions:
  - zpages
```

## 최종 설정을 확인하는 방법 {#how-to-examine-the-final-configuration}

> [!CAUTION]
>
> 이 명령어는 실험적인 기능이다. 사전 경고 없이 동작이 변경될 수 있다.

기본 모드(`--mode=redacted`)와 `--feature-gates=otelcol.printInitialConfig`로
`print-config`를 사용한다.

```bash
otelcol print-config --config=file:examples/local/otel-config.yaml
```

기본적으로 설정은 유효한 경우에만 출력되며, 민감한 정보는 마스킹(redact)된다는
점에 유의한다. 유효하지 않을 수도 있는 설정을 출력하려면 `--validate=false`를
사용한다.

### 민감한 필드를 확인하는 방법 {#how-to-view-sensitive-fields}

`--mode=unredacted`와 `--feature-gates=otelcol.printInitialConfig`로
`print-config`를 사용한다.

```bash
otelcol print-config --mode=unredacted --config=file:examples/local/otel-config.yaml
```

### 최종 설정을 JSON 형식으로 출력하는 방법 {#how-to-print-the-final-configuration-in-json-format}

> [!CAUTION]
>
> 이 명령어는 실험적인 기능이다. 사전 경고 없이 동작이 변경될 수 있다.

`--format=json`과 `--feature-gates=otelcol.printInitialConfig`로
`print-config`를 사용한다. JSON 형식은 불안정한(unstable) 것으로 간주된다는 점에
유의한다.

```bash
otelcol print-config --format=json --config=file:examples/local/otel-config.yaml
```
