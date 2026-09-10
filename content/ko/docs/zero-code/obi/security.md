---
title: OBI 보안, 권한 및 capabilities
linkTitle: 보안
description: OBI에 필요한 권한 및 capabilities
weight: 22
cSpell:ignore: BPF_PROG_TYPE_KPROBE CAP_PERFMON eksctl
default_lang_commit: 348e946b973626a41f9ae316b93516c2c165b664
---

OBI는 애플리케이션을 계측하기 위해 `/proc` 파일시스템 읽기, eBPF 프로그램 로드,
네트워크 인터페이스 필터 관리 등 다양한 Linux 인터페이스에 접근해야 한다. 이러한
작업 중 다수는 상승된 권한을 필요로 한다. 가장 간단한 해결책은 OBI를 root로
실행하는 것이지만, 전체 root 접근이 이상적이지 않은 환경에서는 잘 맞지 않을 수
있다. 이를 해결하기 위해 OBI는 현재 구성에 필요한 특정 Linux 커널 capabilities만
사용하도록 설계되었다.

## Linux 커널 capabilities {#linux-kernel-capabilities}

Linux 커널 capabilities는 권한이 필요한 작업에 대한 접근을 세밀하게 제어하는
시스템이다. 전체 슈퍼유저나 root 접근을 부여하지 않고도 프로세스에 특정 권한을
부여할 수 있게 하며, 이는 최소 권한 원칙을 준수하여 보안을 개선하는 데 도움이
된다. capabilities는 일반적으로 root와 연관된 권한을 커널 내의 더 작은 권한
작업으로 나눈다.

capabilities는 프로세스와 실행 파일에 할당된다. `setcap` 같은 도구를 사용하여
관리자는 바이너리에 특정 capabilities를 할당할 수 있으며, 이를 통해 바이너리가
root로 실행하지 않고도 필요한 작업만 수행할 수 있다. 예를 들면 다음과 같다.

```shell
sudo setcap cap_net_admin,cap_net_raw+ep myprogram
```

이 예시는 `myprogram`에 `CAP_NET_ADMIN`과 `CAP_NET_RAW` capabilities를 부여하여,
전체 슈퍼유저 권한 없이도 네트워크 설정을 관리할 수 있게 한다.

capabilities를 신중하게 선택하고 할당하면 프로세스가 필요한 작업을 계속
수행하도록 하면서도 권한 상승의 위험을 낮출 수 있다.

