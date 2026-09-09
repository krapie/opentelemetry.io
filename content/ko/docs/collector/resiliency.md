---
title: 복원력
description: 복원력 있는 OTel 컬렉터 파이프라인을 구성하는 방법이다.
default_lang_commit: 4edfbfc2ff38123678ca63eca95de94ede457623
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 텔레메트리를 처리하고 내보내는 과정에서
데이터 손실을 최소화하도록 구성 요소와 설정을 설계했다. 하지만
복원력(resilience) 있는 옵저버빌리티(observability) 파이프라인을 위해서는 데이터
손실이 발생할 수 있는 시나리오와 이를 완화하는 방법을 이해하는 것이 중요하다.

## 컬렉터 복원력 이해하기 {#understanding-collector-resilience}

복원력 있는 컬렉터는 불리한 조건에 직면하더라도 텔레메트리 데이터 흐름과 처리
기능을 유지하여, 전체 옵저버빌리티 파이프라인이 계속 정상적으로 동작하도록
보장한다.

컬렉터의 복원력은 주로 설정된 엔드포인트(트레이스, 메트릭, 로그의 목적지)를
사용할 수 없게 되거나 컬렉터 인스턴스 자체가 크래시(crash)와 같은 문제를 겪을 때
데이터를 어떻게 처리하는지를 중심으로 한다.

## 전송 큐(메모리 내 버퍼링) {#sending-queue-in-memory-buffering}

컬렉터의 익스포터에 내장된 가장 기본적인 형태의 복원력은 전송 큐(sending
queue)이다.

- 동작 방식: 익스포터를 설정하면 보통 다운스트림 엔드포인트로 데이터를 보내기
  전에 메모리에 데이터를 버퍼링하는 전송 큐가 포함된다. 엔드포인트를 사용할 수
  있으면 데이터는 빠르게 통과한다.
- 엔드포인트를 사용할 수 없을 때의 처리: 네트워크 문제나 백엔드 재시작 등으로
  엔드포인트를 사용할 수 없게 되면, 익스포터는 즉시 데이터를 보낼 수 없다. 이때
  데이터를 버리는 대신 메모리 내 전송 큐에 추가한다.
- 재시도 메커니즘: 컬렉터는 지수 백오프(exponential backoff)와 지터(jitter)를
  사용하는 재시도 메커니즘을 사용한다. 대기 간격을 둔 후 버퍼링된 데이터를
  전송하기를 반복적으로 시도한다. 기본값으로 최대 5분까지 재시도한다.
- 데이터 손실 시나리오:
  - 큐가 가득 찬 경우: 메모리 내 큐는 설정 가능한 크기를 가진다(기본값은 보통
    1000개의 배치/요청이다). 엔드포인트를 계속 사용할 수 없는 상태에서 새
    데이터가 계속 들어오면 큐가 가득 찰 수 있다. 큐가 가득 차면, 컬렉터가
    메모리를 모두 소진하지 않도록 들어오는 데이터를 버린다.
  - 재시도 타임아웃: 엔드포인트를 설정된 최대 재시도 기간(기본값 5분)보다 오래
    사용할 수 없으면, 컬렉터는 큐에서 가장 오래된 데이터에 대한 재시도를 멈추고
    이를 버린다.
- 설정: 익스포터 설정에서 큐 크기와 재시도 동작을 설정할 수 있다.

  ```yaml
  exporters:
    otlp:
      endpoint: otlp.example.com:4317
      sending_queue:
        storage: file_storage
        queue_size: 5_000 # Increase queue size (default 1000)
      retry_on_failure:
        initial_interval: 5s
        max_interval: 30s
        max_elapsed_time: 10m # Increase max retry time (default 300s)
  ```

> [!TIP] 원격 익스포터에는 전송 큐를 사용한다
>
> 네트워크로 데이터를 보내는 모든 익스포터에 전송 큐를 활성화한다. 예상 데이터
> 양, 사용 가능한 컬렉터 메모리, 엔드포인트의 허용 가능한 다운타임(downtime)에
> 따라 `queue_size`와 `max_elapsed_time`을 조정한다. 큐
> 메트릭(`otelcol_exporter_queue_size`, `otelcol_exporter_queue_capacity`)을
> 모니터링한다.

## 영구 저장소(쓰기 전용 로그, WAL) {#persistent-storage-write-ahead-log---wal}

컬렉터 인스턴스 자체가 크래시하거나 재시작될 때 데이터 손실을 방지하려면,
`file_storage` 익스텐션(extension)을 사용해 전송 큐에 영구 저장소(persistent
storage)를 활성화할 수 있다.

