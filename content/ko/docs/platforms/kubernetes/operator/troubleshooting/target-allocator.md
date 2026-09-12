---
title: 타겟 얼로케이터
cSpell:ignore: bleh targetallocator
default_lang_commit: 48d3ff356dc39a3b1323637f3163d435dc751228
---

[오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator/)에서
[타겟 얼로케이터](/docs/platforms/kubernetes/operator/target-allocator/) 서비스
디스커버리를 활성화했는데 타겟 얼로케이터가 스크레이프 대상(scrape target)을
검색하지 못하고 있다면, 무슨 일이 일어나고 있는지 파악하고 정상 동작을 복구하는
데 도움이 되는 몇 가지 문제 해결 단계가 있다.

## 문제 해결 단계 {#troubleshooting-steps}

### 모든 리소스를 쿠버네티스에 배포했는가? {#did-you-deploy-all-of-your-resources-to-kubernetes}

가장 먼저, 관련된 모든 리소스를 쿠버네티스 클러스터에 배포했는지 확인한다.

### 실제로 메트릭이 스크레이핑되고 있는지 확인했는가? {#do-you-know-if-metrics-are-actually-being-scraped}

모든 리소스를 쿠버네티스에 배포한 후에는, 타겟 얼로케이터가
[`ServiceMonitor`](https://prometheus-operator.dev/docs/getting-started/design/#servicemonitor)나
[PodMonitor][]로부터 스크레이프 대상을 검색하고 있는지 확인한다.

다음과 같은 `ServiceMonitor` 정의가 있다고 가정한다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  namespace: opentelemetry
  labels:
    app.kubernetes.io/name: py-prometheus-app
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - opentelemetry
  endpoints:
    - port: prom
      path: /metrics
    - port: py-client-port
      interval: 15s
    - port: py-server-port
```

다음 `Service` 정의:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: prom
      port: 8080
```

그리고 다음 `OpenTelemetryCollector` 정의:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
      podMonitorSelector: {}
      serviceMonitorSelector: {}
  config:
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
      prometheus:
        config:
          scrape_configs:
            - job_name: 'otel-collector'
              scrape_interval: 10s
              static_configs:
                - targets: ['0.0.0.0:8888']
    exporters:
      debug:
        verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [debug]
        metrics:
          receivers: [otlp, prometheus]
          exporters: [debug]
        logs:
          receivers: [otlp]
          exporters: [debug]
```

먼저, 타겟 얼로케이터 서비스를 노출할 수 있도록 쿠버네티스에서 `port-forward`를
설정한다.

```shell
kubectl port-forward svc/otelcol-targetallocator -n opentelemetry 8080:80
```

여기서 `otelcol-targetallocator`는 `OpenTelemetryCollector` CR의 `metadata.name`
값에 `-targetallocator` 접미사를 붙인 것이고, `opentelemetry`는
`OpenTelemetryCollector` CR이 배포된 네임스페이스이다.

> [!TIP]
>
> 다음을 실행해 서비스 이름을 확인할 수도 있다.
>
> ```shell
> kubectl get svc -l app.kubernetes.io/component=opentelemetry-targetallocator -n <namespace>
> ```

다음으로, 타겟 얼로케이터에 등록된 작업(job) 목록을 확인한다.

```shell
curl localhost:8080/jobs | jq
```

출력 예시는 다음과 같은 모습이어야 한다.

```json
{
  "serviceMonitor/opentelemetry/sm-example/1": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F1/targets"
  },
  "serviceMonitor/opentelemetry/sm-example/2": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F2/targets"
  },
  "otel-collector": {
    "_link": "/jobs/otel-collector/targets"
  },
  "serviceMonitor/opentelemetry/sm-example/0": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets"
  },
  "podMonitor/opentelemetry/pm-example/0": {
    "_link": "/jobs/podMonitor%2Fopentelemetry%2Fpm-example%2F0/targets"
  }
}
```

여기서 `serviceMonitor/opentelemetry/sm-example/0`은 `ServiceMonitor`가 찾아낸
`Service` 포트 중 하나를 나타낸다.

- `opentelemetry`는 `ServiceMonitor` 리소스가 존재하는 네임스페이스이다.
- `sm-example`은 `ServiceMonitor`의 이름이다.
- `0`은 `ServiceMonitor`와 `Service` 사이에 매칭된 포트 엔드포인트 중 하나이다.

마찬가지로 `PodMonitor`는 `curl` 출력에서
`podMonitor/opentelemetry/pm-example/0`으로 나타난다.

이는 스크레이프 구성 검색(discovery)이 잘 동작하고 있다는 것을 알려주므로 좋은
소식이다!

`otel-collector` 항목이 궁금할 수도 있다. 이는 `OpenTelemetryCollector`
리소스(이름은 `otel-collector`)의 `spec.config.receivers.prometheusReceiver`에서
셀프 스크레이핑(self-scrape)이 활성화되어 있기 때문에 발생한다.

```yaml
prometheus:
  config:
    scrape_configs:
      - job_name: 'otel-collector'
        scrape_interval: 10s
        static_configs:
          - targets: ['0.0.0.0:8888']
