---
title: 자동 계측 예제
linkTitle: 예제
weight: 20
aliases: [/docs/languages/python/automatic/example]
# prettier-ignore
cSpell:ignore: Aiohttp ASGI distro instrumentor mkdir MSIE Referer Starlette venv
default_lang_commit: 04a14dc454fac98bb79eb2e2bc13e680733a832f
---

이 페이지에서는 오픈텔레메트리(OpenTelemetry)에서 Python 자동 계측(automatic
instrumentation)을 사용하는 방법을 보여준다. 이 예제는 [OpenTracing
예제][opentracing example]를 기반으로 한다. 이 페이지에서 사용하는 [소스
파일][source files]은 `opentelemetry-python` 저장소에서 내려받거나 확인할 수
있다.

이 예제는 서로 다른 세 가지 스크립트를 사용한다. 이들의 주요 차이는 계측되는
방식이다.

1. `server_manual.py`는 _수동으로_ 계측된다.
2. `server_automatic.py`는 _자동으로_ 계측된다.
3. `server_programmatic.py`는 _프로그래밍 방식으로_ 계측된다.

[_프로그래밍 방식_ 계측](#execute-the-programmatically-instrumented-server)은
애플리케이션에 추가해야 하는 계측 코드가 최소한으로 필요한 계측 방식이다. 일부
계측 라이브러리만이 프로그래밍 방식으로 사용할 때 계측 과정을 더 세밀하게 제어할
수 있는 추가 기능을 제공한다.

첫 번째 스크립트는 자동 계측 에이전트(agent) 없이 실행하고, 두 번째 스크립트는
에이전트와 함께 실행한다. 두 스크립트는 동일한 결과를 내야 하며, 이는 자동 계측
에이전트가 수동 계측과 정확히 같은 일을 한다는 것을 보여준다.

자동 계측은 [몽키 패칭(monkey-patching)][monkey-patching]을 활용하여 [계측
라이브러리][instrumentation]를 통해 런타임에 메서드와 클래스를 동적으로 다시
작성한다. 이렇게 하면 오픈텔레메트리를 애플리케이션 코드에 통합하는 데 필요한
작업량이 줄어든다. 아래에서 수동, 자동, 그리고 프로그래밍 방식으로 계측된 Flask
라우트의 차이를 볼 수 있다.

## 수동으로 계측된 서버 {#manually-instrumented-server}

`server_manual.py`

```python
@app.route("/server_request")
def server_request():
    with tracer.start_as_current_span(
        "server_request",
        context=extract(request.headers),
        kind=trace.SpanKind.SERVER,
        attributes=collect_request_attributes(request.environ),
    ):
        print(request.args.get("param"))
        return "served"
```

## 자동으로 계측된 서버 {#automatically-instrumented-server}

`server_automatic.py`

```python
@app.route("/server_request")
def server_request():
    print(request.args.get("param"))
    return "served"
```

## 프로그래밍 방식으로 계측된 서버 {#programmatically-instrumented-server}

`server_programmatic.py`

```python
instrumentor = FlaskInstrumentor()

app = Flask(__name__)

instrumentor.instrument_app(app)
# instrumentor.instrument_app(app, excluded_urls="/server_request")
@app.route("/server_request")
def server_request():
    print(request.args.get("param"))
    return "served"
```

## 준비 {#prepare}

다음 예제는 별도의 가상 환경(virtual environment)에서 실행한다. 자동 계측을
준비하려면 다음 명령을 실행한다.

```sh
mkdir auto_instrumentation
cd auto_instrumentation
python -m venv venv
source ./venv/bin/activate
```

## 설치 {#install}

적절한 패키지를 설치하려면 다음 명령을 실행한다. `opentelemetry-distro` 패키지는
직접 작성한 코드를 사용자 정의 계측하기 위한 `opentelemetry-sdk`와, 프로그램을
자동으로 계측하는 데 도움이 되는 여러 명령을 제공하는
`opentelemetry-instrumentation` 등 몇몇 다른 패키지에 의존한다.

```sh
pip install opentelemetry-distro
pip install flask requests
```

`opentelemetry-bootstrap` 명령을 실행한다.

```shell
opentelemetry-bootstrap -a install
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

이 섹션에서는 서버를 수동으로 계측하는 과정과 자동으로 계측된 서버를 실행하는
과정을 안내한다.

## 수동으로 계측된 서버 실행하기 {#execute-the-manually-instrumented-server}

이 예제를 구성하는 각 스크립트를 실행하기 위해, 서버를 별도의 콘솔 두 개에서
실행한다.

```sh
source ./venv/bin/activate
python server_manual.py
```

```sh
source ./venv/bin/activate
python client.py
```

`server_manual.py`를 실행하는 콘솔에는 계측으로 생성된 스팬(span)이 JSON으로
표시된다. 스팬은 다음 예시와 비슷하게 나타난다.

```json
{
  "name": "server_request",
  "context": {
    "trace_id": "0xfa002aad260b5f7110db674a9ddfcd23",
    "span_id": "0x8b8bbaf3ca9c5131",
    "trace_state": "{}"
  },
  "kind": "SpanKind.SERVER",
  "parent_id": null,
  "start_time": "2020-04-30T17:28:57.886397Z",
  "end_time": "2020-04-30T17:28:57.886490Z",
  "status": {
    "status_code": "OK"
  },
  "attributes": {
    "http.method": "GET",
    "http.server_name": "127.0.0.1",
    "http.scheme": "http",
    "host.port": 8082,
    "http.host": "localhost:8082",
    "http.target": "/server_request?param=testing",
    "net.peer.ip": "127.0.0.1",
    "net.peer.port": 52872,
    "http.flavor": "1.1"
  },
  "events": [],
  "links": [],
  "resource": {
    "telemetry.sdk.language": "python",
    "telemetry.sdk.name": "opentelemetry",
    "telemetry.sdk.version": "0.16b1"
  }
}
```

## 자동으로 계측된 서버 실행하기 {#execute-the-automatically-instrumented-server}

<kbd>Control+C</kbd>를 눌러 `server_manual.py` 실행을 중지하고, 대신 다음 명령을
실행한다.

```sh
opentelemetry-instrument --traces_exporter console --metrics_exporter none --logs_exporter none python server_automatic.py
```

이전에 `client.py`를 실행했던 콘솔에서 다음 명령을 다시 실행한다.

```sh
python client.py
```

`server_automatic.py`를 실행하는 콘솔에는 계측으로 생성된 스팬이 JSON으로
표시된다. 스팬은 다음 예시와 비슷하게 나타난다.

```json
{
  "name": "server_request",
  "context": {
    "trace_id": "0x9f528e0b76189f539d9c21b1a7a2fc24",
    "span_id": "0xd79760685cd4c269",
    "trace_state": "{}"
  },
  "kind": "SpanKind.SERVER",
  "parent_id": "0xb4fb7eee22ef78e4",
  "start_time": "2020-04-30T17:10:02.400604Z",
  "end_time": "2020-04-30T17:10:02.401858Z",
  "status": {
    "status_code": "OK"
  },
  "attributes": {
    "http.method": "GET",
    "http.server_name": "127.0.0.1",
    "http.scheme": "http",
    "host.port": 8082,
    "http.host": "localhost:8082",
    "http.target": "/server_request?param=testing",
    "net.peer.ip": "127.0.0.1",
    "net.peer.port": 48240,
    "http.flavor": "1.1",
    "http.route": "/server_request",
    "http.status_text": "OK",
    "http.status_code": 200
  },
  "events": [],
  "links": [],
  "resource": {
    "telemetry.sdk.language": "python",
    "telemetry.sdk.name": "opentelemetry",
    "telemetry.sdk.version": "0.16b1",
    "service.name": ""
  }
}
```

자동 계측이 수동 계측과 정확히 같은 일을 하기 때문에 두 출력이 동일하다는 것을
볼 수 있다.

## 프로그래밍 방식으로 계측된 서버 실행하기 {#execute-the-programmatically-instrumented-server}

`opentelemetry-instrumentation-flask`와 같은 계측 라이브러리를 단독으로 사용하는
것도 가능하며, 이는 옵션을 사용자 정의할 수 있다는 장점이 있다. 다만 이렇게
하기로 선택하면, `opentelemetry-instrument`로 애플리케이션을 시작하여 자동
계측을 사용하는 방식은 포기하게 된다. 두 방식은 상호 배타적이기 때문이다.

수동 계측에서와 마찬가지로, 이 예제를 구성하는 각 스크립트를 실행하기 위해
서버를 별도의 콘솔 두 개에서 실행한다.

```sh
source ./venv/bin/activate
python server_programmatic.py
```

```sh
source ./venv/bin/activate
python client.py
```

결과는 수동 계측으로 실행할 때와 동일해야 한다.

### 프로그래밍 방식 계측 기능 사용하기 {#using-programmatic-instrumentation-features}

일부 계측 라이브러리에는 프로그래밍 방식으로 계측하는 동안 더 정밀하게 제어할 수
있는 기능이 포함되어 있으며, Flask용 계측 라이브러리가 그중 하나다.

이 예제에는 주석 처리된 줄이 하나 있는데, 다음과 같이 변경한다.

```python
# instrumentor.instrument_app(app)
instrumentor.instrument_app(app, excluded_urls="/server_request")
```

예제를 다시 실행하면 서버 측에 계측이 나타나지 않아야 한다. 이는
`instrument_app`에 전달된 `excluded_urls` 옵션 때문인데, `server_request`의
URL이 `excluded_urls`에 전달된 정규 표현식과 일치하므로 `server_request` 함수가
계측되지 않도록 사실상 막는다.

## 디버깅 중 계측 {#instrumentation-while-debugging}

Flask 앱에서는 다음과 같이 디버그 모드를 활성화할 수 있다.

```python
if __name__ == "__main__":
    app.run(port=8082, debug=True)
```

디버그 모드는 리로더(reloader)를 활성화하기 때문에 계측이 일어나지 않도록 방해할
수 있다. 디버그 모드가 활성화된 상태에서 계측을 실행하려면 `use_reloader` 옵션을
`False`로 설정한다.

```python
if __name__ == "__main__":
    app.run(port=8082, debug=True, use_reloader=False)
```

## 구성 {#configure}

자동 계측은 환경 변수에서 구성을 가져올 수 있다.

## HTTP 요청 및 응답 헤더 캡처 {#capture-http-request-and-response-headers}

[시맨틱 컨벤션(semantic convention)][semantic convention]에 따라 미리 정의된
HTTP 헤더를 스팬 속성(attribute)으로 캡처할 수 있다.

캡처하려는 HTTP 헤더를 정의하려면, 환경 변수
`OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_REQUEST`와
`OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_RESPONSE`를 통해 HTTP 헤더
이름을 쉼표로 구분한 목록으로 제공한다. 예를 들면 다음과 같다.

```sh
export OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_REQUEST="Accept-Encoding,User-Agent,Referer"
export OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_RESPONSE="Last-Modified,Content-Type"
opentelemetry-instrument --traces_exporter console --metrics_exporter none --logs_exporter none python app.py
```

이러한 구성 옵션은 다음 HTTP 계측에서 지원된다.

- Aiohttp-server
- ASGI
- Django
- Falcon
- FastAPI
- Flask
- Pyramid
- Starlette
- Tornado
- WSGI

해당 헤더를 사용할 수 있으면 스팬에 포함된다.

```json
{
  "attributes": {
    "http.request.header.user-agent": [
      "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0)"
    ],
    "http.request.header.accept_encoding": ["gzip, deflate, br"],
    "http.response.header.last_modified": ["2022-04-20 17:07:13.075765"],
    "http.response.header.content_type": ["text/html; charset=utf-8"]
  }
}
```

### 캡처된 헤더의 정제 {#sanitization-of-captured-headers}

개인 식별 정보(personally identifiable information, PII), 세션 키, 비밀번호 등
민감한 데이터가 저장되는 것을 방지하려면, 환경 변수
`OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SANITIZE_FIELDS`를 정제할 HTTP 헤더
이름을 쉼표로 구분한 목록으로 설정한다. 정규 표현식을 사용할 수 있으며, 모든
헤더 이름은 대소문자를 구분하지 않고 매칭된다.

예를 들어,

```sh
export OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SANITIZE_FIELDS=".*session.*,set-cookie"
```

는 스팬에서 `session-id`, `set-cookie` 같은 헤더의 값을 `[REDACTED]`로 바꾼다.

[semantic convention]: /docs/specs/semconv/http/http-spans/
[api reference]:
  https://opentelemetry-python.readthedocs.io/en/latest/index.html
[instrumentation]:
  https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/opentelemetry-instrumentation
[monkey-patching]:
  https://stackoverflow.com/questions/5626193/what-is-monkey-patching?link-check=no&last-validated=2026-08-02
[opentracing example]:
  https://github.com/yurishkuro/opentracing-tutorial/tree/master/python
[source files]:
  https://github.com/open-telemetry/opentelemetry-python/tree/main/docs/examples/auto-instrumentation
