---
title: 구성 및 설정
linkTitle: 구성
aliases: [/docs/languages/net/automatic/config]
weight: 20
default_lang_commit: 69edfe0c981cb3559854006de46065bde48b07dc
# prettier-ignore
cSpell:ignore: AZUREAPPSERVICE Bitness CLSID CORECLR dylib ILREWRITE LOGRECORD NETFX OPERATINGSYSTEM PROCESSRUNTIME SQLCLIENT UNHANDLEDEXCEPTION
---

## 구성 방법 {#configuration-methods}

다음 방법으로 구성 설정을 적용하거나 편집할 수 있으며, 환경 변수가
`App.config`나 `Web.config` 파일보다 우선한다.

1. 환경 변수

   환경 변수는 설정을 구성하는 주요 방법이다.

2. `App.config` 또는 `Web.config` 파일

   .NET Framework에서 실행되는 애플리케이션의 경우, 웹 구성
   파일(`web.config`)이나 애플리케이션 구성 파일(`app.config`)을 사용하여
   `OTEL_*` 설정을 구성할 수 있다.

   ⚠️ `App.config`나 `Web.config`로는 `OTEL_`로 시작하는 설정만 설정할 수 있다.
   다만, 다음 설정은 지원되지 않는다.
   - `OTEL_DOTNET_AUTO_APP_DOMAIN_STRATEGY`
   - `OTEL_DOTNET_AUTO_HOME`
   - `OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES`
   - `OTEL_DOTNET_AUTO_FAIL_FAST_ENABLED`
   - `OTEL_DOTNET_AUTO_[TRACES|METRICS|LOGS]_INSTRUMENTATION_ENABLED`
   - `OTEL_DOTNET_AUTO_[TRACES|METRICS|LOGS]_{INSTRUMENTATION_ID}_INSTRUMENTATION_ENABLED`
   - `OTEL_DOTNET_AUTO_LOG_DIRECTORY`
   - `OTEL_LOG_LEVEL`
   - `OTEL_DOTNET_AUTO_NETFX_REDIRECT_ENABLED` (지원 중단됨)
   - `OTEL_DOTNET_AUTO_REDIRECT_ENABLED`
   - `OTEL_DOTNET_AUTO_SQLCLIENT_NETFX_ILREWRITE_ENABLED`

   `OTEL_SERVICE_NAME` 설정 예시:

   ```xml
   <configuration>
   <appSettings>
       <add key="OTEL_SERVICE_NAME" value="my-service-name" />
   </appSettings>
   </configuration>
   ```

   > [!NOTE]
   >
   > .NET Framework에서는 `Web.config`나 `App.config`의 `OTEL_*` 값이 시작
   > 시점에 프로세스 수준 환경 변수로 승격되며, OTel SDK는 프로세스당 한 번만
   > 초기화된다. 여러 애플리케이션이 하나의 워커 프로세스(애플리케이션 풀)를
   > 공유할 수 있는 IIS에서는, 가장 먼저 시작되는 애플리케이션이 해당 풀의 모든
   > 애플리케이션에 대한 구성을 결정한다.

3. 서비스 이름 자동 감지

   서비스 이름이 명시적으로 구성되지 않으면 자동으로 생성된다. 이는 일부
   상황에서 유용할 수 있다.
   - .NET Framework에서 애플리케이션이 IIS에 호스팅되는 경우
     `SiteName\VirtualPath` 형식(예: `MySite\MyApp`)이 사용된다.
   - 그렇지 않은 경우 애플리케이션
     [엔트리 어셈블리(entry Assembly)](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.assembly.getentryassembly?view=net-7.0)의
     이름이 사용된다.

기본적으로는 구성에 환경 변수를 사용하는 것을 권장한다. 다만, 해당 설정이
지원한다면 다음과 같이 한다.

- ASP.NET 애플리케이션(.NET Framework)을 구성할 때는 `Web.config`를 사용한다.
- Windows 서비스(.NET Framework)를 구성할 때는 `App.config`를 사용한다.

## 전역 설정 {#global-settings}

