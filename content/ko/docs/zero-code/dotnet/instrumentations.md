---
title: 사용 가능한 계측
linkTitle: 계측
description:
  오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic Instrumentation)이
  지원하는 라이브러리.
weight: 10
# prettier-ignore
cSpell:ignore: ADONET ASPNET ASPNETCORE Bootstrapper DBSTATEMENT ELASTICTRANSPORT ENTITYFRAMEWORKCORE GRPCNETCLIENT HOSTINGSTARTUPASSEMBLIES HTTPCLIENT ILOGGER ILREWRITE MASSTRANSIT MYSQLCONNECTOR MYSQLDATA NETRUNTIME netstandard NLOG npgsql NSERVICEBUS ORACLEMDA RABBITMQ SQLCLIENT SQLITE STACKEXCHANGEREDIS WCFCLIENT WCFCORE WCFSERVICE
default_lang_commit: 69edfe0c981cb3559854006de46065bde48b07dc
---

오픈텔레메트리(OpenTelemetry) .NET 자동 계측(Automatic Instrumentation)은 다양한
라이브러리를 지원한다.

## 계측 {#instrumentations}

모든 계측은 모든 시그널 유형(트레이스, 메트릭, 로그)에 대해 기본적으로
활성화된다.

`OTEL_DOTNET_AUTO_{SIGNAL}_INSTRUMENTATION_ENABLED` 환경 변수를 `false`로
설정하면 특정 시그널 유형의 모든 계측을 비활성화할 수 있다.

더 세밀하게 제어하려면 `OTEL_DOTNET_AUTO_{SIGNAL}_{0}_INSTRUMENTATION_ENABLED`
환경 변수를 `false`로 설정하여 특정 시그널 유형의 특정 계측을 비활성화할 수
있다. 여기서 `{SIGNAL}`은 시그널 유형(예: `TRACES`)이고, `{0}`은 대소문자를
구분하는 계측 이름이다.

| 환경 변수                                              | 설명                                                                                                                                                           | 기본값                                                                  | 상태                                                              |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`             | 모든 계측을 활성화한다.                                                                                                                                        | `true`                                                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_INSTRUMENTATION_ENABLED`      | 모든 트레이스 계측을 활성화한다. `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`를 재정의한다.                                                                      | `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_{0}_INSTRUMENTATION_ENABLED`  | 특정 트레이스 계측을 활성화하기 위한 구성 패턴이며, `{0}`은 활성화하려는 계측의 대문자 ID이다. `OTEL_DOTNET_AUTO_TRACES_INSTRUMENTATION_ENABLED`를 재정의한다. | `OTEL_DOTNET_AUTO_TRACES_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_ENABLED`     | 모든 메트릭 계측을 비활성화한다. `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`를 재정의한다.                                                                      | `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_METRICS_{0}_INSTRUMENTATION_ENABLED` | 특정 메트릭 계측을 활성화하기 위한 구성 패턴이며, `{0}`은 활성화하려는 계측의 대문자 ID이다. `OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_ENABLED`를 재정의한다.  | `OTEL_DOTNET_AUTO_METRICS_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_LOGS_INSTRUMENTATION_ENABLED`        | 모든 로그 계측을 비활성화한다. `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`를 재정의한다.                                                                        | `OTEL_DOTNET_AUTO_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_LOGS_{0}_INSTRUMENTATION_ENABLED`    | 특정 로그 계측을 활성화하기 위한 구성 패턴이며, `{0}`은 활성화하려는 계측의 대문자 ID이다. `OTEL_DOTNET_AUTO_LOGS_INSTRUMENTATION_ENABLED`를 재정의한다.       | `OTEL_DOTNET_AUTO_LOGS_INSTRUMENTATION_ENABLED`의 현재 값에서 상속됨    | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

## 트레이스 계측 {#traces-instrumentations}

**상태**: [혼합(Mixed)](/docs/specs/otel/versioning-and-stability). 트레이스는
안정적이지만, 안정적인 시맨틱 컨벤션이 없어 특정 계측 라이브러리는
실험적(Experimental) 상태이다.