자세한 내용은
[capabilities 매뉴얼 페이지](https://man7.org/linux/man-pages/man7/capabilities.7.html)에서
찾을 수 있다.

## OBI 동작 모드 {#obi-operation-modes}

OBI는 _애플리케이션 옵저버빌리티_ 와 _네트워크 옵저버빌리티_ 라는 두 가지 별개의
모드로 동작할 수 있다. 이 모드들은 상호 배타적이지 않으며 필요에 따라 함께
사용할 수 있다. 이 모드를 활성화하는 방법에 대한 자세한 내용은
[구성 문서](../configure/options/)를 참고한다.

OBI는 구성을 읽고 필요한 capabilities를 확인하며, 누락된 것이 있으면 경고를
표시한다. 예를 들면 다음과 같다.

```shell
time=2025-01-27T17:21:20.197-06:00 level=WARN msg="Required system capabilities not present, OBI may malfunction" error="the following capabilities are required: CAP_DAC_READ_SEARCH, CAP_BPF, CAP_CHECKPOINT_RESTORE"
```

그런 다음 OBI는 계속 실행을 시도하지만, 누락된 capabilities는 나중에 오류로
이어질 수 있다.

`OTEL_EBPF_ENFORCE_SYS_CAPS=1`을 설정할 수 있으며, 이 경우 필요한 capabilities를
사용할 수 없으면 OBI가 즉시 실패한다.

## OBI에 필요한 capabilities 목록 {#list-of-capabilities-required-by-obi}

OBI는 그 기능을 위해 다음 capabilities 목록을 필요로 한다.

| capability               | OBI에서의 용도                                                                                                                                                                                           |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CAP_BPF`                | 일반 BPF 기능과 소켓 필터(`BPF_PROG_TYPE_SOCK_FILTER`) 프로그램을 활성화하며, _네트워크 옵저버빌리티 모드_ 에서 네트워크 흐름을 캡처하는 데 사용된다.                                                    |
| `CAP_NET_RAW`            | `AF_PACKET` raw 소켓을 생성하는 데 사용되며, 이는 _네트워크 옵저버빌리티 모드_ 에서 네트워크 흐름을 캡처하는 소켓 필터 프로그램을 연결하는 메커니즘이다.                                                 |
| `CAP_NET_ADMIN`          | `BPF_PROG_TYPE_SCHED_CLS` TC 프로그램을 로드하는 데 필요하다. 이 프로그램은 네트워크 흐름 캡처와 트레이스 컨텍스트 전파에 사용되며, _네트워크 및 애플리케이션 옵저버빌리티_ 양쪽 모두에 해당한다.        |
| `CAP_PERFMON`            | 트레이스 컨텍스트 전파, 일반적인 _애플리케이션 옵저버빌리티_, 네트워크 흐름 모니터링에 사용된다. TC 프로그램의 직접 패킷 접근, 커널에 eBPF 프로브 로드, 이 프로그램들이 사용하는 포인터 연산을 허용한다. |
| `CAP_DAC_READ_SEARCH`    | 커널 버전을 확인하기 위한 `/proc/self/mem` 접근으로, OBI가 활성화할 적절한 지원 기능 집합을 결정하는 데 사용한다.                                                                                        |
| `CAP_CHECKPOINT_RESTORE` | `/proc` 파일시스템의 심링크 접근으로, OBI가 다양한 프로세스 및 시스템 정보를 얻는 데 사용한다.                                                                                                           |
| `CAP_SYS_PTRACE`         | `/proc/pid/exe`와 실행 가능한 모듈 접근으로, OBI가 실행 파일 심볼을 스캔하고 프로그램의 여러 부분을 계측하는 데 사용한다.                                                                                |
| `CAP_SYS_RESOURCE`       | 사용 가능한 잠긴 메모리 양을 늘린다. **5.11 미만 커널** 에서만 해당한다                                                                                                                                  |
| `CAP_SYS_ADMIN`          | `bpf_probe_write_user()`를 통한 라이브러리 수준 Go 트레이스 컨텍스트 전파와 BPF 메트릭 익스포터의 BTF 데이터 접근                                                                                        |

### 성능 모니터링 작업 {#performance-monitoring-tasks}

`CAP_PERFMON` 접근은 `kernel.perf_event_paranoid` 커널 설정이 관장하는
`perf_events` 접근 제어의 적용을 받으며, 이 설정은 `sysctl`을 통하거나
`/proc/sys/kernel/perf_event_paranoid` 파일을 수정하여 조정할 수 있다.
`kernel.perf_event_paranoid`의 기본 설정은 일반적으로 `2`이며, 이는
[커널 문서](https://www.kernel.org/doc/Documentation/sysctl/kernel.txt)의
`perf_event_paranoid` 섹션과, 더 포괄적으로는
[perf-security 문서](https://www.kernel.org/doc/Documentation/admin-guide/perf-security.rst)에
문서화되어 있다.

일부 Linux 배포판은 `kernel.perf_event_paranoid`에 더 높은 수준을 정의한다. 예를
들어 Debian 기반 배포판은 `kernel.perf_event_paranoid=3`을
[사용하기도 하며](https://lwn.net/Articles/696216/), 이 경우 `CAP_SYS_ADMIN`
없이는 `perf_event_open()` 접근이 허용되지 않는다. `kernel.perf_event_paranoid`
설정이 `2`보다 높은 배포판에서 실행 중이라면, 구성을 수정하여 `2`로 낮추거나
`CAP_PERFMON` 대신 `CAP_SYS_ADMIN`을 사용할 수 있다.

### AKS/EKS에 배포 {#deploy-on-aks-eks}

AKS와 EKS 환경 모두 기본적으로 `sys.perf_event_paranoid > 1`로 설정된 커널을
제공하며, 이는 OBI가 동작하려면 `CAP_SYS_ADMIN`이 필요하다는 것을 의미한다.
자세한 내용은 [작업 성능 모니터링](#performance-monitoring-tasks) 방법에 대한
섹션을 참고한다.

`CAP_PERFMON`만 사용하고 싶다면, `kernel.perf_event_paranoid = 1`로 설정하도록
노드를 구성할 수 있다. 이를 수행하는 몇 가지 예시를 제공하지만, 구체적인 환경에
따라 결과가 다를 수 있다는 점에 유의한다.

#### AKS {#aks}

##### AKS 구성 파일 생성 {#create-aks-configuration-file}

```json
{
  "sysctls": {
    "kernel.sys_paranoid": "1"
  }
}
```

##### AKS 클러스터 생성 또는 업데이트 {#create-or-update-your-aks-cluster}

```sh
az aks create --name myAKSCluster --resource-group myResourceGroup --linux-os-config ./linuxosconfig.json
```

자세한 내용은
"[Azure Kubernetes Service(AKS) 노드 풀의 노드 구성 커스터마이징](https://learn.microsoft.com/en-us/azure/aks/custom-node-configuration?tabs=linux-node-pools)"을
참고한다.

#### EKS(EKS Anywhere 구성 사용) {#eks-using-eks-anywhere-configuration}

##### EKS Anywhere 구성 파일 생성 {#create-eks-anywhere-configuration-file}

```yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereMachineConfig
metadata:
  name: machine-config
spec:
  hostOSConfiguration:
    kernel:
      sysctlSettings:
        kernel.sys_paranoid: '1'
```

##### EKS Anywhere 클러스터 배포 또는 업데이트 {#deploy-or-update-your-eks-anywhere-cluster}

```sh
eksctl create cluster --config-file hostosconfig.yaml
```

#### EKS(노드 그룹 설정 수정) {#eks-modifying-node-group-settings}

##### 노드 그룹 업데이트 {#update-the-node-group}

```yaml
apiVersion: eks.eks.amazonaws.com/v1beta1
kind: ClusterConfig
...
nodeGroups:
  - ...
    os: Bottlerocket
    eksconfig:
      ...
      sysctls:
        kernel.sys_paranoid: "1"
```

AWS Management Console, AWS CLI, 또는 `eksctl`을 사용하여 업데이트된 구성을 EKS
클러스터에 적용한다.

자세한 내용은
[EKS 호스트 OS 구성 문서](https://anywhere.eks.amazonaws.com/docs/getting-started/optional/hostosconfig/)를
참고한다.

## 예시 시나리오 {#example-scenarios}

다음 예시 시나리오는 OBI를 비-root 사용자로 실행하는 방법을 보여준다.

### 소켓 필터를 통한 네트워크 메트릭 {#network-metrics-via-a-socket-filter}

필요한 capabilities:

- `CAP_BPF`
- `CAP_NET_RAW`

필요한 capabilities를 설정하고 OBI를 시작한다.

```shell
sudo setcap cap_bpf,cap_net_raw+ep ./bin/obi
OTEL_EBPF_NETWORK_METRICS=1 OTEL_EBPF_NETWORK_PRINT_FLOWS=1 bin/obi
```

### 트래픽 제어를 통한 네트워크 메트릭 {#network-metrics-via-traffic-control}

필요한 capabilities:

- `CAP_BPF`
- `CAP_NET_ADMIN`
- `CAP_PERFMON`

필요한 capabilities를 설정하고 OBI를 시작한다.

```shell
sudo setcap cap_bpf,cap_net_admin,cap_perfmon+ep ./bin/obi
OTEL_EBPF_NETWORK_METRICS=1 OTEL_EBPF_NETWORK_PRINT_FLOWS=1 OTEL_EBPF_NETWORK_SOURCE=tc bin/obi
```

### 애플리케이션 옵저버빌리티 {#application-observability}

필요한 capabilities:

- `CAP_BPF`
- `CAP_DAC_READ_SEARCH`
- `CAP_CHECKPOINT_RESTORE`
- `CAP_PERFMON`
- `CAP_NET_RAW`
- `CAP_SYS_PTRACE`

필요한 capabilities를 설정하고 OBI를 시작한다.

```shell
sudo setcap cap_bpf,cap_dac_read_search,cap_perfmon,cap_net_raw,cap_sys_ptrace+ep ./bin/obi
OTEL_EBPF_OPEN_PORT=8080 OTEL_EBPF_TRACE_PRINTER=text bin/obi
```

### 트레이스 컨텍스트 전파를 포함한 애플리케이션 옵저버빌리티 {#application-observability-with-trace-context-propagation}

필요한 capabilities:

- `CAP_BPF`
- `CAP_DAC_READ_SEARCH`
- `CAP_CHECKPOINT_RESTORE`
- `CAP_PERFMON`
- `CAP_NET_RAW`
- `CAP_SYS_PTRACE`
- `CAP_NET_ADMIN`

필요한 capabilities를 설정하고 OBI를 시작한다.

```shell
sudo setcap cap_bpf,cap_dac_read_search,cap_perfmon,cap_net_raw,cap_sys_ptrace,cap_net_admin+ep ./bin/obi
OTEL_EBPF_CONTEXT_PROPAGATION=all OTEL_EBPF_OPEN_PORT=8080 OTEL_EBPF_TRACE_PRINTER=text bin/obi
```

## 내부 eBPF 트레이서 capability 요구 사항 참조 {#internal-ebpf-tracer-capability-requirement-reference}

OBI는 기반 기능을 구현하는 eBPF 프로그램 집합인 _트레이서(tracer)_ 를 사용한다.
트레이서는 여러 종류의 eBPF 프로그램을 로드하고 사용할 수 있으며, 각각 고유한
capabilities 집합을 필요로 한다.

아래 목록은 각 내부 트레이서를 필요한 capabilities에 매핑한 것으로, 개발자,
컨트리뷰터, 그리고 OBI의 내부 구조에 관심 있는 사람들을 위한 참조 자료로
제공된다.

**(네트워크 옵저버빌리티) 소켓 흐름 페처(fetcher):**

- `CAP_BPF`: `BPF_PROG_TYPE_SOCK_FILTER`용
- `CAP_NET_RAW`: `AF_PACKET` 소켓을 생성하고 소켓 필터를 네트워크 인터페이스에
  연결하기 위함

**(네트워크 옵저버빌리티) 흐름 페처(tc):**

- `CAP_BPF`
- `CAP_NET_ADMIN`: 네트워크 트래픽을 검사하는 데 사용되는 `PROG_TYPE_SCHED_CLS`
  eBPF TC 프로그램을 로드하기 위함
- `CAP_PERFMON`: `struct __sk_buff::data`를 통한 패킷 메모리 직접 접근과 eBPF
  프로그램의 포인터 연산 허용을 위함

**(애플리케이션 옵저버빌리티) 워처(watcher):**

- `CAP_BPF`
- `CAP_CHECKPOINT_RESTORE`
- `CAP_DAC_READ_SEARCH`: 커널 버전을 확인하기 위한 `/proc/self/mem` 접근용
- `CAP_PERFMON`: 포인터 연산이 필요한 `BPF_PROG_TYPE_KPROBE` eBPF 프로그램을
  로드하기 위함

**(애플리케이션 옵저버빌리티) Go 이외 언어 지원:**

- `CAP_BPF`
- `CAP_DAC_READ_SEARCH`
- `CAP_CHECKPOINT_RESTORE`
- `CAP_PERFMON`
- `CAP_NET_RAW`: `obi_socket__http_filter`를 네트워크 인터페이스에 연결하는 데
  사용되는 `AF_PACKET` 소켓을 생성하기 위함
- `CAP_SYS_PTRACE`: `/proc/pid/exe`와 `/proc`의 다른 노드에 접근하기 위함

**(애플리케이션 및 네트워크 옵저버빌리티) TC 모드에서의 네트워크 모니터링 및
컨텍스트 전파:**

- `CAP_BPF`
- `CAP_DAC_READ_SEARCH`
- `CAP_PERFMON`
- `CAP_NET_ADMIN`: `BPF_PROG_TYPE_SCHED_CLS`, `BPF_PROG_TYPE_SOCK_OPS`,
  `BPF_PROG_TYPE_SK_MSG` 로드를 허용하며, 모두 트레이스 컨텍스트 전파와 네트워크
  모니터링에 사용된다

**(애플리케이션 옵저버빌리티) Go 트레이서:**

- `CAP_BPF`
- `CAP_DAC_READ_SEARCH`
- `CAP_CHECKPOINT_RESTORE`
- `CAP_PERFMON`
- `CAP_NET_RAW`: `obi_socket__http_filter`를 네트워크 인터페이스에 연결하는 데
  사용되는 `AF_PACKET` 소켓을 생성하기 위함
- `CAP_SYS_PTRACE`: `/proc/pid/exe`와 `/proc`의 다른 노드에 접근하기 위함
- `CAP_SYS_ADMIN`: 프로브 기반(`bpf_probe_write_user()`) 라이브러리 수준
  컨텍스트 전파를 위함
