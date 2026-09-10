---
title: OBI 내보내기 모드 구성
linkTitle: 내보내기 모드
description: OTLP 엔드포인트로 데이터를 직접 내보내도록 OBI를 구성한다
weight: 1
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

다이렉트 모드(Direct mode)에서 OBI는 오픈텔레메트리(OpenTelemetry)
프로토콜(OTLP)을 사용하여 메트릭과 트레이스를 원격 엔드포인트로 직접 푸시한다.

OBI는 예를 들어 **풀(pull)** 모드에서 스크레이프할 준비가 된 Prometheus HTTP
엔드포인트를 노출할 수도 있다.

다이렉트 모드를 사용하려면 인증 자격 증명 구성이 필요하다. 다음 환경 변수로 OTLP
엔드포인트 인증 자격 증명을 설정한다.

- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_HEADERS`

Prometheus 스크레이프 엔드포인트를 사용하여 다이렉트 모드로 실행하려면
[구성 문서](../options/)를 참고한다.

## OBI 구성 및 실행 {#configure-and-run-obi}

이 튜토리얼은 OBI와 OTel 컬렉터가 동일한 호스트에서 네이티브로 실행 중이라고
가정하므로, 트래픽을 보호하거나 OTel 컬렉터 OTLP 리시버에 인증을 제공할 필요가
없다.

[오픈텔레메트리 eBPF 계측](../../setup/)을 설치하고 예시
[구성 파일](/docs/zero-code/obi/configure/resources/instrumenter-config.yml)을
다운로드한다.

먼저, 계측할 실행 파일을 지정한다. 포트 `443`에서 실행 중인 서비스 실행 파일의
경우, YAML 문서에 `open_port` 속성을 추가한다.

```yaml
discovery:
  instrument:
    - open_ports: 443
```

다음으로, 트레이스와 메트릭이 전송될 위치를 지정한다. OTel 컬렉터가 로컬
호스트에서 실행 중이면 포트 `4318`을 사용한다.

```yaml
otel_metrics_export:
  endpoint: http://localhost:4318
otel_traces_export:
  endpoint: http://localhost:4318
```

`otel_metrics_export`와 `otel_traces_export` 속성을 조합하여 메트릭, 트레이스
또는 둘 다를 내보낼 수 있다.

이름이 지정된 구성 파일로 OBI를 실행한다.

```shell
obi -config instrument-config.yml
```

또는

```shell
OTEL_EBPF_CONFIG_PATH=instrument-config.yml obi
```
