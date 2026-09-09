---
title: 관리
description: 대규모로 오픈텔레메트리(OpenTelemetry) 컬렉터 배포를 관리하는 방법
weight: 23
cSpell:ignore: backpressure distro opampsupervisor
default_lang_commit: 30b7dbbdd94cec0b2a0c99317272b103315518bf
---

이 문서는 대규모로 오픈텔레메트리(OpenTelemetry) 컬렉터 배포를 관리하는 방법을
설명한다.

이 페이지를 최대한 활용하려면 컬렉터를 설치하고 구성하는 방법을 알아야 한다.
이러한 주제는 다른 곳에서 다룬다.

- 오픈텔레메트리 컬렉터를 설치하는 방법을 이해하려면
  [빠른 시작](/docs/collector/quick-start/)을 참고한다.
- 오픈텔레메트리 컬렉터를 구성하고 텔레메트리 파이프라인을 설정하는 방법은
  [구성][configuration]을 참고한다.

## 기본 사항 {#basics}

대규모의 텔레메트리 수집에는 에이전트를 관리하기 위한 체계적인 접근 방식이
필요하다. 일반적인 에이전트 관리 작업에는 다음이 포함된다.

1. 에이전트 정보와 구성을 조회한다. 에이전트 정보에는 버전, 운영체제 관련 정보,
   기능(capability) 등이 포함될 수 있다. 에이전트의 구성은 텔레메트리 수집
   설정을 의미하며, 예를 들어 오픈텔레메트리 컬렉터 [구성][configuration]이
   있다.
1. 기본 에이전트 기능과 플러그인을 포함해, 에이전트를 업그레이드/다운그레이드
   하고 에이전트별 패키지를 관리한다.
1. 에이전트에 새로운 구성을 적용한다. 이는 환경 변화나 정책 변경으로 인해 필요할
   수 있다.
1. 에이전트의 상태(health) 및 성능을 모니터링한다. 일반적으로 CPU와 메모리
   사용량, 그리고 처리율(rate of processing)이나 백프레셔(backpressure) 관련
   정보와 같은 에이전트별 메트릭이 포함된다.
1. TLS 인증서 처리(폐기 및 순환)와 같은, 컨트롤 플레인(control plane)과 에이전트
   간의 연결을 관리한다.

모든 사용 사례가 위의 모든 에이전트 관리 작업을 지원해야 하는 것은 아니다.
오픈텔레메트리의 맥락에서, _4. 상태 및 성능 모니터링_ 작업은 이상적으로
오픈텔레메트리를 사용해 수행한다.

## OpAMP {#opamp}

옵저버빌리티(observability) 벤더와 클라우드 제공업체는 에이전트 관리를 위한
독점(proprietary) 솔루션을 제공한다. 오픈 소스 옵저버빌리티 분야에는 에이전트
관리를 위해 사용할 수 있는 새롭게 떠오르는 표준이 있다. 바로 [Open Agent
Management Protocol][](OpAMP)이다.

[OpAMP specification][]은 텔레메트리 데이터 에이전트 플릿(fleet)을 관리하는
방법을 정의한다. 이러한 에이전트는 [오픈텔레메트리 컬렉터](/docs/collector/),
Fluent Bit, 또는 임의의 조합으로 이루어진 다른 에이전트일 수 있다.

> [!NOTE]
>
> 여기서 "에이전트"라는 용어는 OpAMP에 응답하는 오픈텔레메트리 구성 요소를
> 통칭하는 용어로 사용되며, 이는 컬렉터일 수도 있고 SDK 구성 요소일 수도 있다.

OpAMP는 HTTP와 WebSocket을 통한 통신을 지원하는 클라이언트/서버 프로토콜이다.

- **OpAMP 서버**는 컨트롤 플레인의 일부로, 오케스트레이터(orchestrator) 역할을
  하며 텔레메트리 에이전트 플릿을 관리한다.
- **OpAMP 클라이언트**는 데이터 플레인(data plane)의 일부이다. OpAMP의
  클라이언트 측은 프로세스 내(in-process)에서 구현될 수 있으며, 예를 들어
  [오픈텔레메트리 컬렉터의 OpAMP 지원][opamp-in-otel-collector]이 이에 해당한다.
  OpAMP의 클라이언트 측은 프로세스 외부(out-of-process)에서 구현될 수도 있다. 이
  후자의 방식에서는, OpAMP 서버와의 OpAMP 관련 통신을 담당하는 동시에 구성을
  적용하거나 업그레이드하는 등 텔레메트리 에이전트를 제어하는
  슈퍼바이저(supervisor)를 사용할 수 있다. 단, 슈퍼바이저와 텔레메트리 간의
  통신은 OpAMP의 일부가 아니라는 점에 유의한다.

