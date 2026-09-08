---
title: 컬렉터 확장하기
linkTitle: 확장
description:
  커스텀 구성 요소로 오픈텔레메트리(OpenTelemetry) 컬렉터를 확장하는 방법을
  알아본다.
weight: 90
default_lang_commit: 6a7f17450ce3edc2e4363013551ee93ba7934a5d
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 확장 가능하도록(extensible) 설계되었다.
핵심 컬렉터에는 다양한 리시버, 프로세서, 익스포터가 기본으로 포함되어 있지만,
커스텀 프로토콜을 지원하거나, 특정 방식으로 데이터를 처리하거나,
독점(proprietary) 백엔드로 데이터를 전송해야 하는 경우도 있을 것이다.

이 섹션에서는
[오픈텔레메트리 컬렉터 빌더(OpenTelemetry Collector Builder, OCB)](./ocb/)를
사용해 컬렉터를 확장하고 커스텀 구성 요소를 만드는 방법을 안내한다.
