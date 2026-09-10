---
title: 에이전트 구성
linkTitle: 구성
weight: 10
aliases:
  - /docs/languages/python/automatic/configuration
  - /docs/languages/python/automatic/agent-config
# prettier-ignore
cSpell:ignore: gevent healthcheck instrumentor monkeypatch pyproject Starlette urllib
default_lang_commit: 7a39e1b95f51cf97fe203ef98a1011d3be33d77e
---

에이전트(agent)는 다음 방법으로 매우 유연하게 구성할 수 있다.

- CLI에서 구성 속성(property) 전달
- [환경 변수](/docs/specs/otel/configuration/sdk-environment-variables/) 설정

## 구성 속성 {#configuration-properties}

다음은 구성 속성을 통한 에이전트 구성 예시다.

```sh
opentelemetry-instrument \
    --traces_exporter console,otlp \
    --metrics_exporter console \
    --service_name your-service-name \
    --exporter_otlp_endpoint 0.0.0.0:4317 \
    python myapp.py
```

각 구성이 하는 일은 다음과 같다.

- `traces_exporter`는 사용할 트레이스(traces) 익스포터(exporter)를 지정한다. 이
  경우 트레이스는 `console`(stdout)과 `otlp`로 내보내진다. `otlp` 옵션은
  `opentelemetry-instrument`가 gRPC를 통해 OTLP를 받는 엔드포인트로 트레이스를
  전송하도록 지시한다. gRPC 대신 HTTP를 사용하려면
  `--exporter_otlp_protocol http/protobuf`를 추가한다. traces_exporter에 사용할
  수 있는 전체 옵션 목록은 Python contrib의
  [OpenTelemetry Instrumentation](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/opentelemetry-instrumentation)을
  참고한다.
- `metrics_exporter`는 사용할 메트릭(metrics) 익스포터를 지정한다. 이 경우
  메트릭은 `console`(stdout)로 내보내진다. 현재는 메트릭 익스포터를 지정해야
  한다. 메트릭을 내보내지 않는다면 대신 값으로 `none`을 지정한다.
- `service_name`은 텔레메트리와 연결된 서비스의 이름을 설정하며,
  [옵저버빌리티 백엔드](/ecosystem/vendors/)로 전송된다.
- `exporter_otlp_endpoint`는 텔레메트리를 내보낼 엔드포인트를 설정한다. 생략하면
  기본 [컬렉터(Collector)](/docs/collector/) 엔드포인트가 사용되며, 이는 gRPC의
  경우 `0.0.0.0:4317`, HTTP의 경우 `0.0.0.0:4318`이다.
- `exporter_otlp_headers`는 선택한 옵저버빌리티 백엔드에 따라 필요하다. OTLP
  익스포터 헤더에 대한 자세한 내용은
  [OTEL_EXPORTER_OTLP_HEADERS](/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_headers)를
  참고한다.

## 환경 변수 {#environment-variables}

경우에 따라 [환경 변수](/docs/languages/sdk-configuration/)를 통한 구성이 더
선호된다. 명령줄 인자로 구성할 수 있는 모든 설정은 환경 변수로도 구성할 수 있다.

원하는 구성 속성에 대한 올바른 이름 매핑은 다음 단계로 알아낼 수 있다.

- 구성 속성을 대문자로 변환한다.
- 환경 변수 앞에 `OTEL_`을 붙인다.

예를 들어 `exporter_otlp_endpoint`는 `OTEL_EXPORTER_OTLP_ENDPOINT`로 변환된다.

## Python 전용 구성 {#python-specific-configuration}

환경 변수 앞에 `OTEL_PYTHON_`을 붙여 설정할 수 있는 Python 전용 구성 옵션이 몇
가지 있다.

### 제외 URL {#excluded-urls}

모든 계측에서 제외할 URL을 나타내는, 쉼표로 구분된 정규 표현식이다.

- `OTEL_PYTHON_EXCLUDED_URLS`

`OTEL_PYTHON_<library>_EXCLUDED_URLS` 변수를 사용하여 특정 계측에 대해서만 URL을
제외할 수도 있다. 여기서 library는 다음 중 하나의 대문자 버전이다. Django,
Falcon, FastAPI, Flask, Pyramid, Requests, Starlette, Tornado, urllib, urllib3.

예시:

```sh
export OTEL_PYTHON_EXCLUDED_URLS="client/.*/info,healthcheck"
export OTEL_PYTHON_URLLIB3_EXCLUDED_URLS="client/.*/info"
export OTEL_PYTHON_REQUESTS_EXCLUDED_URLS="healthcheck"
```

### 요청 속성 이름 {#request-attribute-names}

요청 객체에서 추출하여 스팬의 속성(attribute)으로 설정할 이름을 쉼표로 구분한
목록이다.

- `OTEL_PYTHON_DJANGO_TRACED_REQUEST_ATTRS`
- `OTEL_PYTHON_FALCON_TRACED_REQUEST_ATTRS`
- `OTEL_PYTHON_TORNADO_TRACED_REQUEST_ATTRS`

예시:

