---
title: 시그널
description:
  오픈텔레메트리(OpenTelemetry)가 지원하는 텔레메트리 카테고리에 대해 알아본다.
aliases: [data-sources, otel-concepts]
weight: 11
default_lang_commit: 274bf95abd0cbad3ad9f95b4426f282466cdaade
---

오픈텔레메트리(OpenTelemetry)의 목적은 [시그널][signals]을 수집(collect),
처리(process), 내보내는(export) 것이다. 시그널은 플랫폼에서 실행되는 운영체제와
애플리케이션의 기저 활동(activity)을 나타내는 시스템 출력이다. 시그널은 온도나
메모리 사용량처럼 특정 시점에 측정하고자 하는 대상일 수도 있고, 추적하고자 하는
분산 시스템의 구성 요소를 거쳐 가는 이벤트일 수도 있다. 서로 다른 시그널을 함께
그룹화하면 동일한 기술 요소의 내부 동작을 여러 관점에서 관찰할 수 있다.

오픈텔레메트리는 현재 다음을 지원한다:

- [트레이스](traces)
- [메트릭](metrics)
- [로그](logs)
- [배기지](baggage)

다음은 아직 개발 중이거나 [제안][proposal] 단계에 있다:

- [이벤트][Events], [로그](logs)의 특정 유형
- [프로파일](profiles)

[Events]: /docs/specs/otel/logs/data-model/#events
[proposal]:
  https://github.com/open-telemetry/opentelemetry-specification/tree/main/oteps/#readme
[signals]: /docs/specs/otel/glossary/#signals
