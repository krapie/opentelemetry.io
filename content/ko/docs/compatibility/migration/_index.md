---
title: 마이그레이션
description: 오픈텔레메트리(OpenTelemetry)로 마이그레이션하는 방법
aliases: [/docs/migration/]
weight: 300
default_lang_commit: c99573bcaeda128c288d5ce8cf271913007c8866
---

## OpenTracing과 OpenCensus {#opentracing-and-opencensus}

오픈텔레메트리(OpenTelemetry)는 OpenTracing과 OpenCensus를 통합해 만들어졌다.
오픈텔레메트리는 처음부터 [OpenTracing과 OpenCensus 둘 다의 차기 주요
버전][to be the next major version of both OpenTracing and OpenCensus]으로
여겨졌다. 그렇기 때문에 오픈텔레메트리 프로젝트의 [핵심 목표][key goals] 중
하나는 두 프로젝트와의 하위 호환성을 제공하고, 기존 사용자를 위한 마이그레이션
경로를 제시하는 것이다.

이 두 프로젝트 중 하나를 사용하고 있었다면 [OpenTracing](opentracing/)과
[OpenCensus](opencensus/) 모두에 대한 마이그레이션 가이드를 따라갈 수 있다.

## Jaeger 클라이언트 {#jaeger-client}

[Jaeger 커뮤니티](https://www.jaegertracing.io/)는 자사의 클라이언트
라이브러리를 지원 중단(deprecated)하고 오픈텔레메트리 API, SDK, 계측
라이브러리로
[마이그레이션할 것](https://www.jaegertracing.io/docs/latest/migration/)을
권장한다.

Jaeger 백엔드는 v1.35부터 오픈텔레메트리 프로토콜(OpenTelemetry Protocol,
OTLP)을 사용해 트레이스 데이터를 수신할 수 있다. 오픈텔레메트리 SDK와
컬렉터(Collector)를 Jaeger 익스포터(Exporter)에서 OTLP 익스포터로 마이그레이션할
수 있다.

[to be the next major version of both OpenTracing and OpenCensus]:
  https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/
[key goals]:
  https://medium.com/opentracing/merging-opentracing-and-opencensus-f0fe9c7ca6f0
