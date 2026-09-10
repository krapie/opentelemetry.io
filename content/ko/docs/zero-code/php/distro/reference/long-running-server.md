---
title: 장기 실행 PHP 서버
description: >-
  Laravel Octane(Swoole, RoadRunner) 및 기타 상시 실행 PHP 서버 프로세스를 위한
  오픈텔레메트리(OpenTelemetry) PHP Distro 구성 방법이다.
weight: 5
# prettier-ignore
cSpell:ignore: apache2handler artisan BatchSpanProcessor FPM fpm-fcgi HttpTransportAsync onEnd php-fpm RoadRunner SIGKILL SIGTERM SimpleSpanProcessor Swoole
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

**Laravel Octane**(Swoole 또는 RoadRunner 사용)와 같은 PHP 프레임워크는 요청마다
새 프로세스를 생성하는 대신 PHP를 지속적으로 실행되는 서버 프로세스로 실행한다.
이는 몇 가지 중요한 방식으로 배포판(distro)의 동작을 바꾸며, 특정한 구성 조정이
필요하다.

## 전통적인 PHP 서버의 동작 방식 {#how-traditional-php-servers-work}

**PHP-FPM** 또는 **Apache mod_php**에서는 각 HTTP 요청이 자체 PHP 프로세스 수명
주기에 매핑된다.

1. PHP 프로세스 시작 → 배포판 부트스트랩(OTel SDK 초기화, 자동 계측 훅 등록)
2. 요청 처리 → HTTP 트랜잭션과 계측된 호출(curl, PDO 등)에 대해 스팬 생성
3. 응답 전송 → PHP 종료 함수 실행 → 스팬이 플러시되고 내보내짐
4. PHP 프로세스 종료

배포판의 **트랜잭션 스팬**(`OTEL_PHP_TRANSACTION_SPAN_ENABLED`)은 정확히 하나의
HTTP 요청을 감싼다. 요청이 도착할 때(웹 SAPI, `$_SERVER` 채워짐) 시작되어
프로세스가 종료될 때 끝난다. 이것은 모든 자식 스팬(curl, DB 쿼리 등)이 매달리는
루트 스팬이다.

## 장기 실행 서버의 차이점 {#how-long-running-servers-differ}

**Laravel Octane**(Swoole 또는 RoadRunner)에서는 하나의 PHP 워커 프로세스가 그
사이에 종료되지 않고 여러 HTTP 요청을 연달아 처리한다.

1. PHP 프로세스 시작 — 워커 프로세스가 완전히 초기화된 배포판(SDK, 훅,
   익스포터)과 함께 시작된다
2. 각 HTTP 요청이 워커로 디스패치된다 — 워커의 훅이 실행되고 스팬이 생성된다
3. 응답이 전송된다 — **PHP 종료 함수는 실행되지 않는다**(프로세스가 계속됨)
4. 워커 프로세스는 서버가 중지될 때(정상 종료)에만 종료된다

워커 프로세스는 **CLI 프로세스**(`php artisan octane:start`로 시작됨)이므로,
SAPI는 항상 `fpm-fcgi`나 `apache2handler`가 아닌 `cli`다. 배포판은 이를 이용해
다음을 구분한다.

- `OTEL_PHP_TRANSACTION_SPAN_ENABLED` — 웹 SAPI(FPM/Apache)용 루트 스팬.
  여기서는 관련이 없다.
- `OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI` — CLI 프로세스용 루트 스팬. 장기 실행
  서버에서는 개별 요청이 아니라 **전체 서버 수명**(`octane:start`부터
  `octane:stop`까지)을 감싼다.

| 항목                  | PHP-FPM / Apache              | 장기 실행 서버(Octane)                |
| --------------------- | ----------------------------- | ------------------------------------- |
| SAPI                  | `fpm-fcgi` / `apache2handler` | `cli`                                 |
| 프로세스 수명         | 요청당 프로세스 1개           | 워커 1개가 여러 요청 처리             |
| PHP 종료 함수         | 매 요청 후 실행               | 워커 종료 시에만 실행                 |
| 배포판 부트스트랩     | 요청마다 실행                 | 시작 시 워커당 한 번 실행             |
| 트랜잭션 스팬(`_CLI`) | 해당 없음                     | 전체 서버 수명에 걸침 — 비활성화할 것 |

## 권장 구성 {#recommended-configuration}

### CLI 트랜잭션 스팬 비활성화 {#disable-the-cli-transaction-span}

자동 루트 스팬(`OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI`)은 전체 PHP 프로세스를
감싼다. 장기 실행 서버에서는 서버가 종료될 때까지 지속되는 스팬 하나를 의미하며,
이는 유용한 텔레메트리가 아니다.

```sh
export OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI=false
```

> [!NOTE] `OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI`는 CLI 전용이며 웹
> SAPI(FPM/Apache) 배포에는 영향을 주지 않는다.

### 추론된 스팬 비활성화 {#disable-inferred-spans}

