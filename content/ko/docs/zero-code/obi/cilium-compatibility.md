---
title: OBI와 Cilium 호환성
linkTitle: Cilium 호환성
description: OBI를 Cilium과 함께 실행할 때의 호환성 참고 사항
weight: 23
default_lang_commit: 9d4b4c86dc28205a888a35a093b7e79cfd87b723
---

Cilium은 eBPF를 사용하여 쿠버네티스 클러스터에 네트워킹과 보안을 제공하는 오픈
소스 보안, 네트워킹, 옵저버빌리티(observability) 플랫폼이다. 경우에 따라
Cilium이 사용하는 eBPF 프로그램이 OBI가 사용하는 eBPF 프로그램과 충돌하여 문제를
일으킬 수 있다.

OBI와 Cilium은 eBPF 트래픽 제어(Traffic Control) 분류기 프로그램인
`BPF_PROG_TYPE_SCHED_CLS`를 사용한다. 이 프로그램은 커널 네트워킹 스택의
인그레스(ingress) 및 이그레스(egress) 데이터 경로에 연결(attach)된다. 이들은
함께 패킷이 네트워크 스택을 통과할 때 패킷을 검사하고 필요에 따라 수정할 수 있는
프로그램 체인을 형성한다.

OBI 프로그램은 패킷의 흐름을 절대 방해하지 않지만, Cilium은 동작의 일부로 패킷
흐름을 변경한다. Cilium이 OBI보다 먼저 패킷을 처리하면 OBI의 패킷 처리 능력에
영향을 줄 수 있다.

## 연결 우선순위 {#attachment-priority}

OBI는 트래픽 제어(TC) 프로그램을 연결하기 위해 Linux 커널의 Traffic Control
eXpress(TCX) API 또는 Netlink 인터페이스를 사용한다.

TCX는 프로그램을 목록의 앞(head), 중간(middle), 뒤(tail)에 연결할 수 있는 새로운
API이다. OBI와 Cilium은 커널이 TCX를 지원하는지 자동으로 감지하고 기본적으로
이를 사용한다.

OBI와 Cilium이 TCX를 사용하면 서로 간섭하지 않는다. OBI는 eBPF 프로그램을 목록의
앞에 연결하고 Cilium은 뒤에 연결한다. 가능한 경우 TCX가 선호되는 동작 모드이다.

## Netlink로 폴백 {#fallback-to-netlink}

TCX를 사용할 수 없으면 OBI와 Cilium은 모두 Netlink 인터페이스를 사용하여 eBPF
프로그램을 설치한다. OBI가 Cilium이 우선순위 1로 프로그램을 실행하는 것을
감지하면, OBI는 종료되고 오류를 표시한다. Cilium이 1보다 큰 우선순위를
사용하도록 구성하면 이 오류를 해결할 수 있다.

또한 OBI가 Netlink 연결을 사용하도록 구성되어 있는데 Cilium이 TCX를 사용하는
것을 감지하면, OBI는 실행을 거부한다.

### Cilium의 우선순위 구성 {#ciliums-priority-configuration}

`bpf.tc.priority` Helm 값이나 `tc-filter-priority` CLI 옵션을 사용하여 Cilium의
우선순위를 구성할 수 있다.

```yaml
bpf:
  tc:
    priority: 2
```

이렇게 하면 OBI 프로그램이 항상 Cilium 프로그램보다 먼저 실행된다.

## OBI 연결 모드 구성 {#obi-attachment-mode-configuration}

`OTEL_EBPF_BPF_TC_BACKEND` 구성 옵션을 사용하여 OBI TC 연결 모드를 구성하려면
[구성 문서](../configure/options/)를 참고한다.

다음을 수행할 수 있다.

- TCX API를 사용하려면 값을 `tcx`로 설정한다
- Netlink 인터페이스를 사용하려면 값을 `netlink`로 설정한다
- 사용 가능한 최선의 옵션을 자동으로 감지하려면 값을 `auto`로 설정한다

## OBI와 Cilium 데모 {#obi-and-cilium-demo}

다음 예시는 쿠버네티스 환경에서 트레이스 컨텍스트를 전파하기 위해 OBI와 Cilium이
함께 동작하는 방식을 보여준다.

### 사전 요구 사항 {#prerequisites}

- Cilium이 설치된 쿠버네티스 클러스터
- 클러스터에 접근하도록 구성된 kubectl
- Helm 3.0 이상

### 테스트 서비스 배포 {#deploy-test-services}

다음 정의를 사용하여 동일한 서비스를 배포한다. 이들은 서로 통신하는 작은
토이(toy) 서비스로, OBI가 트레이스 컨텍스트 전파와 함께 동작하는 모습을 볼 수
있게 해준다.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-service
  template:
    metadata:
      labels:
        app: nodejs-service
    spec:
      containers:
        - name: nodejs-service
          image: ghcr.io/open-telemetry/obi-testimg:node-0.1.0
          ports:
            - containerPort: 3030
          env:
            - name: NODEJS_SERVICE_PORT
              value: '3030'
            - name: NODEJS_SERVICE_HOST
              value: '0.0.0.0'
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
spec:
  selector:
    app: nodejs-service
  ports:
    - port: 3030
      targetPort: 3030
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-service
  template:
    metadata:
      labels:
        app: go-service
    spec:
      containers:
        - name: go-service
          image: ghcr.io/open-telemetry/obi-testimg:go-0.1.0
          ports:
            - containerPort: 8080
          env:
            - name: GO_SERVICE_PORT
              value: '8080'
            - name: GO_SERVICE_HOST
              value: '0.0.0.0'
---
apiVersion: v1
kind: Service
metadata:
  name: go-service
