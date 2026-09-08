---
title: 컬렉터 배포하기
linkTitle: 배포
description:
  오픈텔레메트리(OpenTelemetry) 컬렉터를 배포할 때 적용할 수 있는 패턴이다.
aliases: [/docs/collector/deployment]
weight: 3
default_lang_commit: 6cebc46de450dd44481a8a6f17c9b3d6f04aa0f2
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 단일 바이너리로 구성되며, 사용 사례에
따라 다양한 방식으로 배포할 수 있다. 이 섹션에서는 일반적인 배포 패턴과 각각의
사용 사례, 장단점을 설명한다. 또한 크로스 환경(cross-environment) 및 멀티 백엔드
시나리오에서 컬렉터를 설정하기 위한 모범 사례도 제공한다. 배포와 관련된 보안
고려 사항은 [컬렉터 호스팅 모범 사례][security]를 참고한다.

## 추가 자료 {#additional-resources}

- KubeCon NA 2021 발표: [오픈텔레메트리 컬렉터 배포 패턴][y-patterns]
  - 발표에서 다룬 [배포 패턴][gh-patterns]

[security]: /docs/security/hosting-best-practices/
[gh-patterns]:
  https://github.com/jpkrohling/opentelemetry-collector-deployment-patterns/
[y-patterns]: https://www.youtube.com/watch?v=WhRrwSHDBFs
