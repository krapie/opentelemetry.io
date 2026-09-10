---
title: OBI 전역 구성 속성
linkTitle: 전역 속성
description: OBI 코어에 적용되는 전역 구성 속성을 구성한다.
weight: 2
cSpell:ignore: healthz
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름, 예시, 환경 변수를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

OBI는 환경 변수로 구성하거나, `-config` 명령줄 인수 또는 `OTEL_EBPF_CONFIG_PATH`
환경 변수로 전달하는 YAML 구성 파일로 구성할 수 있다. 환경 변수는 구성 파일의
속성보다 우선한다. 예를 들어 다음 명령줄에서는 `OTEL_EBPF_LOG_LEVEL` 옵션이
config.yaml 안의 모든 `log_level` 설정을 재정의한다.

**Config 인수:**

```sh
OTEL_EBPF_LOG_LEVEL=debug obi -config /path/to/config.yaml
```

**Config 환경 변수:**

```sh
OTEL_EBPF_LOG_LEVEL=debug OTEL_EBPF_CONFIG_PATH=/path/to/config.yaml obi
```

구성 파일 템플릿은 [예시 YAML 구성 파일](../example/)을 참고한다.

OBI는 HTTP 및 gRPC 애플리케이션에서 트레이스를 생성, 변환, 내보내는 구성 요소의
파이프라인으로 이루어진다. YAML 구성에서 각 구성 요소는 자체적인 1차 섹션을
가진다.

선택적으로 OBI는 네트워크 수준 메트릭도 제공한다. 자세한 내용은
[네트워크 메트릭 문서](../../network/)를 참고한다.

다음 섹션은 전체 OBI 구성에 적용되는 전역 구성 속성을 설명한다.

예를 들면 다음과 같다.

```yaml
trace_printer: json
shutdown_timeout: 30s
channel_buffer_len: 33
```

