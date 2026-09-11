---
title: 레지스트리에 추가하기
linkTitle: 추가
description: 레지스트리에 항목을 추가하는 방법.
cSpell:ignore: zpages
default_lang_commit: 42ef3b8c965480f4d58b173ed95fcb05fbc7d429
---

{{% include freeze-notice.md %}}

오픈텔레메트리를 위한 통합을 관리하거나 기여하고 있는가? [레지스트리](../)에
프로젝트를 소개하고 싶다!

프로젝트를 추가하려면 [풀 리퀘스트][pull request]를 제출한다. 다음 템플릿인
[registry-entry.yml][]을 사용해 [data/registry][]에 프로젝트의 데이터 파일을
만들어야 한다.

프로젝트 이름과 설명이 [마케팅 가이드라인][marketing guidelines]을 따르고,
리눅스 파운데이션의 브랜딩 및 [상표 사용
가이드라인][trademark usage guidelines]에 부합하는지 확인한다.

## 레지스트리 유형 {#registry-types}

프로젝트를 레지스트리에 추가할 때는 `registryType`을 지정해야 한다. 이 필드는
오픈텔레메트리와의 관계에 따라 프로젝트를 분류한다. 다음은 가능한 값과 그
정의이다.

### `application integration` {#application-integration}

**용도**: 외부 플러그인이나 계측 라이브러리 없이도 오픈텔레메트리가 네이티브로
통합(내장 지원)된 애플리케이션 또는 서비스.

**예시**: [통합](/ecosystem/integrations/) 페이지의 네이티브 애플리케이션 통합
목록을 참고한다.

> [!NOTE]
>
> 상용/독점 라이선스를 허용하는 유일한 레지스트리 유형이다.

### `api` {#api}

**용도**: 특정 언어를 위한 오픈텔레메트리 API를 구현하는 패키지. SDK와는
독립적으로, 계측된 코드와 라이브러리가 의존하는 인터페이스와 no-op 구현을
말한다.

**예시**: Ruby의 `opentelemetry-api`, `opentelemetry-metrics-api`,
`opentelemetry-logs-api` gem.

### `connector` {#connector}

**용도**: 두 파이프라인을 연결하는 오픈텔레메트리 컬렉터 커넥터(connector) 구성
요소. 한 파이프라인의 익스포터(exporter)이자 다른 파이프라인의 리시버(receiver)
역할을 하며, 시그널 유형 간 변환을 수행하기도 한다.

**예시**: Count 커넥터, span-to-metrics 커넥터, failover 커넥터.

### `core` {#core}

**용도**: 오픈텔레메트리 프로젝트의 핵심(core) 구성 요소 전용. 서드파티 구성
요소나 오픈텔레메트리 프로젝트가 아닌 구성 요소에는 절대 해당하지 않는다.

### `exporter` {#exporter}

**용도**: 오픈텔레메트리 컬렉터의 익스포터 구성 요소 또는 언어별 SDK 내의
익스포터 라이브러리.

**예시**: OTLP 익스포터, Prometheus 익스포터, 또는 텔레메트리 데이터를 외부
시스템으로 전송하는 모든 구성 요소.

**참고**: 텔레메트리 데이터를 내보내는 서드파티 구성 요소에는 해당하지 않는다.

### `extension` {#extension}

**용도**: 오픈텔레메트리 기능을 확장하는 컬렉터 또는 SDK 익스텐션(extension).

**예시**: 인증자(authenticator), 구성 소스/프로바이더(provider), 서비스
디스커버리, 헬스 체크/pprof/zpages, 또는 컬렉터/SDK 동작을 보강하는 그 밖의 구성
요소.

### `id-generator` {#id-generator}

**용도**: 트레이스 및 스팬 ID 생성 방식을 커스터마이징하는 SDK 구성 요소.

**예시**: AWS X-Ray 호환 ID 생성기.

### `instrumentation` {#instrumentation}

