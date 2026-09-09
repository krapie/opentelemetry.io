---
title: 게이트웨이 배포 패턴
linkTitle: 게이트웨이 패턴
description:
  시그널을 먼저 단일 OTLP 엔드포인트로 보낸 후 백엔드로 보내는 이유와 방법을
  알아본다
aliases: [/docs/collector/deployment/gateway]
weight: 300
cSpell:ignore: hostnames loadbalancer loadbalancing resourcedetectionprocessor
default_lang_commit: ccb79745a6b30511661b7071ecf1e866fcd2a122
---

게이트웨이(gateway) 컬렉터 배포 패턴은 애플리케이션이나 다른 컬렉터가 텔레메트리
시그널을 단일 [OTLP](/docs/specs/otlp/) 엔드포인트로 전송하는 방식으로 구성된다.
이 엔드포인트는 예를 들어 Kubernetes 배포에서와 같이, 독립형(standalone)
서비스로 실행되는 하나 이상의 컬렉터 인스턴스가 제공한다. 일반적으로 클러스터별,
데이터 센터별, 또는 리전별로 엔드포인트가 제공된다.

일반적으로, 기본 제공되는(out-of-the-box) 로드 밸런서를 사용해 컬렉터 간 부하를
분산할 수 있다.

![게이트웨이 배포 개념도](../../img/otel-gateway-sdk.svg)

텔레메트리 데이터가 반드시 특정 컬렉터에서 처리되어야 하는 사용 사례의 경우,
2계층(tier) 설정을 사용한다. 첫 번째 계층의 컬렉터는 [트레이스 ID/서비스 이름
인식 로드 밸런싱 익스포터][lb-exporter]로 구성된 파이프라인을 가진다. 두 번째
계층에서는 각 컬렉터가 자신에게 특정하게 전달될 수 있는 텔레메트리를 수신하고
처리한다. 예를 들어, 첫 번째 계층에서 로드 밸런싱 익스포터를 사용해 [테일 샘플링
프로세서][tailsample-processor]로 구성된 두 번째 계층의 컬렉터로 데이터를
전송하면, 특정 트레이스의 모든 스팬이 테일 샘플링 정책이 적용되는 동일한 컬렉터
인스턴스로 도달하게 할 수 있다.

다음 다이어그램은 로드 밸런싱 익스포터를 사용하는 이 설정을 보여준다.

![로드 밸런싱 익스포터를 사용한 게이트웨이 배포](../../img/otel-gateway-lb-sdk.svg)

1. 애플리케이션에서는 SDK가 OTLP 데이터를 중앙 위치로 전송하도록 구성된다.
2. 컬렉터는 로드 밸런싱 익스포터를 사용해 시그널을 컬렉터 그룹으로 분산하도록
   구성된다.
3. 컬렉터는 텔레메트리 데이터를 하나 이상의 백엔드로 전송한다.

## 예제 {#examples}

다음 예제는 일반적인 구성 요소로 게이트웨이 컬렉터를 구성하는 방법을 보여준다.

### "기본 제공" 로드 밸런서로서의 NGINX {#nginx-as-an-out-of-the-box-load-balancer}

`collector1`, `collector2`, `collector3`라는 세 개의 컬렉터가 구성되어 있고
NGINX를 사용해 이들 간에 트래픽을 로드 밸런싱하려는 경우, 다음 구성을 사용할 수
있다.

```nginx
server {
    listen 4317 http2;
    server_name _;

    location / {
            grpc_pass      grpc://collector4317;
            grpc_next_upstream     error timeout invalid_header http_500;
            grpc_connect_timeout   2;
            grpc_set_header        Host            $host;
            grpc_set_header        X-Real-IP       $remote_addr;
            grpc_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 4318;
    server_name _;

    location / {
            proxy_pass      http://collector4318;
            proxy_redirect  off;
            proxy_next_upstream     error timeout invalid_header http_500;
            proxy_connect_timeout   2;
            proxy_set_header        Host            $host;
            proxy_set_header        X-Real-IP       $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

upstream collector4317 {
    server collector1:4317;
    server collector2:4317;
    server collector3:4317;
}

upstream collector4318 {
    server collector1:4318;
    server collector2:4318;
    server collector3:4318;
}
```

### 로드 밸런싱 익스포터 {#load-balancing-exporter}

중앙 집중식(centralized) 컬렉터 배포 패턴의 구체적인 예를 보려면, 먼저 로드
밸런싱 익스포터를 살펴본다. 이 익스포터에는 두 가지 주요 구성 필드가 있다.

- `resolver`는 다운스트림 컬렉터나 백엔드를 어디서 찾을지 결정한다. 여기서
  `static` 하위 키를 사용하면 컬렉터 URL을 수동으로 나열해야 한다. 지원되는 또
  다른 리졸버(resolver)는 DNS 리졸버로, 주기적으로 업데이트를 확인하고 IP 주소를
  해석(resolve)한다. 이 리졸버 유형의 경우, `hostname` 하위 키가 IP 주소 목록을
  얻기 위해 질의할 호스트 이름을 지정한다.
