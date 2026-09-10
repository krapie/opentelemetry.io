---
title: PHP 제로 코드 계측
linkTitle: 자동 계측
weight: 20
aliases:
  - /docs/languages/php/automatic
  - /docs/zero-code/php/
cSpell:ignore: centos democlass epel pecl phar remi
default_lang_commit: be35d47dc1ad8f2c4d3607927a14e9c4cb2d2102
---

## 요구 사항 {#requirements}

PHP에서 자동 계측을 사용하려면 다음이 필요하다.

- PHP 8.0 이상
- [오픈텔레메트리(OpenTelemetry) PHP 확장(extension)](https://github.com/open-telemetry/opentelemetry-php-instrumentation)
- [Composer 오토로딩(autoloading)](https://getcomposer.org/doc/01-basic-usage.md#autoloading)
- [오픈텔레메트리 SDK](https://packagist.org/packages/open-telemetry/sdk)
- 하나 이상의
  [계측 라이브러리](/ecosystem/registry/?component=instrumentation&language=php)
- [구성](#configuration)

## 오픈텔레메트리 확장 설치하기 {#install-the-opentelemetry-extension}

> [!IMPORTANT]
>
> 오픈텔레메트리 확장(extension)을 설치하는 것만으로는 트레이스가 생성되지
> 않는다.

이 확장은 pecl, [pickle](https://github.com/FriendsOfPHP/pickle),
[PIE](https://github.com/php/pie),
[php-extension-installer](https://github.com/mlocati/docker-php-extension-installer)(도커
전용)를 통해 설치할 수 있다. 일부 리눅스 패키지 관리자용으로 패키징된 버전의
확장도 제공된다.

### 리눅스 패키지 {#linux-packages}

다음에서 RPM 및 APK 패키지를 제공한다.

- [Remi 저장소](https://blog.remirepo.net/pages/PECL-extensions-RPM-status) -
  RPM
- [Alpine Linux](https://pkgs.alpinelinux.org/packages?name=*pecl-opentelemetry) -
  APK(현재
  [_testing_ 브랜치](https://wiki.alpinelinux.org/wiki/Repositories#Testing)에
  있음)

{{< tabpane text=true >}} {{% tab "RPM" %}}

```sh
#this example is for CentOS 7. The PHP version can be changed by
#enabling remi-<version>, eg "yum config-manager --enable remi-php83"
yum update -y
yum install -y epel-release yum-utils
yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum-config-manager --enable remi-php81
yum install -y php php-pecl-opentelemetry

php --ri opentelemetry
```

{{% /tab %}} {{% tab "APK" %}}

```sh
#At the time of writing, PHP 8.1 was the default PHP version. You may need to
#change "php81" if the default changes. You can alternatively choose a PHP
#version with "apk add php<version>", eg "apk add php83".
echo "@testing https://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories
apk add php php81-pecl-opentelemetry@testing
php --ri opentelemetry
```

{{% /tab %}} {{< /tabpane >}}

### PECL {#pecl}

1. 개발 환경을 설정한다. 소스에서 설치하려면 적절한 개발 환경과 몇 가지 의존성이
   필요하다.

   {{< tabpane text=true >}} {{% tab "Linux (apt)" %}}

   ```sh
   sudo apt-get install gcc make autoconf
   ```

   {{% /tab %}} {{% tab "macOS (homebrew)" %}}

   ```sh
   brew install gcc make autoconf
   ```

   {{% /tab %}} {{< /tabpane >}}

2. 확장을 빌드하거나 설치한다. 환경이 준비되면 확장을 설치할 수 있다.

   {{< tabpane text=true >}} {{% tab pecl %}}

   ```sh
   pecl install opentelemetry
   ```

   {{% /tab %}} {{% tab pickle %}}

   ```sh
   php pickle.phar install opentelemetry
   ```

   {{% /tab %}} {{% tab "php-extension-installer (docker)" %}}

   ```sh
   install-php-extensions opentelemetry
   ```

   {{% /tab %}} {{< /tabpane >}}

3. `php.ini` 파일에 확장을 추가한다.

   ```ini
   [opentelemetry]
   extension=opentelemetry.so
   ```

4. 확장이 설치되고 활성화되었는지 확인한다.

   ```sh
   php -m | grep opentelemetry
   ```

## SDK 및 계측 라이브러리 설치하기 {#install-sdk-and-instrumentation-libraries}

이제 확장이 설치되었으니, 오픈텔레메트리 SDK와 하나 이상의 계측 라이브러리를
설치한다.

자동 계측은 흔히 사용되는 여러 PHP 라이브러리에 대해 제공된다. 전체 목록은
[packagist의 계측 라이브러리](https://packagist.org/search/?query=open-telemetry&tags=instrumentation)를
참고한다.

애플리케이션이 Slim Framework와 PSR-18 HTTP 클라이언트를 사용하고, 트레이스를
OTLP 프로토콜로 내보낸다고 가정한다.

그러면 SDK, 익스포터, 그리고 Slim Framework와 PSR-18용 자동 계측 패키지를
설치한다.

```shell
composer require \
    open-telemetry/sdk \
    open-telemetry/exporter-otlp \
    open-telemetry/opentelemetry-auto-slim \
    open-telemetry/opentelemetry-auto-psr18
```

## 구성 {#configuration}

오픈텔레메트리 SDK와 함께 사용할 때는 환경 변수나 `php.ini` 파일로 자동 계측을
구성할 수 있다.

### 환경 변수 구성 {#environment-configuration}

```sh
OTEL_PHP_AUTOLOAD_ENABLED=true \
OTEL_SERVICE_NAME=your-service-name \
OTEL_TRACES_EXPORTER=otlp \
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf \
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318 \
OTEL_PROPAGATORS=baggage,tracecontext \
php myapp.php
```

### php.ini 구성 {#phpini-configuration}

`php.ini` 또는 PHP가 처리할 다른 `ini` 파일에 다음을 추가한다.

```ini
OTEL_PHP_AUTOLOAD_ENABLED="true"
OTEL_SERVICE_NAME=your-service-name
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
OTEL_PROPAGATORS=baggage,tracecontext
```

## 애플리케이션 실행하기 {#run-your-application}

위의 모든 것을 설치하고 구성했다면, 평소처럼 애플리케이션을 시작한다.

오픈텔레메트리 컬렉터로 내보내지는 트레이스는 설치한 계측 라이브러리와
애플리케이션 내부에서 실행된 코드 경로에 따라 달라진다. 앞의 예시처럼 Slim
Framework와 PSR-18 계측 라이브러리를 사용하는 경우 다음과 같은 스팬을 볼 수
있다.

- HTTP 트랜잭션을 나타내는 루트 스팬
- 실행된 액션에 대한 스팬
- PSR-18 클라이언트가 보낸 각 HTTP 트랜잭션에 대한 스팬

PSR-18 클라이언트 계측은 나가는 HTTP 요청에
[분산 트레이싱](/docs/concepts/context-propagation/#propagation) 헤더를
덧붙인다는 점에 유의한다.

## 동작 방식 {#how-it-works}

> [!NOTE] 선택 사항
>
> 빠르게 실행하는 것만이 목적이고 애플리케이션에 적합한 계측 라이브러리가 있다면
> 이 섹션은 건너뛰어도 된다.

이 확장을 사용하면 클래스와 메서드에 대해 PHP 코드로 옵저버(observer) 함수를
등록하고, 관찰 대상 메서드가 실행되기 전과 후에 해당 함수를 실행할 수 있다.

사용하는 프레임워크나 애플리케이션에 대한 계측 라이브러리가 없다면 직접 작성할
수 있다. 다음 예시는 계측할 코드를 제공한 뒤, 오픈텔레메트리 확장을 사용해 해당
코드의 실행을 추적하는 방법을 보여준다.

```php
<?php

use OpenTelemetry\API\Instrumentation\CachedInstrumentation;
use OpenTelemetry\API\Trace\Span;
use OpenTelemetry\API\Trace\StatusCode;
use OpenTelemetry\Context\Context;

require 'vendor/autoload.php';

/* The class to be instrumented */
class DemoClass
{
    public function run(): void
    {
        echo 'Hello, world';
    }
}

/* The auto-instrumentation code */
OpenTelemetry\Instrumentation\hook(
    class: DemoClass::class,
    function: 'run',
    pre: static function (DemoClass $demo, array $params, string $class, string $function, ?string $filename, ?int $lineno) {
        static $instrumentation;
        $instrumentation ??= new CachedInstrumentation('example');
        $span = $instrumentation->tracer()->spanBuilder('democlass-run')->startSpan();
        Context::storage()->attach($span->storeInContext(Context::getCurrent()));
    },
    post: static function (DemoClass $demo, array $params, $returnValue, ?Throwable $exception) {
        $scope = Context::storage()->scope();
        $scope->detach();
        $span = Span::fromContext($scope->context());
        if ($exception) {
            $span->recordException($exception);
            $span->setStatus(StatusCode::STATUS_ERROR);
        }
        $span->end();
    }
);

/* Run the instrumented code, which will generate a trace */
$demo = new DemoClass();
$demo->run();
```

앞의 예시는 `DemoClass`를 정의한 다음, 그 `run` 메서드에 `pre`와 `post` 훅(hook)
함수를 등록한다. 훅 함수는 `DemoClass::run()` 메서드가 실행될 때마다 그 전과
후에 실행된다. `pre` 함수는 스팬을 시작하고 활성화하며, `post` 함수는 스팬을
종료한다.

`DemoClass::run()`이 예외를 던지면, `post` 함수는 예외 전파에 영향을 주지 않고
예외를 기록한다.

## 다음 단계 {#next-steps}

앱이나 서비스에 대한 자동 계측을 구성했다면, 커스텀 텔레메트리 데이터를 수집하기
위해 [수동 계측](/docs/languages/php/instrumentation)을 추가할 수도 있다.

더 많은 예시는
[opentelemetry-php-contrib/examples](https://github.com/open-telemetry/opentelemetry-php-contrib/tree/main/examples)를
참고한다.
