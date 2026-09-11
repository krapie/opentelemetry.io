---
title: Prometheus로의 OTLP 메트릭 내보내기
linkTitle: OTLP 메트릭 내보내기
cSpell:ignore: uuidgen
default_lang_commit: 734887889291bbe83b1023e127953dc354a7b8f6
---

## 소개 {#introduction}

Prometheus는 풀 기반(pull-based) 모니터링을 위해 설계되고 최적화되었으며,
타겟(target)을 발견하고 일정한 간격으로 메트릭 엔드포인트를
스크레이핑(scraping)한다. 이 모델은 Prometheus 아키텍처의 핵심으로, 서비스
디스커버리(service discovery)와 일관된 타겟 기반 수집 같은 기능을 뒷받침한다.

오픈텔레메트리(OpenTelemetry) 채택이 늘어나면서, 최신 버전의 Prometheus는 OTLP를
통해 푸시 기반(push-based) 메트릭을 수신하는 것을 지원하기 시작했다. 이
구성에서는 오픈텔레메트리 SDK가 HTTP를 통한 OTLP로 메트릭을 내보내고,
Prometheus는 메트릭을 스크레이핑하는 대신 OTLP 리시버(receiver) 역할을 한다. 이
방식은 더 단순한 구성, 실험, 또는 로컬 개발 환경에서 사용할 수 있다. 다만
오픈텔레메트리를 사용하는 프로덕션 배포 환경에서는 중개자로
[오픈텔레메트리 컬렉터](/docs/collector/#when-to-use-a-collector)를 사용할 것을
강력히 권장한다.

이 가이드는 오픈텔레메트리 SDK에서 Prometheus의 OTLP 엔드포인트로 직접 OTLP
메트릭을 내보내도록 구성하는 방법을 설명한다. 필요한 환경 변수,
익스포터(exporter) 구성, 그리고 서비스 식별, 내보내기 간격, 운영상의
트레이드오프(trade-off) 같은 주요 고려 사항을 다룬다.

## 사전 준비 사항 {#prerequisite}

시작하기 전에 다음 요구 사항이 충족되었는지 확인한다.

- Prometheus를 설정한다.
  [이 Prometheus 가이드의 예시 prometheus.yml 구성](https://prometheus.io/docs/guides/opentelemetry/#configuring-prometheus)을
  따른다.
- [OTLP 리시버를 활성화한다](https://prometheus.io/docs/guides/opentelemetry/#enable-the-otlp-receiver)

Prometheus 설정을 마쳤다면, 애플리케이션이 메트릭을 OTLP 수집(ingestion)
엔드포인트로 직접 전송하도록 구성하는 단계로 넘어갈 수 있다.

### 환경 변수 사용 {#use-environment-variables}

오픈텔레메트리 SDK와 계측 라이브러리는
[표준 환경 변수](/docs/languages/sdk-configuration/)로 구성할 수 있다.
애플리케이션을 시작하기 전에 환경 변수를 설정한다. localhost의 Prometheus 서버로
오픈텔레메트리 메트릭을 전송하려면 다음 오픈텔레메트리 변수가 필요하다.

```bash
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:9090/api/v1/otlp
```

메트릭만 필요하다면 Prometheus를 사용할 때 트레이스와 로그를 꺼둔다.

```bash
export OTEL_TRACES_EXPORTER=none
export OTEL_LOGS_EXPORTER=none
```

오픈텔레메트리 메트릭의 기본 푸시 간격은 60초이다. 이는 모니터링 요구 사항에
따라 조정할 수 있다. 예를 들어 15초 간격으로 설정하면 더 응답성이 좋은 메트릭과
더 빠른 알림을 얻을 수 있지만, 그 대가로 네트워크와 처리 오버헤드가 늘어난다.

```bash
export OTEL_METRIC_EXPORT_INTERVAL=15000
```

계측 라이브러리가 `service.name`과 `service.instance.id`를 기본으로 제공하지
않는다면, 이를 직접 설정할 것을 강력히 권장한다. 이 속성들이 없으면 서비스를
신뢰성 있게 식별하거나 인스턴스를 구분하기 어려워져 디버깅과 집계가 훨씬 더
어려워진다. 아래 예시는 시스템에 `uuidgen` 명령을 사용할 수 있다고 가정한다.

```bash
export OTEL_SERVICE_NAME="my-example-service"
export OTEL_RESOURCE_ATTRIBUTES="service.instance.id=$(uuidgen)"
```

> [!NOTE]
>
> `service.instance.id`가 인스턴스마다 고유한지, 그리고 리소스 속성이 변경될
> 때마다 새로운 `service.instance.id`가 생성되는지 확인한다.
> [권장하는 방법](/docs/specs/semconv/resource/service/#service-instance)은
> 인스턴스가 시작될 때마다 새 UUID를 생성하는 것이다.

### 텔레메트리 구성 {#configure-telemetry}

[언어별 SDK 문서](/docs/languages/_index.md)의 OTLP 설정에서 사용한 것과 동일한
`exporter`와 `reader`를 사용하도록 오픈텔레메트리 구성(configuration)을
업데이트한다. 환경 변수가 올바르게 설정되고 로드되어 있다면, 오픈텔레메트리
SDK가 이를 자동으로 읽어들인다.
