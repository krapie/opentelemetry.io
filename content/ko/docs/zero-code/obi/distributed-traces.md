---
title: OBI를 사용한 분산 트레이스
linkTitle: 분산 트레이스
description: OBI의 분산 트레이스 지원에 대해 알아본다.
weight: 22
cSpell:ignore: asyncio chanrecv chansend HPACK uvloop
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

## 소개 {#introduction}

OBI는 일부 제한 사항과 커널 버전 제약이 있지만 애플리케이션에 대한 분산
트레이스를 지원한다.

분산 트레이싱(distributed tracing)은
[W3C `traceparent`](https://www.w3.org/TR/trace-context/) 헤더 값의 전파를 통해
구현된다. OBI는 들어오는 컨텍스트를 자동으로 읽는다. 나가는 네트워크 수준
컨텍스트 전파는 기본적으로 비활성화되어 있으며 아래 설명대로 활성화해야 한다.

해당하는 전파 모드가 활성화되면, OBI는 들어오는 트레이스 컨텍스트를 읽고,
프로그램 실행을 추적하며, 나가는 HTTP 또는 gRPC 요청에 `traceparent`를 추가한다.
애플리케이션이 이미 `traceparent`를 추가했다면, OBI는 자체 생성한 컨텍스트 대신
그 값을 사용한다. OBI가 들어오는 컨텍스트를 찾을 수 없으면, W3C 명세에 따라
컨텍스트를 생성한다.

작업이 스레드, 고루틴, 태스크, 이벤트 루프를 넘나들 때 OBI가 부모 요청을
선택하는 방법에 대한 자세한 내용은
[트레이스 컨텍스트 연관](../context-propagation/)을 참고한다.

## 호환성 {#compatibility}

OBI는 다음 구성에서 분산 트레이싱과 컨텍스트 전파를 지원한다.

| 영역                             | 지원 버전 또는 환경                                                              | 참고                                                                                                                                          |
| :------------------------------- | :------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| 네트워크 수준 HTTP/1 전파        | [OBI 호환성 요구 사항](/docs/zero-code/obi/#compatibility)을 충족하는 Linux 환경 | 여러 프로그래밍 언어에서 동작한다. HTTPS의 경우, 전파는 OBI로 계측된 다른 서비스로 제한되며 프록시나 L7 로드 밸런서에 의해 중단될 수 있다.    |
| 네트워크 수준 gRPC 전파          | HTTP/2를 통한 gRPC `1.0+`                                                        | 여러 언어에서 스트림별 HPACK `traceparent` 헤더를 사용한다. OBI가 시작되기 전에 설정된 Go가 아닌 지속 연결은 인식되지 않을 수 있다.           |
| Go 라이브러리 수준 컨텍스트 전파 | Go `1.18+`                                                                       | 최대 6단계 중첩 고루틴까지 고루틴 컨텍스트 전파를 지원한다. 이 분산 트레이싱 기능은 일반적인 Go 라이브러리 수준 계측보다 최소 버전이 더 높다. |
| Node.js async hooks              | Node.js `8.0+`                                                                   | `SIGUSR1`에 대한 커스텀 처리가 컨텍스트 전파를 방해할 수 있다.                                                                                |
| Ruby Puma                        | Puma `5.0+`로 서비스되는 Ruby 애플리케이션                                       | 컨텍스트 전파 지원에는 Puma 서버가 필요하다.                                                                                                  |
| Java 스레드 풀                   | JDK `8+`                                                                         | 문서화된 추가 런타임 제약이 없다.                                                                                                             |
| Python asyncio                   | `uvloop`을 사용하는 Python `3.9+`                                                | 컨텍스트 전파 지원에는 `uvloop` 이벤트 루프가 필요하다.                                                                                       |

여기에 나열된 버전은 OBI가 분산 트레이싱 기능에 대해 명시적으로 지원하는
버전이다. 다른 버전도 동작할 수 있지만, 별도로 명시되지 않는 한 문서화된 지원
범위에 포함되지 않는다. 특히 여기서의 Go `1.18+` 요구 사항은 분산 트레이싱과
컨텍스트 전파에 적용된다. 다른 OBI Go 라이브러리 수준 계측은 최소 버전이 더
낮다.

## 구현 {#implementation}

트레이스 컨텍스트 전파는 두 가지 서로 다른 방식으로 구현된다.

1. 네트워크 수준에서 나가는 헤더 정보를 기록하는 방식
2. Go의 경우 라이브러리 수준에서 헤더 정보를 기록하는 방식

서비스가 작성된 프로그래밍 언어에 따라, OBI는 컨텍스트 전파의 한 가지 또는 두
가지 접근 방식을 모두 사용한다. eBPF로 메모리를 기록하는 것은 커널 구성과 OBI에
부여된 Linux 시스템 capabilities에 따라 달라지므로, 컨텍스트 전파를 구현하기
위해 이러한 여러 접근 방식을 사용한다. 이 주제에 대한 자세한 내용은 KubeCon NA
2024 발표
[So You Want to Write Memory with eBPF?](https://www.youtube.com/watch?v=TUiVX-44S9s)를
참고한다.

**네트워크 수준**의 컨텍스트 전파는 기본적으로 **비활성화**되어 있으며, 환경
변수 `OTEL_EBPF_BPF_CONTEXT_PROPAGATION=all`을 설정하거나 OBI 구성 파일을
수정하여 활성화할 수 있다.

```yaml
ebpf:
  context_propagation: 'all'
```

### 네트워크 수준 컨텍스트 전파 {#context-propagation-at-network-level}

네트워크 수준 컨텍스트 전파는 트레이스 컨텍스트 정보를 나가는 HTTP 헤더뿐 아니라
TCP/IP 패킷 수준에도 기록하여 구현된다. HTTP 컨텍스트 전파는
오픈텔레메트리(OpenTelemetry) 기반의 다른 모든 트레이싱 라이브러리와 완전히
호환된다. 즉, OBI로 계측된 서비스는 오픈텔레메트리 SDK로 계측된 서비스와
주고받을 때 트레이스 정보를 올바르게 전파한다. 네트워크 패킷 조정을 수행하기
위해
[Linux 트래픽 제어(Traffic Control, TC)](<https://en.wikipedia.org/wiki/Tc_(Linux)>)를
사용하며, 이를 위해서는 Linux 트래픽 제어를 사용하는 다른 eBPF 프로그램이 OBI와
올바르게 체이닝되어야 한다. Cilium CNI에 대한 특별한 고려 사항은
[Cilium 호환성](../cilium-compatibility/) 가이드를 참고한다.

TLS로 암호화된 트래픽(HTTPS)의 경우, OBI는 나가는 HTTP 헤더에 트레이스 정보를
주입할 수 없으며 대신 TCP/IP 패킷 수준에 정보를 주입한다. 이 제한으로 인해 OBI는
OBI로 계측된 다른 서비스에만 트레이스 정보를 보낼 수 있다. L7 프록시와 로드
밸런서는 원본 패킷을 폐기하고 다운스트림으로 재전송하므로 TCP/IP 컨텍스트 전파를
중단시킨다. 오픈텔레메트리 SDK로 계측된 서비스로부터 들어오는 트레이스 컨텍스트
정보를 파싱하는 것은 여전히 동작한다.

gRPC의 경우, OBI는 스트림별 `traceparent` HPACK 헤더를 주입한다. 이는 여러
프로그래밍 언어에서 동작하며 동시 HTTP/2 스트림에 대해 별개의 트레이스
컨텍스트를 보존한다. TCP 옵션은 연결 범위(connection-scoped)이며 여러 다중화된
스트림을 표현할 수 없으므로, OBI는 gRPC에 TCP 옵션을 사용하지 않는다. gRPC가
아닌 일반 HTTP/2 컨텍스트 전파는 여전히 Go 라이브러리 계측으로 제한된다.

더 세밀한 제어가 필요하다면, `context_propagation`은 `headers`, `tcp`,
`headers,tcp`도 허용한다. 이전의 `http` 별칭은 제거되었다. 더 이상 사용되지 않는
`ip` 값은 효과가 없다.

이 유형의 컨텍스트 전파는 모든 프로그래밍 언어에서 동작하며, OBI가 `privileged`
모드로 실행되거나 `CAP_SYS_ADMIN`이 부여될 것을 요구하지 않는다. 자세한 내용은
[분산 트레이스와 컨텍스트 전파](../configure/metrics-traces-attributes/) 구성
섹션을 참고한다.

#### 쿠버네티스 구성 {#kubernetes-configuration}

네트워크 수준 분산 트레이싱 지원과 함께 쿠버네티스에 OBI를 배포하는 권장 방법은
`DaemonSet`으로 배포하는 것이다.

다음 `Kubernetes` 구성을 사용해야 한다.

- OBI는 호스트 네트워크 액세스(`hostNetwork: true`)와 함께 `DaemonSet`으로
  배포해야 한다.
- 호스트의 `/sys/fs/cgroup` 경로를 로컬 `/sys/fs/cgroup` 경로로 볼륨 마운트해야
  한다.
- OBI 컨테이너에 `CAP_NET_ADMIN` capability를 부여해야 한다.

다음 YAML 스니펫은 OBI 배포 구성 예시를 보여준다.

```yaml
spec:
  serviceAccount: obi
  hostPID: true # <-- Important. Required in DaemonSet mode so OBI can discover all monitored processes
  hostNetwork: true # <-- Important. Required in DaemonSet mode so OBI can see all network packets
  dnsPolicy: ClusterFirstWithHostNet
  containers:
    - name: obi
      resources:
        limits:
          memory: 120Mi
      terminationMessagePolicy: FallbackToLogsOnError
      image: 'docker.io/otel/ebpf-instrument:main'
      imagePullPolicy: 'Always'
      env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: 'http://otelcol:4318'
        - name: OTEL_EBPF_KUBE_METADATA_ENABLE
          value: 'autodetect'
        - name: OTEL_EBPF_CONFIG_PATH
          value: '/config/obi-config.yml'
      securityContext:
        runAsUser: 0
        readOnlyRootFilesystem: true
        capabilities:
          add:
            - BPF # <-- Important. Required for most eBPF probes to function correctly.
            - SYS_PTRACE # <-- Important. Allows OBI to access the container namespaces and inspect executables.
            - NET_RAW # <-- Important. Allows OBI to use socket filters for http requests.
            - CHECKPOINT_RESTORE # <-- Important. Allows OBI to open ELF files.
            - DAC_READ_SEARCH # <-- Important. Allows OBI to open ELF files.
            - PERFMON # <-- Important. Allows OBI to load BPF programs.
            - NET_ADMIN # <-- Important. Allows OBI to inject HTTP and TCP context propagation information.
      volumeMounts:
        - name: cgroup
          mountPath: /sys/fs/cgroup # <-- Important. Allows OBI to monitor all newly sockets to track outgoing requests.
        - mountPath: /config
          name: obi-config
  tolerations:
    - effect: NoSchedule
      operator: Exists
    - effect: NoExecute
      operator: Exists
  volumes:
    - name: obi-config
      configMap:
        name: obi-config
    - name: cgroup
      hostPath:
        path: /sys/fs/cgroup
```

OBI `DaemonSet`에 `/sys/fs/cgroup`이 로컬 볼륨 경로로 마운트되지 않으면 일부
요청은 컨텍스트가 전파되지 않을 수 있다. 이 볼륨 경로는 새로 생성된 소켓을
수신하는 데 사용된다.

#### 커널 버전 제한 사항 {#kernel-version-limitations}

네트워크 수준 컨텍스트 전파의 들어오는 헤더 파싱은 BPF 루프의 추가 및 사용을
위해 일반적으로 커널 5.17 이상이 필요하다.

RHEL 9.2 같은 일부 패치된 커널은 이 기능이 백포트되어 있을 수 있다. 커널이 이
기능을 포함하지만 5.17보다 낮은 경우, `OTEL_EBPF_OVERRIDE_BPF_LOOP_ENABLED`를
설정하면 커널 검사를 건너뛴다.

### 라이브러리 수준 계측을 통한 Go 컨텍스트 전파 {#go-context-propagation-by-instrumenting-at-library-level}

이 유형의 컨텍스트 전파는 Go 애플리케이션에서만 지원되며 eBPF 사용자 메모리 쓰기
지원(`bpf_probe_write_user`)을 사용한다. 이 접근 방식의 장점은 HTTP와 HTTPS
모두에서 동작한다는 것이다. HTTP/2와 gRPC의 경우, HTTPS를 사용하지 않을 때 OBI는
새로운 HTTP/2 및 gRPC 연결과 재사용된 연결 모두에 컨텍스트를 주입할 수 있다.
`bpf_probe_write_user`를 사용하려면 OBI에 `CAP_SYS_ADMIN`을 부여하거나 권한 있는
컨테이너로 실행해야 한다.

#### Go Trace API를 사용하는 애플리케이션 계측 {#instrument-applications-that-use-the-go-trace-api}

OBI v0.11.0부터, OBI는 SDK를 등록하지 않고 오픈텔레메트리 Go Trace API를
사용하는 애플리케이션을 계측할 수 있다. 이 통합이 활성화되면, OBI는 Trace API
호출을 감지하고 그 결과로 생성된 수동 스팬을 자체 eBPF 스팬과 함께 내보낸다.
이미 오픈텔레메트리 SDK를 등록한 애플리케이션은 계속해서 자체 SDK 텔레메트리를
관리하고 내보낸다. `otel.SetTracerProvider(auto.TracerProvider())` 호출을
포함하여 전역 `TracerProvider`를 등록하면 이 자동 활성화가 방지된다.

OBI는 다음 조건이 모두 충족될 때만 이 통합을 활성화한다.

- 애플리케이션이 모듈 교체 없이 지원되는 오픈텔레메트리 모듈 버전과 체크섬
  조합을 사용한다.
- 실행 파일과 호스트가 지원되는 64비트 아키텍처를 사용한다.
- OBI가 필요한 심볼과 필드 레이아웃을 확인할 수 있다.
- OBI가 `bpf_probe_write_user`를 사용할 권한이 있다.

검사 중 하나라도 실패하면, Auto SDK는 비활성 상태로 유지되고 전역 Trace API를
통해 생성된 스팬은 기록되지 않는 상태로 유지된다. OBI의 eBPF 계측은 독립적으로
계속 동작한다. OBI가 Trace API 호출을 감지할 수 있으면, 스팬 이름, 부모 관계,
상태, 일부 기본 속성을 포함하는 부분적인 합성(synthetic) 스팬을 내보낼 수 있다.
이러한 스팬에는 계측 스코프, 이벤트, 요청된 스팬 종류가 포함되지 않는다.

OBI v0.11.0에서는, Auto SDK를 통해 내보내는 각 스팬의 인코딩된 페이로드가
16KiB를 초과해서는 안 된다. OBI는 통합을 활성화하거나 크기가 초과된 페이로드를
폐기할 때 메트릭이나 로그 메시지를 내보내지 않는다.

알려진 제한 사항과 후속 작업으로는
[헤드 샘플링](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/2793),
[컨텍스트 핸드오프](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/2794),
[외부 및 원격 부모와 `TraceState`](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/2959),
[더 큰 페이로드와 폐기 옵저버빌리티](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/2958),
[로그 보강](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/2932)이
있다.

지원되는 모듈 버전, 체크섬, 아키텍처 조합은
[활성화 자격 매트릭스](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/blob/v0.12.1/SUPPORT_MATRIX.md#go-global-trace-api-and-auto-sdk-activation)를
참고한다. 업스트림
[Go Trace API 예시](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/tree/v0.12.1/examples/go-trace-api)와
[Auto SDK](/docs/zero-code/go/autosdk) 문서도 살펴볼 수 있다.

#### 커널 무결성 모드 제한 사항 {#kernel-integrity-mode-limitations}

나가는 HTTP/gRPC 요청 헤더에 `traceparent` 값을 기록하려면, OBI는
[**bpf_probe_write_user**](https://www.man7.org/linux/man-pages/man7/bpf-helpers.7.html)
eBPF 헬퍼를 사용하여 프로세스 메모리에 기록해야 한다. 커널 5.14부터(5.10
시리즈로 백포트된 수정 포함) Linux 커널이 `integrity` **lockdown** 모드로 실행
중이면 이 헬퍼는 보호되어 BPF 프로그램에서 사용할 수 없다. 커널 무결성 모드는
커널에 [**Secure Boot**](https://wiki.debian.org/SecureBoot)가 활성화되어 있으면
일반적으로 기본적으로 활성화되지만, 수동으로 활성화할 수도 있다.

OBI는 `bpf_probe_write_user` 헬퍼를 사용할 수 있는지 자동으로 확인하고, 커널
구성에서 허용하는 경우에만 컨텍스트 전파를 활성화한다. 다음 명령을 실행하여
Linux 커널 **lockdown** 모드를 확인한다.

```shell
cat /sys/kernel/security/lockdown
```

해당 파일이 존재하고 모드가 `[none]` 이외의 값이면, OBI는 컨텍스트 전파를 수행할
수 없으며 분산 트레이싱이 비활성화된다.

#### 컨테이너화된 환경(쿠버네티스 포함)에서의 Go 분산 트레이싱 {#distributed-tracing-for-go-in-containerized-environments-including-kubernetes}

커널 **lockdown** 모드 제한으로 인해, Docker 및 쿠버네티스 구성 파일은 호스트
시스템에서 **OBI docker 컨테이너**를 위해 `/sys/kernel/security/` 볼륨을
마운트해야 한다. 이렇게 하면 OBI가 Linux 커널 **lockdown** 모드를 올바르게
확인할 수 있다. 다음은 OBI가 **lockdown** 모드를 확인하기에 충분한 정보를 갖도록
보장하는 Docker compose 구성 예시이다.

```yaml
services:
  ...
  obi:
    image: 'docker.io/otel/ebpf-instrument:main'
    environment:
      OTEL_EBPF_CONFIG_PATH: "/configs/obi-config.yml"
    volumes:
      - /sys/kernel/security:/sys/kernel/security
      - /sys/fs/cgroup:/sys/fs/cgroup
```

`/sys/kernel/security/` 볼륨이 마운트되지 않으면, OBI는 Linux 커널이 무결성
모드로 실행되고 있지 않다고 가정한다.

### Go 채널 스팬 링크 {#go-channel-span-links}

OBI는 Go 채널을 통한 지원되는 작업 핸드오프에 대해 실험적인 리시버 측 스팬
링크를 내보낸다. 송신 측과 수신 측 모두 활성 상태인 OBI 생성 스팬을 가지고
있으면, 리시버 스팬이 센더 스팬에 링크된다. OBI는 트레이스 ID, 부모-자식 관계,
센더 스팬을 변경하지 않는다.

이 동작은 OBI가 대상 바이너리의 채널 런타임 오프셋을 확인할 수 있을 때 Go 전용
트레이싱과 함께 자동으로 활성화된다. `runtime.chansend1`, `runtime.chanrecv1`,
`runtime.chanrecv2`를 통한 직접적인 버퍼링 없는 핸드오프와 버퍼링된 핸드오프가
지원된다. `select`를 통한 채널 작업은 지원되지 않는다. 이러한 프로브를
비활성화하려면 Go 전용 트레이서를 비활성화한다. 별도의 채널 링크 옵션은 없다.

OBI는 `OTEL_SPAN_LINK_COUNT_LIMIT`를 준수하며 유효하지 않거나 중복되거나 자기
참조하는 링크를 폐기한다.

### Node.js 수동 스팬 캡처 {#capture-nodejs-manual-spans}

OBI v0.12.1부터, OBI는 애플리케이션이 오픈텔레메트리 SDK를 등록하지 않은 경우
Node.js 애플리케이션이 `@opentelemetry/api`를 통해 생성하는 스팬을 캡처할 수
있다. OBI는 이러한 수동 스팬을 자체 트레이스 파이프라인을 통해 내보내고 자동으로
캡처된 서버 스팬과 상관 짓는다. 애플리케이션이 SDK를 등록하면, OBI는 스팬 생성과
내보내기를 해당 SDK에 맡긴다.

이 기능은 기본적으로 비활성화되어 있다. 기존 Config v1 배포에서는
`nodejs.manual_spans: true` 또는 `OTEL_EBPF_NODEJS_MANUAL_SPANS=true`로 활성화할
수 있다. Config v2는 v0.12.1에서 이에 상응하는 필드를 제공하지 않는다. 이
기능만을 위해 Config v1을 유지하기보다는 배포를 Config v2로 계속
마이그레이션한다.

OBI는 Node.js 인스펙터에 접근할 수 있어야 하며, 프로세스는 자체 `SIGUSR1`
핸들러를 등록하지 않아야 한다. CommonJS 로더가 접근할 수 없는 번들된
`@opentelemetry/api` 사본은 캡처되지 않는다. 현재 자동 클라이언트 스팬은 활성
수동 스팬의 자식이 아니라, 동일한 서버 스팬 아래에서 수동 스팬의 형제이다.

### uvloop을 사용하는 Python asyncio {#python-asyncio-with-uvloop}

v0.7.0부터, OBI는 [`uvloop`](https://github.com/MagicStack/uvloop)에서 실행되는
Python asyncio 워크로드에 대한 컨텍스트 전파를 지원한다. 이를 통해 표준
`asyncio` 지원에 더해 `uvloop` 이벤트 루프를 사용하는 비동기 Python 서비스의
분산 트레이싱이 가능해진다.

네트워크 수준 컨텍스트 전파는 `uvloop`에서 실행되는 Python 애플리케이션에
적용되어, OBI가 비동기 작업에 대한 트레이스 컨텍스트를 자동으로 계측하고 전파할
수 있게 한다. [소개](#introduction)에서 설명한 대로 컨텍스트 전파를 활성화하는
것 외에 추가 구성은 필요하지 않다.

OBI를 Python asyncio 및 `uvloop`과 함께 사용하려면, Python 애플리케이션이 이벤트
루프 구현으로 `uvloop`을 사용하도록 구성되어 있는지 확인한다.
