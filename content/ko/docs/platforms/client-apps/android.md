---
title: Android
description: >-
  Android 플랫폼에서 실행되는 앱에서 오픈텔레메트리(OpenTelemetry)를 사용한다.
weight: 10
vers:
  ot-android: 1.6.0
cSpell:ignore: inactivity
default_lang_commit: 63c0544931da26813836bc6fe49ee2888cacf64a
---

오픈텔레메트리(OpenTelemetry) Android는 네이티브 Android 애플리케이션을 위한
옵저버빌리티(observability)를 제공한다.
[오픈텔레메트리 Java](/docs/languages/java/) 생태계 위에 구축되어, 모바일 환경에
맞춘 자동 계측(automatic instrumentation), 실사용자 모니터링(Real User
Monitoring, RUM), 수동 계측(manual instrumentation) 기능을 제공한다.

## 기능 {#features}

오픈텔레메트리 Android는 다음과 같은 주요 기능을 포함한다.

- **자동 계측(Automatic Instrumentation)**: 일반적인 Android 패턴을 위한 내장
  모듈이다.
  - Activity 라이프사이클
  - Fragment 라이프사이클
  - ANR(Application Not Responding) 감지
  - 크래시 리포팅(Crash reporting)
  - 네트워크 변경 감지
  - 느린/멈춘 프레임 렌더링 감지
  - 시작 타이밍(Startup timing)
  - 화면 방향
  - View 클릭 이벤트
- **세션 관리(Session Management)**: 구성 가능한 비활성 타임아웃(inactivity
  timeout)과 최대 세션 수명을 사용해 사용자 세션을 추적한다.
- **오프라인 버퍼링(Offline Buffering)**: 기기가 오프라인 상태일 때 텔레메트리
  데이터를 버퍼링하기 위한 디스크 지속성(persistence)이며, 네트워크 중단 중에도
  데이터 손실이 없도록 보장한다.
- **속성 편집(Attribute Redaction)**: 개인정보 보호 준수를 위해 내보내기 전에
  스팬 속성을 편집하거나 수정하는 기능이다.

## 시작하기 {#getting-started}

### 사전 요구 사항 {#prerequisites}

- Android SDK 21(Lollipop) 이상
- Kotlin을 사용하는 Gradle 프로젝트(Java도 가능할 수 있다)

### Gradle 설정 {#gradle-setup}

앱 수준의 `build.gradle.kts` 파일에 오픈텔레메트리 Android 에이전트(Agent)
의존성을 추가한다. 버전 관리에는 BOM(Bill of Materials)을 사용한다.

```kotlin
dependencies {
    implementation(platform("io.opentelemetry.android:opentelemetry-android-bom:{{% param vers.ot-android %}}"))
    implementation("io.opentelemetry.android:android-agent")
}
```

