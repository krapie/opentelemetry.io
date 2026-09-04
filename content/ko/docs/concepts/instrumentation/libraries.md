---
title: 라이브러리
description: 라이브러리에 네이티브 계측을 추가하는 방법을 알아본다.
aliases: [../instrumenting-library]
weight: 40
default_lang_commit: d8e58463c6e7c324b01115ab4f88d1f2bcf802c2
---

오픈텔레메트리(OpenTelemetry)는 많은 라이브러리에 대해 [계측
라이브러리][instrumentation libraries]를 제공하며, 이는 일반적으로 라이브러리
훅(hook)이나 라이브러리 코드에 대한 몽키 패칭(monkey patching)을 통해
이루어진다.

오픈텔레메트리를 사용한 네이티브(native) 라이브러리 계측은 사용자에게 더 나은
옵저버빌리티(observability)와 개발자 경험(developer experience)을 제공하며,
라이브러리가 훅을 노출하고 문서화해야 할 필요성을 없애준다. 네이티브 계측이
제공하는 다른 이점은 다음과 같다.

- 커스텀 로깅 훅을 공통적이고 사용하기 쉬운 오픈텔레메트리 API로 대체할 수
  있으며, 사용자는 오픈텔레메트리만 사용하면 된다.
- 라이브러리와 애플리케이션 코드에서 발생하는 트레이스, 로그, 메트릭이 서로
  연관되고 일관성을 갖는다.
- 공통 컨벤션(convention)을 통해 사용자는 동일한 기술 내에서, 그리고 여러
  라이브러리와 언어에 걸쳐 유사하고 일관된 텔레메트리를 얻을 수 있다.
- 텔레메트리 시그널은 다양한 소비 시나리오에 맞게, 잘 문서화된 다양한
  오픈텔레메트리 확장 지점(extensibility point)을 사용하여 세밀하게 조정(필터링,
  처리, 집계)할 수 있다.

![네이티브 계측과 계측 라이브러리 비교](../native-vs-libraries.svg)

## 시맨틱 컨벤션(Semantic Conventions) {#semantic-conventions}

[시맨틱 컨벤션](/docs/specs/semconv/general/trace/)은 웹 프레임워크, RPC
클라이언트, 데이터베이스, 메시징 클라이언트, 인프라 등에서 생성되는 스팬(span)에
어떤 정보가 포함되는지에 대한 주된 기준(source of truth)이다. 컨벤션은 계측을
일관되게 만들어 준다. 텔레메트리를 다루는 사용자는 라이브러리마다의 특성을 따로
배울 필요가 없고, 옵저버빌리티 벤더는 데이터베이스나 메시징 시스템처럼 다양한
기술을 아우르는 경험을 구축할 수 있다. 라이브러리가 컨벤션을 따르면, 사용자의
별도 입력이나 설정 없이도 많은 시나리오를 지원할 수 있다.

