---
title: OpenTracing에서 마이그레이션
linkTitle: OpenTracing
aliases: [/docs/migration/opentracing/]
default_lang_commit: b4b91dc7bdbb03ae1de2c1e276194d98b9d15b94
---

[OpenTracing][]와의 하위 호환성은 오픈텔레메트리(OpenTelemetry) 프로젝트
초기부터 우선순위였다. 마이그레이션을 쉽게 하기 위해, 오픈텔레메트리는 동일한
코드베이스에서 오픈텔레메트리 API와 OpenTracing API를 _모두_ 사용하는 것을
지원한다. 이를 통해 OpenTracing 계측을 오픈텔레메트리 SDK를 사용해 기록할 수
있다.

이를 위해 각 오픈텔레메트리 SDK는 **OpenTracing 심(shim)** 을 제공하며, 이는
OpenTracing API와 오픈텔레메트리 SDK 사이의 다리 역할을 한다. OpenTracing 심은
기본적으로 비활성화되어 있다는 점에 유의한다.

## 언어 버전 지원 {#language-version-support}

OpenTracing 심을 사용하기 전에 프로젝트의 언어 및 런타임 컴포넌트 버전을
확인하고, 필요하다면 업데이트한다. OpenTracing API와 오픈텔레메트리 API의 최소
**언어** 버전은 아래 표에 나와 있다.

| 언어           | OpenTracing API | OpenTelemetry API |
| -------------- | --------------- | ----------------- |
| [Go][]         | 1.13            | 1.16              |
| [Java][]       | 7               | 8                 |
| [Python][]     | 2.7             | 3.6               |
| [JavaScript][] | 6               | 8.5               |
| [.NET][]       | 1.3             | 1.4               |
| [C++][]        | 11              | 11                |

오픈텔레메트리 API와 SDK는 일반적으로 OpenTracing에 대응하는 것보다 더 높은 언어
버전을 요구한다는 점에 유의한다.

## 마이그레이션 개요 {#migration-overview}

많은 코드베이스가 현재 OpenTracing으로 계측되어 있다. 이러한 코드베이스는
OpenTracing API를 사용해 애플리케이션 코드를 계측하거나, OpenTracing 플러그인을
설치해 라이브러리와 프레임워크를 계측한다.

오픈텔레메트리로 마이그레이션하는 일반적인 접근 방식은 다음과 같이 요약할 수
있다.

1. 오픈텔레메트리 SDK를 설치하고, 현재의 OpenTracing 구현체(예: Jaeger
   클라이언트)를 제거한다.
2. 오픈텔레메트리 계측 라이브러리를 설치하고, OpenTracing에 대응하는 것을
   제거한다.
3. 새로운 오픈텔레메트리 데이터를 사용하도록 대시보드, 알림 등을 업데이트한다.
4. 새 애플리케이션 코드를 작성할 때는 모든 새 계측을 오픈텔레메트리 API를 사용해
   작성한다.
5. 오픈텔레메트리 API를 사용해 애플리케이션을 점진적으로 다시 계측한다.
   애플리케이션에서 기존 OpenTracing API 호출을 제거해야 하는 엄격한 요구 사항은
   없으며, 이는 계속 동작한다.

규모가 큰 애플리케이션을 마이그레이션하려면 상당한 노력이 필요할 수 있지만,
위에서 제안한 것처럼 OpenTracing 사용자는 애플리케이션 코드를 점진적으로
마이그레이션할 것을 권장한다. 이렇게 하면 마이그레이션 부담이 줄어들고
옵저버빌리티(observability)의 중단을 피하는 데 도움이 된다.

아래 단계는 오픈텔레메트리로 전환하는 신중하고 점진적인 접근 방식을 보여준다.

### 1단계: 오픈텔레메트리 SDK 설치 {#step-1-install-the-opentelemetry-sdk}

계측을 변경하기 전에, 애플리케이션이 현재 내보내고 있는 텔레메트리에 어떠한
중단도 일으키지 않고 오픈텔레메트리 SDK로 전환할 수 있는지 확인한다. 새로운
계측을 동시에 도입하지 않고 이 단계만 단독으로 수행하는 것이 권장되는데, 계측에
어떤 형태로든 중단이 있는지 판단하기가 더 쉬워지기 때문이다.

