---
title: 리눅스 호스트에서의 오픈텔레메트리
linkTitle: 리눅스
description:
  오픈텔레메트리(OpenTelemetry)를 시스템 패키지로 설치하여 리눅스 호스트에서
  실행되는 애플리케이션을 자동으로 계측한다.
weight: 250
cSpell:ignore: metapackage
default_lang_commit: edb244ceebdcbbb33c640eaac8d218dbc480e4c0
---

오픈텔레메트리(OpenTelemetry)를 설정하는 방법은 대개 애플리케이션이 실행되는
위치에 따라 달라진다. [쿠버네티스(Kubernetes)](/docs/platforms/kubernetes/)처럼
오픈텔레메트리 오퍼레이터(Operator) 덕분에 상당히 자동화된 환경도 있고,
[서비스형 함수(Functions as a Service)](/docs/platforms/faas/)처럼
오픈텔레메트리 Lambda 레이어를 사용하는 환경도 있다. 하지만 많은 Java, .NET,
Node.js, Python 애플리케이션은 리눅스 호스트에서 직접 실행되며, 이런 환경에서는
전통적으로 에이전트를 직접 다운로드하고 환경 변수를 손수 설정해서 계측해야 했다.

[오픈텔레메트리 패키징 SIG](https://github.com/open-telemetry/opentelemetry-packaging)는
오픈텔레메트리를 호스트 자체의 의존성(dependency)으로 만들어주는 **시스템
패키지**를 제공한다. 단일 패키지를 설치하고 애플리케이션을 재시작하기만 하면,
호스트에서 실행 중인 Java, .NET, Node.js, Python 프로세스가 자동으로 계측되어
텔레메트리를 내보내기 시작한다.

## 동작 방식 {#how-it-works}

`opentelemetry` 패키지는 다음에 의존하는 메타패키지(metapackage)이다.

- [오픈텔레메트리 인젝터(Injector)](https://github.com/open-telemetry/opentelemetry-injector):
  프로세스가 시작될 때 지원되는 런타임이 일치하는 자동 계측을 로드하도록 동적
  링커(dynamic linker)를 설정한다.
- Java, .NET, Node.js, Python용 언어별 자동 계측 패키지.

설치가 완료되면 인젝터는 호스트에서 새로 시작되는 모든 동적 링크(dynamically
linked) 프로세스에 연결(attach)된다. 지원되는 런타임의 프로세스에만 눈에 보이는
효과가 있으며, 해당 프로세스에 일치하는 자동 계측을 로드한다. 오픈텔레메트리
SDK가 없는 런타임의 프로세스는 영향을 받지 않는다. 이미 실행 중이던
애플리케이션은 재시작된 후에 계측된다. 기본적으로 텔레메트리는 OTLP를 사용해
`localhost`의 `4317`(gRPC) 및 `4318`(HTTP) 포트로 내보내지므로, 이를 수신해서
전달할 로컬 [오픈텔레메트리 컬렉터](/docs/collector/)를 일반적으로 함께
실행한다. 컬렉터 자체는
[오픈텔레메트리 컬렉터 릴리스(Releases)](https://github.com/open-telemetry/opentelemetry-collector-releases)
프로젝트에서 시스템 패키지로 배포하며, 이를 이 시스템 패키지 저장소에 통합하는
작업은
[opentelemetry-collector-releases#1561](https://github.com/open-telemetry/opentelemetry-collector-releases/issues/1561)에서
추적하고 있다.

패키징 및 OBI SIG는 또한 [오픈텔레메트리 eBPF 계측](/docs/zero-code/obi/)을
시스템 패키지로 제공하여, 제로 코드 계측을 Go, Rust, C++ 등 추가 런타임으로
확장할 계획이다.

## 시작하기 {#get-started}

- [설치](installation/): Debian, Ubuntu, Fedora, RHEL 및 그 파생 배포판에
  저장소를 추가하고 패키지를 설치한다.
- [구성](configuration/): 인젝터가 컬렉터나 백엔드를 가리키도록 지정하고, 계측
  대상을 조정한다.

## 상태 및 제약 사항 {#status-and-limitations}

> [!WARNING]
>
> 이 시스템 패키지는 아직 초기 단계이며 **프로덕션 워크로드에 사용하도록
> 의도되지 않았다**. 패키징 방식이 성숙해짐에 따라 변경될 수 있다.
>
> 구체적으로는 다음과 같다.
>
> - APT 및 YUM 저장소는 현재 GitHub Pages에서 호스팅되고 있으며, 이는 **최종
>   위치가 아니다**.
> - 패키지는 **아직 서명되지 않았으므로**, 설치 안내에서는 서명 검증을
>   비활성화한다.
> - [오픈텔레메트리 컬렉터](/docs/collector/)는 아직 기본 메타패키지에 포함되어
>   있지 않으므로, 현재는 별도로 설치하고 실행해야 한다.
>
> 패키징 SIG는 최종 사용자의 피드백을 적극적으로 찾고 있다. 패키지를 사용해보고
> [opentelemetry-packaging](https://github.com/open-telemetry/opentelemetry-packaging)
> 저장소에 이슈를 등록하기 바란다.

## 더 알아보기 {#learn-more}

- 블로그 게시물:
  [리눅스 호스트에서의 원커맨드 오픈텔레메트리 설정](/blog/2026/packaging-first-repo/)
- [opentelemetry-packaging](https://github.com/open-telemetry/opentelemetry-packaging)
  저장소와 매주 열리는 SIG 미팅.