추론된 스팬(스택 트레이스 샘플링)은 전통적인 요청 기반 PHP를 위해 설계되었다.
장기 실행 서버에서는 샘플링이 요청 사이에도 계속 실행되어 노이즈를 만들고 CPU를
소모한다.

```sh
export OTEL_PHP_INFERRED_SPANS_ENABLED=false
```

이것이 기본값이므로, 이전에 추론된 스팬을 전역으로 활성화한 경우에만 필요하다.

### 스팬 프로세서와 내보내기 지연 {#span-processor-and-export-latency}

기본적으로 배포판은 `BatchSpanProcessor`를 사용하며, 이는 스팬을 메모리에 쌓아
두었다가 타이머에 따라(기본값: 5초마다) 내보낸다. PHP는 단일 스레드이므로,
타이머 확인은 `onEnd()`를 통해 새 스팬이 끝날 때만 실행되며 백그라운드 틱은
없다. 워커가 현재 요청을 마치고 PHP 종료 함수를 실행하며 종료 전에 익스포터를
플러시할 수 있도록, 항상 정상 종료(`php artisan octane:stop`, SIGTERM)를
사용한다. 강제 종료(SIGKILL)는 이 모든 것을 건너뛴다.

정상 종료를 사용하면, OTLP 엔드포인트에 도달할 수 있는 한 메모리에 버퍼링된
스팬은 프로세스가 종료되기 전에 플러시된다. 다만 `BatchSpanProcessor`는 요청이
도착하는 빈도에 비례하는 **내보내기 지연**을 유발한다. 트래픽이 적은
애플리케이션에서는 10:00에 생성된 스팬이 10:05가 되어서야(다음 요청이 마침내
타이머 확인을 트리거) 컬렉터에 나타날 수 있다. 거의 실시간에 가까운 가시성을
원한다면 `SimpleSpanProcessor`를 사용한다.

```sh
export OTEL_PHP_TRACES_PROCESSOR=simple
```

각 스팬은 트래픽 양과 관계없이 `onEnd()` 시점에 즉시 내보내기 큐로 푸시된다.

배포판의 네이티브 C++ 트랜스포트(`HttpTransportAsync`)는 자체 내부 큐와 OTLP
엔드포인트로의 지속 연결을 갖고 있으므로, `simple`로 전환한다고 해서 스팬당 HTTP
요청 하나를 의미하지는 **않는다**. PHP 계층은 C++ 큐에 동기적으로 푸시하며(빠른
인프로세스 작업), C++ 계층은 지속 연결을 통해 독립적으로 배치 처리하고 전송한다.

## 전체 예시 {#complete-example}

동일한 구성이 Swoole과 RoadRunner 모두에 적용된다.

```sh
export OTEL_SERVICE_NAME="my-laravel-octane-app"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"

# Long-running server adjustments
export OTEL_PHP_TRANSACTION_SPAN_ENABLED_CLI=false
export OTEL_PHP_INFERRED_SPANS_ENABLED=false
export OTEL_PHP_TRACES_PROCESSOR=simple

# Swoole
php artisan octane:start --server=swoole

# RoadRunner
php artisan octane:start --server=roadrunner
```

## 서버 유형별 계측 동작 방식 {#how-instrumentation-works-per-server-type}

### Swoole {#swoole}

Swoole은 마스터 PHP 프로세스를 포크(fork)하여 워커 프로세스를 만든다. 배포판은
마스터에서 한 번 부트스트랩되고, 워커는 초기화된 상태(훅, SDK, 익스포터 연결)를
상속한다. 워커에서 발생하는 각 HTTP 요청은 등록된 자동 계측 훅(Laravel, curl,
PDO 등)을 트리거하며, 스팬은 워커의 TracerProvider 아래에서 생성된다.

### RoadRunner {#roadrunner}

RoadRunner는 PHP 워커 프로세스를 관리하는 Go 기반 애플리케이션 서버다. Swoole과
달리 포크하지 않고, 각 PHP 워커를 별도의 프로세스로 생성한다. 그 결과 각 워커는
시작 시 배포판을 독립적으로 부트스트랩한다. 계측 관점에서 동작은 동일하다.
워커의 SAPI는 `cli`이고, 그 사이에 종료되지 않고 여러 요청을 처리하며, 동일한
구성이 적용된다.

## BatchSpanProcessor 스케줄 지연 {#batchspanprocessor-schedule-delay}

`BatchSpanProcessor`를 유지하면서 내보내기 지연을 줄이고 싶다면, 스케줄 지연을
낮춘다.

```sh
export OTEL_BSP_SCHEDULE_DELAY=500   # ms, default is 5000
```

이것은 트래픽이 꾸준한 애플리케이션에서만 도움이 된다. 타이머는 `onEnd()`에서
실행되므로, 트래픽이 적은 애플리케이션은 여전히 요청 간 간격에 비례하는 지연을
겪는다.
