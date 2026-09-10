---
title: 쿠버네티스에 OBI 배포하기
linkTitle: 쿠버네티스
description: 쿠버네티스에 OBI를 배포하는 방법을 알아본다.
weight: 4
# prettier-ignore
cSpell:ignore: cap_perfmon containerd goblog kubeadm microk8s replicaset statefulset
default_lang_commit: 07668da4a7868f007b58dc7677b5a7eed0baf529
---

> [!NOTE]
>
> 이 문서는 필요한 모든 엔터티를 직접 설정하여 쿠버네티스에 OBI를 수동으로
> 배포하는 방법을 설명한다.
>
> <!-- You might prefer to follow the
> [Deploy OBI in Kubernetes with Helm](../kubernetes-helm/) documentation instead. -->

## 쿠버네티스 메타데이터 데코레이션(decoration) 구성 {#configuring-kubernetes-metadata-decoration}

OBI는 다음 쿠버네티스 레이블로 트레이스를 데코레이션할 수 있다.

- `k8s.namespace.name`
- `k8s.deployment.name`
- `k8s.statefulset.name`
- `k8s.replicaset.name`
- `k8s.daemonset.name`
- `k8s.node.name`
- `k8s.pod.name`
- `k8s.container.name`
- `k8s.pod.uid`
- `k8s.pod.start_time`
- `k8s.cluster.name`

메타데이터 데코레이션을 활성화하려면 다음을 수행해야 한다.

- ServiceAccount를 생성하고, Pod와 ReplicaSet 양쪽 모두에 대한 list 및 watch
  권한을 부여하는 ClusterRole을 바인딩한다. 다음 예시 파일을 배포하여 이를
  수행할 수 있다.

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: obi
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: obi
  rules:
    - apiGroups: ['apps']
      resources: ['replicasets']
      verbs: ['list', 'watch']
    - apiGroups: ['']
      resources: ['pods', 'services', 'nodes']
      verbs: ['list', 'watch']
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: obi
  subjects:
    - kind: ServiceAccount
      name: obi
      namespace: default
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: obi
  ```

  (다른 네임스페이스에 OBI를 배포하는 경우 `namespace: default` 값을 변경해야
  한다).

- `OTEL_EBPF_KUBE_METADATA_ENABLE=true` 환경 변수 또는
  `attributes.kubernetes.enable: true` YAML 구성으로 OBI를 구성한다.

- OBI Pod에 `serviceAccountName: obi` 속성을 지정하는 것을 잊지 않는다(이후 배포
  예시에 나타나 있음).

선택적으로, YAML 구성 파일의 `discovery -> instrument` 섹션에서 계측할
쿠버네티스 서비스를 선택할 수 있다. 자세한 내용은
[구성 문서](../../configure/options/)의 _서비스 디스커버리_ 섹션과, 이 페이지의
[외부 구성 파일 제공하기](#providing-an-external-configuration-file) 섹션을
참고한다.

## OBI 배포하기 {#deploying-obi}

쿠버네티스에서 OBI는 다음 두 가지 방식으로 배포할 수 있다.

- 사이드카(sidecar) 컨테이너로 배포
- DaemonSet으로 배포

### 사이드카 컨테이너로 OBI 배포하기 {#deploy-obi-as-a-sidecar-container}

모든 호스트에 배포되어 있지 않을 수도 있는 특정 서비스를 모니터링하려는 경우 이
방식을 사용할 수 있으며, 이 경우 각 서비스 인스턴스마다 하나의 OBI 인스턴스만
배포하면 된다.

사이드카 컨테이너로 OBI를 배포하려면 다음과 같은 구성 요구 사항이 있다.

- Pod 내 모든 컨테이너 간에 프로세스 네임스페이스가 공유되어야 한다(Pod 변수
  `shareProcessNamespace: true`).
- 자동 계측 컨테이너는 특권(privileged) 모드로 실행되어야 한다(컨테이너 구성의
  `securityContext.privileged: true` 속성).
  - 일부 쿠버네티스 설치 환경에서는 다음과 같은 `securityContext` 구성을
    허용하지만, 일부 컨테이너를 제한하고 일부 권한을 제거하는 컨테이너 런타임
    구성도 있으므로 모든 환경에서 동작하지는 않을 수 있다.

    ```yaml
    securityContext:
      runAsUser: 0
      capabilities:
        add:
          - SYS_ADMIN
          - SYS_RESOURCE # not required for kernels 5.11+
    ```

#### Pod 내 모든 프로세스 계측하기(권장) {#instrumenting-all-processes-in-a-pod-recommended}

OBI가 `shareProcessNamespace: true`로 사이드카 컨테이너로 실행되면, Pod의 PID
네임스페이스를 공유하여 해당 Pod 내의 프로세스만 볼 수 있다. 즉,
`OTEL_EBPF_AUTO_TARGET_EXE=*`를 사용하여 개별 실행 파일 이름이나 포트를 지정할
필요 없이 Pod 내 모든 프로세스를 계측할 수 있다.

이는 사이드카 배포에 권장되는 방식이다. 별도의 수정 없이 여러 Pod에서 동일하게
재사용할 수 있는 간단한 구성을 제공하기 때문이다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goblog
  labels:
    app: goblog
spec:
  replicas: 2
  selector:
    matchLabels:
      app: goblog
  template:
    metadata:
      labels:
        app: goblog
    spec:
      # Required so the sidecar instrument tool can access the service process
      shareProcessNamespace: true
      serviceAccountName: obi # required if you want kubernetes metadata decoration
      containers:
        # Container for the instrumented service
        - name: goblog
          image: mariomac/goblog:dev
          imagePullPolicy: IfNotPresent
          command: ['/goblog']
          ports:
            - containerPort: 8443
              name: https
        # Sidecar container with OBI - the eBPF auto-instrumentation tool
        - name: obi
          image: otel/ebpf-instrument:main
          securityContext: # Privileges are required to install the eBPF probes
            privileged: true
          env:
            # Instrument all processes in the pod (wildcard)
            - name: OTEL_EBPF_AUTO_TARGET_EXE
              value: '*'
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: 'http://otelcol:4318'
            # required if you want kubernetes metadata decoration
            - name: OTEL_EBPF_KUBE_METADATA_ENABLE
              value: 'true'
```

