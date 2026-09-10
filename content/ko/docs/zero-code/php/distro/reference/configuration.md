---
title: 구성
description: 오픈텔레메트리(OpenTelemetry) PHP Distro의 구성 옵션이다.
weight: 1
# prettier-ignore
cSpell:ignore: ComponentProvider keypass opentelemetry-php-contrib stderr syslog yaml
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리(OpenTelemetry) PHP Distro는 표준 오픈텔레메트리 SDK 구성과 배포판
전용 옵션을 지원한다.

## 구성 방법 {#configuration-method}

PHP 프로세스에서 사용할 수 있는 환경 변수로 구성한다.

- 오픈텔레메트리 표준 옵션의 경우 `OTEL_*`
- 배포판 전용 옵션의 경우 `OTEL_PHP_*`

예시:

```sh
export OTEL_EXPORTER_OTLP_ENDPOINT="https://your-endpoint:443/"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <token>"
export OTEL_PHP_LOG_LEVEL_STDERR="INFO"
```

## 오픈텔레메트리 옵션 {#opentelemetry-options}

이 배포판(distro)은 표준 오픈텔레메트리 PHP SDK 옵션을 지원한다.

| 옵션                                    | 기본값                  | 허용 값                          | 설명                                       |
| --------------------------------------- | ----------------------- | -------------------------------- | ------------------------------------------ |
| `OTEL_EXPORTER_OTLP_ENDPOINT`           | `http://localhost:4318` | URL                              | OTLP 엔드포인트 URL                        |
| `OTEL_EXPORTER_OTLP_HEADERS`            | (비어 있음)             | `key=value,key2=value2`          | OTLP 요청 헤더                             |
| `OTEL_EXPORTER_OTLP_INSECURE`           | `false`                 | `true` 또는 `false`              | TLS 검증 비활성화(테스트 전용)             |
| `OTEL_EXPORTER_OTLP_CERTIFICATE`        | (비어 있음)             | 파일 시스템 경로(PEM)            | OTLP TLS용 CA 인증서 경로                  |
| `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` | (비어 있음)             | 파일 시스템 경로(PEM)            | OTLP mTLS용 클라이언트 인증서              |
| `OTEL_EXPORTER_OTLP_CLIENT_KEY`         | (비어 있음)             | 파일 시스템 경로(PEM)            | OTLP mTLS용 클라이언트 키                  |
| `OTEL_EXPORTER_OTLP_CLIENT_KEYPASS`     | (비어 있음)             | 문자열                           | 암호화된 OTLP 클라이언트 키의 패스프레이즈 |
| `OTEL_SERVICE_NAME`                     | `unknown_service`       | 문자열                           | `service.name` 리소스 속성 값              |
| `OTEL_RESOURCE_ATTRIBUTES`              | (비어 있음)             | `key=value,key2=value2`          | 리소스 속성                                |
| `OTEL_TRACES_SAMPLER`                   | `parentbased_always_on` | 샘플러 이름                      | 트레이스 샘플러                            |
| `OTEL_TRACES_SAMPLER_ARG`               | (비어 있음)             | 문자열/숫자                      | 샘플러 인자                                |
| `OTEL_LOG_LEVEL`                        | `info`                  | `error`, `warn`, `info`, `debug` | SDK 내부 로그 레벨                         |

## 배포판 전용 옵션(`OTEL_PHP_*`) {#distro-specific-options-otel_php_}

모든 `OTEL_PHP_*` 옵션은 환경 변수로 설정하거나 `php.ini`에 설정할 수 있다.

`php.ini`에서는 `opentelemetry_distro.` 접두사와 소문자 옵션 이름을 사용한다.

예시:

```sh
export OTEL_PHP_ENABLED=true
```

```ini
opentelemetry_distro.enabled=true
```

### 일반 구성 {#general-configuration}

| 옵션                                                 | 기본값 | 허용 값             | 설명                                                                                                     |
| ---------------------------------------------------- | ------ | ------------------- | -------------------------------------------------------------------------------------------------------- |
| `OTEL_PHP_ENABLED`                                   | `true` | `true` 또는 `false` | 자동 부트스트랩(bootstrap) 활성화                                                                        |
| `OTEL_PHP_OPENTELEMETRY_EXTENSION_EMULATION_ENABLED` | `true` | `true` 또는 `false` | 에뮬레이트된 `opentelemetry` 확장 등록을 활성화하여, `opentelemetry.so` 없이도 자동 계측이 동작하도록 함 |
| `OTEL_PHP_NATIVE_OTLP_SERIALIZER_ENABLED`            | `true` | `true` 또는 `false` | 네이티브 OTLP protobuf 직렬화기 활성화                                                                   |

