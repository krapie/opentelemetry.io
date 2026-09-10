---
title: OBI Prometheus 및 오픈텔레메트리 데이터 내보내기 구성
linkTitle: 데이터 내보내기
description:
  Prometheus 및 오픈텔레메트리(OpenTelemetry) 메트릭과 오픈텔레메트리 트레이스를
  내보내도록 OBI 구성 요소를 구성한다
weight: 10
# prettier-ignore
cSpell:ignore: Aerospike AsterixDB Chroma couchbase genai gonic jackc libcudart memcached Milvus nats Ollama pgxpool Pinecone pyserver Qdrant Qwen rerank segmentio spanmetrics sunrpc Weaviate Zilliz
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

OBI는 오픈텔레메트리(OpenTelemetry) 메트릭과 트레이스를 OTLP 엔드포인트로 내보낼
수 있다.

## 계측 호환성 {#instrumentation-compatibility}

OBI는 트레이스 및 메트릭 계측을 위해 다음 프로토콜과 기능 버전을 지원한다.

| 영역          | 지원 버전                                                                                                                           | 참고                                                                                                  |
| :------------ | :---------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| HTTP          | `1.0/1.1`                                                                                                                           | 컨텍스트 전파가 지원된다.                                                                             |
| HTTP          | `2.0`                                                                                                                               | 컨텍스트 전파에는 Go 라이브러리 수준 계측이 필요하다.                                                 |
| gRPC          | `1.0+`                                                                                                                              | 컨텍스트 전파가 지원된다. OBI 이전에 시작된, 오래 유지되는 연결은 메서드 이름에 `*`를 사용할 수 있다. |
| MySQL         | All                                                                                                                                 | OBI가 시작되기 전에 생성된 준비된 문(prepared statement)에는 쿼리 텍스트가 포함되지 않을 수 있다.     |
| PostgreSQL    | All                                                                                                                                 | OBI가 시작되기 전에 생성된 준비된 문에는 쿼리 텍스트가 포함되지 않을 수 있다.                         |
| MSSQL         | All                                                                                                                                 | OBI가 시작되기 전에 생성된 준비된 문에는 쿼리 텍스트가 포함되지 않을 수 있다.                         |
| Redis         | All                                                                                                                                 | 기존 연결에는 데이터베이스 번호나 `db.namespace`가 포함되지 않을 수 있다.                             |
| MongoDB       | `5.0+`                                                                                                                              | 압축된 페이로드는 지원되지 않는다.                                                                    |
| Couchbase     | All                                                                                                                                 | OBI가 시작되기 전에 협상이 완료된 경우 버킷 또는 컬렉션 이름을 사용할 수 없을 수 있다.                |
| Aerospike     | All                                                                                                                                 | 압축된 페이로드는 파싱되지 않으며, 레코드 및 bin 값은 캡처되지 않는다.                                |
| Memcached     | All                                                                                                                                 | `quit` 및 meta 명령을 제외한 ASCII 텍스트 프로토콜 하위 집합을 지원한다.                              |
| Kafka         | All                                                                                                                                 | fetch API 버전 `13+`에서는 토픽 이름 조회가 실패할 수 있다.                                           |
| MQTT          | `3.1.1/5.0`                                                                                                                         | 페이로드는 캡처되지 않는다.                                                                           |
| NATS          | All                                                                                                                                 | 문서화된 추가 버전 제한이 없다.                                                                       |
| AMQP          | `1.0`                                                                                                                               | transfer performative만 스팬을 생성한다.                                                              |
| SunRPC        | All                                                                                                                                 | TCP를 통한 ONC RPC를 지원하며, UDP는 지원되지 않는다. RPCSEC_GSS는 프로시저 인수를 숨긴다.            |
| GraphQL       | All                                                                                                                                 | 문서화된 추가 버전 제한이 없다.                                                                       |
| Elasticsearch | `7.14+`                                                                                                                             | 문서화된 추가 버전 제한이 없다.                                                                       |
| OpenSearch    | `3.0.0+`                                                                                                                            | 문서화된 추가 버전 제한이 없다.                                                                       |
| AWS S3        | All                                                                                                                                 | 문서화된 추가 버전 제한이 없다.                                                                       |
| AWS SQS       | All                                                                                                                                 | 문서화된 추가 버전 제한이 없다.                                                                       |
| SQL++         | All                                                                                                                                 | 문서화된 추가 버전 제한이 없다.                                                                       |
| GenAI         | OpenAI, Anthropic, Gemini, AWS Bedrock, Qwen, Ollama, OpenAI-compatible gateways, MCP, embedding, rerank, and vector retrieval APIs | 제공자별 페이로드 추출에는 일치하는 `ebpf.payload_extraction.http` 플래그가 필요하다.                 |

일부 애플리케이션 수준 계측은 특정 런타임, 라이브러리 또는 서버 버전에도
의존한다.

| 영역              | 지원 버전                             | 참고                                                                                            |
| :---------------- | :------------------------------------ | :---------------------------------------------------------------------------------------------- |
| Go 애플리케이션   | Go `1.17+`                            | Go 라이브러리 수준 계측에 적용된다. Go 라이브러리 수준 컨텍스트 전파에는 Go `1.18+`가 필요하다. |
| Java 애플리케이션 | JDK `8+`                              | 문서화된 추가 런타임 제약이 없다.                                                               |
| NGINX             | NGINX `1.27.5` 및 `1.29.7`에서 검증됨 | 현재 문서화된 검증이 다루는 NGINX 버전이다.                                                     |

### Go 라이브러리 계측 호환성 {#go-library-instrumentation-compatibility}