- 동작 방식: 전송 큐는 단순히 메모리에 버퍼링하는 대신, 내보내기를 시도하기 전에
  디스크에 있는 쓰기 전용 로그(Write-Ahead Log, WAL)에 데이터를 기록한다.
- 컬렉터 크래시 처리: 큐에 데이터가 있는 상태에서 컬렉터가 크래시하면, 데이터는
  디스크에 유지된다. 컬렉터가 재시작되면 WAL에서 데이터를 읽어 엔드포인트로
  전송을 다시 시도한다.
- 데이터 손실 시나리오: 디스크에 장애가 발생하거나 공간이 부족해지면, 또는
  컬렉터가 재시작된 후에도 엔드포인트를 재시도 한도를 넘어 계속 사용할 수 없으면
  데이터 손실이 여전히 발생할 수 있다. 이때의 보장 수준은 전용 메시지 큐(message
  queue)만큼 강력하지 않을 수 있다.
- 설정:
  1.  `file_storage` 익스텐션을 정의한다.
  2.  익스포터의 `sending_queue` 설정에서 저장소 ID를 참조한다.

  ```yaml
  extensions:
    file_storage: # Define the extension instance
      directory: /var/lib/otelcol/storage # Choose a persistent directory

  exporters:
    otlp:
      endpoint: otlp.example.com:4317
      sending_queue:
        storage: file_storage # Reference the storage extension instance

  service:
    extensions: [file_storage] # Enable the extension in the service pipeline
    pipelines:
      traces:
        receivers: [otlp]
        exporters: [otlp]
  ```

> [!TIP] 선택한 컬렉터에는 WAL을 사용한다
>
> 컬렉터 크래시로 인한 데이터 손실을 용납할 수 없는 중요한 컬렉터(예: 게이트웨이
> 인스턴스나 중요한 데이터를 수집하는 에이전트)에는 영구 저장소를 사용한다.
> 선택한 디렉터리에 충분한 디스크 공간과 적절한 권한이 있는지 확인한다.

## 메시지 큐 {#message-queues}

특히 서로 다른 컬렉터 계층 사이(예: 에이전트에서 게이트웨이로)나 인프라와 벤더
백엔드 사이에서 최고 수준의 복원력을 얻으려면, Kafka와 같은 전용 메시지
큐(message queue)를 도입할 수 있다.

- 동작 방식: 한 컬렉터 인스턴스(에이전트)는 Kafka 익스포터를 사용해 Kafka
  토픽(topic)으로 데이터를 내보낸다. 다른 컬렉터 인스턴스(게이트웨이)는 Kafka
  리시버를 사용해 해당 Kafka 토픽에서 데이터를 소비한다.
- 엔드포인트/컬렉터를 사용할 수 없을 때의 처리:
  - 소비하는 컬렉터(게이트웨이)가 다운되면, 메시지는 (Kafka의 보존 한도까지)
    Kafka 토픽에 그대로 쌓인다. Kafka가 살아 있는 한 생산하는 컬렉터(에이전트)는
    영향을 받지 않는다.
  - 생산하는 컬렉터(에이전트)가 다운되면 새 데이터는 큐에 들어오지 않지만,
    소비자는 기존 메시지 처리를 계속할 수 있다.
  - Kafka 자체가 다운되면, 생산하는 컬렉터는 Kafka로 향하는 데이터를 버퍼링하기
    위해 (WAL을 곁들인 전송 큐와 같은) 자체 복원력 메커니즘이 필요하다.
- 데이터 손실 시나리오: 데이터 손실은 주로 Kafka 자체(클러스터 장애, 토픽 잘못된
  설정, 데이터 만료)나, 적절한 로컬 버퍼링 없이 프로듀서(producer)가 Kafka로
  전송하지 못하는 경우와 관련이 있다.
- 설정:
  - _에이전트 컬렉터 설정(프로듀서):_

    ```yaml
    exporters:
      kafka:
        brokers: ['kafka-broker1:9092', 'kafka-broker2:9092']
        topic: otlp_traces

    receivers:
      otlp:
        protocols:
          grpc:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [kafka]
    ```

  - _게이트웨이 컬렉터 설정(컨슈머):_

    ```yaml
    receivers:
      kafka:
        brokers: ['kafka-broker1:9092', 'kafka-broker2:9092']
        topic: otlp_traces
        initial_offset: earliest # Process backlog

    exporters:
      otlp:
        endpoint: otlp.example.com:4317
        # Consider queue/retry for exporting *from* Gateway to Backend

    service:
      pipelines:
        traces:
          receivers: [kafka]
          exporters: [otlp]
    ```

