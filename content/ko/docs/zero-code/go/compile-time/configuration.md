---
title: 구성
description: otelc 도구와 계측된 애플리케이션이 생성하는 텔레메트리를 구성한다.
weight: 20
cSpell:ignore: nethttp otelc
default_lang_commit: d2e57aa2352b9e64c117d07c304923ac12b5d7e8
---

구성은 두 지점에서 이루어진다. 빌드 타임에는 `otelc` 도구가 애플리케이션을
계측하는 방식을 제어하고, 런타임에는 표준 오픈텔레메트리(OpenTelemetry) 환경
변수가 계측된 애플리케이션이 생성하는 텔레메트리를 제어한다.

## otelc 명령 {#the-otelc-command}

`otelc`는 Go 툴체인(toolchain)을 감싼다. 그 하위 명령은 다음과 같다.

| 명령            | 용도                                                                                        |
| --------------- | ------------------------------------------------------------------------------------------- |
| `otelc go …`    | 계측이 적용된 상태로 `go` 명령(예: `go build`)을 실행한다                                   |
| `otelc setup`   | 계측을 위한 환경을 설정한다                                                                 |
| `otelc pin`     | 현재 모듈의 계측 패키지를 고정하기 위해 `otel.instrumentation.go`를 생성하거나 업데이트한다 |
| `otelc cleanup` | setup 및 build 단계에서 생성된 모든 아티팩트를 제거한다                                     |
| `otelc version` | 도구 버전을 출력한다                                                                        |

플래그는 하위 명령 앞에 전달한다.

| 플래그             | 환경 변수        | 용도                                       |
| ------------------ | ---------------- | ------------------------------------------ |
| `--rules <file>`   |                  | 사용자 정의 계측 규칙 파일을 사용한다      |
| `--debug`, `-d`    | `OTELC_DEBUG=1`  | 빌드에 대한 디버그 로깅을 활성화한다       |
| `--work-dir`, `-w` | `OTELC_WORK_DIR` | 빌드 중 작성되는 작업 파일을 위한 디렉터리 |

예를 들어, 사용자 정의 규칙 파일과 디버그 출력으로 빌드하려면 다음과 같이 한다.

```sh
otelc --rules my-rules.yaml --debug go build -o myapp .
```

## 런타임 환경 변수 {#runtime-environment-variables}

계측된 애플리케이션은 익스포터(exporter), 리소스, 서비스 신원에 대해 표준
오픈텔레메트리 [SDK 환경 변수](/docs/languages/sdk-configuration/)를 따른다.
예를 들면 다음과 같다.

- `OTEL_SERVICE_NAME`: 텔레메트리와 함께 보고되는 서비스 이름
- `OTEL_EXPORTER_OTLP_ENDPOINT`: 내보낼 OTLP 엔드포인트
- `OTEL_RESOURCE_ATTRIBUTES`: 추가 리소스 속성

또한 다음 변수는 런타임에 어떤 주입된 계측이 활성화되는지를 제어한다.

| 변수                                | 용도                                                                                           |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `OTEL_GO_ENABLED_INSTRUMENTATIONS`  | 활성화할 계측을 쉼표로 구분한 목록이다(예: `nethttp,grpc`). 설정하면 나열된 계측만 활성화된다. |
| `OTEL_GO_DISABLED_INSTRUMENTATIONS` | 비활성화할 계측을 쉼표로 구분한 목록이다.                                                      |

## 사용자 정의 계측 규칙 {#custom-instrumentation-rules}

어떤 코드가 계측되는지는 선언적 YAML 규칙으로 결정된다. 각 규칙은 대상 패키지의
이름을 지정하고, 선택적으로 셀렉터로 매칭 범위를 좁히며, 무엇을 주입할지
선언한다. 예를 들어, 다음 규칙은 `(*sql.DB).Exec`의 진입과 종료 시점에
후크(hook) 함수를 호출한다.

```yaml
instrument_sql_exec:
  target: database/sql
  where:
    func: Exec
    recv: '*DB'
  do:
    - inject_hooks:
        before: BeforeExec
        after: AfterExec
        path: github.com/example/sqlinstr
```

`--rules`로 사용자 정의 규칙 파일을 빌드에 전달한다. 규칙은 Go 패키지 내부에
포함될 수도 있다. 패키지는 `otel.instrumentation.go`(또는 `otelc.tool.go`)
파일로 계측을 선언하고 코드 옆의 `otelc.yml` 또는 `*.otelc.yml` 파일에 규칙을
제공하며, 도구는 애플리케이션을 계측할 때 이를 찾아 로드한다. 규칙은 함수 후크
외에도 구조체 필드 주입, 호출 지점 래핑, 파일 추가 등 여러 주입 메커니즘을
지원한다. 전체 스키마와 규칙 타입 참고 자료는 저장소의
[계측 규칙 문서](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/blob/main/docs/rules.md)를
참고한다.