다음은 구체적인 설정 예이다.

![OpAMP 예제 설정](../img/opamp.svg)

1. 오픈텔레메트리 컬렉터는 다음을 수행하도록 구성된 파이프라인을 갖는다.
   - (A) 다운스트림 소스로부터 시그널을 수신한다.
   - (B) 업스트림 대상으로 시그널을 내보내며, 여기에는 컬렉터 자체에 대한
     텔레메트리(OpAMP `own_xxx` 연결 설정으로 표현됨)가 포함될 수 있다.
1. 서버 측 OpAMP 부분을 구현하는 컨트롤 플레인과, 클라이언트 측 OpAMP를 구현하는
   컬렉터(또는 컬렉터를 제어하는 슈퍼바이저) 사이의 양방향 OpAMP 제어 흐름.

### 직접 사용해 보기 {#try-it-out}

[Go로 구현된 OpAMP 프로토콜][opamp-go]을 사용하면 간단한 OpAMP 설정을 직접
사용해 볼 수 있다. 다음 안내를 따라 하려면 Go 1.22+가 필요하다.

예제 OpAMP 서버로 구성된 간단한 OpAMP 컨트롤 플레인을 설정하고, [OpAMP
슈퍼바이저][opamp-supervisor]를 사용해 오픈텔레메트리 컬렉터가 여기에 연결하도록
한다.

#### 1단계 - OpAMP 서버 시작하기 {#step-1---start-the-opamp-server}

`open-telemetry/opamp-go` 저장소를 클론한다.

```sh
git clone https://github.com/open-telemetry/opamp-go.git
```

`./opamp-go/internal/examples/server` 디렉터리에서 OpAMP 서버를 실행한다.

```console
$ go run .
2025/04/20 15:10:35.307207 [MAIN] OpAMP Server starting...
2025/04/20 15:10:35.308201 [MAIN] OpAMP Server running...
```

#### 2단계 - 오픈텔레메트리 컬렉터 설치하기 {#step-2---install-the-opentelemetry-collector}

OpAMP 슈퍼바이저가 관리할 수 있는 오픈텔레메트리 컬렉터 바이너리가 필요하다.
이를 위해 [OpenTelemetry Collector Contrib][otelcolcontrib] 배포판을 설치한다.
컬렉터 바이너리를 설치한 경로는 이후 설정에서 `$OTEL_COLLECTOR_BINARY`로
지칭한다.

#### 3단계 - OpAMP 슈퍼바이저 설치하기 {#step-3---install-the-opamp-supervisor}

`opampsupervisor` 바이너리는 오픈텔레메트리 컬렉터의 [`cmd/opampsupervisor`
태그가 붙은 릴리스][tags]에서 다운로드 가능한 애셋으로 제공된다. OS와 칩셋을
기준으로 이름이 지정된 애셋 목록을 확인할 수 있으므로, 자신의 환경에 맞는 것을
다운로드한다.

{{< tabpane text=true >}}

