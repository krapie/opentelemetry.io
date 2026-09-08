---
title: Python 제로 코드 계측
linkTitle: Python
weight: 40
aliases: [/docs/languages/python/automatic]
cascade:
  collector_vers: 0.159.0
cSpell:ignore: distro
default_lang_commit: de42414bea5449f67be2211e96836fbc70660548
---

Python의 자동 계측(Automatic instrumentation)은 모든 Python 애플리케이션에
연결할 수 있는 Python 에이전트(agent)를 사용한다. 이 에이전트는 주로
[몽키 패칭(monkey patching)](https://en.wikipedia.org/wiki/Monkey_patch)을
사용하여 런타임에 라이브러리 함수를 수정함으로써, 다양한 인기 라이브러리 및
프레임워크로부터 텔레메트리 데이터를 캡처할 수 있게 한다.

## 설정 {#setup}

적절한 패키지를 설치하려면 다음 명령을 실행한다.

```sh
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

`opentelemetry-distro` 패키지는 API, SDK, 그리고 `opentelemetry-bootstrap` 및
`opentelemetry-instrument` 도구를 설치한다.

> [!NOTE]
>
> 자동 계측이 동작하려면 배포판(distro) 패키지를 설치해야 한다.
> `opentelemetry-distro` 패키지에는 사용자를 위해 일부 공통 옵션을 자동으로
> 구성하는 기본 배포판(distro)이 포함되어 있다. 자세한 내용은
> [오픈텔레메트리 배포판(distro)](/docs/languages/python/distro/)을 참고한다.

`opentelemetry-bootstrap -a install` 명령은 활성 `site-packages` 폴더에 설치된
패키지 목록을 읽어, 해당하는 경우 이 패키지들에 대응하는 계측 라이브러리를
설치한다. 예를 들어 이미 `flask` 패키지를 설치했다면,
`opentelemetry-bootstrap -a install`을 실행하면
`opentelemetry-instrumentation-flask`가 설치된다. 오픈텔레메트리 Python
에이전트는 몽키 패칭을 사용하여 런타임에 이 라이브러리들의 함수를 수정한다.

인자 없이 `opentelemetry-bootstrap`을 실행하면 설치가 권장되는 계측 라이브러리
목록이 표시된다. 자세한 내용은
[`opentelemetry-bootstrap`](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/opentelemetry-instrumentation#opentelemetry-bootstrap)을
참고한다.

> [!WARNING] `uv` 사용 중인가?
>
> [uv](https://docs.astral.sh/uv/) 패키지 관리자를 사용 중이라면,
> `opentelemetry-bootstrap -a install`을 실행할 때 어려움을 겪을 수 있다. 자세한
> 내용은 [uv로 부트스트랩하기](troubleshooting/#bootstrap-using-uv)를 참고한다.

{#configuring-the-agent}

## 에이전트 구성하기

에이전트는 매우 유연하게 구성할 수 있다.

한 가지 방법은 CLI에서 구성 속성(property)을 통해 에이전트를 구성하는 것이다.

```sh
opentelemetry-instrument \
    --traces_exporter console,otlp \
    --metrics_exporter console \
    --service_name your-service-name \
    --exporter_otlp_endpoint 0.0.0.0:4317 \
    python myapp.py
```

또는 환경 변수를 사용하여 에이전트를 구성할 수도 있다.

```sh
OTEL_SERVICE_NAME=your-service-name \
OTEL_TRACES_EXPORTER=console,otlp \
OTEL_METRICS_EXPORTER=console \
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=0.0.0.0:4317
opentelemetry-instrument \
    python myapp.py
```

전체 구성 옵션을 확인하려면 [에이전트 구성](configuration)을 참고한다.

## 지원되는 라이브러리 및 프레임워크 {#supported-libraries-and-frameworks}

여러 인기 Python 라이브러리가 자동으로 계측되며, 그 예로
[Flask](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/instrumentation/opentelemetry-instrumentation-flask)와
[Django](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/instrumentation/opentelemetry-instrumentation-django)가
있다. 전체 목록은
[레지스트리(Registry)](/ecosystem/registry/?language=python&component=instrumentation)를
참고한다.

## 문제 해결 {#troubleshooting}

일반적인 문제 해결 단계와 특정 문제에 대한 해결 방법은
[문제 해결](./troubleshooting/)을 참고한다.