### 비동기 데이터 전송 {#asynchronous-data-sending}

| 옵션                                        | 기본값 | 허용 값                                | 설명                                |
| ------------------------------------------- | ------ | -------------------------------------- | ----------------------------------- |
| `OTEL_PHP_ASYNC_TRANSPORT`                  | `true` | `true` 또는 `false`                    | 텔레메트리의 백그라운드 전송 활성화 |
| `OTEL_PHP_ASYNC_TRANSPORT_SHUTDOWN_TIMEOUT` | `30s`  | 기간(`ms`, `s`, `m`)                   | 종료 시 플러시 타임아웃             |
| `OTEL_PHP_MAX_SEND_QUEUE_SIZE`              | `2MB`  | `B`, `MB`, `GB`를 선택적으로 붙인 정수 | 워커당 최대 비동기 버퍼 크기        |

### 로깅 {#logging}

| 옵션                        | 기본값      | 허용 값                                                         | 설명                  |
| --------------------------- | ----------- | --------------------------------------------------------------- | --------------------- |
| `OTEL_PHP_LOG_FILE`         | (비어 있음) | 파일 시스템 경로                                                | 로그 출력 파일 경로   |
| `OTEL_PHP_LOG_LEVEL_FILE`   | `OFF`       | `OFF`, `CRITICAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`, `TRACE` | 파일 싱크 로그 레벨   |
| `OTEL_PHP_LOG_LEVEL_STDERR` | `OFF`       | `OFF`, `CRITICAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`, `TRACE` | stderr 싱크 로그 레벨 |
| `OTEL_PHP_LOG_LEVEL_SYSLOG` | `OFF`       | `OFF`, `CRITICAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`, `TRACE` | syslog 싱크 로그 레벨 |
| `OTEL_PHP_LOG_FEATURES`     | (비어 있음) | `FEATURE=LEVEL,...`                                             | 기능별 로그 레벨      |

### 트랜잭션 스팬 {#transaction-span}

| 옵션                                    | 기본값      | 허용 값                  | 설명                     |
| --------------------------------------- | ----------- | ------------------------ | ------------------------ |
| `OTEL_PHP_TRANSACTION_SPAN_ENABLED`     | `true`      | `true` 또는 `false`      | 웹 SAPI용 자동 루트 스팬 |
| `OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI` | `true`      | `true` 또는 `false`      | CLI용 자동 루트 스팬     |
| `OTEL_PHP_TRANSACTION_URL_GROUPS`       | (비어 있음) | 쉼표로 구분된 와일드카드 | URL 그룹화 패턴          |

### 어트리뷰트 기반 계측 {#attribute-based-instrumentation}

| 옵션                          | 기본값  | 허용 값             | 설명                                                                                                                                                                           |
| ----------------------------- | ------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OTEL_PHP_ATTR_HOOKS_ENABLED` | `false` | `true` 또는 `false` | `#[WithSpan]` / `#[SpanAttribute]` 어트리뷰트 기반 스팬 생성을 활성화한다. [어트리뷰트 기반 계측](/docs/zero-code/php/distro/reference/attribute-instrumentation/)을 참고한다. |

### 스코프 의존성 브리지 {#scoped-dependencies-bridge}