와일드카드(wildcard) 방식은 개별 실행 파일 이름이나 포트를 지정하는 것보다
오류가 발생하기 쉽지 않다. Pod에 새 서비스를 추가할 때 OBI 구성을 업데이트할
필요가 없기 때문이다. OBI는 (PID 네임스페이스가 공유되어 있으므로) 동일한 Pod
내의 프로세스에만 접근할 수 있으므로, 실수로 Pod 밖의 프로세스를 계측할 위험이
없다.

#### Pod 내 특정 프로세스 계측하기 {#instrumenting-specific-processes-in-a-pod}

Pod 내에서 계측할 프로세스를 더 세분화해서 제어해야 한다면, 와일드카드 대신 실행
파일 이름이나 열린 포트를 지정할 수 있다.

다음 예시는 OBI를 컨테이너로 연결하여(이미지는 `otel/ebpf-instrument:main`에서
사용 가능) `goblog` Pod를 계측한다. 자동 계측 도구는 동일한 네임스페이스 내의
`otelcol` 서비스 뒤에서 접근 가능한 오픈텔레메트리 컬렉터로 메트릭과 트레이스를
전달하도록 구성되어 있다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goblog
  labels:
    app: goblog
spec:
  replicas: 2
  selector:
    matchLabels:
      app: goblog
  template:
    metadata:
      labels:
        app: goblog
    spec:
      # Required so the sidecar instrument tool can access the service process
      shareProcessNamespace: true
      serviceAccountName: obi # required if you want kubernetes metadata decoration
      containers:
        # Container for the instrumented service
        - name: goblog
          image: mariomac/goblog:dev
          imagePullPolicy: IfNotPresent
          command: ['/goblog']
          ports:
            - containerPort: 8443
              name: https
        # Sidecar container with OBI - the eBPF auto-instrumentation tool
        - name: obi
          image: otel/ebpf-instrument:main
          securityContext: # Privileges are required to install the eBPF probes
            privileged: true
          env:
            # The internal port of the goblog application container
            - name: OTEL_EBPF_OPEN_PORT
              value: '8443'
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: 'http://otelcol:4318'
              # required if you want kubernetes metadata decoration
            - name: OTEL_EBPF_KUBE_METADATA_ENABLE
              value: 'true'
