---
title: 오픈텔레메트리 eBPF 계측
linkTitle: OBI
description:
  오픈텔레메트리(OpenTelemetry) eBPF 계측을 사용하여 자동 계측을 수행하는 방법을
  알아본다.
weight: 3
cascade:
  OTEL_RESOURCE_ATTRIBUTES_APPLICATION: obi
  OTEL_RESOURCE_ATTRIBUTES_NAMESPACE: obi
  OTEL_RESOURCE_ATTRIBUTES_POD: obi
cSpell:ignore: Aerospike HotSpot Ollama Qwen rerank SunRPC uprobe
default_lang_commit: ef4bd8a4e59c4c5eed5a19c0210b468fed33c140
---

오픈텔레메트리(OpenTelemetry) 라이브러리는 인기 있는 프로그래밍 언어와
프레임워크를 위한 텔레메트리 수집(collect)을 제공한다. 하지만 분산
트레이싱(distributed tracing)을 시작하는 것은 복잡할 수 있다. Go나 Rust와 같은
일부 컴파일 언어에서는 코드에 트레이스포인트(tracepoint)를 수동으로 추가해야
한다.

오픈텔레메트리 eBPF 계측(OpenTelemetry eBPF Instrumentation, OBI)은 애플리케이션
옵저버빌리티(Application Observability)를 쉽게 시작할 수 있게 해주는 자동 계측
도구이다. OBI는 eBPF를 사용하여 애플리케이션 실행 파일과 OS 네트워킹 계층을
자동으로 검사하고, 지원되는 Linux 워크로드에 대해 트레이스 스팬, Rate Errors
Duration(RED) 메트릭, 런타임 메트릭, 애플리케이션 및 네트워크 관계를 캡처한다.
모든 데이터 캡처는 애플리케이션 코드나 구성에 대한 수정 없이 이루어진다.

OBI는 다음과 같은 기능을 제공한다.

- **광범위한 언어 지원**: Java(JDK 8+), .NET, Go, Python, Ruby, Node.js, C, C++,
  Rust
- **경량**: 코드 변경 불필요, 설치할 라이브러리 없음, 재시작 불필요
- **효율적인 계측**: 트레이스와 메트릭을 최소한의 오버헤드로 eBPF
  프로브(probe)가 캡처한다
- **분산 트레이싱**: 분산 트레이스 스팬을 캡처하여 컬렉터로 보고한다
- **로그 보강(enrichment)**: 상관관계(correlation)를 위해 JSON 및 일반 텍스트
  로그를 트레이스 컨텍스트로 보강한다
- **쿠버네티스 네이티브**: 쿠버네티스 애플리케이션에 대해 구성이 필요 없는 자동
  계측을 제공한다
- **암호화된 통신에 대한 가시성**: 복호화 없이 TLS/SSL을 통한 트랜잭션을
  캡처한다
- **컨텍스트 전파**: 서비스 간에 트레이스 컨텍스트를 자동으로 전파한다
- **프로토콜 지원(클라이언트 및 서버)**: HTTP/S, HTTP/2, gRPC, Kafka, NATS,
  MQTT, Memcached, SunRPC(NFS 포함), JSON-RPC
- **프로토콜 지원(클라이언트 전용)**: AMQP 1.0, DNS 쿼리
- **데이터베이스 계측(클라이언트 및 서버)**: PostgreSQL(pgx 드라이버 포함),
  MySQL, MSSQL, Redis
- **데이터베이스 계측(클라이언트 전용)**: MongoDB, Couchbase(N1QL/SQL++ 및 KV
  프로토콜), Aerospike, Elasticsearch, OpenSearch
- **HTTP 페이로드 계측**: 서버 측 GraphQL 및 클라이언트 측 Elasticsearch,
  OpenSearch, AWS S3, AWS SQS, 그리고 클라이언트와 서버 양쪽 모두에서 JSON-RPC를
  통한 MCP
