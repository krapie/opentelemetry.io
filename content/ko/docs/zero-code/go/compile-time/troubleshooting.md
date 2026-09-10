---
title: 문제 해결
description: Go 컴파일 타임 계측의 문제를 진단한다.
weight: 50
cSpell:ignore: otelc
default_lang_commit: f23a9af2244bae83dfc52bd677450fcf24ad80b3
---

## 디버그 로깅 활성화 {#enable-debug-logging}

빌드 중에 도구가 하는 일을 보려면 디버그 모드를 활성화한다.

```sh
otelc --debug go build -o myapp .
```

`debug.log` 파일을 포함한 디버그 출력은 도구의 작업 디렉터리(기본값은 모듈의
`.otelc-build` 디렉터리)에 작성된다. 이를 살펴보면 어떤 규칙이 매칭되었고 어떤
계측이 주입되었는지 알 수 있다.

## 텔레메트리가 생성되지 않음 {#no-telemetry-is-produced}

1. 바이너리가 일반 `go build`가 아니라 `otelc`를 통해 빌드되었는지 확인한다.
2. 애플리케이션이 실제로 [지원되는 라이브러리](../supported-libraries)를
   사용하는지, 그리고 의존하는 버전이 계측 규칙이 선언한 지원 범위 안에 있는지
   확인한다.
3. 익스포터(exporter) 구성을 확인한다. `OTEL_EXPORTER_OTLP_ENDPOINT`가 설정되지
   않았거나 잘못되어 있으면 텔레메트리가 갈 곳이 없다. `OTEL_LOG_LEVEL=debug`를
   설정하여 내보내기 오류를 드러낸다.
4. 계측이 `OTEL_GO_ENABLED_INSTRUMENTATIONS`나
   `OTEL_GO_DISABLED_INSTRUMENTATIONS`를 통해 비활성화되지 않았는지 확인한다.

## 빌드 아티팩트 정리 {#clean-up-build-artifacts}

빌드가 예상과 다르게 동작하면, 이전 setup 및 build 단계에서 생성된 아티팩트를
제거하고 깨끗한 상태에서 다시 빌드한다.

```sh
otelc cleanup
```

## 도움 받기 {#getting-help}

- 버그에 대해서는
  [GitHub 이슈](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/issues)
- 질문에 대해서는
  [GitHub 디스커션](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/discussions)
- CNCF Slack의
  [#otel-go-compt-instr-sig](https://cloud-native.slack.com/archives/C088D8GSSSF)
  채널
