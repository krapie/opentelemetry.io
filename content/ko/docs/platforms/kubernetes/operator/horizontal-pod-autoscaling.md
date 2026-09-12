---
title: 수평 파드 자동 확장
description:
  오픈텔레메트리 컬렉터와 함께 수평 파드 자동 확장(Horizontal Pod Autoscaling)을
  구성한다.
cSpell:ignore: autoscaler mebibyte mebibytes statefulset
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

오픈텔레메트리 오퍼레이터가 관리하는 컬렉터는
[수평 파드 자동 확장(Horizontal Pod Autoscaling, HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)을
기본으로 지원한다. HPA는 일련의 메트릭을 기준으로 쿠버네티스 파드의
레플리카(복제본) 수를 늘리거나 줄인다. 이 메트릭은 일반적으로 CPU 및/또는 메모리
사용량이다.

오픈텔레메트리 오퍼레이터가 컬렉터의 HPA 기능을 관리하게 하면, 컬렉터를
오토스케일링하기 위해 별도의 쿠버네티스 `HorizontalPodAutoscaler` 리소스를 만들
필요가 없다.

HPA는 쿠버네티스에서 `StatefulSet`과 `Deployment`에만 적용되므로, 컬렉터의
`spec.mode`가 `deployment`나 `statefulset` 중 하나인지 확인한다.

> [!NOTE]
>
> HPA를 사용하려면 쿠버네티스 클러스터에서
> [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)가 실행되고
> 있어야 한다.
>
> - [GKE(Google)](https://cloud.google.com/kubernetes-engine?hl=en)나
>   [AKS(Microsoft Azure)](https://azure.microsoft.com/en-us/products/kubernetes-service)와
>   같은 관리형 쿠버네티스 클러스터는 클러스터 프로비저닝 과정에서 Metrics
>   Server를 자동으로 설치해준다.
> - [EKS(AWS)는 기본적으로 Metrics Server가 설치되어 있지 않다](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html).
> - 비관리형 쿠버네티스 클러스터와 로컬 데스크톱 쿠버네티스 클러스터(예:
>   [MiniKube](https://minikube.sigs.k8s.io/docs/),
>   [KinD](https://kind.sigs.k8s.io/), [k0s](https://k0sproject.io))는 Metrics
>   Server를 수동으로 설치해야 한다.
>
> 관리형 쿠버네티스 클러스터에 Metrics Server가 사전 설치되어 있는지는 클라우드
> 제공업체의 문서를 참고해 확인한다.

HPA를 구성하려면 먼저 `OpenTelemetryCollector` YAML에 `spec.resources` 구성을
추가하여 리소스 요청과 제한을 정의해야 한다.

```yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi
```

> [!NOTE]
>
> 실제 값은 환경에 따라 다를 수 있다.

`limits` 구성은 최대 메모리와 CPU 값을 지정한다. 이 예에서는 CPU 제한이
100밀리코어(millicore, 0.1 코어)이고 RAM 제한이 128Mi(메비바이트, mebibyte,
1메비바이트 == 1024킬로바이트)이다.

`requests` 구성은 컨테이너에 할당되는 리소스의 최소 보장량을 지정한다. 이
예에서는 최소 할당량이 CPU 100밀리코어와 RAM 64메비바이트이다.

다음으로, `OpenTelemetryCollector` YAML에 `spec.autoscaler` 구성을 추가하여
오토스케일링 규칙을 구성한다.

```yaml
autoscaler:
  minReplicas: 1
  maxReplicas: 2
  targetCPUUtilization: 50
  targetMemoryUtilization: 60
```

> [!NOTE]
>
> 실제 값은 환경에 따라 다를 수 있다.

모두 합치면, `OpenTelemetryCollector` YAML의 시작 부분은 다음과 같은 모습이다.

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  image:
    otel/opentelemetry-collector-contrib:{{% version-from-registry
    collector-processor-batch %}}
  serviceAccount: otelcontribcol
  autoscaler:
    minReplicas: 1
    maxReplicas: 2
    targetCPUUtilization: 50
    targetMemoryUtilization: 60
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 100m
      memory: 64Mi
```

HPA가 활성화된 상태로 `OpenTelemetryCollector`가 쿠버네티스에 배포되면,
오퍼레이터는 컬렉터를 위한 `HorizontalPodAutoscaler` 리소스를 쿠버네티스에
생성한다. 다음을 실행하면 이를 확인할 수 있다.

`kubectl get hpa -n <your_namespace>`

정상적으로 동작했다면 명령의 출력은 다음과 같은 모습이어야 한다.

```nocode
NAME                REFERENCE                        TARGETS                         MINPODS   MAXPODS   REPLICAS   AGE
otelcol-collector   OpenTelemetryCollector/otelcol   memory: 68%/60%, cpu: 37%/50%   1         3         2          77s
```

더 자세한 정보를 확인하려면 다음 명령으로 HPA 리소스의 세부 정보를 조회할 수
있다.

`kubectl describe hpa <your_collector_name> -n <your_namespace>`

정상적으로 동작했다면 명령의 출력은 다음과 같은 모습이어야 한다.

```nocode
Name:                                                     otelcol-collector
Namespace:                                                opentelemetry
Labels:                                                   app.kubernetes.io/benchmark-test=otelcol-contrib
                                                          app.kubernetes.io/component=opentelemetry-collector
                                                          app.kubernetes.io/destination=dynatrace
                                                          app.kubernetes.io/instance=opentelemetry.otelcol
                                                          app.kubernetes.io/managed-by=opentelemetry-operator
                                                          app.kubernetes.io/name=otelcol-collector
                                                          app.kubernetes.io/part-of=opentelemetry
                                                          app.kubernetes.io/version=0.126.0
Annotations:                                              <none>
CreationTimestamp:                                        Mon, 02 Jun 2025 17:23:52 +0000
Reference:                                                OpenTelemetryCollector/otelcol
Metrics:                                                  ( current / target )
  resource memory on pods  (as a percentage of request):  71% (95779498666m) / 60%
  resource cpu on pods  (as a percentage of request):     12% (12m) / 50%
Min replicas:                                             1
Max replicas:                                             3
OpenTelemetryCollector pods:                              3 current / 3 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from memory resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type     Reason                   Age                  From                       Message
  ----     ------                   ----                 ----                       -------
  Warning  FailedGetResourceMetric  2m (x4 over 2m29s)   horizontal-pod-autoscaler  unable to get metric memory: no metrics returned from resource metrics API
  Warning  FailedGetResourceMetric  89s (x7 over 2m29s)  horizontal-pod-autoscaler  No recommendation
  Normal   SuccessfulRescale        89s                  horizontal-pod-autoscaler  New size: 2; reason: memory resource utilization (percentage of request) above target
  Normal   SuccessfulRescale        59s                  horizontal-pod-autoscaler  New size: 3; reason: memory resource utilization (percentage of request) above target
```
