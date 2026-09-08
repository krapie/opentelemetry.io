---
title: .NET 제로 코드 계측
description: .NET 애플리케이션과 서비스에서 트레이스와 메트릭을 전송한다.
linkTitle: .NET
aliases: [net]
redirects: [{ from: /docs/languages/net/automatic/*, to: ':splat' }]
weight: 30
cSpell:ignore: coreutils HKLM iisreset Sonoma
default_lang_commit: f304b22c356b4ad4046b8ce5550ddf503a510d69
---

오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic Instrumentation)을
사용하면 소스 코드를 수정하지 않고도 .NET 애플리케이션과 서비스에서 트레이스와
메트릭을 옵저버빌리티 백엔드로 전송할 수 있다.

서비스나 애플리케이션 코드를 계측하는 방법을 알아보려면
[수동 계측](/docs/languages/dotnet/instrumentation)을 참고한다.

## 호환성 {#compatibility}

오픈텔레메트리 .NET 자동 계측은 공식적으로 지원되는 모든 운영체제 및
[.NET](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
버전에서 동작해야 한다.

지원되는
[.NET Framework](https://dotnet.microsoft.com/download/dotnet-framework)의 최소
버전은 `4.6.2`이다.

지원되는 프로세서 아키텍처는 다음과 같다.

- x86
- AMD64 (x86-64)
- ARM64 ([실험적(Experimental)](/docs/specs/otel/versioning-and-stability))

> [!NOTE]
>
> ARM64 빌드는 CentOS 기반 이미지를 지원하지 않는다.

CI 테스트는 다음 운영체제에서 실행된다.

- [Alpine x64](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docker/alpine.dockerfile)
- [Alpine ARM64](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docker/alpine.dockerfile)
- [Debian x64](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docker/debian.dockerfile)
- [Debian ARM64](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docker/debian-arm64.dockerfile)
- [CentOS Stream 9 x64](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docker/centos-stream9.dockerfile)
- [macOS Sonoma 14 ARM64](https://github.com/actions/runner-images/blob/main/images/macos/macos-14-Readme.md)
- [Microsoft Windows Server 2022 x64](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md)
- [Microsoft Windows Server 2025 x64](https://github.com/actions/runner-images/blob/main/images/windows/Windows2025-Readme.md)
- [Ubuntu 22.04 LTS x64](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md)
- [Ubuntu 22.04 LTS ARM64](https://github.com/actions/partner-runner-images/blob/main/images/arm-ubuntu-22-image.md)

## 설정 {#setup}

.NET 애플리케이션을 자동으로 계측하려면, 사용 중인 운영체제에 맞는 설치
스크립트를 다운로드하여 실행한다.

### Linux 및 macOS {#linux-and-macos}

`.sh` 스크립트를 다운로드하여 실행한다.

> [!NOTE]
>
> 에어갭(air-gapped) 환경에서는 `LOCAL_PATH` 변수를 사용하여 설치 파일을 직접
> 제공한다.
>
> ```shell
> LOCAL_PATH=<PATH_TO_INSTALLER> sh ./otel-dotnet-auto-install.sh
> ```
>
> 또는 `DOWNLOAD_DIR`을 사용하여 파일이 있는 폴더를 제공하면, 설치 스크립트가
> 사용할 올바른 파일을 자동으로 결정한다.
>
> ```shell
> DOWNLOAD_DIR=<PATH_TO_FOLDER_WITH_FILES> sh ./otel-dotnet-auto-install.sh
> ```

```shell
# Download the bash script
curl -sSfL https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/otel-dotnet-auto-install.sh -O

# Install core files
sh ./otel-dotnet-auto-install.sh

# Enable execution for the instrumentation script
chmod +x $HOME/.otel-dotnet-auto/instrument.sh

# Setup the instrumentation for the current shell session
. $HOME/.otel-dotnet-auto/instrument.sh

# Run your application with instrumentation
OTEL_SERVICE_NAME=myapp OTEL_RESOURCE_ATTRIBUTES=deployment.environment.name=staging,service.version=1.0.0 ./MyNetApp
```

> [!IMPORTANT]
>
> macOS에서는 [`coreutils`](https://formulae.brew.sh/formula/coreutils)가
> 필요하다. [homebrew](https://brew.sh/)가 설치되어 있다면 다음을 실행하여
> 설치할 수 있다.
>
> ```shell
> brew install coreutils
> ```

### Windows(PowerShell) {#windows-powershell}

Windows에서는 관리자(Administrator) 권한으로 PowerShell 모듈을 사용한다.

> [!NOTE] 버전 참고
>
> Windows
> [PowerShell Desktop](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_windows_powershell_5.1#powershell-editions)
> (v5.1)가 필요하다. PowerShell Core(v6.0+)를 포함한 다른
> [버전](https://learn.microsoft.com/previous-versions/powershell/scripting/overview)은
> 현재 지원되지 않는다.

```powershell
# PowerShell 5.1 is required
#Requires -PSEdition Desktop

# Download the module
$module_url = "https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/OpenTelemetry.DotNet.Auto.psm1"
$download_path = Join-Path $env:temp "OpenTelemetry.DotNet.Auto.psm1"
Invoke-WebRequest -Uri $module_url -OutFile $download_path -UseBasicParsing

# Import the module to use its functions
Import-Module $download_path

# Install core files (online vs offline method)
Install-OpenTelemetryCore
Install-OpenTelemetryCore -LocalPath "C:\Path\To\OpenTelemetry.zip"

# Set up the instrumentation for the current PowerShell session
Register-OpenTelemetryForCurrentSession -OTelServiceName "MyServiceDisplayName"

# Run your application with instrumentation
.\MyNetApp.exe

# You can get usage information by calling the following commands

# List all available commands
Get-Command -Module OpenTelemetry.DotNet.Auto

# Get command's usage information
Get-Help Install-OpenTelemetryCore -Detailed
```

## .NET 애플리케이션을 실행하는 Windows 서비스 계측하기 {#instrument-a-windows-service-running-a-net-application}

Windows 서비스에 대한 자동 계측을 설정하려면 `OpenTelemetry.DotNet.Auto.psm1`
PowerShell 모듈을 사용한다.

```powershell
# Import the module
Import-Module "OpenTelemetry.DotNet.Auto.psm1"

# Install core files
Install-OpenTelemetryCore

# Set up your Windows Service instrumentation
Register-OpenTelemetryForWindowsService -WindowsServiceName "WindowsServiceName" -OTelServiceName "MyServiceDisplayName"
```

> [!CAUTION]
>
> `Register-OpenTelemetryForWindowsService`는 서비스를 재시작한다.

### Windows 서비스를 위한 구성 {#configuration-for-windows-service}

> [!IMPORTANT]
>
> 구성 변경 후에는 Windows 서비스를 재시작해야 한다는 점을 기억한다.
> PowerShell에서 `Restart-Service -Name $WindowsServiceName -Force`를 실행하여
> 재시작할 수 있다.

.NET Framework 애플리케이션의 경우 `App.config`의 `appSettings`를 통해
[가장 일반적인 `OTEL_` 설정](/docs/specs/otel/configuration/sdk-environment-variables/#general-sdk-configuration)
(예: `OTEL_RESOURCE_ATTRIBUTES`)을 구성할 수 있다.

대안으로 Windows 레지스트리(Registry)에서 Windows 서비스의 환경 변수를 설정할
수도 있다.

특정 Windows 서비스(이름이 `$svcName`인 경우)의 레지스트리 키는 다음 위치에
있다.

```powershell
HKLM\SYSTEM\CurrentControlSet\Services\$svcName
```

환경 변수는 `Environment`라는 이름의 `REG_MULTI_SZ`(여러 줄 레지스트리 값)에
다음 형식으로 정의된다.

```env
Var1=Value1
Var2=Value2
```

## IIS에 배포된 ASP.NET 애플리케이션 계측하기 {#instrument-an-aspnet-application-deployed-on-iis}

> [!NOTE]
>
> 다음 안내는 .NET Framework 애플리케이션에 적용된다.

IIS에 대한 자동 계측을 설정하려면 `OpenTelemetry.DotNet.Auto.psm1` PowerShell
모듈을 사용한다.

```powershell
# Import the module
Import-Module "OpenTelemetry.DotNet.Auto.psm1"

# Install core files
Install-OpenTelemetryCore

# Setup IIS instrumentation
Register-OpenTelemetryForIIS
```

> [!CAUTION]
>
> `Register-OpenTelemetryForIIS`는 IIS를 재시작한다.

### ASP.NET 애플리케이션을 위한 구성 {#configuration-for-aspnet-applications}

> [!NOTE]
>
> 다음 안내는 .NET Framework 애플리케이션에 적용된다.

ASP.NET 애플리케이션의 경우 `Web.config`의 `appSettings`를 통해
[가장 일반적인 `OTEL_` 설정](/docs/specs/otel/configuration/sdk-environment-variables/#general-sdk-configuration)
(예: `OTEL_SERVICE_NAME`)을 구성할 수 있다.

서비스 이름이 명시적으로 구성되지 않으면 자동으로 생성된다. .NET Framework에서
애플리케이션이 IIS에 호스팅되는 경우 `SiteName\VirtualDirectoryPath` 형식(예:
`MySite\MyApp`)이 사용된다.

ASP.NET Core 애플리케이션의 경우 `Web.config` 파일의 `<aspNetCore>` 블록 내부에
있는
[`<environmentVariable>`](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/web-config#set-environment-variables)
요소를 사용하여 환경 변수로 구성을 설정할 수 있다.

> [!IMPORTANT]
>
> 구성 변경 후에는 IIS를 재시작해야 한다는 점을 기억한다. `iisreset.exe`를
> 실행하여 재시작할 수 있다.

### 고급 구성 {#advanced-configuration}

`applicationHost.config`에
[`<environmentVariables>`](https://docs.microsoft.com/en-us/iis/configuration/system.applicationhost/applicationpools/add/environmentvariables/)를
추가하여 특정 애플리케이션 풀(pool)에 대한 환경 변수를 설정할 수 있다.

IIS에 배포된 모든 애플리케이션에 공통 환경 변수를 설정하려면, `W3SVC` 및 `WAS`
Windows 서비스의 환경 변수를 설정하는 것을 고려한다.

> [!TIP]
>
> 10.0보다 이전 버전의 IIS에서는, 별도의 사용자를 생성하고 해당 사용자의 환경
> 변수를 설정한 뒤 이를 애플리케이션 풀 사용자로 사용하는 것을 고려할 수 있다.

## NuGet 패키지 {#nuget-package}

NuGet 패키지를 사용하여
[`self-contained`](https://learn.microsoft.com/en-us/dotnet/core/deploying/#publish-as-self-contained)
애플리케이션을 계측할 수 있다. 자세한 내용은 [NuGet 패키지](./nuget-packages)를
참고한다.

## 컨테이너 계측하기 {#instrument-a-container}

Docker 컨테이너 계측 예시는 GitHub의
[예시](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/tree/main/examples/demo)를
참고한다.

[쿠버네티스용 오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator/)를
사용할 수도 있다.

## 에이전트 구성하기 {#configuring-the-agent}

전체 구성 옵션을 확인하려면 [구성 및 설정](./configuration)을 참고한다.

## 로그와 트레이스 상관관계(correlation) {#log-to-trace-correlation}

> [!NOTE]
>
> 오픈텔레메트리 .NET 자동 계측이 제공하는 자동 로그-트레이스
> 상관관계(correlation)는 현재 `Microsoft.Extensions.Logging`을 사용하는 .NET
> 애플리케이션에서만 동작한다. 자세한 내용은 [#2310][]를 참고한다.

[#2310]:
  https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2310

오픈텔레메트리 .NET SDK는 로그를 트레이스 데이터와 자동으로 연관짓는다. 활성
트레이스의 컨텍스트 내에서 로그가 발생하면, 트레이스 컨텍스트
[필드](/docs/specs/otel/logs/data-model#trace-context-fields)인 `TraceId`,
`SpanId`, `TraceState`가 자동으로 채워진다.

다음은 샘플 콘솔 애플리케이션이 생성한 로그이다.

```json
"logRecords": [
    {
        "timeUnixNano": "1679392614538226700",
        "severityNumber": 9,
        "severityText": "Information",
        "body": {
            "stringValue": "Success! Today is: {Date:MMMM dd, yyyy}"
        },
        "flags": 1,
        "traceId": "21df288eada1ce4ace6c40f39a6d7ce1",
        "spanId": "a80119e5a05fed5a"
    }
]
```

자세한 내용은 다음을 참고한다.

- [오픈텔레메트리 .NET SDK](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs/logs/correlation)
- [오픈텔레메트리 명세](/docs/specs/otel/logs/data-model#trace-context-fields)

## 지원되는 라이브러리 및 프레임워크 {#supported-libraries-and-frameworks}

오픈텔레메트리 .NET 자동 계측은 다양한 라이브러리를 지원한다. 전체 목록은
[계측 목록](./instrumentations)을 참고한다.

## 문제 해결 {#troubleshooting}

애플리케이션을 실행하기 전에 다음 환경 변수 값에 `console`을 추가하면
애플리케이션의 텔레메트리를 표준 출력에서 직접 확인할 수 있다.

- `OTEL_TRACES_EXPORTER`
- `OTEL_METRICS_EXPORTER`
- `OTEL_LOGS_EXPORTER`

일반적인 문제 해결 단계와 특정 문제에 대한 해결 방법은
[문제 해결](./troubleshooting)을 참고한다.

## 다음 단계 {#next-steps}

애플리케이션이나 서비스에 자동 계측을 구성한 후에는,
[커스텀 트레이스와 메트릭 전송하기](./custom)를 진행하거나
[수동 계측](/docs/languages/dotnet/instrumentation)을 추가하여 커스텀 텔레메트리
데이터를 수집(collect)하고 싶을 수 있다.

## 제거 {#uninstall}

### Linux 및 macOS {#uninstall-unix}

Linux 및 macOS에서는 설치 단계가 현재 셸 세션에만 영향을 미치므로, 별도의 제거
작업이 필요하지 않다.

### Windows(PowerShell) {#uninstall-windows}

Windows에서는 관리자(Administrator) 권한으로 PowerShell 모듈을 사용한다.

> [!IMPORTANT] 버전 참고
>
> Windows [PowerShell Desktop][] (v5.1)가 필요하다. PowerShell Core(v6.0+)를
> 포함한 다른 [버전][versions]은 현재 지원되지 않는다.

[PowerShell Desktop]:
  https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_windows_powershell_5.1#powershell-editions
[versions]:
  https://learn.microsoft.com/previous-versions/powershell/scripting/overview

```powershell
# PowerShell 5.1 is required
#Requires -PSEdition Desktop

# Import the previously installed module
Import-Module "OpenTelemetry.DotNet.Auto.psm1"

# If IIS was previously registered, unregister it
Unregister-OpenTelemetryForIIS

# If Windows services were previously registered, unregister them
Unregister-OpenTelemetryForWindowsService -WindowsServiceName "WindowsServiceName"

# Finally, uninstall OpenTelemetry instrumentation
Uninstall-OpenTelemetryCore
```
