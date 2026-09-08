---
title: Go 컴파일 타임 계측
linkTitle: 컴파일 타임
description: 코드 변경 없이 빌드 시점에 Go 애플리케이션을 계측한다.
weight: 20
cSpell:ignore: otelc toolexec
default_lang_commit: d2e57aa2352b9e64c117d07c304923ac12b5d7e8
---

Go를 위한 컴파일 타임 계측(compile-time instrumentation)은 빌드 중에
애플리케이션을 자동으로 계측하여, 소스 코드를 전혀 변경하지 않고도 다양한 인기
라이브러리 및 프레임워크로부터 텔레메트리를 캡처한다. Go 빌드 프로세스에
후킹(hook)하여 애플리케이션이 컴파일되는 동안 계측을 주입하는 방식으로 동작하며,
그 결과로 생성되는 바이너리에는 런타임 에이전트 없이도 계측 코드가 포함된다.

> [!NOTE]
>
> 이 프로젝트는 v1.0.0부터 안정적이며 프로덕션에서 사용할 준비가 되어 있다. 개발
> 진행 상황을 따라가거나 참여하려면
> [opentelemetry-go-compile-instrumentation 저장소](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation)를
> 방문한다.

## 동작 방식 {#how-it-works}

`otelc` 도구는 일반적인 `go build` 호출을 감싼다. 빌드 중에 다음과 같은 작업을
수행한다.

1. Go 툴체인(toolchain)의 `-toolexec` 메커니즘을 사용하여 각 패키지의 컴파일을
   가로챈다.
2. 텔레메트리 후크(hook)가 위치해야 할 곳을 설명하는 일련의 계측 규칙에 대해
   패키지와 함수를 매칭한다.
3. 매칭된 함수에 경량 후크 지점을 주입하고, 이를 오픈텔레메트리 계측 코드와
   연결한다.

계측이 컴파일 타임에 바이너리에 내장되기 때문에, 직접 제어할 수 없는
서드파티(third-party) 의존성까지도 포함하며, 런타임 연결(attach)이나 시작 단계가
추가로 필요하지 않다.

## 사용 시점 {#when-to-use-it}

컴파일 타임 계측은 Go를 위한 여러
[제로 코드](/docs/concepts/instrumentation/zero-code/) 옵션 중 하나로,
[Auto SDK](/docs/zero-code/go/autosdk/)를 보완하며, eBPF 기반 계측은
[OBI](/docs/zero-code/obi/)를 통해 사용할 수 있다. 컴파일 타임 계측은 소스
코드는 수정할 수 없지만 빌드 파이프라인은 수정할 수 있는 경우, 서드파티
라이브러리 내부까지 계측하고 싶은 경우, 또는 애플리케이션과 함께 권한이
있는(privileged) 에이전트를 실행할 수 없는 경우에 적합하다.
