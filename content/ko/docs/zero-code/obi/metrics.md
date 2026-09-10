---
title: OBI가 내보내는 메트릭
linkTitle: 내보내는 메트릭
description:
  OBI가 내보낼 수 있는 애플리케이션, 런타임, 네트워크 메트릭에 대해 알아본다.
weight: 21
cSpell:ignore: eventloop gogc replicaset statefulset stddev
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

다음 표는 오픈텔레메트리(OpenTelemetry)와 Prometheus 형식 모두에서 내보내는
메트릭을 설명한다.

| 패밀리       | 이름(OTel)                            | 이름(Prometheus)                              | 유형      | 단위        | 설명                                                                                       |
| ------------ | ------------------------------------- | --------------------------------------------- | --------- | ----------- | ------------------------------------------------------------------------------------------ |
| 애플리케이션 | `http.client.request.duration`        | `http_client_request_duration_seconds`        | Histogram | seconds     | 클라이언트 측 HTTP 서비스 호출의 지속 시간                                                 |
| 애플리케이션 | `http.client.request.body.size`       | `http_client_request_body_size_bytes`         | Histogram | bytes       | 클라이언트가 보낸 HTTP 요청 본문의 크기                                                    |
| 애플리케이션 | `http.client.response.body.size`      | `http_client_response_body_size_bytes`        | Histogram | bytes       | 클라이언트가 보낸 HTTP 응답 본문의 크기                                                    |
| 애플리케이션 | `http.server.request.duration`        | `http_server_request_duration_seconds`        | Histogram | seconds     | 서버 측 HTTP 서비스 호출의 지속 시간                                                       |
| 애플리케이션 | `http.server.request.body.size`       | `http_server_request_body_size_bytes`         | Histogram | bytes       | 서버 측에서 수신한 HTTP 요청 본문의 크기                                                   |
| 애플리케이션 | `http.server.response.body.size`      | `http_server_response_body_size_bytes`        | Histogram | bytes       | 서버 측에서 수신한 HTTP 응답 본문의 크기                                                   |
| 애플리케이션 | `rpc.client.call.duration`            | `rpc_client_call_duration_seconds`            | Histogram | seconds     | 클라이언트 측 RPC 서비스 호출의 지속 시간                                                  |
| 애플리케이션 | `rpc.server.call.duration`            | `rpc_server_call_duration_seconds`            | Histogram | seconds     | 서버 측 RPC 서비스 호출의 지속 시간                                                        |
| 애플리케이션 | `db.client.operation.duration`        | `db_client_operation_duration_seconds`        | Histogram | seconds     | 데이터베이스 클라이언트 작업의 지속 시간(실험적)                                           |
| 애플리케이션 | `db.server.operation.duration`        | `db_server_operation_duration_seconds`        | Histogram | seconds     | 서버 측 Redis, Memcached, SQL 작업의 지속 시간(실험적)                                     |
| 애플리케이션 | `messaging.client.operation.duration` | `messaging_client_operation_duration_seconds` | Histogram | seconds     | Kafka, MQTT, NATS, AMQP 등 지원 시스템 전반의 메시징 클라이언트 작업의 지속 시간(실험적)   |
| 애플리케이션 | `messaging.process.duration`          | `messaging_process_duration_seconds`          | Histogram | seconds     | Kafka, MQTT, NATS, AMQP 등 지원 시스템 전반의 메시징 프로세스 작업의 지속 시간(실험적)     |
| 애플리케이션 | `gen_ai.client.operation.duration`    | `gen_ai_client_operation_duration_seconds`    | Histogram | seconds     | GenAI 클라이언트 작업의 지속 시간(실험적)                                                  |
| 애플리케이션 | `gen_ai.client.token.usage`           | `gen_ai_client_token_usage`                   | Histogram | 1           | 토큰 유형별로 레이블이 지정된, 소비된 GenAI 입력/출력 토큰 수(실험적)                      |
| Go 런타임    | `go.memory.limit`                     | `go_memory_limit_bytes`                       | Gauge     | bytes       | 계측된 Go 서비스에 구성된 런타임 메모리 제한                                               |
| Go 런타임    | `go.memory.gc.goal`                   | `go_memory_gc_goal_bytes`                     | Gauge     | bytes       | 현재 가비지 컬렉션 힙 목표                                                                 |
| Go 런타임    | `go.memory.gc.cycles`                 | `go_memory_gc_cycles_total`                   | Counter   | cycles      | 완료된 Go 가비지 컬렉션 사이클                                                             |
| Go 런타임    | `go.memory.gc.pause.duration`         | `go_memory_gc_pause_duration_seconds`         | Histogram | seconds     | 누적 stop-the-world 가비지 컬렉션 일시 중지 시간                                           |
| Go 런타임    | `go.memory.used`                      | `go_memory_used_bytes`                        | Gauge     | bytes       | 스택 및 기타 런타임 메모리 범주가 사용하는 메모리                                          |
| Go 런타임    | `go.memory.allocated`                 | `go_memory_allocated_bytes_total`             | Counter   | bytes       | 누적 할당된 힙 바이트                                                                      |
| Go 런타임    | `go.memory.allocations`               | `go_memory_allocations_total`                 | Counter   | allocations | 누적 힙 할당 횟수                                                                          |
| Go 런타임    | `go.cpu.time`                         | `go_cpu_time_seconds_total`                   | Counter   | seconds     | 상태별 누적 런타임 CPU 시간                                                                |
| Go 런타임    | `go.goroutine.count`                  | `go_goroutine_count`                          | Gauge     | goroutines  | 현재 고루틴 수                                                                             |
| Go 런타임    | `go.processor.limit`                  | `go_processor_limit`                          | Gauge     | threads     | 현재 `GOMAXPROCS` 값                                                                       |
| Go 런타임    | `go.config.gogc`                      | `go_config_gogc_percent`                      | Gauge     | percent     | 현재 `GOGC` 힙 목표 백분율                                                                 |
| Go 런타임    | `go.schedule.duration`                | `go_schedule_duration_seconds`                | Histogram | seconds     | 누적 runnable에서 running 상태로의 고루틴 지연                                             |
| JVM 런타임   | `jvm.memory.used`                     | `jvm_memory_used_bytes`                       | Gauge     | bytes       | 메모리 유형과 풀별로 레이블이 지정된, 현재 사용 중인 JVM 메모리                            |
| JVM 런타임   | `jvm.memory.committed`                | `jvm_memory_committed_bytes`                  | Gauge     | bytes       | 메모리 유형과 풀별로 레이블이 지정된, 현재 커밋된 JVM 메모리                               |
| JVM 런타임   | `jvm.memory.limit`                    | `jvm_memory_limit_bytes`                      | Gauge     | bytes       | 메모리 유형과 풀별로 레이블이 지정된, 현재 JVM 메모리 제한                                 |
| JVM 런타임   | `jvm.memory.used_after_last_gc`       | `jvm_memory_used_after_last_gc_bytes`         | Gauge     | bytes       | 마지막 가비지 컬렉션 후 사용된 JVM 메모리                                                  |
| Node 런타임  | `nodejs.eventloop.time`               | `nodejs_eventloop_time_seconds_total`         | Counter   | seconds     | `idle` 또는 `active`로 레이블이 지정된 누적 이벤트 루프 시간                               |
| Node 런타임  | `nodejs.eventloop.utilization`        | `nodejs_eventloop_utilization_ratio`          | Gauge     | 1           | 마지막 샘플링 간격 동안 이벤트 루프의 활성 비율                                            |
| Node 런타임  | `nodejs.eventloop.delay.min`          | `nodejs_eventloop_delay_min_seconds`          | Gauge     | seconds     | 마지막 샘플링 간격 동안의 최소 이벤트 루프 지연                                            |
| Node 런타임  | `nodejs.eventloop.delay.max`          | `nodejs_eventloop_delay_max_seconds`          | Gauge     | seconds     | 마지막 샘플링 간격 동안의 최대 이벤트 루프 지연                                            |
| Node 런타임  | `nodejs.eventloop.delay.mean`         | `nodejs_eventloop_delay_mean_seconds`         | Gauge     | seconds     | 마지막 샘플링 간격 동안의 평균 이벤트 루프 지연                                            |
| Node 런타임  | `nodejs.eventloop.delay.stddev`       | `nodejs_eventloop_delay_stddev_seconds`       | Gauge     | seconds     | 마지막 샘플링 간격 동안 이벤트 루프 지연의 표준 편차                                       |
| Node 런타임  | `nodejs.eventloop.delay.p50`          | `nodejs_eventloop_delay_p50_seconds`          | Gauge     | seconds     | 마지막 샘플링 간격 동안의 50번째 백분위수 이벤트 루프 지연                                 |
| Node 런타임  | `nodejs.eventloop.delay.p90`          | `nodejs_eventloop_delay_p90_seconds`          | Gauge     | seconds     | 마지막 샘플링 간격 동안의 90번째 백분위수 이벤트 루프 지연                                 |
| Node 런타임  | `nodejs.eventloop.delay.p99`          | `nodejs_eventloop_delay_p99_seconds`          | Gauge     | seconds     | 마지막 샘플링 간격 동안의 99번째 백분위수 이벤트 루프 지연                                 |
| 네트워크     | `obi.network.flow.bytes`              | `obi_network_flow_bytes_total`                | Counter   | bytes       | 소스 네트워크 엔드포인트에서 대상 네트워크 엔드포인트로 전송된 바이트                      |
| 네트워크     | `obi.network.flow.packets`            | `obi_network_flow_packets_total`              | Counter   | packets     | 소스 네트워크 엔드포인트에서 대상 네트워크 엔드포인트로 관찰된 패킷                        |
| 네트워크     | `obi.network.inter.zone.bytes`        | `obi_network_inter_zone_bytes_total`          | Counter   | bytes       | 클러스터 내 클라우드 가용 영역 간에 흐르는 바이트(실험적, 현재 쿠버네티스에서만 사용 가능) |
| 네트워크     | `obi.stat.tcp.rtt`                    | `obi_stat_tcp_rtt_seconds`                    | Histogram | seconds     | 네트워크 엔드포인트 간에 관찰된 TCP 왕복 시간(RTT) 지연(StatsO11y)                         |
| 네트워크     | `obi.stat.tcp.failed.connections`     | `obi_stat_tcp_failed_connections_total`       | Counter   | 1           | 실패 사유별로 레이블이 지정된, 엔드포인트 간 실패한 TCP 연결 시도(StatsO11y)               |
| 네트워크     | `obi.stat.tcp.retransmits`            | `obi_stat_tcp_retransmits_total`              | Counter   | 1           | 연결별로 관찰된 TCP 재전송(StatsO11y)                                                      |
| 네트워크     | `obi.stat.tcp.io`                     | `obi_stat_tcp_io_bytes_total`                 | Counter   | bytes       | TCP 연결 및 I/O 방향별로 소켓 계층에서 전송된 바이트(StatsO11y)                            |

