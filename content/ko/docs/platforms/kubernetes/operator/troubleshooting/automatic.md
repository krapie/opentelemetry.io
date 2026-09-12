---
title: 자동 계측
cSpell:ignore: PYTHONPATH
default_lang_commit: 48d3ff356dc39a3b1323637f3163d435dc751228
---

[오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator)의
[자동 계측](/docs/platforms/kubernetes/operator/automatic) 주입 기능을 사용하고
있는데 트레이스나 메트릭이 보이지 않는다면, 다음 문제 해결 단계를 따라 무슨 일이
일어나고 있는지 파악한다.

## 문제 해결 단계 {#troubleshooting-steps}

### 설치 상태 확인 {#check-installation-status}

`Instrumentation` 리소스를 설치한 후, 다음 명령을 실행해 올바르게 설치되었는지
확인한다.

```shell
kubectl describe otelinst -n <namespace>
```

여기서 `<namespace>`는 `Instrumentation` 리소스가 배포된 네임스페이스이다.

출력은 다음과 같은 모습이어야 한다.

```yaml
Name:         python-instrumentation
Namespace:    application
Labels:       app.kubernetes.io/managed-by=opentelemetry-operator
Annotations:  instrumentation.opentelemetry.io/default-auto-instrumentation-apache-httpd-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-apache-httpd:1.0.3
             instrumentation.opentelemetry.io/default-auto-instrumentation-dotnet-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:0.7.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-go-image:
               ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.2.1-alpha
             instrumentation.opentelemetry.io/default-auto-instrumentation-java-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.26.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-nodejs-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.40.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-python-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
API Version:  opentelemetry.io/v1alpha1
Kind:         Instrumentation
Metadata:
 Creation Timestamp:  2023-07-28T03:42:12Z
 Generation:          1
 Resource Version:    3385
 UID:                 646661d5-a8fc-4b64-80b7-8587c9865f53
Spec:
...
 Exporter:
   Endpoint:  http://otel-collector-collector.opentelemetry.svc.cluster.local:4318
...
 Propagators:
   tracecontext
   baggage
 Python:
   Image:  ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
   Resource Requirements:
     Limits:
       Cpu:     500m
       Memory:  32Mi
     Requests:
       Cpu:     50m
       Memory:  32Mi
 Resource:
 Sampler:
Events:  <none>
```

### 오픈텔레메트리 오퍼레이터 로그 확인 {#check-the-opentelemetry-operator-logs}

다음 명령을 실행해 오픈텔레메트리 오퍼레이터 로그에서 오류를 확인한다.

```shell
kubectl logs -l app.kubernetes.io/name=opentelemetry-operator --container manager -n opentelemetry-operator-system --follow
```

로그에 자동 계측 관련 오류가 표시되지 않아야 한다.

### 배포 순서 확인 {#check-deployment-order}

배포 순서가 올바른지 확인한다. `Instrumentation` 리소스는 자동 계측 대상이 되는
해당 `Deployment` 리소스를 배포하기 전에 먼저 배포되어야 한다.

다음 자동 계측 어노테이션 스니펫을 살펴본다.

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-python: 'true'
```

파드가 시작될 때, 이 어노테이션은 오퍼레이터에게 파드의 네임스페이스에서
`Instrumentation` 리소스를 찾아 파드에 Python 자동 계측을 주입하라고 지시한다.
이는 애플리케이션의 파드에
[init-container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)인
`opentelemetry-auto-instrumentation`을 추가하며, 이 컨테이너는 앱 컨테이너에
자동 계측을 주입하는 데 사용된다.

다음을 실행하면 이를 확인할 수 있다.

```shell
kubectl describe pod <your_pod_name> -n <namespace>
```

여기서 `<namespace>`는 파드가 배포된 네임스페이스이다. 결과 출력은 자동 계측
주입 후 파드 스펙이 어떤 모습일 수 있는지 보여주는 다음 예시와 같은 모습이어야
한다.

```text
Name:             py-otel-server-f89fdbc4f-mtsps
Namespace:        opentelemetry
Priority:         0
Service Account:  default
Node:             otel-target-allocator-talk-control-plane/172.24.0.2
Start Time:       Mon, 15 Jul 2024 17:23:45 -0400
Labels:           app=my-app
                  app.kubernetes.io/name=py-otel-server
                  pod-template-hash=f89fdbc4f
Annotations:      instrumentation.opentelemetry.io/inject-python: true
Status:           Running
IP:               10.244.0.10
IPs:
  IP:           10.244.0.10
