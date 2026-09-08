---
title: 오픈텔레메트리 PHP Distro
linkTitle: PHP Distro
description: >-
  네이티브 Linux 패키지로 제공되는, 프로덕션에 바로 사용할 수 있는 PHP용
  오픈텔레메트리(OpenTelemetry) 제로 코드 계측이다.
weight: 30
aliases: [/docs/zero-code/php-distro/]
cSpell:ignore: apk rpm
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리 PHP Distro는 오픈텔레메트리(OpenTelemetry)로 PHP 애플리케이션을
계측하기 위한 프로덕션 중심의 배포판(distribution)이다.

많은 PHP 환경은 Composer만 사용하는 워크플로로는 계측하기 어렵다(잠긴 호스트,
제한된 빌드 도구, 엄격한 배포 파이프라인 등). 오픈텔레메트리 PHP Distro는 이러한
프로덕션 환경의 현실에 초점을 맞춘다.

- OS 패키지(`deb`, `rpm`, `apk`)를 통해 설치
- PHP 프로세스 재시작
- 텔레메트리 전송 시작

일반적인 설정에서는 애플리케이션 코드 변경이 필요하지 않다.

## 포함된 구성 요소 {#what-is-included}

이 배포판(distro)은 다음을 결합한다.

- 네이티브 PHP 확장(extension) 및 로더(`.so` 아티팩트)
- PHP 런타임/부트스트랩(bootstrap) 로직
- 인기 라이브러리 및 프레임워크를 위한 자동 계측 의존성
- Linux 배포판(distribution)을 위한 패키징 스크립트

## 주요 기능 {#key-features}

- `deb`, `rpm`, `apk` 워크플로를 위한 네이티브 OS 패키지
- 설치 후 자동 부트스트랩 및 자동 계측
- 백그라운드 텔레메트리 전송(논블로킹)
- 추론된 스팬 및 자동 루트 스팬 생성
- 트랜잭션 루트 스팬을 위한 URL 그룹화
- 네이티브 OTLP protobuf 직렬화(별도의 `ext-protobuf` 요구 사항 없음)
- PHP `8.1`부터 `8.4`까지 지원

## 다른 OTel PHP 프로젝트와의 관계 {#relationship-to-other-otel-php-projects}

오픈텔레메트리 PHP Distro는 `opentelemetry-php` 및
`opentelemetry-php-instrumentation`을 보완한다.

- 패키지로 관리되는, 프로덕션 우선의 제로 코드 온보딩을 원한다면
  배포판(distro)을 선택한다.
- 최대한의 수동 제어나 플랫폼 유연성이 필요하다면 Composer 중심 계측을 선택한다.

## 빠른 시작 {#quick-start}

1. 사용 중인 플랫폼에 맞는 배포판(distro) 패키지를 설치한다(`deb`, `rpm`, 또는
   `apk`).
2. `OTEL_EXPORTER_OTLP_ENDPOINT` 및 `OTEL_EXPORTER_OTLP_HEADERS`를 설정한다.
3. PHP 프로세스를 재시작하고 백엔드에서 트레이스를 확인한다.

전체 안내는 [설정 가이드](getting-started/setup/)를 참고한다.