- **GenAI 계측**: OpenAI, OpenAI 호환 게이트웨이, Ollama, Anthropic Claude,
  Google AI Studio(Gemini), AWS Bedrock, Qwen(DashScope), JSON-RPC를 통한 MCP,
  임베딩 및 재순위(rerank) API, 벡터 검색 시스템을 위한 트레이스 및 메트릭
- **런타임 메트릭**: SDK 변경 없이 Go, HotSpot JVM, Node.js 이벤트 루프 메트릭을
  수집(collect)한다
- **GPU 계측**: Linux에서 지원되는 CUDA 런타임 작업을 캡처한다
- **스팬 및 서비스 그래프 메트릭**: 애플리케이션 스팬 메트릭과 서비스 간 관계를
  내보낸다(export)
- **낮은 카디널리티(cardinality) 메트릭**: 비용 절감을 위해 카디널리티가 낮은
  Prometheus 호환 메트릭
- **네트워크 옵저버빌리티**: 바이트 및 패킷 카운터, TCP RTT, 재전송(retransmit),
  연결, 소켓 I/O 메트릭으로 서비스 간 네트워크 흐름을 캡처한다
- **향상된 서비스 디스커버리**: DNS 확인(resolution)을 통한 서비스 이름 조회
  개선
- **컬렉터 통합**: OBI를 오픈텔레메트리 컬렉터 리시버 구성 요소로 실행한다

## 최근 하이라이트(v0.12.1) {#recent-highlights-v0121}

OBI v0.12.1은 v0.12.0을 위해 준비된 변경 사항이 배포된(published) 릴리스이다.
v0.12.0에는 태그가 지정되었지만, 릴리스 검증이 실패한 후 배포되지 않았다.
의도했던 v0.12.0 변경 사항과 릴리스 수정 사항이 모두 포함된 v0.12.1을 설치한다.

주목할 만한 변경 사항은 다음과 같다.

- **Node.js 수동 스팬**: 애플리케이션이 오픈텔레메트리 SDK를 등록하지 않은 경우
  `@opentelemetry/api`로 생성된 스팬을 캡처한다
- **Node.js 런타임 메트릭**: 이벤트 루프 시간, 사용률, 지연 메트릭을 추가한다
- **더 정밀한 Config v2 필터**: 프로토콜과 시그널별로 독립적으로 애플리케이션
  필터를 적용한다
- **프로세스 컨텍스트 보강(enrichment)**: 계측된 프로세스가 실험적인 `OTEL_CTX`
  매핑을 통해 게시하는 리소스 속성 및 메타데이터를 읽는다
- **데이터베이스 서버 메트릭**: 서버 측 Redis, Memcached, SQL 작업에 대해
  `db.server.operation.duration`을 추가한다
- **개선된 Java 서비스 이름**: `java`로 대체되기 전에 Spring Boot 애플리케이션
  이름, JAR 매니페스트 제목, 또는 JAR 기본 이름을 사용한다
- **신뢰성 수정**: private-stack 커널을 uprobe 선점(preemption)으로부터
  보호하고, 필요한 프로브가 연결(attach)될 수 없을 때 안전하지 않은 컨텍스트
  전파를 방지하며, 트레이스 부모 관계, 래핑된 Go TLS 연결, 수명이 짧은
  프로세스의 로그 보강, OTLP 속성 처리를 수정한다

전체 변경 사항 목록 및 업그레이드 참고 사항은
[릴리스 노트](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases/tag/v0.12.1)를
참고한다.

업스트림(upstream) 예시를 살펴보고 싶다면
[NGINX 안내서(walkthrough)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/tree/v0.12.1/examples/nginx)와
[Apache 안내서(walkthrough)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/tree/v0.12.1/examples/apache)를
참고한다.

## OBI 동작 방식 {#how-obi-works}

다음 다이어그램은 OBI의 상위 수준 아키텍처와 eBPF 계측이 텔레메트리
파이프라인에서 어디에 위치하는지를 보여준다.

![OBI eBPF 아키텍처](./ebpf-arch.svg)

## 호환성 {#compatibility}

OBI는 다음 요구 사항을 충족하는 Linux 환경을 지원한다.