> [!NOTE]
>
> - v0.10.0은 오픈텔레메트리 시맨틱 컨벤션 이름 `rpc.client.call.duration`과
>   `rpc.server.call.duration`을 채택한다. 이에 상응하는 Prometheus 메트릭
>   이름은 표에 표시된 대로 `_call_`을 포함한다.
> - v0.12.1부터, 서버 측 Redis 및 Memcached 측정은
>   `db.client.operation.duration` 대신 `db.server.operation.duration`을
>   사용한다. 해당 서버 측 측정을 쿼리하는 대시보드와 알림을 업데이트한다.

OBI는
[스팬 메트릭](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector)과
[서비스 그래프 메트릭](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/servicegraphconnector)도
내보낼 수 있으며, 이는 [features](../configure/options/) 구성 옵션으로 활성화할
수 있다.

## OBI 메트릭의 속성 {#attributes-of-obi-metrics}

간결성을 위해, 이 목록의 메트릭과 속성은 OTel `dot.notation`을 사용한다.
Prometheus 익스포터를 사용할 때는 메트릭이 `underscore_notation`을 사용한다.

어떤 속성을 표시하거나 숨길지 구성하려면, [구성 문서](../configure/options/)의
`attributes`->`select` 섹션을 확인한다.

| 메트릭                                | 이름                                      | 기본값                                           |
| ------------------------------------- | ----------------------------------------- | ------------------------------------------------ |
| 애플리케이션(전체)                    | `http.request.method`                     | 표시됨                                           |
| 애플리케이션(전체)                    | `http.response.status_code`               | 표시됨                                           |
| 애플리케이션(전체)                    | `http.route`                              | `routes` 구성 섹션이 있으면 표시됨               |
| 애플리케이션(전체)                    | `k8s.daemonset.name`                      | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.deployment.name`                     | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.namespace.name`                      | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.node.name`                           | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.owner.name`                          | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.pod.name`                            | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.container.name`                      | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.pod.start_time`                      | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.pod.uid`                             | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.replicaset.name`                     | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.statefulset.name`                    | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `k8s.cluster.name`                        | 쿠버네티스 메타데이터가 활성화되면 표시됨        |
| 애플리케이션(전체)                    | `container.id`                            | Docker 메타데이터가 활성화되면 표시됨            |
| 애플리케이션(전체)                    | `container.name`                          | Docker 메타데이터가 활성화되면 표시됨            |
| 애플리케이션(전체)                    | `cloud.provider`                          | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `cloud.platform`                          | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `cloud.region`                            | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `cloud.account.id`                        | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `cloud.availability_zone`                 | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `cloud.resource_id`                       | 클라우드 메타데이터가 활성화되면 표시됨(Azure만) |
| 애플리케이션(전체)                    | `host.id`                                 | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `host.type`                               | 클라우드 메타데이터가 활성화되면 표시됨          |
| 애플리케이션(전체)                    | `host.image.id`                           | 클라우드 메타데이터가 활성화되면 표시됨(AWS만)   |
| 애플리케이션(전체)                    | `gcp.gce.instance.name`                   | 클라우드 메타데이터가 활성화되면 표시됨(GCP만)   |
| 애플리케이션(전체)                    | `gcp.gce.instance.hostname`               | 클라우드 메타데이터가 활성화되면 표시됨(GCP만)   |
| 애플리케이션(전체)                    | `service.name`                            | 표시됨                                           |
| 애플리케이션(전체)                    | `service.namespace`                       | 표시됨                                           |
| 애플리케이션(전체)                    | `target.instance`                         | 표시됨                                           |
| 애플리케이션(전체)                    | `url.path`                                | 숨김                                             |
| 애플리케이션(클라이언트)              | `server.address`                          | 숨김                                             |
| 애플리케이션(클라이언트)              | `server.port`                             | 숨김                                             |
| 애플리케이션 `rpc.*`                  | `rpc.response.status_code`                | 표시됨                                           |
| 애플리케이션 `rpc.*`                  | `rpc.method`                              | 표시됨                                           |
| 애플리케이션 `rpc.*`                  | `rpc.system.name`                         | 표시됨                                           |
| 애플리케이션(서버)                    | `client.address`                          | 숨김                                             |
| `obi.network.flow.bytes`              | `obi.ip`                                  | 숨김                                             |
| `db.client.operation.duration`        | `db.operation.name`                       | 표시됨                                           |
| `db.client.operation.duration`        | `db.collection.name`                      | 숨김                                             |
| `db.server.operation.duration`        | `db.operation.name`                       | 표시됨                                           |
| `db.server.operation.duration`        | `db.collection.name`                      | 숨김                                             |
| `messaging.client.operation.duration` | `messaging.system`                        | 표시됨                                           |
| `messaging.client.operation.duration` | `messaging.destination.name`              | 표시됨                                           |
| `messaging.process.duration`          | `messaging.system`                        | 표시됨                                           |
| `messaging.process.duration`          | `messaging.destination.name`              | 표시됨                                           |
| `obi.network.flow.bytes`              | `client.port`                             | 숨김                                             |
| `obi.network.flow.bytes`              | `direction`                               | 숨김                                             |
| `obi.network.flow.bytes`              | `dst.address`                             | 숨김                                             |
| `obi.network.flow.bytes`              | `dst.cidr`                                | `cidrs` 구성 섹션이 있으면 표시됨                |
| `obi.network.flow.bytes`              | `dst.name`                                | 숨김                                             |
| `obi.network.flow.bytes`              | `dst.port`                                | 숨김                                             |
| `obi.network.flow.bytes`              | `dst.zone`(쿠버네티스만)                  | 숨김                                             |
| `obi.network.flow.bytes`              | `iface`                                   | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.cluster.name`                        | 쿠버네티스가 활성화되면 표시됨                   |
| `obi.network.flow.bytes`              | `k8s.dst.name`                            | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.dst.namespace`                       | 쿠버네티스가 활성화되면 표시됨                   |
| `obi.network.flow.bytes`              | `k8s.dst.node.ip`                         | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.dst.node.name`                       | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.dst.owner.type`                      | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.dst.type`                            | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.dst.owner.name`                      | 쿠버네티스가 활성화되면 표시됨                   |
| `obi.network.flow.bytes`              | `k8s.src.name`                            | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.src.namespace`                       | 쿠버네티스가 활성화되면 표시됨                   |
| `obi.network.flow.bytes`              | `k8s.src.node.ip`                         | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.src.owner.name`                      | 쿠버네티스가 활성화되면 표시됨                   |
| `obi.network.flow.bytes`              | `k8s.src.owner.type`                      | 숨김                                             |
| `obi.network.flow.bytes`              | `k8s.src.type`                            | 숨김                                             |
| `obi.network.flow.bytes`              | `server.port`                             | 숨김                                             |
| `obi.network.flow.bytes`              | `src.address`                             | 숨김                                             |
| `obi.network.flow.bytes`              | `src.cidr`                                | `cidrs` 구성 섹션이 있으면 표시됨                |
| `obi.network.flow.bytes`              | `src.name`                                | 숨김                                             |
| `obi.network.flow.bytes`              | `src.port`                                | 숨김                                             |
| `obi.network.flow.bytes`              | `src.zone`(쿠버네티스만)                  | 숨김                                             |
| `obi.network.flow.bytes`              | `transport`                               | 숨김                                             |
| `obi.network.flow.bytes`              | `network.type`                            | 숨김                                             |
| `obi.network.flow.bytes`              | `network.protocol.name`                   | 숨김                                             |
| `obi.network.flow.bytes`              | `src.country`                             | `geoip` 구성 섹션이 있으면 표시됨                |
| `obi.network.flow.bytes`              | `src.asn`                                 | `geoip` 구성 섹션이 있으면 표시됨                |
| `obi.network.flow.bytes`              | `dst.country`                             | `geoip` 구성 섹션이 있으면 표시됨                |
| `obi.network.flow.bytes`              | `dst.asn`                                 | `geoip` 구성 섹션이 있으면 표시됨                |
| `obi.stat.tcp.rtt`                    | `obi.ip`                                  | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `src.address`                             | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `dst.address`                             | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `src.port`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `dst.port`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `src.name`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `dst.name`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `src.zone`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `dst.zone`                                | 숨김                                             |
| `obi.stat.tcp.rtt`                    | `network.tcp.handshake.role`              | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `obi.ip`                                  | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `src.address`                             | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `dst.address`                             | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `src.port`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `dst.port`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `src.name`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `dst.name`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `src.zone`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `dst.zone`                                | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `reason`                                  | 숨김                                             |
| `obi.stat.tcp.failed.connections`     | `network.tcp.handshake.role`              | 숨김                                             |
| `obi.stat.tcp.retransmits`            | 소스/대상 주소, 포트, 이름, 존(zone) 속성 | 숨김                                             |
| `obi.stat.tcp.io`                     | 소스/대상 주소, 포트, 이름, 존(zone) 속성 | 숨김                                             |
| `obi.stat.tcp.io`                     | `network.io.direction`                    | 표시됨                                           |
| 트레이스(HTTP)                        | `url.query`                               | 쿼리 문자열이 있을 때 표시됨                     |
| 트레이스(GraphQL)                     | `graphql.document`                        | 숨김                                             |
| 트레이스(SQL, Redis)                  | `db.query.text`                           | 숨김                                             |

