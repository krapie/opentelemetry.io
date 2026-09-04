---
title: 계측 스코프
weight: 80
default_lang_commit: e8873fbb81024f7a5ec6d7a627b5242f2915c291
---

[계측 스코프(instrumentation scope)](/docs/specs/otel/common/instrumentation-scope/)는
내보내는 텔레메트리와 연관되는 소프트웨어의 논리적 단위이다. 모듈, 패키지,
클래스, 라이브러리, 프레임워크 등 개발자가 하나의 텔레메트리 소스를 다른 소스와
구분하기 위해 선택하는 의미 있는 경계라면 무엇이든 나타낼 수 있다.

## 스코프는 어떻게 정의되는가 {#how-a-scope-is-defined}

스코프는 `(name, version, schema_url, attributes)` 튜플(tuple)로 식별되며, 이 중
`version`, `schema_url`, `attributes`는 선택 사항이다. `name`은 소프트웨어의
논리적 단위를 고유하게 식별해야 한다. 예를 들어 라이브러리, 클래스, 모듈의
정규화된 이름(fully qualified name)이 여기에 해당한다.

프로바이더로부터 트레이서, 미터, 로거를 가져올 때 스코프를 지정한다. 이후 해당
인스턴스가 생성하는 모든 스팬, 메트릭, 로그 레코드에는 스코프가 태그로 지정된다.

- **라이브러리 및 프레임워크의 경우**: 라이브러리의 정규화된 이름과 버전을
  스코프로 사용한다. 오픈텔레메트리를 기본 지원하지 않는 라이브러리를 위한 계측
  라이브러리를 작성하는 경우에는, 해당 계측 라이브러리 자체의 이름과 버전을
  사용한다.
- **애플리케이션 코드의 경우**: 일반적으로 `CheckoutService`와 같은 클래스 또는
  모듈 이름을 선택한다.

## 스코프가 중요한 이유 {#why-scopes-matter}

옵저버빌리티 백엔드에서는 스코프별로 텔레메트리를 필터링, 그룹화, 비교할 수
있다. 이를 통해 어떤 라이브러리 버전이 지연(latency)을 유발하는지 식별하거나,
특정 모듈에서 발생하는 시그널을 분리하거나, 동일한 구성 요소의 여러 버전 간
동작을 비교할 수 있다.

## 트레이스에서의 스코프 {#scopes-in-a-trace}

다음 다이어그램은 색상으로 구분되고 범례(legend)에 표시된, 서로 다른 여섯 개의
계측 스코프에서 발생한 스팬을 포함하는 트레이스를 보여준다.

- `http-framework` 스코프는 루트 `/api/placeOrder` 스팬을 생성한다.
- `CheckoutService` 스코프는 `CheckoutService::placeOrder`,
  `CheckoutService::prepareOrderItems`, `CheckoutService::checkout`을 생성한다.
  이 세 스팬은 모두 이름 `CheckoutService`로 가져온 동일한 트레이서 인스턴스가
  생성했기 때문에 동일한 계측 스코프를 공유한다.
- `CartService`와 `ProductService` 스코프는 각각 자신의 애플리케이션 구성
  요소에서 하나씩 스팬을 생성한다.
- `Cache library`와 `DB library` 스코프는 라이브러리 코드에서 발생한 스팬을
  생성하며, 라이브러리 이름과 버전으로 그룹화된다.

![계측 스코프별로 색상이 구분된 스팬을 보여주는 트레이스 워터폴. 하단의 범례는 각 색상을 해당 스코프 이름에 매핑한다.](spans-with-instrumentation-scope.svg)