1. 현재 사용 중인 OpenTracing Tracer 구현체를 오픈텔레메트리 SDK로 교체한다.
   예를 들어 Jaeger를 사용하고 있다면, Jaeger 클라이언트를 제거하고 이에
   대응하는 오픈텔레메트리 클라이언트를 설치한다.
2. OpenTracing 심을 설치한다. 이 심을 통해 오픈텔레메트리 SDK가 OpenTracing
   계측을 소비할 수 있다.
3. OpenTracing 클라이언트가 사용하던 것과 동일한 프로토콜과 형식으로 데이터를
   내보내도록 오픈텔레메트리 SDK를 구성한다. 예를 들어 Zipkin 형식으로 트레이스
   데이터를 내보내는 OpenTracing 클라이언트를 사용하고 있었다면, 오픈텔레메트리
   클라이언트도 동일하게 구성한다.
4. 또는 오픈텔레메트리 SDK가 OTLP를 내보내도록 구성하고, 데이터를
   컬렉터(Collector)로 전송해 여러 형식으로 데이터를 내보내는 것을 컬렉터에서
   관리할 수도 있다.

오픈텔레메트리 SDK를 설치했다면, 애플리케이션을 배포해도 동일한 OpenTracing 기반
텔레메트리를 계속 수신할 수 있는지 확인한다. 즉, 대시보드, 알림, 그리고 다른
트레이스 기반 분석 도구가 여전히 동작하는지 확인한다.

### 2단계: 계측을 점진적으로 교체 {#step-2-progressively-replace-instrumentation}