```

다양한 구성 옵션에 대한 자세한 내용은 이 문서 사이트의
[구성](../../configure/options/) 섹션을 확인한다.

### DaemonSet으로 OBI 배포하기 {#deploy-obi-as-a-daemonset}

OBI를 DaemonSet으로 배포할 수도 있다. 다음과 같은 경우 이 방식이 선호된다.

- DaemonSet을 계측하려는 경우
- 하나의 OBI 인스턴스로 여러 프로세스, 또는 클러스터 내 모든 프로세스를
  계측하려는 경우

앞서 사용한 예시(`goblog` Pod)에서는 포트가 Pod 내부에 속하므로 열린 포트를
사용하여 계측할 프로세스를 선택할 수 없다. 동시에 서비스의 여러 인스턴스는 서로
다른 열린 포트를 가지게 된다. 이 경우 애플리케이션 서비스의 실행 파일 이름을
사용하여 계측해야 한다(다음 예시 참고).

사이드카 시나리오의 권한 요구 사항에 더해, 동일한 호스트에서 실행 중인 모든
프로세스에 접근할 수 있도록 `hostPID: true` 옵션을 활성화한 자동 계측 Pod
템플릿을 구성해야 한다.

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: obi
  labels:
    app: obi
spec:
  selector:
    matchLabels:
      app: obi
  template:
    metadata:
      labels:
        app: obi
    spec:
      hostPID: true # Required to access the processes on the host
      serviceAccountName: obi # required if you want kubernetes metadata decoration
      containers:
        - name: autoinstrument
          image: otel/ebpf-instrument:main
          securityContext:
            privileged: true
          env:
            # Select the executable by its name instead of OTEL_EBPF_OPEN_PORT
            - name: OTEL_EBPF_AUTO_TARGET_EXE
              value: '*/goblog'
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: 'http://otelcol:4318'
              # required if you want kubernetes metadata decoration
            - name: OTEL_EBPF_KUBE_METADATA_ENABLE
              value: 'true'
```

### OBI를 비특권(unprivileged) 모드로 배포하기 {#deploy-obi-unprivileged}

지금까지의 모든 예시에서는 OBI 배포의 `securityContext` 섹션에 `privileged:true`
또는 `SYS_ADMIN` Linux capability를 사용했다. 이 방식은 모든 환경에서
동작하지만, 보안 구성상 필요하다면 더 제한된 권한으로 쿠버네티스에 OBI를
배포하는 방법도 있다. 가능 여부는 사용 중인 쿠버네티스 버전과 기반이 되는
컨테이너 런타임(예: **Containerd**, **CRI-O**, **Docker**)에 따라 달라진다.

다음 가이드는 주로 `GKE`, `kubeadm`, `k3s`, `microk8s`, `kind`에서
`containerd`를 실행하며 수행한 테스트를 기반으로 한다.

