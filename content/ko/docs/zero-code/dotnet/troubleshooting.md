---
title: .NET 자동 계측 문제 해결
linkTitle: 문제 해결
weight: 50
default_lang_commit: f7b06a73758a4972539dcc651ca553fd4f97f211
cSpell:ignore: corehost netfx pjanotti's TRACEFILE
---

## 일반적인 절차 {#general-steps}

오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic Instrumentation)에서
문제가 발생하면, 문제를 파악하는 데 도움이 되는 몇 가지 절차가 있다.

### 상세 로깅 활성화 {#enable-detailed-logging}

상세 디버그 로그는 계측 문제를 해결하는 데 도움이 되며, 조사를 돕기 위해 이
프로젝트의 이슈에 첨부할 수 있다.

오픈텔레메트리 .NET 자동 계측에서 상세 로그를 얻으려면, 계측 대상 프로세스가
시작되기 전에 [`OTEL_LOG_LEVEL`](../configuration#internal-logs) 환경 변수를
`debug`로 설정한다.

기본적으로 라이브러리는 미리 정의된 [위치](../configuration#internal-logs)에
로그 파일을 기록한다. 필요하다면 `OTEL_DOTNET_AUTO_LOG_DIRECTORY` 환경 변수를
갱신하여 기본 위치를 변경한다.

로그를 얻은 후에는 불필요한 오버헤드를 피하기 위해 `OTEL_LOG_LEVEL` 환경 변수를
제거하거나 덜 상세한 수준으로 설정한다.

### 호스트 트레이싱 활성화 {#enable-host-tracing}

[호스트 트레이싱](https://github.com/dotnet/runtime/blob/edd23fcb1b350cb1a53fa409200da55e9c33e99e/docs/design/features/host-tracing.md#host-tracing)은
어셈블리를 찾지 못하는 등 다양한 문제를 조사하는 데 필요한 정보를 수집하는 데
사용할 수 있다. 다음 환경 변수를 설정한다.

```terminal
COREHOST_TRACE=1
COREHOST_TRACEFILE=corehost_verbose_tracing.log
```

그런 다음 애플리케이션을 재시작하여 로그를 수집한다.

## 자주 발생하는 문제 {#common-issues}

### 텔레메트리가 생성되지 않음 {#no-telemetry-is-produced}

텔레메트리가 생성되지 않는다. 오픈텔레메트리 .NET 자동 계측 내부 로그
[위치](../configuration#internal-logs)에도 로그가 없다.

.NET 프로파일러가 연결되지 못해 로그가 전혀 남지 않는 경우가 있을 수 있다.

가장 흔한 원인은 계측 대상 애플리케이션에 오픈텔레메트리 .NET 자동 계측
어셈블리를 로드할 권한이 없는 것이다.

### 'OpenTelemetry.AutoInstrumentation.Runtime.Native' 패키지를 설치할 수 없음 {#could-not-install-package-opentelemetryautoinstrumentationruntimenative}

프로젝트에 NuGet 패키지를 추가할 때 다음과 비슷한 오류 메시지가 나타난다.

```txt
Could not install package 'OpenTelemetry.AutoInstrumentation.Runtime.Native 1.6.0'. You are trying to install this package into a project that targets '.NETFramework,Version=v4.7.2', but the package does not contain any assembly references or content files that are compatible with that framework. For more information, contact the package author.
```

NuGet 패키지는 구식 스타일의 `csproj` 프로젝트를 지원하지 않는다. NuGet 패키지를
사용하는 대신 자동 계측을 머신에 배포하거나, 프로젝트를 SDK 스타일 `csproj`로
마이그레이션한다.

### 성능 문제 {#performance-issues}

CPU 사용량이 높다면, 시스템 범위나 사용자 범위에서 환경 변수를 설정하여 자동
계측을 전역으로 활성화하지 않았는지 확인한다.

시스템 범위나 사용자 범위 사용이 의도된 것이라면,
[`OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES`](../configuration#global-settings) 환경
변수를 사용하여 자동 계측에서 애플리케이션을 제외한다.

### `dotnet` CLI 도구가 비정상 종료됨 {#dotnet-cli-tool-is-crashing}

예를 들어 `dotnet run`으로 앱을 실행할 때 다음과 비슷한 오류 메시지가 나타난다.

```txt
PS C:\Users\Administrator\Desktop\OTelConsole-NET6.0> dotnet run My.Simple.Console
Unhandled exception. System.Reflection.TargetInvocationException: Exception has been thrown by the target of an invocation.
---> System.Reflection.TargetInvocationException: Exception has been thrown by the target of an invocation.
---> System.TypeInitializationException: The type initializer for 'OpenTelemetry.AutoInstrumentation.Loader.Startup' threw an exception.
---> System.Reflection.TargetInvocationException: Exception has been thrown by the target of an invocation.
---> System.IO.FileNotFoundException: Could not load file or assembly 'Microsoft.Extensions.Configuration.Abstractions, Version=7.0.0.0, Culture=neutral, PublicKeyToken=adb9793829ddae60'. The system cannot find the file specified.
```

`v0.6.0-beta.1` 이하 버전에서는 `dotnet` CLI 도구를 계측할 때 문제가 있었다.

따라서 이 버전 중 하나를 사용하고 있다면, 터미널 세션을 계측하기 전에
`dotnet build`를 실행하거나 별도의 터미널 세션에서 실행하는 것을 권장한다.

자세한 내용은
[#1744](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/1744)를
참고한다.

### 어셈블리 버전 충돌 {#assembly-version-conflicts}

다음과 비슷한 오류 메시지가 나타난다.

```txt
Unhandled exception. System.IO.FileNotFoundException: Could not load file or assembly 'Microsoft.Extensions.DependencyInjection.Abstractions, Version=7.0.0.0, Culture=neutral, PublicKeyToken=adb9793829ddae60'. The system cannot find the file specified.

File name: 'Microsoft.Extensions.DependencyInjection.Abstractions, Version=7.0.0.0, Culture=neutral, PublicKeyToken=adb9793829ddae60'
   at Microsoft.AspNetCore.Builder.WebApplicationBuilder..ctor(WebApplicationOptions options, Action`1 configureDefaults)
   at Microsoft.AspNetCore.Builder.WebApplication.CreateBuilder(String[] args)
   at Program.<Main>$(String[] args) in /Blog.Core/Blog.Core.Api/Program.cs:line 26
```

오픈텔레메트리 .NET NuGet 패키지와 그 의존성은 오픈텔레메트리 .NET 자동 계측과
함께 배포된다.

의존성 버전 충돌을 처리하려면, 계측 대상 애플리케이션의 프로젝트 참조를
오픈텔레메트리 .NET 자동 계측과 동일한 버전을 사용하도록 갱신한다.

이러한 충돌이 발생하지 않도록 하는 간단한 방법은 애플리케이션에
`OpenTelemetry.AutoInstrumentation` 패키지를 추가하는 것이다. 애플리케이션에
추가하는 방법은
[OpenTelemetry.AutoInstrumentation NuGet 패키지 사용하기](../nuget-packages)를
참고한다.

또는 충돌하는 패키지만 프로젝트에 추가한다. 오픈텔레메트리 .NET 자동 계측은 다음
의존성을 사용한다.

- [OpenTelemetry.AutoInstrumentation](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/src/OpenTelemetry.AutoInstrumentation/OpenTelemetry.AutoInstrumentation.csproj)
- [OpenTelemetry.AutoInstrumentation.AdditionalDeps](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/c27acd9bd0f82de47217fba660d9f979e0a0cc2d/src/OpenTelemetry.AutoInstrumentation.AdditionalDeps/Directory.Build.props)

해당 버전은 다음 위치에서 확인한다.

- [Directory.Packages.props](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/Directory.Packages.props)
- [src/Directory.Packages.props](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/src/Directory.Packages.props)
- [src/OpenTelemetry.AutoInstrumentation.AdditionalDeps/Directory.Packages.props](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/f2d70bd0f095852bf0270aad61b60dfe1ea7834f/src/OpenTelemetry.AutoInstrumentation.AdditionalDeps/Directory.Packages.props)

기본적으로 .NET Framework 애플리케이션의 어셈블리 참조는 런타임 중에 자동 계측이
사용하는 버전으로 리디렉션된다. 이 동작은
[`OTEL_DOTNET_AUTO_NETFX_REDIRECT_ENABLED`](../configuration) 설정으로 제어할 수
있다.

애플리케이션이 자동 계측에서 사용하는 어셈블리에 대해 이미 바인딩 리디렉션을
제공하고 있다면 이 자동 리디렉션이 실패할 수 있다.
[#2833](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2833)를
참고한다.
[netfx_assembly_redirection.h](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/62b4a6a855608a925caeea95752167df5a0960a0/src/OpenTelemetry.AutoInstrumentation.Native/netfx_assembly_redirection.h)에
나열된 버전으로의 리디렉션을 기존 바인딩 리디렉션이 막고 있지는 않은지 확인한다.

위 자동 리디렉션이 동작하려면, .NET Framework 애플리케이션을 계측하는 데
사용되는 어셈블리, 즉 설치 디렉터리의 `netfx` 폴더 아래에 있는 어셈블리를 전역
어셈블리 캐시(Global Assembly Cache, GAC)에도 설치해야 하는 두 가지 특정
시나리오가 있다.

1. 도메인 중립(domain-neutral)으로 로드된 어셈블리의
   [**몽키 패치 계측**](https://en.wikipedia.org/wiki/Monkey_patch).
2. 앱이 `netfx` 폴더에 포함된 것과 다른 버전의 어셈블리도 함께 배포하는 경우,
   강력한 이름(strong-named)이 지정된 애플리케이션에 대한 어셈블리 리디렉션.

위 시나리오 중 하나에서 문제가 발생한다면, PowerShell 설치 모듈에서
`Install-OpenTelemetryCore` 명령을 다시 실행하여 필요한 GAC 설치가 갱신되도록
한다.

자동 계측의 GAC 사용에 대한 자세한 내용은
[pjanotti의 댓글](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/1906#issuecomment-1376292814)을
참고한다.

자세한 내용은
[#2269](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2269)와
[#2296](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2296)를
참고한다.

### AdditionalDeps에 있는 어셈블리를 찾을 수 없음 {#assembly-in-additionaldeps-was-not-found}

#### 증상 {#symptoms}

다음과 비슷한 오류 메시지가 나타난다.

```txt
An assembly specified in the application dependencies manifest (OpenTelemetry.AutoInstrumentation.AdditionalDeps.deps.json) was not found
```

이는 다음 이슈와 관련이 있을 수 있다.

- [#1744](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/1744)
- [#2181](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2181)

## 기타 문제 {#other-issues}

이 페이지에 나열되지 않은 문제가 발생하면, [일반적인 절차](#general-steps)를
참고하여 추가 진단 정보를 수집한다. 이렇게 하면 문제 해결에 도움이 될 수 있다.