OBI는 애플리케이션 계측을 위해 다음 Go 라이브러리와 최소 버전을 지원한다.

| 라이브러리                       | 지원 버전                       |
| :------------------------------- | :------------------------------ |
| `net/http`                       | `>= 1.17`                       |
| `golang.org/x/net/http2`         | `>= 0.12.0`                     |
| `github.com/gorilla/mux`         | `>= v1.5.0`                     |
| `github.com/gin-gonic/gin`       | `>= v1.6.0`, `!= v1.7.5`        |
| `google.golang.org/grpc`         | `>= 1.40`                       |
| `net/rpc/jsonrpc`                | `>= 1.17`                       |
| `database/sql`                   | `>= 1.17`                       |
| `github.com/go-sql-driver/mysql` | `>= v1.5.0`                     |
| `github.com/lib/pq`              | all versions                    |
| `github.com/redis/go-redis/v9`   | `>= v9.0.0`                     |
| `github.com/segmentio/kafka-go`  | `>= v0.4.11`                    |
| `github.com/IBM/sarama`          | `>= 1.37`                       |
| `go.mongodb.org/mongo-driver`    | `v1: >= v1.10.1; v2: >= v2.0.1` |

이 표에 나열된 버전은 OBI가 명시적으로 지원하는 버전이다. 다른 버전도 동작할 수
있지만, 별도로 명시되지 않는 한 문서화된 지원 범위에 속하지 않는다.

벡터 검색 페이로드 파싱을 활성화하려면
`ebpf.payload_extraction.http.genai.retrieval.enabled: true` 또는
`OTEL_EBPF_HTTP_RETRIEVAL_ENABLED=true`를 설정한다. 지원되는 API에는 Pinecone,
Qdrant, Milvus 및 Zilliz, Chroma, Weaviate가 포함된다. 페이로드 파싱에는 0이
아닌 `ebpf.buffer_sizes.http` 값도 필요하다. v0.10.0에서 최대 HTTP 캡처 크기는
요청 방향별로 262144바이트이며, 캡처는 기본적으로 비활성화된 상태로 유지된다.

### GPU 계측 호환성 {#gpu-instrumentation-compatibility}