OBI를 비특권으로 실행하려면 `privileged:true` 설정을 Linux
[capabilities](https://www.man7.org/linux/man-pages/man7/capabilities.7.html)
집합으로 대체해야 한다. OBI에 필요한 capability 전체 목록은
[보안, 권한 및 capabilities](../../security/)에서 확인할 수 있다.

**참고** BPF 프로그램을 로드하려면 OBI가 Linux 성능 이벤트를 읽을 수 있어야
하며, 최소한 Linux 커널 API `perf_event_open()`을 실행할 수 있어야 한다.

이 권한은 `CAP_PERFMON`에 의해, 또는 더 폭넓게는 `CAP_SYS_ADMIN`에 의해
부여된다. `CAP_PERFMON`과 `CAP_SYS_ADMIN` 모두 OBI에 성능 이벤트를 읽을 권한을
부여하므로, 더 적은 권한을 부여하는 `CAP_PERFMON`을 사용해야 한다. 하지만 시스템
수준에서 성능 이벤트에 대한 접근은 `kernel.perf_event_paranoid` 설정으로
제어되며, 이 값은 `sysctl`을 사용하거나 `/proc/sys/kernel/perf_event_paranoid`
파일을 수정하여 읽거나 쓸 수 있다. `kernel.perf_event_paranoid`의 기본 설정값은
일반적으로 `2`이며, 이는
[커널 문서](https://www.kernel.org/doc/Documentation/sysctl/kernel.txt)의
`perf_event_paranoid` 섹션에 문서화되어 있다. 일부 Linux 배포판은
`kernel.perf_event_paranoid`에 대해 더 높은 수준을 정의한다. 예를 들어 Debian
계열 배포판은
[`kernel.perf_event_paranoid=3`을 함께 사용하기도 하는데](https://lwn.net/Articles/696216/),
이 경우 `CAP_SYS_ADMIN` 없이는 `perf_event_open()`에 대한 접근이 허용되지
않는다. `kernel.perf_event_paranoid` 설정이 `2`보다 높은 배포판을 사용 중이라면,
구성을 수정하여 `2`로 낮추거나 `CAP_PERFMON` 대신 `CAP_SYS_ADMIN`을 사용할 수
있다.

비특권 OBI 컨테이너 구성 예시는 다음과 같다.

```yaml
...
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: obi
  namespace: obi-demo
  labels:
    k8s-app: obi
spec:
  selector:
    matchLabels:
      k8s-app: obi
  template:
    metadata:
      labels:
        k8s-app: obi
    spec:
      serviceAccount: obi
      hostPID: true           # <-- Important. Required in Daemonset mode so OBI can discover all monitored processes
      containers:
        - name: obi
          terminationMessagePolicy: FallbackToLogsOnError
          image: otel/ebpf-instrument:main
          env:
            - name: OTEL_EBPF_TRACE_PRINTER
              value: "text"
            - name: OTEL_EBPF_KUBE_METADATA_ENABLE
              value: "autodetect"
            - name: KUBE_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            ...
          securityContext:
            runAsUser: 0
            readOnlyRootFilesystem: true
            capabilities:
              add:
                - BPF                 # <-- Important. Required for most eBPF probes to function correctly.
                - SYS_PTRACE          # <-- Important. Allows OBI to access the container namespaces and inspect executables.
                - NET_RAW             # <-- Important. Allows OBI to use socket filters for http requests.
                - CHECKPOINT_RESTORE  # <-- Important. Allows OBI to open ELF files.
                - DAC_READ_SEARCH     # <-- Important. Allows OBI to open ELF files.
                - PERFMON             # <-- Important. Allows OBI to load BPF programs.
                #- SYS_RESOURCE       # <-- pre 5.11 only. Allows OBI to increase the amount of locked memory.
                #- SYS_ADMIN          # <-- Required for Go application trace context propagation, or if kernel.perf_event_paranoid >= 3 on Debian distributions.
              drop:
                - ALL
          volumeMounts:
            - name: var-run-obi
              mountPath: /var/run/obi
            - name: cgroup
              mountPath: /sys/fs/cgroup
      tolerations:
        - effect: NoSchedule
          operator: Exists
        - effect: NoExecute
          operator: Exists
      volumes:
        - name: var-run-obi
          emptyDir: { }
        - name: cgroup
          hostPath:
            path: /sys/fs/cgroup
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: some-service
  namespace: obi-demo
  ...
---
```

## k8s-cache로 쿠버네티스 메타데이터 중앙 집중화하기 {#centralizing-kubernetes-metadata-with-k8s-cache}

OBI가 DaemonSet으로 실행되면, 각 OBI Pod는 로컬 노드 메타데이터뿐 아니라
클러스터 전체의 메타데이터까지 메트릭과 트레이스를 데코레이션하는 데 필요한
정보를 가져오기 위해 쿠버네티스 API 서버에 각자의 `list` 및 `watch` 커넥션을
연다. 이는 로컬 노드 바깥의 정보를 보강하기 위한 것으로, 예를 들어 클러스터 내
노드 간 요청에
[피어(peer)](/docs/specs/semconv/registry/attributes/service/#service-attributes-for-peer-services)
속성을 스팬에 추가하는 데 사용된다. 대규모 클러스터에서는 이러한
팬아웃(fan-out)이 API 서버에 상당한 부하를 줄 수 있으며, 심한 경우 클러스터
전체에 영향을 미칠 수 있다.

이를 방지하기 위해 OBI는 `k8s-cache`라는 선택적 보조 서비스(companion service)를
제공한다. 이 서비스는 작은 Deployment로 실행되며, 모든 OBI Pod를 대신하여
쿠버네티스 API를 한 번만 관찰(watch)하고 gRPC를 통해 OBI 인스턴스로 메타데이터를
스트리밍한다. 이를 통해 OBI의 Pod별 인포머 트래픽이 API 서버로 가는 것을
제거하여 API 부하를 크게 줄이지만, OBI는 여전히 노드 및 클러스터 메타데이터에
대해 제한적으로 쿠버네티스 API를 직접 조회할 수 있다.

`k8s-cache` 사용은 항상 권장되지만, 특히 다음과 같은 경우에 더욱 그렇다.

- 대규모 클러스터에서 OBI를 DaemonSet으로 실행하는 경우
- 동일한 클러스터에서 많은 OBI 레플리카(대규모 `Deployment`, 다수의 사이드카
  등)를 실행하는 경우
- 쿠버네티스 API 서버에 부하가 걸리거나 속도 제한(rate limit)이 걸려 있는 경우

캐시 주소를 구성하지 않으면, 각 OBI 인스턴스는 자체적인 프로세스 내(in-process)
로컬 인포머를 유지하며, 이는 소규모 클러스터에서는 문제가 없다.

`k8s-cache`는 쿠버네티스에서 OBI를 실행할 때만 관련이 있으며, 독립형(standalone)
또는 Docker 설정에서는 영향을 미치지 않는다.

캐시를 사용하려면, 캐시를 배포하고 `OTEL_EBPF_KUBE_META_CACHE_ADDRESS` 환경
변수(또는 YAML의 `attributes.kubernetes.meta_cache_address`)로 OBI가 캐시의
`Service` 주소를 가리키도록 한다. 가장 쉬운 방법은 OBI Helm 차트를 사용하는
것으로, `k8sCache.replicas`를 0이 아닌 값으로 설정하면 `Deployment`, `Service`,
그리고 OBI 연결 구성을 대신 설정해 준다.

수동으로 배포하려는 경우, 캐시는
`ghcr.io/open-telemetry/opentelemetry-ebpf-instrumentation/opentelemetry-ebpf-k8s-cache`
컨테이너 이미지로 게시되어 있다. 최소한의 매니페스트 예시는 다음과 같다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-cache
  template:
    metadata:
      labels:
        app: k8s-cache
    spec:
      serviceAccountName: obi # needs list/watch on pods, nodes, services
      containers:
        - name: k8s-cache
          image: ghcr.io/open-telemetry/opentelemetry-ebpf-instrumentation/opentelemetry-ebpf-k8s-cache:latest
          ports:
            - containerPort: 50055
              name: grpc
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-cache
spec:
  selector:
    app: k8s-cache
  ports:
    - port: 50055
      name: grpc
      protocol: TCP
```

그런 다음 DaemonSet에서 OBI가 이를 가리키도록 한다.

```yaml
env:
  - name: OTEL_EBPF_KUBE_METADATA_ENABLE
    value: 'true'
  - name: OTEL_EBPF_KUBE_META_CACHE_ADDRESS
    value: 'k8s-cache.default.svc:50055'
```

단일 레플리카로도 대개 충분하다. 고가용성을 위해서는 동일한 `Service` 뒤에서
여러 레플리카를 실행한다. 각 OBI Pod는 그중 하나에 연결되며, 장애 발생 시 다른
레플리카로 재연결한다.

## 외부 구성 파일 제공하기 {#providing-an-external-configuration-file}

앞선 예시에서는 환경 변수를 통해 OBI를 구성했다. 하지만 (이 사이트의
[구성](../../configure/options/) 섹션에 문서화된 대로) 외부 YAML 파일을 통해
구성할 수도 있다.

구성을 파일로 제공하는 권장 방법은 원하는 구성이 담긴 ConfigMap을 배포한 뒤 이를
OBI Pod에 마운트하고, `OTEL_EBPF_CONFIG_PATH` 환경 변수로 이를 참조하는 것이다.

OBI YAML 문서가 담긴 ConfigMap 예시:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: obi-config
data:
  obi-config.yml: |
    trace_printer: text
    otel_traces_export:
      endpoint: http://otelcol:4317
      sampler:
        name: parentbased_traceidratio
        arg: "0.01"
    routes:
      patterns:
        - /factorial/{num}
```

앞의 ConfigMap을 마운트하고 접근하는 OBI DaemonSet 구성 예시:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: obi
spec:
  selector:
    matchLabels:
      instrumentation: obi
  template:
    metadata:
      labels:
        instrumentation: obi
    spec:
      serviceAccountName: obi
      hostPID: true #important!
      containers:
        - name: obi
          image: otel/ebpf-instrument:main
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true
            readOnlyRootFilesystem: true
          # mount the previous ConfigMap as a folder
          volumeMounts:
            - mountPath: /config
              name: obi-config
            - mountPath: /var/run/obi
              name: var-run-obi
          env:
            # tell OBI where to find the configuration file
            - name: OTEL_EBPF_CONFIG_PATH
              value: '/config/obi-config.yml'
      volumes:
        - name: obi-config
          configMap:
            name: obi-config
        - name: var-run-obi
          emptyDir: {}
```

## 비밀 구성 제공하기 {#providing-secret-configuration}

앞의 예시는 일반 구성에는 적합하지만, 비밀번호나 API 키와 같은 비밀 정보를
전달하는 데 사용해서는 안 된다.

비밀 정보를 제공하는 권장 방법은 쿠버네티스 Secret을 배포하는 것이다. 예를 들어
다음 시크릿은 가상의 오픈텔레메트리 컬렉터 자격 증명을 담고 있다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: otelcol-secret
type: Opaque
stringData:
  headers: 'Authorization=Bearer Z2hwX0l4Y29QOWhr....ScQo='
```

그런 다음 시크릿 값을 환경 변수로 접근할 수 있다. 앞의 DaemonSet 예시를
기준으로, OBI 컨테이너에 다음 `env` 섹션을 추가하여 이를 구현할 수 있다.

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_HEADERS
    valueFrom:
      secretKeyRef:
        key: otelcol-secret
        name: headers
```
