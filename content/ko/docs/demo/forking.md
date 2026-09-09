---
title: 데모 저장소 포크하기
linkTitle: 포크
default_lang_commit: 5472965d7714ed898b008d41fa97561591320196
---

[데모 저장소][demo repository]는 포크(fork)하여 오픈텔레메트리(OpenTelemetry)로
무엇을 하고 있는지 보여주는 도구로 사용할 수 있도록 설계되었다.

포크나 데모를 설정하려면 일반적으로 몇 가지 환경 변수를 재정의하고, 필요하다면
일부 컨테이너 이미지를 교체하기만 하면 된다.

라이브 데모는 데모
[README](https://github.com/open-telemetry/opentelemetry-demo/blob/main/README.md?plain=1)에
추가할 수 있다.

## 포크 관리자를 위한 제안 {#suggestions-for-fork-maintainers}

- 데모에서 내보내거나(emit) 수집(collect)하는 텔레메트리 데이터를 개선하고
  싶다면, 변경 사항을 이 저장소에 백포트(backport)할 것을 강력히 권장한다. 벤더
  또는 구현별 변경 사항의 경우, 기본 코드를 변경하기보다는 구성(config)을 통해
  파이프라인에서 텔레메트리를 수정하는 전략이 더 바람직하다.
- 교체하기보다는 확장한다. 기존 API와 인터페이스하는 완전히 새로운 서비스를
  추가하는 것은, 텔레메트리 수정만으로는 구현할 수 없는 벤더별 또는 도구별
  기능을 추가하는 좋은 방법이다.
- 확장성(extensibility)을 지원하려면 큐, 데이터베이스, 캐시 등의 리소스에
  리포지토리(repository) 또는 퍼사드(facade) 패턴을 사용한다. 이렇게 하면
  플랫폼에 따라 이러한 서비스의 서로 다른 구현체를 끼워 넣을(shim) 수 있다.
- 벤더 또는 도구별 개선 사항을 이 저장소에 백포트하려고 시도하지 않는다.

궁금한 점이 있거나 포크 관리자로서 더 편하게 작업할 수 있는 방법을 제안하고
싶다면 이슈(issue)를 등록한다.

[demo repository]: <{{% param repo %}}>