```

`serviceMonitor/opentelemetry/sm-example/0`을 더 자세히 살펴보기 위해, 위
`_link` 출력 값에 대해 `curl`을 실행하면 어떤 스크레이프 대상이 검색되고 있는지
확인할 수 있다.

```shell
curl localhost:8080/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets | jq
```

출력 예시:

```json
{
  "otelcol-collector-0": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets?collector_id=otelcol-collector-0",
    "targets": [
      {
        "targets": ["10.244.0.11:8080"],
        "labels": {
          "__meta_kubernetes_endpointslice_port_name": "prom",
          "__meta_kubernetes_pod_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_endpointslice_port_protocol": "TCP",
          "__meta_kubernetes_endpointslice_address_target_name": "py-prometheus-app-575cfdd46-nfttj",
          "__meta_kubernetes_endpointslice_annotation_endpoints_kubernetes_io_last_change_trigger_time": "2024-06-21T20:01:37Z",
          "__meta_kubernetes_endpointslice_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_pod_name": "py-prometheus-app-575cfdd46-nfttj",
          "__meta_kubernetes_pod_controller_name": "py-prometheus-app-575cfdd46",
          "__meta_kubernetes_pod_label_app_kubernetes_io_name": "py-prometheus-app",
          "__meta_kubernetes_endpointslice_address_target_kind": "Pod",
          "__meta_kubernetes_pod_node_name": "otel-target-allocator-talk-control-plane",
          "__meta_kubernetes_pod_labelpresent_pod_template_hash": "true",
          "__meta_kubernetes_endpointslice_label_kubernetes_io_service_name": "py-prometheus-app",
          "__meta_kubernetes_endpointslice_annotationpresent_endpoints_kubernetes_io_last_change_trigger_time": "true",
          "__meta_kubernetes_service_name": "py-prometheus-app",
          "__meta_kubernetes_pod_ready": "true",
          "__meta_kubernetes_pod_labelpresent_app": "true",
          "__meta_kubernetes_pod_controller_kind": "ReplicaSet",
          "__meta_kubernetes_endpointslice_labelpresent_app": "true",
          "__meta_kubernetes_pod_container_image": "otel-target-allocator-talk:0.1.0-py-prometheus-app",
          "__address__": "10.244.0.11:8080",
          "__meta_kubernetes_service_label_app_kubernetes_io_name": "py-prometheus-app",
          "__meta_kubernetes_pod_uid": "495d47ee-9a0e-49df-9b41-fe9e6f70090b",
          "__meta_kubernetes_endpointslice_port": "8080",
          "__meta_kubernetes_endpointslice_label_endpointslice_kubernetes_io_managed_by": "endpointslice-controller.k8s.io",
          "__meta_kubernetes_endpointslice_label_app": "my-app",
          "__meta_kubernetes_service_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_pod_host_ip": "172.24.0.2",
          "__meta_kubernetes_namespace": "opentelemetry",
          "__meta_kubernetes_endpointslice_endpoint_conditions_serving": "true",
          "__meta_kubernetes_endpointslice_labelpresent_kubernetes_io_service_name": "true",
          "__meta_kubernetes_endpointslice_endpoint_conditions_ready": "true",
          "__meta_kubernetes_service_annotation_kubectl_kubernetes_io_last_applied_configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"my-app\",\"app.kubernetes.io/name\":\"py-prometheus-app\"},\"name\":\"py-prometheus-app\",\"namespace\":\"opentelemetry\"},\"spec\":{\"ports\":[{\"name\":\"prom\",\"port\":8080}],\"selector\":{\"app\":\"my-app\",\"app.kubernetes.io/name\":\"py-prometheus-app\"}}}\n",
          "__meta_kubernetes_endpointslice_endpoint_conditions_terminating": "false",
          "__meta_kubernetes_pod_container_port_protocol": "TCP",
          "__meta_kubernetes_pod_phase": "Running",
          "__meta_kubernetes_pod_container_name": "my-app",
          "__meta_kubernetes_pod_container_port_name": "prom",
          "__meta_kubernetes_pod_ip": "10.244.0.11",
          "__meta_kubernetes_service_annotationpresent_kubectl_kubernetes_io_last_applied_configuration": "true",
          "__meta_kubernetes_service_labelpresent_app": "true",
          "__meta_kubernetes_endpointslice_address_type": "IPv4",
          "__meta_kubernetes_service_label_app": "my-app",
          "__meta_kubernetes_pod_label_app": "my-app",
          "__meta_kubernetes_pod_container_port_number": "8080",
          "__meta_kubernetes_endpointslice_name": "py-prometheus-app-bwbvn",
          "__meta_kubernetes_pod_label_pod_template_hash": "575cfdd46",
          "__meta_kubernetes_endpointslice_endpoint_node_name": "otel-target-allocator-talk-control-plane",
          "__meta_kubernetes_endpointslice_labelpresent_endpointslice_kubernetes_io_managed_by": "true",
          "__meta_kubernetes_endpointslice_label_app_kubernetes_io_name": "py-prometheus-app"
        }
      }
    ]
  }
}
```

위 출력에서 `_link` 필드의 쿼리 매개변수 `collector_id`는 이 대상들이
`otelcol-collector-0`(해당 `OpenTelemetryCollector` 리소스를 위해 생성된
`StatefulSet`의 이름)에 속한다는 것을 나타낸다.

> [!NOTE]
>
> `/jobs` 엔드포인트에 대한 더 자세한 정보는
> [타겟 얼로케이터 readme](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/target-allocator/README.md#endpoints)를
> 참고한다.

### 타겟 얼로케이터가 활성화되어 있는가? Prometheus 서비스 디스커버리가 활성화되어 있는가? {#is-the-target-allocator-enabled-is-prometheus-service-discovery-enabled}

위 `curl` 명령이 예상한 `ServiceMonitor`와 `PodMonitor` 목록을 보여주지
않는다면, 해당 값을 채워주는 기능이 켜져 있는지 확인해야 한다.

기억해야 할 점은, `OpenTelemetryCollector` CR에 `targetAllocator` 섹션을
포함했다고 해서 그것이 활성화된다는 뜻은 아니라는 것이다. 명시적으로 활성화해야
한다. 또한
[Prometheus 서비스 디스커버리](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/target-allocator/README.md#discovery-of-prometheus-custom-resources)를
사용하려면 명시적으로 활성화해야 한다.

- `spec.targetAllocator.enabled`를 `true`로 설정한다.
- `spec.targetAllocator.prometheusCR.enabled`를 `true`로 설정한다.

그러면 `OpenTelemetryCollector` 리소스는 다음과 같은 모습이 된다.

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
```

