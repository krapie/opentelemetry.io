---
title: OpenTelemetry.AutoInstrumentation NuGet 패키지 사용하기
linkTitle: NuGet 패키지
weight: 40
default_lang_commit: f304b22c356b4ad4046b8ce5550ddf503a510d69
cSpell:ignore: buildtasks
---

다음과 같은 상황에서 NuGet 패키지를 사용한다.

1. 배포를 단순화한다. 예를 들어 단일 애플리케이션을 실행하는 컨테이너가 있다.
1. [`self-contained`](https://learn.microsoft.com/en-us/dotnet/core/deploying/#publish-as-self-contained)
   애플리케이션의 계측을 지원한다.
1. NuGet 패키지를 통해 개발자가 자동 계측을 손쉽게 실험할 수 있도록 한다.
1. 애플리케이션이 사용하는 의존성과 자동 계측 사이의 버전 충돌을 해결한다.

## 제약 사항 {#limitations}

NuGet 패키지는 자동 계측을 배포하는 편리한 방법이지만, 모든 경우에 사용할 수
있는 것은 아니다. NuGet 패키지를 사용하지 않는 가장 일반적인 이유는 다음과 같다.

1. 애플리케이션 프로젝트에 패키지를 추가할 수 없다. 예를 들어 애플리케이션이
   패키지를 추가할 수 없는 서드파티 제품인 경우가 있다.
1. 계측 대상 애플리케이션 여러 개가 한 대의 머신에 설치되어 있을 때, 디스크
   사용량이나 가상 머신의 크기를 줄이려는 경우가 있다. 이 경우 머신에서 실행되는
   모든 .NET 애플리케이션에 대해 단일 배포를 사용할 수 있다.
1. [SDK 스타일 프로젝트](https://learn.microsoft.com/en-us/nuget/resources/check-project-format#check-the-project-format)로
   마이그레이션할 수 없는 레거시 애플리케이션인 경우가 있다.

## NuGet 패키지 사용하기 {#using-the-nuget-packages}

오픈텔레메트리(OpenTelemetry) .NET으로 애플리케이션을 자동으로 계측하려면
프로젝트에 `OpenTelemetry.AutoInstrumentation` 패키지를 추가한다.

```terminal
dotnet add [<PROJECT>] package OpenTelemetry.AutoInstrumentation
```

애플리케이션이 계측할 수 있는 패키지를 참조하지만 계측이 동작하려면 다른
패키지가 필요한 경우, 빌드가 실패하면서 누락된 계측 라이브러리를 추가하거나 해당
패키지의 계측을 건너뛰도록 안내한다.

```terminal
~packages/opentelemetry.autoinstrumentation.buildtasks/1.6.0/build/OpenTelemetry.AutoInstrumentation.BuildTasks.targets(29,5): error : OpenTelemetry.AutoInstrumentation: add a reference to the instrumentation package 'MongoDB.Driver.Core.Extensions.DiagnosticSources' version 1.4.0 or add 'MongoDB.Driver.Core' to the property 'SkippedInstrumentations' to suppress this error.
```

이 오류를 해결하려면 권장 계측 라이브러리를 추가하거나, 나열된 패키지를
`SkippedInstrumentation` 속성에 추가하여 해당 패키지의 계측을 건너뛴다. 예시는
다음과 같다.

```csproj
<PropertyGroup>
   <SkippedInstrumentations>MongoDB.Driver.Core;StackExchange.Redis</SkippedInstrumentations>
</PropertyGroup>
```

같은 속성을 CLI에서 직접 지정할 수도 있다. 이때 구분자 `;`는 '%3B'로 적절히
이스케이프해야 한다.

```powershell
dotnet build -p:SkippedInstrumentations=StackExchange.Redis%3BMongoDB.Driver.Core
```

.NET 애플리케이션과 함께 적절한 네이티브 런타임 구성 요소를 배포하려면,
`dotnet build`나 `dotnet publish`로 애플리케이션을 빌드할 때
[런타임 식별자(Runtime Identifier, RID)](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog)를
지정한다. 이렇게 하려면
[_self-contained_ 또는 _framework-dependent_](https://learn.microsoft.com/en-us/dotnet/core/deploying/)
애플리케이션 중 무엇을 배포할지 선택해야 할 수 있다. 두 유형 모두 자동 계측과
호환된다.

빌드 결과가 출력되는 폴더에 있는 스크립트를 사용하여 자동 계측이 활성화된 상태로
애플리케이션을 실행한다.

- Windows에서는 `instrument.cmd <application_executable>`을 사용한다.
- Linux나 Unix에서는 `instrument.sh <application_executable>`을 사용한다.

`dotnet` CLI로 애플리케이션을 실행한다면 스크립트 뒤에 `dotnet`을 추가한다.

- Windows에서는 `instrument.cmd dotnet <application>`을 사용한다.
- Linux와 Unix에서는 `instrument.sh dotnet <application>`을 사용한다.

이 스크립트는 사용자가 제공한 모든 명령줄 매개변수를 애플리케이션에 전달한다.
