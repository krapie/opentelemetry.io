---
title: Python 자동 계측 문제 해결
linkTitle: 문제 해결
weight: 40
cSpell:ignore: ASGI gunicorn uvicorn
default_lang_commit: 9d436315001f7d398401b064b7a8a095750f1c73
---

## 설치 문제 {#installation-issues}

### Python 패키지 설치 실패 {#python-package-installation-failure}

Python 패키지 설치에는 `gcc`와 `gcc-c++`가 필요하며, CentOS와 같은 슬림 버전의
Linux를 실행 중이라면 이를 설치해야 할 수도 있다.

<!-- markdownlint-disable blanks-around-fences -->

{{< tabpane text=true >}} {{% tab "CentOS" %}}

```sh
yum -y install python3-devel
yum -y install gcc-c++
```

{{% /tab %}} {{% tab "Debian/Ubuntu" %}}

```sh
apt install -y python3-dev
apt install -y build-essential
```

{{% /tab %}} {{% tab "Alpine" %}}

```sh
apk add python3-dev
apk add build-base
```

{{% /tab %}} {{< /tabpane >}}

### uv로 부트스트랩하기 {#bootstrap-using-uv}

[uv](https://docs.astral.sh/uv/) 패키지 관리자를 사용할 때
`opentelemetry-bootstrap -a install`을 실행하면 의존성 설정이 오류가 나거나
예상과 다르게 구성될 수 있다.

대신, 오픈텔레메트리(OpenTelemetry) 요구 사항을 동적으로 생성하여 `uv`로 설치할
수 있다.

먼저 적절한 패키지를 설치한다(또는 프로젝트 파일에 추가하고 `uv sync`를
실행한다).

```sh
uv add opentelemetry-distro opentelemetry-exporter-otlp
```

이제 자동 계측을 설치할 수 있다.

```sh
uv run opentelemetry-bootstrap -a requirements | uv add --requirement -
```

마지막으로 `uv run`으로 애플리케이션을 시작한다(자세한 내용은
[에이전트 구성하기](/docs/zero-code/python/#configuring-the-agent)를 참고한다).

```sh
uv run opentelemetry-instrument python myapp.py
```

`uv sync`를 실행하거나 기존 패키지를 업데이트할 때마다 자동 계측을 다시 설치해야
한다는 점에 유의한다. 따라서 설치를 빌드 파이프라인의 일부로 만드는 것을
권장한다.

## 계측 문제 {#instrumentation-issues}

### 리로더가 있는 Flask 디버그 모드가 계측을 방해함 {#flask-debug-mode-with-reloader-breaks-instrumentation}

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

### 프리포크(pre-fork) 서버 문제 {#pre-fork-server-issues}

워커가 여러 개인 Gunicorn과 같은 프리포크 서버는 다음과 같이 실행할 수 있다.

```sh
gunicorn myapp.main:app --workers 4
```

그러나 `--workers`를 두 개 이상 지정하면 자동 계측이 적용될 때 메트릭 생성이
깨질 수 있다. 이는 포크, 즉 워커/자식 프로세스의 생성이 주요 오픈텔레메트리 SDK
구성 요소가 가정하는 백그라운드 스레드와 락(lock) 사이에 각 자식마다 불일치를
일으키기 때문이다. 구체적으로, `PeriodicExportingMetricReader`는 자체 스레드를
생성하여 주기적으로 데이터를 익스포터로 플러시한다. 이슈
[#2767](https://github.com/open-telemetry/opentelemetry-python/issues/2767)과
[#3307](https://github.com/open-telemetry/opentelemetry-python/issues/3307#issuecomment-1579101152)도
참고한다. 포크 이후 각 자식은 메모리에서 실제로 실행되지 않는 스레드 객체를
찾으며, 원래의 락은 각 자식에 대해 해제되지 않을 수 있다.
[Python 이슈 6721](https://bugs.python.org/issue6721)에 설명된 포크와
데드락(deadlock)도 참고한다.

#### 해결 방법 {#workarounds}

오픈텔레메트리로 프리포크 서버를 사용할 때 쓸 수 있는 몇 가지 해결 방법이 있다.
다음 표는 워커를 여러 개로 프리포크한, 자동 계측된 여러 웹 서버 게이트웨이
스택의 시그널 내보내기(export) 지원 현황을 요약한 것이다. 자세한 내용과 선택지는
아래를 참고한다.

| 워커가 여러 개인 스택    | 트레이스 | 메트릭 | 로그 |
| ------------------------ | -------- | ------ | ---- |
| Uvicorn                  | x        |        | x    |
| Gunicorn                 | x        |        | x    |
| Gunicorn + UvicornWorker | x        | x      | x    |

##### Gunicorn과 UvicornWorker로 배포하기 {#deploy-with-gunicorn-and-uvicornworker}

워커가 여러 개인 서버를 자동 계측하려면, ASGI(Asynchronous Server Gateway
Interface) 앱(FastAPI, Starlette 등)인 경우 `uvicorn.workers.UvicornWorker`와
함께 Gunicorn을 사용하여 배포하는 것을 권장한다. UvicornWorker 클래스는
백그라운드 프로세스와 스레드를 보존하면서 포크를 처리하도록 특별히 설계되었다.
예를 들면 다음과 같다.

```sh
opentelemetry-instrument gunicorn \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  myapp.main:app
```

##### 프로그래밍 방식 자동 계측 사용하기 {#use-programmatic-auto-instrumentation}

`opentelemetry-instrument` 대신, 서버 포크 이후 워커 프로세스 내부에서
[프로그래밍 방식 자동 계측](https://github.com/open-telemetry/opentelemetry-python-contrib/blob/main/opentelemetry-instrumentation/README.rst#programmatic-auto-instrumentation)으로
오픈텔레메트리를 초기화한다. 예를 들면 다음과 같다.

```python
from opentelemetry.instrumentation.auto_instrumentation import initialize
initialize()

from your_app import app
```

FastAPI를 사용하는 경우, 계측이 패치되는 방식 때문에 `FastAPI`를 임포트하기 전에
`initialize()`를 호출해야 한다. 예를 들면 다음과 같다.

```python
from opentelemetry.instrumentation.auto_instrumentation import initialize
initialize()

from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

그런 다음 서버를 다음과 같이 실행한다.

```sh
uvicorn main:app --workers 2
```

##### 프로그래밍 방식 자동 계측을 위해 Gunicorn 포스트포크 사용하기 {#use-gunicorn-post-fork-for-programmatic-auto-instrumentation}

Gunicorn을 사용하는 경우, `opentelemetry-instrument` 대신 포스트포크 훅(hook)의
일부로
[프로그래밍 방식 자동 계측](https://github.com/open-telemetry/opentelemetry-python-contrib/blob/main/opentelemetry-instrumentation/README.rst#programmatic-auto-instrumentation)으로
오픈텔레메트리를 초기화할 수 있다. 예를 들어, `your_app.py`와 나란히 있는
`gunicorn_config.py` 파일에서 다음과 같이 한다.

```python
from opentelemetry.instrumentation.auto_instrumentation import initialize

def post_fork(server, worker):
    initialize()
```

그런 다음 다음과 같이 실행한다.

```sh
gunicorn --config gunicorn_config.py --workers=2 your_app:app
```

##### 직접 OTLP로 Prometheus 사용하기 {#use-prometheus-with-direct-otlp}

OTLP 메트릭을 직접 수신하려면 최신 버전의
[Prometheus](/docs/languages/python/exporters/#prometheus-setup) 사용을
고려한다. `PeriodicExportingMetricReader`와 프로세스당 하나의 OTLP 워커를
설정하여 Prometheus 서버로 푸시한다. 포크와 함께 `PrometheusMetricReader`를
사용하지 _않는_ 것을 권장한다. 이슈
[#3747](https://github.com/open-telemetry/opentelemetry-python/issues/3747)을
참고한다.

##### 단일 워커 사용하기 {#use-a-single-worker}

또는, 제로 코드 계측으로 프리포크에서 단일 워커를 사용한다.

```sh
opentelemetry-instrument gunicorn your_app:app --workers 1
```

## 연결 문제 {#connectivity-issues}

### gRPC 연결 {#grpc-connectivity}

Python gRPC 연결 문제를 디버깅하려면 다음 gRPC 디버그 환경 변수를 설정한다.

```sh
export GRPC_VERBOSITY=debug
export GRPC_TRACE=http,call_error,connectivity_state
opentelemetry-instrument python YOUR_APP.py
```