시맨틱 컨벤션은 계속 발전하고 있으며 새로운 컨벤션이 끊임없이 추가된다.
라이브러리에 대한 컨벤션이 아직 없다면,
[컨벤션을 추가](https://github.com/open-telemetry/semantic-conventions/issues)하는
것을 고려한다. 스팬 이름에 특히 주의를 기울인다. 의미 있는 이름을 사용하도록
노력하고, 이름을 정의할 때 카디널리티(cardinality)를 고려한다. 또한 사용 중인
시맨틱 컨벤션의 버전을 기록하는 데 사용할 수 있는
[`schema_url`](/docs/specs/otel/schemas/#schema-url) 속성도 설정한다.

피드백이 있거나 새로운 컨벤션을 추가하고 싶다면,
[Instrumentation Slack](https://cloud-native.slack.com/archives/C01QZFGMLQ7)에
참여하거나
[명세 저장소(Specification repository)](https://github.com/open-telemetry/opentelemetry-specification)에
이슈나 풀 리퀘스트를 열어 기여한다.

### 스팬 정의하기 {#defining-spans}

라이브러리 사용자의 관점에서 라이브러리를 생각해 보고, 사용자가 라이브러리의
동작과 활동에 대해 어떤 것을 알고 싶어할지 생각해 본다. 라이브러리 관리자로서
내부 구조는 잘 알고 있겠지만, 사용자는 라이브러리 내부 동작보다는 자신의
애플리케이션 기능에 더 관심이 있을 가능성이 높다. 라이브러리 사용 현황을
분석하는 데 어떤 정보가 도움이 될지 생각한 다음, 해당 데이터를 모델링하는 적절한
방법을 생각해 본다. 고려할 만한 몇 가지 측면은 다음과 같다.

- 스팬과 스팬 계층 구조
- 집계된 메트릭의 대안으로서 스팬에 부여하는 수치 속성
- 스팬 이벤트
- 집계된 메트릭

예를 들어 라이브러리가 데이터베이스에 요청을 보내는 경우, 데이터베이스에 대한
논리적 요청에 대해서만 스팬을 생성한다. 네트워크를 통한 물리적 요청은 해당
기능을 구현하는 라이브러리 내에서 계측해야 한다. 또한 객체/데이터
직렬화(serialization)와 같은 다른 활동은 추가 스팬으로 만들기보다는 스팬
이벤트로 캡처하는 것을 선호해야 한다.

스팬 속성을 설정할 때는 시맨틱 컨벤션을 따른다.

## 계측하지 말아야 할 때 {#when-not-to-instrument}

일부 라이브러리는 네트워크 호출을 감싸는 씬 클라이언트(thin client)이다.
오픈텔레메트리에 기반이 되는 RPC 클라이언트에 대한 계측 라이브러리가 이미 있을
가능성이 크다. 기존 라이브러리를 찾으려면
[레지스트리(registry)](/ecosystem/registry/)를 확인한다. 라이브러리가 이미
존재한다면, 래퍼 라이브러리를 계측할 필요는 없을 수도 있다.

일반적인 지침으로, 라이브러리는 자신의 수준에서만 계측해야 한다. 다음 경우가
모두 해당된다면 계측하지 않는다.

- 라이브러리가 문서화되어 있거나 자명한(self-explanatory) API 위에 있는 얇은
  프록시(thin proxy)이다.
- 오픈텔레메트리가 기반 네트워크 호출에 대한 계측을 이미 제공한다.
- 라이브러리가 텔레메트리를 보강하기 위해 따라야 할 컨벤션이 없다.

확신이 서지 않으면 계측하지 않는다. 계측하지 않기로 했더라도, 내부 RPC
클라이언트 인스턴스에 대해 오픈텔레메트리 핸들러를 설정할 수 있는 방법을
제공하는 것이 여전히 유용할 수 있다. 이는 완전한 자동 계측을 지원하지 않는
언어에서는 필수적이며, 다른 언어에서도 여전히 유용하다.

이 문서의 나머지 부분에서는 애플리케이션을 계측하는 방법과 대상에 대한 지침을
제공한다.

## 오픈텔레메트리 API {#opentelemetry-api}

애플리케이션을 계측하는 첫 단계는 오픈텔레메트리 API 패키지를 의존성으로
포함하는 것이다.

오픈텔레메트리에는 [두 가지 주요 모듈](/docs/specs/otel/overview/)인 API와 SDK가
있다. 오픈텔레메트리 API는 추상화와 비운영(non-operational) 구현체의 집합이다.
애플리케이션이 오픈텔레메트리 SDK를 임포트하지 않는 한, 계측은 아무 동작도 하지
않으며 애플리케이션 성능에 영향을 주지 않는다.

### 라이브러리는 오픈텔레메트리 API만 사용해야 한다 {#libraries-should-only-use-the-opentelemetry-api}

새로운 의존성을 추가하는 것이 걱정된다면, 의존성 충돌을 최소화하는 방법을
결정하는 데 도움이 되는 몇 가지 고려 사항을 소개한다.

- 오픈텔레메트리 트레이스(Trace) API는 2021년 초에 안정화되었다. 이는
  [시맨틱 버저닝(Semantic Versioning) 2.0](/docs/specs/otel/versioning-and-stability/)을
  따른다.
- 가장 이른 안정 버전의 오픈텔레메트리 API(1.0.\*)를 사용하고, 새로운 기능을
  사용해야 하는 경우가 아니라면 업데이트를 피한다.
- 계측이 안정화되는 동안에는 별도의 패키지로 배포하는 것을 고려하여, 이를
  사용하지 않는 사용자에게 절대 문제가 되지 않도록 한다. 이를 자체 저장소에
  보관하거나,
  [오픈텔레메트리에 추가](https://github.com/open-telemetry/opentelemetry-specification/blob/main/oteps/0155-external-modules.md#contrib-components)하여
  다른 계측 라이브러리와 함께 배포되도록 할 수 있다.
- 시맨틱 컨벤션은 [안정적이지만 계속
  발전한다][stable, but subject to evolution]. 이것이 기능적인 문제를 일으키지는
  않지만, 가끔 계측을 업데이트해야 할 수도 있다. 이를 프리뷰 플러그인이나
  오픈텔레메트리 contrib 저장소에 두면, 사용자에게 호환성이 깨지는 변경(breaking
  change) 없이 컨벤션을 최신 상태로 유지하는 데 도움이 될 수 있다.

  [stable, but subject to evolution]:
    /docs/specs/otel/versioning-and-stability/#semantic-conventions-stability

### 트레이서 가져오기 {#getting-a-tracer}

모든 애플리케이션 설정은 트레이서(Tracer) API를 통해 라이브러리로부터 숨겨진다.
라이브러리는 의존성 주입(dependency injection)과 손쉬운 테스트를 위해
애플리케이션이 `TracerProvider`의 인스턴스를 전달하도록 허용하거나,
[전역(global) `TracerProvider`](/docs/specs/otel/trace/api/#get-a-tracer)로부터
이를 얻을 수 있다. 오픈텔레메트리 언어별 구현체는 각 프로그래밍 언어에서
관용적인 방식에 따라 인스턴스를 전달하거나 전역 객체에 접근하는 것에 대해 서로
다른 선호를 가질 수 있다.

트레이서를 얻을 때는 라이브러리(또는 트레이싱 플러그인)의 이름과 버전을
제공한다. 이는 텔레메트리에 표시되어 사용자가 텔레메트리를 처리하고 필터링하며,
어디서 왔는지 이해하고, 계측 문제를 디버깅하거나 보고하는 데 도움이 된다.

## 무엇을 계측할지 {#what-to-instrument}

### 공개 API {#public-apis}

공개 API는 트레이싱하기 좋은 대상이다. 공개 API 호출에 대해 생성된 스팬을 통해
사용자는 텔레메트리를 애플리케이션 코드에 매핑하고, 라이브러리 호출의 지속
시간과 결과를 파악할 수 있다. 트레이싱할 호출에는 다음이 포함된다.

- 내부적으로 네트워크 호출을 하는 공개 메서드나, 상당한 시간이 걸리고 실패할 수
  있는 로컬 연산(예: I/O).
- 요청이나 메시지를 처리하는 핸들러.

#### 계측 예시 {#instrumentation-example}

다음 예시는 자바(Java) 애플리케이션을 계측하는 방법을 보여준다.

```java
private static Tracer tracer =  getTracer(TracerProvider.noop());

public static void setTracerProvider(TracerProvider tracerProvider) {
    tracer = getTracer(tracerProvider);
}

private static Tracer getTracer(TracerProvider tracerProvider) {
    return tracerProvider.getTracer("demo-db-client", "0.1.0-beta1");
}

private Response selectWithTracing(Query query) {
    // 스팬 이름과 속성에 대한 안내는 컨벤션을 참고한다
    Span span = tracer.spanBuilder(String.format("SELECT %s.%s", dbName, collectionName))
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("db.name", dbName)
            ...
            .startSpan();

    // 스팬을 활성 상태로 만들어 로그와 중첩된 스팬을 연관 지을 수 있게 한다
    try (Scope unused = span.makeCurrent()) {
        Response response = query.runWithRetries();
        if (response.isSuccessful()) {
            span.setStatus(StatusCode.OK);
        }

        if (span.isRecording()) {
           // 응답 코드 및 기타 정보에 대한 응답 속성을 채운다
        }
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getClass().getSimpleName());
        throw e;
    } finally {
        span.end();
    }
}
```

속성을 채울 때는 컨벤션을 따른다. 적용 가능한 컨벤션이 없다면
[일반 컨벤션](/docs/specs/semconv/general/attributes/)을 참고한다.

### 중첩된 네트워크 스팬 및 기타 스팬 {#nested-network-and-other-spans}

네트워크 호출은 일반적으로 해당 클라이언트 구현체를 통해 오픈텔레메트리 자동
계측(auto-instrumentation)으로 트레이싱된다.

![Jaeger UI에서 중첩된 데이터베이스 및 HTTP 스팬](../nested-spans.svg)

오픈텔레메트리가 네트워크 클라이언트에 대한 트레이싱을 지원하지 않는다면, 최선의
방법을 결정하는 데 도움이 되는 몇 가지 고려 사항이 있다.

- 네트워크 호출을 트레이싱하면 사용자에 대한 옵저버빌리티나 사용자를 지원하는
  능력이 향상되는가?
- 라이브러리가 공개적으로 문서화된 RPC API 위에 있는 래퍼인가? 문제가 발생했을
  때 사용자가 기반 서비스로부터 지원을 받아야 하는가?
  - 라이브러리를 계측하고, 개별 네트워크 시도(try)를 반드시 트레이싱한다.
- 이러한 호출을 스팬으로 트레이싱하면 매우 장황해지는가? 아니면 성능에 눈에 띄게
  영향을 미치는가?
  - 세부 수준(verbosity)이 있는 로그나 스팬 이벤트를 사용한다. 로그는 부모(공개
    API 호출)와 연관 지을 수 있고, 스팬 이벤트는 공개 API 스팬에 설정해야 한다.
  - 고유한 트레이스 컨텍스트를 전달하고 전파하기 위해 반드시 스팬이어야 한다면,
    이를 설정 옵션 뒤에 두고 기본값으로는 비활성화한다.

오픈텔레메트리가 이미 네트워크 호출에 대한 트레이싱을 지원한다면, 이를 중복해서
구현하고 싶지는 않을 것이다. 몇 가지 예외가 있을 수 있다.

- 특정 환경에서 동작하지 않거나 사용자가 몽키 패칭에 우려를 갖는 등, 자동 계측을
  사용하지 않는 사용자를 지원하기 위해서.
- 기반 서비스와의 커스텀 또는 레거시 연관(correlation) 및 컨텍스트 전파
  프로토콜을 지원하기 위해서.
- 자동 계측이 다루지 않는, 필수적인 라이브러리 또는 서비스별 정보로 RPC 스팬을
  보강하기 위해서.

중복을 피하기 위한 범용 솔루션은 현재 개발 중이다.

### 이벤트 {#events}

트레이스는 애플리케이션이 내보낼 수 있는 시그널의 한 종류이다. 이벤트(또는
로그)와 트레이스는 서로 중복되지 않고 보완한다. 특정 수준의 세부
정보(verbosity)가 필요한 경우라면 트레이스보다 로그가 더 나은 선택이다.

애플리케이션이 로깅이나 이와 유사한 모듈을 사용한다면, 해당 로깅 모듈에는 이미
오픈텔레메트리 통합이 있을 수 있다. 이를 확인하려면
[레지스트리](/ecosystem/registry/)를 참고한다. 통합은 일반적으로 활성 트레이스
컨텍스트를 모든 로그에 남겨서(stamp), 사용자가 이를 연관 지을 수 있게 한다.

사용 중인 언어와 생태계가 일반적인 로깅을 지원하지 않는다면, 추가적인
애플리케이션 세부 정보를 공유하기 위해 [스팬 이벤트][span events]를 사용한다.
속성도 함께 추가하고 싶다면 이벤트가 더 편리할 수 있다.

일반적인 원칙으로, 세부적인(verbose) 데이터에는 스팬 대신 이벤트나 로그를
사용한다. 이벤트는 항상 계측이 생성한 스팬 인스턴스에 첨부한다. 활성 스팬이
무엇을 가리키는지 제어할 수 없으므로, 가능하면 활성 스팬을 사용하는 것을 피한다.

## 컨텍스트 전파 {#context-propagation}

### 컨텍스트 추출 {#extracting-context}

웹 프레임워크나 메시징 컨슈머와 같이 업스트림 호출을 수신하는 라이브러리나
서비스에서 작업한다면, 들어오는 요청이나 메시지에서 컨텍스트를 추출한다.
오픈텔레메트리는 특정 전파 표준을 감추고 트레이스 `Context`를 와이어(wire)에서
읽어 들이는 `Propagator`(전파자) API를 제공한다. 단일 응답의 경우, 와이어에는
하나의 컨텍스트만 있으며, 이는 라이브러리가 생성하는 새 스팬의 부모가 된다.

스팬을 생성한 후에는, 가능하다면 명시적으로 스팬을 활성 상태로 만들어
애플리케이션 코드(콜백 또는 핸들러)에 새 트레이스 컨텍스트를 전달한다. 다음 자바
예시는 트레이스 컨텍스트를 추가하고 스팬을 활성화하는 방법을 보여준다. 더 많은
예시는
[자바에서의 컨텍스트 추출](/docs/languages/java/api/#contextpropagators)을
참고한다.

```java
// 컨텍스트를 추출한다
Context extractedContext = propagator.extract(Context.current(), httpExchange, getter);
Span span = tracer.spanBuilder("receive")
            .setSpanKind(SpanKind.SERVER)
            .setParent(extractedContext)
            .startSpan();

// 중첩된 텔레메트리가 연관되도록 스팬을 활성 상태로 만든다
try (Scope unused = span.makeCurrent()) {
  userCode();
} catch (Exception e) {
  span.recordException(e);
  span.setStatus(StatusCode.ERROR);
  throw e;
} finally {
  span.end();
}
```

메시징 시스템의 경우, 한 번에 둘 이상의 메시지를 수신할 수도 있다. 수신된
메시지는 생성하는 스팬에 대한 링크가 된다. 자세한 내용은
[메시징 컨벤션](/docs/specs/semconv/messaging/messaging-spans/)을 참고한다.

### 컨텍스트 주입 {#injecting-context}

아웃바운드 호출을 할 때는 일반적으로 다운스트림 서비스로 컨텍스트를 전파하고
싶을 것이다. 이 경우, 나가는 호출을 트레이싱하기 위해 새 스팬을 생성하고
`Propagator` API를 사용해 메시지에 컨텍스트를 주입한다. 예를 들어 비동기 처리를
위한 메시지를 생성할 때와 같이, 컨텍스트를 주입하고 싶은 다른 경우도 있을 수
있다. 다음 자바 예시는 컨텍스트를 전파하는 방법을 보여준다. 더 많은 예시는
[자바에서의 컨텍스트 주입](/docs/languages/java/instrumentation/#context-propagation)을
참고한다.

```java
Span span = tracer.spanBuilder("send")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();

// 중첩된 텔레메트리가 연관되도록 스팬을 활성 상태로 만든다
// 네트워크 호출조차도 스팬, 로그, 이벤트의 중첩된 계층을 가질 수 있다
try (Scope unused = span.makeCurrent()) {
  // 컨텍스트를 주입한다
  propagator.inject(Context.current(), transportLayer, setter);
  send();
} catch (Exception e) {
  span.recordException(e);
  span.setStatus(StatusCode.ERROR);
  throw e;
} finally {
  span.end();
}
```

컨텍스트를 전파할 필요가 없는 몇 가지 예외가 있을 수 있다.

- 다운스트림 서비스가 메타데이터를 지원하지 않거나 알 수 없는 필드를 금지한다.
- 다운스트림 서비스가 연관 프로토콜을 정의하지 않는다. 향후 버전에서 컨텍스트
  전파에 대한 지원을 추가하는 것을 고려한다.
- 다운스트림 서비스가 커스텀 연관 프로토콜을 지원한다.
  - 커스텀 전파자(propagator)로 최선을 다한다. 호환된다면 오픈텔레메트리
    트레이스 컨텍스트를 사용하거나, 커스텀 연관 ID를 생성하여 스팬에 남긴다.

### 프로세스 내부 {#in-process}

- 스팬을 활성 또는 현재(current) 상태로 만든다. 이렇게 하면 스팬을 로그 및
  중첩된 자동 계측과 연관 지을 수 있다.
- 라이브러리에 컨텍스트라는 개념이 있다면, 활성 스팬 외에도 선택적인 명시적
  트레이스 컨텍스트 전파를 지원한다.
  - 라이브러리가 생성한 스팬(트레이스 컨텍스트)을 컨텍스트에 명시적으로 넣고,
    이에 접근하는 방법을 문서화한다.
  - 사용자가 자신의 컨텍스트에 트레이스 컨텍스트를 전달할 수 있도록 허용한다.
- 라이브러리 내부에서는 트레이스 컨텍스트를 명시적으로 전파한다. 활성 스팬은
  콜백 중에 바뀔 수 있다.
  - 공개 API 표면에서 가능한 한 빨리 사용자로부터 활성 컨텍스트를 캡처하여, 이를
    스팬의 부모 컨텍스트로 사용한다.
  - 컨텍스트를 전달하고, 명시적으로 전파되는 인스턴스에 속성, 예외, 이벤트를
    남긴다.
  - 스레드를 명시적으로 시작하거나 백그라운드 처리 등 사용 언어의 비동기
    컨텍스트 흐름 제약으로 인해 문제가 발생할 수 있는 작업을 수행하는 경우 이는
    필수적이다.

## 추가 고려 사항 {#additional-considerations}

### 계측 레지스트리 {#instrumentation-registry}

사용자가 계측 라이브러리를 찾을 수 있도록
[오픈텔레메트리 레지스트리](/ecosystem/registry/)에 등록한다.

### 성능 {#performance}

오픈텔레메트리 API는 애플리케이션에 SDK가 없을 때 아무 동작도 하지 않으며(no-op)
매우 높은 성능을 낸다. 오픈텔레메트리 SDK가 설정되면,
[제한된 리소스를 소비한다](/docs/specs/otel/performance/).

특히 대규모(high scale) 환경에서 실제 애플리케이션은 헤드 기반 샘플링(head-based
sampling)을 설정하는 경우가 많다. 샘플링되지 않은(sampled-out) 스팬은 부담이
적으며, 속성을 채우는 동안 추가적인 할당(allocation)과 잠재적으로 비용이 큰
계산을 피하기 위해 스팬이 기록 중(recording)인지 확인할 수 있다. 다음 자바
예시는 샘플링을 위한 속성을 제공하고 스팬 기록 여부를 확인하는 방법을 보여준다.

```java
// 일부 속성은 샘플링에 중요하므로 생성 시점에 제공해야 한다
Span span = tracer.spanBuilder(String.format("SELECT %s.%s", dbName, collectionName))
        .setSpanKind(SpanKind.CLIENT)
        .setAttribute("db.name", dbName)
        ...
        .startSpan();

// 계산 비용이 큰 속성을 비롯한 다른 속성은
// 스팬이 기록 중일 때만 추가해야 한다
if (span.isRecording()) {
    span.setAttribute("db.statement", sanitize(query.statement()))
}
```

### 오류 처리 {#error-handling}

오픈텔레메트리 API는 잘못된 인자에도 실패하지 않고, 절대 예외를 던지지 않으며,
예외를 삼킨다. 이는
[런타임에 관대하다](/docs/specs/otel/error-handling/#basic-error-handling-principles)는
의미이다. 이러한 방식으로 계측 문제가 애플리케이션 로직에 영향을 주지 않는다.
오픈텔레메트리가 런타임에 숨기는 문제를 발견하려면 계측을 테스트한다.

### 테스트 {#testing}

오픈텔레메트리에는 다양한 자동 계측이 있으므로, 계측이 들어오는 요청, 나가는
요청, 로그 등 다른 텔레메트리와 어떻게 상호작용하는지 시도해 본다. 계측을 시도할
때는 널리 쓰이는 프레임워크와 라이브러리, 그리고 모든 트레이싱이 활성화된
일반적인 애플리케이션을 사용한다. 자신의 라이브러리와 유사한 라이브러리가 어떻게
나타나는지 확인해 본다.

단위 테스트(unit testing)를 위해서는 다음 자바 예시처럼 일반적으로
`SpanProcessor`와 `SpanExporter`를 모킹(mock)하거나 페이크(fake)로 만들 수 있다.

```java
@Test
public void checkInstrumentation() {
  SpanExporter exporter = new TestExporter();

  Tracer tracer = OpenTelemetrySdk.builder()
           .setTracerProvider(SdkTracerProvider.builder()
              .addSpanProcessor(SimpleSpanProcessor.create(exporter)).build()).build()
           .getTracer("test");
  // 테스트를 실행한다 ...

  validateSpans(exporter.exportedSpans);
}

class TestExporter implements SpanExporter {
  public final List<SpanData> exportedSpans = Collections.synchronizedList(new ArrayList<>());

  @Override
  public CompletableResultCode export(Collection<SpanData> spans) {
    exportedSpans.addAll(spans);
    return CompletableResultCode.ofSuccess();
  }
  ...
}
```

[instrumentation libraries]:
  /docs/specs/otel/overview/#instrumentation-libraries
[span events]: /docs/specs/otel/trace/api/#add-events
