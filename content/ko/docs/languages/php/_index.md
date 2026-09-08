---
title: PHP
description: >-
  <img width="35" class="img-initial otel-icon" src="/img/logos/32x32/PHP.svg"
  alt="PHP"> PHP에서의 오픈텔레메트리(OpenTelemetry) 언어별 구현체이다.
redirects:
  - { from: /php/*, to: ':splat' }
  - { from: /docs/php/*, to: ':splat' }
weight: 180
default_lang_commit: d8f5ed285d009cc6baac6d7141bfde8d0956a756
cSpell:ignore: mbstring opcache
---

{{% docs/languages/index-intro php /%}}

## 추가 자료 {#further-reading}

- [GitHub의 OpenTelemetry for PHP](https://github.com/open-telemetry/opentelemetry-php)
- [예제](https://github.com/open-telemetry/opentelemetry-php/tree/main/examples)

## 요구 사항 {#requirements}

PHP용 오픈텔레메트리 SDK는
[www.php.net/supported-versions](https://www.php.net/supported-versions.php)에
따라 공식적으로 지원되는 모든 PHP 버전을 지원하는 것을 목표로 하며, 해당 버전의
지원 종료(End of Life) 후 12개월이 지나면 그 버전에 대한 지원이 중단된다.

자동 계측을 사용하려면 PHP 8.0 이상이 필요하다.

### 종속성 {#dependencies}

일부 `SDK` 및 `Contrib` 패키지는
[HTTP Factories (PSR-17)](https://www.php-fig.org/psr/psr-17/)와
[php-http/async-client](https://docs.php-http.org/en/latest/clients.html) 구현체
모두에 종속성을 갖는다. 해당 표준을 구현하는 적절한 composer 패키지는
[packagist.org](https://packagist.org/)에서 찾을 수 있다.

`PSR-17 (HTTP factories)` 구현체를 찾으려면
[http-factory-implementations](https://packagist.org/providers/psr/http-factory-implementation)를,
`php-http/async-client` 구현체를 찾으려면
[async-client-implementations](https://packagist.org/providers/php-http/async-client-implementation)를
참고한다.

### 선택적 PHP 확장 {#optional-php-extensions}

| 확장                                                                      | 용도                                                            |
| ------------------------------------------------------------------------- | --------------------------------------------------------------- |
| [ext-grpc](https://github.com/grpc/grpc/tree/master/src/php)              | OTLP 익스포터의 전송 방식으로 gRPC를 사용하기 위해 필요         |
| [ext-mbstring](https://www.php.net/manual/en/book.mbstring.php)           | 대체 수단인 `symfony/polyfill-mbstring`보다 더 나은 성능을 제공 |
| [ext-zlib](https://www.php.net/manual/en/book.zlib.php)                   | 내보낸 데이터를 압축하려는 경우 사용                            |
| [ext-ffi](https://www.php.net/manual/en/book.ffi.php)                     | Fiber 기반 컨텍스트 저장소                                      |
| [ext-protobuf](https://github.com/protocolbuffers/protobuf/tree/main/php) | otlp+protobuf 내보내기에서 _상당한_ 성능 향상을 제공            |

#### ext-ffi {#ext-ffi}

`OTEL_PHP_FIBERS_ENABLED` 환경 변수를 `true`로 설정하면 Fiber 지원을 활성화할 수
있다. `CLI`가 아닌 SAPI와 함께 Fiber를 사용하려면 바인딩을 미리 로드해야 할 수
있다. 이를 위한 한 가지 방법은
[`ffi.preload`](https://www.php.net/manual/en/ffi.configuration.php#ini.ffi.preload)를
`src/Context/fiber/zend_observer_fiber.h`로 설정하고,
[`opcache.preload`](https://www.php.net/manual/en/opcache.preloading.php)를
`vendor/autoload.php`로 설정하는 것이다.

#### ext-protobuf {#ext-protobuf}

[네이티브 protobuf 라이브러리](https://packagist.org/packages/google/protobuf)는
확장보다 훨씬 느리다. 확장을 사용할 것을 강력히 권장한다.

## 설정 {#setup}

PHP용 오픈텔레메트리는
[packagist](https://packagist.org/packages/open-telemetry/)를 통해 여러 패키지로
배포된다. 필요한 패키지만 설치할 것을 권장하며, 최소한 `API`, `Context`, `SDK`와
익스포터 하나가 일반적으로 필요하다.

코드가 `API` 패키지에 있는 클래스와 인터페이스에만 의존하도록 할 것을 강력히
권장한다.
