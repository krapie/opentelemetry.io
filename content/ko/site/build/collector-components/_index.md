---
title: 컬렉터(Collector) 컴포넌트 자동화
linkTitle: 컬렉터 컴포넌트 자동화
description: >-
  오픈텔레메트리(OpenTelemetry) 컬렉터 컴포넌트의 자동화 프로세스에 대한
  설명이다.
weight: 50
default_lang_commit: acd9ecb678353aa5fb0c9ed508dd33131681dcbe
---

오픈텔레메트리 컬렉터 컴포넌트 페이지 내의 표는
[OpenTelemetry Ecosystem Explorer registry](https://github.com/open-telemetry/opentelemetry-ecosystem-explorer/tree/main/ecosystem-registry/collector)의
데이터와 자동으로 동기화된다. 이 프로세스를 관리하는 코드는
[`scripts/collector-sync`][]에 있다.

동기화 프로세스는 예약된 일정에 따라 실행되는 GitHub Action이 관리한다
([`collector-sync.yml`][]).

매일 밤 이 GitHub Action은 다음 단계를 수행한다:

1. OpenTelemetry Ecosystem Explorer registry에서 최신 데이터를 가져온다.
2. 레지스트리 데이터를 기반으로 [`data/collector/`][]에 있는 관련 컴포넌트
   데이터 파일을 업데이트한다.
3. 컴포넌트 데이터 파일에 변경 사항이 있으면 해당 업데이트를 반영한 PR을
   생성한다.

모든 컴포넌트 페이지는 [`data/collector/`][] 디렉터리에서 관련 데이터를 가져오는
숏코드(shortcode)를 참조하므로, 데이터 파일이 업데이트되면 컴포넌트 페이지의
표도 자동으로 최신 정보를 반영한다.

관련 파일 및 디렉터리:

- [`data/collector/`][]: 컴포넌트 데이터 파일이 저장되는 디렉터리로, 컴포넌트
  페이지의 표를 채우는 데 사용된다.
- [`scripts/collector-sync`][]: 레지스트리 데이터를 가져오고 컴포넌트 데이터
  파일을 업데이트하는 코드가 담긴 디렉터리이다.
- [`.github/workflows/collector-sync.yml`][`collector-sync.yml`]: 동기화
  프로세스를 예약하고 실행하는 GitHub Action 워크플로이다.
- [`layouts/_shortcodes/collector-component-rows.html`][]: 데이터 파일로부터
  완전한 HTML 표를 렌더링한다.
- [`layouts/_shortcodes/component-link.html`][]: 컴포넌트 표에서 사용되는,
  컴포넌트 소스 코드 저장소로의 링크를 렌더링한다.
- [`i18n/<language>.yml`][]: `collector_component_` 접두사가 붙은, 컴포넌트 표
  페이지용 번역이 담겨 있다(숏코드에서 참조된다).

## 번역 {#translations}

컬렉터 컴포넌트 페이지의 새 번역을 만들려면 다음 단계를 따를 수 있다:

- 기존 영문 콘텐츠를 `content/en/docs/collector/components`에서 새 언어에
  해당하는 디렉터리로 복사한다(예: 스페인어의 경우
  `content/es/docs/collector/components`).
- 새 언어로 정적 콘텐츠(제목, 설명 등)를 번역한다.
- 연관된 [`i18n/<language>.yml`][] 파일이 존재하고, 컴포넌트 표에서 사용되는
  `collector_components_` 접두사가 붙은 키의 항목을 포함하는지 확인한다. 영문
  항목을 복사한 뒤 값을 번역하면 된다.

[`scripts/collector-sync`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/scripts/collector-sync.sh
[`collector-sync.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/workflows/collector-sync.yml
[`data/collector/`]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/data/collector
[`layouts/_shortcodes/collector-component-rows.html`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/layouts/_shortcodes/collector-component-rows.html
[`layouts/_shortcodes/component-link.html`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/layouts/_shortcodes/component-link.html
[`i18n/<language>.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/i18n
