---
title: 어트리뷰트 기반 계측
description: >-
  오픈텔레메트리(OpenTelemetry) PHP Distro에서 PHP 8 어트리뷰트(attribute)를
  사용해 스팬을 자동으로 생성한다.
weight: 4
cSpell:ignore: SpanAttribute WithSpan
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

오픈텔레메트리(OpenTelemetry) PHP Distro는 PHP 8 어트리뷰트(attribute)를 사용한
자동 스팬 생성을 지원한다. 메서드나 함수에 `#[WithSpan]`을 붙이면 계측 코드를
수동으로 작성하지 않고도 스팬을 만들 수 있다.

> **표기 규칙**: 이 문서에서 "어트리뷰트(attribute)"는 `#[...]` 문법의 PHP 언어
> 기능을 가리킨다. 스팬에 부여되는 오픈텔레메트리 속성(Attribute)과는 구분되며,
> 후자는 "스팬 속성(span attribute)"으로 표기한다.

## 사전 요구 사항 {#prerequisites}

- PHP 8.0 이상(PHP 어트리뷰트는 PHP 8 이상이 필요하다).
- 애플리케이션에 `open-telemetry/api` 패키지 설치.
- 환경에 `OTEL_PHP_ATTR_HOOKS_ENABLED=true` 설정(기본적으로 비활성화됨).

## 활성화 {#enable}

```sh
export OTEL_PHP_ATTR_HOOKS_ENABLED=true
```

또는 `php.ini`에서:

```ini
opentelemetry_distro.attr_hooks_enabled=true
```

## 기본 사용법 {#basic-usage}

```php
use OpenTelemetry\API\Instrumentation\WithSpan;

class OrderService
{
    #[WithSpan]
    public function processOrder(int $orderId): string
    {
        // A span named "OrderService::processOrder" is created automatically.
        return "processed-{$orderId}";
    }
}
```

## `#[WithSpan]` 옵션 {#withspan-options}

```php
#[WithSpan(
    span_name: 'custom.span.name',          // default: "ClassName::methodName"
    span_kind: SpanKind::KIND_SERVER,        // default: KIND_INTERNAL
    attributes: ['key' => 'value'],          // static attributes added to the span
)]
```

모든 인자는 선택 사항이며, 위치로 전달하거나 이름으로 전달할 수 있다.

```php
// Positional
#[WithSpan('payment.charge', SpanKind::KIND_CLIENT, ['db.system' => 'redis'])]

// Named — any subset
#[WithSpan(span_kind: SpanKind::KIND_PRODUCER)]
#[WithSpan(span_name: 'message.publish', span_kind: SpanKind::KIND_PRODUCER)]
```

## `#[SpanAttribute]`로 매개변수 값 캡처하기 {#capturing-parameter-values-with-spanattribute}

함수 매개변수에 `#[SpanAttribute]`를 추가하면 해당 런타임 값이 스팬 속성으로
포함된다.

```php
use OpenTelemetry\API\Instrumentation\WithSpan;
use OpenTelemetry\API\Instrumentation\SpanAttribute;

class UserService
{
    #[WithSpan]
    public function createUser(
        #[SpanAttribute] string $username,               // attribute key = "username"
        string                  $password,               // not captured
        #[SpanAttribute('user.email')] string $email,   // attribute key = "user.email"
    ): int {
        // ...
    }
}
```

## `#[SpanAttribute]`로 프로퍼티 값 캡처하기 {#capturing-property-values-with-spanattribute}

클래스 프로퍼티에 `#[SpanAttribute]`를 적용하면 메서드가 호출되는 시점의 값이
캡처된다.

```php
class InvoiceService
{
    #[SpanAttribute]
    public string $customerId = '';

    #[SpanAttribute('invoice.currency')]
    public string $currency = 'EUR';

    #[WithSpan('invoice.generate')]
    public function generate(): string
    {
        // Span attributes include: customerId, invoice.currency
    }
}
```

## 예외 기록 {#exception-recording}

어노테이션이 붙은 메서드가 예외를 던지면, 스팬은 자동으로 예외를 기록하고 상태를
`ERROR`로 설정한다. 예외는 정상적으로 전파된다.

```php
#[WithSpan]
public function riskyOperation(): void
{
    throw new \RuntimeException('something went wrong');
    // Span is ended with STATUS_ERROR and exception event attached.
}
```

## 중첩 스팬 {#nested-spans}

한 `#[WithSpan]` 메서드에서 다른 `#[WithSpan]` 메서드를 호출하면 중첩 스팬이
자동으로 생성된다.

```php
class Pipeline
{
    #[WithSpan('pipeline.run')]
    public function run(): void
    {
        $this->step1(); // child span: "pipeline.step1"
        $this->step2(); // child span: "pipeline.step2"
    }

    #[WithSpan('pipeline.step1')]
    private function step1(): void {}

    #[WithSpan('pipeline.step2')]
    private function step2(): void {}
}
```

## 독립 함수 {#standalone-functions}

`#[WithSpan]`은 메서드뿐만 아니라 독립 함수에서도 동작한다.

```php
#[WithSpan('compute.result')]
function computeResult(#[SpanAttribute] int $input): int
{
    return $input * 2;
}
```

## 표준 스팬 속성 {#standard-span-attributes}

모든 `#[WithSpan]` 스팬은 선언 위치에서 가져온 다음 속성을 포함한다.

| 속성             | 값                                      |
| ---------------- | --------------------------------------- |
| `code.function`  | 함수 또는 메서드 이름                   |
| `code.namespace` | 클래스 이름(독립 함수의 경우 비어 있음) |
| `code.filepath`  | 소스 파일 경로                          |
| `code.lineno`    | 선언의 줄 번호                          |

## 호환성 {#compatibility}

`#[WithSpan]`과 `#[SpanAttribute]`는 공식
[opentelemetry-php-instrumentation](https://github.com/open-telemetry/opentelemetry-php-instrumentation)
확장에서 사용하는 것과 동일한 어트리뷰트다. 이미 해당 확장을 사용 중인
애플리케이션은 코드 변경 없이 이 기능을 활성화할 수 있다.
