---
title: 컬렉터 없음
description: 애플리케이션에서 백엔드로 직접 시그널을 전송한다
aliases: [/docs/collector/deployment/no-collector]
weight: 200
default_lang_commit: 6cebc46de450dd44481a8a6f17c9b3d6f04aa0f2
---

가장 직접적인 배포 패턴은 컬렉터를 전혀 사용하지 않는다. 이 방식에서는
오픈텔레메트리(OpenTelemetry) SDK로 [계측된][instrumentation] 애플리케이션이
텔레메트리 시그널(트레이스, 메트릭, 로그)을 백엔드로 곧바로 내보낸다(export).

![컬렉터 없음 배포 개념도](../../../img/otel-sdk.svg)

## 예제 {#example}

애플리케이션에서 백엔드로 직접 시그널을 내보내는 방법을 보여주는 엔드투엔드
예제는 [계측 문서][instrumentation]를 참고한다.

## 트레이드오프 {#trade-offs}

컬렉터를 건너뛸 때의 주요 장단점은 다음과 같다.

장점:

- 특히 개발 및 테스트 환경에서 사용하기 간단하다.
- 배포하거나 운영할 추가적인 구성 요소가 없다.

단점:

- 수집(collection), 처리(processing), 또는 유입(ingestion) 요구 사항이 변경되면
  코드 변경이 필요하다.
- 애플리케이션 코드와 백엔드 저장소 또는 시각화 사이에 강한 결합(coupling)이
  존재한다.
- 각 언어 구현체는 제한된 수의 익스포터만 지원한다.

[instrumentation]: /docs/languages/
