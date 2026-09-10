---
title: 어노테이션
description: Java 에이전트와 함께 계측 어노테이션 사용하기.
aliases: [/docs/instrumentation/java/annotations]
weight: 20
cSpell:ignore: Flowable javac reactivestreams reactivex
default_lang_commit: 115477dad3237f21d1ee15052e8cb3c56bd14819
---

대부분의 사용자에게는 기본 제공 계측만으로 충분하며 더 할 일이 없다. 하지만
때로는 코드를 많이 바꾸지 않고도 직접 작성한 커스텀 코드에 대해
[스팬](/docs/concepts/signals/traces/#spans)을 생성하고 싶을 때가 있다.
`WithSpan` 및 `SpanAttribute` 어노테이션이 이러한 사용 사례를 지원한다.

## 의존성 {#dependencies}

`@WithSpan` 어노테이션을 사용하려면 `opentelemetry-instrumentation-annotations`
라이브러리에 대한 의존성을 추가해야 한다.

{{< tabpane text=true >}} {{% tab "Maven" %}}

```xml
<dependencies>
  <dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-instrumentation-annotations</artifactId>
    <version>{{% param vers.instrumentation %}}</version>
  </dependency>
</dependencies>
```

{{% /tab %}} {{% tab "Gradle" %}}

### Gradle {#gradle}

```groovy
dependencies {
    implementation('io.opentelemetry.instrumentation:opentelemetry-instrumentation-annotations:{{% param vers.instrumentation %}}')
}
```

{{% /tab %}} {{< /tabpane >}}

## `@WithSpan`으로 메서드 주위에 스팬 생성하기 {#creating-spans-around-methods-with-withspan}

특정 메서드를 계측하는 [스팬](/docs/concepts/signals/traces/#spans)을
생성하려면, 해당 메서드에 `@WithSpan`을 붙인다.

```java
import io.opentelemetry.instrumentation.annotations.WithSpan;

public class MyClass {
  @WithSpan
  public void myMethod() {
      <...>
  }
}
```

애플리케이션이 어노테이션이 붙은 메서드를 호출할 때마다, 그 지속 시간을 나타내고
발생한 예외를 제공하는 스팬을 생성한다. 기본적으로 스팬 이름은
`<className>.<methodName>`이 되며, `value` 어노테이션 매개변수로 이름을 제공하면
그 이름이 사용된다.

`@WithSpan`이 붙은 메서드의 반환 타입이 아래에 나열된
[future 또는 promise 유사](https://en.wikipedia.org/wiki/Futures_and_promises)
타입 중 하나라면, future가 완료될 때까지 스팬이 종료되지 않는다.

- [java.util.concurrent.CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [java.util.concurrent.CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)
- [com.google.common.util.concurrent.ListenableFuture](https://guava.dev/releases/10.0/api/docs/com/google/common/util/concurrent/ListenableFuture.html)
- [org.reactivestreams.Publisher](https://www.reactive-streams.org/reactive-streams-1.0.1-javadoc/org/reactivestreams/Publisher.html)
- [reactor.core.publisher.Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)
- [reactor.core.publisher.Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)
- [io.reactivex.Completable](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/Completable.html)
- [io.reactivex.Maybe](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/Maybe.html)
- [io.reactivex.Single](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/Single.html)
- [io.reactivex.Observable](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/Observable.html)
- [io.reactivex.Flowable](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/Flowable.html)
- [io.reactivex.parallel.ParallelFlowable](https://reactivex.io/RxJava/2.x/javadoc/index.html?io/reactivex/parallel/ParallelFlowable.html)

### 매개변수 {#parameters}

`@WithSpan` 어트리뷰트는 스팬 커스터마이즈를 위해 다음과 같은 선택적 매개변수를
지원한다.

| 이름             | 타입              | 기본값     | 설명                                                                                                                |
| ---------------- | ----------------- | ---------- | ------------------------------------------------------------------------------------------------------------------- |
| `value`          | `String`          | `""`       | 스팬 이름. 지정하지 않으면 기본값 `<className>.<methodName>`이 사용된다.                                            |
| `kind`           | `SpanKind` (enum) | `INTERNAL` | [스팬의 종류](/docs/specs/otel/trace/api/#spankind).                                                                |
| `inheritContext` | `boolean`         | `true`     | 2.14.0부터 지원. 새 스팬이 기존(현재) 컨텍스트를 부모로 삼을지 여부를 제어한다. `false`이면 새 컨텍스트가 생성된다. |

매개변수 사용 예시:

```java
@WithSpan(kind = SpanKind.CLIENT, inheritContext = false, value = "my span name")
public void myMethod() {
    <...>
}

@WithSpan("my span name")
public void myOtherMethod() {
    <...>
}
```

## `@SpanAttribute`로 스팬에 속성 추가하기 {#adding-attributes-to-the-span-with-spanattribute}

어노테이션이 붙은 메서드에 대해 [스팬](/docs/concepts/signals/traces/#spans)이
생성되면, 그 메서드 호출에 전달된 인자 값이 생성된 스팬에
[속성](/docs/concepts/signals/traces/#attributes)으로 자동 추가될 수 있다.
메서드 매개변수에 `@SpanAttribute` 어노테이션을 붙이기만 하면 된다.

```java
import io.opentelemetry.instrumentation.annotations.SpanAttribute;
import io.opentelemetry.instrumentation.annotations.WithSpan;

public class MyClass {

    @WithSpan
    public void myMethod(@SpanAttribute("parameter1") String parameter1,
        @SpanAttribute("parameter2") long parameter2) {
        <...>
    }
}
```

어노테이션의 인자로 지정하지 않는 한, 속성 이름은 형식 매개변수 이름에서
파생된다. 단, `javac` 컴파일러에 `-parameters` 옵션을 전달하여 매개변수 이름이
`.class` 파일에 컴파일되어 있어야 한다.

## `@WithSpan` 계측 억제하기 {#suppressing-withspan-instrumentation}

`@WithSpan`으로 과도하게 계측된 코드가 있고 코드를 수정하지 않은 채 그중 일부를
억제하고 싶을 때 `@WithSpan` 억제가 유용하다.

{{% config_option
  name="otel.instrumentation.opentelemetry-instrumentation-annotations.exclude-methods" %}} 특정
메서드에 대해 `@WithSpan` 계측을 억제한다. 형식은
`my.package.MyClass1[method1,method2];my.package.MyClass2[method3]`이다.
{{% /config_option %}}

## `otel.instrumentation.methods.include`로 메서드 주위에 스팬 생성하기 {#creating-spans-around-methods-with-otelinstrumentationmethodsinclude}

코드를 수정할 수 없는 경우에도, 특정 메서드 주위에 스팬을 캡처하도록 Java
에이전트를 구성할 수 있다.

{{% config_option name="otel.instrumentation.methods.include" %}} `@WithSpan`
대신 특정 메서드에 대한 계측을 추가한다. 형식은
`my.package.MyClass1[method1,method2];my.package.MyClass2[method3]`이다. {{%
/config_option %}}

메서드가 오버로드된 경우(같은 클래스에 같은 이름이지만 매개변수가 다른 메서드가
여러 번 나타나는 경우), 해당 메서드의 모든 버전이 계측된다.

## 다음 단계 {#next-steps}

어노테이션 사용을 넘어, 오픈텔레메트리(OpenTelemetry) API를 사용하면
[커스텀 계측](../api)에 사용할 수 있는 트레이서를 얻을 수 있다.