- `routing_key` 필드는 스팬을 특정 다운스트림 컬렉터로 라우팅한다. 이 필드를
  `traceID`로 설정하면, 로드 밸런싱 익스포터는 스팬의 `traceID`를 기준으로
  내보낸다. 반대로 `routing_key`에 `service`를 사용하면, 서비스 이름을 기준으로
  스팬을 내보낸다. 이 라우팅은 [스팬 메트릭 커넥터][spanmetrics-connector]와
  같은 커넥터를 사용할 때 유용한데, 서비스의 모든 스팬이 메트릭 수집을 위해
  동일한 다운스트림 컬렉터로 전송되어 정확한 집계를 보장하기 때문이다.

OTLP 엔드포인트를 제공하는 첫 번째 계층의 컬렉터는 다음과 같이 구성된다.

{{< tabpane text=true >}} {{% tab Static %}}

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  loadbalancing:
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      static:
        hostnames:
          - collector-1.example.com:4317
          - collector-2.example.com:5317
          - collector-3.example.com

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [loadbalancing]
```

{{% /tab %}} {{% tab DNS %}}

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  loadbalancing:
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      dns:
        hostname: collectors.example.com

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [loadbalancing]
```

{{% /tab %}} {{% tab "DNS with service" %}}

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  loadbalancing:
    routing_key: service
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      dns:
        hostname: collectors.example.com
        port: 5317

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [loadbalancing]
```

{{% /tab %}} {{< /tabpane >}}

로드 밸런싱 익스포터는 `otelcol_loadbalancer_num_backends`와
`otelcol_loadbalancer_backend_latency`를 포함한
[메트릭](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter#metrics)을
내보내며, 이를 사용해 OTLP 엔드포인트를 제공하는 컬렉터의 상태와 성능을
모니터링할 수 있다.

## 트레이드오프 {#trade-offs}

장점:

- 중앙에서 관리되는 자격 증명과 같은 관심사의 분리(separation of concerns)
- 중앙 집중식 정책 관리(예: 특정 로그 필터링 또는 샘플링)

단점:

- 유지 관리해야 할 요소가 하나 더 늘어나며 장애 지점이 될 수 있다(복잡성).
- 컬렉터가 연쇄적으로 연결된 경우 추가되는 지연 시간.
- 전체적으로 더 높은 리소스 사용량(비용).

[lb-exporter]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter
[tailsample-processor]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor
[spanmetrics-connector]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector

## 여러 컬렉터와 단일 작성자 원칙 {#multiple-collectors-and-the-single-writer-principle}

OTLP 내의 모든 메트릭 데이터 스트림은
[단일 작성자(single writer)](/docs/specs/otel/metrics/data-model/#single-writer)를
가져야 한다. 게이트웨이 구성에서 여러 컬렉터를 배포할 때는, 모든 메트릭 데이터
스트림이 단일 작성자와 전역적으로 고유한 아이덴티티(identity)를 갖도록 해야
한다.

### 잠재적인 문제 {#potential-problems}

동일한 데이터를 수정하거나 보고하는 여러 애플리케이션의 동시(concurrent) 접근은
데이터 손실이나 데이터 품질 저하로 이어질 수 있다. 예를 들어, 동일한 리소스에
대해 여러 소스에서 일관되지 않은 데이터가 나타날 수 있는데, 이는 리소스가
고유하게 식별되지 않아 서로 다른 소스가 서로를 덮어쓸 수 있기 때문이다.

이러한 현상이 발생하고 있는지 여부에 대한 단서를 제공할 수 있는 패턴이 데이터에
나타날 수 있다. 예를 들어, 육안으로 확인했을 때 동일한 시리즈에서 설명할 수 없는
간격이나 급격한 변화가 있다면 여러 컬렉터가 동일한 샘플을 전송하고 있다는 단서일
수 있다. 또한 백엔드에서 오류가 발생할 수도 있다. 예를 들어 Prometheus 백엔드의
경우 다음과 같다.

`Error on ingesting out-of-order samples`

이 오류는 동일한 대상(target)이 두 개의 작업(job)에 존재하며 타임스탬프 순서가
올바르지 않다는 것을 나타낼 수 있다. 예를 들면 다음과 같다.

- 타임스탬프 13:56:04, 값 `100`으로 `T1`에 수신된 메트릭 `M1`
- 타임스탬프 13:56:24, 값 `120`으로 `T2`에 수신된 메트릭 `M1`
- 타임스탬프 13:56:04, 값 `110`으로 `T3`에 수신된 메트릭 `M1`
- 타임스탬프 13:56:24, 값 `120`으로 수신된 메트릭 `M1`
- 타임스탬프 13:56:04, 값 `110`으로 수신된 메트릭 `M1`

### 모범 사례 {#best-practices}

- 서로 다른 Kubernetes 리소스에 레이블을 추가하려면
  [Kubernetes 속성 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor)를
  사용한다.
- 호스트에서 리소스 정보를 감지하고 리소스 메타데이터를 수집하려면
  [리소스 감지기 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourcedetectionprocessor/README.md)를
  사용한다.

## 다음 단계 {#next-steps}

견고하고 확장 가능한 컬렉터 아키텍처를 만들기 위해 에이전트 패턴과 게이트웨이
패턴을 [결합](/docs/collector/deploy/other/agent-to-gateway/)하는 방법을
알아본다.
