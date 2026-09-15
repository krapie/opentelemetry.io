---
title: API로 텔레메트리 기록하기
weight: 11
aliases: [/docs/languages/java/api-components]
logBridgeWarning: >-
  `LoggerProvider`/`Logger` API는 구조적으로 트레이스 및 메트릭의 대응 API와
  유사하지만, 사용 사례는 다르다. 현재 `LoggerProvider`/`Logger` 및 관련
  클래스는 [로그 브리지(Log Bridge) API](/docs/specs/otel/logs/api/)를 나타내며,
  이는 다른 로그 API/프레임워크로 기록된 로그를 오픈텔레메트리로 연결하는 로그
  어펜더(appender)를 작성하기 위해 존재한다. 이들은 Log4j/SLF4J/Logback 등을
  대체하는 최종 사용자용 API로 사용하기 위한 것이 아니다.
cSpell:ignore: kotlint updowncounter
default_lang_commit: 55e6ad73fcc68806a2af2dbbfc7b1905659844dd
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/api"?>

API는 핵심 옵저버빌리티 시그널 전반에 걸쳐 텔레메트리를 기록하기 위한 클래스와
인터페이스의 집합이다. [SDK](../sdk/)는 텔레메트리를 처리하고 내보내도록
[구성](../configuration/)된, API의 내장 참조 구현체이다. 이 페이지는 설명, 관련
Javadoc 링크, 아티팩트 좌표, API 사용 예시를 포함하는 API에 대한 개념적
개요이다.

API는 다음과 같은 최상위 구성 요소로 이루어진다.