OBI는 환경이 [OBI 호환성 요구 사항](/docs/zero-code/obi/#compatibility)을
충족하고 애플리케이션이 지원되는 CUDA 런타임 라이브러리를 사용할 때 GPU 계측을
지원한다.

| 요구 사항              | 지원                         |
| :--------------------- | :--------------------------- |
| 운영체제               | Linux                        |
| CPU 아키텍처           | `amd64`, `arm64`             |
| CUDA 런타임 라이브러리 | CUDA `7.0+`용 `libcudart.so` |

OBI는 다음 CUDA 작업을 계측한다.

| 작업               |
| :----------------- |
| `cudaLaunchKernel` |
| `cudaGraphLaunch`  |
| `cudaMalloc`       |
| `cudaMemcpy`       |
| `cudaMemcpyAsync`  |

GPU 계측은 위에 나열된 지원되는 CUDA 런타임 라이브러리와 작업을 사용하는
애플리케이션에만 적용된다. 다른 GPU API, 프레임워크 또는 라이브러리는 별도로
명시되지 않는 한 문서화된 지원 범위를 벗어난다.

## 공통 메트릭 구성 {#common-metrics-configuration}

YAML 섹션: `metrics`.

`metrics` 섹션에는 오픈텔레메트리 메트릭 및 트레이스 익스포터의 공통 구성이
포함된다.

현재 내보낼 다양한 메트릭 집합을 선택하는 기능을 지원한다.

예시:

```yaml
metrics:
  features: ['network', 'network_inter_zone']
```

| YAML<br>환경 변수                          | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                               | 유형            | 기본값            |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------- | ----------------- |
| `features`<br>`OTEL_EBPF_METRICS_FEATURES` | OBI가 데이터를 내보내는 메트릭 그룹의 목록이다. [메트릭 내보내기 기능](#metrics-export-features)을 참고한다. 허용 값 `all`, `*`, `application`, `application_span`, `application_span_otel`, `application_span_sizes`, `application_host`, `application_runtime`, `application_service_graph`, `network`, `network_flow_packets`, `network_inter_zone`, `stats`, `stats_tcp_rtt`, `stats_tcp_failed_connections`, `stats_tcp_retransmits`, `stats_tcp_io`, `ebpf`. | list of strings | `["application"]` |

### 메트릭 내보내기 기능 {#metrics-export-features}

OBI 메트릭 익스포터는 [메트릭 디스커버리](./) 구성의 항목과 일치하는 프로세스에
대해 다음 메트릭 데이터 그룹을 내보낼 수 있다.

- `all` 또는 `*`: 모든 메트릭 그룹(모든 메트릭을 활성화하는 편의 옵션)
- `application`: 애플리케이션 수준 메트릭.
- `application_host`: 호스트 기반 과금(host-based pricing)을 위한 애플리케이션
  수준 호스트 메트릭.
- `application_runtime`: 계측된 서비스에서 수집한 Go 런타임, HotSpot JVM 메모리,
  Node.js 이벤트 루프 메트릭. [런타임 메트릭](#runtime-metrics)을 참고한다.
- `application_span`: 레거시 형식의 애플리케이션 수준 트레이스 스팬 메트릭(예:
  `traces_spanmetrics_latency`). `spanmetrics`는 별도로 분리되지 않는다.
- `application_span_otel`: 오픈텔레메트리 형식의 애플리케이션 수준 트레이스 스팬
  메트릭(예: `traces.span.metrics.calls`). `span_metrics`는 별도로 분리된다.
- `application_span_sizes`: 요청 및 응답 크기에 대한 정보를 보고하는
  애플리케이션 수준 트레이스 스팬 메트릭.
- `application_service_graph`: 애플리케이션 수준 서비스 그래프 메트릭. 서비스
  디스커버리에 DNS를 사용하고 DNS 이름이 OBI가 사용하는 오픈텔레메트리 서비스
  이름과 일치하도록 하는 것이 권장된다. 쿠버네티스 환경에서는 서비스 이름
  디스커버리가 설정한 오픈텔레메트리 서비스 이름이 서비스 그래프 메트릭에 가장
  적합한 선택이다.
- `network`: 네트워크 수준 메트릭. 자세한 내용은
  [네트워크 메트릭](../../network) 구성 문서를 참고한다.
- `network_flow_packets`: 네트워크 패킷 카운터 메트릭. 자세한 내용은
  [네트워크 메트릭](../../network/) 구성 문서를 참고한다.
- `network_inter_zone`: 네트워크 존 간(inter-zone) 메트릭. 자세한 내용은
  [네트워크 메트릭](../../network/) 구성 문서를 참고한다.
- `stats`: 모든 TCP 통계 메트릭.
- `stats_tcp_rtt`: TCP 왕복 시간(round-trip time) 히스토그램 메트릭.
- `stats_tcp_failed_connections`: 실패 사유로 레이블이 지정된 TCP 실패 연결
  카운터 메트릭.
- `stats_tcp_retransmits`: TCP 재전송 카운터 메트릭.
- `stats_tcp_io`: I/O 방향으로 레이블이 지정된, 소켓 계층에서 전송된 TCP 바이트.
  이 기능은 모든 TCP 전송 및 수신 호출을 관찰하며 다른 stats 기능보다 오버헤드가
  클 수 있다.
- `ebpf`: 로드된 프로브와 맵에 대한 eBPF 런타임 메트릭으로, Prometheus
  익스포터와 내부 메트릭 리포터를 통해 노출된다.

> [!NOTE]
>
> v0.11.0부터 직접 OTLP 내보내기는 다음 메트릭 이름에 오픈텔레메트리 점
> 표기법(dot notation)을 사용한다.
>
> - `target_info`는 이제 `target.info`이다.
> - `traces_target_info`는 이제 `traces.target.info`이다.
> - `traces_host_info`는 이제 `traces.host.info`이다.
> - `traces_span_metrics_calls_total`은 이제 `traces.span.metrics.calls`이다.
> - `traces_span_metrics_duration`은 이제 `traces.span.metrics.duration`이다.
>
> Prometheus 메트릭 이름은 변경되지 않는다. v0.11.0으로 업그레이드할 때 OTLP
> 쿼리와 대시보드를 업데이트한다.

### 애플리케이션별 메트릭 내보내기 기능 {#per-application-metrics-export-features}

또한 OBI는 각 `discovery > instrument` 항목에 `metrics > features`를 속성으로
추가하여 전역 메트릭 내보내기 기능을 애플리케이션별로 재정의할 수 있게 한다.

예를 들어 다음 구성에서:

- `apache`, `nginx`, `tomcat` 서비스 인스턴스는 (최상위 `metrics > features`
  구성에 정의된 대로) `application_service_graph` 메트릭만 내보낸다.

- `pyserver` 서비스는 `application` 메트릭 그룹만 내보낸다.

- 포트 3030 또는 3040에서 수신 대기하는 서비스는 `application`,
  `application_span`, `application_service_graph` 메트릭 그룹을 내보낸다.

```yaml
metrics:
  features: ['application_service_graph']
discovery:
  instrument:
    - open_ports: 3030,3040
      metrics:
        features:
          - 'application'
          - 'application_span'
          - 'application_service_graph'
    - name: pyserver
      open_ports: 7773
      metrics:
        features:
          - 'application'
    - name: apache
      open_ports: 8080
    - name: nginx
      open_ports: 8085
    - name: tomcat
      open_ports: 8090
```

## 런타임 메트릭 {#runtime-metrics}

OBI는 대상 프로세스에서 SDK를 변경하지 않고도 런타임 메트릭을 수집할 수 있다.

`application_runtime` 메트릭 기능으로 Go, HotSpot JVM, Node.js 런타임 메트릭을
활성화한다. Go 런타임 값은 대상이 가비지 컬렉션(garbage collection) 주기를
완료한 후에 보고되므로, 새 프로세스는 이 메트릭을 즉시 내보내지 않을 수 있다.
`GOGC`, `GOMEMLIMIT`, `GOMAXPROCS`의 변경 사항은 다음 완료된 주기 이후에
나타난다.

Node.js의 경우, OBI는 이벤트 루프 시간, 사용률, 지연을 초당 한 번 샘플링한다.
이벤트 루프 시간과 사용률에는 Node.js 14.10 이상이 필요하다. 지연 메트릭에는
Node.js 16.14 이상이 필요하다. OBI는 메인 스레드 이벤트 루프만 보고하며, OBI가
런타임 메트릭 에이전트를 주입하려면 inspector에 접근할 수 있어야 한다.

`jvm_runtime_metrics.sampling_interval`을 사용하여 OBI가 HotSpot JVM 메모리
메트릭을 샘플링하는 빈도를 제어한다.

```yaml
metrics:
  features: [application_runtime]
jvm_runtime_metrics:
  sampling_interval: 1s
```

| YAML<br>환경 변수                                                  | 설명                                                  | 유형     | 기본값 |
| ------------------------------------------------------------------ | ----------------------------------------------------- | -------- | ------ |
| `sampling_interval`<br>`OBI_JVM_RUNTIME_METRICS_SAMPLING_INTERVAL` | OBI가 JVM 런타임 메트릭을 샘플링하는 빈도를 설정한다. | Duration | `1s`   |

## 오픈텔레메트리 메트릭 익스포터 구성 요소 {#opentelemetry-metrics-exporter-component}

YAML 섹션: `otel_metrics_export`

구성 파일에서 또는 환경 변수를 통해 endpoint 속성을 설정하여 오픈텔레메트리
메트릭 내보내기 구성 요소를 활성화한다.
[메트릭 내보내기 구성 옵션](#opentelemetry-metrics-exporter-component)을
참고한다.

YAML 구성의 `otel_metrics_export` 섹션이나 환경 변수로 이 구성 요소를 구성한다.

이 문서에 설명된 구성 외에도, 이 구성 요소는
[표준 오픈텔레메트리 익스포터 구성](/docs/languages/sdk-configuration/otlp-exporter/)의
환경 변수를 지원한다.

예를 들면 다음과 같다.

```yaml
otel_metrics_export:
  ttl: 5m
  endpoint: http://otelcol:4318
  protocol: grpc
  buckets:
    duration_histogram: [0, 1, 2]
  histogram_aggregation: base2_exponential_bucket_histogram
```

| YAML<br>환경 변수                                                                        | 설명                                                                                                                                                                                                                                                                                           | 유형            | 기본값                      |
| ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | --------------------------- |
| `endpoint`<br>`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`                                      | OBI가 메트릭을 보내는 엔드포인트. HTTP(S), gRPC 및 `unix:///var/run/otel.sock`이나 `unix://@otel` 같은 Unix 도메인 소켓 엔드포인트를 지원한다.                                                                                                                                                 | URL             |                             |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                            | 메트릭 및 트레이스 익스포터의 공유 엔드포인트. OBI는 오픈텔레메트리 표준에 따라 메트릭을 보낼 때 URL에 `/v1/metrics` 경로를 추가한다. 이 동작을 방지하려면 메트릭 전용 설정을 사용한다.                                                                                                        | URL             |                             |
| `protocol`<br>`OTEL_EXPORTER_OTLP_METRICS_PROTOCOL`                                      | 오픈텔레메트리 엔드포인트의 프로토콜 전송/인코딩. [메트릭 내보내기 프로토콜](#metrics-export-protocol)을 참고한다. [허용 값](/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_protocol) `http/json`, `http/protobuf`, `grpc`.                                               | string          | 포트 사용에서 추론됨        |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                            | 공유 엔드포인트와 유사하게, 메트릭 및 트레이스의 프로토콜.                                                                                                                                                                                                                                     | string          | 포트 사용에서 추론됨        |
| `insecure_skip_verify`<br>`OTEL_EBPF_INSECURE_SKIP_VERIFY`                               | `true`이면 OBI는 검증을 건너뛰고 모든 서버 인증서를 수락한다. 비프로덕션 환경에서만 이 설정을 재정의한다.                                                                                                                                                                                      | boolean         | `false`                     |
| `interval`<br>`OTEL_EBPF_METRICS_INTERVAL`                                               | 내보내기 사이의 기간.                                                                                                                                                                                                                                                                          | Duration        | `60s`                       |
| `allow_service_graph_self_references`<br>`OTEL_EBPF_ALLOW_SERVICE_GRAPH_SELF_REFERENCES` | 서비스 그래프 생성에서 OBI가 자기 참조 서비스(예: 자신을 호출하는 서비스)를 포함할지 제어한다. 자기 참조는 서비스 그래프의 유용성을 떨어뜨리고 데이터 카디널리티를 증가시킨다.                                                                                                                 | boolean         | `false`                     |
| `instrumentations`<br>`OTEL_EBPF_METRICS_INSTRUMENTATIONS`                               | OBI가 데이터를 수집하는 메트릭 계측의 목록. [메트릭 계측](#metrics-instrumentation) 섹션을 참고한다.                                                                                                                                                                                           | list of strings | `["*"]`                     |
| `buckets`                                                                                | 다양한 히스토그램의 버킷 경계를 재정의하는 방법을 설정한다. [히스토그램 버킷 재정의](../metrics-histograms/)를 참고한다.                                                                                                                                                                       | (n/a)           | Object                      |
| `histogram_aggregation`<br>`OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION`    | OBI가 히스토그램 계측기에 사용하는 기본 집계를 설정한다. 허용 값 [`explicit_bucket_histogram`](/docs/specs/otel/metrics/sdk/#explicit-bucket-histogram-aggregation) 또는 [`base2_exponential_bucket_histogram`](/docs/specs/otel/metrics/sdk/#base2-exponential-bucket-histogram-aggregation). | `string`        | `explicit_bucket_histogram` |

### 메트릭 내보내기 프로토콜 {#metrics-export-protocol}

프로토콜을 설정하지 않으면 OBI는 다음과 같이 프로토콜을 설정한다.

- `grpc`: 포트가 `4317`로 끝나는 경우(예: `4317`, `14317`, `24317`).
- `http/protobuf`: 포트가 `4318`로 끝나는 경우(예: `4318`, `14318`, `24318`).

### 메트릭 계측 {#metrics-instrumentation}

OBI가 데이터를 수집할 수 있는 계측 영역 목록:

- `*`: 모든 계측. `*`가 있으면 OBI는 다른 값을 무시한다
- `http`: HTTP/HTTPS/HTTP/2 애플리케이션 메트릭
- `grpc`: gRPC 애플리케이션 메트릭
- `sql`: SQL 데이터베이스 클라이언트 호출 메트릭(PostgreSQL, MySQL 및 pgx 같은
  Go `database/sql` 드라이버 포함)
- `redis`: Redis 클라이언트/서버 데이터베이스 메트릭
- `kafka`: Kafka 클라이언트/서버 메시지 큐 메트릭
- `mqtt`: MQTT 게시/구독 메시지 메트릭(MQTT 3.1.1 및 5.0)
- `nats`: NATS 게시/구독 메시지 메트릭
- `amqp`: AMQP 1.0 게시/처리 메시지 메트릭
- `couchbase`: memcached 프로토콜 기반의 Couchbase N1QL/SQL++ 쿼리 메트릭 및
  KV(Key-Value) 프로토콜 메트릭
- `memcached`: Memcached ASCII 프로토콜 메트릭
- `genai`: GenAI 클라이언트 메트릭(OpenAI, Anthropic, Gemini, AWS Bedrock, Qwen,
  MCP 및 지원되는 임베딩, 재순위(rerank), 벡터 검색 API)
- `gpu`: GPU 성능 메트릭
- `mongo`: MongoDB 클라이언트 호출 메트릭
- `dns`: DNS 쿼리 메트릭
- `sunrpc`: TCP를 통한 ONC RPC 클라이언트 및 서버 메트릭

예를 들어 `instrumentations` 옵션을 `http,grpc`로 설정하면 `HTTP/HTTPS/HTTP2`와
`gRPC` 애플리케이션 메트릭 수집이 활성화되고 다른 계측은 비활성화된다.

| YAML<br>환경 변수                                          | 설명                                                                                          | 유형            | 기본값  |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------- | ------- |
| `instrumentations`<br>`OTEL_EBPF_METRICS_INSTRUMENTATIONS` | OBI가 데이터를 수집하는 계측의 목록. [메트릭 계측](#metrics-instrumentation) 섹션을 참고한다. | list of strings | `["*"]` |

## 오픈텔레메트리 트레이스 익스포터 구성 요소 {#opentelemetry-traces-exporter-component}

YAML 섹션: `otel_traces_export`

YAML 구성의 `otel_traces_export` 섹션이나 환경 변수로 이 구성 요소를 구성할 수
있다.

이 문서에 설명된 구성 외에도, 이 구성 요소는
[표준 오픈텔레메트리 익스포터 구성](/docs/languages/sdk-configuration/otlp-exporter/)의
환경 변수를 지원한다.

```yaml
otel_traces_export:
  endpoint: http://jaeger:4317
  protocol: grpc
  instrumentations: ['http', 'sql']
```

| YAML<br>환경 변수                                                                   | 설명                                                                                                                                                                                                                                                    | 유형            | 기본값                                                                                                       |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------ |
| `endpoint`<br>`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`<br>`OTEL_EXPORTER_OTLP_ENDPOINT` | OBI가 트레이스를 보내는 엔드포인트. `unix:///var/run/otel.sock`이나 `unix://@otel` 같은 Unix 도메인 소켓을 지원한다. `OTEL_EXPORTER_OTLP_ENDPOINT`를 사용하면 OBI는 오픈텔레메트리 표준을 따르며 HTTP 내보내기에 대해 자동으로 `/v1/traces`를 추가한다. | URL             |                                                                                                              |
| `protocol`<br>`OTEL_EXPORTER_OTLP_TRACES_PROTOCOL`<br>`OTEL_EXPORTER_OTLP_PROTOCOL` | 오픈텔레메트리 엔드포인트의 프로토콜 전송/인코딩. [트레이스 내보내기 프로토콜](#traces-export-protocol)을 참고한다. [허용 값](/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_protocol) `http/json`, `http/protobuf`, `grpc`.       | string          | 포트 사용에서 추론됨                                                                                         |
| `insecure_skip_verify`<br>`OTEL_EBPF_INSECURE_SKIP_VERIFY`                          | `true`이면 OBI는 검증을 건너뛰고 모든 서버 인증서를 수락한다. 비프로덕션 환경에서만 이 설정을 재정의한다.                                                                                                                                               | boolean         | `false`                                                                                                      |
| `instrumentations`<br>`OTEL_EBPF_TRACES_INSTRUMENTATIONS`                           | OBI가 데이터를 수집하는 계측의 목록. [트레이스 계측](#traces-instrumentation) 섹션을 참고한다.                                                                                                                                                          | list of strings | `http`, `grpc`, `sql`, `redis`, `kafka`, `mqtt`, `nats`, `amqp`, `mongo`, `couchbase`, `memcached`, `sunrpc` |

### 트레이스 내보내기 프로토콜 {#traces-export-protocol}

프로토콜을 설정하지 않으면 OBI는 다음과 같이 프로토콜을 설정한다.

- `grpc`: 포트가 `4317`로 끝나는 경우(예: `4317`, `14317`, `24317`).
- `http/protobuf`: 포트가 `4318`로 끝나는 경우(예: `4318`, `14318`, `24318`).

### 트레이스 계측 {#traces-instrumentation}

OBI가 데이터를 수집할 수 있는 계측 영역 목록:

- `*`: 모든 계측. `*`가 있으면 OBI는 다른 값을 무시한다
- `http`: HTTP/HTTPS/HTTP/2 애플리케이션 트레이스
- `grpc`: gRPC 애플리케이션 트레이스
- `sql`: SQL 데이터베이스 클라이언트 호출 메트릭(PostgreSQL, MySQL 및 pgx 같은
  Go `database/sql` 드라이버 포함)
- `redis`: Redis 클라이언트/서버 데이터베이스 트레이스
- `kafka`: Kafka 클라이언트/서버 메시지 큐 트레이스
- `mqtt`: MQTT 게시/구독 메시지 트레이스(MQTT 3.1.1 및 5.0)
- `nats`: NATS 게시/구독 메시지 트레이스
- `amqp`: AMQP 1.0 게시/처리 메시지 트레이스
- `couchbase`: 쿼리 텍스트 및 작업 세부 정보를 포함한 Couchbase N1QL/SQL++ 쿼리
  트레이스 및 KV(Key-Value) 프로토콜 트레이스
- `memcached`: Memcached ASCII 프로토콜 트레이스
- `genai`: GenAI 클라이언트 트레이스(OpenAI, Anthropic, Gemini, AWS Bedrock,
  Qwen, MCP 및 지원되는 임베딩, 재순위(rerank), 벡터 검색 API)
- `gpu`: GPU 성능 트레이스
- `mongo`: MongoDB 클라이언트 호출 트레이스
- `dns`: DNS 쿼리 트레이스
- `sunrpc`: TCP를 통한 ONC RPC 클라이언트 및 서버 트레이스

예를 들어 `instrumentations` 옵션을 `http,grpc`로 설정하면 `HTTP/HTTPS/HTTP2`와
`gRPC` 애플리케이션 트레이스 수집이 활성화되고 다른 계측은 비활성화된다.

#### MQTT 계측 {#mqtt-instrumentation}

OBI는 IoT 및 임베디드 시스템에서 흔히 사용되는 경량 메시징 프로토콜인 MQTT
통신을 자동으로 계측한다.

**지원되는 작업**:

- `publish`: 토픽으로의 메시지 게시
- `subscribe`: 토픽 구독 요청

**프로토콜 버전**:

- MQTT 3.1.1
- MQTT 5.0

**캡처되는 항목**:

- 토픽 이름(subscribe 작업의 경우 첫 번째 토픽 필터로 제한됨)
- 작업 지연 시간
- 클라이언트-서버 통신 패턴

**제한 사항**:

- subscribe 작업의 경우 첫 번째 토픽 필터만 캡처된다
- 오버헤드를 최소화하기 위해 메시지 페이로드는 캡처되지 않는다

**예시 사용 사례**: 센서 데이터를 MQTT 브로커에 게시하는 IoT 게이트웨이를
모니터링하여 메시지 전달률을 추적하고 통신 문제를 식별한다.

#### PostgreSQL pgx 드라이버 계측 {#postgresql-pgx-driver-instrumentation}

OBI는 PostgreSQL 데이터베이스용 고성능 네이티브 Go 드라이버인 pgx에 대한 전용
계측을 제공한다.

**pgx가 특별한 이유**: pgx 계측은 Go 전용 eBPF 트레이싱을 사용하여 Go 드라이버에
직접 연결되며, 범용 네트워크 수준 SQL 계측의 오버헤드 없이 데이터베이스별
옵저버빌리티를 제공한다.

**지원되는 작업**:

- `Query`: 결과 집합이 있는 SQL 쿼리 실행
- 커넥션 풀링(pgxpool을 통해)
- 네이티브 pgx API와 database/SQL 래퍼 인터페이스 모두

**캡처되는 항목**:

- SQL 쿼리 텍스트
- PostgreSQL 서버 호스트 이름(pgx 연결 구성에서 추출됨)
- 작업 지속 시간 및 오류 세부 정보
- 모든 표준 database/SQL 메트릭 레이블

**지원되는 pgx 버전**: pgx v5.0.0 이상(v5.8.0까지 테스트됨). database/SQL 래퍼를
통해서도 지원됨: `github.com/jackc/pgx/v5/stdlib`

#### Couchbase 계측 {#couchbase-instrumentation}

Couchbase는 직접 키-값 액세스와 SQL++를 통한 SQL과 유사한 쿼리를 모두 지원하는
NoSQL 문서 데이터베이스로, 유연한 스키마와 고가용성 요구 사항이 있는
애플리케이션에 흔히 사용된다. OBI는 두 가지 프로토콜을 통해 Couchbase 작업을
계측한다.

- **KV(Key-Value) 프로토콜**: 포트 11210에서 직접 키-값 액세스를 위한 바이너리
  프로토콜로,
  [Memcached 바이너리 프로토콜](https://github.com/couchbase/memcached/blob/master/docs/BinaryProtocol.md)의
  확장에 기반한다.
- **SQL++(N1QL)**: `/query/service` 엔드포인트를 통해 포트 8093에서 동작하는
  HTTP 기반 쿼리 프로토콜.

##### KV(Key-Value) 프로토콜 {#kv-key-value-protocol}

**캡처되는 항목**:

| 속성                      | 소스               | 예시                |
| ------------------------- | ------------------ | ------------------- |
| `db.system.name`          | 상수               | `couchbase`         |
| `db.operation.name`       | 옵코드(opcode)     | `GET`, `SET`        |
| `db.namespace`            | 버킷               | `travel-sample`     |
| `db.collection.name`      | 스코프 + 컬렉션    | `inventory.airline` |
| `db.collection.name`      | 컬렉션             | `airline`           |
| `db.response.status_code` | 상태 코드(오류 시) | `1`                 |
| `server.address`          | 연결 정보          | 서버 호스트 이름    |
| `server.port`             | 연결 정보          | `11210`             |

**버킷, 스코프, 컬렉션 추적**: Couchbase는 계층적 네임스페이스(버킷 → 스코프 →
컬렉션)를 사용한다. 요청별 네임스페이스 프로토콜과 달리, 네임스페이스는 연결
수준에서 설정된다.

- `SELECT_BUCKET`(트레이스되지 않음): 연결에서 이후의 모든 작업에 대한 활성
  버킷을 설정한다. MySQL의 `USE database`나 Redis의 `SELECT db_number`와
  유사하다.
- `GET_COLLECTION_ID`(트레이스되지 않음): `scope.collection` 경로를 숫자 컬렉션
  ID로 확인한다. OBI는 이를 사용하여 스코프 및 컬렉션 이름으로 스팬 속성을
  보강한다.

OBI는 버킷, 스코프, 컬렉션 이름의 연결별 캐시를 유지하며, 이를 사용하여 이후의
모든 스팬에 주석을 단다.

**제한 사항**:

- `SELECT_BUCKET`이 OBI가 시작되기 전에 발생하면 해당 연결에 대한 버킷 이름을 알
  수 없다
- `GET_COLLECTION_ID`가 OBI가 시작되기 전에 발생하면 컬렉션 이름을 사용할 수
  없다
- 인증 및 메타데이터 작업은 캡처되지 않는다
- 이러한 제한 사항은 OBI 초기화 전에 설정된 연결에만 영향을 미친다

##### SQL++(N1QL) 작업 {#sql-n1ql-operations}

SQL++ 쿼리(N1QL 쿼리 언어의 최신 이름)는 포트 8093의 `/query/service`
엔드포인트에서 Couchbase의 HTTP 쿼리 서비스를 통해 자동으로 감지된다.

**지원되는 작업**:

- 모든 SQL++ 쿼리 유형: SELECT, INSERT, UPDATE, DELETE, UPSERT
- SQL 경로를 통해 액세스되는 버킷 및 컬렉션 작업(예: `bucket.scope.collection`)
- 컬렉션 간 및 버킷 간 쿼리

**캡처되는 항목**:

| 속성                      | 소스                          | 예시                         |
| ------------------------- | ----------------------------- | ---------------------------- |
| `db.system.name`          | N1QL 버전 헤더                | `couchbase` 또는 `other_sql` |
| `db.operation.name`       | SQL 파서                      | `SELECT`, `INSERT`, `UPDATE` |
| `db.namespace`            | 테이블 경로 / `query_context` | `travel-sample`              |
| `db.collection.name`      | 테이블 경로                   | `inventory.airline`          |
| `db.query.text`           | 요청 본문                     | 전체 SQL++ 쿼리 텍스트       |
| `db.response.status_code` | 오류 코드(오류 시)            | `12003`                      |
| `error.type`              | 오류 메시지(오류 시)          | Couchbase의 오류 메시지      |

**지원되는 데이터베이스**:

- **Couchbase Server**: 응답의 N1QL 버전 헤더를 통해 감지됨
- **기타 SQL++ 구현**: Apache AsterixDB 및 호환 데이터베이스도 일반 `other_sql`
  지정으로 지원됨

**요청 형식**: SQL++ 요청은 `/query/service`로 보내는 JSON 본문과 form-encoded
POST 형태로 모두 받는다.

{{< tabpane text=true >}} {{% tab "JSON 본문" %}}

```json
{
  "statement": "SELECT * FROM `bucket`.`scope`.`collection` WHERE id = $1",
  "query_context": "default:`bucket`.`scope`"
}
```

{{% /tab %}} {{% tab "폼 인코딩" %}}

```text
statement=SELECT+*+FROM+users&query_context=default:`travel-sample`.`inventory`
```

{{% /tab %}} {{< /tabpane >}}

**네임스페이스 확인**: 파서는 다음에서 버킷과 컬렉션을 추출한다.

1. SQL 문의 테이블 경로: `` `bucket`.`scope`.`collection` ``
2. 존재하는 경우 `query_context` 필드
3. 단일 식별자: 컬렉션 이름(`query_context`가 있는 경우)이나 버킷
   이름(`query_context`가 없는 경우, 레거시 모드)으로 처리됨

**구성**: SQL++ 계측은 명시적 활성화가 필요하다.

```bash
export OTEL_EBPF_HTTP_SQLPP_ENABLED=true
export OTEL_EBPF_BPF_BUFFER_SIZE_HTTP=2048  # Larger than default; needed to capture request/response bodies
```

**제한 사항**:

- 버킷 및 컬렉션 디스커버리에는 쿼리의 SQL 경로 표기법(예:
  `bucket.scope.collection`)이나 요청의 `query_context` 필드가 필요하다
- Couchbase 버전 헤더가 없는 응답은 일반 `other_sql` 작업으로 레이블이 지정된다

**예시 사용 사례**: 세션 저장 및 콘텐츠 관리에 Couchbase를 사용하는 트래픽이
많은 웹 애플리케이션을 모니터링하여 쿼리 성능을 추적하고 비효율적인 N1QL 쿼리를
식별한다.

## Prometheus 익스포터 구성 요소 {#prometheus-exporter-component}

YAML 섹션: `prometheus_export`

YAML 구성의 `prometheus_export` 섹션이나 환경 변수로 이 구성 요소를 구성할 수
있다. 이 구성 요소는 자동 계측 도구에 HTTP 엔드포인트를 열어, 외부 스크레이퍼가
Prometheus 형식으로 메트릭을 가져갈 수 있게 한다. `port` 속성이 설정되면
활성화된다.

```yaml
prometheus_export:
  port: 8999
  path: /metrics
  extra_resource_attributes: ['deployment_environment']
  ttl: 1s
  buckets:
    request_size_histogram: [0, 10, 20, 22]
    response_size_histogram: [0, 10, 20, 22]
  instrumentations: ['http', 'sql']
```

| YAML<br>환경 변수                                                                                   | 설명                                                                                                                                                                                 | 유형            | 기본값       |
| --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------- | ------------ |
| `port`<br>`OTEL_EBPF_PROMETHEUS_PORT`                                                               | Prometheus 스크레이프 엔드포인트의 HTTP 포트. 설정하지 않거나 0이면 Prometheus 엔드포인트가 열리지 않는다.                                                                           | int             |              |
| `path`<br>`OTEL_EBPF_PROMETHEUS_PATH`                                                               | Prometheus 메트릭 목록을 가져올 HTTP 쿼리 경로.                                                                                                                                      | string          | `/metrics`   |
| `extra_resource_attributes`<br>`OTEL_EBPF_PROMETHEUS_EXTRA_RESOURCE_ATTRIBUTES`                     | 보고되는 `target_info` 메트릭에 추가할 추가 리소스 속성의 목록. 런타임에 발견된 속성에 대한 중요한 세부 사항은 [추가 리소스 속성](#prometheus-extra-resource-attributes)을 참고한다. | list of strings |              |
| `ttl`<br>`OTEL_EBPF_PROMETHEUS_TTL`                                                                 | 메트릭 인스턴스가 업데이트되지 않은 경우 보고되지 않게 되는 기간. 이미 종료된 애플리케이션 인스턴스를 무기한 보고하는 것을 방지하는 데 사용된다.                                     | Duration        | `5m`         |
| `buckets`                                                                                           | 다양한 히스토그램의 버킷 경계를 재정의하는 방법을 설정한다. [히스토그램 버킷 재정의](../metrics-histograms/)를 참고한다.                                                             | Object          |              |
| `exemplar_filter`<br>`OTEL_EBPF_PROMETHEUS_EXEMPLAR_FILTER`                                         | 이그젬플러(exemplar)가 Prometheus 메트릭에 첨부되는 시점을 제어한다. 허용 값: `always_on`, `always_off`, `trace_based`.                                                              | string          | `always_off` |
| `allow_service_graph_self_references`<br>`OTEL_EBPF_PROMETHEUS_ALLOW_SERVICE_GRAPH_SELF_REFERENCES` | 서비스 그래프 생성에서 OBI가 자기 참조 서비스를 포함하는지 여부. 자기 참조는 서비스 그래프에 유용하지 않으며 데이터 카디널리티를 증가시킨다.                                         | boolean         | `false`      |
| `instrumentations`<br>`OTEL_EBPF_PROMETHEUS_INSTRUMENTATIONS`                                       | OBI가 데이터를 수집하는 계측의 목록. [Prometheus 계측](#prometheus-instrumentation) 섹션을 참고한다.                                                                                 | list of strings | `["*"]`      |

### Prometheus 추가 리소스 속성 {#prometheus-extra-resource-attributes}

Prometheus API 클라이언트의 내부 제한으로 인해, OBI는 각 메트릭에 대해 어떤
속성이 노출되는지 미리 알아야 한다. 이로 인해 계측 중 런타임에 발견되는 일부
속성은 기본적으로 표시되지 않는다. 예를 들어 쿠버네티스 어노테이션을 통해 각
애플리케이션에 정의된 속성이나, 대상 애플리케이션의 `OTEL_RESOURCE_ATTRIBUTES`
환경 변수에 정의된 속성이 그렇다.

예를 들어 `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production`을 환경
변수로 정의하는 애플리케이션의 경우, 메트릭이 오픈텔레메트리를 통해 내보내지면
`target_info{deployment.environment="production"}` 속성이 기본적으로 표시되지만
Prometheus를 통해 내보내지면 표시되지 않는다.

Prometheus에서 `deployment_environment`를 표시하려면 `extra_resource_attributes`
목록에 추가해야 한다.

### Prometheus 계측 {#prometheus-instrumentation}

OBI가 데이터를 수집할 수 있는 계측 영역 목록:

- `*`: 모든 계측. `*`가 있으면 OBI는 다른 값을 무시한다
- `http`: HTTP/HTTPS/HTTP/2 애플리케이션 메트릭
- `grpc`: gRPC 애플리케이션 메트릭
- `sql`: SQL 데이터베이스 클라이언트 호출 메트릭(PostgreSQL, MySQL 및 pgx 같은
  Go `database/sql` 드라이버 포함)
- `redis`: Redis 클라이언트/서버 데이터베이스 메트릭
- `kafka`: Kafka 클라이언트/서버 메시지 큐 메트릭
- `mqtt`: MQTT 게시/구독 메시지 메트릭
- `nats`: NATS 게시/구독 메시지 메트릭
- `amqp`: AMQP 1.0 게시/처리 메시지 메트릭
- `couchbase`: Couchbase N1QL/SQL++ 쿼리 메트릭 및 KV 프로토콜 메트릭
- `memcached`: Memcached ASCII 프로토콜 메트릭
- `genai`: GenAI 클라이언트 메트릭(OpenAI, Anthropic, Gemini, AWS Bedrock, Qwen,
  MCP 및 지원되는 임베딩, 재순위(rerank), 벡터 검색 API)
- `gpu`: GPU 성능 메트릭
- `mongo`: MongoDB 클라이언트 호출 메트릭
- `dns`: DNS 쿼리 메트릭
- `sunrpc`: TCP를 통한 ONC RPC 클라이언트 및 서버 메트릭

예를 들어 `instrumentations` 옵션을 `http,grpc`로 설정하면 `HTTP/HTTPS/HTTP2`와
`gRPC` 애플리케이션 메트릭 수집이 활성화되고 다른 계측은 비활성화된다.