{{% tab "Linux (AMD 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o opampsupervisor \
"https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_linux_amd64"
chmod +x opampsupervisor
```

{{% /tab %}} {{% tab "Linux (ARM 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o opampsupervisor \
"https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_linux_arm64"
chmod +x opampsupervisor
```

{{% /tab %}} {{% tab "Linux (ppc64le) "%}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o opampsupervisor \
"https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_linux_ppc64le"
chmod +x opampsupervisor
```

{{% /tab %}} {{% tab "macOS (AMD 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o opampsupervisor \
"https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_darwin_amd64"
chmod +x opampsupervisor
```

{{% /tab %}} {{% tab "macOS (ARM 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o opampsupervisor \
"https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_darwin_arm64"
chmod +x opampsupervisor
```

{{% /tab %}} {{% tab "Windows (AMD 64)" %}}

```sh
Invoke-WebRequest -Uri "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fopampsupervisor%2F{{% version-from-registry collector-cmd-opampsupervisor %}}/opampsupervisor_{{% version-from-registry collector-cmd-opampsupervisor noPrefix %}}_windows_amd64.exe" -OutFile "opampsupervisor.exe"
Unblock-File -Path "opampsupervisor.exe"
```

{{% /tab %}} {{< /tabpane >}}

#### 4단계 - OpAMP 슈퍼바이저 구성 파일 생성하기 {#step-4---create-an-opamp-supervisor-configuration-file}

다음 내용으로 `supervisor.yaml`이라는 파일을 생성한다.

```yaml
server:
  endpoint: wss://127.0.0.1:4320/v1/opamp
  tls:
    insecure_skip_verify: true

capabilities:
  accepts_remote_config: true
  reports_effective_config: true
  reports_own_metrics: false
  reports_own_logs: true
  reports_own_traces: false
  reports_health: true
  reports_remote_config: true

agent:
  executable: $OTEL_COLLECTOR_BINARY

storage:
  directory: ./storage
```

> [!NOTE]
>
> `$OTEL_COLLECTOR_BINARY`를 실제 파일 경로로 반드시 교체한다. 예를 들어 Linux나
> macOS에서 컬렉터를 `/usr/local/bin/`에 설치했다면, `$OTEL_COLLECTOR_BINARY`를
> `/usr/local/bin/otelcol`로 교체하면 된다.

#### 5단계 - OpAMP 슈퍼바이저 실행하기 {#step-5---run-the-opamp-supervisor}

이제 슈퍼바이저를 실행할 차례이며, 이 슈퍼바이저가 이어서 오픈텔레메트리
컬렉터를 실행한다.

```console
$ ./opampsupervisor --config=./supervisor.yaml
{"level":"info","ts":1745154644.746028,"logger":"supervisor","caller":"supervisor/supervisor.go:340","msg":"Supervisor starting","id":"01965352-9958-72da-905c-e40329c32c64"}
{"level":"info","ts":1745154644.74608,"logger":"supervisor","caller":"supervisor/supervisor.go:1086","msg":"No last received remote config found"}
```

모든 것이 정상적으로 동작했다면, 이제
[http://localhost:4321/](http://localhost:4321/)로 이동해 OpAMP 서버 UI에 접속할
수 있다. 슈퍼바이저가 관리하는 에이전트 목록에 컬렉터가 표시되는 것을 확인할 수
있다.

![OpAMP 예제 설정](../img/opamp-server-ui.png)

#### 6단계 - 오픈텔레메트리 컬렉터 원격 구성하기 {#step-6---configure-the-opentelemetry-collector-remotely}

서버 UI에서 컬렉터를 클릭하고, 다음 내용을 `Additional Configuration` 상자에
붙여넣는다.

```yaml
receivers:
  host_metrics:
    collection_interval: 10s
    scrapers:
      cpu:

exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [host_metrics]
      exporters: [debug]
```

`Save and Send to Agent`를 클릭한다.

![OpAMP 추가 구성](../img/opamp-server-additional-config.png)

페이지를 새로고침하고 에이전트 상태가 `Up: true`로 표시되는지 확인한다.

![OpAMP 에이전트](../img/opamp-server-agent.png)

내보내진 메트릭을 컬렉터에 질의할 수 있다(레이블 값에 유의한다).

```console
$ curl localhost:8888/metrics
# HELP otelcol_exporter_send_failed_metric_points Number of metric points in failed attempts to send to destination. [alpha]
# TYPE otelcol_exporter_send_failed_metric_points counter
otelcol_exporter_send_failed_metric_points{exporter="debug",service_instance_id="01965352-9958-72da-905c-e40329c32c64",service_name="otelcol-contrib",service_version="0.124.1"} 0
# HELP otelcol_exporter_sent_metric_points Number of metric points successfully sent to destination. [alpha]
# TYPE otelcol_exporter_sent_metric_points counter
otelcol_exporter_sent_metric_points{exporter="debug",service_instance_id="01965352-9958-72da-905c-e40329c32c64",service_name="otelcol-contrib",service_version="0.124.1"} 132
# HELP otelcol_process_cpu_seconds Total CPU user and system time in seconds [alpha]
# TYPE otelcol_process_cpu_seconds counter
otelcol_process_cpu_seconds{service_instance_id="01965352-9958-72da-905c-e40329c32c64",service_name="otelcol-contrib",service_version="0.124.1"} 0.127965
...
```

컬렉터의 로그도 확인할 수 있다.

```console
$ cat ./storage/agent.log
{"level":"info","ts":"2025-04-20T15:11:12.996+0200","caller":"service@v0.124.0/service.go:199","msg":"Setting up own telemetry..."}
{"level":"info","ts":"2025-04-20T15:11:12.996+0200","caller":"builders/builders.go:26","msg":"Development component. May change in the future."}
{"level":"info","ts":"2025-04-20T15:11:12.997+0200","caller":"service@v0.124.0/service.go:266","msg":"Starting otelcol-contrib...","Version":"0.124.1","NumCPU":11}
{"level":"info","ts":"2025-04-20T15:11:12.997+0200","caller":"extensions/extensions.go:41","msg":"Starting extensions..."}
{"level":"info","ts":"2025-04-20T15:11:12.997+0200","caller":"extensions/extensions.go:45","msg":"Extension is starting..."}
{"level":"info","ts":"2025-04-20T15:11:13.022+0200","caller":"extensions/extensions.go:62","msg":"Extension started."}
{"level":"info","ts":"2025-04-20T15:11:13.022+0200","caller":"extensions/extensions.go:45","msg":"Extension is starting..."}
{"level":"info","ts":"2025-04-20T15:11:13.022+0200","caller":"healthcheckextension@v0.124.1/healthcheckextension.go:32","msg":"Starting health_check extension","config":{"Endpoint":"localhost:58760","TLSSetting":null,"CORS":null,"Auth":null,"MaxRequestBodySize":0,"IncludeMetadata":false,"ResponseHeaders":null,"CompressionAlgorithms":null,"ReadTimeout":0,"ReadHeaderTimeout":0,"WriteTimeout":0,"IdleTimeout":0,"Path":"/","ResponseBody":null,"CheckCollectorPipeline":{"Enabled":false,"Interval":"5m","ExporterFailureThreshold":5}}}
{"level":"info","ts":"2025-04-20T15:11:13.022+0200","caller":"extensions/extensions.go:62","msg":"Extension started."}
{"level":"info","ts":"2025-04-20T15:11:13.024+0200","caller":"healthcheck/handler.go:132","msg":"Health Check state change","status":"ready"}
{"level":"info","ts":"2025-04-20T15:11:13.024+0200","caller":"service@v0.124.0/service.go:289","msg":"Everything is ready. Begin running and processing data."}
{"level":"info","ts":"2025-04-20T15:11:14.025+0200","msg":"Metrics","resource metrics":1,"metrics":1,"data points":44}
```

## 기타 정보 {#other-information}

- 블로그 게시물:
  - [Open Agent Management Protocol (OpAMP) State of the Nation
    2023][blog-opamp-status]
  - [Using OpenTelemetry OpAMP to modify service telemetry on the
    go][blog-opamp-service-telemetry]
- YouTube 동영상:
  - [Smooth Scaling With the OpAMP Supervisor: Managing Thousands of
    OpenTelemetry Collectors][video-opamp-smooth-scaling]
  - [Remote Control for Observability Using the Open Agent Management
    Protocol][video-opamp-remote-control]
  - [What is OpAMP & What is BindPlane][video-opamp-bindplane]
  - [Lightning Talk: Managing OpenTelemetry Through the OpAMP
    Protocol][video-opamp-lt]

[configuration]: /docs/collector/configuration/
[Open Agent Management Protocol]: https://github.com/open-telemetry/opamp-spec
[OpAMP specification]: /docs/specs/opamp/
[opamp-in-otel-collector]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/cmd/opampsupervisor/specification/README.md
[opamp-go]: https://github.com/open-telemetry/opamp-go
[opamp-supervisor]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/opampsupervisor
[otelcolcontrib]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases
[tags]: https://github.com/open-telemetry/opentelemetry-collector-releases/tags
[blog-opamp-status]: /blog/2023/opamp-status/
[blog-opamp-service-telemetry]: /blog/2022/opamp/
[video-opamp-smooth-scaling]: https://www.youtube.com/watch?v=g8rtqqNTL9Q
[video-opamp-remote-control]: https://www.youtube.com/watch?v=t550FzDi054
[video-opamp-bindplane]: https://www.youtube.com/watch?v=N18z2dOJSd8
[video-opamp-lt]: https://www.youtube.com/watch?v=LUsfZFRM4yo
