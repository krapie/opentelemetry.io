---
title: 익스텐션 빌드하기
linkTitle: 익스텐션
description: 컬렉터를 위한 커스텀 익스텐션을 빌드하는 방법에 대한 안내이다.
weight: 300
default_lang_commit: 6a7f17450ce3edc2e4363013551ee93ba7934a5d
---

오픈텔레메트리(OpenTelemetry) 컬렉터의 익스텐션은 핵심 파이프라인(리시버,
프로세서, 익스포터)에 속하지 않는 기능을 제공하기 위해 컬렉터에 추가할 수 있는
구성 요소이다. 익스텐션은 인증(authentication), 헬스 체크(health check), 서비스
디스커버리(service discovery) 등과 같은 기능을 제공할 수 있다.
