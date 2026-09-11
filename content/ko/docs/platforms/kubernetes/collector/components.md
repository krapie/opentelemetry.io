---
title: 쿠버네티스를 위한 중요 컴포넌트
linkTitle: 컴포넌트
# prettier-ignore
cSpell:ignore: alertmanagers filelog horizontalpodautoscalers hostfs hostmetrics k8sattributes kubelet kubeletstats replicasets replicationcontrollers resourcequotas statefulsets varlibdockercontainers varlogpods
default_lang_commit: 30b7dbbdd94cec0b2a0c99317272b103315518bf
---

[오픈텔레메트리 컬렉터](/docs/collector/)는 쿠버네티스 모니터링을 돕기 위해
다양한 리시버와 프로세서를 지원한다. 이 섹션에서는 쿠버네티스 데이터를 수집하고
보강하는 데 가장 중요한 컴포넌트를 다룬다.

이 페이지에서 다루는 컴포넌트는 다음과 같다.

- [쿠버네티스 속성 프로세서(Kubernetes Attributes Processor)](#kubernetes-attributes-processor):
  수신되는 애플리케이션 텔레메트리에 쿠버네티스 메타데이터를 추가한다.
- [Kubeletstats 리시버(Kubeletstats Receiver)](#kubeletstats-receiver):
  kubelet의 API 서버에서 노드, 파드, 컨테이너 메트릭을 가져온다.
- [Filelog 리시버(Filelog Receiver)](#filelog-receiver): stdout/stderr에 기록된
  쿠버네티스 로그와 애플리케이션 로그를 수집한다.
- [쿠버네티스 클러스터 리시버(Kubernetes Cluster Receiver)](#kubernetes-cluster-receiver):
  클러스터 수준의 메트릭과 엔티티 이벤트를 수집한다.
- [쿠버네티스 오브젝트 리시버(Kubernetes Objects Receiver)](#kubernetes-objects-receiver):
  쿠버네티스 API 서버로부터 이벤트와 같은 오브젝트를 수집한다.
- [Prometheus 리시버(Prometheus Receiver)](#prometheus-receiver):
  [Prometheus](https://prometheus.io/) 형식의 메트릭을 수신한다.
- [호스트 메트릭 리시버(Host Metrics Receiver)](#host-metrics-receiver):
  쿠버네티스 노드에서 호스트 메트릭을 스크레이핑한다.

애플리케이션 트레이스, 메트릭, 로그의 경우
[OTLP 리시버](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)를
권장하지만, 데이터에 적합한 리시버라면 어떤 것이든 사용할 수 있다.

## 쿠버네티스 속성 프로세서(Kubernetes Attributes Processor) {#kubernetes-attributes-processor}

| 배포 패턴              | 사용 가능 여부 |
| ---------------------- | -------------- |
| DaemonSet(에이전트)    | 예             |
| Deployment(게이트웨이) | 예             |
| Sidecar                | 아니오         |

쿠버네티스 속성 프로세서는 쿠버네티스 파드를 자동으로 검색하고, 메타데이터를
추출하여, 추출한 메타데이터를 리소스 속성으로 스팬, 메트릭, 로그에 추가한다.

**쿠버네티스 속성 프로세서는 쿠버네티스에서 실행되는 컬렉터에 있어 가장 중요한
컴포넌트 중 하나이다. 애플리케이션 데이터를 수신하는 모든 컬렉터는 이를 사용해야
한다.** 이 프로세서는 텔레메트리에 쿠버네티스 컨텍스트를 추가하므로,
애플리케이션의 트레이스, 메트릭, 로그 시그널을 파드 메트릭 및 트레이스와 같은
쿠버네티스 텔레메트리와 연관 지을 수 있게 해준다.

쿠버네티스 속성 프로세서는 쿠버네티스 API를 사용해 클러스터에서 실행 중인 모든
파드를 검색하고, 이들의 IP 주소, 파드 UID, 기타 유용한 메타데이터를 기록해둔다.
기본적으로 프로세서를 통과하는 데이터는 수신 요청의 IP 주소를 기준으로 파드와
연관되지만, 다른 규칙을 구성할 수도 있다. 이 프로세서는 쿠버네티스 API를
사용하므로 특별한 권한이 필요하다(아래 예제 참고).
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용한다면
[`kubernetesAttributes` 프리셋](/docs/platforms/kubernetes/helm/collector/#kubernetes-attributes-preset)으로
쉽게 시작할 수 있다.

다음 속성이 기본적으로 추가된다.

- `k8s.namespace.name`
- `k8s.pod.name`
- `k8s.pod.uid`
- `k8s.pod.start_time`
- `k8s.deployment.name`
- `k8s.node.name`

쿠버네티스 속성 프로세서는 파드와 네임스페이스에 추가한 쿠버네티스 레이블과
쿠버네티스 어노테이션(annotation)을 사용해 트레이스, 메트릭, 로그에 대한 커스텀
리소스 속성을 설정할 수도 있다.

```yaml
k8sattributes:
  auth_type: 'serviceAccount'
  extract:
    metadata: # extracted from the pod
      - k8s.namespace.name
      - k8s.pod.name
      - k8s.pod.start_time
      - k8s.pod.uid
      - k8s.deployment.name
      - k8s.node.name
    annotations:
      # Extracts the value of a pod annotation with key `annotation-one` and inserts it as a resource attribute with key `a1`
      - tag_name: a1
        key: annotation-one
        from: pod
      # Extracts the value of a namespaces annotation with key `annotation-two` with regexp and inserts it as a resource  with key `a2`
      - tag_name: a2
        key: annotation-two
        regex: field=(?P<value>.+)
        from: namespace
    labels:
      # Extracts the value of a namespaces label with key `label1` and inserts it as a resource attribute with key `l1`
      - tag_name: l1
        key: label1
        from: namespace
      # Extracts the value of a pod label with key `label2` with regexp and inserts it as a resource attribute with key `l2`
      - tag_name: l2
        key: label2
        regex: field=(?P<value>.+)
        from: pod
  pod_association: # How to associate the data to a pod (order matters)
    - sources: # First try to use the value of the resource attribute k8s.pod.ip
        - from: resource_attribute
          name: k8s.pod.ip
    - sources: # Then try to use the value of the resource attribute k8s.pod.uid
        - from: resource_attribute
          name: k8s.pod.uid
    - sources: # If neither of those work, use the request's connection to get the pod IP.
        - from: connection
```

컬렉터가 쿠버네티스 DaemonSet(에이전트)이나 쿠버네티스 Deployment(게이트웨이)로
배포될 때를 위한 특수 구성 옵션도 있다. 자세한 내용은
[배포 시나리오](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor#deployment-scenarios)를
참고한다.

쿠버네티스 속성 프로세서 구성에 대한 자세한 내용은
[Kubernetes Attributes Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor)를
참고한다.

프로세서가 쿠버네티스 API를 사용하므로 올바르게 동작하려면 적절한 권한이
필요하다. 대부분의 경우, 컬렉터를 실행하는 서비스 계정(ServiceAccount)에
ClusterRole을 통해 다음 권한을 부여해야 한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: collector
  namespace: <OTEL_COL_NAMESPACE>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups:
      - ''
    resources:
      - 'pods'
      - 'namespaces'
    verbs:
      - 'get'
      - 'watch'
      - 'list'
  - apiGroups:
      - 'apps'
    resources:
      - 'replicasets'
    verbs:
      - 'get'
      - 'list'
      - 'watch'
  - apiGroups:
      - 'extensions'
    resources:
      - 'replicasets'
    verbs:
      - 'get'
      - 'list'
      - 'watch'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: collector
    namespace: <OTEL_COL_NAMESPACE>
roleRef:
  kind: ClusterRole
  name: otel-collector
  apiGroup: rbac.authorization.k8s.io
```

## Kubeletstats 리시버(Kubeletstats Receiver) {#kubeletstats-receiver}

| 배포 패턴              | 사용 가능 여부                         |
| ---------------------- | -------------------------------------- |
| DaemonSet(에이전트)    | 권장                                   |
| Deployment(게이트웨이) | 예, 단 배포된 노드의 메트릭만 수집한다 |
| Sidecar                | 아니오                                 |

각 쿠버네티스 노드는 API 서버를 포함하는 kubelet을 실행한다. Kubeletstats
리시버는 이 API 서버를 통해 kubelet에 연결하여 노드와 노드에서 실행 중인
워크로드에 대한 메트릭을 수집한다.

인증 방법에는 여러 가지가 있지만 일반적으로 서비스 계정을 사용한다. 서비스
계정은 kubelet에서 데이터를 가져오기 위한 적절한 권한도 필요하다(아래 참고).
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용한다면
[`kubeletMetrics` 프리셋](/docs/platforms/kubernetes/helm/collector/#kubelet-metrics-preset)으로
쉽게 시작할 수 있다.

기본적으로 파드와 노드에 대한 메트릭이 수집되지만, 컨테이너와 볼륨 메트릭도
수집하도록 리시버를 구성할 수 있다. 리시버는 메트릭을 수집하는 주기도 구성할 수
있게 해준다.

```yaml
receivers:
  kubeletstats:
    collection_interval: 10s
    auth_type: 'serviceAccount'
    endpoint: '${env:K8S_NODE_NAME}:10250'
    insecure_skip_verify: true
    metric_groups:
      - node
      - pod
      - container
```

수집되는 메트릭에 대한 구체적인 내용은
[기본 메트릭](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/documentation.md)을
참고한다. 구체적인 구성 내용은
[Kubeletstats Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver)를
참고한다.

프로세서가 쿠버네티스 API를 사용하므로 올바르게 동작하려면 적절한 권한이
필요하다. 대부분의 경우, 컬렉터를 실행하는 서비스 계정에 ClusterRole을 통해 다음
권한을 부여해야 한다.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: ['']
    resources: ['nodes/stats']
    verbs: ['get', 'watch', 'list']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: default
```

## Filelog 리시버(Filelog Receiver) {#filelog-receiver}

| 배포 패턴              | 사용 가능 여부                       |
| ---------------------- | ------------------------------------ |
| DaemonSet(에이전트)    | 권장                                 |
| Deployment(게이트웨이) | 예, 단 배포된 노드의 로그만 수집한다 |
| Sidecar                | 예, 단 고급 구성으로 간주된다        |

Filelog 리시버는 파일의 로그를 테일링(tail)하고 파싱한다. 쿠버네티스 전용
리시버는 아니지만, 쿠버네티스에서 로그를 수집하기 위한 사실상의(de facto) 표준
솔루션이다.

Filelog 리시버는 로그를 처리하기 위해 체이닝된 오퍼레이터(Operator)들로
구성된다. 각 오퍼레이터는 타임스탬프나 JSON을 파싱하는 등 단순한 역할을
수행한다. Filelog 리시버를 구성하는 것은 간단하지 않다.
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용한다면
[`logsCollection` 프리셋](/docs/platforms/kubernetes/helm/collector/#logs-collection-preset)으로
쉽게 시작할 수 있다.

쿠버네티스 로그는 일반적으로 일련의 표준 형식을 따르므로, 쿠버네티스용 일반적인
Filelog 리시버 구성은 다음과 같다.

```yaml
filelog:
  include:
    - /var/log/pods/*/*/*.log
  exclude:
    # Exclude logs from all containers named otel-collector
    - /var/log/pods/*/otel-collector/*.log
  start_at: end
  include_file_path: true
  include_file_name: false
  operators:
    # parse container logs
    - type: container
      id: container-parser
```

Filelog 리시버 구성에 대한 자세한 내용은
[Filelog Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver)를
참고한다.

Filelog 리시버 구성 외에도, 쿠버네티스에 설치된 오픈텔레메트리 컬렉터는
수집하려는 로그에 접근할 수 있어야 한다. 일반적으로 컬렉터 매니페스트에
볼륨(volume)과 volumeMounts를 추가하는 것을 의미한다.

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
        - name: opentelemetry-collector
          ...
          volumeMounts:
            ...
            # Mount the volumes to the collector container
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            ...
      volumes:
        ...
        # Typically the collector will want access to pod logs and container logs
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        ...
```

## 쿠버네티스 클러스터 리시버(Kubernetes Cluster Receiver) {#kubernetes-cluster-receiver}

| 배포 패턴              | 사용 가능 여부                                          |
| ---------------------- | ------------------------------------------------------- |
| DaemonSet(에이전트)    | 예, 단 중복 데이터가 발생한다                           |
| Deployment(게이트웨이) | 예, 단 레플리카가 두 개 이상이면 중복 데이터가 발생한다 |
| Sidecar                | 아니오                                                  |

쿠버네티스 클러스터 리시버는 쿠버네티스 API 서버를 사용해 클러스터 전체에 대한
메트릭과 엔티티 이벤트를 수집한다. 이 리시버를 사용하면 파드 단계(phase), 노드
상태, 기타 클러스터 전반의 질문에 답할 수 있다. 이 리시버는 클러스터 전체에 대한
텔레메트리를 수집하므로, 모든 데이터를 수집하는 데는 클러스터 전체에서 단 하나의
리시버 인스턴스만 있으면 된다.

인증 방법에는 여러 가지가 있지만 일반적으로 서비스 계정을 사용한다. 서비스
계정은 쿠버네티스 API 서버에서 데이터를 가져오기 위한 적절한 권한도
필요하다(아래 참고).
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용한다면
[`clusterMetrics` 프리셋](/docs/platforms/kubernetes/helm/collector/#cluster-metrics-preset)으로
쉽게 시작할 수 있다.

노드 상태의 경우, 리시버는 기본적으로 `Ready`만 수집하지만 더 많은 항목을
수집하도록 구성할 수 있다. 또한 `cpu`, `memory`와 같은 할당 가능한 (allocatable)
리소스 집합을 보고하도록 리시버를 구성할 수도 있다.

```yaml
k8s_cluster:
  auth_type: serviceAccount
  node_conditions_to_report:
    - Ready
    - MemoryPressure
  allocatable_types_to_report:
    - cpu
    - memory
```

수집되는 메트릭에 대해 더 알아보려면
[기본 메트릭](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/documentation.md)을
참고한다. 구성에 대한 자세한 내용은
[Kubernetes Cluster Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sclusterreceiver)를
참고한다.

프로세서가 쿠버네티스 API를 사용하므로 올바르게 동작하려면 적절한 권한이
필요하다. 대부분의 경우, 컬렉터를 실행하는 서비스 계정에 ClusterRole을 통해 다음
권한을 부여해야 한다.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-opentelemetry-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-opentelemetry-collector
rules:
  - apiGroups:
      - ''
    resources:
      - events
      - namespaces
      - namespaces/status
      - nodes
      - nodes/spec
      - pods
      - pods/status
      - replicationcontrollers
      - replicationcontrollers/status
      - resourcequotas
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - daemonsets
      - deployments
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - daemonsets
      - deployments
      - replicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - batch
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-opentelemetry-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-opentelemetry-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: default
```

## 쿠버네티스 오브젝트 리시버(Kubernetes Objects Receiver) {#kubernetes-objects-receiver}

| 배포 패턴              | 사용 가능 여부                                          |
| ---------------------- | ------------------------------------------------------- |
| DaemonSet(에이전트)    | 예, 단 중복 데이터가 발생한다                           |
| Deployment(게이트웨이) | 예, 단 레플리카가 두 개 이상이면 중복 데이터가 발생한다 |
| Sidecar                | 아니오                                                  |

쿠버네티스 오브젝트 리시버는 풀링(pulling)하거나 워칭(watching)하는 방식으로
쿠버네티스 API 서버로부터 오브젝트를 수집한다. 이 리시버의 가장 일반적인 사용
사례는 쿠버네티스 이벤트를 워칭하는 것이지만, 모든 유형의 쿠버네티스 오브젝트를
수집하는 데 사용할 수 있다. 이 리시버는 클러스터 전체에 대한 텔레메트리를
수집하므로, 모든 데이터를 수집하는 데는 클러스터 전체에서 단 하나의 리시버
인스턴스만 있으면 된다.

현재는 인증에 서비스 계정만 사용할 수 있다. 서비스 계정은 쿠버네티스 API
서버에서 데이터를 가져오기 위한 적절한 권한도 필요하다(아래 참고).
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용하며 이벤트를 유입(ingest)하고자 한다면
[`kubernetesEvents` 프리셋](/docs/platforms/kubernetes/helm/collector/#cluster-metrics-preset)으로
쉽게 시작할 수 있다.

풀링용으로 구성된 오브젝트의 경우, 리시버는 쿠버네티스 API를 사용해 클러스터의
모든 오브젝트를 주기적으로 나열한다. 각 오브젝트는 각각 하나의 로그로 변환된다.
워칭용으로 구성된 오브젝트의 경우, 리시버는 쿠버네티스 API와 스트림을 생성하여
오브젝트가 변경될 때 업데이트를 수신한다.

클러스터에서 수집 가능한 오브젝트를 확인하려면 `kubectl api-resources`를
실행한다.

<!-- cspell:disable -->

```console
kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v2                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1                               true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1                              true         PodDisruptionBudget
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
csistoragecapacities                           storage.k8s.io/v1                      true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

<!-- cspell:enable -->

구체적인 구성 내용은
[Kubernetes Objects Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sobjectsreceiver)를
참고한다.

프로세서가 쿠버네티스 API를 사용하므로 올바르게 동작하려면 적절한 권한이
필요하다. 서비스 계정이 유일한 인증 옵션이므로 서비스 계정에 적절한 접근 권한을
부여해야 한다. 수집하려는 오브젝트마다 해당 이름이 ClusterRole에 추가되어 있는지
확인해야 한다. 예를 들어 파드를 수집하고자 한다면 ClusterRole은 다음과 같다.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-opentelemetry-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-opentelemetry-collector
rules:
  - apiGroups:
      - ''
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-opentelemetry-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-opentelemetry-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: default
```

## Prometheus 리시버(Prometheus Receiver) {#prometheus-receiver}

| 배포 패턴              | 사용 가능 여부 |
| ---------------------- | -------------- |
| DaemonSet(에이전트)    | 예             |
| Deployment(게이트웨이) | 예             |
| Sidecar                | 아니오         |

Prometheus는 쿠버네티스와 쿠버네티스에서 실행되는 서비스 모두에서 널리 쓰이는
메트릭 형식이다. Prometheus 리시버는 이러한 메트릭을 수집하기 위한 최소한의
드롭인(drop-in) 대체재이다. Prometheus의 전체
[`scrape_config` 옵션](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)을
지원한다.

이 리시버가 지원하지 않는 몇 가지 고급 Prometheus 기능이 있다. 구성 YAML/코드에
다음 중 하나라도 포함되어 있으면 리시버는 오류를 반환한다.

- `alert_config.alertmanagers`
- `alert_config.relabel_configs`
- `remote_read`
- `remote_write`
- `rule_files`

구체적인 구성 내용은
[Prometheus Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver)를
참고한다.

Prometheus 리시버는
[상태 저장(Stateful)](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/standard-warnings.md#statefulness)
방식이므로, 사용할 때 고려해야 할 중요한 세부 사항이 있다.

- 컬렉터의 레플리카가 여러 개 실행 중일 때, 컬렉터는 스크레이핑 프로세스를
  오토스케일링할 수 없다.
- 동일한 구성으로 컬렉터의 여러 레플리카를 실행하면, 대상을 여러 번
  스크레이핑하게 된다.
- 스크레이핑 프로세스를 수동으로 샤딩하려면 사용자는 각 레플리카를 서로 다른
  스크레이핑 구성으로 설정해야 한다.

Prometheus 리시버 구성을 더 쉽게 만들기 위해, 오픈텔레메트리 오퍼레이터는
[타겟 얼로케이터(Target Allocator)](/docs/platforms/kubernetes/operator/target-allocator)라는
선택적 컴포넌트를 포함한다. 이 컴포넌트를 사용하면 컬렉터에게 어떤 Prometheus
엔드포인트를 스크레이핑해야 하는지 알려줄 수 있다.

리시버의 설계에 대한 자세한 내용은
[설계](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/prometheusreceiver/DESIGN.md)를
참고한다.

## 호스트 메트릭 리시버(Host Metrics Receiver) {#host-metrics-receiver}

| 배포 패턴              | 사용 가능 여부                         |
| ---------------------- | -------------------------------------- |
| DaemonSet(에이전트)    | 권장                                   |
| Deployment(게이트웨이) | 예, 단 배포된 노드의 메트릭만 수집한다 |
| Sidecar                | 아니오                                 |

호스트 메트릭 리시버는 다양한 스크레이퍼(scraper)를 사용해 호스트로부터 메트릭을
수집한다. [Kubeletstats 리시버](#kubeletstats-receiver)와 일부 겹치는 부분이
있으므로, 둘 다 사용하기로 했다면 중복되는 메트릭을 비활성화하는 것이 좋을 수
있다.

쿠버네티스에서 이 리시버가 제대로 동작하려면 `hostfs` 볼륨에 접근할 수 있어야
한다.
[오픈텔레메트리 컬렉터 Helm 차트](/docs/platforms/kubernetes/helm/collector/)를
사용한다면
[`hostMetrics` 프리셋](/docs/platforms/kubernetes/helm/collector/#host-metrics-preset)으로
쉽게 시작할 수 있다.

사용 가능한 스크레이퍼는 다음과 같다.

| 스크레이퍼 | 지원 OS               | 설명                                              |
| ---------- | --------------------- | ------------------------------------------------- |
| cpu        | macOS 제외 전체[^1]   | CPU 사용률 메트릭                                 |
| disk       | macOS 제외 전체[^1]   | 디스크 I/O 메트릭                                 |
| load       | 전체                  | CPU 로드 메트릭                                   |
| filesystem | 전체                  | 파일 시스템 사용률 메트릭                         |
| memory     | 전체                  | 메모리 사용률 메트릭                              |
| network    | 전체                  | 네트워크 인터페이스 I/O 메트릭 및 TCP 연결 메트릭 |
| paging     | 전체                  | 페이징/스왑 공간 사용률 및 I/O 메트릭             |
| processes  | Linux, macOS          | 프로세스 수 메트릭                                |
| process    | Linux, macOS, Windows | 프로세스별 CPU, 메모리, 디스크 I/O 메트릭         |

[^1]:
    cgo 없이 컴파일된 경우 macOS에서 지원되지 않으며, 이는 컬렉터 SIG가
    릴리스하는 이미지의 기본값이다.

수집되는 메트릭과 구체적인 구성 내용에 대한 자세한 사항은
[Host Metrics Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver)를
참고한다.

컴포넌트를 직접 구성해야 한다면, 컨테이너가 아닌 노드의 메트릭을 수집하려는 경우
`hostfs` 볼륨을 마운트해야 한다.

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
        - name: opentelemetry-collector
          ...
          volumeMounts:
            ...
            - name: hostfs
              mountPath: /hostfs
              readOnly: true
              mountPropagation: HostToContainer
      volumes:
        ...
        - name: hostfs
          hostPath:
            path: /
      ...
```

그런 다음 `volumeMount`를 사용하도록 호스트 메트릭 리시버를 구성한다.

```yaml
receivers:
  host_metrics:
    root_path: /hostfs
    collection_interval: 10s
    scrapers:
      cpu:
      load:
      memory:
      disk:
      filesystem:
      network:
```

컨테이너 내부에서 리시버를 사용하는 방법에 대한 자세한 내용은
[컨테이너 내부에서 호스트 메트릭 수집하기(Linux 전용)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver#collecting-host-metrics-from-inside-a-container-linux-only)를
참고한다.
