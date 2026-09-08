---
title: 오픈텔레메트리 컬렉터와 쿠버네티스(Kubernetes)
linkTitle: 컬렉터
default_lang_commit: fe623719bc24346e9dcd77e9769026cf1c720cc5
---

[오픈텔레메트리(OpenTelemetry) 컬렉터](/docs/collector/)는 텔레메트리 데이터를
수신, 처리, 내보내기 위한 벤더 중립적인(vendor-agnostic) 방법이다. 컬렉터는 여러
곳에서 사용할 수 있지만, 이 문서에서는 쿠버네티스와 쿠버네티스에서 실행되는
서비스를 모니터링하기 위해 컬렉터를 사용하는 방법에 초점을 맞춘다. 설정 및 문제
해결과 같은 더 일반적인 컬렉터 문서는 [컬렉터](/docs/collector/)를 참고한다.
