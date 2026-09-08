---
title: Python
description: >-
  <img width="35" class="img-initial otel-icon"
  src="/img/logos/32x32/Python_SDK.svg" alt="Python"> Python에서의
  오픈텔레메트리(OpenTelemetry) 언어별 구현체이다.
aliases: [/python, /python/metrics, /python/tracing]
weight: 190
default_lang_commit: 3520383f58d1519e9feee8ef32907c3c0c6b9cde
---

{{% docs/languages/index-intro python /%}}

## 버전 지원 {#version-support}

OpenTelemetry-Python은 Python 3.10 이상을 지원한다.

## 설치 {#installation}

API 및 SDK 패키지는 PyPI에서 제공되며, pip을 통해 설치할 수 있다.

```sh
pip install opentelemetry-api
pip install opentelemetry-sdk
```

또한 별도로 설치할 수 있는 여러 확장 패키지가 있다.

```sh
pip install opentelemetry-exporter-{exporter}
pip install opentelemetry-instrumentation-{instrumentation}
```

이는 각각 익스포터와 계측 라이브러리를 위한 것이다. Jaeger, Zipkin, Prometheus,
OTLP, OpenCensus 익스포터는 저장소의
[exporter](https://github.com/open-telemetry/opentelemetry-python/blob/main/exporter/)
디렉터리에서 찾을 수 있다. 계측 라이브러리와 추가 익스포터는 contrib 저장소의
[instrumentation](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/instrumentation)
및
[exporter](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/exporter)
디렉터리에서 찾을 수 있다.

## 확장 {#extensions}

익스포터, 계측 라이브러리, 트레이서 구현체 등 관련 프로젝트를 찾으려면
[레지스트리](/ecosystem/registry/?s=python)를 참고한다.

### 최신 개발 버전 패키지 설치 {#installing-cutting-edge-packages}

아직 PyPI에 릴리스되지 않은 기능이 일부 존재한다. 이 경우 저장소에서 직접
패키지를 설치할 수 있다. 저장소를 클론한 뒤
[editable install](https://pip.pypa.io/en/stable/topics/local-project-installs/#editable-installs)을
수행하면 된다.

```sh
git clone https://github.com/open-telemetry/opentelemetry-python.git
cd opentelemetry-python
pip install -e ./opentelemetry-api -e ./opentelemetry-sdk -e ./opentelemetry-semantic-conventions
```

## 저장소 및 벤치마크 {#repositories-and-benchmarks}

- 메인 저장소: [opentelemetry-python][]
- Contrib 저장소: [opentelemetry-python-contrib][]

[opentelemetry-python]: https://github.com/open-telemetry/opentelemetry-python
[opentelemetry-python-contrib]:
  https://github.com/open-telemetry/opentelemetry-python-contrib
