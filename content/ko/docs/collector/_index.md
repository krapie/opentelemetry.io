---
title: 컬렉터
description:
  텔레메트리 데이터를 수신, 처리, 내보내기 위한 벤더 중립적인 방법이다.
aliases: [collector/about]
sidebar_root_for: children
cascade:
  vers: 0.159.0
weight: 270
default_lang_commit: de42414bea5449f67be2211e96836fbc70660548
---

![Jaeger, OTLP, Prometheus 통합을 보여주는 오픈텔레메트리 컬렉터 다이어그램](img/otel-collector.svg)

## 소개 {#introduction}

오픈텔레메트리 컬렉터는 텔레메트리 데이터를 수신, 처리, 내보내는 방법을 벤더
중립적으로(vendor-agnostic) 구현한 것이다. 여러 에이전트/컬렉터를 실행하고,
운영하고, 유지 관리해야 하는 부담을 없애준다. 향상된 확장성(scalability)을
지원하며, 오픈 소스 옵저버빌리티 데이터 포맷(예: Jaeger, Prometheus, Fluent Bit
등)을 하나 이상의 오픈 소스 또는 상용 백엔드로 전송하는 것도 지원한다.

## 목표 {#objectives}

- _사용성(Usability)_: 합리적인 기본 설정을 제공하고, 널리 쓰이는 프로토콜을
  지원하며, 별도 설정 없이도 실행되어 데이터를 수집한다.
- _성능(Performance)_: 다양한 부하와 설정에서도 안정적이고 성능이 뛰어나다.
- _옵저버빌리티(Observability)_: 관찰 가능한 서비스의 본보기(exemplar)이다.
- _확장성(Extensibility)_: 핵심 코드를 건드리지 않고도 커스터마이징할 수 있다.
- _통합성(Unification)_: 단일 코드베이스로, 트레이스·메트릭·로그를 지원하는
  에이전트 또는 컬렉터로 배포할 수 있다.

## 컬렉터를 사용해야 할 때 {#when-to-use-a-collector}

대부분의 언어별 계측 라이브러리에는 널리 쓰이는 백엔드와 OTLP를 위한 익스포터가
있다. 이럴 때 다음과 같은 의문이 들 수 있다.

> 각 서비스가 백엔드로 직접 데이터를 전송하는 대신, 어떤 상황에서 컬렉터를
> 사용해 데이터를 전송해야 할까?

오픈텔레메트리(OpenTelemetry)를 처음 시도하고 시작하는 단계에서는, 데이터를
백엔드로 직접 전송하는 것이 빠르게 가치를 확인할 수 있는 좋은 방법이다. 또한
개발 환경이나 소규모 환경에서는 컬렉터 없이도 꽤 괜찮은 결과를 얻을 수 있다.

하지만 일반적으로는 서비스와 함께 컬렉터를 사용하는 것을 권장한다. 컬렉터를
사용하면 서비스가 데이터를 빠르게 넘길 수 있고, 재시도, 배치(batching), 암호화,
심지어 민감한 데이터 필터링과 같은 추가 처리를 컬렉터가 대신 처리할 수 있기
때문이다.

[컬렉터를 설정](quick-start)하는 것도 생각보다 쉽다. 각 언어의 기본 OTLP
익스포터는 로컬 컬렉터 엔드포인트를 가정하므로, 컬렉터를 실행하기만 하면
자동으로 텔레메트리 수신이 시작된다.

## 컬렉터 보안 {#collector-security}

컬렉터가 안전하게 [호스팅][hosted]되고 [구성][configured]되도록 모범 사례(best
practice)를 따른다.

## 상태 {#status}

**컬렉터**의 상태는 [혼합(mixed)][mixed]이다. 핵심 컬렉터 구성 요소가 현재
[안정성 수준][stability levels]이 혼재되어 있기 때문이다.

**컬렉터 구성 요소**는 성숙도(maturity) 수준이 저마다 다르다. 각 구성 요소의
안정성은 해당 구성 요소의 `README.md`에 문서화되어 있다. 사용 가능한 모든 컬렉터
구성 요소 목록은 [레지스트리][registry]에서 확인할 수 있다.

컬렉터 소프트웨어 아티팩트(artifact)는 그 대상 사용자층에 따라 일정 기간 동안
지원이 보장된다. 이 지원에는 최소한 심각한 버그와 보안 문제에 대한 수정이
포함된다. 자세한 내용은
[지원 정책](https://github.com/open-telemetry/opentelemetry-collector/blob/main/VERSIONING.md)을
참고한다.

## 배포판 및 릴리스 {#releases}

컬렉터 배포판 및 릴리스에 대한 정보([최신 릴리스][latest release] 포함)는
[배포판](distributions/)을 참고한다.

[configured]: /docs/security/config-best-practices/
[hosted]: /docs/security/hosting-best-practices/
[latest release]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases/latest
[mixed]: /docs/specs/otel/document-status/#mixed
[registry]: /ecosystem/registry/?language=collector
[stability levels]:
  https://github.com/open-telemetry/opentelemetry-collector#stability-levels
