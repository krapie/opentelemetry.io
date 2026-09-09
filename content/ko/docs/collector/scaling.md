---
title: 컬렉터 스케일링
weight: 26
cSpell:ignore: fluentd Linkerd loadbalancer loadbalancing sharded statefulset
default_lang_commit: 30b7dbbdd94cec0b2a0c99317272b103315518bf
---

오픈텔레메트리(OpenTelemetry) 컬렉터로 옵저버빌리티(observability) 파이프라인을
계획할 때는, 텔레메트리 수집량이 늘어남에 따라 파이프라인을
스케일링(scaling)하는 방법을 고려해야 한다.

다음 절에서는 어떤 구성 요소를 스케일링할지, 스케일 업(scale up)할 시점을 어떻게
판단할지, 그리고 계획을 어떻게 실행할지 논의하며 계획 단계를 안내한다.

## 무엇을 스케일링할 것인가 {#what-to-scale}

오픈텔레메트리 컬렉터는 모든 텔레메트리 시그널(signal) 유형을 단일 바이너리에서
처리하지만, 실제로는 유형마다 스케일링 요구 사항이 다를 수 있고 서로 다른
스케일링 전략이 필요할 수 있다. 먼저 워크로드(workload)를 살펴보고 어떤 시그널
유형이 가장 큰 부하 비중을 차지할 것으로 예상되는지, 그리고 컬렉터가 어떤 포맷을
수신할 것으로 예상되는지 파악하는 것부터 시작한다. 예를 들어,
스크레이핑(scraping) 클러스터를 스케일링하는 것은 로그 리시버를 스케일링하는
것과 크게 다르다. 워크로드가 얼마나 탄력적인지도 고려한다. 하루 중 특정 시간대에
부하가 몰리는가, 아니면 24시간 내내 부하가 비슷한가? 이 정보를 모으고 나면
무엇을 스케일링해야 하는지 파악할 수 있다.

예를 들어 스크레이핑해야 할 Prometheus 엔드포인트가 수백 개 있고, fluentd
인스턴스에서 매분 1테라바이트의 로그가 들어오며, 최신 마이크로서비스에서 OTLP
포맷으로 애플리케이션 메트릭과 트레이스가 도착하는 상황을 가정해보자. 이런
시나리오에서는 각 시그널을 개별적으로 스케일링할 수 있는 아키텍처가 필요하다.
Prometheus 리시버를 스케일링하려면 어떤 스크레이퍼가 어떤 엔드포인트를 담당할지
결정하기 위해 스크레이퍼 간 조정이 필요하다. 반면 스테이트리스(stateless)한 로그
리시버는 필요에 따라 수평으로 스케일링할 수 있다. 메트릭과 트레이스를 위한 OTLP
리시버를 세 번째 컬렉터 클러스터에 두면 장애를 격리하고, 바쁜 파이프라인을
재시작할 걱정 없이 더 빠르게 반복 작업을 할 수 있다. OTLP 리시버는 모든
텔레메트리 유형의 유입(ingestion)을 지원하므로, 애플리케이션 메트릭과 트레이스를
동일한 인스턴스에 유지하면서 필요할 때 수평으로 스케일링할 수 있다.

## 언제 스케일링할 것인가 {#when-to-scale}

다시 말하지만, 스케일 업 또는 스케일 다운할 시점을 결정하려면 워크로드를
이해해야 하지만, 컬렉터가 내보내는 몇 가지 메트릭이 언제 조치를 취해야 할지에
대한 좋은 힌트를 줄 수 있다.

파이프라인에 memory_limiter 프로세서가 포함되어 있을 때 컬렉터가 줄 수 있는
유용한 힌트 중 하나는 `otelcol_processor_refused_spans` 메트릭이다. 이
프로세서를 사용하면 컬렉터가 사용할 수 있는 메모리 양을 제한할 수 있다. 컬렉터는
이 프로세서에 설정된 최대치보다 조금 더 많은 메모리를 소비할 수도 있지만, 결국
memory_limiter가 새 데이터가 파이프라인을 통과하지 못하도록 차단하고 이 사실을
해당 메트릭에 기록한다. 동일한 메트릭이 다른 모든 텔레메트리 데이터 유형에도
존재한다. 데이터가 파이프라인 유입을 너무 자주 거부당한다면 컬렉터 클러스터를
스케일 업해야 할 가능성이 높다. 노드 전체의 메모리 소비량이 이 프로세서에 설정된
한도보다 충분히 낮아지면 스케일 다운할 수 있다.

