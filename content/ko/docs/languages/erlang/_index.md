---
title: Erlang/Elixir
weight: 130
description: >
  <img width="35" class="img-initial otel-icon"
  src="/img/logos/32x32/Erlang_SDK.svg" alt="Erlang/Elixir"> Erlang/Elixir에서의
  오픈텔레메트리(OpenTelemetry) 언어별 구현체이다.
cascade:
  versions:
    otelSdk: 1.5
    otelApi: 1.4
    otelSemconv: 1.27
    otelApiExperimental: 0.6
    otelSdkExperimental: 0.6
    otelExporter: 1.8
    otelPhoenix: 2.0
    otelCowboy: 1.0
    otelEcto: 1.2
default_lang_commit: d8f5ed285d009cc6baac6d7141bfde8d0956a756
cSpell:ignore: ecto
---

{{% docs/languages/index-intro erlang %}}

API, SDK, OTLP 익스포터 패키지는 [`hex.pm`](https://hex.pm)에
[`opentelemetry_api`](https://hex.pm/packages/opentelemetry_api),
[`opentelemetry`](https://hex.pm/packages/opentelemetry),
[`opentelemetry_exporter`](https://hex.pm/packages/opentelemetry_exporter)로
게시된다.

## 버전 지원 {#version-support}

OpenTelemetry Erlang은 Erlang 23 이상, Elixir 1.13 이상을 지원한다.

## 저장소 {#repositories}

- [opentelemetry-erlang](https://github.com/open-telemetry/opentelemetry-erlang):
  API, SDK, OTLP 익스포터를 포함하는 메인 저장소
- [opentelemetry-erlang-contrib](https://github.com/open-telemetry/opentelemetry-erlang-contrib):
  [Phoenix](https://www.phoenixframework.org/)나
  [Ecto](https://hexdocs.pm/ecto/Ecto.html)와 같은 Erlang/Elixir 프로젝트를 위한
  유용한 라이브러리와 계측 라이브러리

{{% /docs/languages/index-intro %}}