| 환경 변수                            | 설명                                                                                                                                                                                                   | 기본값  | 상태                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_HOME`              | 설치 위치.                                                                                                                                                                                             |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES` | 프로파일러가 계측할 수 없는 실행 파일의 이름. 쉼표로 구분된 여러 값을 지원한다. 예: `ReservedProcess.exe,powershell.exe`. 설정하지 않으면 프로파일러는 기본적으로 모든 프로세스에 연결된다. \[1\]\[2\] |         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_FAIL_FAST_ENABLED` | 자동 계측을 실행할 수 없을 때 프로세스가 실패할 수 있도록 한다. 디버깅 목적으로 설계되었다. 운영 환경에서는 사용하면 안 된다. \[1\]                                                                    | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

\[1\] `OTEL_DOTNET_AUTO_FAIL_FAST_ENABLED`가 `true`로 설정되면,
`OTEL_DOTNET_AUTO_EXCLUDE_PROCESSES`에 의해 계측에서 제외된 프로세스는 조용히
계속 실행되는 대신 실패한다.

\[2\] `dotnet MyApp.dll`로 실행된 애플리케이션의 프로세스 이름은 `dotnet` 또는
`dotnet.exe`라는 점에 유의한다.

## 리소스 {#resources}

리소스는 텔레메트리를 생성하는 엔티티(entity)의 변경 불가능한 표현이다. 자세한
내용은 [리소스 시맨틱 컨벤션](/docs/specs/semconv/resource/)을 참고한다.

### 리소스 속성 {#resource-attributes}

| 환경 변수                  | 설명                                                                                                                                                                             | 기본값                                                                                                                              | 상태                                                      |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `OTEL_RESOURCE_ATTRIBUTES` | 리소스 속성으로 사용할 키-값 쌍. 자세한 내용은 [Resource SDK](/docs/specs/otel/resource/sdk#specifying-resource-information-via-an-environment-variable)를 참고한다.             | 자세한 내용은 [리소스 시맨틱 컨벤션](/docs/specs/semconv/resource/#semantic-attributes-with-sdk-provided-default-value)을 참고한다. | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_SERVICE_NAME`        | [`service.name`](/docs/specs/semconv/resource/#service) 리소스 속성의 값을 설정한다. `OTEL_RESOURCE_ATTRIBUTES`에 `service.name`이 제공되면 `OTEL_SERVICE_NAME`의 값이 우선한다. | 구성 방법 섹션의 [서비스 이름 자동 감지](#configuration-methods)를 참고한다.                                                        | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

### 리소스 탐지기 {#resource-detectors}

| 환경 변수                                        | 설명                                                                                                                                                               | 기본값 | 상태                                                              |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_RESOURCE_DETECTOR_ENABLED`     | 모든 리소스 탐지기를 활성화한다.                                                                                                                                   | `true` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_{0}_RESOURCE_DETECTOR_ENABLED` | 특정 리소스 탐지기를 활성화하기 위한 구성 패턴이며, `{0}`은 활성화하려는 리소스 탐지기의 대문자 ID이다. `OTEL_DOTNET_AUTO_RESOURCE_DETECTOR_ENABLED`를 재정의한다. | `true` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

다음 리소스 탐지기가 포함되어 있으며 기본적으로 활성화된다.

| ID                | 설명                     | 문서                                                                                                                                                                                                                                    | 상태                                                              |
| ----------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `AZUREAPPSERVICE` | Azure App Service 탐지기 | [Azure resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.Azure-1.15.1-beta.1/src/OpenTelemetry.Resources.Azure/README.md)                                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `CONTAINER`       | 컨테이너 탐지기          | [Container resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.Container-1.15.1-beta.1/src/OpenTelemetry.Resources.Container/README.md) **.NET Framework에서 지원되지 않음** | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `HOST`            | 호스트 탐지기            | [Host resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.Host-1.15.1-beta.1/src/OpenTelemetry.Resources.Host/README.md)                                                     | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OPERATINGSYSTEM` | 운영체제 탐지기          | [Operating System resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.OperatingSystem-1.15.1-beta.1/src/OpenTelemetry.Resources.OperatingSystem/README.md)                   | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `PROCESS`         | 프로세스 탐지기          | [Process resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.Process-1.15.1-beta.2/src/OpenTelemetry.Resources.Process/README.md)                                            | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `PROCESSRUNTIME`  | 프로세스 런타임 탐지기   | [Process Runtime resource detector documentation](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/Resources.ProcessRuntime-1.15.1-beta.1/src/OpenTelemetry.Resources.ProcessRuntime/README.md)                      | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

## 전파자 {#propagators}

전파자(Propagator)를 사용하면 애플리케이션이 컨텍스트를 공유할 수 있다. 자세한
내용은
[오픈텔레메트리(OpenTelemetry) 명세](/docs/specs/otel/context/api-propagators)를
참고한다.

| 환경 변수          | 설명                                                                                                                                                                                                                                                                               | 기본값                 |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `OTEL_PROPAGATORS` | 쉼표로 구분된 전파자 목록. 지원 옵션: `tracecontext`, `baggage`, `b3multi`, `b3`. 자세한 내용은 [오픈텔레메트리 명세](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.14.0/specification/sdk-environment-variables.md#general-sdk-configuration)를 참고한다. | `tracecontext,baggage` |

## 샘플러 {#samplers}

샘플러(Sampler)를 사용하면 수집하고 내보낼 트레이스를 선택하여, 오픈텔레메트리
계측이 유발할 수 있는 노이즈와 오버헤드를 제어할 수 있다. 자세한 내용은
[오픈텔레메트리 명세](/docs/specs/otel/configuration/sdk-environment-variables/#general-sdk-configuration)를
참고한다.

| 환경 변수                 | 설명                                 | 기본값                  | 상태                                                      |
| ------------------------- | ------------------------------------ | ----------------------- | --------------------------------------------------------- |
| `OTEL_TRACES_SAMPLER`     | 트레이스에 사용할 샘플러 \[1\]       | `parentbased_always_on` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_TRACES_SAMPLER_ARG` | 샘플러 인수로 사용할 문자열 값 \[2\] |                         | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

\[1\]: 지원되는 값은 다음과 같다.

- `always_on`,
- `always_off`,
- `traceidratio`,
- `parentbased_always_on`,
- `parentbased_always_off`,
- `parentbased_traceidratio`.

\[2\]: `traceidratio`와 `parentbased_traceidratio` 샘플러의 경우: 샘플링
확률이며, [0..1] 범위의 숫자이다(예: "0.25"). 기본값은 1.0이다.

## 익스포터 {#exporters}

익스포터는 텔레메트리를 출력한다.

| 환경 변수               | 설명                                                                             | 기본값 | 상태                                                      |
| ----------------------- | -------------------------------------------------------------------------------- | ------ | --------------------------------------------------------- |
| `OTEL_TRACES_EXPORTER`  | 쉼표로 구분된 익스포터 목록. 지원 옵션: `otlp`, `zipkin` [1], `console`, `none`. | `otlp` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_METRICS_EXPORTER` | 쉼표로 구분된 익스포터 목록. 지원 옵션: `otlp`, `prometheus`, `console`, `none`. | `otlp` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_LOGS_EXPORTER`    | 쉼표로 구분된 익스포터 목록. 지원 옵션: `otlp`, `console`, `none`.               | `otlp` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

**[1]**: `zipkin`은 지원 중단되었으며 다음 릴리스에서 제거될 예정이다.

### 트레이스 익스포터 {#traces-exporter}

| 환경 변수                        | 설명                                                     | 기본값  | 상태                                                      |
| -------------------------------- | -------------------------------------------------------- | ------- | --------------------------------------------------------- |
| `OTEL_BSP_SCHEDULE_DELAY`        | 연속된 두 번의 내보내기 사이의 지연 간격(밀리초).        | `5000`  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_BSP_EXPORT_TIMEOUT`        | 데이터를 내보내는 데 허용되는 최대 시간(밀리초)          | `30000` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_BSP_MAX_QUEUE_SIZE`        | 최대 큐 크기.                                            | `2048`  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | 최대 배치 크기. `OTEL_BSP_MAX_QUEUE_SIZE` 이하여야 한다. | `512`   | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

### 메트릭 익스포터 {#metrics-exporter}

| 환경 변수                     | 설명                                                 | 기본값                                           | 상태                                                      |
| ----------------------------- | ---------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| `OTEL_METRIC_EXPORT_INTERVAL` | 두 번의 내보내기 시도 시작 사이의 시간 간격(밀리초). | OTLP 익스포터는 `60000`, 콘솔 익스포터는 `10000` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_METRIC_EXPORT_TIMEOUT`  | 데이터를 내보내는 데 허용되는 최대 시간(밀리초).     | OTLP 익스포터는 `30000`, 콘솔 익스포터는 없음    | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

### 로그 익스포터 {#logs-exporter}

| 환경 변수                                         | 설명                                  | 기본값  | 상태                                                              |
| ------------------------------------------------- | ------------------------------------- | ------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_LOGS_INCLUDE_FORMATTED_MESSAGE` | 형식화된 로그 메시지를 설정할지 여부. | `false` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

### OTLP {#otlp}

**상태**: [안정(Stable)](/docs/specs/otel/versioning-and-stability)

OTLP 익스포터를 활성화하려면
`OTEL_TRACES_EXPORTER`/`OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER` 환경 변수를
`otlp`로 설정한다.

환경 변수로 OTLP 익스포터를 커스터마이즈하려면
[OTLP 익스포터 문서](https://github.com/open-telemetry/opentelemetry-dotnet/tree/core-1.16.0/src/OpenTelemetry.Exporter.OpenTelemetryProtocol#environment-variables)를
참고한다. 주요 환경 변수는 다음과 같다.

| 환경 변수                                           | 설명                                                                                                                                                                                   | 기본값                                                                               | 상태                                                      |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                       | OTLP 익스포터의 대상 엔드포인트. 자세한 내용은 [오픈텔레메트리 명세](/docs/specs/otel/protocol/exporter/)를 참고한다.                                                                  | `http/protobuf`: `http://localhost:4318`, `grpc`: `http://localhost:4317`            | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`                | `OTEL_EXPORTER_OTLP_ENDPOINT`와 동일하지만 트레이스에만 적용된다.                                                                                                                      | `http/protobuf`: `http://localhost:4318/v1/traces`, `grpc`: `http://localhost:4317`  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`               | `OTEL_EXPORTER_OTLP_ENDPOINT`와 동일하지만 메트릭에만 적용된다.                                                                                                                        | `http/protobuf`: `http://localhost:4318/v1/metrics`, `grpc`: `http://localhost:4317` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`                  | `OTEL_EXPORTER_OTLP_ENDPOINT`와 동일하지만 로그에만 적용된다.                                                                                                                          | `http/protobuf`: `http://localhost:4318/v1/logs`, `grpc`: `http://localhost:4317`    | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                       | OTLP 익스포터 전송 프로토콜. 지원되는 값은 `grpc`, `http/protobuf`이다. [1]                                                                                                            | `http/protobuf`                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL`                | `OTEL_EXPORTER_OTLP_PROTOCOL`과 동일하지만 트레이스에만 적용된다.                                                                                                                      | `http/protobuf`                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL`               | `OTEL_EXPORTER_OTLP_PROTOCOL`과 동일하지만 메트릭에만 적용된다.                                                                                                                        | `http/protobuf`                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL`                  | `OTEL_EXPORTER_OTLP_PROTOCOL`과 동일하지만 로그에만 적용된다.                                                                                                                          | `http/protobuf`                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TIMEOUT`                        | 백엔드가 각 배치를 처리하기까지 기다리는 최대 시간(밀리초).                                                                                                                            | `10000` (10s)                                                                        | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TRACES_TIMEOUT`                 | `OTEL_EXPORTER_OTLP_TIMEOUT`과 동일하지만 트레이스에만 적용된다.                                                                                                                       | `10000` (10s)                                                                        | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_TIMEOUT`                | `OTEL_EXPORTER_OTLP_TIMEOUT`과 동일하지만 메트릭에만 적용된다.                                                                                                                         | `10000` (10s)                                                                        | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_LOGS_TIMEOUT`                   | `OTEL_EXPORTER_OTLP_TIMEOUT`과 동일하지만 로그에만 적용된다.                                                                                                                           | `10000` (10s)                                                                        | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_COMPRESSION`                    | OTLP 익스포터의 압축 방식. 지원되는 값은 `gzip`, `none`이다.                                                                                                                           | `none`                                                                               | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TRACES_COMPRESSION`             | `OTEL_EXPORTER_OTLP_COMPRESSION`과 동일하지만 트레이스에만 적용된다.                                                                                                                   | `none`                                                                               | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_COMPRESSION`            | `OTEL_EXPORTER_OTLP_COMPRESSION`과 동일하지만 메트릭에만 적용된다.                                                                                                                     | `none`                                                                               | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_LOGS_COMPRESSION`               | `OTEL_EXPORTER_OTLP_COMPRESSION`과 동일하지만 로그에만 적용된다.                                                                                                                       | `none`                                                                               | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_HEADERS`                        | 각 내보내기와 함께 전송되는 추가 HTTP 헤더의 쉼표로 구분된 목록. 예: `Authorization=secret,X-Key=Value`.                                                                               |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS`                 | `OTEL_EXPORTER_OTLP_HEADERS`와 동일하지만 트레이스에만 적용된다.                                                                                                                       |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS`                | `OTEL_EXPORTER_OTLP_HEADERS`와 동일하지만 메트릭에만 적용된다.                                                                                                                         |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS`                   | `OTEL_EXPORTER_OTLP_HEADERS`와 동일하지만 로그에만 적용된다.                                                                                                                           |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_CERTIFICATE`                    | 서버의 TLS 인증서를 검증하는 데 사용되는 CA 인증서 파일(PEM 형식)의 경로. \[3\]                                                                                                        |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE`             | mTLS 인증을 위한 클라이언트 인증서 파일(PEM 형식)의 경로. \[3\]                                                                                                                        |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_CLIENT_KEY`                     | mTLS 인증을 위한 클라이언트 개인 키 파일(PEM 형식)의 경로. \[3\]                                                                                                                       |                                                                                      | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT`                 | 허용되는 최대 속성 값 크기.                                                                                                                                                            | 없음                                                                                 | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_ATTRIBUTE_COUNT_LIMIT`                        | 허용되는 최대 스팬 속성 개수.                                                                                                                                                          | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT`            | 허용되는 최대 속성 값 크기. [메트릭에는 적용되지 않는다.](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.15.0/specification/metrics/sdk.md#attribute-limits).   | 없음                                                                                 | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT`                   | 허용되는 최대 스팬 속성 개수. [메트릭에는 적용되지 않는다.](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.15.0/specification/metrics/sdk.md#attribute-limits). | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_SPAN_EVENT_COUNT_LIMIT`                       | 허용되는 최대 스팬 이벤트 개수.                                                                                                                                                        | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_SPAN_LINK_COUNT_LIMIT`                        | 허용되는 최대 스팬 링크 개수.                                                                                                                                                          | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EVENT_ATTRIBUTE_COUNT_LIMIT`                  | 스팬 이벤트당 허용되는 최대 속성 개수.                                                                                                                                                 | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_LINK_ATTRIBUTE_COUNT_LIMIT`                   | 스팬 링크당 허용되는 최대 속성 개수.                                                                                                                                                   | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_LOGRECORD_ATTRIBUTE_VALUE_LENGTH_LIMIT`       | 허용되는 최대 로그 레코드 속성 값 크기.                                                                                                                                                | 없음                                                                                 | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_LOGRECORD_ATTRIBUTE_COUNT_LIMIT`              | 허용되는 최대 로그 레코드 속성 개수.                                                                                                                                                   | 128                                                                                  | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | 계측기 종류에 따라 사용할 집계 템포럴리티(aggregation temporality). [2]                                                                                                                | `cumulative`                                                                         | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

**[1]**: `OTEL_EXPORTER_OTLP_PROTOCOL`에 대한 고려 사항:

- 오픈텔레메트리 .NET 자동 계측의 기본값은 `http/protobuf`이며, 이는 기본값이
  `grpc`인 오픈텔레메트리 .NET SDK와 다르다.
- .NET 8 이상에서는 `grpc` OTLP 익스포터 프로토콜을 사용하려면 애플리케이션이
  [`Grpc.Net.Client`](https://www.nuget.org/packages/Grpc.Net.Client/)를
  참조해야 한다. 예를 들어 `.csproj` 파일에
  `<PackageReference Include="Grpc.Net.Client" Version="2.65.0" />`를 추가한다.
- .NET Framework에서는 `grpc` OTLP 익스포터 프로토콜이 지원되지 않는다.

**[2]**: `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE`에서
인식되는(대소문자를 구분하지 않는) 값은 다음과 같다.

- `Cumulative`: 모든 계측기 종류에 대해 누적(cumulative) 집계 템포럴리티를
  선택한다.
- `Delta`: Counter, Asynchronous Counter, Histogram 계측기 종류에 대해서는 Delta
  집계 템포럴리티를 선택하고, UpDownCounter와 Asynchronous UpDownCounter 계측기
  종류에 대해서는 Cumulative 집계를 선택한다.
- `LowMemory`: 이 구성은 Synchronous Counter와 Histogram에 대해서는 Delta 집계
  템포럴리티를 사용하고, Synchronous UpDownCounter, Asynchronous Counter,
  Asynchronous UpDownCounter 계측기 종류에 대해서는 Cumulative 집계 템포럴리티를
  사용한다.
  - ⚠️
    [명세](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.35.0/specification/metrics/sdk_exporters/otlp.md?plain=1#L48)에
    정의된 이 값은 지원되지 않는다.

**[3]**: mTLS(상호 TLS) 구성에 대한 고려 사항:

- mTLS는 .NET 8.0 이상에서만 지원된다.
- 모든 인증서 파일은 PEM 형식이어야 한다.
- mTLS를 사용할 때 `OTEL_EXPORTER_OTLP_ENDPOINT`는 `https://`를 사용해야 한다.
- mTLS는 .NET Framework에서 지원되지 않는다.

### Prometheus {#prometheus}

**상태**: [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)

> [!WARNING] 경고: **운영 환경에서 사용하지 말 것**
>
> Prometheus 익스포터는 내부 개발 루프(inner dev loop)를 위한 것이다. 운영
> 환경에서는
> [`otlp` 리시버](https://github.com/open-telemetry/opentelemetry-collector/tree/v0.97.0/receiver/otlpreceiver)와
> [`prometheus` 익스포터](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.97.0/exporter/prometheusexporter)를
> 갖춘
> [오픈텔레메트리 컬렉터](https://github.com/open-telemetry/opentelemetry-collector-releases)와
> OTLP 익스포터를 조합하여 사용할 수 있다.

Prometheus 익스포터를 활성화하려면 `OTEL_METRICS_EXPORTER` 환경 변수를
`prometheus`로 설정한다.

익스포터는 `http://localhost:9464/metrics`에 메트릭 HTTP 엔드포인트를 노출하며,
응답을 300밀리초 동안 캐시한다.

자세히 알아보려면
[Prometheus Exporter HttpListener 문서](https://github.com/open-telemetry/opentelemetry-dotnet/tree/coreunstable-1.16.0-beta.1/src/OpenTelemetry.Exporter.Prometheus.HttpListener)를
참고한다.

### Zipkin {#zipkin}

**상태**: [안정(Stable)](/docs/specs/otel/versioning-and-stability)

Zipkin 익스포터를 활성화하려면 `OTEL_TRACES_EXPORTER` 환경 변수를 `zipkin`으로
설정한다.

환경 변수로 Zipkin 익스포터를 커스터마이즈하려면
[Zipkin 익스포터 문서](https://github.com/open-telemetry/opentelemetry-dotnet/tree/core-1.16.0/src/OpenTelemetry.Exporter.Zipkin#configuration-using-environment-variables)를
참고한다. 주요 환경 변수는 다음과 같다.

| 환경 변수                       | 설명       | 기본값                               | 상태                                                      |
| ------------------------------- | ---------- | ------------------------------------ | --------------------------------------------------------- |
| `OTEL_EXPORTER_ZIPKIN_ENDPOINT` | Zipkin URL | `http://localhost:9411/api/v2/spans` | [안정(Stable)](/docs/specs/otel/versioning-and-stability) |

## 추가 설정 {#additional-settings}

| 환경 변수                                           | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 기본값                           | 상태                                                               |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------ |
| `OTEL_DOTNET_AUTO_TRACES_ENABLED`                   | 트레이스를 활성화한다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | `true`                           | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_OPENTRACING_ENABLED`              | OpenTracing 트레이서를 활성화한다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `false`                          | [지원 중단(Deprecated)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_LOGS_ENABLED`                     | 로그를 활성화한다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `true`                           | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_METRICS_ENABLED`                  | 메트릭을 활성화한다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | `true`                           | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_NETFX_REDIRECT_ENABLED`           | **지원 중단됨. .NET Framework 전용.** 기본 변수가 설정되지 않았을 때 `OTEL_DOTNET_AUTO_REDIRECT_ENABLED`의 대체값. 대신 `OTEL_DOTNET_AUTO_REDIRECT_ENABLED`를 사용한다.                                                                                                                                                                                                                                                                                                                                   |                                  | [지원 중단(Deprecated)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_REDIRECT_ENABLED`                 | 어셈블리 참조를 자동 계측이 사용하는 버전보다 낮지 않은 버전으로 리디렉션하도록 한다. 독립형(standalone) 배포에서는 기본값이 `true`이고, 비독립형 배포(예: NuGet 패키지 배포)에서는 `false`이다.                                                                                                                                                                                                                                                                                                          |                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES`        | 시작 시점에 트레이서에 추가할 `System.Diagnostics.ActivitySource` 이름의 쉼표로 구분된 목록. 수동으로 계측한 스팬을 캡처하는 데 사용한다.                                                                                                                                                                                                                                                                                                                                                                 |                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES` | 시작 시점에 트레이서에 추가할 추가 레거시 소스 이름의 쉼표로 구분된 목록. `System.Diagnostics.ActivitySource` API를 사용하지 않고 생성된 `System.Diagnostics.Activity` 객체를 캡처하는 데 사용한다.                                                                                                                                                                                                                                                                                                       |                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_FLUSH_ON_UNHANDLEDEXCEPTION`      | [AppDomain.UnhandledException](https://docs.microsoft.com/en-us/dotnet/api/system.appdomain.unhandledexception) 이벤트가 발생할 때 텔레메트리 데이터를 플러시할지 여부를 제어한다. 텔레메트리 데이터 누락 문제를 겪고 있고 처리되지 않은 예외도 발생하고 있다고 의심되면 `true`로 설정한다.                                                                                                                                                                                                               | `false`                          | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES`       | 시작 시점에 미터에 추가할 `System.Diagnostics.Metrics.Meter` 이름의 쉼표로 구분된 목록. 수동으로 생성한 메트릭을 캡처하는 데 사용한다.                                                                                                                                                                                                                                                                                                                                                                    |                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_PLUGINS`                          | OTel SDK 계측 플러그인 유형의 콜론으로 구분된 목록이며, [어셈블리 정규화 이름(assembly-qualified name)](https://docs.microsoft.com/en-us/dotnet/api/system.type.assemblyqualifiedname?view=net-6.0#system-type-assemblyqualifiedname)으로 지정한다. _참고: 유형 이름에 쉼표가 포함될 수 있으므로 이 목록은 콜론으로 구분해야 한다._ 플러그인 작성 방법에 대한 자세한 내용은 [plugins.md](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/v1.16.0/docs/plugins.md)를 참고한다. |                                  | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |
| `OTEL_DOTNET_AUTO_APP_DOMAIN_STRATEGY`              | .NET Framework 전용. 자동 계측이 기본이 아닌 AppDomain을 처리하는 방식을 정의한다. 지원되는 값은 `LoaderOptimizationSingleDomain`, `AssemblyRedirect`, `None`이다. 자세한 내용은 [.NET Framework AppDomain 전략](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/v1.16.0/docs/netfx-appdomain-strategy.md)을 참고한다.                                                                                                                                                        | `LoaderOptimizationSingleDomain` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability)  |

## RuleEngine {#ruleengine}

RuleEngine는 오픈텔레메트리 API, SDK, 계측(Instrumentation), 익스포터
어셈블리에서 지원되지 않는 시나리오를 검증하는 기능으로, 오픈텔레메트리 자동
계측이 충돌하는 대신 물러나도록 하여 더 안정적이도록 보장한다. .NET 8 이상에서
동작한다.

RuleEngine는 애플리케이션의 첫 실행 시, 또는 배포가 변경되거나 자동 계측
라이브러리가 업그레이드될 때만 활성화한다. 한 번 검증되면 애플리케이션이
재시작될 때 규칙을 다시 검증할 필요가 없다.

| 환경 변수                              | 설명                     | 기본값 | 상태                                                              |
| -------------------------------------- | ------------------------ | ------ | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_RULE_ENGINE_ENABLED` | RuleEngine를 활성화한다. | `true` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

## .NET CLR 프로파일러 {#net-clr-profiler}

CLR은 프로파일러를 설정하기 위해 다음 환경 변수를 사용한다. 자세한 내용은
[.NET 런타임 프로파일러 로딩](https://github.com/dotnet/runtime/blob/d8302cef7946be82775ba5b94a88ad8eee800714/docs/design/coreclr/profiling/Profiler%20Loading.md)을
참고한다.

| .NET Framework 환경 변수 | .NET 환경 변수             | 설명                                                                      | 필수 값                                                                                                                                                                                                                                                                    | 상태                                                              |
| ------------------------ | -------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `COR_ENABLE_PROFILING`   | `CORECLR_ENABLE_PROFILING` | 프로파일러를 활성화한다.                                                  | `1`                                                                                                                                                                                                                                                                        | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `COR_PROFILER`           | `CORECLR_PROFILER`         | 프로파일러의 CLSID.                                                       | `{918728DD-259F-4A6A-AC2B-B85E1B658318}`                                                                                                                                                                                                                                   | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `COR_PROFILER_PATH`      | `CORECLR_PROFILER_PATH`    | 프로파일러의 경로.                                                        | Linux glibc의 경우 `$INSTALL_DIR/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so`, Linux musl의 경우 `$INSTALL_DIR/linux-musl-x64/OpenTelemetry.AutoInstrumentation.Native.so`, macOS의 경우 `$INSTALL_DIR/osx-arm64/OpenTelemetry.AutoInstrumentation.Native.dylib` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `COR_PROFILER_PATH_32`   | `CORECLR_PROFILER_PATH_32` | 32비트 프로파일러의 경로. 비트 수에 특화된 경로가 일반 경로보다 우선한다. | Windows의 경우 `$INSTALL_DIR/win-x86/OpenTelemetry.AutoInstrumentation.Native.dll`                                                                                                                                                                                         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `COR_PROFILER_PATH_64`   | `CORECLR_PROFILER_PATH_64` | 64비트 프로파일러의 경로. 비트 수에 특화된 경로가 일반 경로보다 우선한다. | Windows의 경우 `$INSTALL_DIR/win-x64/OpenTelemetry.AutoInstrumentation.Native.dll`                                                                                                                                                                                         | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

.NET Framework에서는 오픈텔레메트리 .NET 자동 계측을 .NET CLR 프로파일러로
설정해야 한다.

.NET에서는 .NET CLR 프로파일러가 바이트코드 계측에만 사용된다. 소스 계측만으로
충분하다면 다음 환경 변수를 설정 해제하거나 제거할 수 있다.

```env
COR_ENABLE_PROFILING
COR_PROFILER
COR_PROFILER_PATH_32
COR_PROFILER_PATH_64
CORECLR_ENABLE_PROFILING
CORECLR_PROFILER
CORECLR_PROFILER_PATH
CORECLR_PROFILER_PATH_32
CORECLR_PROFILER_PATH_64
```

## .NET 런타임 {#net-runtime}

.NET에서는 .NET CLR 프로파일러를 사용하지 않는 경우
[`DOTNET_STARTUP_HOOKS`](https://github.com/dotnet/runtime/blob/main/docs/design/features/host-startup-hook.md)
환경 변수를 설정해야 한다.

[`DOTNET_ADDITIONAL_DEPS`](https://github.com/dotnet/runtime/blob/main/docs/design/features/additional-deps.md)와
[`DOTNET_SHARED_STORE`](https://docs.microsoft.com/en-us/dotnet/core/deploying/runtime-store)
환경 변수는 .NET에서 어셈블리 버전 충돌을 완화하는 데 사용된다.

| 환경 변수                | 필수 값                                                              | 상태                                                              |
| ------------------------ | -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `DOTNET_STARTUP_HOOKS`   | `$INSTALL_DIR/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `DOTNET_ADDITIONAL_DEPS` | `$INSTALL_DIR/AdditionalDeps`                                        | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `DOTNET_SHARED_STORE`    | `$INSTALL_DIR/store`                                                 | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

.NET CLR 프로파일러가 사용되고
[`DOTNET_STARTUP_HOOKS`](https://github.com/dotnet/runtime/blob/main/docs/design/features/host-startup-hook.md)
환경 변수가 설정되지 않은 경우, 프로파일러는
`OpenTelemetry.AutoInstrumentation.Native.dll` 파일 위치를 기준으로 적절한
디렉터리에서 `OpenTelemetry.AutoInstrumentation.StartupHook.dll`을 찾는다. 폴더
구조는 ZIP 아카이브 구조나 NuGet 패키지 구조(플랫폼 종속 또는 독립)와 일치할 수
있다. 시작 후크 어셈블리를 찾지 못하면 프로파일러 로딩이 중단된다.

## 내부 로그 {#internal-logs}

내부 로그의 기본 디렉터리 경로는 다음과 같다.

- Windows: `%ProgramData%\OpenTelemetry .NET AutoInstrumentation\logs`
- Linux: `/var/log/opentelemetry/dotnet`
- macOS: `/var/log/opentelemetry/dotnet`

기본 로그 디렉터리를 생성할 수 없으면, 계측은 대신 현재 사용자의
[임시 폴더](https://docs.microsoft.com/en-us/dotnet/api/System.IO.Path.GetTempPath?view=net-6.0)
경로를 사용한다.

| 환경 변수                        | 설명                                                                       | 기본값                                   | 상태                                                              |
| -------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_LOG_DIRECTORY` | .NET 트레이서 로그의 디렉터리.                                             | _기본 경로에 대한 앞의 참고 내용을 확인_ | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_LOG_LEVEL`                 | SDK 로그 수준. (지원되는 값: `none`,`error`,`warn`,`info`,`debug`)         | `info`                                   | [안정(Stable)](/docs/specs/otel/versioning-and-stability)         |
| `OTEL_DOTNET_AUTO_LOGGER`        | AutoInstrumentation 진단 로그 싱크. (지원되는 값: `none`,`file`,`console`) | `file`                                   | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_LOG_FILE_SIZE` | 자동 계측이 생성하는 단일 로그 파일의 최대 크기(바이트)                    | 10 485 760 (10 MB)                       | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |

## OpAMP 클라이언트 {#opamp-client}

| 환경 변수                           | 설명                           | 기본값                            | 상태                                                              |
| ----------------------------------- | ------------------------------ | --------------------------------- | ----------------------------------------------------------------- |
| `OTEL_DOTNET_AUTO_OPAMP_ENABLED`    | OpAMP 클라이언트를 활성화한다. | `false`                           | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
| `OTEL_DOTNET_AUTO_OPAMP_SERVER_URL` | OpAMP 서버 URL.                | `https://localhost:4320/v1/opamp` | [실험적(Experimental)](/docs/specs/otel/versioning-and-stability) |
