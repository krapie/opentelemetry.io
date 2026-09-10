---
title: 어노테이션
weight: 50
default_lang_commit: 2d89b60b2e09d42ba96757b0afdbc31f54a2b0e7
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/spring-starter"?>

대부분의 사용자에게는 기본 제공 계측(out-of-the-box instrumentation)으로
충분하며, 그 이상 할 일이 없다. 하지만 때로는 코드 변경을 많이 하지 않고도
사용자 정의 코드에 대한 [스팬](/docs/concepts/signals/traces/#spans)을 생성하고
싶을 때가 있다.

메서드에 `WithSpan` 어노테이션(annotation)을 추가하면 해당 메서드가 스팬으로
감싸진다. `SpanAttribute` 어노테이션을 사용하면 메서드 인자를 속성으로 캡처할 수
있다.

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/TracedClass.java"?>
```java
package otel;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.instrumentation.annotations.SpanAttribute;
import io.opentelemetry.instrumentation.annotations.WithSpan;
import org.springframework.stereotype.Component;

/** Test WithSpan */
@Component
public class TracedClass {

  @WithSpan
  public void tracedMethod() {}

  @WithSpan(value = "span name")
  public void tracedMethodWithName() {
    Span currentSpan = Span.current();
    currentSpan.addEvent("ADD EVENT TO tracedMethodWithName SPAN");
    currentSpan.setAttribute("isTestAttribute", true);
  }

  @WithSpan(kind = SpanKind.CLIENT)
  public void tracedClientSpan() {}

  @WithSpan
  public void tracedMethodWithAttribute(@SpanAttribute("attributeName") String parameter) {}
}
```
<!-- prettier-ignore-end -->

> [!NOTE]
>
> 오픈텔레메트리(OpenTelemetry) 어노테이션은 프록시 기반의 Spring AOP를
> 사용한다.
>
> 이 어노테이션은 프록시의 메서드에 대해서만 동작한다. 자세한 내용은
> [Spring 문서](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)에서
> 확인할 수 있다.
>
> 다음 예제에서 `WithSpan` 어노테이션은 GET 엔드포인트가 호출될 때 아무 동작도
> 하지 않는다:
>
> ```java
> @RestController
> public class MyControllerManagedBySpring {
>
>     @GetMapping("/ping")
>     public void aMethod() {
>         anotherMethod();
>     }
>
>     @WithSpan
>     public void anotherMethod() {
>     }
> }
> ```

{{< comment >}} Note that we have to use the alert shortcode because it contains
tab panes.

<!-- markdownlint-capture -->
<!-- markdownlint-disable prefer-blockquote-vs-docsy-alerts -->

{{< /comment >}}

{{% alert title="참고" %}}

오픈텔레메트리 어노테이션을 사용하려면, 프로젝트에 Spring Boot Starter AOP
의존성을 추가해야 한다:

{{< tabpane text=true >}} {{% tab header="Maven (`pom.xml`)" lang=Maven %}}

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
  </dependency>
</dependencies>
```

{{% /tab %}} {{% tab header="Gradle (`build.gradle`)" lang=Gradle %}}

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-aop")
}
```

{{% /tab %}} {{< /tabpane >}}

{{% /alert %}}

<!-- markdownlint-restore -->

`otel.instrumentation.annotations.enabled` 프로퍼티를 `false`로 설정하면
오픈텔레메트리 어노테이션을 비활성화할 수 있다.

[선언적 구성](../declarative-configuration/)에서는 대신 중앙 집중식 계측 목록을
사용한다:

```yaml
otel:
  distribution:
    spring_starter:
      instrumentation:
        disabled:
          - annotations
```

`WithSpan` 어노테이션의 요소를 사용해 스팬을 커스터마이징할 수 있다:

| 이름    | 타입       | 설명                   | 기본값              |
| ------- | ---------- | ---------------------- | ------------------- |
| `value` | `String`   | 스팬 이름              | ClassName.Method    |
| `kind`  | `SpanKind` | 스팬의 종류(span kind) | `SpanKind.INTERNAL` |

`SpanAttribute` 어노테이션의 `value` 요소로 속성 이름을 설정할 수 있다:

| 이름    | 타입     | 설명      | 기본값               |
| ------- | -------- | --------- | -------------------- |
| `value` | `String` | 속성 이름 | 메서드 매개변수 이름 |

## 다음 단계 {#next-steps}

어노테이션 사용을 넘어서, 오픈텔레메트리 API를 사용하면
[사용자 정의 계측](../api)에 사용할 수 있는 트레이서(tracer)를 얻을 수 있다.
