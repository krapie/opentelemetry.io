---
title: PHP 제로 코드 계측
linkTitle: PHP
weight: 30
cSpell:ignore: PECL
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리(OpenTelemetry)는 PHP를 위한 두 가지 제로 코드 계측(Zero-code
instrumentation) 방식을 제공한다.

|                 | [자동 계측](auto/)             | [PHP Distro](distro/)               |
| --------------- | ------------------------------ | ----------------------------------- |
| **설치**        | Composer + PECL 확장           | OS 패키지(`deb`, `rpm`, `apk`)      |
| **플랫폼**      | Linux, macOS, Windows          | Linux만 해당                        |
| **설정**        | Composer 오토로딩(autoloading) | 패키지 설치 후 PHP 재시작           |
| **제어**        | 완전한 수동 제어               | 고정된 기본값(opinionated defaults) |
| **적합한 경우** | 유연한 환경, 커스텀 구성       | 프로덕션 Linux 배포 환경            |

## 자동 계측 선택하기 {#choose-auto-instrumentation}

다음과 같은 경우 [PHP 제로 코드 자동 계측](auto/)을 사용한다.

- 이미 Composer를 사용 중인 경우
- macOS 또는 Windows에서 실행해야 하는 경우
- 계측 및 구성에 대한 최대한의 제어가 필요한 경우

## PHP Distro 선택하기 {#choose-php-distro}

다음과 같은 경우 [오픈텔레메트리 PHP Distro](distro/)를 사용한다.

- Linux에 배포하며 패키지로 관리되는 설치(`deb`, `rpm`, `apk`)를 원하는 경우
- 애플리케이션 코드나 Composer를 건드리지 않고 제로 코드로 온보딩해야 하는 경우
- 프로덕션에 맞춰 조정된 기본값(백그라운드 내보내기, 추론된 스팬, OpAMP)을
  원하는 경우