```sh
export OTEL_PYTHON_DJANGO_TRACED_REQUEST_ATTRS='path_info,content_type'
export OTEL_PYTHON_FALCON_TRACED_REQUEST_ATTRS='query_string,uri_template'
export OTEL_PYTHON_TORNADO_TRACED_REQUEST_ATTRS='uri,query'
```

### 로깅 {#logging}

출력되는 로그를 제어하는 데 사용되는 구성 옵션이 몇 가지 있다.

- `OTEL_PYTHON_LOG_CORRELATION`: 로그에 트레이스 컨텍스트 주입을
  활성화한다(true, false).
- `OTEL_PYTHON_LOG_FORMAT`: 계측이 사용자 정의 로깅 형식을 사용하도록 지시한다.
- `OTEL_PYTHON_LOG_LEVEL`: 사용자 정의 로그 레벨을 설정한다(info, error, debug,
  warning).
- `OTEL_PYTHON_LOG_AUTO_INSTRUMENTATION`: 로깅 핸들러가 자동으로 구성되는지
  여부를 제어하며(true, false), 기본적으로 활성화되어 있다.
  [로그 자동 계측](/docs/zero-code/python/logs-example/)을 참고한다.
- `OTEL_PYTHON_LOG_CODE_ATTRIBUTES`: 로그에 `code` 속성(`code.file.path`,
  `code.function.name`, `code.line.number`) 추가를 활성화한다(true, false).

예시:

```sh
export OTEL_PYTHON_LOG_CORRELATION=true
export OTEL_PYTHON_LOG_FORMAT="%(msg)s [span_id=%(span_id)s]"
export OTEL_PYTHON_LOG_LEVEL=debug
export OTEL_PYTHON_LOG_AUTO_INSTRUMENTATION=false
export OTEL_PYTHON_LOG_CODE_ATTRIBUTES=true
```

> 오픈텔레메트리(OpenTelemetry) Python 1.40.0 이전에는 로그 자동 계측이
> 기본적으로 비활성화되어 있었고 `opentelemetry-sdk` 패키지에 구현되어 있었다.
> `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED`를 `true`로 설정하면
> 활성화되었다.

### 기타 {#other}

특정 범주에 속하지 않는, 설정할 수 있는 구성 옵션이 몇 가지 더 있다.

- `OTEL_PYTHON_DJANGO_INSTRUMENT`: Django 계측의 기본 활성화 상태를
  비활성화하려면 `false`로 설정한다.
- `OTEL_PYTHON_ELASTICSEARCH_NAME_PREFIX`: Elasticsearch 오퍼레이션 이름의 기본
  접두사를 "Elasticsearch"에서 여기에 지정한 값으로 변경한다.
- `OTEL_PYTHON_GRPC_EXCLUDED_SERVICES`: gRPC 계측에서 제외할 특정 서비스를
  쉼표로 구분한 목록이다.
- `OTEL_PYTHON_ID_GENERATOR`: 전역 Tracer Provider에 사용할 ID 생성기를
  지정한다.
- `OTEL_PYTHON_INSTRUMENTATION_SANITIZE_REDIS`: 쿼리 정제를 활성화한다.
- `OTEL_PYTHON_AUTO_INSTRUMENTATION_EXPERIMENTAL_GEVENT_PATCH`: SDK를 초기화하기
  전에 gevent monkeypatch의 `patch_all` 메서드를 호출하려면 `patch_all`로
  설정한다.

예시:

```sh
export OTEL_PYTHON_DJANGO_INSTRUMENT=false
export OTEL_PYTHON_ELASTICSEARCH_NAME_PREFIX=my-custom-prefix
export OTEL_PYTHON_GRPC_EXCLUDED_SERVICES="GRPCTestServer,GRPCHealthServer"
export OTEL_PYTHON_ID_GENERATOR=xray
export OTEL_PYTHON_INSTRUMENTATION_SANITIZE_REDIS=true
export OTEL_PYTHON_AUTO_INSTRUMENTATION_EXPERIMENTAL_GEVENT_PATCH=patch_all
```

## 특정 계측 비활성화 {#disabling-specific-instrumentations}

Python 에이전트는 기본적으로 Python 프로그램의 패키지를 감지하여 계측할 수 있는
모든 패키지를 계측한다. 이는 계측을 쉽게 만들어 주지만, 데이터가 너무 많거나
원치 않는 데이터가 생길 수 있다.

`OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` 환경 변수를 사용하여 특정 패키지를
계측에서 제외할 수 있다. 이 환경 변수는 계측에서 제외할 계측 엔트리 포인트
이름을 쉼표로 구분한 목록으로 설정할 수 있다. 대부분의 경우 엔트리 포인트 이름은
패키지 이름과 동일하며, 패키지의 `pyproject.toml` 파일에 있는
`project.entry-points.opentelemetry_instrumentor` 테이블에 설정되어 있다.

예를 들어 Python 프로그램이 `redis`, `kafka-python`, `grpc` 패키지를 사용하는
경우, 기본적으로 에이전트는 이들을 계측하기 위해
`opentelemetry-instrumentation-redis`,
`opentelemetry-instrumentation-kafka-python`,
`opentelemetry-instrumentation-grpc` 패키지를 사용한다. 이를 비활성화하려면
`OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=redis,kafka,grpc_client`로 설정할 수
있다.
