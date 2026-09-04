---
title: 제로 코드
description: >-
  코드를 작성하지 않고도 애플리케이션에 옵저버빌리티(observability)를 추가하는
  방법을 알아본다.
weight: 10
aliases: [automatic]
default_lang_commit: d1ef521ee4a777881fb99c3ec2b506e068cdec4c
---

[운영(ops)](/docs/getting-started/ops/) 담당자로서, 소스를 편집하지 않고도 하나
이상의 애플리케이션에 옵저버빌리티를 추가하고 싶을 수 있다.
오픈텔레메트리(OpenTelemetry)를 사용하면
[코드 기반 계측](/docs/concepts/instrumentation/code-based)을 위한
오픈텔레메트리 API 및 SDK를 사용하지 않고도 서비스에 대한 옵저버빌리티를 빠르게
확보할 수 있다.

![제로 코드](./zero-code.svg)

제로 코드 계측(Zero-code instrumentation)은 일반적으로 에이전트(agent) 또는
에이전트와 유사한 설치 방식으로 오픈텔레메트리 API 및 SDK 기능을 애플리케이션에
추가한다. 관련된 구체적인 메커니즘은 언어에 따라 다를 수 있으며, 바이트코드
조작(bytecode manipulation), 몽키 패칭(monkey patching), eBPF를 통해
오픈텔레메트리 API 및 SDK에 대한 호출을 애플리케이션에 주입한다.

일반적으로 제로 코드 계측은 사용 중인 라이브러리에 대한 계측을 추가한다. 즉
요청과 응답, 데이터베이스 호출, 메시지 큐(message queue) 호출 등이 계측 대상이
된다는 의미이다. 하지만 애플리케이션의 코드 자체는 일반적으로 계측되지 않는다.
코드를 계측하려면 [코드 기반 계측](/docs/concepts/instrumentation/code-based)을
사용해야 한다.

또한 제로 코드 계측을 사용하면 로드되는
[계측 라이브러리](/docs/concepts/instrumentation/libraries)와
[익스포터](/docs/concepts/components/#exporters)를 구성할 수 있다.

제로 코드 계측은 환경 변수와, 시스템 속성(system property)이나 초기화 메서드에
전달되는 인자 등의 다른 언어별 메커니즘을 통해 구성할 수 있다. 시작하려면 원하는
옵저버빌리티 백엔드에서 서비스를 식별할 수 있도록 서비스 이름만 구성하면 된다.

다음과 같은 다른 구성 옵션도 사용할 수 있다.

- 데이터 소스별 구성
- 익스포터 구성
- 전파자 구성
- 리소스 구성

다음 언어들에 대해 자동 계측을 사용할 수 있다.

- [.NET](/docs/zero-code/dotnet/)
- [Go](/docs/zero-code/go)
- [Java](/docs/zero-code/java/)
- [JavaScript](/docs/zero-code/js/)
- [PHP](/docs/zero-code/php/)
- [Python](/docs/zero-code/python/)
