---
title: 문제 해결
description: 컬렉터 문제를 해결하기 위한 권장 사항
weight: 25
cSpell:ignore: confmap pprof tracez zpages
default_lang_commit: e75af5ff9e415f62cc1136f3170708c2567b3370
---

이 페이지에서는 오픈텔레메트리(OpenTelemetry) 컬렉터의 상태(health)와 성능
문제를 해결하는 방법을 알아본다.

## 문제 해결 도구 {#troubleshooting-tools}

컬렉터는 문제를 디버깅하기 위한 다양한 메트릭, 로그, 익스텐션을 제공한다.

### 내부 텔레메트리 {#internal-telemetry}

컬렉터 자체의 [내부 텔레메트리](/docs/collector/internal-telemetry/)를 구성하고
사용하여 성능을 모니터링할 수 있다.

### 로컬 익스포터 {#local-exporters}

구성 검증이나 네트워크 디버깅과 같은 특정 유형의 문제의 경우, 로컬 로그로
출력하도록 구성된 컬렉터에 소량의 테스트 데이터를 전송할 수 있다.
[로컬 익스포터](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter#general-information)를
사용하면 컬렉터가 처리 중인 데이터를 확인할 수 있다.

실시간 문제 해결에는
[`debug` 익스포터](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/debugexporter/README.md)를
사용하는 것을 고려한다. 이 익스포터는 컬렉터가 데이터를 수신, 처리, 내보내고
있는지 확인할 수 있다. 예를 들면 다음과 같다.

```yaml
receivers:
  zipkin:
exporters:
  debug:
service:
  pipelines:
    traces:
      receivers: [zipkin]
      processors: []
      exporters: [debug]
```

테스트를 시작하려면 Zipkin 페이로드(payload)를 생성한다. 예를 들어 다음 내용을
담은 `trace.json` 파일을 생성할 수 있다.

```json
[
  {
    "traceId": "5982fe77008310cc80f1da5e10147519",
    "parentId": "90394f6bcffb5d13",
    "id": "67fae42571535f60",
    "kind": "SERVER",
    "name": "/m/n/2.6.1",
    "timestamp": 1516781775726000,
    "duration": 26000,
    "localEndpoint": {
      "serviceName": "api"
    },
    "remoteEndpoint": {
      "serviceName": "apip"
    },
    "tags": {
      "data.http_response_code": "201"
    }
  }
]
```

컬렉터가 실행 중인 상태에서 이 페이로드를 컬렉터로 전송한다.

```shell
curl -X POST localhost:9411/api/v2/spans -H'Content-Type: application/json' -d @trace.json
```

다음과 같은 로그 항목이 표시되어야 한다.

```shell
2023-09-07T09:57:43.468-0700    info    TracesExporter  {"kind": "exporter", "data_type": "traces", "name": "debug", "resource spans": 1, "spans": 2}
```

전체 페이로드가 출력되도록 `debug` 익스포터를 구성할 수도 있다.

```yaml
exporters:
  debug:
    verbosity: detailed
```

수정된 구성으로 이전 테스트를 다시 실행하면, 로그 출력은 다음과 같은 모습이다.

```shell
2023-09-07T09:57:12.820-0700    info    TracesExporter  {"kind": "exporter", "data_type": "traces", "name": "debug", "resource spans": 1, "spans": 2}
2023-09-07T09:57:12.821-0700    info    ResourceSpans #0
Resource SchemaURL: https://opentelemetry.io/schemas/1.4.0
Resource attributes:
     -> service.name: Str(telemetrygen)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope telemetrygen
Span #0
    Trace ID       : 0c636f29e29816ea76e6a5b8cd6601cf
    Parent ID      : 1a08eba9395c5243
    ID             : 10cebe4b63d47cae
    Name           : okey-dokey
    Kind           : Internal
    Start time     : 2023-09-07 16:57:12.045933 +0000 UTC
    End time       : 2023-09-07 16:57:12.046058 +0000 UTC
    Status code    : Unset
    Status message :
Attributes:
     -> span.kind: Str(server)
     -> net.peer.ip: Str(1.2.3.4)
     -> peer.service: Str(telemetrygen)
```

### 컬렉터 구성 요소 확인하기 {#check-collector-components}

다음 하위 명령어를 사용하면 안정성 수준을 포함해 컬렉터 배포판에서 사용 가능한
구성 요소 목록을 확인할 수 있다. 출력 형식은 버전에 따라 달라질 수 있다는 점에
유의한다.

```shell
otelcol components
```

샘플 출력:

```yaml
buildinfo:
  command: otelcol
  description: OpenTelemetry Collector
  version: 0.96.0
receivers:
  - name: opencensus
    stability:
      logs: Undefined
      metrics: Beta
      traces: Beta
  - name: prometheus
    stability:
      logs: Undefined
      metrics: Beta
      traces: Undefined
  - name: zipkin
    stability:
      logs: Undefined
      metrics: Undefined
      traces: Beta
  - name: otlp
    stability:
      logs: Beta
      metrics: Stable
      traces: Stable
processors:
  - name: resource
    stability:
      logs: Beta
      metrics: Beta
      traces: Beta
  - name: span
    stability:
      logs: Undefined
      metrics: Undefined
      traces: Alpha
  - name: probabilistic_sampler
    stability:
      logs: Alpha
      metrics: Undefined
      traces: Beta
exporters:
  - name: otlp
    stability:
      logs: Beta
      metrics: Stable
      traces: Stable
  - name: otlphttp
    stability:
      logs: Beta
      metrics: Stable
      traces: Stable
  - name: debug
    stability:
      logs: Development
      metrics: Development
      traces: Development
  - name: prometheus
    stability:
      logs: Undefined
      metrics: Beta
      traces: Undefined
connectors:
  - name: forward
    stability:
      logs-to-logs: Beta
      logs-to-metrics: Undefined
      logs-to-traces: Undefined
      metrics-to-logs: Undefined
      metrics-to-metrics: Beta
      traces-to-traces: Beta
extensions:
  - name: zpages
    stability:
      extension: Beta
  - name: health_check
    stability:
      extension: Beta
  - name: pprof
    stability:
      extension: Beta
```

### 익스텐션 {#extensions}

컬렉터를 디버깅하기 위해 활성화할 수 있는 익스텐션 목록은 다음과 같다.

#### 성능 프로파일러(Performance Profiler, pprof) {#performance-profiler-pprof}

로컬에서 `1777` 포트로 사용할 수 있는
[pprof 익스텐션](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/extension/pprofextension/README.md)을
사용하면 컬렉터가 실행되는 동안 프로파일링할 수 있다. 이는 고급 사용 사례이며
대부분의 상황에서는 필요하지 않다.

#### zPages {#zpages}

로컬에서 `55679` 포트로 노출되는
[zPages 익스텐션](https://github.com/open-telemetry/opentelemetry-collector/tree/main/extension/zpagesextension/README.md)은
컬렉터의 리시버와 익스포터에서 나오는 실시간 데이터를 확인하는 데 사용할 수
있다.

`/debug/tracez`에 노출되는 TraceZ 페이지는 다음과 같은 트레이스 작업을
디버깅하는 데 유용하다.

- 지연 시간(latency) 문제. 애플리케이션에서 느린 부분을 찾는다.
- 데드락(deadlock)과 계측 문제. 끝나지 않는 실행 중인 스팬을 식별한다.
- 오류. 어떤 유형의 오류가 어디서 발생하는지 파악한다.

`zpages`에는 컬렉터 자체가 내보내지 않는 오류 로그가 포함될 수 있다는 점에
유의한다.

컨테이너화된 환경에서는 이 포트를 로컬로만 노출하는 대신 공개 인터페이스에
노출하고 싶을 수 있다. `endpoint`는 `extensions` 구성 섹션을 사용해 설정할 수
있다.

```yaml
extensions:
  zpages:
    endpoint: 0.0.0.0:55679
```

## 복잡한 파이프라인 디버깅을 위한 체크리스트 {#checklist-for-debugging-complex-pipelines}

텔레메트리가 여러 컬렉터와 네트워크를 거쳐 흐를 때는 문제를 격리하기 어려울 수
있다. 파이프라인의 컬렉터 또는 다른 구성 요소를 거치는 텔레메트리의 각
"홉(hop)"에 대해 다음을 확인하는 것이 중요하다.

- 컬렉터의 로그에 오류 메시지가 있는가?
- 이 구성 요소로 텔레메트리가 어떻게 유입되는가?
- 이 구성 요소가 텔레메트리를 어떻게 수정하는가(예: 샘플링 또는 편집)?
- 이 구성 요소에서 텔레메트리가 어떻게 내보내지는가?
- 텔레메트리는 어떤 형식으로 되어 있는가?
- 다음 홉은 어떻게 구성되어 있는가?
- 데이터의 유입 또는 유출을 막는 네트워크 정책이 있는가?

## Kubernetes 환경에서 문제 해결하기 {#troubleshooting-in-kubernetes-environments}

Kubernetes에서 오픈텔레메트리 컬렉터를 실행할 때는
[임시 디버그 컨테이너(ephemeral debug container)](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)를
사용해 컬렉터 관련 문제를 조사할 수 있다.

## 일반적인 컬렉터 문제 {#common-collector-issues}

이 섹션에서는 일반적인 컬렉터 문제를 해결하는 방법을 다룬다.

### 컬렉터에 데이터 문제가 발생하는 경우 {#collector-is-experiencing-data-issues}

컬렉터와 그 구성 요소에 데이터 문제가 발생할 수 있다.

#### 컬렉터가 데이터를 드롭하는 경우 {#collector-is-dropping-data}

컬렉터는 다양한 이유로 데이터를 드롭(drop)할 수 있으며, 가장 흔한 이유는 다음과
같다.

- 컬렉터의 크기가 적절하지 않아 데이터를 수신하는 속도만큼 빠르게 처리하고
  내보내지 못하는 경우.
- 익스포터 대상이 사용 불가능하거나 데이터를 너무 느리게 수신하는 경우.

드롭을 완화하려면, 활성화된 익스포터에서
[큐 재시도 옵션](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/exporterhelper#configuration)을,
특히
[전송 큐 배치 설정](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/exporterhelper#sending-queue-batch-settings)을
구성한다.

#### 컬렉터가 데이터를 수신하지 못하는 경우 {#collector-is-not-receiving-data}

컬렉터가 데이터를 수신하지 못하는 데는 다음과 같은 이유가 있을 수 있다.

- 네트워크 구성 문제.
- 잘못된 리시버 구성.
- 잘못된 클라이언트 구성.
- 리시버가 `receivers` 섹션에 정의되어 있지만 어떤 `pipelines`에도 활성화되지
  않은 경우.

잠재적인 문제를 확인하려면 컬렉터의
[로그](/docs/collector/internal-telemetry/#configure-internal-logs)와
[zPages](https://github.com/open-telemetry/opentelemetry-collector/blob/main/extension/zpagesextension/README.md)를
확인한다.

#### 컬렉터가 데이터를 처리하지 못하는 경우 {#collector-is-not-processing-data}

대부분의 처리 문제는 프로세서의 작동 방식에 대한 오해나 프로세서의 잘못된
구성에서 비롯된다. 예를 들면 다음과 같다.

- attributes 프로세서는 스팬의 "태그"에 대해서만 작동한다. 스팬 이름은 span
  프로세서가 처리한다.
- 트레이스 데이터용 프로세서는(tail sampling 제외) 개별 스팬에 대해서만
  작동한다.

#### 컬렉터가 데이터를 내보내지 못하는 경우 {#collector-is-not-exporting-data}

컬렉터가 데이터를 내보내지 못하는 데는 다음과 같은 이유가 있을 수 있다.

- 네트워크 구성 문제.
- 잘못된 익스포터 구성.
- 대상을 사용할 수 없는 경우.

잠재적인 문제를 확인하려면 컬렉터의
[로그](/docs/collector/internal-telemetry/#configure-internal-logs)와
[zPages](https://github.com/open-telemetry/opentelemetry-collector/blob/main/extension/zpagesextension/README.md)를
확인한다.

데이터 내보내기가 작동하지 않는 경우는 흔히 방화벽, DNS, 프록시 문제와 같은
네트워크 구성 문제 때문이다. 컬렉터는
[프록시 지원](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter#proxy-support)을
제공한다는 점에 유의한다.

### 컬렉터에 제어 문제가 발생하는 경우 {#collector-is-experiencing-control-issues}

컬렉터는 시작 실패나 예기치 않은 종료 또는 재시작을 겪을 수 있다.

#### 컬렉터가 종료되거나 재시작되는 경우 {#collector-exits-or-restarts}

컬렉터는 다음과 같은 이유로 종료되거나 재시작될 수 있다.

- `memory_limiter`
  [프로세서](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/memorylimiterprocessor/README.md)가
  누락되었거나 잘못 구성되어 발생하는 메모리 압박(memory pressure).
- 부하에 맞지 않는 부적절한 크기 설정.
- 잘못된 구성. 예를 들어 사용 가능한 메모리보다 크게 설정된 큐.
- 인프라 리소스 제한. 예를 들어 Kubernetes.

#### 컬렉터가 Windows Docker 컨테이너에서 시작에 실패하는 경우 {#collector-fails-to-start-in-windows-docker-containers}

v0.90.1 이하 버전에서는 컬렉터가 Windows Docker 컨테이너에서 시작에 실패하며
`The service process could not connect to the service controller` 오류 메시지가
발생할 수 있다. 이 경우, Windows 서비스로 실행을 시도하지 않고 대화형 터미널에서
실행되는 것처럼 컬렉터를 강제로 시작하려면 `NO_WINDOWS_SERVICE=1` 환경 변수를
설정해야 한다.

### 컬렉터에 구성 문제가 발생하는 경우 {#collector-is-experiencing-configuration-issues}

컬렉터는 구성 문제로 인해 문제를 겪을 수 있다.

#### Null 맵 {#null-maps}

여러 구성을 해석(resolution)하는 과정에서, 이후 구성의 값이 null이더라도 이전
구성의 값은 제거되고 이후 구성의 값이 우선한다. 이 문제는 다음과 같이 해결할 수
있다.

- `processors:` 대신 `processors: {}`와 같이 빈 맵을 나타내는 `{}`를 사용한다.
- `processors:`와 같은 빈 구성을 설정에서 생략한다.

자세한 내용은
[confmap 문제 해결](https://github.com/open-telemetry/opentelemetry-collector/blob/main/confmap/README.md#null-maps)을
참고한다.
