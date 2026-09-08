---
title: JavaScript
description: >-
  <img width="35" class="img-initial otel-icon"
  src="/img/logos/32x32/JS_SDK.svg" alt="JavaScript"> JavaScript(Node.js 및
  브라우저용)에서의 오픈텔레메트리(OpenTelemetry) 언어별 구현체이다.
aliases: [/js/metrics, /js/tracing, nodejs]
redirects:
  - { from: /js/*, to: ':splat' }
  - { from: /docs/js/*, to: ':splat' }
weight: 160
default_lang_commit: 368f811f81c27798a031b4c92024ecdd65cddc19
---

{{% docs/languages/index-intro js /%}}

{{% include browser-instrumentation-warning.md %}}

## 버전 지원 {#version-support}

OpenTelemetry JavaScript는 Node.js의 모든 활성 또는 유지 관리 LTS 버전을
지원한다. 이전 버전의 Node.js도 동작할 수는 있지만, 오픈텔레메트리에서 테스트를
거치지 않는다.

OpenTelemetry JavaScript는 공식적으로 지원하는 브라우저 목록이 없다. 현재
지원되는 주요 브라우저 버전에서 동작하는 것을 목표로 한다.

OpenTelemetry JavaScript는 TypeScript에 대해 DefinitelyTyped의 지원 정책을
따르며, 이는 2년의 지원 기간을 설정한다. 2년이 지난 TypeScript 버전에 대한
지원은 OpenTelemetry JavaScript의 마이너 릴리스에서 중단된다.

런타임 지원에 대한 자세한 내용은
[이 개요](https://github.com/open-telemetry/opentelemetry-js#supported-runtimes)를
참고한다.

## 저장소 {#repositories}

OpenTelemetry JavaScript는 다음 저장소로 구성된다.

- [opentelemetry-js](https://github.com/open-telemetry/opentelemetry-js): API 및
  SDK의 핵심 배포판을 포함하는 핵심 저장소
- [opentelemetry-js-contrib](https://github.com/open-telemetry/opentelemetry-js-contrib):
  API 및 SDK의 핵심 배포판에 포함되지 않은 기여 코드

## 도움말 및 피드백 {#help-or-feedback}

OpenTelemetry JavaScript에 대해 질문이 있다면,
[GitHub Discussions](https://github.com/open-telemetry/opentelemetry-js/discussions)나
[CNCF Slack](https://slack.cncf.io/)의 `#otel-js` 채널을 통해 문의한다.

OpenTelemetry JavaScript에 기여하고 싶다면,
[기여 안내](https://github.com/open-telemetry/opentelemetry-js/blob/main/CONTRIBUTING.md)를
참고한다.
