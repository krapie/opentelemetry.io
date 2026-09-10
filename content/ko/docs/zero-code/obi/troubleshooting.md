---
title: 문제 해결
description: OBI의 일반적인 문제 및 오류 해결
weight: 22
cSpell:ignore: Clickhouse uprobe uprobes userland
default_lang_commit: f4cc67cd44fa9d9f23de8f5a121f15d7eea9b043
---

이 페이지에서는 일반적인 OBI 오류와 문제를 진단하고 해결하는 방법을 알아본다.

## 문제 해결 도구 {#troubleshooting-tools}

OBI는 문제를 진단하고 해결하는 데 도움이 되는 다양한 도구와 구성 옵션을
제공한다.

### 상세 로깅 {#detailed-logging}

`log_level` 구성이나 `OTEL_EBPF_LOG_LEVEL` 환경 변수를 `debug`로 설정하여 OBI의
로깅 상세 수준을 높일 수 있다. 이렇게 하면 문제 진단에 도움이 될 수 있는 더
상세한 로그가 제공된다.

BPF 프로그램의 로깅을 활성화하려면 `ebpf.bpf_debug` 구성이나
`OTEL_EBPF_BPF_DEBUG` 환경 변수를 `true`로 설정한다. 많은 양의 로그를 생성할 수
있으므로 **디버깅 용도로만 사용한다**.

### 구성 로깅 {#configuration-logging}

기본적으로 OBI는 우선순위가 낮은 것부터 높은 것 순으로 세 가지 서로 다른
소스에서 구성을 병합한다.

- 내장 기본 구성
- `--config` 플래그나 `OTEL_EBPF_CONFIG_PATH`로 제공되는 구성 파일
- 일반적으로 `OTEL_EBPF_`로 시작하는 환경 변수

최종 병합된 구성을 확인하는 것이 유용한 경우가 많다. `log_config` 구성 값(또는
`OTEL_EBPF_LOG_CONFIG` 환경 변수)을 사용하면 OBI가 시작 시 최종 구성을
로깅하도록 지시할 수 있다.

`log_config`는 다음 값을 지원한다.

- `yaml` — 최종 구성을 YAML 형식으로 로깅한다. 구성 파일 구조와 일치하므로
  사람이 읽기에 가장 좋다
- `json` — 최종 구성을 JSON 형식으로 로깅한다. 단일 구조화된 줄이므로 로그
  수집기(shipper)에 가장 적합하다

### 내부 메트릭 {#internal-metrics}