| 옵션                                  | 기본값  | 허용 값             | 설명                                                                                                                                                                                                         |
| ------------------------------------- | ------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OTEL_PHP_SCOPED_DEPS_BRIDGE_ENABLED` | `false` | `true` 또는 `false` | 애플리케이션 자체의 오픈텔레메트리 사용이 배포판의 런타임(트레이서 프로바이더, 컨텍스트)을 공유하도록 하여, 그 스팬이 배포판의 트레이스에 합류하도록 한다. [아래 참고](#scoped-dependencies-bridge-interop). |

### 추론된 스팬 {#inferred-spans}

| 옵션                                         | 기본값  | 허용 값              | 설명                             |
| -------------------------------------------- | ------- | -------------------- | -------------------------------- |
| `OTEL_PHP_INFERRED_SPANS_ENABLED`            | `false` | `true` 또는 `false`  | 추론된 스팬 활성화               |
| `OTEL_PHP_INFERRED_SPANS_REDUCTION_ENABLED`  | `true`  | `true` 또는 `false`  | 연속된 중복 프레임 축소          |
| `OTEL_PHP_INFERRED_SPANS_STACKTRACE_ENABLED` | `true`  | `true` 또는 `false`  | 추론된 스팬에 스택 트레이스 첨부 |
| `OTEL_PHP_INFERRED_SPANS_SAMPLING_INTERVAL`  | `50ms`  | 기간(`ms`, `s`, `m`) | 스택 트레이스 샘플링 간격        |
| `OTEL_PHP_INFERRED_SPANS_MIN_DURATION`       | `0`     | 기간(`ms`, `s`, `m`) | 최소 추론된 스팬 지속 시간       |

### 중앙 구성(OpAMP) {#central-configuration-opamp}

| 옵션                                | 기본값      | 허용 값                             | 설명                                                                                           |
| ----------------------------------- | ----------- | ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `OTEL_PHP_OPAMP_ENDPOINT`           | (비어 있음) | `/v1/opamp`로 끝나는 HTTP/HTTPS URL | OpAMP 엔드포인트                                                                               |
| `OTEL_PHP_OPAMP_HEADERS`            | (비어 있음) | `key=value,key2=value2`             | OpAMP 요청 헤더                                                                                |
| `OTEL_PHP_OPAMP_HEARTBEAT_INTERVAL` | `30s`       | 기간(`ms`, `s`, `m`)                | OpAMP 서버로 전송되는 하트비트 메시지 사이의 간격이다.                                         |
| `OTEL_PHP_OPAMP_POLLING_INTERVAL`   | `30s`       | 기간(`ms`, `s`, `m`)                | 에이전트가 갱신된 구성을 얻기 위해 OpAMP 서버를 폴링하는 간격이다. 하트비트 간격과 독립적이다. |
| `OTEL_PHP_OPAMP_SEND_TIMEOUT`       | `10s`       | 기간(`ms`, `s`, `m`)                | OpAMP 전송 타임아웃                                                                            |
| `OTEL_PHP_OPAMP_SEND_MAX_RETRIES`   | `3`         | 0 이상의 정수                       | 재시도 횟수                                                                                    |
| `OTEL_PHP_OPAMP_SEND_RETRY_DELAY`   | `10s`       | 기간(`ms`, `s`, `m`)                | 재시도 지연                                                                                    |
| `OTEL_PHP_OPAMP_INSECURE`           | `false`     | `true` 또는 `false`                 | TLS 검증 비활성화(테스트 전용)                                                                 |
| `OTEL_PHP_OPAMP_CERTIFICATE`        | (비어 있음) | 파일 시스템 경로(PEM)               | OpAMP TLS용 CA 인증서 경로                                                                     |
| `OTEL_PHP_OPAMP_CLIENT_CERTIFICATE` | (비어 있음) | 파일 시스템 경로(PEM)               | OpAMP mTLS용 클라이언트 인증서 경로                                                            |
| `OTEL_PHP_OPAMP_CLIENT_KEY`         | (비어 있음) | 파일 시스템 경로(PEM)               | OpAMP mTLS용 클라이언트 키 경로                                                                |
| `OTEL_PHP_OPAMP_CLIENT_KEYPASS`     | (비어 있음) | 문자열                              | 암호화된 클라이언트 키의 패스프레이즈                                                          |

### 지원성 {#supportability}

| 옵션                           | 기본값 | 허용 값             | 설명                                                                                                                                                      |
| ------------------------------ | ------ | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_PHP_SCOPED_DEPS_ENABLED` | `true` | `true` 또는 `false` | 배포판이 스코프가 지정된(네임스페이스 접두사가 붙은) 의존성을 사용할지 원본 의존성을 사용할지 제어한다. [아래 참고](#scoped-dependencies-bridge-interop). |

## 참고 사항 {#notes}

- 백그라운드 전송은 OTLP HTTP/protobuf 모드에서 동작한다.
- `OTEL_PHP_AUTOLOAD_ENABLED`는 배포판 런타임에서 활성화 상태로 강제된다.
- 배포판 패키지에는 여러 의존성(오픈텔레메트리 SDK, 다양한 자동 계측 패키지,
  그리고 그 전이 의존성)이 포함된다. 애플리케이션 자체 의존성과의 네임스페이스
  충돌을 방지하기 위해, 배포판은 기본적으로 **스코프가 지정된**(네임스페이스
  접두사가 붙은) 의존성을 사용한다. 스코프가 지정되지 않은 의존성으로 되돌리려면
  `OTEL_PHP_SCOPED_DEPS_ENABLED=false`로 설정한다.

### 스코프 의존성 브리지 상호 운용 {#scoped-dependencies-bridge-interop}

기본적으로 배포판의 오픈텔레메트리 런타임은 **스코프가 지정되어(scoped)** 있다.
즉, 배포판의 클래스는 애플리케이션이 Composer로 설치하는 표준 `OpenTelemetry\*`
클래스와 분리된 고유한 네임스페이스 접두사 아래에 존재한다. 그 결과 애플리케이션
자체의 오픈텔레메트리 사용은 별도의 런타임에서 실행되며, 그 스팬은 내보내지지도
않고 배포판의 트레이스에 연결되지도 않는다.

`OTEL_PHP_SCOPED_DEPS_BRIDGE_ENABLED=true`로 설정하면 둘을 연결한다.
애플리케이션의 Composer 오토로더가 실행되기 전에, 배포판은 스코프가 지정되지
않은 `OpenTelemetry\*` API를 자신의 스코프 구현에 매핑하는 클래스 별칭을
등록한다. 그러면 애플리케이션 자체의 오픈텔레메트리 사용은 배포판의 트레이서
프로바이더와 컨텍스트를 투명하게 사용하므로, 그 스팬이 내보내지고 배포판의
트레이스 안에서 올바르게 상위-하위 관계로 연결된다.

이 옵션은 스코프 지정이 비활성화된
경우(`OTEL_PHP_SCOPED_DEPS_ENABLED=false`)에는 아무 효과가 없다. 스코프 지정이
없으면 배포판은 이미 스코프가 지정되지 않은 `OpenTelemetry\*` 클래스를
사용하므로, 별도의 브리지 없이 공유가 이루어진다.

## 파일 기반 구성(선언형) {#file-based-configuration-declarative}

환경 변수 대신, `OTEL_CONFIG_FILE` 환경 변수를 설정하여 YAML 구성 파일로 SDK를
구성할 수 있다.

```sh
export OTEL_CONFIG_FILE=/path/to/otel-config.yaml
```

`OTEL_CONFIG_FILE`이 설정되면 다음과 같이 동작한다.

- SDK는 개별 `OTEL_*` 환경 변수 대신 YAML 파일에서 모든 구성을 읽는다.
- YAML 파일 내에서 환경 변수 치환(`${MY_VAR:-default}`)이 지원된다.
- 중앙 구성(OpAMP)이 자동으로 비활성화된다. 파일 기반 구성과 원격 구성은 상호
  배타적이다.
- 배포판 전용 옵션(`OTEL_PHP_*`)은 SDK와 무관한 네이티브 확장 옵션이므로 계속
  동작한다.

### 배포판 리소스 디텍터 {#distro-resource-detector}

배포판은 `telemetry.distro.name` 및 `telemetry.distro.version` 리소스 속성을
추가하는 `distro` 리소스 디텍터(resource detector)를 제공한다. 파일 기반
구성에서 이를 활성화하려면 `resource.detection/development.detectors` 섹션에
추가한다.

```yaml
file_format: '1.0-rc.2'

resource:
  attributes:
    - name: service.name
      value: my-service
  detection/development:
    detectors:
      - distro: {}

propagator:
  composite:
    - tracecontext:
    - baggage:

tracer_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: http://localhost:4318/v1/traces

meter_provider:
  readers:
    - periodic:
        exporter:
          otlp_http:
            endpoint: http://localhost:4318/v1/metrics

logger_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: http://localhost:4318/v1/logs
```

전체 YAML 스키마는
[오픈텔레메트리 구성 스키마](https://github.com/open-telemetry/opentelemetry-configuration/blob/main/schema-docs.md)를
참고한다.

### 제한 사항 {#limitations}

- 파일 기반 구성이 활성화된 경우 중앙 구성(OpAMP)을 사용할 수 없다.
- `Registry::registerResourceDetector()`를 통해 등록된 리소스 디텍터(예:
  `opentelemetry-php-contrib`의 클라우드 제공자 디텍터)는 자동으로 활성화되지
  않는다. 이러한 디텍터는 `ComponentProvider`를 제공하고 YAML
  `resource.detection/development.detectors` 섹션에 명시적으로 나열되어야 한다.
