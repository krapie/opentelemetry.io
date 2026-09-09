---
title: 회계 서비스
linkTitle: 회계
aliases: [accountingservice]
default_lang_commit: 54b262d6b6703e304f6c42b41805a715d7ebee20
---

이 서비스는 판매된 상품의 총액을 계산한다. 이 계산은 현재 모킹(mock)되어 있으며,
수신된 주문은 출력만 된다. Kafka에서 레코드를 가져오면, 이는
데이터베이스(PostgreSQL)에 저장된다.

[회계 서비스](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/accounting/)

## 자동 계측 {#auto-instrumentation}

이 서비스는 오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic
Instrumentation)에 의존하여 Kafka와 같은 라이브러리를 자동으로 계측하고
오픈텔레메트리 SDK를 구성한다. 이 계측은 Nuget 패키지
[OpenTelemetry.AutoInstrumentation](https://www.nuget.org/packages/OpenTelemetry.AutoInstrumentation)을
통해 추가되며, `instrument.sh`에서 가져온 환경 변수를 사용해 활성화된다. 이러한
설치 방식을 사용하면 모든 계측 의존성이 애플리케이션과 올바르게 맞춰지는 것도
보장된다.

## 게시 {#publishing}

적절한 네이티브 런타임 구성 요소를 배포하려면 `dotnet publish` 명령에
`--use-current-runtime`을 추가한다.

```sh
dotnet publish "./AccountingService.csproj" --use-current-runtime -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false
```
