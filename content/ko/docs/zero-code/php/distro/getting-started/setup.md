---
title: 오픈텔레메트리 PHP Distro 설정하기
description: >-
  PHP 애플리케이션에서 텔레메트리 데이터를 전송하기 시작하도록
  오픈텔레메트리(OpenTelemetry) PHP Distro를 설치하고 구성하는 방법을 배운다.
weight: 1
cSpell:ignore: apk dpkg fpm RoadRunner Swoole
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리(OpenTelemetry) PHP Distro로 PHP 애플리케이션을 계측하고 OTLP 호환
백엔드로 텔레메트리 데이터를 전송하는 방법을 배운다.

## 사전 요구 사항 {#prerequisites}

- 텔레메트리 데이터를 받을 대상(OTLP 엔드포인트)을 준비한다.
- 지원되는 리눅스 배포판(distribution)과 PHP 버전을 사용한다.
- 같은 프로세스에서 다른 PHP APM이나 오픈텔레메트리 에이전트를 실행하지 않는다.

지원되는 운영체제와 PHP 버전은
[지원 기술](/docs/zero-code/php/distro/reference/supported-technologies/)을
참고한다.

## 제한 사항 {#limitations}

알려진 런타임 및 호환성 제한 사항은
[제한 사항](/docs/zero-code/php/distro/getting-started/limitations/)에 설명되어
있다.

## 패키지 다운로드 및 설치 {#download-and-install-packages}

[GitHub Releases](https://github.com/open-telemetry/opentelemetry-php-distro/releases)
페이지에서 사용하는 플랫폼에 맞는 패키지를 다운로드하여 설치한다.

### RPM(RHEL/CentOS/Fedora) {#rpm-rhelcentosfedora}

```sh
sudo rpm -ivh <package-file>.rpm
```

### DEB(Debian/Ubuntu) {#deb-debianubuntu}

```sh
sudo dpkg -i <package-file>.deb
```

### APK(Alpine) {#apk-alpine}

```sh
sudo apk add --allow-untrusted <package-file>.apk
```

## 익스포터 구성 {#configure-exporter}

최소한 다음을 설정한다.

- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_HEADERS`

예시:

```sh
export OTEL_EXPORTER_OTLP_ENDPOINT="https://your-otlp-endpoint:443/"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <token>"
```

## PHP 프로세스 재시작 {#restart-php-processes}

설치와 구성을 마친 뒤, 확장(extension)이 로드되도록 PHP 프로세스(예: `php-fpm`,
Apache 워커, 장기 실행 CLI 워커)를 재시작한다.

## 텔레메트리 확인 {#confirm-telemetry}

1. 옵저버빌리티(observability) 백엔드를 연다.
2. 트레이스에서 자신의 서비스를 찾는다.
3. 아직 트레이스가 보이지 않으면 트래픽을 발생시킨다.

## 문제 해결 {#troubleshooting}

- [구성](/docs/zero-code/php/distro/reference/configuration/)에서 구성 옵션을
  확인한다.
- [제한 사항](/docs/zero-code/php/distro/getting-started/limitations/)에서
  알려진 제약을 확인한다.
- Laravel Octane(Swoole 또는 RoadRunner)을 사용하는 경우
  [장기 실행 PHP 서버](/docs/zero-code/php/distro/reference/long-running-server/)를
  참고한다.