- [Context](#context-api): 트레이스 컨텍스트와 배기지를 포함하여, 애플리케이션
  전체와 애플리케이션 경계를 넘어 컨텍스트를 전파하기 위한 독립형 API이다.
- [TracerProvider](#tracerprovider): 트레이스를 위한 API 진입점이다.
- [MeterProvider](#meterprovider): 메트릭을 위한 API 진입점이다.
- [LoggerProvider](#loggerprovider): 로그를 위한 API 진입점이다.
- [OpenTelemetry](#opentelemetry): 최상위 API 구성 요소(즉, `TracerProvider`,
  `MeterProvider`, `LoggerProvider`, `ContextPropagators`)를 담는 홀더로, 계측에
  전달하기 편리하다.

API는 여러 구현체를 지원하도록 설계되었다. 오픈텔레메트리는 두 가지 구현체를
제공한다.

- [SDK](../sdk/) 참조 구현체. 대부분의 사용자에게 적합한 선택이다.
- [무동작(No-op)](#no-op-implementation) 구현체. 사용자가 인스턴스를 설치하지
  않았을 때 계측이 기본값으로 사용하는, 최소한의 의존성 없는 구현체이다.

API는 라이브러리, 프레임워크, 애플리케이션 소유자가 직접적인 의존성으로 가져다
쓰도록 설계되었다. 이는
[강력한 하위 호환성 보장](https://github.com/open-telemetry/opentelemetry-java/blob/main/VERSIONING.md#compatibility-requirements),
전이 의존성 없음(zero transitive dependencies), 그리고
[Java 8 이상 지원](https://github.com/open-telemetry/opentelemetry-java/blob/main/VERSIONING.md#language-version-compatibility)을
제공한다. 라이브러리와 프레임워크는 API에만 의존하고 API의 메서드만 호출해야
하며, 애플리케이션/최종 사용자에게 SDK에 대한 의존성을 추가하고 구성된
인스턴스를 설치하도록 안내해야 한다.

> [!NOTE] Javadoc
>
> 모든 오픈텔레메트리(OpenTelemetry) Java 구성 요소의 Javadoc 참조는
> [javadoc.io/doc/io.opentelemetry](https://javadoc.io/doc/io.opentelemetry)를
> 참고한다.

## API 구성 요소 {#api-components}

다음 절에서는 오픈텔레메트리 API에 대해 설명한다. 각 구성 요소 절은 다음을
포함한다.

- Javadoc 타입 참조 링크를 포함한 간단한 설명.
- API 메서드와 인자를 이해하는 데 도움이 되는 관련 리소스 링크.
- API 사용에 대한 간단한 탐구.

## Context(컨텍스트) API {#context-api}

`io.opentelemetry:opentelemetry-api-context:{{% param vers.otel %}}` 아티팩트는
애플리케이션 전체와 애플리케이션 경계를 넘어 컨텍스트를 전파하기 위한 독립형(즉,
[오픈텔레메트리 API](#opentelemetry-api)와 별도로 패키징된) API를 포함한다.

이는 다음으로 구성된다.

- [Context](#context): 애플리케이션을 통해 암묵적으로 또는 명시적으로 전파되는,
  변경 불가능한 키-값 쌍 묶음이다.
- [ContextStorage](#contextstorage): 현재 컨텍스트를 저장하고 조회하는
  메커니즘으로, 기본값은 스레드 로컬(thread local)이다.
- [ContextPropagators](#context): 애플리케이션 경계를 넘어 `Context`를 전파하기
  위해 등록된 전파자(propagator)의 컨테이너이다.

`io.opentelemetry:opentelemetry-extension-kotlint:{{% param vers.otel %}}`는
코루틴(coroutine)으로 컨텍스트를 전파하기 위한 도구를 제공하는 익스텐션이다.

### Context(컨텍스트) {#context}

[Context](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-context/latest/io/opentelemetry/context/Context.html)는
애플리케이션 전체와 스레드를 넘어 암묵적으로 전파하기 위한 유틸리티를 갖춘, 변경
불가능한 키-값 쌍 묶음이다. 암묵적 전파란 컨텍스트를 인자로 명시적으로 전달하지
않고도 접근할 수 있음을 의미한다. 컨텍스트는 오픈텔레메트리 API에서 반복적으로
등장하는 개념이다.

- 현재 활성 [Span](#span)은 컨텍스트에 저장되며, 기본적으로 스팬의 부모는 현재
  컨텍스트에 있는 스팬으로 지정된다.
- [메트릭 계측기](#meter)에 기록되는 측정값은 컨텍스트 인자를 받으며, 이는
  [예시값(exemplar)](/docs/specs/otel/metrics/data-model/#exemplars)을 통해
  측정값을 스팬과 연결하는 데 사용되고, 기본값은 현재 컨텍스트에 있는 스팬이다.
- [LogRecords](#logrecordbuilder)는 컨텍스트 인자를 받으며, 이는 로그 레코드와
  스팬을 연결하는 데 사용되고, 기본값은 현재 컨텍스트에 있는 스팬이다.

다음 코드 스니펫은 `Context` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ContextUsage.java"?>
```java
package otel;

import io.opentelemetry.context.Context;
import io.opentelemetry.context.ContextKey;
import io.opentelemetry.context.Scope;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ContextUsage {
  public static void contextUsage() throws Exception {
    // Define an example context key
    ContextKey<String> exampleContextKey = ContextKey.named("example-context-key");

    // Context doesn't contain the key until we add it
    // Context.current() accesses the current context
    // output => current context value: null
    System.out.println("current context value: " + Context.current().get(exampleContextKey));

    // Add entry to context
    Context context = Context.current().with(exampleContextKey, "value");

    // The local context var contains the added value
    // output => context value: value
    System.out.println("context value: " + context.get(exampleContextKey));
    // The current context still doesn't contain the value
    // output => current context value: null
    System.out.println("current context value: " + Context.current().get(exampleContextKey));

    // Calling context.makeCurrent() sets Context.current() to the context until the scope is
    // closed, upon which Context.current() is restored to the state prior to when
    // context.makeCurrent() was called. The resulting Scope implements AutoCloseable and is
    // normally used in a try-with-resources block. Failure to call Scope.close() is an error and
    // may cause memory leaks or other issues.
    try (Scope scope = context.makeCurrent()) {
      // The current context now contains the added value
      // output => context value: value
      System.out.println("context value: " + Context.current().get(exampleContextKey));
    }

    // The local context var still contains the added value
    // output => context value: value
    System.out.println("context value: " + context.get(exampleContextKey));
    // The current context no longer contains the value
    // output => current context value: null
    System.out.println("current context value: " + Context.current().get(exampleContextKey));

    ExecutorService executorService = Executors.newSingleThreadExecutor();
    ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(1);

    // Context instances can be explicitly passed around application code, but it's more convenient
    // to use implicit context, calling Context.makeCurrent() and accessing via Context.current().
    // Context provides a number of utilities for implicit context propagation. These utilities wrap
    // utility classes like Scheduler, ExecutorService, ScheduledExecutorService, Runnable,
    // Callable, Consumer, Supplier, Function, etc and modify their behavior to call
    // Context.makeCurrent() before running.
    context.wrap(ContextUsage::callable).call();
    context.wrap(ContextUsage::runnable).run();
    context.wrap(executorService).submit(ContextUsage::runnable);
    context.wrap(scheduledExecutorService).schedule(ContextUsage::runnable, 1, TimeUnit.SECONDS);
    context.wrapConsumer(ContextUsage::consumer).accept(new Object());
    context.wrapConsumer(ContextUsage::biConsumer).accept(new Object(), new Object());
    context.wrapFunction(ContextUsage::function).apply(new Object());
    context.wrapSupplier(ContextUsage::supplier).get();
  }

  /** Example {@link java.util.concurrent.Callable}. */
  private static Object callable() {
    return new Object();
  }

  /** Example {@link Runnable}. */
  private static void runnable() {}

  /** Example {@link java.util.function.Consumer}. */
  private static void consumer(Object object) {}

  /** Example {@link java.util.function.BiConsumer}. */
  private static void biConsumer(Object object1, Object object2) {}

  /** Example {@link java.util.function.Function}. */
  private static Object function(Object object) {
    return object;
  }

  /** Example {@link java.util.function.Supplier}. */
  private static Object supplier() {
    return new Object();
  }
}
```
<!-- prettier-ignore-end -->

### ContextStorage(컨텍스트 스토리지) {#contextstorage}

[ContextStorage](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-context/latest/io/opentelemetry/context/ContextStorage.html)는
현재 `Context`를 저장하고 조회하는 메커니즘이다.

기본 `ContextStorage` 구현체는 `Context`를 스레드 로컬에 저장한다.

### ContextPropagators(컨텍스트 전파자) {#contextpropagators}

[ContextPropagators](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-context/latest/io/opentelemetry/context/propagation/ContextPropagators.html)는
애플리케이션 경계를 넘어 `Context`를 전파하기 위해 등록된 전파자의 컨테이너이다.
컨텍스트는 애플리케이션을 벗어날 때(즉, 아웃바운드 HTTP 요청) 캐리어(carrier)에
주입되고, 애플리케이션에 진입할 때(즉, HTTP 요청을 처리할 때) 캐리어에서
추출된다.

전파자 구현체는 [SDK TextMapPropagator](../sdk/#textmappropagator)를 참고한다.

다음 코드 스니펫은 주입을 위한 `ContextPropagators` API를 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/InjectContextUsage.java"?>
```java
package otel;

import io.opentelemetry.api.baggage.propagation.W3CBaggagePropagator;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class InjectContextUsage {
  private static final TextMapSetter<HttpRequest.Builder> TEXT_MAP_SETTER = new HttpRequestSetter();

  public static void injectContextUsage() throws Exception {
    // Create a ContextPropagators instance which propagates w3c trace context and w3c baggage
    ContextPropagators propagators =
        ContextPropagators.create(
            TextMapPropagator.composite(
                W3CTraceContextPropagator.getInstance(), W3CBaggagePropagator.getInstance()));

    // Create an HttpRequest builder
    HttpClient httpClient = HttpClient.newBuilder().build();
    HttpRequest.Builder requestBuilder =
        HttpRequest.newBuilder().uri(new URI("http://127.0.0.1:8080/resource")).GET();

    // Given a ContextPropagators instance, inject the current context into the HTTP request carrier
    propagators.getTextMapPropagator().inject(Context.current(), requestBuilder, TEXT_MAP_SETTER);

    // Send the request with the injected context
    httpClient.send(requestBuilder.build(), HttpResponse.BodyHandlers.discarding());
  }

  /** {@link TextMapSetter} with a {@link HttpRequest.Builder} carrier. */
  private static class HttpRequestSetter implements TextMapSetter<HttpRequest.Builder> {
    @Override
    public void set(HttpRequest.Builder carrier, String key, String value) {
      if (carrier == null) {
        return;
      }
      carrier.setHeader(key, value);
    }
  }
}
```
<!-- prettier-ignore-end -->

다음 코드 스니펫은 추출을 위한 `ContextPropagators` API를 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ExtractContextUsage.java"?>
```java
package otel;

import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;
import io.opentelemetry.api.baggage.propagation.W3CBaggagePropagator;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.context.propagation.TextMapGetter;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.util.List;

public class ExtractContextUsage {
  private static final TextMapGetter<HttpExchange> TEXT_MAP_GETTER = new HttpRequestGetter();

  public static void extractContextUsage() throws Exception {
    // Create a ContextPropagators instance which propagates w3c trace context and w3c baggage
    ContextPropagators propagators =
        ContextPropagators.create(
            TextMapPropagator.composite(
                W3CTraceContextPropagator.getInstance(), W3CBaggagePropagator.getInstance()));

    // Create a server, which uses the propagators to extract context from requests
    HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.createContext("/path", new Handler(propagators));
    server.setExecutor(null);
    server.start();
  }

  private static class Handler implements HttpHandler {
    private final ContextPropagators contextPropagators;

    private Handler(ContextPropagators contextPropagators) {
      this.contextPropagators = contextPropagators;
    }

    @Override
    public void handle(HttpExchange exchange) throws IOException {
      // Extract the context from the request and make the context current
      Context extractedContext =
          contextPropagators
              .getTextMapPropagator()
              .extract(Context.current(), exchange, TEXT_MAP_GETTER);
      try (Scope scope = extractedContext.makeCurrent()) {
        // Do work with the extracted context
      } finally {
        String response = "success";
        exchange.sendResponseHeaders(200, response.length());
        OutputStream os = exchange.getResponseBody();
        os.write(response.getBytes(StandardCharsets.UTF_8));
        os.close();
      }
    }
  }

  /** {@link TextMapSetter} with a {@link HttpExchange} carrier. */
  private static class HttpRequestGetter implements TextMapGetter<HttpExchange> {
    @Override
    public Iterable<String> keys(HttpExchange carrier) {
      return carrier.getRequestHeaders().keySet();
    }

    @Override
    public String get(HttpExchange carrier, String key) {
      if (carrier == null) {
        return null;
      }
      List<String> headers = carrier.getRequestHeaders().get(key);
      if (headers == null || headers.isEmpty()) {
        return null;
      }
      return headers.get(0);
    }
  }
}
```
<!-- prettier-ignore-end -->

## OpenTelemetry API {#opentelemetry-api}

`io.opentelemetry:opentelemetry-api:{{% param vers.otel %}}` 아티팩트는
트레이스, 메트릭, 로그, 무동작 구현체, 배기지, 핵심 `TextMapPropagator` 구현체와
[Context API](#context-api)에 대한 의존성을 포함한 오픈텔레메트리 API를 담고
있다.

### 프로바이더(Provider)와 스코프(Scope) {#providers-and-scopes}

프로바이더와 스코프는 오픈텔레메트리 API에서 반복적으로 등장하는 개념이다.
스코프란 텔레메트리가 연관되는 애플리케이션 내의 논리적 단위이다. 프로바이더는
특정 스코프에 상대적인 텔레메트리를 기록하기 위한 구성 요소를 제공한다.

- [TracerProvider](#tracerprovider)는 스팬을 기록하기 위한 스코프가 지정된
  [Tracer](#tracer)를 제공한다.
- [MeterProvider](#meterprovider)는 메트릭을 기록하기 위한 스코프가 지정된
  [Meter](#meter)를 제공한다.
- [LoggerProvider](#loggerprovider)는 로그를 기록하기 위한 스코프가 지정된
  [Logger](#logger)를 제공한다.

> [!WARNING]
>
> {{% param logBridgeWarning %}}

스코프는 (name, version, schemaUrl) 삼중항(triplet)으로 식별된다. 스코프
아이덴티티가 고유하도록 주의를 기울여야 한다. 일반적인 접근 방식은 스코프 이름을
패키지 이름이나 정규화된 클래스 이름으로 설정하고, 스코프 버전을 라이브러리
버전으로 설정하는 것이다. 여러 시그널(즉, 메트릭과 트레이스)에 대해 텔레메트리를
내보내는 경우, 동일한 스코프를 사용해야 한다. 자세한 내용은
[계측 스코프](/docs/concepts/instrumentation-scope/)를 참고한다.

다음 코드 스니펫은 프로바이더 및 스코프 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/ProvidersAndScopes.java"?>
```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.logs.Logger;
import io.opentelemetry.api.logs.LoggerProvider;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.MeterProvider;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.TracerProvider;

public class ProvidersAndScopes {

  private static final String SCOPE_NAME = "fully.qualified.name";
  private static final String SCOPE_VERSION = "1.0.0";
  private static final String SCOPE_SCHEMA_URL = "https://example";

  public static void providersUsage(OpenTelemetry openTelemetry) {
    // Access providers from an OpenTelemetry instance
    TracerProvider tracerProvider = openTelemetry.getTracerProvider();
    MeterProvider meterProvider = openTelemetry.getMeterProvider();
    // NOTE: LoggerProvider is a special case and should only be used to bridge logs from other
    // logging APIs / frameworks into OpenTelemetry.
    LoggerProvider loggerProvider = openTelemetry.getLogsBridge();

    // Access tracer, meter, logger from providers to record telemetry for a particular scope
    Tracer tracer =
        tracerProvider
            .tracerBuilder(SCOPE_NAME)
            .setInstrumentationVersion(SCOPE_VERSION)
            .setSchemaUrl(SCOPE_SCHEMA_URL)
            .build();
    Meter meter =
        meterProvider
            .meterBuilder(SCOPE_NAME)
            .setInstrumentationVersion(SCOPE_VERSION)
            .setSchemaUrl(SCOPE_SCHEMA_URL)
            .build();
    Logger logger =
        loggerProvider
            .loggerBuilder(SCOPE_NAME)
            .setInstrumentationVersion(SCOPE_VERSION)
            .setSchemaUrl(SCOPE_SCHEMA_URL)
            .build();

    // ...optionally, shorthand versions are available if scope version and schemaUrl aren't
    // available
    tracer = tracerProvider.get(SCOPE_NAME);
    meter = meterProvider.get(SCOPE_NAME);
    logger = loggerProvider.get(SCOPE_NAME);
  }
}
```
<!-- prettier-ignore-end -->

### 속성(Attributes) {#attributes}

[Attributes](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/common/Attributes.html)는
[속성 정의](/docs/specs/otel/common/#attribute)를 나타내는 키-값 쌍 묶음이다.
`Attributes`는 오픈텔레메트리 API에서 반복적으로 등장하는 개념이다.

- [Span](#span), 스팬 이벤트, 스팬 링크는 속성을 가진다.
- [메트릭 계측기](#meter)에 기록되는 측정값은 속성을 가진다.
- [LogRecords](#logrecordbuilder)는 속성을 가진다.

시맨틱 컨벤션에서 생성된 속성 상수는 [시맨틱 속성](#semantic-attributes)을
참고한다.

속성 명명에 대한 가이드는 [속성 명명](/docs/specs/semconv/general/naming/)을
참고한다.

다음 코드 스니펫은 `Attributes` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/AttributesUsage.java"?>
```java
package otel;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributesBuilder;
import java.util.Map;

public class AttributesUsage {
  // Establish static constant for attribute keys and reuse to avoid allocations
  private static final AttributeKey<String> SHOP_ID = AttributeKey.stringKey("com.acme.shop.id");
  private static final AttributeKey<String> SHOP_NAME =
      AttributeKey.stringKey("com.acme.shop.name");
  private static final AttributeKey<Long> CUSTOMER_ID =
      AttributeKey.longKey("com.acme.customer.id");
  private static final AttributeKey<String> CUSTOMER_NAME =
      AttributeKey.stringKey("com.acme.customer.name");

  public static void attributesUsage() {
    // Use a varargs initializer and pre-allocated attribute keys. This is the most efficient way to
    // create attributes.
    Attributes attributes =
        Attributes.of(
            SHOP_ID,
            "abc123",
            SHOP_NAME,
            "opentelemetry-demo",
            CUSTOMER_ID,
            123L,
            CUSTOMER_NAME,
            "Jack");

    // ...or use a builder.
    attributes =
        Attributes.builder()
            .put(SHOP_ID, "abc123")
            .put(SHOP_NAME, "opentelemetry-demo")
            .put(CUSTOMER_ID, 123)
            .put(CUSTOMER_NAME, "Jack")
            // Optionally initialize attribute keys on the fly
            .put(AttributeKey.stringKey("com.acme.string-key"), "value")
            .put(AttributeKey.booleanKey("com.acme.bool-key"), true)
            .put(AttributeKey.longKey("com.acme.long-key"), 1L)
            .put(AttributeKey.doubleKey("com.acme.double-key"), 1.1)
            .put(AttributeKey.stringArrayKey("com.acme.string-array-key"), "value1", "value2")
            .put(AttributeKey.booleanArrayKey("come.acme.bool-array-key"), true, false)
            .put(AttributeKey.longArrayKey("come.acme.long-array-key"), 1L, 2L)
            .put(AttributeKey.doubleArrayKey("come.acme.double-array-key"), 1.1, 2.2)
            // Optionally omit initializing AttributeKey
            .put("com.acme.string-key", "value")
            .put("com.acme.bool-key", true)
            .put("come.acme.long-key", 1L)
            .put("come.acme.double-key", 1.1)
            .put("come.acme.string-array-key", "value1", "value2")
            .put("come.acme.bool-array-key", true, false)
            .put("come.acme.long-array-key", 1L, 2L)
            .put("come.acme.double-array-key", 1.1, 2.2)
            .build();

    // Attributes has a variety of methods for manipulating and reading data.
    // Read an attribute key:
    String shopIdValue = attributes.get(SHOP_ID);
    // Inspect size:
    int size = attributes.size();
    boolean isEmpty = attributes.isEmpty();
    // Convert to a map representation:
    Map<AttributeKey<?>, Object> map = attributes.asMap();
    // Iterate through entries, printing each to the template: <key> (<type>): <value>\n
    attributes.forEach(
        (attributeKey, value) ->
            System.out.printf(
                "%s (%s): %s%n", attributeKey.getKey(), attributeKey.getType(), value));
    // Convert to a builder, remove the com.acme.customer.id and any entry whose key starts with
    // com.acme.shop, and build a new instance:
    AttributesBuilder builder = attributes.toBuilder();
    builder.remove(CUSTOMER_ID);
    builder.removeIf(attributeKey -> attributeKey.getKey().startsWith("com.acme.shop"));
    Attributes trimmedAttributes = builder.build();
  }
}
```
<!-- prettier-ignore-end -->

### OpenTelemetry {#opentelemetry}

> **표기 규칙**: 이 절의 `OpenTelemetry`는 오픈텔레메트리 프로젝트 전체가
> 아니라, 최상위 API 구성 요소를 담는 자바 API 타입 `OpenTelemetry`를 가리킨다.

> [!NOTE] Spring Boot Starter
>
> Spring Boot 스타터는 `OpenTelemetry`를 Spring 빈(bean)으로 사용할 수 있는
> 특별한 경우이다. `OpenTelemetry`를 Spring 컴포넌트에 그대로 주입하면 된다.
>
> 자세한 내용은
> [커스텀 수동 계측으로 Spring Boot 스타터 확장하기](/docs/zero-code/java/spring-boot-starter/api/)를
> 참고한다.

[OpenTelemetry](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/OpenTelemetry.html)는
최상위 API 구성 요소를 담는 홀더로, 계측에 전달하기 편리하다.

`OpenTelemetry`는 다음으로 구성된다.

- [TracerProvider](#tracerprovider): 트레이스를 위한 API 진입점이다.
- [MeterProvider](#meterprovider): 메트릭을 위한 API 진입점이다.
- [LoggerProvider](#loggerprovider): 로그를 위한 API 진입점이다.
- [ContextPropagators](#contextpropagators): 컨텍스트 전파를 위한 API
  진입점이다.

다음 코드 스니펫은 `OpenTelemetry` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/OpenTelemetryUsage.java"?>
```java
package otel;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.logs.LoggerProvider;
import io.opentelemetry.api.metrics.MeterProvider;
import io.opentelemetry.api.trace.TracerProvider;
import io.opentelemetry.context.propagation.ContextPropagators;

public class OpenTelemetryUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void openTelemetryUsage(OpenTelemetry openTelemetry) {
    // Access TracerProvider, MeterProvider, LoggerProvider, ContextPropagators
    TracerProvider tracerProvider = openTelemetry.getTracerProvider();
    MeterProvider meterProvider = openTelemetry.getMeterProvider();
    LoggerProvider loggerProvider = openTelemetry.getLogsBridge();
    ContextPropagators propagators = openTelemetry.getPropagators();
  }
}
```
<!-- prettier-ignore-end -->

### GlobalOpenTelemetry {#globalopentelemetry}

> [!NOTE] Java 에이전트
>
> Java 에이전트는 에이전트가 `GlobalOpenTelemetry`를 설정하는 특별한 경우이다.
> `GlobalOpenTelemetry.getOrNoop()`을 호출하기만 하면 `OpenTelemetry` 인스턴스에
> 접근할 수 있다.
>
> 자세한 내용은
> [커스텀 수동 계측으로 Java 에이전트 확장하기](/docs/zero-code/java/agent/api/)를
> 참고한다.

[GlobalOpenTelemetry](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/GlobalOpenTelemetry.html)는
전역 싱글턴 [OpenTelemetry](#opentelemetry) 인스턴스를 보관한다.

`GlobalOpenTelemetry`는 초기화 순서 문제를 피하기 위해 아주 특별한 방식으로
설계되었으며, 그 결과 신중하게 사용해야 한다. 구체적으로,
`GlobalOpenTelemetry.set(..)`이 호출되었는지 여부와 관계없이
`GlobalOpenTelemetry.get()`은 항상 동일한 결과를 반환한다. 내부적으로
`set()`보다 먼저 `get()`이 호출되면, 구현체는 내부적으로
[무동작 구현체](#no-op-implementation)로 `set(..)`을 호출하고 이를 반환한다.
`set(..)`은 두 번 이상 호출되면 예외를 발생시키므로, `get()` 이후에 `set(..)`을
호출하면 조용히 실패하는 대신 예외가 발생한다.

Java 에이전트는 특별한 경우로,
[네이티브 계측](../instrumentation/#native-instrumentation)과
[수동 계측](../instrumentation/#manual-instrumentation)이 에이전트가 설치한
`OpenTelemetry` 인스턴스에 텔레메트리를 기록할 수 있는 유일한 메커니즘이
`GlobalOpenTelemetry`이다. 이 인스턴스를 사용하는 것은 중요하고 유용하며, 다음과
같이 `GlobalOpenTelemetry`에 접근할 것을 권장한다.

**네이티브 계측의 경우, 기본값으로 `GlobalOpenTelemetry.getOrNoop()`을
사용한다.**

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/GlobalOpenTelemetryNativeInstrumentationUsage.java"?>
```java
package otel;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.OpenTelemetry;

public class GlobalOpenTelemetryNativeInstrumentationUsage {

  public static void globalOpenTelemetryUsage(OpenTelemetry openTelemetry) {
    // Initialized with OpenTelemetry from java agent if present, otherwise no-op implementation.
    MyClient client1 = new MyClientBuilder().build();

    // Initialized with an explicit OpenTelemetry instance, overriding the java agent instance.
    MyClient client2 = new MyClientBuilder().setOpenTelemetry(openTelemetry).build();
  }

  /**
   * An example library with native OpenTelemetry instrumentation, initialized via {@link
   * MyClientBuilder}.
   */
  public static class MyClient {
    private final OpenTelemetry openTelemetry;

    private MyClient(OpenTelemetry openTelemetry) {
      this.openTelemetry = openTelemetry;
    }

    // ... library methods omitted
  }

  /** Builder for {@link MyClient}. */
  public static class MyClientBuilder {
    // OpenTelemetry defaults to the GlobalOpenTelemetry instance if set, e.g. by the java agent or
    // by the application, else to a no-op implementation.
    private OpenTelemetry openTelemetry = GlobalOpenTelemetry.getOrNoop();

    /** Explicitly set the OpenTelemetry instance to use. */
    public MyClientBuilder setOpenTelemetry(OpenTelemetry openTelemetry) {
      this.openTelemetry = openTelemetry;
      return this;
    }

    /** Build the client. */
    public MyClient build() {
      return new MyClient(openTelemetry);
    }
  }
}
```
<!-- prettier-ignore-end -->

`GlobalOpenTelemetry.getOrNoop()`은 `get()`이 `set(..)`을 호출하는 부작용 없이
설계되어, 애플리케이션 코드가 나중에 예외를 일으키지 않고 `set(..)`을 호출할 수
있는 능력을 그대로 유지한다는 점에 유의한다.

그 결과:

- Java 에이전트가 있으면, 계측은 기본적으로 에이전트가 설치한 `OpenTelemetry`
  인스턴스로 초기화된다.
- Java 에이전트가 없으면, 계측은 기본적으로 무동작 구현체로 초기화된다.
- 사용자는 별도의 인스턴스로 `setOpenTelemetry(..)`를 호출하여 기본값을
  명시적으로 재정의할 수 있다.

**수동 계측의 경우, 기본값으로 다음을 사용한다.**

<!-- prettier-ignore-start -->
<!-- temporarily change except path to resolve relative to configuration directory, and revert after -->
<?code-excerpt path-base="examples/java/configuration"?>
<?code-excerpt "src/main/java/otel/GlobalOpenTelemetryManualInstrumentationUsage.java"?>
```java
package otel;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;

public class GlobalOpenTelemetryManualInstrumentationUsage {

  public static void globalOpenTelemetryUsage() {
    // If GlobalOpenTelemetry is already set, e.g. by the java agent, use it.
    // Else, initialize an OpenTelemetry SDK instance and use it.
    OpenTelemetry openTelemetry =
        GlobalOpenTelemetry.isSet() ? GlobalOpenTelemetry.get() : initializeOpenTelemetry();

    // Install into manual instrumentation. This may involve setting as a singleton in the
    // application's dependency injection framework.
  }

  /** Initialize OpenTelemetry SDK using autoconfiguration. */
  public static OpenTelemetry initializeOpenTelemetry() {
    return AutoConfiguredOpenTelemetrySdk.initialize().getOpenTelemetrySdk();
  }
}
```
<?code-excerpt path-base="examples/java/api"?>
<!-- prettier-ignore-end -->

그 결과:

- Java 에이전트가 있으면, 애플리케이션은 에이전트가 설치한 `OpenTelemetry`
  인스턴스로 수동 계측을 초기화한다.
- Java 에이전트가 없으면, 애플리케이션은
  [OpenTelemetrySdk](../sdk/#opentelemetrysdk) 인스턴스를 초기화하고 이를 사용해
  수동 계측을 초기화한다.

### 트레이서 프로바이더(TracerProvider) {#tracerprovider}

[TracerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/trace/TracerProvider.html)는
트레이스를 위한 API 진입점이며 [Tracer](#tracer)를 제공한다. 프로바이더와
스코프에 대한 정보는 [프로바이더와 스코프](#providers-and-scopes)를 참고한다.

#### 트레이서(Tracer) {#tracer}

[Tracer](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/trace/Tracer.html)는
계측 스코프에 대해 [스팬을 기록](#span)하는 데 사용된다. 프로바이더와 스코프에
대한 정보는 [프로바이더와 스코프](#providers-and-scopes)를 참고한다.

#### 스팬(Span) {#span}

[SpanBuilder](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/trace/SpanBuilder.html)와
[Span](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/trace/Span.html)은
스팬에 데이터를 구성하고 기록하는 데 사용된다.

`SpanBuilder`는 `Span startSpan()`을 호출해 시작하기 전에 스팬에 데이터를
추가하는 데 사용된다. 시작한 후에는 다양한 `Span` 업데이트 메서드를 호출하여
데이터를 추가/갱신할 수 있다. 시작 전에 `SpanBuilder`에 제공된 데이터는
[Sampler](../sdk/#sampler)에 입력값으로 제공된다.

다음 코드 스니펫은 `SpanBuilder`/`Span` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SpanUsage.java"?>
```java
package otel;

import static io.opentelemetry.context.Context.current;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanContext;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import java.util.Arrays;

public class SpanUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void spanUsage(Tracer tracer) {
    // Get a span builder by providing the span name
    Span span =
        tracer
            .spanBuilder("span name")
            // Set span kind
            .setSpanKind(SpanKind.INTERNAL)
            // Set attributes
            .setAttribute(AttributeKey.stringKey("com.acme.string-key"), "value")
            .setAttribute(AttributeKey.booleanKey("com.acme.bool-key"), true)
            .setAttribute(AttributeKey.longKey("com.acme.long-key"), 1L)
            .setAttribute(AttributeKey.doubleKey("com.acme.double-key"), 1.1)
            .setAttribute(
                AttributeKey.stringArrayKey("com.acme.string-array-key"),
                Arrays.asList("value1", "value2"))
            .setAttribute(
                AttributeKey.booleanArrayKey("come.acme.bool-array-key"),
                Arrays.asList(true, false))
            .setAttribute(
                AttributeKey.longArrayKey("come.acme.long-array-key"), Arrays.asList(1L, 2L))
            .setAttribute(
                AttributeKey.doubleArrayKey("come.acme.double-array-key"), Arrays.asList(1.1, 2.2))
            // Optionally omit initializing AttributeKey
            .setAttribute("com.acme.string-key", "value")
            .setAttribute("com.acme.bool-key", true)
            .setAttribute("come.acme.long-key", 1L)
            .setAttribute("come.acme.double-key", 1.1)
            .setAllAttributes(WIDGET_RED_CIRCLE)
            // Uncomment to optionally explicitly set the parent span context. If omitted, the
            // span's parent will be set using Context.current().
            // .setParent(parentContext)
            // Uncomment to optionally add links.
            // .addLink(linkContext, linkAttributes)
            // Start the span
            .startSpan();

    // Check if span is recording before computing additional data
    if (span.isRecording()) {
      // Update the span name with information not available when starting
      span.updateName("new span name");

      // Add additional attributes not available when starting
      span.setAttribute("com.acme.string-key2", "value");

      // Add additional span links not available when starting
      span.addLink(exampleLinkContext());
      // optionally include attributes on the link
      span.addLink(exampleLinkContext(), WIDGET_RED_CIRCLE);

      // Add span events
      span.addEvent("my-event");
      // optionally include attributes on the event
      span.addEvent("my-event", WIDGET_RED_CIRCLE);

      // Record exception, syntactic sugar for a span event with a specific shape
      span.recordException(new RuntimeException("error"));

      // Set the span status
      span.setStatus(StatusCode.OK, "status description");
    }

    // Finally, end the span
    span.end();
  }

  /** Return a dummy link context. */
  private static SpanContext exampleLinkContext() {
    return Span.fromContext(current()).getSpanContext();
  }
}
```
<!-- prettier-ignore-end -->

스팬 부모 지정(parenting)은 트레이싱의 중요한 측면이다. 각 스팬에는 선택적인
부모가 있다. 트레이스에 있는 모든 스팬을 모으고 각 스팬의 부모를 따라가면 계층
구조를 구성할 수 있다. 스팬 API는 [컨텍스트](#context) 위에 구축되며, 이를 통해
스팬 컨텍스트가 애플리케이션 전체와 스레드를 넘어 암묵적으로 전달될 수 있다.
스팬이 생성되면, 스팬이 없거나 컨텍스트가 명시적으로 재정의되지 않는 한 그
부모는 `Context.current()`에 있는 스팬으로 설정된다.

컨텍스트 API 사용에 대한 대부분의 안내는 스팬에도 적용된다. 스팬 컨텍스트는
[W3CTraceContextPropagator](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/trace/propagation/W3CTraceContextPropagator.html)와
그 외 [TextMapPropagator](../sdk/#textmappropagator)를 통해 애플리케이션 경계를
넘어 전파된다.

다음 코드 스니펫은 `Span` API의 컨텍스트 전파를 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SpanAndContextUsage.java"?>
```java
package otel;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;

public class SpanAndContextUsage {
  private final Tracer tracer;

  SpanAndContextUsage(Tracer tracer) {
    this.tracer = tracer;
  }

  public void nestedSpanUsage() {
    // Start a span. Since we don't call makeCurrent(), we must explicitly call setParent on
    // children. Wrap code in try / finally to ensure we end the span.
    Span span = tracer.spanBuilder("span").startSpan();
    try {
      // Start a child span, explicitly setting the parent.
      Span childSpan =
          tracer
              .spanBuilder("span child")
              // Explicitly set parent.
              .setParent(span.storeInContext(Context.current()))
              .startSpan();
      // Call makeCurrent(), adding childSpan to Context.current(). Spans created inside the scope
      // will have their parent set to childSpan.
      try (Scope childSpanScope = childSpan.makeCurrent()) {
        // Call another method which creates a span. The span's parent will be childSpan since it is
        // started in the childSpan scope.
        doWork();
      } finally {
        childSpan.end();
      }
    } finally {
      span.end();
    }
  }

  private int doWork() {
    Span doWorkSpan = tracer.spanBuilder("doWork").startSpan();
    try (Scope scope = doWorkSpan.makeCurrent()) {
      int result = 0;
      for (int i = 0; i < 10; i++) {
        result += i;
      }
      return result;
    } finally {
      doWorkSpan.end();
    }
  }
}
```
<!-- prettier-ignore-end -->

### 미터 프로바이더(MeterProvider) {#meterprovider}

[MeterProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/MeterProvider.html)는
메트릭을 위한 API 진입점이며 [Meter](#meter)를 제공한다. 프로바이더와 스코프에
대한 정보는 [프로바이더와 스코프](#providers-and-scopes)를 참고한다.

#### 미터(Meter) {#meter}

[Meter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/Meter.html)는
특정 [계측 스코프](#providers-and-scopes)에 대한 계측기(instrument)를 얻는 데
사용된다. 프로바이더와 스코프에 대한 정보는
[프로바이더와 스코프](#providers-and-scopes)를 참고한다. 계측기는 여러 종류가
있으며, 각각 SDK에서 서로 다른 시맨틱과 기본 동작을 가진다. 각 사용 사례에 맞는
올바른 계측기를 선택하는 것이 중요하다.

| 계측기(Instrument)                          | 동기/비동기 | 설명                                                                              | 예시                                     | 기본 SDK 집계(Aggregation)                                                                     |
| ------------------------------------------- | ----------- | --------------------------------------------------------------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| [Counter](#counter)                         | 동기        | 단조(monotonic)(양의) 값을 기록한다.                                              | 사용자 로그인 기록                       | [sum (monotonic=true)](/docs/specs/otel/metrics/sdk/#sum-aggregation)                          |
| [Async Counter](#async-counter)             | 비동기      | 단조 합계를 관측한다.                                                             | JVM에서 로드된 클래스 수 관측            | [sum (monotonic=true)](/docs/specs/otel/metrics/sdk/#sum-aggregation)                          |
| [UpDownCounter](#updowncounter)             | 동기        | 비단조(양수와 음수) 값을 기록한다.                                                | 큐에 항목이 추가되거나 제거될 때 기록    | [sum (monotonic=false)](/docs/specs/otel/metrics/sdk/#sum-aggregation)                         |
| [Async UpDownCounter](#async-updowncounter) | 비동기      | 비단조 합계를 관측한다.                                                           | JVM 메모리 풀 사용량 관측                | [sum (monotonic=false)](/docs/specs/otel/metrics/sdk/#sum-aggregation)                         |
| [Histogram](#histogram)                     | 동기        | 분포가 중요한 경우, 단조(양의) 값을 기록한다.                                     | 서버가 처리한 HTTP 요청의 처리 시간 기록 | [ExplicitBucketHistogram](/docs/specs/otel/metrics/sdk/#explicit-bucket-histogram-aggregation) |
| [Gauge](#gauge)                             | 동기        | 공간 재집계(spatial re-aggregation)가 의미 없는 경우 **[1]**, 최신 값을 기록한다. | 온도 기록                                | [LastValue](/docs/specs/otel/metrics/sdk/#last-value-aggregation)                              |
| [Async Gauge](#async-gauge)                 | 비동기      | 공간 재집계가 의미 없는 경우 **[1]**, 최신 값을 관측한다.                         | CPU 사용률 관측                          | [LastValue](/docs/specs/otel/metrics/sdk/#last-value-aggregation)                              |

**[1]**: 공간 재집계란 필요하지 않은 속성을 제거하여 속성 스트림을 병합하는
과정이다. 예를 들어, 속성이 `{"color": "red", "shape": "square"}`,
`{"color": "blue", "shape": "square"}`인 시리즈가 있을 때, `color` 속성을
제거하고 `color`를 제거한 후 속성이 같은 시리즈를 병합함으로써 공간 재집계를
수행할 수 있다. 대부분의 집계에는 유용한 공간 집계 병합 함수가 있지만(즉, 합은
서로 더해짐), `LastValue` 집계로 집계되는 게이지는 예외이다. 예를 들어, 앞서
언급한 시리즈가 위젯의 온도를 추적한다고 하자. `color` 속성을 제거할 때 두
시리즈를 어떻게 병합해야 할까? 동전을 던져 무작위 값을 선택하는 것 외에는 좋은
답이 없다.

계측기 API는 다양한 기능을 공유한다.

- 빌더 패턴을 사용해 생성된다.
- 계측기 이름이 필수이다.
- 단위와 설명은 선택 사항이다.
- 빌더를 통해 구성되는 `long` 또는 `double` 값을 기록한다.

메트릭 명명과 단위에 대한 자세한 내용은
[메트릭 가이드라인](/docs/specs/semconv/general/metrics/#general-guidelines)을
참고한다.

계측기 선택에 대한 추가 안내는
[계측 라이브러리 작성자를 위한 가이드라인](/docs/specs/otel/metrics/supplementary-guidelines/#guidelines-for-instrumentation-library-authors)을
참고한다.

#### Counter(카운터) {#counter}

[LongCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/LongCounter.html)와
[DoubleCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/DoubleCounter.html)는
단조(양의) 값을 기록하는 데 사용된다.

다음 코드 스니펫은 카운터 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/CounterUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;
import static otel.Util.customContext;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.Meter;

public class CounterUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void counterUsage(Meter meter) {
    // Construct a counter to record measurements that are always positive (monotonically
    // increasing).
    LongCounter counter =
        meter
            .counterBuilder("fully.qualified.counter")
            .setDescription("A count of produced widgets")
            .setUnit("{widget}")
            // optionally change the type to double
            // .ofDoubles()
            .build();

    // Record a measurement with no attributes or context.
    // Attributes defaults to Attributes.empty(), context to Context.current().
    counter.add(1L);

    // Record a measurement with attributes, using pre-allocated attributes whenever possible.
    counter.add(1L, WIDGET_RED_CIRCLE);
    // Sometimes, attributes must be computed using application context.
    counter.add(
        1L, Attributes.of(WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));

    // Record a measurement with attributes, and context.
    // Most users will opt to omit the context argument, preferring the default Context.current().
    counter.add(1L, WIDGET_RED_CIRCLE, customContext());
  }
}
```
<!-- prettier-ignore-end -->

#### Async Counter(비동기 카운터) {#async-counter}

[ObservableLongCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableLongCounter.html)와
[ObservableDoubleCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableDoubleCounter.html)는
단조(양의) 합계를 관측하는 데 사용된다.

다음 코드 스니펫은 비동기 카운터 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/AsyncCounterUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.ObservableLongCounter;
import java.util.concurrent.atomic.AtomicLong;

public class AsyncCounterUsage {
  // Pre-allocate attributes whenever possible
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void asyncCounterUsage(Meter meter) {
    AtomicLong widgetCount = new AtomicLong();

    // Construct an async counter to observe an existing counter in a callback
    ObservableLongCounter asyncCounter =
        meter
            .counterBuilder("fully.qualified.counter")
            .setDescription("A count of produced widgets")
            .setUnit("{widget}")
            // Uncomment to optionally change the type to double
            // .ofDoubles()
            .buildWithCallback(
                // the callback is invoked when a MetricReader reads metrics
                observableMeasurement -> {
                  long currentWidgetCount = widgetCount.get();

                  // Record a measurement with no attributes.
                  // Attributes defaults to Attributes.empty().
                  observableMeasurement.record(currentWidgetCount);

                  // Record a measurement with attributes, using pre-allocated attributes whenever
                  // possible.
                  observableMeasurement.record(currentWidgetCount, WIDGET_RED_CIRCLE);
                  // Sometimes, attributes must be computed using application context.
                  observableMeasurement.record(
                      currentWidgetCount,
                      Attributes.of(
                          WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));
                });

    // Optionally close the counter to unregister the callback when required
    asyncCounter.close();
  }
}
```
<!-- prettier-ignore-end -->

#### UpDownCounter(업다운카운터) {#updowncounter}

[LongUpDownCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/LongUpDownCounter.html)와
[DoubleUpDownCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/DoubleUpDownCounter.html)는
비단조(양수와 음수) 값을 기록하는 데 사용된다.

다음 코드 스니펫은 updowncounter API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/UpDownCounterUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;
import static otel.Util.customContext;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongUpDownCounter;
import io.opentelemetry.api.metrics.Meter;

public class UpDownCounterUsage {

  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void usage(Meter meter) {
    // Construct an updowncounter to record measurements that go up and down.
    LongUpDownCounter upDownCounter =
        meter
            .upDownCounterBuilder("fully.qualified.updowncounter")
            .setDescription("Current length of widget processing queue")
            .setUnit("{widget}")
            // Uncomment to optionally change the type to double
            // .ofDoubles()
            .build();

    // Record a measurement with no attributes or context.
    // Attributes defaults to Attributes.empty(), context to Context.current().
    upDownCounter.add(1L);

    // Record a measurement with attributes, using pre-allocated attributes whenever possible.
    upDownCounter.add(-1L, WIDGET_RED_CIRCLE);
    // Sometimes, attributes must be computed using application context.
    upDownCounter.add(
        -1L, Attributes.of(WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));

    // Record a measurement with attributes, and context.
    // Most users will opt to omit the context argument, preferring the default Context.current().
    upDownCounter.add(1L, WIDGET_RED_CIRCLE, customContext());
  }
}
```
<!-- prettier-ignore-end -->

#### Async UpDownCounter(비동기 업다운카운터) {#async-updowncounter}

[ObservableLongUpDownCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableLongUpDownCounter.html)와
[ObservableDoubleUpDownCounter](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableDoubleUpDownCounter.html)는
비단조(양수와 음수) 합계를 관측하는 데 사용된다.

다음 코드 스니펫은 비동기 updowncounter API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/AsyncUpDownCounterUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.ObservableLongUpDownCounter;
import java.util.concurrent.atomic.AtomicLong;

public class AsyncUpDownCounterUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void asyncUpDownCounterUsage(Meter meter) {
    AtomicLong queueLength = new AtomicLong();

    // Construct an async updowncounter to observe an existing up down counter in a callback
    ObservableLongUpDownCounter asyncUpDownCounter =
        meter
            .upDownCounterBuilder("fully.qualified.updowncounter")
            .setDescription("Current length of widget processing queue")
            .setUnit("{widget}")
            // Uncomment to optionally change the type to double
            // .ofDoubles()
            .buildWithCallback(
                // the callback is invoked when a MetricReader reads metrics
                observableMeasurement -> {
                  long currentWidgetCount = queueLength.get();

                  // Record a measurement with no attributes.
                  // Attributes defaults to Attributes.empty().
                  observableMeasurement.record(currentWidgetCount);

                  // Record a measurement with attributes, using pre-allocated attributes whenever
                  // possible.
                  observableMeasurement.record(currentWidgetCount, WIDGET_RED_CIRCLE);
                  // Sometimes, attributes must be computed using application context.
                  observableMeasurement.record(
                      currentWidgetCount,
                      Attributes.of(
                          WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));
                });

    // Optionally close the counter to unregister the callback when required
    asyncUpDownCounter.close();
  }
}
```
<!-- prettier-ignore-end -->

#### Histogram(히스토그램) {#histogram}

[DoubleHistogram](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/DoubleHistogram.html)과
[LongHistogram](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/LongHistogram.html)은
분포가 중요한 경우 단조(양의) 값을 기록하는 데 사용된다.

다음 코드 스니펫은 히스토그램 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/HistogramUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;
import static otel.Util.customContext;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;

public class HistogramUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void histogramUsage(Meter meter) {
    // Construct a histogram to record measurements where the distribution is important.
    DoubleHistogram histogram =
        meter
            .histogramBuilder("fully.qualified.histogram")
            .setDescription("Length of time to process a widget")
            .setUnit("s")
            // Uncomment to optionally provide advice on useful default explicit bucket boundaries
            // .setExplicitBucketBoundariesAdvice(Arrays.asList(1.0, 2.0, 3.0))
            // Uncomment to optionally change the type to long
            // .ofLongs()
            .build();

    // Record a measurement with no attributes or context.
    // Attributes defaults to Attributes.empty(), context to Context.current().
    histogram.record(1.1);

    // Record a measurement with attributes, using pre-allocated attributes whenever possible.
    histogram.record(2.2, WIDGET_RED_CIRCLE);
    // Sometimes, attributes must be computed using application context.
    histogram.record(
        3.2, Attributes.of(WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));

    // Record a measurement with attributes, and context.
    // Most users will opt to omit the context argument, preferring the default Context.current().
    histogram.record(4.4, WIDGET_RED_CIRCLE, customContext());
  }
}
```
<!-- prettier-ignore-end -->

#### Gauge(게이지) {#gauge}

[DoubleGauge](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/DoubleGauge.html)와
[LongGauge](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/LongGauge.html)는
공간 재집계가 의미 없는 경우 최신 값을 기록하는 데 사용된다.

다음 코드 스니펫은 게이지 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/GaugeUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;
import static otel.Util.customContext;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleGauge;
import io.opentelemetry.api.metrics.Meter;

public class GaugeUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void gaugeUsage(Meter meter) {
    // Construct a gauge to record measurements as they occur, which cannot be spatially
    // re-aggregated.
    DoubleGauge gauge =
        meter
            .gaugeBuilder("fully.qualified.gauge")
            .setDescription("The current temperature of the widget processing line")
            .setUnit("K")
            // Uncomment to optionally change the type to long
            // .ofLongs()
            .build();

    // Record a measurement with no attributes or context.
    // Attributes defaults to Attributes.empty(), context to Context.current().
    gauge.set(273.0);

    // Record a measurement with attributes, using pre-allocated attributes whenever possible.
    gauge.set(273.0, WIDGET_RED_CIRCLE);
    // Sometimes, attributes must be computed using application context.
    gauge.set(
        273.0,
        Attributes.of(WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));

    // Record a measurement with attributes, and context.
    // Most users will opt to omit the context argument, preferring the default Context.current().
    gauge.set(1L, WIDGET_RED_CIRCLE, customContext());
  }
}
```
<!-- prettier-ignore-end -->

#### Async Gauge(비동기 게이지) {#async-gauge}

[ObservableDoubleGauge](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableDoubleGauge.html)와
[ObservableLongGauge](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/metrics/ObservableLongGauge.html)는
공간 재집계가 의미 없는 경우 최신 값을 관측하는 데 사용된다.

다음 코드 스니펫은 비동기 게이지 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/AsyncGaugeUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_SHAPE;
import static otel.Util.computeWidgetColor;
import static otel.Util.computeWidgetShape;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.ObservableDoubleGauge;
import java.util.concurrent.atomic.AtomicReference;

public class AsyncGaugeUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void asyncGaugeUsage(Meter meter) {
    AtomicReference<Double> processingLineTemp = new AtomicReference<>(273.0);

    // Construct an async gauge to observe an existing gauge in a callback
    ObservableDoubleGauge asyncGauge =
        meter
            .gaugeBuilder("fully.qualified.gauge")
            .setDescription("The current temperature of the widget processing line")
            .setUnit("K")
            // Uncomment to optionally change the type to long
            // .ofLongs()
            .buildWithCallback(
                // the callback is invoked when a MetricReader reads metrics
                observableMeasurement -> {
                  double currentWidgetCount = processingLineTemp.get();

                  // Record a measurement with no attributes.
                  // Attributes defaults to Attributes.empty().
                  observableMeasurement.record(currentWidgetCount);

                  // Record a measurement with attributes, using pre-allocated attributes whenever
                  // possible.
                  observableMeasurement.record(currentWidgetCount, WIDGET_RED_CIRCLE);
                  // Sometimes, attributes must be computed using application context.
                  observableMeasurement.record(
                      currentWidgetCount,
                      Attributes.of(
                          WIDGET_SHAPE, computeWidgetShape(), WIDGET_COLOR, computeWidgetColor()));
                });

    // Optionally close the gauge to unregister the callback when required
    asyncGauge.close();
  }
}
```
<!-- prettier-ignore-end -->

### 로거 프로바이더(LoggerProvider) {#loggerprovider}

[LoggerProvider](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/logs/LoggerProvider.html)는
로그를 위한 API 진입점이며 [Logger](#logger)를 제공한다. 프로바이더와 스코프에
대한 정보는 [프로바이더와 스코프](#providers-and-scopes)를 참고한다.

> [!WARNING]
>
> {{% param logBridgeWarning %}}

#### 로거(Logger) {#logger}

[Logger](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/logs/Logger.html)는
계측 스코프에 대해 [로그 레코드를 내보내는](#logrecordbuilder) 데 사용된다.
프로바이더와 스코프에 대한 정보는 [프로바이더와 스코프](#providers-and-scopes)를
참고한다.

#### LogRecordBuilder {#logrecordbuilder}

[LogRecordBuilder](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/logs/LogRecordBuilder.html)는
로그 레코드를 구성하고 내보내는 데 사용된다.

다음 코드 스니펫은 `LogRecordBuilder` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/LogRecordUsage.java"?>
```java
package otel;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.Value;
import io.opentelemetry.api.logs.Logger;
import io.opentelemetry.api.logs.Severity;
import java.util.Arrays;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class LogRecordUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void logRecordUsage(Logger logger) {
    logger
        .logRecordBuilder()
        // Set body. Note, setBody(..) is called multiple times for demonstration purposes but only
        // the last call is used.
        // Set the body to a string, syntactic sugar for setBody(Value.of("log message"))
        .setBody("log message")
        // Optionally set the body to a Value to record arbitrarily complex structured data
        .setBody(Value.of("log message"))
        .setBody(Value.of(1L))
        .setBody(Value.of(1.1))
        .setBody(Value.of(true))
        .setBody(Value.of(new byte[] {'a', 'b', 'c'}))
        .setBody(Value.of(Value.of("entry1"), Value.of("entry2")))
        .setBody(
            Value.of(
                Map.of(
                    "stringKey",
                    Value.of("entry1"),
                    "mapKey",
                    Value.of(Map.of("stringKey", Value.of("entry2"))))))
        // Set severity
        .setSeverity(Severity.DEBUG)
        .setSeverityText("debug")
        // Set timestamp
        .setTimestamp(System.currentTimeMillis(), TimeUnit.MILLISECONDS)
        // Optionally set the timestamp when the log was observed
        .setObservedTimestamp(System.currentTimeMillis(), TimeUnit.MILLISECONDS)
        // Set attributes
        .setAttribute(AttributeKey.stringKey("com.acme.string-key"), "value")
        .setAttribute(AttributeKey.booleanKey("com.acme.bool-key"), true)
        .setAttribute(AttributeKey.longKey("com.acme.long-key"), 1L)
        .setAttribute(AttributeKey.doubleKey("com.acme.double-key"), 1.1)
        .setAttribute(
            AttributeKey.stringArrayKey("com.acme.string-array-key"),
            Arrays.asList("value1", "value2"))
        .setAttribute(
            AttributeKey.booleanArrayKey("come.acme.bool-array-key"), Arrays.asList(true, false))
        .setAttribute(AttributeKey.longArrayKey("come.acme.long-array-key"), Arrays.asList(1L, 2L))
        .setAttribute(
            AttributeKey.doubleArrayKey("come.acme.double-array-key"), Arrays.asList(1.1, 2.2))
        .setAllAttributes(WIDGET_RED_CIRCLE)
        // Uncomment to optionally explicitly set the context used to correlate with spans. If
        // omitted, Context.current() is used.
        // .setContext(context)
        // Emit the log record
        .emit();
  }
}
```
<!-- prettier-ignore-end -->

### 무동작(No-op) 구현체 {#no-op-implementation}

`OpenTelemetry#noop()` 메서드는 [OpenTelemetry](#opentelemetry)와 그것이 접근할
수 있게 해주는 모든 API 구성 요소의 무동작 구현체에 대한 접근을 제공한다.
이름에서 알 수 있듯이, 무동작 구현체는 아무 일도 하지 않으며 성능에 영향을 주지
않도록 설계되었다. 계측이 속성 값이나 텔레메트리를 기록하는 데 필요한 다른
데이터를 계산/할당하고 있다면, 무동작 구현체를 사용하더라도 성능에 영향이 있을
수 있다. 무동작 구현체는 사용자가 [SDK](../sdk/)와 같은 구체적인 구현체를
구성하고 설치하지 않았을 때 유용한 `OpenTelemetry`의 기본 인스턴스이다.

다음 코드 스니펫은 `OpenTelemetry#noop()` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/NoopUsage.java"?>
```java
package otel;

import static otel.Util.WIDGET_COLOR;
import static otel.Util.WIDGET_RED_CIRCLE;
import static otel.Util.WIDGET_SHAPE;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.logs.Logger;
import io.opentelemetry.api.logs.Severity;
import io.opentelemetry.api.metrics.DoubleGauge;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.LongUpDownCounter;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;

public class NoopUsage {
  private static final String SCOPE_NAME = "fully.qualified.name";

  public static void noopUsage() {
    // Access the no-op OpenTelemetry instance
    OpenTelemetry noopOpenTelemetry = OpenTelemetry.noop();

    // No-op tracing
    Tracer noopTracer = OpenTelemetry.noop().getTracer(SCOPE_NAME);
    noopTracer
        .spanBuilder("span name")
        .startSpan()
        .setAttribute(WIDGET_SHAPE, "square")
        .setStatus(StatusCode.OK)
        .addEvent("event-name", Attributes.builder().put(WIDGET_COLOR, "red").build())
        .end();

    // No-op metrics
    Attributes attributes = WIDGET_RED_CIRCLE;
    Meter noopMeter = OpenTelemetry.noop().getMeter(SCOPE_NAME);
    DoubleHistogram histogram = noopMeter.histogramBuilder("fully.qualified.histogram").build();
    histogram.record(1.0, attributes);
    // counter
    LongCounter counter = noopMeter.counterBuilder("fully.qualified.counter").build();
    counter.add(1, attributes);
    // async counter
    noopMeter
        .counterBuilder("fully.qualified.counter")
        .buildWithCallback(observable -> observable.record(10, attributes));
    // updowncounter
    LongUpDownCounter upDownCounter =
        noopMeter.upDownCounterBuilder("fully.qualified.updowncounter").build();
    // async updowncounter
    noopMeter
        .upDownCounterBuilder("fully.qualified.updowncounter")
        .buildWithCallback(observable -> observable.record(10, attributes));
    upDownCounter.add(-1, attributes);
    // gauge
    DoubleGauge gauge = noopMeter.gaugeBuilder("fully.qualified.gauge").build();
    gauge.set(1.1, attributes);
    // async gauge
    noopMeter
        .gaugeBuilder("fully.qualified.gauge")
        .buildWithCallback(observable -> observable.record(10, attributes));

    // No-op logs
    Logger noopLogger = OpenTelemetry.noop().getLogsBridge().get(SCOPE_NAME);
    noopLogger
        .logRecordBuilder()
        .setBody("log message")
        .setAttribute(WIDGET_SHAPE, "square")
        .setSeverity(Severity.INFO)
        .emit();
  }
}
```
<!-- prettier-ignore-end -->

### 시맨틱 속성(Semantic attributes) {#semantic-attributes}

[시맨틱 컨벤션](/docs/specs/semconv/)은 공통적인 작업에 대해 표준화된 방식으로
텔레메트리를 수집하는 방법을 설명한다. 여기에는
[속성 레지스트리](/docs/specs/semconv/registry/attributes/)가 포함되며, 이는
컨벤션에서 참조하는 모든 속성에 대한 정의를 도메인별로 나열한다.
[semantic-conventions-java](https://github.com/open-telemetry/semantic-conventions-java)
프로젝트는 시맨틱 컨벤션으로부터 상수를 생성하며, 이는 계측이 규격을 준수하는 데
도움을 준다.

| 설명                                        | 아티팩트                                                                                     |
| ------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 안정된 시맨틱 컨벤션에 대해 생성된 코드     | `io.opentelemetry.semconv:opentelemetry-semconv:{{% param vers.semconv %}}-alpha`            |
| 인큐베이팅 시맨틱 컨벤션에 대해 생성된 코드 | `io.opentelemetry.semconv:opentelemetry-semconv-incubating:{{% param vers.semconv %}}-alpha` |

> [!NOTE]
>
> `opentelemetry-semconv`와 `opentelemetry-semconv-incubating` 모두 `-alpha`
> 접미사를 포함하며 호환성이 깨지는 변경이 있을 수 있지만, 의도는
> `opentelemetry-semconv`를 안정화하고 `opentelemetry-semconv-incubating`에는
> `-alpha` 접미사를 영구적으로 남겨두는 것이다. 라이브러리는 테스트 목적으로
> `opentelemetry-semconv-incubating`을 사용할 수 있지만, 의존성으로 포함해서는
> 안 된다. 속성은 버전마다 추가되거나 사라질 수 있으므로, 의존성으로 포함하면
> 전이 버전 충돌이 발생할 때 최종 사용자가 런타임 오류에 노출될 수 있다.

시맨틱 컨벤션으로부터 생성된 속성 상수는 `AttributeKey<T>`의 인스턴스이며,
오픈텔레메트리 API가 속성을 받는 어디에서든 사용할 수 있다.

다음 코드 스니펫은 시맨틱 컨벤션 속성 API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/SemanticAttributesUsage.java"?>
```java
package otel;

import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.semconv.HttpAttributes;
import io.opentelemetry.semconv.ServerAttributes;
import io.opentelemetry.semconv.incubating.HttpIncubatingAttributes;

public class SemanticAttributesUsage {
  public static void semanticAttributesUsage() {
    // Semantic attributes are organized by top-level domain and whether they are stable or
    // incubating.
    // For example:
    // - stable attributes starting with http.* are in the HttpAttributes class.
    // - stable attributes starting with server.* are in the ServerAttributes class.
    // - incubating attributes starting with http.* are in the HttpIncubatingAttributes class.
    // Attribute keys which define an enumeration of values are accessible in an inner
    // {AttributeKey}Values class.
    // For example, the enumeration of http.request.method values is available in the
    // HttpAttributes.HttpRequestMethodValues class.
    Attributes attributes =
        Attributes.builder()
            .put(HttpAttributes.HTTP_REQUEST_METHOD, HttpAttributes.HttpRequestMethodValues.GET)
            .put(HttpAttributes.HTTP_ROUTE, "/users/:id")
            .put(ServerAttributes.SERVER_ADDRESS, "example")
            .put(ServerAttributes.SERVER_PORT, 8080L)
            .put(HttpIncubatingAttributes.HTTP_RESPONSE_BODY_SIZE, 1024)
            .build();
  }
}
```
<!-- prettier-ignore-end -->

### 배기지(Baggage) {#baggage}

[Baggage](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/baggage/Baggage.html)는
분산 요청이나 워크플로 실행과 연관된, 애플리케이션이 정의한 키-값 쌍 묶음이다.
배기지의 키와 값은 문자열이며, 값에는 선택적인 문자열 메타데이터가 있다. 스팬,
메트릭, 로그 레코드에 항목을 속성으로 추가하도록 [SDK](../sdk/)를 구성함으로써,
배기지의 데이터로 텔레메트리를 보강할 수 있다. 배기지 API는 [컨텍스트](#context)
위에 구축되며, 이를 통해 스팬 컨텍스트가 애플리케이션 전체와 스레드를 넘어
암묵적으로 전달될 수 있다. 컨텍스트 API 사용에 대한 대부분의 안내는 배기지에도
적용된다.

배기지는
[W3CBaggagePropagator](https://www.javadoc.io/doc/io.opentelemetry/opentelemetry-api/latest/io/opentelemetry/api/baggage/propagation/W3CBaggagePropagator.html)를
통해 애플리케이션 경계를 넘어 전파된다(자세한 내용은
[TextMapPropagator](../sdk/#textmappropagator) 참고).

다음 코드 스니펫은 `Baggage` API 사용법을 살펴본다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/BaggageUsage.java"?>
```java
package otel;

import static io.opentelemetry.context.Context.current;

import io.opentelemetry.api.baggage.Baggage;
import io.opentelemetry.api.baggage.BaggageEntry;
import io.opentelemetry.api.baggage.BaggageEntryMetadata;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.context.Scope;
import java.util.Map;
import java.util.stream.Collectors;

public class BaggageUsage {
  private static final Attributes WIDGET_RED_CIRCLE = Util.WIDGET_RED_CIRCLE;

  public static void baggageUsage() {
    // Access current baggage with Baggage.current()
    // output => context baggage: {}
    Baggage currentBaggage = Baggage.current();
    System.out.println("current baggage: " + asString(currentBaggage));
    // ...or from a Context
    currentBaggage = Baggage.fromContext(current());

    // Baggage has a variety of methods for manipulating and reading data.
    // Convert to builder and add entries:
    Baggage newBaggage =
        Baggage.current().toBuilder()
            .put("shopId", "abc123")
            .put("shopName", "opentelemetry-demo", BaggageEntryMetadata.create("metadata"))
            .build();
    // ...or uncomment to start from empty
    // newBaggage = Baggage.empty().toBuilder().put("shopId", "abc123").build();
    // output => new baggage: {shopId=abc123(), shopName=opentelemetry-demo(metadata)}
    System.out.println("new baggage: " + asString(newBaggage));
    // Read an entry:
    String shopIdValue = newBaggage.getEntryValue("shopId");
    // Inspect size:
    int size = newBaggage.size();
    boolean isEmpty = newBaggage.isEmpty();
    // Convert to map representation:
    Map<String, BaggageEntry> map = newBaggage.asMap();
    // Iterate through entries:
    newBaggage.forEach((s, baggageEntry) -> {});

    // The current baggage still doesn't contain the new entries
    // output => context baggage: {}
    System.out.println("current baggage: " + asString(Baggage.current()));

    // Calling Baggage.makeCurrent() sets Baggage.current() to the baggage until the scope is
    // closed, upon which Baggage.current() is restored to the state prior to when
    // Baggage.makeCurrent() was called.
    try (Scope scope = newBaggage.makeCurrent()) {
      // The current baggage now contains the added value
      // output => context baggage: {shopId=abc123(), shopName=opentelemetry-demo(metadata)}
      System.out.println("current baggage: " + asString(Baggage.current()));
    }

    // The current baggage no longer contains the new entries:
    // output => context baggage: {}
    System.out.println("current baggage: " + asString(Baggage.current()));
  }

  private static String asString(Baggage baggage) {
    return baggage.asMap().entrySet().stream()
        .map(
            entry ->
                String.format(
                    "%s=%s(%s)",
                    entry.getKey(),
                    entry.getValue().getValue(),
                    entry.getValue().getMetadata().getValue()))
        .collect(Collectors.joining(", ", "{", "}"));
  }
}
```
<!-- prettier-ignore-end -->

## 실험적(Incubating) API {#incubating-api}

`io.opentelemetry:opentelemetry-api-incubator:{{% param vers.otel %}}-alpha`
아티팩트는 실험적인 트레이스, 메트릭, 로그, 컨텍스트 API를 포함한다. 인큐베이팅
API는 마이너 릴리스에서 호환성이 깨지는 API 변경이 있을 수 있다. 이들은 종종
실험적인 명세 기능이나, 확정하기 전에 사용자 피드백으로 검증하고 싶은 API 설계를
나타낸다. 사용자가 이러한 API를 사용해보고 (긍정적이든 부정적이든) 피드백과 함께
이슈를 열어보길 권장한다. 라이브러리는 인큐베이팅 API에 의존해서는 안 되는데,
전이 버전 충돌이 발생할 때 사용자가 런타임 오류에 노출될 수 있기 때문이다.

사용 가능한 API와 사용 예시는
[인큐베이터 README](https://github.com/open-telemetry/opentelemetry-java/tree/main/api/incubator)를
참고한다.