spec:
  selector:
    app: go-service
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-service
  template:
    metadata:
      labels:
        app: python-service
    spec:
      containers:
        - name: python-service
          image: ghcr.io/open-telemetry/obi-testimg:python-0.1.0
          ports:
            - containerPort: 8380
          env:
            - name: PYTHON_SERVICE_PORT
              value: '8380'
            - name: PYTHON_SERVICE_HOST
              value: '0.0.0.0'
---
apiVersion: v1
kind: Service
metadata:
  name: python-service
spec:
  selector:
    app: python-service
  ports:
    - port: 8380
      targetPort: 8380
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ruby-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ruby-service
  template:
    metadata:
      labels:
        app: ruby-service
    spec:
      containers:
        - name: ruby-service
          image: ghcr.io/open-telemetry/obi-testimg:rails-0.1.0
          ports:
            - containerPort: 3040
          env:
            - name: RAILS_SERVICE_PORT
              value: '3040'
            - name: RAILS_SERVICE_HOST
              value: '0.0.0.0'
---
apiVersion: v1
kind: Service
metadata:
  name: ruby-service
spec:
  selector:
    app: ruby-service
  ports:
    - port: 3040
      targetPort: 3040
```

### OBI 배포 {#deploy-obi}

OBI 네임스페이스를 생성한다.

```bash
kubectl create namespace obi
```

권한을 적용한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: obi
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
    namespace: obi
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: obi
```

OBI를 배포한다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: obi
  name: obi-config
data:
  obi-config.yml: |
    attributes:
      kubernetes:
        enable: true
    routes:
      unmatched: heuristic
    # let's instrument only the docs server
    discovery:
      instrument:
        - k8s_deployment_name: "nodejs-service"
        - k8s_deployment_name: "go-service"
        - k8s_deployment_name: "python-service"
        - k8s_deployment_name: "ruby-service"
    trace_printer: text
    ebpf:
      context_propagation: all
      traffic_control_backend: tcx
      disable_blackbox_cp: true
      track_request_headers: true
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace: obi
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
      hostPID: true
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: obi
          image: otel/ebpf-instrument:main
          securityContext:
            privileged: true
            readOnlyRootFilesystem: true
          volumeMounts:
            - mountPath: /config
              name: obi-config
            - mountPath: /var/run/obi
              name: var-run-obi
          env:
            - name: OTEL_EBPF_CONFIG_PATH
              value: '/config/obi-config.yml'
      volumes:
        - name: obi-config
          configMap:
            name: obi-config
        - name: var-run-obi
          emptyDir: {}
```

호스트로 포트를 포워딩하고 요청을 발생시킨다.

```shell
kubectl port-forward services/nodejs-service 3030:3030 &
curl http://localhost:3030/traceme
```

마지막으로 OBI Pod 로그를 확인한다.

```shell
for i in `kubectl get pods -n obi -o name | cut -d '/' -f2`; do kubectl logs -n obi $i | grep "GET " | sort; done
```

다음과 비슷하게, 트레이스 컨텍스트 전파와 함께 OBI가 감지한 요청을 보여주는
출력을 확인할 수 있다.

```text
2025-01-17 21:42:18.11794218 (5.045099ms[5.045099ms]) HTTPClient 200 GET /tracemetoo [10.244.1.92 as go-service.default:37450]->[10.96.214.17 as python-service.default:8080] size:0B svc=[default/go-service go] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-319fb03373427a41[cfa6d5d448e40b00]-01]
2025-01-17 21:42:18.11794218 (5.284521ms[5.164701ms]) HTTP 200 GET /gotracemetoo [10.244.2.144 as nodejs-service.default:57814]->[10.244.1.92 as go-service.default:8080] size:0B svc=[default/go-service go] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-cfa6d5d448e40b00[cce1e6b5e932b89a]-01]
2025-01-17 21:42:18.11794218 (1.934744ms[1.934744ms]) HTTP 403 GET /users [10.244.2.32 as ruby-service.default:46876]->[10.244.2.176 as ruby-service.default:3000] size:222B svc=[default/ruby-service ruby] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-57d77d99e9665c54[3d97d26b0051112b]-01]
2025-01-17 21:42:18.11794218 (2.116628ms[2.116628ms]) HTTPClient 403 GET /users [10.244.2.32 as ruby-service.default:46876]->[10.96.69.89 as ruby-service.default:3000] size:256B svc=[default/ruby-service ruby] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-ff48ab147cc92f93[2770ac4619aa0042]-01]
2025-01-17 21:42:18.11794218 (4.281525ms[4.281525ms]) HTTP 200 GET /tracemetoo [10.244.1.92 as go-service.default:37450]->[10.244.2.32 as ruby-service.default:8080] size:178B svc=[default/ruby-service ruby] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-2770ac4619aa0042[319fb03373427a41]-01]
2025-01-17 21:42:18.11794218 (5.391191ms[5.391191ms]) HTTPClient 200 GET /gotracemetoo [10.244.2.144 as nodejs-service.default:57814]->[10.96.134.167 as go-service.default:8080] size:256B svc=[default/nodejs-service nodejs] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-202ee68205e4ef3b[9408610968fa20f8]-01]
2025-01-17 21:42:18.11794218 (6.939027ms[6.939027ms]) HTTP 200 GET /traceme [127.0.0.1 as 127.0.0.1:44720]->[127.0.0.1 as 127.0.0.1.default:3000] size:86B svc=[default/nodejs-service nodejs] traceparent=[00-14f07e11b5e57f14fd2da0541f0ddc2f-9408610968fa20f8[0000000000000000]-01]
```