> [!NOTE]
>
> `obi.network.inter.zone.bytes` 메트릭은 `obi.network.flow.bytes`와 동일한 속성
> 집합을 지원하지만, `k8s.cluster.name`, `src.zone`, `dst.zone`를 제외한 모든
> 속성은 기본적으로 숨겨진다.
>
> `obi.network.flow.packets` 메트릭은 `obi.network.flow.bytes`와 동일한 속성 및
> 기본값을 지원한다. JVM 메모리 메트릭에는 `jvm.memory.type`과
> `jvm.memory.pool.name`이 포함된다. `nodejs.eventloop.time` 메트릭에는 `idle`
> 또는 `active` 값을 갖는 `nodejs.eventloop.state`가 포함된다.

## 내부 메트릭 {#internal-metrics}

> [!WARNING]
>
> OBI v0.12.1은 네 개의 내부 OTLP 메트릭 이름에서 Prometheus 스타일 접미사를
> 제거한다. OTLP 쿼리를 `obi.bpf.map.entries_total`에서 `obi.bpf.map.entries`로,
> `obi.bpf.map.max_entries_total`에서 `obi.bpf.map.max_entries`로,
> `obi.bpf.network.packets.total`에서 `obi.bpf.network.packets`로,
> `obi.bpf.network.ignored.packets.total`에서
> `obi.bpf.network.ignored.packets`로 업데이트한다. 이에 상응하는 Prometheus
> 카운터 이름은 변경되지 않는다. Prometheus 게이지 이름은
> `obi_bpf_map_entries`와 `obi_bpf_map_max_entries`가 된다.