전체 `OpenTelemetryCollector` 리소스 정의는
["실제로 메트릭이 스크레이핑되고 있는지 확인했는가?"](#do-you-know-if-metrics-are-actually-being-scraped)를
참고한다.

### ServiceMonitor(또는 PodMonitor) 셀렉터를 구성했는가? {#did-you-configure-a-servicemonitor-or-podmonitor-selector}

[`ServiceMonitor`](https://observability.thomasriley.co.uk/prometheus/configuring-prometheus/using-service-monitors/)
셀렉터를 구성했다면, 타겟 얼로케이터는
[`serviceMonitorSelector`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscrservicemonitorselector)에
설정한 값과 일치하는 `metadata.label`을 가진 `ServiceMonitor`만 찾는다는 뜻이다.

타겟 얼로케이터에 다음 예제와 같이
[`serviceMonitorSelector`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscrservicemonitorselector)를
구성했다고 가정한다.

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
      serviceMonitorSelector:
        matchLabels:
          app: my-app
```

`spec.targetAllocator.prometheusCR.serviceMonitorSelector.matchLabels` 값을
`app: my-app`으로 설정하면, `ServiceMonitor` 리소스도 `metadata.labels`에 동일한
값을 가지고 있어야 한다는 뜻이다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  labels:
    app: my-app
    release: prometheus
spec:
```

전체 `ServiceMonitor` 리소스 정의는
["실제로 메트릭이 스크레이핑되고 있는지 확인했는가?"](#do-you-know-if-metrics-are-actually-being-scraped)를
참고한다.

이 경우 `OpenTelemetryCollector` 리소스의
`prometheusCR.serviceMonitorSelector.matchLabels`는 앞의 예제에서 본 것처럼
`app: my-app` 레이블을 가진 `ServiceMonitor`만 찾는다.

`ServiceMonitor` 리소스에 해당 레이블이 없으면, 타겟 얼로케이터는 그
`ServiceMonitor`로부터 스크레이프 대상을 검색하지 못한다.

> [!TIP]
>
> [PodMonitor][]를 사용하는 경우에도 마찬가지다. 이 경우
> `serviceMonitorSelector` 대신 [`podMonitorSelector`][]를 사용하게 된다.

### serviceMonitorSelector와 podMonitorSelector 구성을 아예 빠뜨리지 않았는가? {#did-you-leave-out-the-servicemonitorselector-andor-podmonitorselector-configuration-altogether}

["ServiceMonitor 또는 PodMonitor 셀렉터를 구성했는가"](#did-you-configure-a-servicemonitor-or-podmonitor-selector)에서
언급했듯이, `serviceMonitorSelector`와 `podMonitorSelector`에 일치하지 않는 값을
설정하면 타겟 얼로케이터가 각각 `ServiceMonitor`와 `PodMonitor`로부터 스크레이프
대상을 검색하지 못하게 된다.

마찬가지로, `OpenTelemetryCollector` CR의
[`v1beta1`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/opentelemetrycollectors.md#opentelemetryiov1beta1)에서
이 구성을 아예 빠뜨리는 경우에도 타겟 얼로케이터가 `ServiceMonitor`와
`PodMonitor`로부터 스크레이프 대상을 검색하지 못하게 된다.

`OpenTelemetryOperator`의 `v1beta1`부터는 사용할 의도가 없더라도
`serviceMonitorSelector`와 `podMonitorSelector`를 다음과 같이 포함해야 한다.

```yaml
prometheusCR:
  enabled: true
  podMonitorSelector: {}
  serviceMonitorSelector: {}
```

이 구성은 모든 `PodMonitor`와 `ServiceMonitor` 리소스에 매칭된다는 뜻이다. 전체
`OpenTelemetryCollector` 정의는
["실제로 메트릭이 스크레이핑되고 있는지 확인했는가?"](#do-you-know-if-metrics-are-actually-being-scraped)를
참고한다.

### ServiceMonitor와 Service(또는 PodMonitor와 Pod)의 레이블, 네임스페이스, 포트가 일치하는가? {#do-your-labels-namespaces-and-ports-match-for-your-servicemonitor-and-your-service-or-podmonitor-and-your-pod}

`ServiceMonitor`는 다음 기준에 일치하는 쿠버네티스
[Service](https://kubernetes.io/docs/concepts/services-networking/service/)를
찾도록 구성된다.

- 레이블
- 네임스페이스(선택 사항)
- 포트(엔드포인트)

다음과 같은 `ServiceMonitor`가 있다고 가정한다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  labels:
    app: my-app
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - opentelemetry
  endpoints:
    - port: prom
      path: /metrics
    - port: py-client-port
      interval: 15s
    - port: py-server-port
```

앞의 `ServiceMonitor`는 다음 조건을 만족하는 서비스를 찾는다.

- `app: my-app` 레이블을 가진다
- `opentelemetry`라는 네임스페이스에 있다
- `prom`, `py-client-port`, _또는_ `py-server-port`라는 이름의 포트를 가진다

예를 들어 다음 `Service` 리소스는 앞의 조건과 일치하므로 `ServiceMonitor`에
검색된다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: prom
      port: 8080
```

다음 `Service` 리소스는 검색되지 않는다. `ServiceMonitor`가 `prom`,
`py-client-port`, _또는_ `py-server-port`라는 이름의 포트를 찾는데, 이 서비스의
포트 이름은 `bleh`이기 때문이다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: bleh
      port: 8080
```

> [!TIP]
>
> `PodMonitor`를 사용하는 경우에도 마찬가지지만, 레이블, 네임스페이스, 이름이
> 지정된 포트가 일치하는 쿠버네티스 파드를 찾는다는 점이 다르다.

[`podMonitorSelector`]:
  https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscr
[PodMonitor]:
  https://prometheus-operator.dev/docs/developer/getting-started/#using-podmonitors
