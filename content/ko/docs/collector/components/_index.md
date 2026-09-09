---
title: 구성 요소
description:
  오픈텔레메트리(OpenTelemetry) 컬렉터 구성 요소 - 리시버, 프로세서, 익스포터,
  커넥터, 익스텐션
weight: 22
default_lang_commit: 1c2b0563e8e66ef0952c442e3662e4bec18a8762
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 텔레메트리 데이터를 처리하는 구성 요소로
이루어져 있다. 각 구성 요소는 데이터 파이프라인에서 특정 역할을 담당한다.

## 구성 요소 유형 {#component-types}

- **[리시버(Receiver)](receiver/)** - 다양한 소스와 포맷으로부터 텔레메트리
  데이터를 수집한다.
- **[프로세서(Processor)](processor/)** - 텔레메트리 데이터를 변환하고,
  필터링하고, 보강한다.
- **[익스포터(Exporter)](exporter/)** - 텔레메트리 데이터를 옵저버빌리티
  백엔드로 전송한다.
- **[커넥터(Connector)](connector/)** - 두 파이프라인을 연결하며, 익스포터와
  리시버 역할을 동시에 수행한다.
- **[익스텐션(Extension)](extension/)** - 헬스 체크(health check)와 같은 추가
  기능을 제공한다.
