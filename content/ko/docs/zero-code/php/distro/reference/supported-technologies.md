---
title: 지원 기술
description: >-
  오픈텔레메트리(OpenTelemetry) PHP Distro가 지원하는 PHP 버전, SAPI, 운영체제,
  프레임워크, 라이브러리다.
weight: 2
cSpell:ignore: apk httplug musl mysqli psr
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리(OpenTelemetry) PHP Distro는 오픈텔레메트리 PHP의
배포판(distribution)이다. 오픈텔레메트리 호환성을 상속하고, 네이티브 구성 요소로
런타임 기능을 확장한다.

## 자동 계측 범위 {#auto-instrumentation-scope}

자동 계측은 지원되는 프레임워크와 라이브러리의 텔레메트리를 수집하지만, 다음은
계측하지 않는다.

- 독점 또는 커스텀 프레임워크 내부
- 계측 훅이 없는 비공개 소스 구성 요소
- 애플리케이션 고유의 비즈니스 로직

지원되지 않는 영역에는 수동 오픈텔레메트리 계측을 사용한다.

## PHP 버전 {#php-versions}

지원되는 PHP 버전: `8.1`부터 `8.5`까지.

## 지원되는 SAPI {#supported-sapis}

- `php-cli`
- `php-fpm`
- `php-cgi`/`fcgi`
- `mod_php`(prefork)

## 지원되는 운영체제 {#supported-operating-systems}

- Linux
  - 아키텍처: `x86_64`, `arm64`
  - glibc 기반 시스템: `deb`, `rpm`
  - musl 기반 시스템(Alpine): `apk`

## 계측되는 프레임워크 {#instrumented-frameworks}

- Laravel `6.x`부터 `13.x`까지
- Slim `4.x`

## 계측되는 라이브러리 {#instrumented-libraries}

- cURL
- HTTP 비동기(`php-http/httplug`)
- MySQLi
- PDO
- PostgreSQL
- PSR-18 HTTP 클라이언트(`psr/http-client`)

## 포함된 자동 계측 패키지 {#included-auto-instrumentation-packages}

| 이름                | 최초 포함 배포판 버전 | 패키지                                                                                                                      |
| ------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `curl`              | 1.0                   | [open-telemetry/opentelemetry-auto-curl](https://packagist.org/packages/open-telemetry/opentelemetry-auto-curl)             |
| `http-async-client` | 1.0                   | [open-telemetry/opentelemetry-auto-http-async](https://packagist.org/packages/open-telemetry/opentelemetry-auto-http-async) |
| `laravel`           | 1.0                   | [open-telemetry/opentelemetry-auto-laravel](https://packagist.org/packages/open-telemetry/opentelemetry-auto-laravel)       |
| `mysqli`            | 1.0                   | [open-telemetry/opentelemetry-auto-mysqli](https://packagist.org/packages/open-telemetry/opentelemetry-auto-mysqli)         |
| `pdo`               | 1.0                   | [open-telemetry/opentelemetry-auto-pdo](https://packagist.org/packages/open-telemetry/opentelemetry-auto-pdo)               |
| `postgresql`        | 1.2                   | [open-telemetry/opentelemetry-auto-postgresql](https://packagist.org/packages/open-telemetry/opentelemetry-auto-postgresql) |
| `psr18`             | 0.5                   | [open-telemetry/opentelemetry-auto-psr18](https://packagist.org/packages/open-telemetry/opentelemetry-auto-psr18)           |
| `slim`              | 1.0                   | [open-telemetry/opentelemetry-auto-slim](https://packagist.org/packages/open-telemetry/opentelemetry-auto-slim)             |

## 포함된 메트릭 패키지 {#included-metrics-packages}

| 최초 포함 배포판 버전 | 패키지                                                                                                                      | 방출되는 메트릭                           |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 0.6.0                 | [open-telemetry/opentelemetry-metrics-runtime](https://packagist.org/packages/open-telemetry/opentelemetry-metrics-runtime) | PHP 메모리 사용량, GC 사이클, 최대 메모리 |

## 추가 런타임 기능 {#additional-runtime-features}

- 자동 루트 스팬 생성
- 루트 스팬 URL 그룹화
- 추론된 스팬
- [어트리뷰트 기반 계측](/docs/zero-code/php/distro/reference/attribute-instrumentation/)(`#[WithSpan]`,
  `#[SpanAttribute]`)
- 백그라운드 텔레메트리 전송
- PHP 런타임 메트릭(메모리, GC — 네이티브 비동기 트랜스포트를 통해 자동으로
  내보내짐)

백그라운드 전송(논블로킹 내보내기)은 OTLP `http/protobuf`(기본값)에서 동작한다.
익스포터나 프로토콜을 지원되지 않는 트랜스포트(예: gRPC)로 변경하면 내보내기가
동기적으로 이루어진다.
