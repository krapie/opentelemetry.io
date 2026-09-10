---
title: OBI 성능 구성
linkTitle: 성능 튜닝
description:
  eBPF 트레이서 구성 요소가 외부 프로세스의 HTTP 및 GRPC 서비스를 계측하고,
  파이프라인의 다음 단계로 전달할 트레이스를 생성하는 방식을 구성한다.
weight: 90
cSpell:ignore: qdisc ringbuffer
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2에서는 이 설정들을
> `capture.engine`과 프로토콜별 섹션 아래에서 구성한다. 자세한 내용은
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

eBPF 트레이서를 사용하여 OBI 성능을 세밀하게 튜닝할 수 있다.

YAML 구성의 `ebpf` 섹션이나 환경 변수로 이 구성 요소를 구성할 수 있다.

| YAML<br>환경 변수                                                   | 설명                                                                                                                                                                         | 유형    | 기본값  |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | ------- |
| `wakeup_len`<p>`OTEL_EBPF_BPF_WAKEUP_LEN`</p>                       | 사용자 공간에 웨이크업 요청을 보내기 전에 OBI가 eBPF 링 버퍼에 누적하는 메시지 수를 설정한다. [웨이크업 길이](#wake-up-length)를 참고한다.                                   | int     | 500     |
| `stats_wakeup_data_bytes`<p>`OTEL_EBPF_STATS_WAKEUP_DATA_BYTES`</p> | TCP 통계 링 버퍼가 사용자 공간 소비자를 깨우기 전에 큐에 쌓이는 최소 바이트 수를 설정한다. [통계 웨이크업 임계값](#stats-wake-up-threshold)을 참고한다.                      | int     | 4096    |
| `traffic_control_backend`<p>`OTEL_EBPF_BPF_TC_BACKEND`</p>          | 트래픽 제어 프로브를 연결할 백엔드를 선택한다. 자세한 내용은 [트래픽 제어 백엔드](#traffic-control-backend) 섹션을 참고한다.                                                 | string  | `auto`  |
| `http_request_timeout`<p>`OTEL_EBPF_BPF_HTTP_REQUEST_TIMEOUT`</p>   | OBI가 HTTP 요청을 타임아웃으로 간주하는 시간 간격을 설정한다. 자세한 내용은 [HTTP 요청 타임아웃](#http-request-timeout) 섹션을 참고한다.                                     | string  | (0ms)   |
| `high_request_volume`<p>`OTEL_EBPF_BPF_HIGH_REQUEST_VOLUME`</p>     | OBI가 응답을 감지하자마자 텔레메트리 이벤트를 보낸다. 자세한 내용은 [높은 요청량](#high-request-volume) 섹션을 참고한다.                                                     | boolean | (false) |
| `maps_config.global_scale_factor`                                   | eBPF 맵 크기를 2의 거듭제곱으로 조정한다. 양수 값은 맵 크기를 키우고, 음수 값은 맵 크기를 줄이며, 0은 기본값을 유지한다. [eBPF 맵 크기 조정](#ebpf-map-resizing)을 참고한다. | int     | 0       |

## 웨이크업 길이 {#wake-up-length}

OBI는 eBPF 링 버퍼에 메시지를 누적하고, 이 값에 도달하면 사용자 공간에 웨이크업
요청을 보낸다.

부하가 높은 서비스의 경우, 이 옵션을 더 높게 설정하여 CPU 오버헤드를 줄인다.

부하가 낮은 서비스의 경우, 값이 높으면 OBI가 메트릭을 제출하는 시점과 메트릭이
가시화되는 시점이 지연될 수 있다.

## 통계 웨이크업 임계값 {#stats-wake-up-threshold}

`stats_wakeup_data_bytes`는 `wakeup_len`과 독립적으로 TCP 통계 링 버퍼를
제어한다. 값이 높으면 부하 상황에서 사용자 공간 웨이크업이 줄어드는 대신 메트릭
전달 지연이 늘어난다. 제출된 모든 이벤트마다 소비자를 깨우려면 `0`으로 설정한다.
큐에 쌓인 이벤트가 버퍼가 가득 차기 전에 비워지도록 값을 링 버퍼 크기보다 충분히
낮게 유지한다.

## 트래픽 제어 백엔드 {#traffic-control-backend}

이 옵션은 트래픽 제어 프로브를 연결할 백엔드를 선택한다. Linux 6.6은 파일
디스크립터 기반 트래픽 제어 연결 방식인 TCX 지원을 추가한다. TCX는 더 견고하고,
명시적인 qdisc 관리를 요구하지 않으며, 프로브를 결정론적으로 체이닝한다. 커널 >=
6.6에서는 `tcx` 백엔드를 권장한다. `auto`로 설정하면 OBI가 커널에 가장 적합한
백엔드를 선택한다.

허용되는 백엔드: `tc`, `tcx`, `auto`. 이 값을 비워 두거나 설정하지 않으면 OBI는
`auto`를 사용한다.

## HTTP 요청 타임아웃 {#http-request-timeout}

이 옵션은 OBI가 HTTP 요청을 타임아웃으로 간주하기 전에 얼마나 기다릴지 설정한다.
OBI는 타임아웃되어 반환되지 않는 HTTP 트랜잭션을 보고할 수 있다. 자동 HTTP 요청
타임아웃을 활성화하려면 이 옵션을 0이 아닌 값으로 설정한다. 요청이 타임아웃되면
OBI는 HTTP 상태 코드 408을 보고한다. 연결 끊김은 타임아웃처럼 보일 수 있으므로,
이 값을 설정하면 요청 평균이 증가할 수 있다.

## 높은 요청량 {#high-request-volume}

이 옵션은 OBI가 응답을 감지하자마자 텔레메트리 이벤트를 보내도록 한다. 응답이 큰
요청에 대한 타이밍 정확도는 떨어지지만, 대량 시나리오에서는 트레이스 이벤트
누락을 줄이는 데 도움이 된다.

## eBPF 맵 크기 조정 {#ebpf-map-resizing}

`maps_config.global_scale_factor` 옵션을 사용하면 워크로드 특성에 맞춰 성능을
튜닝하기 위해 런타임에 eBPF 맵 크기를 동적으로 조정할 수 있다.

- **0보다 큰 값**은 맵 크기를 2의 거듭제곱으로 늘린다(예: `1`은 2배, `2`는 4배).
- **0보다 작은 값**은 맵 크기를 2의 거듭제곱으로 줄인다(예: `-1`은 1/2배, `-2`는
  1/4배).
- **기본값 `0`**은 표준 맵 크기를 유지한다.
- 유효 범위: `-3`에서 `3`까지.

예시 구성:

```yaml
ebpf:
  maps_config:
    global_scale_factor: 1 # Double the default map sizes
```

이는 리소스 제약으로 인해 eBPF 메모리 사용량을 신중하게 튜닝해야 하는 컨테이너
또는 쿠버네티스 환경에서 특히 유용하다.