**용도**: 특정 라이브러리/프레임워크를 위한 계측 라이브러리 또는 네이티브 계측.

**예시**: HTTP 계측, 데이터베이스 계측, 프레임워크별 계측, 또는 해당하는 경우
자동 계측 에이전트.

### `log-bridge` {#log-bridge}

**용도**: 기존 로깅 프레임워크/API를 오픈텔레메트리 로깅에 연결하는 언어별
어댑터. 앱이 익숙한 로깅 API를 통해 OTel 로그를 내보낼 수 있게 한다.

**예시**: Java SLF4J/Log4j/Logback, Python logging, JavaScript Winston/Pino, Go
log/slog/zap 등 프레임워크를 위한 브리지/핸들러/어펜더(appender).

### `metric-producer` {#metric-producer}

**용도**: 서드파티 소스의 메트릭을 SDK 메트릭 리더(reader)로 연결하는 SDK 구성
요소.

### `processor` {#processor}

**용도**: 오픈텔레메트리 컬렉터 프로세서 구성 요소.

**예시**: 배치 프로세서, 속성(attribute) 프로세서, 샘플링 프로세서, 또는 컬렉터
파이프라인 내에서 텔레메트리 데이터를 처리하는 모든 구성 요소.

### `propagator` {#propagator}

**용도**: 특정 와이어 포맷(wire format)으로 프로세스 경계를 넘어 트레이스
컨텍스트와 배기지(baggage)를 전달하는 컨텍스트 전파자(propagator).

**예시**: B3, Jaeger, AWS X-Ray 전파자.

### `provider` {#provider}

**용도**: 오픈텔레메트리 컬렉터 프로바이더 구성 요소.

**예시**: 구성(configuration) 프로바이더, 자격 증명 프로바이더, 또는 컬렉터에
리소스나 구성을 제공하는 모든 구성 요소.

### `receiver` {#receiver}

**용도**: 오픈텔레메트리 컬렉터 리시버 구성 요소.

**예시**: OTLP 리시버, Prometheus 리시버, 또는 외부 소스로부터 텔레메트리
데이터를 수신하는 모든 구성 요소.

> [!NOTE]
>
> 오픈텔레메트리 텔레메트리를 수신하는 서드파티 구성 요소에는 해당하지 않는다.

### `resource-detector` {#resource-detector}

**용도**: 언어별 SDK를 위한 리소스 디텍터(resource detector).

**예시**: AWS 리소스 디텍터, GCP 리소스 디텍터, 또는 텔레메트리에 리소스 정보를
자동으로 감지하여 추가하는 모든 구성 요소.

### `sampler` {#sampler}

**용도**: 어떤 스팬을 기록하고 내보낼지 결정하는 SDK 샘플러(sampler).

**예시**: AWS X-Ray 원격 샘플러 또는 규칙 기반 샘플러.

### `sdk` {#sdk}

**용도**: 특정 언어를 위한 오픈텔레메트리 SDK를 구현하는 패키지.

**예시**: Ruby의 `opentelemetry-sdk`, `opentelemetry-metrics-sdk`,
`opentelemetry-logs-sdk` gem.

### `semantic-convention` {#semantic-convention}

**용도**: 특정 언어를 위한 시맨틱 컨벤션(semantic convention) 상수를 제공하는
패키지.

**예시**: Ruby의 `opentelemetry-semantic_conventions` gem.

### `utilities` {#utilities}

**용도**: 오픈텔레메트리 작업에 사용할 수 있는 그 밖의 모든 도구.

**예시**: 테스트 유틸리티, 디버깅 도구, 마이그레이션 도구, 또는 오픈텔레메트리
작업을 돕는 모든 헬퍼 라이브러리.

[data/registry]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/data/registry
[pull request]:
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request
[registry-entry.yml]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/templates/registry-entry.yml
[marketing guidelines]: /community/marketing-guidelines/
[trademark usage guidelines]:
  https://www.linuxfoundation.org/legal/trademark-usage