> [!TIP] 중요한 홉(hop)에는 메시지 큐를 사용한다
>
> 특히 네트워크 경계를 넘나드는(예: 데이터 센터 간, 가용 영역(availability zone)
> 간, 또는 클라우드 벤더로) 높은 내구성이 필요한 중요한 데이터 경로에는 메시지
> 큐를 사용한다. 이 방식은 Kafka와 같은 시스템에 내장된 견고한 복원력을
> 활용하지만, 운영 복잡성이 늘어나고 메시지 큐 시스템을 관리하는 전문성이
> 필요하다.

## 데이터 손실이 발생하는 상황 {#circumstances-of-data-loss}

다음과 같은 상황에서 데이터 손실이 발생할 수 있다.

1.  네트워크를 사용할 수 없음 + 타임아웃: `retry_on_failure` 설정에 지정된
    `max_elapsed_time`보다 오래 다운스트림 엔드포인트를 사용할 수 없는 경우
2.  네트워크를 사용할 수 없음 + 큐 오버플로우: 다운스트림 엔드포인트를 사용할 수
    없는 상태에서 엔드포인트가 복구되기 전에 전송 큐(메모리 내 또는 영구)가 가득
    차는 경우. 새 데이터는 버려진다.
3.  컬렉터 크래시(지속성 없음): 컬렉터 인스턴스가 크래시하거나 종료되었는데,
    메모리 내 전송 큐만 사용하고 있던 경우. 메모리에 있던 데이터가 유실된다.
4.  영구 저장소 장애: `file_storage` 익스텐션이 사용하는 디스크에 장애가
    발생하거나 공간이 부족해지는 경우
5.  메시지 큐 장애: 외부 메시지 큐(예: Kafka)에 장애나 데이터 손실 이벤트가
    발생했는데, 생산하는 컬렉터에 적절한 로컬 버퍼링이 없는 경우
6.  잘못된 설정: 익스포터나 리시버가 잘못 설정되어 데이터 흐름을 막는 경우
7.  복원력 비활성화: 설정에서 전송 큐나 재시도 메커니즘이 명시적으로 비활성화된
    경우

## 데이터 손실 방지를 위한 권장 사항 {#recommendations-for-preventing-data-loss}

데이터 손실을 최소화하고 신뢰할 수 있는 텔레메트리 데이터 수집을 보장하려면 다음
권장 사항을 따른다.

1.  항상 전송 큐 사용하기: 네트워크로 데이터를 보내는 익스포터에
    `sending_queue`를 활성화한다.
2.  컬렉터 메트릭 모니터링하기: 잠재적인 문제를 조기에 발견하기 위해
    `otelcol_exporter_queue_size`, `otelcol_exporter_queue_capacity`,
    `otelcol_exporter_send_failed_spans`(그리고 메트릭/로그에 대응하는 메트릭)를
    적극적으로 모니터링한다.
3.  큐 크기 및 재시도 조정하기: 예상 부하, 메모리/디스크 리소스, 엔드포인트의
    허용 가능한 다운타임에 따라 `queue_size`와 `retry_on_failure` 매개변수를
    조정한다.
4.  영구 저장소(WAL) 사용하기: 컬렉터 재시작 중 데이터 손실을 용납할 수 없는
    에이전트나 게이트웨이의 경우, 전송 큐에 `file_storage` 익스텐션을 설정한다.
5.  메시지 큐 고려하기: 네트워크 세그먼트 전반에 걸쳐 최대 내구성을 확보하거나
    컬렉터 계층을 분리하려면, 운영 오버헤드(overhead)가 감당할 만하다면 Kafka와
    같은 관리형 메시지 큐를 사용한다.
6.  적절한 배포 패턴 사용하기:
    - 에이전트 + 게이트웨이 아키텍처를 채택한다. 에이전트는 로컬 수집을
      담당하고, 게이트웨이는 처리, 배치(batching), 복원력 있는 내보내기를
      담당한다.
    - 복원력 확보 노력(큐, WAL, Kafka)을 네트워크 홉에 집중한다. 에이전트 ->
      게이트웨이, 게이트웨이 -> 백엔드.
    - 애플리케이션(SDK)과 로컬 에이전트(사이드카/DaemonSet) 사이의 복원력은 로컬
      네트워킹이 신뢰할 수 있기 때문에 대체로 덜 중요하다. 여기에 큐를 추가하면
      에이전트를 사용할 수 없을 때 오히려 애플리케이션에 부정적인 영향을 줄 수
      있다.

이런 메커니즘을 이해하고 적절한 설정을 적용하면, 오픈텔레메트리 컬렉터 배포의
복원력을 크게 향상시키고 데이터 손실을 최소화할 수 있다.
