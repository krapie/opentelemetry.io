---
title: 로그 자동 계측 예제
linkTitle: 로그 예제
weight: 20
aliases: [/docs/languages/python/automatic/logs-example]
cSpell:ignore: distro mkdir virtualenv
default_lang_commit: 7a39e1b95f51cf97fe203ef98a1011d3be33d77e
---

이 페이지에서는 오픈텔레메트리(OpenTelemetry)에서 Python 로그 자동
계측(auto-instrumentation)을 사용하는 방법을 보여준다.

트레이스(Traces)나 메트릭(Metrics)과 달리, 로그에는 이에 상응하는 API가 없다.
SDK만 존재한다. Python에서는 Python `logger` 라이브러리를 사용하며, OTel SDK가
루트 로거에 OTLP 핸들러를 연결하여 Python 로거를 OTLP 로거로 바꾼다. 이를
수행하는 한 가지 방법은 [OpenTelemetry Python
저장소][OpenTelemetry Python repository]의 로그 예제에 문서화되어 있다.

이를 수행하는 또 다른 방법은 Python의 로그 자동 계측 지원을 통하는 것이다. 아래
예제는 [OpenTelemetry Python 저장소][OpenTelemetry Python repository]의 로그
예제를 기반으로 한다.

> 로그 브리지 API가 있기는 하지만, 이는 트레이스나 메트릭 API와 다르다.
> 애플리케이션 개발자가 로그를 생성하는 데 사용하지 않기 때문이다. 대신,
> 개발자는 이 브리지 API를 사용하여 표준 언어별 로깅 라이브러리에 로그
> 어펜더(appender)를 설정한다. 자세한 내용은
> [로그 API](/docs/specs/otel/logs/api/)를 참고한다.

먼저 예제 디렉터리와 예제 Python 파일을 생성한다.

```sh
mkdir python-logs-example
cd python-logs-example
touch example.py
```

다음 내용을 `example.py`에 붙여 넣는다.

```python
import logging

from opentelemetry import trace

tracer = trace.get_tracer_provider().get_tracer(__name__)

# Trace context correlation
with tracer.start_as_current_span("foo"):
    # Do something
    current_span = trace.get_current_span()
    current_span.add_event("This is a span event")
    logging.getLogger().error("This is a log message")
```

[otel-collector-config.yaml](https://github.com/open-telemetry/opentelemetry-python/blob/main/docs/examples/logs/otel-collector-config.yaml)
예제를 열어서 복사한 뒤, `python-logs-example/otel-collector-config.yaml`에
저장한다.

## 준비 {#prepare}

다음 예제를 실행하며, 이때 가상 환경 사용을 권장한다. 로그 자동 계측을
준비하려면 다음 명령을 실행한다.

```sh
mkdir python_logs_example
virtualenv python_logs_example
source python_logs_example/bin/activate
```

## 설치 {#install}

다음 명령은 적절한 패키지를 설치한다. `opentelemetry-distro` 패키지는 직접
작성한 코드를 사용자 정의 계측하기 위한 `opentelemetry-sdk`와, 프로그램을
자동으로 계측하는 데 도움이 되는 여러 명령을 제공하는
`opentelemetry-instrumentation` 등 몇몇 다른 패키지에 의존한다.

```sh
pip install opentelemetry-distro
pip install opentelemetry-exporter-otlp
pip install opentelemetry-instrumentation-logging
```

다음에 이어지는 예제는 계측 결과를 콘솔로 전송한다. 오픈텔레메트리
컬렉터(Collector)와 같은 다른 대상으로 텔레메트리를 전송하도록
[오픈텔레메트리 배포판(Distro)](/docs/languages/python/distro)을 설치하고
구성하는 방법을 자세히 알아본다.

> **참고**: `opentelemetry-instrument`를 통해 자동 계측을 사용하려면 환경 변수나
> 명령줄로 구성해야 한다. 에이전트는 이러한 수단을 통해서만 수정할 수 있는
> 텔레메트리 파이프라인을 만든다. 텔레메트리 파이프라인을 더 세부적으로 사용자
> 정의해야 한다면, 에이전트를 포기하고 오픈텔레메트리 SDK와 계측 라이브러리를
> 코드로 가져와 그곳에서 구성해야 한다. 또한 오픈텔레메트리 API를 가져와서 자동
> 계측을 확장할 수도 있다. 자세한 내용은 [API 참고 문서][api reference]를
> 참고한다.

## 실행 {#execute}

이 섹션에서는 자동으로 계측된 로그를 실행하는 과정을 안내한다.

새 터미널 창을 열고 OTel 컬렉터를 시작한다.

```sh
docker run -it --rm -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/otel-collector-config.yaml:/etc/otelcol-config.yml \
  --name otelcol \
  otel/opentelemetry-collector:{{% param collector_vers %}} \
  "--config=/etc/otelcol-config.yml"
```

다른 터미널을 열고 Python 프로그램을 실행한다.

```sh
source python_logs_example/bin/activate

opentelemetry-instrument \
  --traces_exporter console,otlp \
  --metrics_exporter console,otlp \
  --logs_exporter console,otlp \
  --service_name python-logs-example \
  python $(pwd)/example.py
```

> 오픈텔레메트리 Python 1.40.0 이전에는
> `export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`로 로그 계측을
> 활성화해야 했다.

출력 예시:

```text
...
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope __main__
Span #0
    Trace ID       : 389d4ac130a390d3d99036f9cd1db75e
    Parent ID      :
    ID             : f318281c4654edc5
    Name           : foo
    Kind           : Internal
    Start time     : 2023-08-18 17:04:05.982564 +0000 UTC
    End time       : 2023-08-18 17:04:05.982667 +0000 UTC
    Status code    : Unset
    Status message :
Events:
SpanEvent #0
     -> Name: This is a span event
     -> Timestamp: 2023-08-18 17:04:05.982586 +0000 UTC

...

ScopeLogs #0
ScopeLogs SchemaURL:
InstrumentationScope opentelemetry.sdk._logs._internal
LogRecord #0
ObservedTimestamp: 1970-01-01 00:00:00 +0000 UTC
Timestamp: 2023-08-18 17:04:05.982605056 +0000 UTC
SeverityText: ERROR
SeverityNumber: Error(17)
Body: Str(This is a log message)
Attributes:
     -> otelSpanID: Str(f318281c4654edc5)
     -> otelTraceID: Str(389d4ac130a390d3d99036f9cd1db75e)
     -> otelTraceSampled: Bool(true)
     -> otelServiceName: Str(python-logs-example)
Trace ID: 389d4ac130a390d3d99036f9cd1db75e
Span ID: f318281c4654edc5
...
```

Span Event와 Log가 모두 동일한 SpanID(`f318281c4654edc5`)를 가진다는 점에
유의한다. 로깅 SDK는 텔레메트리를 상관 짓는 능력을 개선하기 위해 현재 Span의
SpanID를 로그로 남겨진 모든 이벤트에 덧붙인다.

[api reference]:
  https://opentelemetry-python.readthedocs.io/en/latest/index.html
[OpenTelemetry Python repository]:
  https://github.com/open-telemetry/opentelemetry-python/tree/main/docs/examples/logs