[OBI 내부 메트릭](../metrics/#internal-metrics)을 구성하고 사용하여 성능과 내부
상태를 모니터링할 수 있다.

내부 메트릭을 켜려면 `internal_metrics.exporter`를 다음 값 중 하나로 구성한다.

- `none`(기본값): 내부 메트릭을 비활성화한다
- `prometheus`: HTTP 서버를 통해 내부 메트릭을 Prometheus 형식으로 내보낸다
- `otlp`: OTLP 익스포터를 통해 내부 메트릭을 내보낸다

### 디버그 트레이스 익스포터 {#debug-traces-exporter}

OBI가 생성한 원시 트레이스 스팬을 디버깅하려면 `otel_traces_exporter.protocol`
구성 값이나 `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` 환경 변수를 `debug`로 설정할 수
있다. 이렇게 하면 원시 트레이스 스팬을 사람이 읽을 수 있는 형식으로 콘솔에
로깅하며, 이는 `verbosity: detailed`로 설정한 OTel 컬렉터 디버그 익스포터와
일치한다. 콘솔에 출력되는 스팬 예시는 다음과 같다.

```text
Traces	{"resource spans": 1, "spans": 1}
ResourceSpans #0
Resource SchemaURL:
Resource attributes:
     -> service.name: Str(flagd)
     -> telemetry.sdk.language: Str(go)
     -> telemetry.sdk.name: Str(opentelemetry)
     -> telemetry.distro.name: Str(opentelemetry-ebpf-instrumentation)
     -> telemetry.sdk.version: Str(main)
     -> host.name: Str(flagd-5cccb4c4f5-sfkcm)
     -> os.type: Str(linux)
     -> service.namespace: Str(opentelemetry-demo)
     -> k8s.owner.name: Str(flagd)
     -> k8s.kind: Str(Deployment)
     -> k8s.replicaset.name: Str(flagd-5cccb4c4f5)
     -> k8s.pod.name: Str(flagd-5cccb4c4f5-sfkcm)
     -> k8s.container.name: Str(flagd)
     -> k8s.deployment.name: Str(flagd)
     -> service.version: Str(2.0.2)
     -> k8s.namespace.name: Str(default)
     -> otel.library.name: Str(go.opentelemetry.io/obi)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope
Span #0
    Trace ID       : 63a2723a58e0033170e58b1ff27ef03d
    Parent ID      :
    ID             : fab47609b60cc4e0
    Name           : /opentelemetry.proto.collector.metrics.v1.MetricsService/Export
    Kind           : Client
    Start time     : 2025-11-28 16:10:35.4241749 +0000 UTC
    End time       : 2025-11-28 16:10:35.42555658 +0000 UTC
    Status code    : Unset
    Status message :
Attributes:
     -> rpc.method: Str(/opentelemetry.proto.collector.metrics.v1.MetricsService/Export)
     -> rpc.system.name: Str(grpc)
     -> rpc.response.status_code: Str(OK)
     -> server.address: Str(otel-collector.default)
     -> peer.service: Str(otel-collector.default)
     -> server.port: Int(4317)
```

OBI v0.6.0부터 `telemetry.sdk.name`은 사용 가능한 경우 기반 SDK를 반영하며,
OBI는 `telemetry.distro.name`을 사용하여 자신을 식별한다.

### 성능 프로파일러(pprof) {#performance-profiler-pprof}

OBI는 성능 프로파일링을 허용하기 위해 `pprof` 포트를 노출할 수 있다. 이를
활성화하려면 `profile_port` 구성 값이나 `OTEL_EBPF_PROFILE_PORT` 환경 변수를
원하는 포트로 설정한다.

이는 고급 사용 사례이며 일반적으로 필요하지 않다.

## 일반적인 OBI 문제 {#common-obi-issues}

이 섹션에서는 일반적인 OBI 문제를 해결하는 방법을 다룬다.

### OBI 실행 중 ClickHouse 인스턴스가 크래시한다 {#clickhouse-instances-crash-when-obi-is-running}

OBI와 동일한 노드에서 [Clickhouse](https://github.com/ClickHouse/ClickHouse)를
실행 중이라면, 다음과 같은 로그와 함께 ClickHouse가 크래시하는 것을 볼 수 있다.

```text
Application: Code: 246. DB::Exception: Calculated checksum of the executable (...) does not correspond to the reference checksum ...
```

이 문제는 OBI가 ClickHouse 바이너리에 eBPF uprobe를 연결하기 때문에 발생할
가능성이 높다.
[관련 GitHub](https://github.com/ClickHouse/ClickHouse/issues/83637) 이슈가 이
동작을 설명한다.

> uprobe를 연결하면, 커널은 연결 주소에 트랩 명령어를 삽입하기 위해 대상
> 프로세스 메모리를 수정한다. 이로 인해 시작 중 ClickHouse 바이너리 체크섬
> 검증이 실패한다.

**해결책:**

[skip_binary_checksum_checks](https://clickhouse.com/docs/operations/server-configuration-parameters/settings#skip_binary_checksum_checks)
플래그와 함께 ClickHouse를 시작한다

### Go 애플리케이션 또는 TLS 요청의 텔레메트리 데이터 누락 {#missing-telemetry-data-for-go-applications-or-tls-requests}

Go 애플리케이션이나 TLS 요청(예: HTTPS 통신)에서 오는 텔레메트리가 누락된다면,
uprobe 연결에 필요한 권한이 부족하기 때문일 수 있다. 많은 이전 커널 버전으로
백포트된 최근의 일부 커널 보안 변경으로 인해, 이제 uprobe에는 `CAP_SYS_ADMIN`
capability가 필요하다. OBI는 다른 런타임/언어별 계측과 함께 Golang
애플리케이션과 TLS 요청을 계측하는 데 uprobe를 사용한다. OBI 배포 보안 구성이
권한 있는 작업(예: `privileged:true` 또는 Docker와 쿠버네티스)을 사용하지 않거나
`CAP_SYS_ADMIN`을 보안 capability로 제공하지 않으면, 일부 또는 모든 텔레메트리가
보이지 않을 수 있다.

이 문제를 해결하려면 `OTEL_EBPF_LOG_LEVEL=debug`로 상세한 OBI 로깅을 활성화한다.
모든 uprobe 주입이 "setting uprobe (offset)..." 오류와 함께 실패하는 것이
보인다면, 이 문제를 겪고 있을 가능성이 높다.

**해결책:**

다음 중 하나를 수행할 수 있다.

- OBI를 권한 있는 모드로 실행한다.
- 배포 보안 구성의 capabilities 목록에 `CAP_SYS_ADMIN`을 추가한다.

### 클라이언트 메트릭 또는 스팬이 잘못된 서비스에 귀속됨 {#client-metrics-or-spans-attributed-to-the-wrong-service}

OBI가 호스트 PID 네임스페이스에 접근하며 실행되고(예: Docker Compose의
`pid: host` 또는 쿠버네티스의 `hostPID: true`)
[열린 포트](../configure/service-discovery/#open-ports)로 서비스를 선택하면,
예상치 못한 나가는(클라이언트) 메트릭이나 스팬이 여러분의 서비스 중 하나에
귀속되는 것을 볼 수 있다. 흔한 증상은 대상 서비스가 Go로 작성되지 않았는데도
`GET /v1.43/containers/json` 같은 Docker Engine API 호출이 의미 없는
`server.port`와 `telemetry.sdk.language=go`를 가진 클라이언트 요청으로 보고되는
것이다.

```text
http_client_request_body_size_bytes_sum{
  http_request_method="GET",
  http_route="/v1.43/containers/json",
  server_address="docker", server_port="4",
  service_name="python-service", telemetry_sdk_language="go", ...
}
```

**원인:**

컨테이너 포트를 게시하면(예: Docker의 `-p 7773:7773` 또는 Compose `ports:`
항목), 호스트 측 포워더(forwarder) 프로세스가 호스트 네트워크 네임스페이스에서
해당 포트를 수신한다. 컨테이너 런타임에 따라 이는 `docker-proxy`(userland
프록시가 활성화된 Docker)이거나 이에 상응하는 에이전트이다.

OBI는 호스트 프로세스를 볼 수 있고 포워더는 여러분이 선택한 것과 동일한 포트를
수신하므로, OBI는 포워더를 `open_ports` 기준과 일치시켜 여러분 서비스의 식별자로
계측한다. 포워더(호스트 네트워크 네임스페이스)와 실제 서비스(해당 컨테이너
네트워크 네임스페이스) 모두 실제로 그 포트를 수신하므로, 포트만으로는 둘을
구분할 수 없다. 포워더는 일반적으로 Go 바이너리이며, 그래서 스팬에
`telemetry.sdk.language=go`가 포함된다. 그러면 포워더 자체가 생성하는 모든
트래픽(예: Docker 소켓을 통해 수행하는 Docker Engine API 호출)이 여러분의
서비스에 귀속된다.

**해결책:**

실행 파일 경로로 계측에서 포워더를 제외한다.

```yaml
discovery:
  exclude_instrument:
    - exe_path: '{*/docker-proxy,*/scon-agent}'
```

또는 열린 포트보다 더 구체적인 기준으로 서비스를 선택하여, 우연히 포트를
공유하는 호스트 측 포워더가 일치하지 않도록 한다. 예를 들어
[실행 파일 경로](../configure/service-discovery/#executable-path),
[쿠버네티스 메타데이터](../configure/service-discovery/#k8s-namespace), 또는
[컨테이너 이름](../configure/service-discovery/#container-name) 선택기를
사용한다.

## v0.7.0으로 마이그레이션: 네트워크 포트 추측 변경 사항 {#migration-to-v070-network-port-guessing-changes}

OBI v0.7.0은 호환성이 깨지는 변경을 도입한다. **이제 네트워크 포트 추측이
기본적으로 비활성화된다**. 이 변경은 네트워크 흐름에서 알 수 없는
개시자(initiator)에 대해 가정하지 않음으로써 네트워크 메트릭 정확도를 개선한다.

### 변경된 내용 {#what-changed}

v0.6.0 및 이전 버전에서는, 개시자를 확정할 수 없는 네트워크 흐름에서 OBI가 어느
엔드포인트가 클라이언트이고 어느 것이 서버인지 추측하려고 시도했다. 이 추측은
서수(ordinal) 휴리스틱에 기반했다(일반적으로 더 낮은 포트 번호를 서버로, 더 높은
포트 번호를 클라이언트로 가정).

v0.7.0에서는 이 추측이 기본적으로 비활성화되며, 이는 다음을 의미한다.

- OBI가 개시자를 확정할 수 없는 흐름에서는 `client.port`와 `server.port` 속성이
  비어 있을 수 있다
- 네트워크 메트릭이 더 정확해지지만 알 수 없는 흐름에 대한 정보를 잃을 수 있다

### 마이그레이션 방법 {#how-to-migrate}

이전 동작에 의존하고 있으며 개시자를 알 수 없을 때에도 `client.port`와
`server.port`가 추론되기를 원한다면, 서수 휴리스틱으로 포트 추측을 다시
활성화한다.

**YAML 구성:**

```yaml
network:
  guess_ports: ordinal
```

**환경 변수:**

```sh
OTEL_EBPF_NETWORK_GUESS_PORTS=ordinal
```

자세한 내용은 [네트워크 구성 문서](../network/config/)를 참고한다.

### 권장 사항 {#recommendation}

포트 추측이 필요한 특정 사용 사례가 없다면 비활성화 상태로 두는 것을 권장한다.
기본 동작은 오분류 가능성이 낮은, 더 깔끔하고 정확한 네트워크 메트릭을 제공한다.
