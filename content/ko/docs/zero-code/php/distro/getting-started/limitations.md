---
title: 제한 사항
description:
  오픈텔레메트리(OpenTelemetry) PHP Distro의 알려진 제한 사항과 제약이다.
weight: 2
# prettier-ignore
cSpell:ignore: basedir ComponentProvider opentelemetry-php-contrib passenv xdebug
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

이 페이지는 오픈텔레메트리(OpenTelemetry) PHP Distro의 알려진 제한 사항과 제약을
설명한다.

## 다른 PHP 텔레메트리 에이전트와 함께 실행 {#running-with-another-php-telemetry-agent}

오픈텔레메트리 PHP Distro를 다른 PHP APM이나 오픈텔레메트리 에이전트와 같은
프로세스에서 함께 실행하지 않는다. 둘 다 실행하면 충돌, 중복 계측, 불안정한
동작이 발생할 수 있다.

## `open_basedir` {#open_basedir}

`php.ini`에서 `open_basedir`가 활성화된 경우, 배포판(distro) 설치 경로가 허용
경로에 포함되어야 한다. 그렇지 않으면 에이전트가 로드되지 않을 수 있다.

## `xdebug` {#xdebug}

`xdebug`와 함께 실행하는 것은 프로덕션에서 권장되지 않으며, 계측된 프로세스에서
안정성 또는 메모리 문제를 일으킬 수 있다.

## 파일 기반 구성(`OTEL_CONFIG_FILE`) {#file-based-configuration-otel_config_file}

파일 기반(선언형) 구성을 사용할 때는 다음에 유의한다.

- 원격 구성(OpAMP)을 사용할 수 없다. 파일 기반 구성과 원격 구성은 상호
  배타적이다.
- `Registry::registerResourceDetector()`를 통해 등록된 리소스 디텍터(예:
  `opentelemetry-php-contrib`의 클라우드 제공자 디텍터)는 자동으로 활성화되지
  않는다. 이러한 디텍터는 `ComponentProvider`를 제공하고 YAML
  `resource.detection/development.detectors` 섹션에 명시적으로 나열되어야 한다.
- 배포판은 `telemetry.distro.name` 및 `telemetry.distro.version` 속성을 위한
  내장 `distro` 디텍터를 제공한다. 사용법은
  [구성](/docs/zero-code/php/distro/reference/configuration/#distro-resource-detector)을
  참고한다.
- YAML 파일에서 환경 변수 치환(`${VAR_NAME}`)은 값을 읽기 위해 `$_SERVER`에
  의존한다. 웹 서버 환경(Apache, nginx+FPM)에서는 프로세스 환경 변수가
  `$_SERVER`에 자동으로 제공되지 않는다. YAML 구성에서 `${VAR_NAME}` 치환을
  사용하려면, 변수가 PHP에 노출되도록 한다.
  - **Apache(mod_php)**: 가상 호스트 구성에서 `PassEnv VAR_NAME` 또는
    `SetEnv VAR_NAME value`를 사용한다.
  - **PHP-FPM**: FPM 풀 구성에서 `env[VAR_NAME] = value`를 설정하거나, 모든
    프로세스 환경 변수를 전달하려면 `clear_env = no`를 설정한다.
  - 또는 `${VAR_NAME}` 치환을 사용하는 대신 YAML 파일에 값을 직접 하드코딩한다.
