---
title: 견적 서비스
linkTitle: 견적
aliases: [quoteservice]
cSpell:ignore: getquote
default_lang_commit: 98a528997da383a8e152021f920ce510572b1b87
---

이 서비스는 배송할 상품 수를 기준으로 배송 비용을 계산하는 역할을 담당한다. 견적
서비스는 배송 서비스에서 HTTP를 통해 호출된다.

견적 서비스는 Slim 프레임워크와, 의존성 주입(Dependency Injection)을 관리하기
위한 php-di를 사용해 구현되어 있다.

PHP 계측은 사용하는 프레임워크에 따라 달라질 수 있다.

[견적 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/quote/)

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

이 데모에서 오픈텔레메트리(OpenTelemetry) SDK는 composer 오토로딩의 일부로
이루어지는 SDK 오토로딩의 일환으로 자동 생성되었다.

이는 환경 변수 `OTEL_PHP_AUTOLOAD_ENABLED=true`를 설정하여 활성화된다.

```php
require __DIR__ . '/../vendor/autoload.php';
```

`Tracer`를 생성하거나 얻는 방법은 여러 가지가 있는데, 이 예시에서는 위에서 SDK
오토로딩의 일부로 초기화된 전역 트레이서 프로바이더에서 하나를 얻어온다.

```php
$tracer = Globals::tracerProvider()->getTracer('manual-instrumentation');
```

### 수동으로 스팬 생성하기 {#manually-creating-spans}

`Tracer`를 통해 수동으로 스팬을 생성할 수 있다. 이 스팬은 기본적으로 현재 실행
컨텍스트에서 활성 스팬의 자식이 된다.

```php
$span = Globals::tracerProvider()
    ->getTracer('manual-instrumentation')
    ->spanBuilder('calculate-quote')
    ->setSpanKind(SpanKind::KIND_INTERNAL)
    ->startSpan();
/* calculate quote */
$span->end();
```

### 스팬 속성 추가 {#add-span-attributes}

`OpenTelemetry\API\Trace\Span`을 사용해 현재 스팬을 얻을 수 있다.

```php
$span = Span::getCurrent();
```

스팬에 속성을 추가하려면 스팬 객체의 `setAttribute`를 사용한다. `calculateQuote`
함수에서는 `childSpan`에 2개의 속성이 추가된다.

```php
$childSpan->setAttribute('app.quote.items.count', $numberOfItems);
$childSpan->setAttribute('app.quote.cost.total', $quote);
```

### 스팬 이벤트 추가 {#add-span-events}

스팬 이벤트를 추가하려면 스팬 객체의 `addEvent`를 사용한다. `getquote`
라우트에서 스팬 이벤트가 추가된다. 일부 이벤트에는 추가 속성이 있고, 그렇지 않은
이벤트도 있다.

속성 없이 스팬 이벤트를 추가하는 경우:

```php
$span->addEvent('Received get quote request, processing it');
```

추가 속성과 함께 스팬 이벤트를 추가하는 경우:

```php
$span->addEvent('Quote processed, response sent back', [
    'app.quote.cost.total' => $payload
]);
```

## 메트릭 {#metrics}

이 데모에서 메트릭은 배치 트레이스 및 로그 프로세서가 발생시킨다. 이 메트릭은
내보낸 스팬이나 로그의 수, 큐 한도, 큐 사용량 등 프로세서의 내부 상태를
설명한다.

환경 변수 `OTEL_PHP_INTERNAL_METRICS_ENABLED`를 `true`로 설정하면 메트릭을
활성화할 수 있다.

또한 생성된 견적 수를 세는 수동 메트릭도 발생하며, 여기에는 항목 수에 대한
속성이 포함된다.

카운터는 전역으로 구성된 미터 프로바이더에서 생성되며, 견적이 생성될 때마다
증가한다.

```php
static $counter;
$counter ??= Globals::meterProvider()
    ->getMeter('quotes')
    ->createCounter('quotes', 'quotes', 'number of quotes calculated');
$counter->add(1, ['number_of_items' => $numberOfItems]);
```

메트릭은 누적되며, `OTEL_METRIC_EXPORT_INTERVAL`에 구성된 값을 기준으로
주기적으로 내보내진다.

## 로그 {#logs}

견적 서비스는 견적이 계산된 후 로그 메시지를 발생시킨다. Monolog 로깅 패키지는
Monolog 로그를 오픈텔레메트리 형식으로 변환하는
[로그 브리지](/docs/concepts/signals/logs/#log-appender--bridge)와 함께 구성되어
있다. 이 로거로 전송된 로그는 전역으로 구성된 오픈텔레메트리 로거를 통해
내보내진다.
