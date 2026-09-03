---
title: 로그
description: 이벤트의 기록이다.
weight: 3
cSpell:ignore: filelogreceiver semistructured transformprocessor
default_lang_commit: c161165987d527c1efd6bc969d7fef905946c561
---

**로그(log)** 는 타임스탬프가 찍힌 텍스트 레코드이며, 구조화되어 있거나(권장)
비구조화되어 있을 수 있고, 메타데이터는 선택 사항이다. 모든 텔레메트리 시그널
중에서 로그가 가장 오래된 역사를 가진다. 대부분의 프로그래밍 언어는 내장된 로깅
기능을 갖추고 있거나, 잘 알려져 널리 사용되는 로깅 라이브러리를 갖추고 있다.

## 오픈텔레메트리(OpenTelemetry) 로그 {#opentelemetry-logs}

오픈텔레메트리는 로그 레코드를 생성하기 위한 로그 API 및 SDK와, 기존 로깅
프레임워크와 통합하기 위한 언어별 SDK 및 로깅 브리지를 제공한다. 로그는 로거
프로바이더(Logger Provider)를 통해 전송하는 모든 것이며, 이벤트는 로그의 특수한
유형이다. 모든 로그가 이벤트인 것은 아니지만, 모든 이벤트는 로그이다. 로그 API는
공개되어 있으며 애플리케이션 코드에서 직접 사용하거나, 기존 로깅 라이브러리와
브리지를 통해 간접적으로 사용할 수 있다.

오픈텔레메트리는 이미 생성하고 있는 로그와 함께 동작하도록 설계되어 있으며,
로그를 다른 시그널과 상관관계 짓고, 컨텍스트 속성을 추가하고, 서로 다른 소스를
처리(process) 및 내보내기(export)를 위한 공통 표현으로 정규화하는 도구를
제공한다.

### 오픈텔레메트리 컬렉터에서의 오픈텔레메트리 로그 {#opentelemetry-logs-in-the-opentelemetry-collector}

[오픈텔레메트리 컬렉터](/docs/collector/)는 로그를 다루기 위한 여러 도구를
제공한다.

- 특정하고 잘 알려진 로그 데이터 소스로부터 로그를 파싱하는 여러
  리시버(receiver).
- 어떤 파일에서든 로그를 읽어 들이고, 이를 다양한 포맷으로 파싱하거나 정규
  표현식을 사용할 수 있는 기능을 제공하는 `filelogreceiver`.
- 중첩된 데이터를 파싱하고, 중첩된 구조를 평탄화하고, 값을 추가/제거/업데이트
  하는 등의 작업을 할 수 있게 해주는 `transformprocessor`와 같은
  프로세서(processor).
- 로그 데이터를 오픈텔레메트리가 아닌 포맷으로 내보낼 수 있게 해주는
  익스포터(exporter).

오픈텔레메트리를 도입하는 첫 단계는 흔히 범용 로깅 에이전트로서 컬렉터를
배포하는 것을 포함한다.

### 애플리케이션을 위한 오픈텔레메트리 로그 {#opentelemetry-logs-for-applications}

애플리케이션에서는 어떤 로깅 라이브러리나 내장된 로깅 기능을 사용해도
오픈텔레메트리 로그를 생성할 수 있다. 자동 계측(autoinstrumentation)을
추가하거나 SDK를 활성화하면, 오픈텔레메트리는 기존 로그를 활성 트레이스 및
스팬과 자동으로 연관 지으며, 로그 본문에 해당 ID를 포함시킨다. 다시 말해,
오픈텔레메트리는 로그와 트레이스를 자동으로 연관 짓는다.

### 언어 지원 {#language-support}