주시해야 할 또 다른 메트릭 집합은 익스포터의 큐(queue) 크기와 관련된 것으로,
`otelcol_exporter_queue_capacity`와 `otelcol_exporter_queue_size`이다. 컬렉터는
데이터를 전송할 워커(worker)가 사용 가능해질 때까지 메모리에 데이터를
큐잉(queue)한다. 워커 수가 부족하거나 백엔드가 너무 느리면 데이터가 큐에 쌓이기
시작한다. 큐가 용량에 도달하면(`otelcol_exporter_queue_size` >
`otelcol_exporter_queue_capacity`) 데이터를 거부한다
(`otelcol_exporter_enqueue_failed_spans`). 워커를 추가하면 컬렉터가 더 많은
데이터를 내보내게 되는 경우가 많은데, 이는 반드시 원하는 결과가 아닐 수
있다([스케일링하지 않아야 할 때](#when-not-to-scale) 참고). 일반적인 지침은 큐
크기를 모니터링하여 용량의 60~70%에 도달하면 스케일 업을 고려하고, 지속적으로
낮으면 스케일 다운하되, 복원력(resilience)을 위해 예를 들어 3개와 같은 최소
레플리카(replica) 수를 유지하는 것이다.

사용하려는 구성 요소에 익숙해지는 것도 중요한데, 구성 요소마다 다른 메트릭을
생성할 수 있기 때문이다. 예를 들어
[로드 밸런싱(load-balancing) 익스포터는 내보내기 작업에 대한 타이밍 정보를 기록하며](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter#metrics),
이를 `otelcol_loadbalancer_backend_latency` 히스토그램의 일부로 노출한다. 이
정보를 추출하면 모든 백엔드가 요청을 처리하는 데 비슷한 시간이 걸리는지 확인할
수 있다. 특정 백엔드만 느리다면 컬렉터 외부의 문제를 나타내는 것일 수 있다.

Prometheus 리시버처럼 스크레이핑을 수행하는 리시버의 경우, 모든 대상의
스크레이핑을 마치는 데 걸리는 시간이 스크레이프 간격(scrape interval)에 자주
위험할 정도로 가까워지면 스크레이핑을 스케일링하거나 샤딩(sharding)해야 한다.
이런 상황이 되면 스크레이퍼를, 보통은 컬렉터의 새 인스턴스를 추가할 시점이다.

### 스케일링하지 않아야 할 때 {#when-not-to-scale}

언제 스케일링해야 하는지 아는 것 못지않게, 어떤 징후가 스케일링 작업이 아무런
이점도 가져오지 않음을 나타내는지 이해하는 것도 중요하다. 한 가지 예는
텔레메트리 데이터베이스가 부하를 따라가지 못하는 경우로, 데이터베이스를 스케일
업하지 않고 클러스터에 컬렉터를 추가해도 도움이 되지 않는다. 마찬가지로 컬렉터와
백엔드 사이의 네트워크 연결이 포화 상태일 때 컬렉터를 더 추가하면 오히려 해로운
부작용을 일으킬 수 있다.

이런 상황을 포착하는 한 가지 방법은 역시 `otelcol_exporter_queue_size`와
`otelcol_exporter_queue_capacity` 메트릭을 살펴보는 것이다. 큐 크기가 큐 용량에
계속 가까워진다면, 이는 데이터를 내보내는 속도가 데이터를 수신하는 속도보다
느리다는 신호이다. 큐 용량을 늘려볼 수 있는데, 이렇게 하면 컬렉터가 더 많은
메모리를 소비하게 되지만 텔레메트리 데이터를 영구적으로 유실하지 않도록 백엔드에
숨 돌릴 여유를 줄 수 있다. 하지만 큐 용량을 계속 늘려도 큐 크기가 같은 비율로
계속 증가한다면, 이는 컬렉터 외부를 살펴봐야 한다는 신호이다. 이럴 때 워커를 더
추가하는 것은 도움이 되지 않는다는 점도 중요한데, 이미 높은 부하로 어려움을 겪고
있는 시스템에 부담을 더 가중시킬 뿐이기 때문이다.

백엔드에 문제가 있을 수 있음을 나타내는 또 다른 신호는
`otelcol_exporter_send_failed_spans` 메트릭의 증가로, 이는 백엔드로 데이터를
전송하는 데 영구적으로 실패했음을 나타낸다. 이런 현상이 지속적으로 발생할 때
컬렉터를 스케일 업하면 상황을 더 악화시킬 가능성이 크다.

## 어떻게 스케일링할 것인가 {#how-to-scale}

이 시점에서 파이프라인의 어느 부분을 스케일링해야 하는지 알게 되었다. 스케일링과
관련해서 구성 요소는 스테이트리스(stateless), 스크레이퍼, 스테이트풀(stateful)
세 가지 유형으로 나뉜다.

대부분의 컬렉터 구성 요소는 스테이트리스하다. 메모리에 일부 상태를 유지하더라도
스케일링 목적에서는 중요하지 않다.

Prometheus 리시버와 같은 스크레이퍼는 외부 위치에서 텔레메트리 데이터를
가져오도록 설정된다. 이후 리시버는 대상을 하나씩 스크레이핑하여 파이프라인에
데이터를 넣는다.

테일 샘플링(tail sampling) 프로세서와 같은 구성 요소는 자신의 작업을 위해 관련
상태를 메모리에 유지하기 때문에 쉽게 스케일링할 수 없다. 이런 구성 요소는 스케일
업하기 전에 신중한 고려가 필요하다.

### 스테이트리스 컬렉터 스케일링과 로드 밸런서 사용 {#scaling-stateless-collectors-and-using-load-balancers}

좋은 소식은 대부분의 경우 컬렉터를 스케일링하는 것이 쉽다는 점이다. 새
레플리카를 추가하고 로드 밸런서(load balancer)를 사용해 트래픽을 분산시키기만
하면 되기 때문이다.

로드 밸런서는 다음이 필요할 때 필수적이다.

- 단일 인스턴스에 부하가 몰리지 않도록 스테이트리스 컬렉터의 여러 인스턴스에
  걸쳐 들어오는 텔레메트리 트래픽을 분산해야 할 때
- 수집 파이프라인의 가용성과 장애 허용성(fault tolerance)을 개선해야 할 때.
  컬렉터 인스턴스 하나가 장애를 일으키면 로드 밸런서가 정상 인스턴스로 트래픽을
  리디렉션할 수 있다.
- 수요에 따라 컬렉터 계층을 수평으로 스케일링해야 할 때

쿠버네티스(Kubernetes) 환경에서 운영할 때는 Istio나 Linkerd 같은 서비스
메시(service mesh), 또는 클라우드 제공업체의 로드 밸런서가 제공하는 견고하고
기성품(off-the-shelf)인 로드 밸런싱 및 속도 제한(rate-limiting) 솔루션을
활용한다. 이런 시스템은 기본적인 부하 분산을 넘어서는 트래픽 관리, 복원력,
옵저버빌리티 관련 성숙한 기능을 제공한다.

OTLP에서 흔한 시나리오인 gRPC로 데이터를 수신할 때는 gRPC를 이해하는 로드
밸런서(L7 로드 밸런서)를 사용한다. 일반적인 L4 로드 밸런서는 단일 백엔드 컬렉터
인스턴스에 지속적인 연결을 맺을 수 있는데, 이렇게 되면 클라이언트가 항상 같은
백엔드 컬렉터에만 접속하게 되어 스케일링의 이점이 사라진다. 그럼에도 신뢰성을
염두에 두고 수집 파이프라인을 분리하는 것을 고려해야 한다. 예를 들어 워크로드가
쿠버네티스에서 실행 중이라면 DaemonSet을 사용해 워크로드와 동일한 물리 노드에
컬렉터를 두고, 데이터를 스토리지로 보내기 전에 전처리를 담당하는 원격 중앙
컬렉터를 별도로 둘 수 있다. 노드 수가 적고 파드(pod) 수가 많을 때는 사이드카를
사용하는 것이 더 합리적일 수 있다. gRPC 전용 로드 밸런서 없이도 컬렉터 계층 간
gRPC 연결에 대해 더 나은 로드 밸런싱을 얻을 수 있기 때문이다. 또한 사이드카를
사용하면 DaemonSet 파드 하나가 장애를 일으켰을 때 노드의 모든 파드에 중요한 구성
요소가 함께 중단되는 것을 방지할 수 있다.

사이드카(sidecar) 패턴은 워크로드 파드에 컨테이너를 추가하는 것으로 구성된다.
[OpenTelemetry Operator](/docs/platforms/kubernetes/operator/)를 사용하면 이를
자동으로 추가할 수 있다. 이를 위해서는 오픈텔레메트리 컬렉터 CR이 필요하며,
Operator가 사이드카를 주입하도록 지시하려면 PodSpec 또는 Pod에
어노테이션(annotation)을 추가해야 한다.

```yaml
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: sidecar-for-my-workload
spec:
  mode: sidecar
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:

    exporters:
      # Note: Prior to v0.86.0 use the `logging` instead of `debug`.
      debug:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [debug]
---
apiVersion: v1
kind: Pod
metadata:
  name: my-microservice
  annotations:
    sidecar.opentelemetry.io/inject: 'true'
spec:
  containers:
    - name: my-microservice
      image: my-org/my-microservice:v0.0.0
      ports:
        - containerPort: 8080
          protocol: TCP
```

Operator를 건너뛰고 사이드카를 수동으로 추가하고 싶다면 다음은 그 예시이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-microservice
spec:
  containers:
    - name: my-microservice
      image: my-org/my-microservice:v0.0.0
      ports:
        - containerPort: 8080
          protocol: TCP
    - name: sidecar
      image: ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:0.69.0
      ports:
        - containerPort: 8888
          name: metrics
          protocol: TCP
        - containerPort: 4317
          name: otlp-grpc
          protocol: TCP
      args:
        - --config=/conf/collector.yaml
      volumeMounts:
        - mountPath: /conf
          name: sidecar-conf
  volumes:
    - name: sidecar-conf
      configMap:
        name: sidecar-for-my-workload
        items:
          - key: collector.yaml
            path: collector.yaml
```

### 스크레이퍼 스케일링 {#scaling-the-scrapers}

host_metrics 리시버나 prometheus 리시버처럼 일부 리시버는 파이프라인에 넣을
텔레메트리 데이터를 능동적으로 가져온다. 호스트 메트릭을 가져오는 작업은
일반적으로 스케일 업할 대상이 아니지만, Prometheus 리시버의 경우 수천 개
엔드포인트를 스크레이핑하는 작업을 나눠야 할 수도 있다. 그리고 동일한 설정으로
인스턴스를 단순히 더 추가할 수는 없는데, 클러스터의 모든 컬렉터가 서로 동일한
엔드포인트를 스크레이핑하려고 시도해서 순서가 어긋난 샘플(out-of-order
samples)과 같은 더 많은 문제를 일으키기 때문이다.

해결책은 엔드포인트를 컬렉터 인스턴스별로 샤딩(sharding)하여, 컬렉터의
레플리카를 추가하면 각 레플리카가 서로 다른 엔드포인트 집합을 담당하도록 하는
것이다.

이를 수행하는 한 가지 방법은 컬렉터마다 설정 파일을 하나씩 두어, 각 컬렉터가
자신과 관련된 엔드포인트만 검색하도록 하는 것이다. 예를 들어 각 컬렉터가 하나의
쿠버네티스 네임스페이스나 워크로드의 특정 레이블(label)을 담당하게 할 수 있다.

Prometheus 리시버를 스케일링하는 또 다른 방법은
[Target Allocator](/docs/platforms/kubernetes/operator/target-allocator/)를
사용하는 것이다. 이는 OpenTelemetry Operator의 일부로 배포할 수 있는 추가
바이너리로, 주어진 설정에 대한 Prometheus 스크레이프 대상을 컬렉터 클러스터
전체에 분배한다. 다음과 같은 커스텀 리소스(Custom Resource, CR)를 사용하면
Target Allocator를 활용할 수 있다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: collector-with-ta
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs:
          - job_name: 'otel-collector'
            scrape_interval: 10s
            static_configs:
            - targets: [ '0.0.0.0:8888' ]

    exporters:
      # Note: Prior to v0.86.0 use the `logging` instead of `debug`.
      debug:

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: []
          exporters: [debug]
```

조정(reconciliation) 이후, OpenTelemetry Operator는 컬렉터의 설정을 다음과 같이
변환한다.

```yaml
exporters:
   # Note: Prior to v0.86.0 use the `logging` instead of `debug`.
   debug: null
 receivers:
   prometheus:
     config:
       global:
         scrape_interval: 1m
         scrape_timeout: 10s
         evaluation_interval: 1m
       scrape_configs:
       - job_name: otel-collector
         honor_timestamps: true
         scrape_interval: 10s
         scrape_timeout: 10s
         metrics_path: /metrics
         scheme: http
         follow_redirects: true
         http_sd_configs:
         - follow_redirects: false
           url: http://collector-with-ta-targetallocator:80/jobs/otel-collector/targets?collector_id=$POD_NAME
service:
   pipelines:
     metrics:
       exporters:
       - debug
       processors: []
       receivers:
       - prometheus
```

Operator가 `otel-collector` 스크레이프 설정에 `global` 섹션과 새로운
`http_sd_configs`를 추가하여, 자신이 프로비저닝한 Target Allocator 인스턴스를
가리키도록 한 것에 주목한다. 이제 컬렉터를 스케일링하려면 CR의 "replicas" 속성을
변경하면 되고, Target Allocator는 컬렉터 인스턴스(파드)마다 커스텀
`http_sd_config`를 제공하여 그에 맞게 부하를 분배한다.

### 스테이트풀 컬렉터 스케일링 {#scaling-stateful-collectors}

특정 구성 요소는 데이터를 메모리에 유지하기 때문에 스케일 업할 때 결과가 달라질
수 있다. 테일 샘플링(tail-sampling) 프로세서가 그런 경우로, 이 프로세서는 일정
기간 동안 스팬을 메모리에 유지하다가 트레이스가 완료된 것으로 간주될 때만 샘플링
결정을 평가한다. 레플리카를 추가해 컬렉터 클러스터를 스케일링하면 서로 다른
컬렉터가 동일한 트레이스에 속한 스팬을 받게 되어, 각 컬렉터가 해당 트레이스를
샘플링할지 여부를 각자 평가하게 되고 그 결과가 서로 다를 수 있다. 이런 동작은
트레이스에서 스팬이 누락되게 만들어 해당 트랜잭션(transaction)에서 일어난 일을
잘못 표현하게 된다.

서비스 메트릭을 생성하기 위해 span-to-metrics 프로세서를 사용할 때도 비슷한
상황이 발생한다. 서로 다른 컬렉터가 동일한 서비스와 관련된 데이터를 받으면,
서비스 이름을 기준으로 한 집계(aggregation)가 부정확해진다.

이를 해결하려면 테일 샘플링이나 span-to-metrics 처리를 수행하는 컬렉터 앞에 로드
밸런싱(load-balancing) 익스포터를 포함한 컬렉터 계층을 배포하면 된다. 로드
밸런싱 익스포터는 트레이스 ID나 서비스 이름을 일관되게 해시(hash)하여 해당
트레이스의 스팬을 받을 컬렉터 백엔드를 결정한다. 로드 밸런싱 익스포터가
쿠버네티스 헤드리스 서비스(headless service)와 같이 주어진 DNS A 레코드 뒤에
있는 호스트 목록을 사용하도록 설정할 수 있다. 해당 서비스를 지원하는
디플로이먼트(deployment)가 스케일 업 또는 스케일 다운되면 로드 밸런싱 익스포터는
결국 갱신된 호스트 목록을 확인하게 된다. 또는 로드 밸런싱 익스포터가 사용할 정적
호스트 목록을 직접 지정할 수도 있다. 레플리카 수를 늘려서 로드 밸런싱 익스포터로
설정된 컬렉터 계층을 스케일 업할 수 있다. 각 컬렉터가 서로 다른 시점에 DNS
쿼리를 실행할 수 있어, 잠시 동안 클러스터 뷰(view)에 차이가 생길 수 있다는 점에
유의한다. 탄력성이 매우 높은(highly-elastic) 환경에서는 간격(interval) 값을
낮춰서 클러스터 뷰의 차이가 짧은 기간에만 발생하도록 하는 것을 권장한다.

다음은 백엔드 정보의 입력으로 DNS A 레코드(observability 네임스페이스의
쿠버네티스 서비스 otelcol)를 사용하는 설정 예시이다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:

exporters:
  loadbalancing:
    protocol:
      otlp:
    resolver:
      dns:
        hostname: otelcol.observability.svc.cluster.local

service:
  pipelines:
    traces:
      receivers:
        - otlp
      processors: []
      exporters:
        - loadbalancing
```
