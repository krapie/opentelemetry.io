---
title: 오픈텔레메트리 오퍼레이터를 사용하여 자동 계측 주입하기
linkTitle: 오퍼레이터
aliases: [/docs/languages/python/automatic/operator]
weight: 30
cSpell:ignore: gevent grpcio monkeypatch psutil PYTHONPATH
default_lang_commit: 5d68eec62bc16a5558678eae6d2f8c5083113823
---

Python 서비스를 쿠버네티스(Kubernetes)에서 실행한다면,
[오픈텔레메트리 오퍼레이터(Operator)](https://github.com/open-telemetry/opentelemetry-operator)를
활용하여 각 서비스를 직접 수정하지 않고도 자동 계측(auto-instrumentation)을
주입할 수 있다.
[자세한 내용은 오픈텔레메트리 오퍼레이터 자동 계측 문서를 참고한다.](/docs/platforms/kubernetes/operator/automatic/)

## Python 전용 주제 {#python-specific-topics}

### 바이너리 휠(wheel)을 포함하는 라이브러리 {#libraries-with-binary-wheels}

계측 대상이거나 계측 라이브러리에 필요한 일부 Python 패키지는 바이너리 코드와
함께 배포될 수 있다. 예를 들어 `grpcio`와
`psutil`(`opentelemetry-instrumentation-system-metrics`에서 사용됨)이 이에
해당한다.

바이너리 코드는 특정 C 라이브러리 버전(glibc 또는 musl)과 특정 Python 버전에
묶여 있다.
[오픈텔레메트리 오퍼레이터](https://github.com/open-telemetry/opentelemetry-operator)는
glibc C 라이브러리를 기반으로 하는 단일 Python 버전용 이미지를 제공한다. 이를
사용하려면 Python 자동 계측을 위한 자체 이미지 오퍼레이터 Docker 이미지를
빌드해야 할 수도 있다.

오퍼레이터 v0.113.0부터는 glibc 기반과 musl 기반 자동 계측을 모두 포함하는
이미지를 빌드하고
[런타임에 구성](/docs/platforms/kubernetes/operator/automatic/#annotations-python-musl)하는
것이 가능하다.

### Django 애플리케이션 {#django-applications}

Django처럼 자체 실행 파일로 실행되는 애플리케이션은 배포 파일에 다음 두 환경
변수를 설정해야 한다.

- `PYTHONPATH`: Django 애플리케이션 루트 디렉터리 경로(예: "/app")
- `DJANGO_SETTINGS_MODULE`: Django 설정 모듈의 이름(예: "myapp.settings")

### gevent 애플리케이션 {#gevent-applications}

오픈텔레메트리 Python 1.37.0/0.58b0 릴리스부터, 배포 파일에서
`OTEL_PYTHON_AUTO_INSTRUMENTATION_EXPERIMENTAL_GEVENT_PATCH` 환경 변수를
`patch_all`로 설정하면 자동 계측 코드가 스스로 초기화하기 전에 같은 이름의
gevent monkeypatch 메서드를 호출한다.