| YAML<br>환경 변수                                  | 설명                                                                                                                          | 유형                    | 기본값     |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------- | ---------- |
| _(YAML 없음)_<br>`OTEL_EBPF_AUTO_TARGET_EXE`       | 전체 실행 파일 경로에 대한 [Glob](<https://en.wikipedia.org/wiki/Glob_(programming)>) 매칭으로 계측할 프로세스를 선택한다.    | string                  | unset      |
| _(YAML 없음)_<br>`OTEL_EBPF_AUTO_TARGET_LANGUAGE`  | 감지된 프로그래밍 언어(glob 매처)를 기준으로 계측할 프로세스를 선택한다(예: `go`, `java`, `nodejs`).                          | string                  | unset      |
| `open_port`<br>`OTEL_EBPF_OPEN_PORT`               | 열린 포트를 기준으로 계측할 프로세스를 선택한다. 쉼표로 구분된 포트 목록과 포트 범위를 받는다.                                | string                  | unset      |
| `target_pids`<br>`OTEL_EBPF_TARGET_PID`            | PID를 기준으로 계측할 프로세스를 선택한다. YAML 목록, 단일 값, 또는 쉼표로 구분된 환경 변수 목록을 받는다.                    | integer or integer list | unset      |
| `shutdown_timeout`<br>`OTEL_EBPF_SHUTDOWN_TIMEOUT` | 정상 종료(graceful shutdown)의 타임아웃을 설정한다                                                                            | string                  | "10s"      |
| `log_level`<br>`OTEL_EBPF_LOG_LEVEL`               | 프로세스 로거의 상세 수준을 설정한다. 유효한 값: `DEBUG`, `INFO`, `WARN`, `ERROR`.                                            | string                  | `INFO`     |
| `log_format`<br>`OTEL_EBPF_LOG_FORMAT`             | 로거 출력 형식을 설정한다. 유효한 값: `text`, `json`.                                                                         | string                  | `text`     |
| `trace_printer`<br>`OTEL_EBPF_TRACE_PRINTER`       | 계측된 트레이스를 지정된 형식으로 stdout에 인쇄한다. 자세한 내용은 [트레이스 프린터 형식](#trace-printer-formats)을 참고한다. | string                  | `disabled` |
| `enforce_sys_caps`<br>`OTEL_EBPF_ENFORCE_SYS_CAPS` | 시작 시 누락된 시스템 capabilities를 OBI가 어떻게 처리할지 제어한다.                                                          | boolean                 | `false`    |

## 헬스 체크 엔드포인트 {#health-check-endpoint}

OBI는 TCP 또는 Unix 도메인 소켓을 통해 `/healthz`를 노출할 수 있다. JSON 응답은
스키마 버전, 현재 Unix 시간, 프로세스 가동 시간을 보고한다. 이 엔드포인트는
기본적으로 비활성화되어 있다.

```yaml
health_check:
  port: 8080
```

| YAML<br>환경 변수                                                            | 설명                                                                                                        | 유형   | 기본값  |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ------ | ------- |
| `health_check.port`<br>`OTEL_EBPF_HEALTH_CHECK_PORT`                         | 지정된 TCP 포트에서 헬스 엔드포인트를 연다. `0`은 TCP 헬스 체크를 비활성화한다.                             | int    | `0`     |
| `health_check.unix_socket_path`<br>`OTEL_EBPF_HEALTH_CHECK_UNIX_SOCKET_PATH` | 헬스 엔드포인트를 절대 파일 시스템 경로 또는 `@` 접두사가 붙은 추상 소켓에 바인딩한다. `port`보다 우선한다. | string | (unset) |

예를 들어 `unix_socket_path: /var/run/obi-health.sock`는 파일 시스템 소켓을
생성하고, `unix_socket_path: '@obi-health'`는 추상 소켓을 사용한다.

## 실행 파일 이름 매칭 {#executable-name-matching}

이 속성은 전체 실행 명령줄(파일 시스템에서 실행 파일이 위치한 디렉터리 포함)과
일치시킬 [glob](<https://en.wikipedia.org/wiki/Glob_(programming)>)을 받는다.
OBI는 하나의 프로세스, 또는 유사한 특성을 가진 여러 프로세스를 선택한다. 더
자세한 프로세스 선택 및 그룹화는
[서비스 디스커버리 문서](../service-discovery/)를 참고한다.

실행 파일 이름으로 계측할 때는 대상 시스템에서 실행 파일 하나와 일치하는
모호하지 않은 이름을 선택한다. 예를 들어 `OTEL_EBPF_AUTO_TARGET_EXE=*/server`를
설정했는데 Glob과 일치하는 프로세스가 두 개 있으면 OBI는 둘 다 선택한다. 대신
정확한 일치를 위해 전체 애플리케이션 경로를 사용한다(예:
`OTEL_EBPF_AUTO_TARGET_EXE=/opt/app/server` 또는
`OTEL_EBPF_AUTO_TARGET_EXE=/server`).

`OTEL_EBPF_AUTO_TARGET_EXE`와 `OTEL_EBPF_OPEN_PORT` 속성을 모두 설정하면 OBI는
두 선택 기준을 모두 만족하는 실행 파일만 선택한다.

## 언어 매칭 {#language-matching}

`OTEL_EBPF_AUTO_TARGET_LANGUAGE`를 사용하여 감지된 언어 런타임을 기준으로
프로세스를 타겟팅한다.

예를 들면 다음과 같다.

```shell
OTEL_EBPF_AUTO_TARGET_LANGUAGE=go
```

glob 표현식을 사용할 수도 있다.

```shell
OTEL_EBPF_AUTO_TARGET_LANGUAGE='java*'
```

이 옵션을 `OTEL_EBPF_AUTO_TARGET_EXE` 및/또는 `OTEL_EBPF_OPEN_PORT`와 조합하면,
프로세스는 구성된 모든 선택자를 만족해야 한다.

## 대상 PID 매칭 {#target-pid-matching}

`target_pids`(YAML) 또는 `OTEL_EBPF_TARGET_PID`(환경 변수)를 사용하여 특정
프로세스 ID만 계측한다.

YAML 예시:

```yaml
target_pids: [1234, 5678]
```

```yaml
target_pids: 1234
```

환경 변수 예시:

```shell
OTEL_EBPF_TARGET_PID=1234,5678
```

이는 정확한 프로세스 ID를 알고 있는 타겟팅된 문제 해결과 통제된 릴리스에
유용하다.

## 열린 포트 매칭 {#open-port-matching}

이 속성은 쉼표로 구분된 포트 또는 포트 범위 목록을 받는다. 실행 파일이 포트 중
하나라도 일치하면 OBI는 이를 선택한다. 예를 들면 다음과 같다.

```shell
OTEL_EBPF_OPEN_PORT=80,443,8000-8999
```

이 예시에서 OBI는 포트 `80`, `443`, 또는 `8000`에서 `8999` 사이의 포트를 여는
모든 실행 파일을 선택한다. 하나의 프로세스 또는 유사한 특성을 가진 여러
프로세스를 선택할 수 있다. 더 자세한 프로세스 선택 및 그룹화는
[서비스 디스커버리 문서](../service-discovery/)의 지침을 따른다.

실행 파일이 여러 포트를 열면, OBI가 모든 애플리케이션 포트에서 HTTP/S 및 gRPC
요청을 모두 계측하도록 하려면 그 포트 중 하나만 지정하면 된다. 현재는 계측을
특정 포트의 요청으로만 제한할 방법이 없다.

지정된 포트 범위가 `1-65535`처럼 넓으면, OBI는 그 범위의 포트 중 하나를 소유한
모든 프로세스를 실행하려고 시도한다.

`OTEL_EBPF_AUTO_TARGET_EXE`와 `OTEL_EBPF_OPEN_PORT` 속성을 모두 설정하면 OBI는
두 선택 기준을 모두 만족하는 실행 파일만 선택한다.

## 트레이스 프린터 형식 {#trace-printer-formats}

이 옵션은 계측된 모든 트레이스를 다음 형식 중 하나로 표준 출력에 인쇄한다.

- **`disabled`**: 프린터를 비활성화한다
- **`text`**: 간결한 텍스트 한 줄을 인쇄한다
- **`json`**: 압축된 JSON 객체를 인쇄한다
- **`json_indent`**: 들여쓰기된 JSON 객체를 인쇄한다

## 시스템 capabilities {#system-capabilities}

`enforce_sys_caps`를 true로 설정했는데 필요한 시스템 capabilities가 누락되면
OBI는 시작을 중단하고 누락된 capabilities를 로그로 남긴다. 이 옵션을 `false`로
설정하면 OBI는 누락된 capabilities를 로그로만 남긴다.