| 요구 사항    | 지원 내용                                                                  |
| :----------- | :------------------------------------------------------------------------- |
| CPU 아키텍처 | `amd64`, `arm64`                                                           |
| Linux 커널   | `5.8+`, 또는 필요한 eBPF 백포트(backport)가 적용된 RHEL 계열 Linux `4.18+` |
| 커널 기능    | BTF                                                                        |
| 권한         | Root, 또는 활성화된 OBI 기능에 필요한 Linux capabilities                   |

OBI는 다음과 같은 지원되는 릴리스 아티팩트를 게시한다.

| 아티팩트                                            | 지원 플랫폼                  |
| :-------------------------------------------------- | :--------------------------- |
| `obi` 바이너리 아카이브                             | Linux `amd64`, Linux `arm64` |
| `otel/ebpf-instrument` 컨테이너 이미지              | Linux `amd64`, Linux `arm64` |
| `otel/opentelemetry-ebpf-k8s-cache` 컨테이너 이미지 | Linux `amd64`, Linux `arm64` |

OBI는 위 요구 사항을 충족하는 환경이라면 독립형(standalone) Linux 호스트,
컨테이너, 쿠버네티스에 배포할 수 있다.

OBI는 Linux 이외의 운영체제, `amd64` 및 `arm64` 이외의 Linux 아키텍처, BTF가
없는 Linux 환경, 그리고 문서화된 RHEL 계열 `4.18+` 예외를 벗어난 Linux `5.8`
이전 커널 버전은 지원하지 않는다.

기능별 지원 세부 사항은 다음 가이드에 문서화되어 있다.

- [분산 트레이스](distributed-traces/): 컨텍스트 전파 지원, 런타임별 요구 사항,
  분산 트레이싱의 제한 사항
- [트레이스 컨텍스트 연관(association)](context-propagation/): 비동기 및 스레드
  기반 요청 처리를 위한 부모-자식 연관 지원
- [데이터 내보내기](configure/export-data/): 프로토콜, 데이터베이스, 메시징,
  GenAI, GPU, Go 라이브러리 계측 지원

## 제한 사항 {#limitations}

OBI는 코드 변경 없이 애플리케이션 및 프로토콜 옵저버빌리티를 제공하지만, 모든
시나리오에서 언어 수준의 계측을 대체하지는 않는다. eBPF 기반 계측이 자동으로
도출할 수 없는 커스텀 스팬, 애플리케이션별 속성, 비즈니스 이벤트, 기타 프로세스
내부(in-process) 텔레메트리가 필요한 경우 언어 에이전트나 수동 계측을 사용한다.

OBI는 네트워크 및 프로토콜 활동을 자동으로 캡처할 수 있지만, eBPF 관찰 지점에서
보이지 않는 애플리케이션별 세부 정보는 항상 복구할 수 있는 것은 아니다.

일부 기능은 핵심 플랫폼 요구 사항보다 추가적인 주의 사항이 있거나 지원 범위가 더
좁을 수 있다. 자세한 내용은 [분산 트레이스](distributed-traces/) 및
[내보낸 계측](configure/export-data/)에 대한 기능별 문서를 참고한다.

OBI에 필요한 전체 capabilities 목록을 확인하려면
[보안, 권한 및 capabilities](security/)를 참고한다.

## OBI 시작하기 {#get-started-with-obi}

- Docker 또는 쿠버네티스로 OBI를 시작하려면 [설정](setup/) 문서를 따른다.
- 트레이스를 애플리케이션 로그와 연결하고 JSON 로그를 트레이스 컨텍스트로
  보강하려면 [트레이스-로그 상관관계(correlation)](./trace-log-correlation/)를
  알아본다.
- 중앙 집중식 텔레메트리 처리를 위해
  [OBI를 컬렉터 리시버로 실행하는 방법](./configure/collector-receiver/)을
  알아본다.

## 문제 해결 {#troubleshooting}

- 일반적인 문제에 대한 도움이 필요하다면 [문제 해결](./troubleshooting) 가이드를
  참고한다.