오픈텔레메트리 SDK가 설치되면, 이제 모든 새 계측을 오픈텔레메트리 API를 사용해
작성할 수 있다. 몇 가지 예외를 제외하면 오픈텔레메트리와 OpenTracing 계측은
매끄럽게 함께 동작한다(아래 [호환성의 한계](#limits-on-compatibility) 참고).

기존 계측은 어떻게 해야 할까? 기존 애플리케이션 코드를 오픈텔레메트리로
마이그레이션해야 하는 엄격한 요구 사항은 없다. 다만 웹 프레임워크, HTTP
클라이언트, 데이터베이스 클라이언트 등을 계측하는 데 사용되는 OpenTracing 계측
라이브러리는 이에 대응하는 오픈텔레메트리 라이브러리로 마이그레이션할 것을
권장한다. 많은 OpenTracing 라이브러리가 지원 종료(retire)되어 더 이상
업데이트되지 않을 수 있으므로, 이렇게 하면 지원이 개선된다.

오픈텔레메트리 계측 라이브러리로 전환하면 생성되는 데이터가 달라진다는 점에
유의해야 한다. 오픈텔레메트리는 소프트웨어를 계측하는 방식에 대해 개선된
모델("시맨틱 컨벤션(semantic conventions)"이라고 부르는 것)을 가지고 있다. 많은
경우 오픈텔레메트리는 더 낫고 더 포괄적인 트레이스 데이터를 생성한다. 하지만 "더
낫다"는 것은 "다르다"는 것을 의미하기도 한다. 즉, 기존의 오래된 OpenTracing 계측
라이브러리를 기반으로 한 대시보드, 알림 등은 해당 라이브러리가 교체되면 더 이상
동작하지 않을 수 있다.

기존 계측에 대해서는 다음을 권장한다.

1. OpenTracing 계측 하나를 이에 대응하는 오픈텔레메트리 계측으로 교체한다.
2. 이로 인해 애플리케이션이 생성하는 텔레메트리가 어떻게 달라지는지 관찰한다.
3. 이 새로운 텔레메트리를 사용하는 새 대시보드, 알림 등을 만든다. 새
   오픈텔레메트리 라이브러리를 프로덕션에 배포하기 전에 이 대시보드들을 미리
   구성해 둔다.
4. 선택적으로, 새로운 텔레메트리를 기존 텔레메트리로 다시 변환하는 처리 규칙을
   컬렉터에 추가한다. 그러면 컬렉터가 동일한 텔레메트리의 두 버전을 모두
   내보내도록 구성할 수 있어 데이터가 겹치는 구간이 생긴다. 이를 통해 기존
   대시보드를 계속 사용하면서 새 대시보드에도 데이터가 채워지도록 할 수 있다.

## 호환성의 한계 {#limits-on-compatibility}

이 섹션에서는 앞서 언급한 [언어 버전 제약 조건](#language-version-support) 외의
호환성 한계를 설명한다.

### 시맨틱 컨벤션 {#semantic-conventions}

위에서 언급했듯이, 오픈텔레메트리는 소프트웨어를 계측하는 방식에 대해 개선된
모델을 가지고 있다. 즉, OpenTracing 계측이 설정하는 "태그(tag)"는
오픈텔레메트리가 설정하는 "속성(attribute)"과 다를 수 있다. 다시 말해, 기존
계측을 교체할 때 오픈텔레메트리가 생성하는 데이터는 OpenTracing이 생성하는
데이터와 다를 수 있다.

다시 한번 명확히 하자면, 계측을 변경할 때는 기존 데이터에 의존하던 대시보드,
알림 등도 반드시 함께 업데이트해야 한다.

### 배기지 {#baggage}

OpenTracing에서 배기지(baggage)는 스팬(Span)에 연결된 SpanContext 객체에 담겨
전달된다. 오픈텔레메트리에서 컨텍스트와 전파는 더 저수준의 개념이다. 스팬,
배기지, 메트릭 계측기 등의 항목은 컨텍스트 객체 안에 담겨 전달된다.

이러한 변경의 결과로, OpenTracing API를 사용해 설정한 배기지는 오픈텔레메트리
전파자(Propagator)에서 사용할 수 없다. 따라서 배기지를 사용할 때는
오픈텔레메트리 API와 OpenTracing API를 함께 사용하는 것을 권장하지 않는다.

구체적으로, OpenTracing API를 사용해 배기지를 설정하면 다음과 같다.

- 오픈텔레메트리 API를 통해서는 접근할 수 없다.
- 오픈텔레메트리 전파자에 의해 주입되지 않는다.

배기지를 사용하고 있다면, 배기지 관련 API 호출을 모두 한 번에 오픈텔레메트리로
전환할 것을 권장한다. 이러한 변경을 프로덕션에 반영하기 전에, 중요한 배기지
항목이 여전히 전파되고 있는지 반드시 확인한다.

### JavaScript에서의 컨텍스트 관리 {#context-management-in-javascript}

JavaScript에서 오픈텔레메트리 API는 Node.js의 `async_hooks`나 브라우저의
`Zones.js`처럼 일반적으로 사용 가능한 컨텍스트 관리자(context manager)를
활용한다. 이러한 컨텍스트 관리자 덕분에, 트레이스가 필요한 모든 메서드에 스팬을
매개변수로 추가하는 것에 비해 트레이스 계측이 훨씬 덜 침습적이고 부담스럽지 않은
작업이 된다.

하지만 OpenTracing API는 이러한 컨텍스트 관리자가 일반적으로 사용되기 이전에
만들어졌다. 현재 활성 스팬을 매개변수로 전달하는 OpenTracing 코드는, 활성 스팬을
컨텍스트 관리자에 저장하는 오픈텔레메트리 코드와 혼용될 때 문제를 일으킬 수
있다. 동일한 트레이스 안에서 두 방식을 함께 사용하면 깨지거나 일치하지 않는
스팬이 생길 수 있으므로 권장하지 않는다.

동일한 트레이스 안에서 두 API를 혼용하는 대신, 코드 경로 전체를 하나의 단위로
OpenTracing에서 오픈텔레메트리로 마이그레이션하여 한 번에 하나의 API만 사용할
것을 권장한다.

## 명세와 구현 세부 사항 {#specification-and-implementation-details}

각 OpenTracing 심이 동작하는 방식에 대한 자세한 내용은 해당 언어별 문서를
참고한다. OpenTracing 심의 설계에 대한 자세한 내용은 [OpenTracing
호환성][ot_spec]을 참고한다.

[.net]: /docs/languages/dotnet/shim/
[go]: https://pkg.go.dev/go.opentelemetry.io/otel/bridge/opentracing
[java]:
  https://github.com/open-telemetry/opentelemetry-java/tree/main/opentracing-shim
[javascript]: https://www.npmjs.com/package/@opentelemetry/shim-opentracing
[opentracing]: https://opentracing.io
[ot_spec]: /docs/specs/otel/compatibility/opentracing/
[python]:
  https://opentelemetry-python.readthedocs.io/en/stable/shim/opentracing_shim/opentracing_shim.html
[c++]:
  https://github.com/open-telemetry/opentelemetry-cpp/tree/main/opentracing-shim