OBI는 Prometheus 형식으로
[내부 메트릭을 보고하도록 구성](../configure/internal-metrics-reporter/)할 수
있다.

| 이름                                    | 유형       | 설명                                                                           |
| --------------------------------------- | ---------- | ------------------------------------------------------------------------------ |
| `obi_ebpf_tracer_flushes`               | Histogram  | eBPF 트레이서에서 다음 파이프라인 단계로 플러시된 트레이스 그룹의 길이         |
| `obi_metric_exports_total`              | Counter    | 원격 OTel 컬렉터로 제출된 메트릭 배치의 길이                                   |
| `obi_metric_export_errors_total`        | CounterVec | 오류 유형별로, 실패한 각 OTel 메트릭 내보내기의 오류 수                        |
| `obi_trace_exports_total`               | Counter    | 원격 OTel 컬렉터로 제출된 트레이스 배치의 길이                                 |
| `obi_trace_export_errors_total`         | CounterVec | 오류 유형별로, 실패한 각 OTel 트레이스 내보내기의 오류 수                      |
| `obi_prometheus_http_requests_total`    | CounterVec | HTTP 포트와 경로별로 분류된, Prometheus 스크레이프 엔드포인트로 향하는 요청 수 |
| `obi_bpf_network_ignored_packets_total` | Counter    | 흐름 집계 전에 OBI 네트워크 필터가 폐기한 네트워크 패킷 수                     |
| `obi_instrumented_processes`            | GaugeVec   | 프로세스 이름과 함께, OBI가 계측한 프로세스                                    |
| `obi_internal_build_info`               | GaugeVec   | 빌드 시간과 커밋 해시를 포함한 OBI 바이너리의 버전 정보                        |
| `obi_avoided_services`                  | GaugeVec   | 오픈텔레메트리 SDK 내보내기를 감지한 후 OBI가 텔레메트리를 억제한 서비스       |