로그는 오픈텔레메트리 명세에서
[안정적인(stable)](/docs/specs/otel/versioning-and-stability/#stable)
시그널이다. 개별 언어별 로그 API 및 SDK 구현의 상태는 다음과 같다.

{{% signal-support-table "logs" %}}

## 구조화된(Structured), 비구조화된(unstructured), 반구조화된(semistructured) 로그 {#structured-unstructured-and-semistructured-logs}

오픈텔레메트리는 어떤 로그 포맷이든 받아들이지만, 모든 포맷이 분석에 동일하게
유용한 것은 아니다. 다음 절에서는 구조화된 로그, 반구조화된 로그, 비구조화된
로그의 차이를 설명한다. 중요: JSON으로 인코딩된 로그라고 해서 안정적인
스키마(schema)를 가진다는 의미에서 자동으로 "구조화된" 것은 아니며, 반구조화된
로그일 수도 있다. 구조화된 로그는 다운스트림(downstream) 처리가 신뢰성 있게
의존할 수 있는 일관된 스키마 또는 잘 정의된 타입 필드(typed field)를 내포한다.

### 구조화된 로그(Structured logs) {#structured-logs}

구조화된 로그란 다운스트림 시스템이 신뢰성 있게 파싱하고 해석할 수 있는,
정의되고 일관된 스키마 또는 타입 필드를 가진 로그이다. 텍스트 인코딩은 JSON,
protobuf, 또는 다른 포맷일 수 있지만, 로그를 구조화된 것으로 만드는 것은
안정적인 스키마(필드 이름, 타입, 의미)의 존재이지, 단순히 유효한 JSON이라는
사실이 아니다. 예를 들어 구조화된 JSON 로그는 다음과 같은 모습일 수 있다.

```json
{
  "timestamp": "2024-08-04T12:34:56.789Z",
  "level": "INFO",
  "service": "user-authentication",
  "environment": "production",
  "message": "User login successful",
  "context": {
    "userId": "12345",
    "username": "johndoe",
    "ipAddress": "192.168.1.1",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
  },
  "transactionId": "abcd-efgh-ijkl-mnop",
  "duration": 200,
  "request": {
    "method": "POST",
    "url": "/api/v1/login",
    "headers": {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    "body": {
      "username": "johndoe",
      "password": "******"
    }
  },
  "response": {
    "statusCode": 200,
    "body": {
      "success": true,
      "token": "jwt-token-here"
    }
  }
}
```

그리고 인프라 구성 요소의 경우, 공통 로그 포맷(Common Log Format, CLF)이 흔히
사용된다.

```text
127.0.0.1 - johndoe [04/Aug/2024:12:34:56 -0400] "POST /api/v1/login HTTP/1.1" 200 1234
```

CLF 필드에 후행 JSON 블롭(blob)이 결합된 것과 같은, 하이브리드 또는 확장 포맷을
마주치는 것도 흔한 일이다.

```text
192.168.1.1 - johndoe [04/Aug/2024:12:34:56 -0400] "POST /api/v1/login HTTP/1.1" 200 1234 "http://example.com" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36" {"transactionId": "abcd-efgh-ijkl-mnop", "responseTime": 150, "requestBody": {"username": "johndoe"}, "responseHeaders": {"Content-Type": "application/json"}}
```

이런 경우에는 필요한 부분을 파싱하거나 추출하여 정규화된 레코드로 만들면,
다운스트림 도구가 일관되게 분석할 수 있다.
[오픈텔레메트리 컬렉터](/docs/collector/)의 `filelogreceiver`는 혼합된 포맷을
파싱하는 헬퍼를 제공한다.

구조화된 로그는 안정적인 스키마 덕분에 검증, 파싱, 트레이스 및 메트릭과의 연관
짓기, 대규모 분석이 간단하기 때문에 프로덕션 환경에서 선호된다.

### 비구조화된 로그(Unstructured logs) {#unstructured-logs}

비구조화된 로그는 일관된 구조를 따르지 않는 로그이다. 사람이 읽기에 더 편할 수
있으며, 개발 과정에서 자주 사용된다. 하지만 프로덕션 옵저버빌리티(observability)
목적으로는 비구조화된 로그 사용이 선호되지 않는데, 대규모로 파싱하고 분석하기가
훨씬 더 어렵기 때문이다.

비구조화된 로그의 예:

```text
[ERROR] 2024-08-04 12:45:23 - Failed to connect to database. Exception: java.sql.SQLException: Timeout expired. Attempted reconnect 3 times. Server: db.example.com, Port: 5432

System reboot initiated at 2024-08-04 03:00:00 by user: admin. Reason: Scheduled maintenance. Services stopped: web-server, database, cache. Estimated downtime: 15 minutes.

DEBUG - 2024-08-04 09:30:15 - User johndoe performed action: file_upload. Filename: report_Q3_2024.pdf, Size: 2.3 MB, Duration: 5.2 seconds. Result: Success
```

프로덕션 환경에서도 비구조화된 로그를 저장하고 분석하는 것이 가능하지만,
머신에서 읽을 수 있게 파싱하거나 사전 처리하는 데 상당한 작업이 필요할 수 있다.
예를 들어, 위의 세 가지 로그는 타임스탬프를 파싱하기 위한 정규 표현식과, 로그
메시지의 본문을 일관되게 추출하기 위한 커스텀 파서가 필요하다. 이는 일반적으로
로깅 백엔드가 타임스탬프 기준으로 로그를 정렬하고 정리하는 방법을 알기 위해
필요하다. 분석 목적으로 비구조화된 로그를 파싱하는 것이 가능하긴 하지만, 이는
애플리케이션에서 표준 로깅 프레임워크를 사용하는 것과 같이 구조화된 로깅으로
전환하는 것보다 더 많은 작업이 필요할 수 있다.

### 반구조화된 로그(Semistructured logs) {#semistructured-logs}

반구조화된 로그는 머신에서 읽을 수 있는 키/값 쌍이나 구분된 필드를 포함하지만,
이미터(emitter) 간에 안정적인 스키마를 보장하지는 않는다. 예를 들어 (아래에
표시된) key=value 로깅이나, 메시지마다 필드 이름과 타입이 달라지는 JSON 블롭이
있다. 반구조화된 로그는 비구조화된 로그보다 파싱하기 쉬운 경우가 많지만, 분석
전에 여전히 처리와 정규화가 필요할 수 있다.

반구조화된 로그의 예:

```text
2024-08-04T12:45:23Z level=ERROR service=user-authentication userId=12345 action=login message="Failed login attempt" error="Invalid password" ipAddress=192.168.1.1 userAgent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
```

반구조화된 로그는 다운스트림 분석에 완전히 유용하려면, 유입(ingestion) 과정에서
매핑과 타입 강제 변환(type coercion)이 필요할 수 있다.

## 로깅이 활성화되어 있는지 확인하기 {#checking-whether-logging-is-enabled}

애플리케이션 코드가 로깅 호출을 하기 전에 로깅이 활성화되어 있는지 확인해야
하는지는 흔히 나오는 질문이다. 예를 들면 다음과 같다.

```text
if (logger.Enabled(...)) {
  logger.Info("Hello {name}", name);
}
```

대부분의 경우 이 확인은 불필요하며 권장되지 않는다. 오픈텔레메트리 SDK는
효율적으로 설계되어 있어서, 로거가 활성화되어 있지 않을 때 로깅 API 호출의
오버헤드가 최소화된다. `logger.Enabled`를 추가로 호출하면 성능이 저하되고 코드가
더 복잡해진다.

`Enabled` API는 로깅 호출에 전달되는 _인자를 평가하는 것_ 자체가 비용이 큰
경우에만 유용하며, 로거가 활성화되어 있지 않을 때 그 비용을 피하고 싶을 때
유용하다. 예를 들어 본문이나 속성을 데이터베이스에서 가져오거나, 비용이 큰
연산을 통해 계산해야 하는 경우이다.

```text
if (logger.Enabled(...)) {
  logger.Info("Order total {total}", ComputeExpensiveTotal());
}
```

가드(guard)로 보호된 코드는 로깅이 활성화된 경우에만 실행되므로, 부작용(side
effect)이 없는 표현식만 가드해야 한다. 부작용이 있거나 다른 로직이 의존하는
코드를 가드하면, 애플리케이션의 동작이 로깅 설정에 의존하게 되며, 이는 보통
미묘한 버그의 원인이 된다.

그렇다 하더라도, `Enabled`의 결과는 정적이지 않다는 점을 기억한다. 설정이
변경됨에 따라 시간이 지나면서 값이 달라질 수 있으므로, 로그 레코드마다 평가해야
하며 캐시해서는 안 된다.

규범적인(normative) API 가이드는
[로그 API 명세](/docs/specs/otel/logs/api/#enabled)를 참고한다.

## 오픈텔레메트리 로깅 구성 요소 {#opentelemetry-logging-components}

다음 개념 및 구성 요소 목록이 오픈텔레메트리의 로깅 지원을 뒷받침한다.

### 로그 어펜더(Appender) / 브리지(Bridge) {#log-appender--bridge}

애플리케이션 개발자라면 **로그 브리지 API(Logs Bridge API)** 를 직접 호출해서는
안 된다. 이는 로깅 라이브러리 작성자가 로그 어펜더/브리지를 구축할 수 있도록
제공되는 것이기 때문이다. 대신 선호하는 로깅 라이브러리를 사용하고,
오픈텔레메트리 LogRecordExporter로 로그를 내보낼 수 있는 로그 어펜더(또는 로그
브리지)를 사용하도록 구성하면 된다.

오픈텔레메트리 언어 SDK가 이 기능을 제공한다.

### 로거 프로바이더(Logger Provider) {#logger-provider}

> **로그 브리지 API**의 일부이며, 로깅 라이브러리 작성자인 경우에만 사용해야
> 한다.

로거 프로바이더(때로는 `LoggerProvider`라고도 불린다)는 `Logger`의
팩토리(factory)이다. 대부분의 경우 로거 프로바이더는 한 번만 초기화되며, 그 수명
주기는 애플리케이션의 수명 주기와 일치한다. 로거 프로바이더 초기화에는 리소스와
익스포터 초기화도 포함된다.

### 로거(Logger) {#logger}

> **로그 브리지 API**의 일부이며, 로깅 라이브러리 작성자인 경우에만 사용해야
> 한다.

로거는 로그 레코드를 생성한다. 로거는 로거 프로바이더로부터 생성된다.

### 로그 레코드 익스포터(Log Record Exporter) {#log-record-exporter}

로그 레코드 익스포터는 로그 레코드를 소비자(consumer)에게 전송한다. 이 소비자는
디버깅 및 개발 시점을 위한 표준 출력일 수도 있고, 오픈텔레메트리 컬렉터, 또는
원하는 오픈소스나 벤더 백엔드일 수도 있다.

### 로그 레코드(Log Record) {#log-record}

로그 레코드는 이벤트의 기록을 나타낸다. 오픈텔레메트리에서 로그 레코드는 두 가지
종류의 필드를 포함한다.

- 특정 타입과 의미를 가진 이름 있는 최상위 필드
- 임의의 값과 타입을 가진 리소스 및 속성 필드

최상위 필드는 다음과 같다.

> **표기 규칙**: 아래 표의 "필드 이름"은 로그 데이터 모델에 정의된 실제 필드
> 식별자이므로 영문 그대로 표기한다. 본문에서 개념으로 언급하는 리소스, 속성과는
> 구분된다.

| 필드 이름            | 설명                                       |
| -------------------- | ------------------------------------------ |
| Timestamp            | 이벤트가 발생한 시간.                      |
| ObservedTimestamp    | 이벤트가 관측된 시간.                      |
| TraceId              | 요청의 트레이스 ID.                        |
| SpanId               | 요청의 스팬 ID.                            |
| TraceFlags           | W3C 트레이스 플래그.                       |
| SeverityText         | 심각도 텍스트(로그 레벨이라고도 함).       |
| SeverityNumber       | 심각도의 숫자 값.                          |
| Body                 | 로그 레코드의 본문.                        |
| Resource             | 로그의 소스를 설명한다.                    |
| InstrumentationScope | 로그를 내보낸 스코프를 설명한다.           |
| Attributes           | 이벤트에 대한 추가 정보.                   |
| EventName            | 이벤트의 클래스 또는 유형을 식별하는 이름. |

로그 레코드와 로그 필드에 대한 자세한 내용은
[로그 데이터 모델](/docs/specs/otel/logs/data-model/)을 참고한다.

### 명세(Specification) {#specification}

오픈텔레메트리에서 로그에 대해 더 알아보려면 [로그 명세][logs specification]를
참고한다.

[logs specification]: /docs/specs/otel/overview/#log-signal
