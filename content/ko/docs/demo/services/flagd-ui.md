---
title: Flagd-UI 서비스
linkTitle: Flagd-UI
aliases: [flagd-uiservice]
cSpell:ignore: uiservice
default_lang_commit: 753d9126150332a7949bbe32683a83b7d215d4b2
---

이 서비스는 사용자가 기능 플래그(feature flag)를 토글하고 편집하여 데모 환경의
동작을 변경할 수 있는 프론트엔드 역할을 한다.

[Flagd-UI 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/flagd-ui/)

## 트레이싱 초기화 {#initializing-tracing}

Phoenix 엔드포인트와 요청의 자동 계측에 필요한 의존성을 설치한 뒤,
[공식 문서](/docs/languages/erlang/getting-started/)에 따라 `config/runtime.exs`
파일을 편집하여 이를 구성한다.

```elixir
otel_endpoint =
  System.get_env("OTEL_EXPORTER_OTLP_ENDPOINT") ||
    raise """
    environment variable OTEL_EXPORTER_OTLP_ENDPOINT is missing.
    """

config :opentelemetry, :processors,
    otel_batch_processor: %{
      exporter: {:opentelemetry_exporter, %{endpoints: [otel_endpoint]}}
    }
```

그리고
[`lib/flagd_ui/application.ex`](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/flagd-ui/lib/flagd_ui/application.ex)
내부에서 오픈텔레메트리(OpenTelemetry) Bandit 어댑터와 Phoenix 라이브러리도 함께
초기화한다.

```elixir
OpentelemetryBandit.setup()
OpentelemetryPhoenix.setup(adapter: :bandit)
```

## 트레이스 {#traces}

Phoenix와 Bandit은 전용 라이브러리를 통해 자동으로 계측된다.

## 메트릭 {#metrics}

TBD

## 로그 {#logs}

TBD