| ID                    | 계측된 라이브러리                                                                                                                                                                                                   | 지원 버전              | 계측 유형                  | 상태                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | -------------------------- | ----------------------------------------------------------------- |
| `ADONET`              | ADO.NET \[1\]                                                                                                                                                                                                       | \[2\]                  | 바이트코드                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ASPNET`              | ASP.NET (.NET Framework) MVC / WebApi \[3\] **.NET에서 지원되지 않음**                                                                                                                                              | \* \[4\]               | 소스 및 바이트코드         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ASPNETCORE`          | ASP.NET Core **.NET Framework에서 지원되지 않음**                                                                                                                                                                   | \*                     | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `AZURE`               | [Azure SDK](https://azure.github.io/azure-sdk/releases/latest/index.html)                                                                                                                                           | \[5\]                  | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ELASTICSEARCH`       | [Elastic.Clients.Elasticsearch](https://www.nuget.org/packages/Elastic.Clients.Elasticsearch)                                                                                                                       | \* \[6\]               | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ELASTICTRANSPORT`    | [Elastic.Transport](https://www.nuget.org/packages/Elastic.Transport)                                                                                                                                               | ≥0.4.16                | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ENTITYFRAMEWORKCORE` | [Microsoft.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore) **.NET Framework에서 지원되지 않음**                                                                                  | ≥6.0.12                | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `GRAPHQL`             | [GraphQL](https://www.nuget.org/packages/GraphQL) **.NET Framework에서 지원되지 않음**                                                                                                                              | ≥7.5.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `GRPCNETCLIENT`       | [Grpc.Net.Client](https://www.nuget.org/packages/Grpc.Net.Client)                                                                                                                                                   | ≥2.52.0 & < 3.0.0      | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `HTTPCLIENT`          | [System.Net.Http.HttpClient](https://docs.microsoft.com/dotnet/api/system.net.http.httpclient) and [System.Net.HttpWebRequest](https://docs.microsoft.com/dotnet/api/system.net.httpwebrequest)                     | \*                     | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `KAFKA`               | [Confluent.Kafka](https://www.nuget.org/packages/Confluent.Kafka)                                                                                                                                                   | ≥1.4.0 & < 3.0.0 \[7\] | 바이트코드                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `MASSTRANSIT`         | [MassTransit](https://www.nuget.org/packages/MassTransit) **.NET Framework에서 지원되지 않음**                                                                                                                      | ≥8.0.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `MONGODB`             | [MongoDB.Driver.Core](https://www.nuget.org/packages/MongoDB.Driver.Core) / [MongoDB.Driver](https://www.nuget.org/packages/MongoDB.Driver)                                                                         | ≥2.7.0                 | 소스 또는 바이트코드 \[8\] | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `MYSQLCONNECTOR`      | [MySqlConnector](https://www.nuget.org/packages/MySqlConnector)                                                                                                                                                     | ≥2.0.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `MYSQLDATA`           | [MySql.Data](https://www.nuget.org/packages/MySql.Data) **.NET Framework에서 지원되지 않음**                                                                                                                        | ≥8.1.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NPGSQL`              | [Npgsql](https://www.nuget.org/packages/Npgsql)                                                                                                                                                                     | ≥6.0.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NSERVICEBUS`         | [NServiceBus](https://www.nuget.org/packages/NServiceBus)                                                                                                                                                           | ≥8.0.0 & < 10.0.0      | 소스 및 바이트코드         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ORACLEMDA`           | [Oracle.ManagedDataAccess.Core](https://www.nuget.org/packages/Oracle.ManagedDataAccess.Core) and [Oracle.ManagedDataAccess](https://www.nuget.org/packages/Oracle.ManagedDataAccess) **ARM64에서 지원되지 않음**   | ≥23.4.0                | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `RABBITMQ`            | [RabbitMQ.Client](https://www.nuget.org/packages/RabbitMQ.Client/)                                                                                                                                                  | ≥5.0.0                 | 소스 또는 바이트코드 \[9\] | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `QUARTZ`              | [Quartz](https://www.nuget.org/packages/Quartz) **.NET Framework 4.7.1 및 이전 버전에서 지원되지 않음**                                                                                                             | ≥3.4.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `SQLCLIENT`           | [Microsoft.Data.SqlClient](https://www.nuget.org/packages/Microsoft.Data.SqlClient), [System.Data.SqlClient](https://www.nuget.org/packages/System.Data.SqlClient) \[10\] and `System.Data` (.NET Framework에 포함) | \* \[11\]              | 소스 및 바이트코드 \[12\]  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `SQLITE`              | [Microsoft.Data.Sqlite](https://www.nuget.org/packages/Microsoft.Data.Sqlite)                                                                                                                                       | ≥8.0.0 & < 12.0.0      | 바이트코드                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `STACKEXCHANGEREDIS`  | [StackExchange.Redis](https://www.nuget.org/packages/StackExchange.Redis)                                                                                                                                           | \[13\]                 | 소스 및 바이트코드         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `WCFCLIENT`           | WCF                                                                                                                                                                                                                 | \*                     | 소스 및 바이트코드         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `WCFCORE`             | [CoreWCF.Primitives](https://www.nuget.org/packages/CoreWCF.Primitives) **.NET Framework에서 지원되지 않음**                                                                                                        | ≥1.8.0                 | 소스                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `WCFSERVICE`          | WCF **.NET에서 지원되지 않음**.                                                                                                                                                                                     | \*                     | 소스 및 바이트코드         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

\[1\]: `DbCommand` 메서드를 다루는 일반 ADO.NET 계측이다. 재정의된(overridden)
메서드의 구현만 다룬다. 일반적으로 많은 구현이 C#의 `new` 키워드를 사용하는데,
이런 경우에는 별도의 구현이 필요하다. 이 계측은 `SQLCLIENT`, `NPGSQL`,
`MYSQLDATA`, `MYSQLCONNECTOR`, `ORACLEMDA`, `SQLITE` 같은 특정 계측이 다루는
라이브러리는 다루지 않는다.

\[2\]: `System.Data.Common`(≥4.0.0 및 <12.0.0), .NET Framework용
`System.Data`(≥2.0.0 및 <5.0.0), `netstandard`(≥2.0.0 및 <3.0.0)를 지원한다.

\[3\]: 통합 파이프라인(integrated pipeline) 모드만 지원된다.

\[4\]: `ASP.NET (.NET Framework) MVC / WebApi`는 ARM64에서 지원되지 않는다.

\[5\]: 2021년 10월 1일 이후에 릴리스된 `Azure.` 접두사 패키지.

\[6\]: `Elastic.Clients.Elasticsearch` 버전 ≥8.0.0 및 <8.10.0. 버전 ≥8.10.0은
`Elastic.Transport` 계측이 지원한다.

\[7\]: `Confluent.Kafka`는 Windows와 Linux의 ARM64에서는 버전 ≥1.8.2부터,
macOS에서는 ≥1.9.2부터 지원된다.

\[8\]: `MongoDB.Driver`는 `3.7.0` 미만 버전에서만 바이트코드 계측이 필요하다.
`3.7.0+`는 소스 계측만 사용한다.

\[9\]: `RabbitMq.Client`는 `5.*`와 `6.*` 버전에서만 바이트코드 계측이 필요하며,
`7.0.0+`는 소스 계측만 사용한다.

\[10\]: `System.Data.SqlClient`는
[지원 중단](https://www.nuget.org/packages/System.Data.SqlClient/4.9.0#readme-body-tab)되었다.

\[11\]: `Microsoft.Data.SqlClient` v3.\*는
[이슈](https://github.com/open-telemetry/opentelemetry-dotnet/issues/4243)
때문에 .NET Framework에서 지원되지 않는다. `System.Data.SqlClient`는 버전
4.8.5부터 지원된다.

\[12\]: 바이트코드 계측은 컨텍스트 전파에만 사용되며,
`OTEL_DOTNET_EXPERIMENTAL_SQLCLIENT_ENABLE_TRACE_CONTEXT_PROPAGATION`으로 구성할
수 있다.

\[13\]: `StackExchange.Redis` 버전 ≥2.6.122 및 <4.0.0은 .NET에서 지원되고, 버전
≥2.6.122 및 <3.0.0은 .NET Framework에서 지원된다.

## 메트릭 계측 {#metrics-instrumentations}

**상태**: [혼합(Mixed)](/docs/specs/otel/versioning-and-stability). 메트릭은
안정적이지만, 안정적인 시맨틱 컨벤션이 없어 특정 계측은 실험적(Experimental)
상태이다.

| ID            | 계측된 라이브러리                                                                                                                                                                                                  | 문서                                                                                                                                                                                                          | 지원 버전         | 계측 유형          | 상태                                                              |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- | ------------------ | ----------------------------------------------------------------- |
| `ASPNET`      | ASP.NET Framework \[1\] **.NET에서 지원되지 않음**                                                                                                                                                                 | [ASP.NET metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Instrumentation.AspNet-1.16.0/src/OpenTelemetry.Instrumentation.AspNet/README.md#list-of-metrics-produced)              | \*                | 소스 및 바이트코드 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `ASPNETCORE`  | ASP.NET Core **.NET Framework에서 지원되지 않음**                                                                                                                                                                  | [ASP.NET Core metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Instrumentation.AspNetCore-1.16.0/src/OpenTelemetry.Instrumentation.AspNetCore/README.md#list-of-metrics-produced) | \*                | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `HTTPCLIENT`  | [System.Net.Http.HttpClient](https://docs.microsoft.com/dotnet/api/system.net.http.httpclient) and [System.Net.HttpWebRequest](https://docs.microsoft.com/dotnet/api/system.net.httpwebrequest)                    | [HttpClient metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Instrumentation.Http-1.16.0/src/OpenTelemetry.Instrumentation.Http/README.md#list-of-metrics-produced)               | \*                | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NETRUNTIME`  | [OpenTelemetry.Instrumentation.Runtime](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.Runtime)                                                                                                      | [Runtime metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Instrumentation.Runtime-1.15.1/src/OpenTelemetry.Instrumentation.Runtime/README.md#metrics)                             | \*                | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NPGSQL`      | [Npgsql](https://www.nuget.org/packages/Npgsql) **.NET Framework에서 지원되지 않음**                                                                                                                               | [Npgsql metrics](https://www.npgsql.org/doc/diagnostics/metrics.html)                                                                                                                                         | ≥6.0.0            | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NSERVICEBUS` | [NServiceBus](https://www.nuget.org/packages/NServiceBus)                                                                                                                                                          | [NServiceBus metrics](https://docs.particular.net/samples/open-telemetry/prometheus-grafana/#reporting-metric-values)                                                                                         | ≥8.0.0 & < 10.0.0 | 소스 및 바이트코드 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `PROCESS`     | [OpenTelemetry.Instrumentation.Process](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.Process)                                                                                                      | [Process metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Instrumentation.Process-1.16.0-beta.1/src/OpenTelemetry.Instrumentation.Process/README.md#metrics)                      | \*                | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `SQLCLIENT`   | [Microsoft.Data.SqlClient](https://www.nuget.org/packages/Microsoft.Data.SqlClient), [System.Data.SqlClient](https://www.nuget.org/packages/System.Data.SqlClient) \[2\] and `System.Data` (.NET Framework에 포함) | [SqlClient metrics](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/releases/tag/Instrumentation.SqlClient-1.16.0)                                                                             | \* \[3\]          | 소스               | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

\[1\]: ASP.NET 메트릭은 `AspNet` 트레이스 계측도 활성화된 경우에만 생성된다.

\[2\]: `System.Data.SqlClient`는
[지원 중단](https://www.nuget.org/packages/System.Data.SqlClient/4.9.0#readme-body-tab)되었다.

\[3\]: `Microsoft.Data.SqlClient` v3.\*는
[이슈](https://github.com/open-telemetry/opentelemetry-dotnet/issues/4243)
때문에 .NET Framework에서 지원되지 않는다. `System.Data.SqlClient`는 버전
4.8.5부터 지원된다.

## 로그 계측 {#logs-instrumentations}

**상태**: [실험적(Experimental)](/docs/specs/otel/versioning-and-stability).

| ID        | 계측된 라이브러리                                                                                                                | 지원 버전          | 계측 유형                  | 상태                                                              |
| --------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------ | -------------------------- | ----------------------------------------------------------------- |
| `ILOGGER` | [Microsoft.Extensions.Logging](https://www.nuget.org/packages/Microsoft.Extensions.Logging) **.NET Framework에서 지원되지 않음** | ≥8.0.0 && < 12.0.0 | 바이트코드 또는 소스 \[1\] | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `LOG4NET` | [log4net](https://www.nuget.org/packages/log4net) \[2\]                                                                          | ≥2.0.13 && < 4.0.0 | 바이트코드                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `NLOG`    | [NLog](https://www.nuget.org/packages/NLog) \[2\]                                                                                | ≥5.0.0 && < 7.0.0  | 바이트코드                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

\[1\]: ASP.NET Core 애플리케이션의 경우, `ASPNETCORE_HOSTINGSTARTUPASSEMBLIES`
환경 변수를 `OpenTelemetry.AutoInstrumentation.AspNetCoreBootstrapper`로
설정하면 .NET CLR 프로파일러를 사용하지 않고도 `LoggingBuilder` 계측을 활성화할
수 있다.

\[2\]: 이 계측은 트레이스 컨텍스트 주입과 로그 브리지(logs bridge)를 모두
제공한다.

### 계측 옵션 {#instrumentation-options}

| 환경 변수                                                                         | 설명                                                                                                                                                                                                                                                               | 기본값  | 상태                                                              |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_ENTITYFRAMEWORKCORE_SET_DBSTATEMENT_FOR_TEXT`                   | Entity Framework Core 계측이 `db.statement` 속성을 통해 SQL 문을 전달할 수 있는지 여부. 쿼리에는 민감한 정보가 포함될 수 있다. `false`로 설정하면 `db.statement`는 저장 프로시저를 실행할 때만 기록된다.                                                           | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_GRAPHQL_SET_DOCUMENT`                                           | GraphQL 계측이 `graphql.document` 속성을 통해 원시 쿼리를 전달할 수 있는지 여부. 쿼리에는 민감한 정보가 포함될 수 있다.                                                                                                                                            | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_ORACLEMDA_DATABASE_OPENTELEMETRY_TRACING`                       | Oracle Client 계측이 데이터베이스 오픈텔레메트리 트레이싱을 활성화하고 서버로 컨텍스트를 전파하는지 여부. \[1\]                                                                                                                                                    | `true`  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_ORACLEMDA_SET_DBSTATEMENT_FOR_TEXT`                             | Oracle Client 계측이 `db.statement` 속성을 통해 SQL 문을 전달할 수 있는지 여부. 쿼리에는 민감한 정보가 포함될 수 있다. `false`로 설정하면 `db.statement`는 저장 프로시저를 실행할 때만 기록된다.                                                                   | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_SQLCLIENT_SET_DBSTATEMENT_FOR_TEXT`                             | SQL Client 계측이 `db.statement` 속성을 통해 SQL 문을 전달할 수 있는지 여부. 쿼리에는 민감한 정보가 포함될 수 있다. `false`로 설정하면 `db.statement`는 저장 프로시저를 실행할 때만 기록된다. **System.Data.SqlClient의 경우 .NET Framework에서 지원되지 않는다.** | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_ASPNET_INSTRUMENTATION_CAPTURE_REQUEST_HEADERS`          | 쉼표로 구분된 HTTP 헤더 이름 목록. ASP.NET 계측은 구성된 모든 헤더 이름에 대해 HTTP 요청 헤더 값을 캡처한다.                                                                                                                                                       |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_ASPNET_INSTRUMENTATION_CAPTURE_RESPONSE_HEADERS`         | 쉼표로 구분된 HTTP 헤더 이름 목록. ASP.NET 계측은 구성된 모든 헤더 이름에 대해 HTTP 응답 헤더 값을 캡처한다. **IIS Classic 모드에서는 지원되지 않는다.**                                                                                                           |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_ASPNETCORE_INSTRUMENTATION_CAPTURE_REQUEST_HEADERS`      | 쉼표로 구분된 HTTP 헤더 이름 목록. ASP.NET Core 계측은 구성된 모든 헤더 이름에 대해 HTTP 요청 헤더 값을 캡처한다.                                                                                                                                                  |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_ASPNETCORE_INSTRUMENTATION_CAPTURE_RESPONSE_HEADERS`     | 쉼표로 구분된 HTTP 헤더 이름 목록. ASP.NET Core 계측은 구성된 모든 헤더 이름에 대해 HTTP 응답 헤더 값을 캡처한다.                                                                                                                                                  |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_GRPCNETCLIENT_INSTRUMENTATION_CAPTURE_REQUEST_METADATA`  | 쉼표로 구분된 gRPC 메타데이터 이름 목록. Grpc.Net.Client 계측은 구성된 모든 메타데이터 이름에 대해 gRPC 요청 메타데이터 값을 캡처한다.                                                                                                                             |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_GRPCNETCLIENT_INSTRUMENTATION_CAPTURE_RESPONSE_METADATA` | 쉼표로 구분된 gRPC 메타데이터 이름 목록. Grpc.Net.Client 계측은 구성된 모든 메타데이터 이름에 대해 gRPC 응답 메타데이터 값을 캡처한다.                                                                                                                             |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_HTTP_INSTRUMENTATION_CAPTURE_REQUEST_HEADERS`            | 쉼표로 구분된 HTTP 헤더 이름 목록. HTTP Client 계측은 구성된 모든 헤더 이름에 대해 HTTP 요청 헤더 값을 캡처한다.                                                                                                                                                   |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_TRACES_HTTP_INSTRUMENTATION_CAPTURE_RESPONSE_HEADERS`           | 쉼표로 구분된 HTTP 헤더 이름 목록. HTTP Client 계측은 구성된 모든 헤더 이름에 대해 HTTP 응답 헤더 값을 캡처한다.                                                                                                                                                   |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_EXPERIMENTAL_ASPNETCORE_DISABLE_URL_QUERY_REDACTION`                 | ASP.NET Core 계측이 `url.query` 속성 값의 삭제(redaction)를 끄는지 여부.                                                                                                                                                                                           | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_EXPERIMENTAL_HTTPCLIENT_DISABLE_URL_QUERY_REDACTION`                 | HTTP 클라이언트 계측이 `url.full` 속성 값의 삭제(redaction)를 끄는지 여부.                                                                                                                                                                                         | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_EXPERIMENTAL_ASPNET_DISABLE_URL_QUERY_REDACTION`                     | ASP.NET 계측이 `url.query` 속성 값의 삭제(redaction)를 끄는지 여부.                                                                                                                                                                                                | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_EXPERIMENTAL_SQLCLIENT_ENABLE_TRACE_CONTEXT_PROPAGATION`             | .NET Framework 전용. `SQLCLIENT` 계측이 SQL Server로의 컨텍스트 전파를 켜는지 여부.                                                                                                                                                                                | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_SQLCLIENT_NETFX_ILREWRITE_ENABLED`                              | \[2\]                                                                                                                                                                                                                                                              | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

\[1\]: 이 기능은 `23.26.200` 이상 패키지에서만 지원된다. 또한 Oracle AI Database
26ai 서버가 필요하다.

\[2\]: .NET Framework에서 `SqlCommand`의 IL 재작성(rewriting)을 활성화하여
`SqlClient` 계측에 `CommandText`가 존재하도록 보장한다. 이는 `db.query.text`와
`db.query.summary`가 채워지는 데 필요하다. 이전에는 `CommandText`가 저장
프로시저에서만 사용 가능했다. 이 설정을 활성화하면 원시 쿼리에서도 사용할 수
있다. 이는
[`SqlEventSource`](https://github.com/dotnet/SqlClient/blob/v1.0.19239.1/src/Microsoft.Data.SqlClient/netfx/src/Microsoft/Data/SqlClient/SqlCommand.cs#L6369)가
발생시키는 이벤트의 동작을 변경하므로, 이 메커니즘을 사용하는 경우
애플리케이션의 다른 부분에 영향을 줄 수 있다.