> [!NOTE]
>
> 최신 버전은
> [오픈텔레메트리 Android 릴리스](https://github.com/open-telemetry/opentelemetry-android/releases)에서
> 확인한다.

### 에이전트 초기화 {#initialize-the-agent}

`Application` 클래스의 `onCreate()` 메서드에서 오픈텔레메트리를 초기화한다.

```kotlin
class MyApplication : Application() {
    lateinit var openTelemetryRum: OpenTelemetryRum

    override fun onCreate() {
        super.onCreate()
        openTelemetryRum = initializeOpenTelemetry(this)
    }
}

private fun initializeOpenTelemetry(context: Context): OpenTelemetryRum =
    OpenTelemetryRumInitializer.initialize(
        context = context,
        configuration = {
            httpExport {
                baseUrl = "https://your-collector-endpoint:4318"
                baseHeaders = mapOf("Authorization" to "Bearer <token>")
            }
            instrumentations {
                // All instrumentations are enabled by default.
                // Disable specific ones as needed:
                slowRendering { enabled(false) }
            }
            session {
                backgroundInactivityTimeout = 15.minutes
                maxLifetime = 4.days
            }
        }
    )
```

## 구성 {#configuration}

오픈텔레메트리 Android는 위 초기화 예제에서 보듯 구성(configuration)을 위해
Kotlin DSL을 사용한다. 다음 표는 사용 가능한 구성 옵션을 설명한다.

### 구성 옵션 {#configuration-options}

| 블록                                      | 설명                                           |
| ----------------------------------------- | ---------------------------------------------- |
| `httpExport { baseUrl }`                  | 텔레메트리를 내보내기 위한 OTLP 엔드포인트 URL |
| `httpExport { baseHeaders }`              | 내보내기 요청에 포함할 커스텀 헤더             |
| `globalAttributes`                        | 모든 텔레메트리에 추가되는 속성                |
| `session { backgroundInactivityTimeout }` | 새 세션을 시작하기 전의 비활성 타임아웃        |
| `session { maxLifetime }`                 | 최대 세션 수명                                 |
| `instrumentations`                        | 개별 자동 계측 모듈 구성                       |

## 자동 계측 {#automatic-instrumentation}

오픈텔레메트리 Android는 활성화하거나 비활성화할 수 있는 자동 계측 모듈을
제공한다. 방출되는 텔레메트리와 구성 옵션을 포함해 각 계측에 대한 자세한 정보는
연결된 문서를 참고한다.

### Activity 라이프사이클 {#activity-lifecycle}

Activity 라이프사이클 이벤트(`onCreate`, `onStart`, `onResume`, `onPause`,
`onStop`, `onDestroy`)에 대한 스팬을 자동으로 캡처한다.
[Activity 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/activity/README.md)을
참고한다.

### Fragment 라이프사이클 {#fragment-lifecycle}

Fragment 라이프사이클 이벤트에 대한 스팬을 캡처하며, 단일 Activity 아키텍처
내에서 내비게이션을 추적하는 데 유용하다.
[Fragment 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/fragment/README.md)을
참고한다.

### ANR 감지 {#anr-detection}

ANR(Application Not Responding) 상태를 감지하여 스팬으로 보고하며, UI 스레드
차단 문제를 식별하는 데 도움을 준다.
[ANR 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/anr/README.md)을
참고한다.

### 크래시 리포팅 {#crash-reporting}

처리되지 않은 예외(unhandled exception)를 캡처하고 스택 트레이스(stack trace)와
함께 보고하여, 크래시를 사용자 세션 및 트레이스와 연관 지을 수 있게 해준다.
[크래시 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/crash/README.md)을
참고한다.

### 네트워크 모니터링 {#network-monitoring}

네트워크 상태 변경을 감지하고 연결 정보를 텔레메트리에 추가하여, 오류 발생 시
네트워크 상태를 파악하는 데 도움을 준다.
[네트워크 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/network/README.md)을
참고한다.

### 느린 프레임과 멈춘 프레임 {#slow-and-frozen-frames}

프레임 렌더링 성능을 모니터링하고 느린 렌더링(16ms 초과)과 멈춘 프레임(700ms
초과)을 보고하여 UI 성능 병목을 식별하는 데 도움을 준다.
[느린 렌더링 계측](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/slowrendering/README.md)을
참고한다.

## 수동 계측 {#manual-instrumentation}

수동 계측을 위해 오픈텔레메트리 API에 접근한다.

```kotlin
val openTelemetry = openTelemetryRum.openTelemetry
val tracer = openTelemetry.getTracer("com.example.myapp")

val span = tracer.spanBuilder("my-operation")
    .startSpan()

try {
    span.makeCurrent().use {
        // Your code here
    }
} finally {
    span.end()
}
```

## HTTP 클라이언트 계측 {#http-client-instrumentation}

네트워크 요청을 트레이싱하기 위해 OkHttp 클라이언트를 계측한다.

```kotlin
val okHttpClient = OkHttpTelemetry.builder(openTelemetryRum.openTelemetry)
    .build()
    .newCallFactory(OkHttpClient.Builder().build())
```

## 모범 사례 {#best-practices}

### 리소스 제약 {#resource-constraints}

모바일 기기는 리소스가 제한적이다. 다음과 같은 모범 사례를 고려한다.

- **배치 내보내기(batch export)**: 네트워크 호출과 배터리 소모를 줄이기 위해
  배치 처리(batch processing)가 기본적으로 활성화되어 있다.
- **샘플링(Sampling)**: 대표성 있는 텔레메트리를 유지하면서 데이터 양을 줄이기
  위한 샘플링 전략을 구현한다.
- **오프라인 버퍼링(Offline buffering)**: 간헐적인 연결 상태를 처리하기 위해
  디스크 지속성(persistence)이 기본적으로 활성화되어 있다.

### 개인정보 보호 고려 사항 {#privacy-considerations}

- 내보내기 전에 민감한 데이터를 제거하려면 속성 편집(attribute redaction)을
  사용한다.
- 텔레메트리를 수집하기 위한 사용자 동의 요구 사항을 고려한다.
- 스팬 이름이나 속성에 개인 식별 정보(personally identifiable information,
  PII)를 캡처하지 않도록 한다.

### 테스트 {#testing}

에뮬레이터로 테스트할 때는 로컬 머신의 컬렉터에 접근하기 위한 호스트 주소로
`10.0.2.2`를 사용한다.

```kotlin
httpExport {
    baseUrl = "http://10.0.2.2:4318"
}
```

## 리소스 {#resources}

- [오픈텔레메트리 Android GitHub](https://github.com/open-telemetry/opentelemetry-android)
- [오픈텔레메트리 Java 문서](/docs/languages/java/)
- [Android 시맨틱 컨벤션](/docs/specs/semconv/registry/attributes/android/)
- [예제 애플리케이션](https://github.com/open-telemetry/opentelemetry-android/tree/main/demo-app)

## 도움말 및 피드백 {#help-and-feedback}

질문이 있다면
[GitHub Issues](https://github.com/open-telemetry/opentelemetry-android/issues)나
[CNCF Slack](https://slack.cncf.io/)의
[#otel-android](https://cloud-native.slack.com/archives/C05J0T9K27Q) 채널을 통해
문의한다.
