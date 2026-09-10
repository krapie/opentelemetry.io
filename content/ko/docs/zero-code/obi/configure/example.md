---
title: OBI Config v1 YAML 예시
linkTitle: Config v1 YAML 예시
description: OBI를 위한 Config v1 YAML 파일 예시.
weight: 100
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

## YAML 파일 예시 {#yaml-file-example}

```yaml
discovery:
  instrument:
    - open_ports: 8443
log_level: DEBUG

ebpf:
  context_propagation: all

otel_traces_export:
  endpoint: http://localhost:4318

prometheus_export:
  port: 8999
  path: /metrics
```

이 구성에는 다음 옵션이 포함된다.

- `discovery.instrument.open_ports`: 포트 8443에서 수신 대기 중인 서비스를
  계측한다
- `log_level`: 로깅 상세 수준을 `DEBUG`로 설정한다
- `ebpf.context_propagation`: 지원되는 모든 캐리어를 사용하여 컨텍스트 전파를
  활성화한다
- `otel_traces_export.endpoint`: `http://localhost:4318`의 오픈텔레메트리
  컬렉터(OpenTelemetry Collector)로 트레이스를 보낸다
- `prometheus_export`: `http://localhost:8999/metrics`에서 메트릭을 노출한다