Controlled By:  ReplicaSet/py-otel-server-f89fdbc4f
Init Containers:
  opentelemetry-auto-instrumentation-python:
    Container ID:  containerd://20ecf8766247e6043fcad46544dba08c3ef534ee29783ca552d2cf758a5e3868
    Image:         ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.45b0
    Image ID:      ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python@sha256:3ed1122e10375d527d84c826728f75322d614dfeed7c3a8d2edd0d391d0e7973
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      -r
      /autoinstrumentation/.
      /otel-auto-instrumentation-python
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 15 Jul 2024 17:23:51 -0400
      Finished:     Mon, 15 Jul 2024 17:23:51 -0400
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  32Mi
    Requests:
      cpu:        50m
      memory:     32Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation-python from opentelemetry-auto-instrumentation-python (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-x2nmj (ro)
Containers:
  py-otel-server:
    Container ID:   containerd://95fb6d06b08ead768f380be2539a93955251be6191fa74fa2e6e5616036a8f25
    Image:          otel-target-allocator-talk:0.1.0-py-otel-server
    Image ID:       docker.io/library/import-2024-07-15@sha256:a2ed39e9a39ca090fedbcbd474c43bac4f8c854336a8500e874bd5b577e37c25
    Port:           8082/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 15 Jul 2024 17:23:52 -0400
    Ready:          True
    Restart Count:  0
    Environment:
      OTEL_NODE_IP:                                       (v1:status.hostIP)
      OTEL_POD_IP:                                        (v1:status.podIP)
      OTEL_METRICS_EXPORTER:                             console,otlp_proto_http
      OTEL_LOGS_EXPORTER:                                otlp_proto_http
      PYTHONPATH:                                        /otel-auto-instrumentation-python/opentelemetry/instrumentation/auto_instrumentation:/otel-auto-instrumentation-python
      OTEL_TRACES_EXPORTER:                              otlp
      OTEL_EXPORTER_OTLP_TRACES_PROTOCOL:                http/protobuf
      OTEL_EXPORTER_OTLP_METRICS_PROTOCOL:               http/protobuf
      OTEL_SERVICE_NAME:                                 py-otel-server
      OTEL_EXPORTER_OTLP_ENDPOINT:                       http://otelcol-collector.opentelemetry.svc.cluster.local:4318
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:                 py-otel-server-f89fdbc4f-mtsps (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:                 (v1:spec.nodeName)
      OTEL_PROPAGATORS:                                  tracecontext,baggage
      OTEL_RESOURCE_ATTRIBUTES:                          service.name=py-otel-server,service.version=0.1.0,k8s.container.name=py-otel-server,k8s.deployment.name=py-otel-server,k8s.namespace.name=opentelemetry,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=py-otel-server-f89fdbc4f,service.instance.id=opentelemetry.$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME).py-otel-server
    Mounts:
      /otel-auto-instrumentation-python from opentelemetry-auto-instrumentation-python (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-x2nmj (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-x2nmj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  opentelemetry-auto-instrumentation-python:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:   200Mi
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  99s   default-scheduler  Successfully assigned opentelemetry/py-otel-server-f89fdbc4f-mtsps to otel-target-allocator-talk-control-plane
  Normal  Pulling    99s   kubelet            Pulling image "ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.45b0"
  Normal  Pulled     93s   kubelet            Successfully pulled image "ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.45b0" in 288.756166ms (5.603779501s including waiting)
  Normal  Created    93s   kubelet            Created container opentelemetry-auto-instrumentation-python
  Normal  Started    93s   kubelet            Started container opentelemetry-auto-instrumentation-python
  Normal  Pulled     92s   kubelet            Container image "otel-target-allocator-talk:0.1.0-py-otel-server" already present on machine
  Normal  Created    92s   kubelet            Created container py-otel-server
  Normal  Started    92s   kubelet            Started container py-otel-server
```

`Deployment`가 배포되는 시점에 `Instrumentation` 리소스가 없으면
`init-container`를 생성할 수 없다. 즉, `Instrumentation` 리소스를 배포하기 전에
`Deployment` 리소스가 배포되면 자동 계측이 초기화에 실패한다.

다음 명령을 실행해 `opentelemetry-auto-instrumentation` `init-container`가
올바르게 시작되었는지(또는 아예 시작이라도 되었는지) 확인한다.

```shell
kubectl get events -n <namespace>
```

여기서 `<namespace>`는 파드가 배포된 네임스페이스이다. 결과 출력은 다음 예시와
같은 모습이어야 한다.

```text
53s         Normal   Created             pod/py-otel-server-7f54bf4cbc-p8wmj    Created container opentelemetry-auto-instrumentation
53s         Normal   Started             pod/py-otel-server-7f54bf4cbc-p8wmj    Started container opentelemetry-auto-instrumentation
```

출력에서 `opentelemetry-auto-instrumentation`에 대한 `Created`나 `Started`
항목이 빠져 있다면 자동 계측 구성에 문제가 있을 수 있다. 다음 중 하나가 원인일
수 있다.

- `Instrumentation` 리소스가 설치되지 않았거나 올바르게 설치되지 않았다.
- `Instrumentation` 리소스가 애플리케이션이 배포된 후에 설치되었다.
- 자동 계측 어노테이션에 오류가 있거나, 어노테이션이 잘못된 위치에 있다. 다음
  섹션을 참고한다.

이벤트 명령의 출력에서 오류가 있는지도 확인해보면 문제를 파악하는 데 도움이 될
수 있다.

### 자동 계측 어노테이션 확인 {#check-the-auto-instrumentation-annotation}

다음 자동 계측 어노테이션 스니펫을 살펴본다.

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-python: 'true'
```

`Deployment` 리소스가 `application`이라는 네임스페이스에 배포되고,
`my-instrumentation`이라는 `Instrumentation` 리소스가 `opentelemetry`라는
네임스페이스에 배포되어 있다면, 위 어노테이션은 동작하지 않는다.

대신 어노테이션은 다음과 같아야 한다.

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-python: 'opentelemetry/my-instrumentation'
```

여기서 `opentelemetry`는 `Instrumentation` 리소스의 네임스페이스이고,
`my-instrumentation`은 `Instrumentation` 리소스의 이름이다.

[어노테이션에 사용할 수 있는 값은 다음과 같다](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/collector/sidecar-injection.md?plain=1#L54-L59).

- `"true"` - 네임스페이스의 `OpenTelemetryCollector` 리소스를 주입한다.
- `"sidecar-for-my-app"` - 현재 네임스페이스에 있는 `OpenTelemetryCollector` CR
  인스턴스의 이름이다.
- `"my-other-namespace/my-instrumentation"` - 다른 네임스페이스에 있는
  `OpenTelemetryCollector` CR 인스턴스의 이름과 네임스페이스이다.
- `"false"` - 주입하지 않는다.

### 자동 계측 구성 확인 {#check-the-auto-instrumentation-configuration}

자동 계측 어노테이션이 올바르게 추가되지 않았을 수 있다. 다음을 확인한다.

- 올바른 언어에 대해 자동 계측을 하고 있는가? 예를 들어 Python 애플리케이션을
  자동 계측하려다가 실수로 JavaScript 자동 계측 어노테이션을 추가하지는
  않았는가?
- 자동 계측 어노테이션을 올바른 위치에 추가했는가? `Deployment` 리소스를 정의할
  때 어노테이션을 추가할 수 있는 위치는 `spec.metadata.annotations`와
  `spec.template.metadata.annotations` 두 곳이다. 자동 계측 어노테이션은
  `spec.template.metadata.annotations`에 추가해야 하며, 그렇지 않으면 동작하지
  않는다.

### 자동 계측 엔드포인트 구성 확인 {#check-auto-instrumentation-endpoint-configuration}

`Instrumentation` 리소스의 `spec.exporter.endpoint` 구성을 사용하면 텔레메트리
데이터의 대상을 정의할 수 있다. 생략하면 기본값인 `http://localhost:4317`이
사용되며, 이 경우 데이터가 드롭된다.

텔레메트리를 [컬렉터](/docs/collector/)로 전송하고 있다면,
`spec.exporter.endpoint` 값은 컬렉터의
[`Service`](https://kubernetes.io/docs/concepts/services-networking/service/)
이름을 참조해야 한다.

예를 들면 `http://otel-collector.opentelemetry.svc.cluster.local:4318`이다.

여기서 `otel-collector`는 OTel 컬렉터 쿠버네티스
[`Service`](https://kubernetes.io/docs/concepts/services-networking/service/)의
이름이다.

또한 컬렉터가 다른 네임스페이스에서 실행 중이라면, 컬렉터의 서비스 이름 뒤에
`opentelemetry.svc.cluster.local`을 붙여야 한다. 여기서 `opentelemetry`는
컬렉터가 위치한 네임스페이스이며, 원하는 어떤 네임스페이스든 사용할 수 있다.

마지막으로 올바른 컬렉터 포트를 사용하고 있는지 확인한다. 일반적으로
`4317`(gRPC)이나 `4318`(HTTP) 중 하나를 선택할 수 있지만,
[Python 자동 계측에서는 `4318`만 사용할 수 있다](/docs/platforms/kubernetes/operator/automatic/#python).

### 구성 소스 확인 {#check-configuration-sources}

자동 계측은 현재 Docker 이미지에 설정되어 있거나 `ConfigMap`에 정의되어 있는
경우 Java의 `JAVA_TOOL_OPTIONS`, Python의 `PYTHONPATH`, Node.js의
`NODE_OPTIONS`를 재정의(override)한다. 이는 알려진 문제이며, 문제가 해결되기
전까지는 이런 방식으로 이 환경 변수들을 설정하는 것을 피해야 한다.

관련 이슈는
[Java](https://github.com/open-telemetry/opentelemetry-operator/issues/1814),
[Python](https://github.com/open-telemetry/opentelemetry-operator/issues/1884),
[Node.js](https://github.com/open-telemetry/opentelemetry-operator/issues/1393)를
참고한다.
