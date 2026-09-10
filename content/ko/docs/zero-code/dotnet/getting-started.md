---
title: 시작하기
description: 5분 이내에 애플리케이션의 텔레메트리를 확인한다!
weight: 5
default_lang_commit: d1ef521ee4a777881fb99c3ec2b506e068cdec4c
cSpell:ignore: ASPNETCORE rolldice
---

이 페이지에서는 오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic
Instrumentation)을 시작하는 방법을 설명한다.

애플리케이션을 수동으로 계측하는 방법을 찾고 있다면
[이 가이드](/docs/languages/dotnet/getting-started)를 확인한다.

간단한 .NET 애플리케이션을 자동으로 계측하여 [트레이스][traces],
[메트릭][metrics], [로그][logs]가 콘솔로 전송되도록 하는 방법을 배운다.

## 사전 준비 사항 {#prerequisites}

로컬에 다음이 설치되어 있는지 확인한다.

- [.NET SDK](https://dotnet.microsoft.com/download/dotnet) 6 이상

## 예시 애플리케이션 {#example-application}

다음 예시는 기본적인
[ASP.NET Core 기반 Minimal API](https://learn.microsoft.com/aspnet/core/tutorials/min-web-api)
애플리케이션을 사용한다. ASP.NET Core를 사용하지 않아도 괜찮다. 여전히
오픈텔레메트리 .NET 자동 계측을 사용할 수 있다.

더 정교한 예시는 [예시](/docs/languages/dotnet/examples/)를 참고한다.

### HTTP 서버 만들고 실행하기 {#create-and-launch-an-http-server}

먼저 `dotnet-simple`이라는 새 디렉터리에 환경을 설정한다. 그 디렉터리 안에서
다음 명령을 실행한다.

```sh
dotnet new web
```

같은 디렉터리에서 `Program.cs`의 내용을 다음 코드로 교체한다.

```csharp
using System.Globalization;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var logger = app.Logger;

int RollDice()
{
    return Random.Shared.Next(1, 7);
}

string HandleRollDice(string? player)
{
    var result = RollDice();

    if (string.IsNullOrEmpty(player))
    {
        logger.LogInformation("Anonymous player is rolling the dice: {result}", result);
    }
    else
    {
        logger.LogInformation("{player} is rolling the dice: {result}", player, result);
    }

    return result.ToString(CultureInfo.InvariantCulture);
}

app.MapGet("/rolldice/{player?}", HandleRollDice);

app.Run();
```

`Properties` 하위 디렉터리에서 `launchSettings.json`의 내용을 다음으로 교체한다.

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:8080",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

다음 명령으로 애플리케이션을 빌드하고 실행한 뒤, 웹 브라우저에서
<http://localhost:8080/rolldice>를 열어 정상 동작하는지 확인한다.

```sh
dotnet build
dotnet run
```

## 계측 {#instrumentation}

다음으로, [오픈텔레메트리 .NET 자동 계측](../)을 사용하여 실행 시점에
애플리케이션을 계측한다. [.NET 자동 계측
구성][configure .NET Automatic Instrumentation]은 여러 가지 방법으로 할 수
있지만, 아래 단계에서는 Unix 셸이나 PowerShell 스크립트를 사용한다.

> **참고**: PowerShell 명령은 상승된(관리자) 권한이 필요하다.

1. `opentelemetry-dotnet-instrumentation` 저장소의 [Releases][]에서 설치
   스크립트를 다운로드한다.

   {{< tabpane text=true >}} {{% tab Unix-shell %}}

   ```sh
   curl -L -O https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/otel-dotnet-auto-install.sh
   ```

   {{% /tab %}} {{% tab PowerShell - Windows %}}

   ```powershell
   $module_url = "https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/OpenTelemetry.DotNet.Auto.psm1"
   $download_path = Join-Path $env:temp "OpenTelemetry.DotNet.Auto.psm1"
   Invoke-WebRequest -Uri $module_url -OutFile $download_path -UseBasicParsing
   ```

   {{% /tab %}} {{< /tabpane >}}

2. 다음 스크립트를 실행하여 개발 환경에 맞는 자동 계측을 다운로드한다.

   {{< tabpane text=true >}} {{% tab Unix-shell %}}

   ```sh
   ./otel-dotnet-auto-install.sh
   ```

   {{% /tab %}} {{% tab PowerShell - Windows %}}

   ```powershell
   Import-Module $download_path
   Install-OpenTelemetryCore
   ```

   {{% /tab %}} {{< /tabpane >}}

3. [콘솔 익스포터][console exporter]를 지정하는 변수를 설정하고 export한 뒤,
   사용하는 셸/터미널 환경에 맞는 표기법으로 나머지 필요한 환경 변수를 구성하는
   스크립트를 실행한다. 여기서는 bash 계열 셸과 PowerShell의 표기법을 예로 든다.

   {{< tabpane text=true >}} {{% tab Unix-shell %}}

   ```sh
   export OTEL_TRACES_EXPORTER=console \
     OTEL_METRICS_EXPORTER=console \
     OTEL_LOGS_EXPORTER=console
     OTEL_SERVICE_NAME=RollDiceService
   . $HOME/.otel-dotnet-auto/instrument.sh
   ```

   {{% /tab %}} {{% tab PowerShell - Windows %}}

   ```powershell
   $env:OTEL_TRACES_EXPORTER="console"
   $env:OTEL_METRICS_EXPORTER="console"
   $env:OTEL_LOGS_EXPORTER="console"
   Register-OpenTelemetryForCurrentSession -OTelServiceName "RollDiceService"
   ```

   {{% /tab %}} {{< /tabpane >}}

4. **애플리케이션**을 다시 한번 실행한다.

   ```sh
   dotnet run
   ```

   `dotnet run`의 출력을 확인한다.

5. _다른_ 터미널에서 `curl`로 요청을 보낸다.

   ```sh
   curl localhost:8080/rolldice
   ```

6. 약 30초 후에 서버 프로세스를 중지한다.

이 시점에서 다음과 비슷한 서버와 클라이언트의 트레이스 및 로그 출력을 볼 수 있다
(가독성을 위해 출력을 줄 바꿈했다).

<details>
<summary>트레이스와 로그</summary>

```log
LogRecord.Timestamp:               2023-08-14T06:44:53.9279186Z
LogRecord.TraceId:                 3961d22b5f90bf7662ad4933318743fe
LogRecord.SpanId:                  93d5fcea422ff0ac
LogRecord.TraceFlags:              Recorded
LogRecord.CategoryName:            simple-dotnet
LogRecord.LogLevel:                Information
LogRecord.StateValues (Key:Value):
    result: 1
    OriginalFormat (a.k.a Body): Anonymous player is rolling the dice: {result}

Resource associated with LogRecord:
service.name: simple-dotnet
telemetry.auto.version: 0.7.0
telemetry.sdk.name: opentelemetry
telemetry.sdk.language: dotnet
telemetry.sdk.version: 1.4.0.802

info: simple-dotnet[0]
      Anonymous player is rolling the dice: 1
Activity.TraceId:            3961d22b5f90bf7662ad4933318743fe
Activity.SpanId:             93d5fcea422ff0ac
Activity.TraceFlags:         Recorded
Activity.ActivitySourceName: OpenTelemetry.Instrumentation.AspNetCore
Activity.DisplayName:        /rolldice
Activity.Kind:               Server
Activity.StartTime:          2023-08-14T06:44:53.9278162Z
Activity.Duration:           00:00:00.0049754
Activity.Tags:
    net.host.name: localhost
    net.host.port: 8080
    http.method: GET
    http.scheme: http
    http.target: /rolldice
    http.url: http://localhost:8080/rolldice
    http.flavor: 1.1
    http.user_agent: curl/8.0.1
    http.status_code: 200
Resource associated with Activity:
    service.name: simple-dotnet
    telemetry.auto.version: 0.7.0
    telemetry.sdk.name: opentelemetry
    telemetry.sdk.language: dotnet
    telemetry.sdk.version: 1.4.0.802
```

</details>

또한 서버를 중지하면 수집된 모든 메트릭의 출력을 볼 수 있다(일부 발췌 표시).

<details>
<summary>메트릭</summary>

```log
Export process.runtime.dotnet.gc.collections.count, Number of garbage collections that have occurred since process start., Meter: OpenTelemetry.Instrumentation.Runtime/1.1.0.2
(2023-08-14T06:12:05.8500776Z, 2023-08-14T06:12:23.7750288Z] generation: gen2 LongSum
Value: 2
(2023-08-14T06:12:05.8500776Z, 2023-08-14T06:12:23.7750288Z] generation: gen1 LongSum
Value: 2
(2023-08-14T06:12:05.8500776Z, 2023-08-14T06:12:23.7750288Z] generation: gen0 LongSum
Value: 6

...

Export http.client.duration, Measures the duration of outbound HTTP requests., Unit: ms, Meter: OpenTelemetry.Instrumentation.Http/1.0.0.0
(2023-08-14T06:12:06.2661140Z, 2023-08-14T06:12:23.7750388Z] http.flavor: 1.1 http.method: POST http.scheme: https http.status_code: 200 net.peer.name: dc.services.visualstudio.com Histogram
Value: Sum: 1330.4766000000002 Count: 5 Min: 50.0333 Max: 465.7936
(-Infinity,0]:0
(0,5]:0
(5,10]:0
(10,25]:0
(25,50]:0
(50,75]:2
(75,100]:0
(100,250]:0
(250,500]:3
(500,750]:0
(750,1000]:0
(1000,2500]:0
(2500,5000]:0
(5000,7500]:0
(7500,10000]:0
(10000,+Infinity]:0
```

</details>

## 다음은 무엇을 할까? {#what-next}

더 알아보려면 다음을 참고한다.

- 익스포터, 샘플러, 리소스 등을 구성하려면 [구성 및 설정](../configuration)을
  참고한다.
- [사용 가능한 계측](../instrumentations) 목록을 확인한다.
- 자동 계측과 수동 계측을 함께 사용하고 싶다면
  [커스텀 트레이스와 메트릭을 만드는 방법](../custom)을 알아본다.
- 문제가 발생하면 [문제 해결 가이드](../troubleshooting)를 확인한다.

[traces]: /docs/concepts/signals/traces/
[metrics]: /docs/concepts/signals/metrics/
[logs]: /docs/concepts/signals/logs/
[configure .NET Automatic Instrumentation]: ../configuration
[console exporter]:
  https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/config.md#internal-logs
[releases]:
  https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases
