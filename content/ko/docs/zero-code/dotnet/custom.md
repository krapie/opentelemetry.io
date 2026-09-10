---
title: 커스텀 트레이스와 메트릭 만들기
linkTitle: 커스텀 계측
description: .NET 자동 계측을 사용한 커스텀 트레이스와 메트릭.
weight: 30
default_lang_commit: 66d51e71882ea2bbdfba5c47ccf0efd6e5175b39
cSpell:ignore: meterprovider tracerprovider
---

자동 계측은 `TracerProvider`와 `MeterProvider`를 구성하므로, 직접 작성한 수동
계측을 추가할 수 있다. 자동 계측과 수동 계측을 함께 사용하면 애플리케이션,
클라이언트, 프레임워크의 로직과 기능을 더 잘 계측할 수 있다.

## 트레이스 {#traces}

커스텀 트레이스를 수동으로 만들려면 다음 단계를 따른다.

1. 프로젝트에 `System.Diagnostics.DiagnosticSource` 의존성을 추가한다.

   ```xml
   <PackageReference Include="System.Diagnostics.DiagnosticSource" Version="8.0.0" />
   ```

2. `ActivitySource` 인스턴스를 생성한다.

   ```csharp
   private static readonly ActivitySource RegisteredActivity = new ActivitySource("Examples.ManualInstrumentations.Registered");
   ```

3. `Activity`를 생성한다. 필요하면 태그를 설정한다.

   ```csharp
   using (var activity = RegisteredActivity.StartActivity("Main"))
   {
      activity?.SetTag("foo", "bar1");
      // your logic for Main activity
   }
   ```

4. `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES` 환경 변수를 설정하여
   OpenTelemetry.AutoInstrumentation에 `ActivitySource`를 등록한다. 값으로
   `Examples.ManualInstrumentations.Registered`를 설정하거나, 접두사 전체를
   등록하는 `Examples.ManualInstrumentations.*`를 설정할 수 있다.

> [!WARNING]
>
> `NonRegistered.ManualInstrumentations` `ActivitySource`에 대해 생성된
> `Activity`는 오픈텔레메트리(OpenTelemetry) 자동 계측이 처리하지 않는다.

## 메트릭 {#metrics}

커스텀 메트릭을 수동으로 만들려면 다음 단계를 따른다.

1. 프로젝트에 `System.Diagnostics.DiagnosticSource` 의존성을 추가한다.

   ```xml
   <PackageReference Include="System.Diagnostics.DiagnosticSource" Version="8.0.0" />
   ```

2. `Meter` 인스턴스를 생성한다.

   ```csharp
   using var meter = new Meter("Examples.Service", "1.0");
   ```

3. `Instrument`(계측기)를 생성한다.

   ```csharp
   var successCounter = meter.CreateCounter<long>("srv.successes.count", description: "Number of successful responses");
   ```

4. `Instrument` 값을 갱신한다. 필요하면 태그를 설정한다.

   ```csharp
   successCounter.Add(1, new KeyValuePair<string, object?>("tagName", "tagValue"));
   ```

5. `OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES` 환경 변수를 설정하여
   OpenTelemetry.AutoInstrumentation에 `Meter`를 등록한다.

   ```bash
   OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES=Examples.Service
   ```

   값으로 `Examples.Service`를 설정하거나, 접두사 전체를 등록하는 `Examples.*`를
   설정할 수 있다.

## 더 읽어보기 {#further-reading}

- [.NET 수동 계측에 대한 OpenTelemetry.io 문서](/docs/languages/dotnet/instrumentation#setting-up-an-activitysource)
