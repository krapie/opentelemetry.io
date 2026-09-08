---
title: 기능 플래그
aliases:
  - feature_flags
  - scenarios
  - services/feature-flag
  - services/featureflagservice
cSpell:ignore: loadgenerator OLJCESPC7Z
default_lang_commit: 8387f1584794f10784684cff32d0a4b8769ce50b
---

데모는 다양한 시나리오를 시뮬레이션하는 데 사용할 수 있는 여러 기능
플래그(feature flag)를 제공한다. 이 플래그는
[OpenFeature](https://openfeature.dev)를 지원하는 간단한 기능 플래그 서비스인
[`flagd`](https://flagd.dev)로 관리된다.

데모 실행 중 <http://localhost:8080/feature>에서 제공되는 사용자 인터페이스를
통해 플래그 값을 변경할 수 있다. 이 사용자 인터페이스를 통한 변경 사항은 flagd
서비스에 반영된다.

사용자 인터페이스를 통해 기능 플래그를 변경할 때는 두 가지 옵션이 있다.

- **Basic View**: 각 기능 플래그에 대해 기본 변형(원시 파일을 통해 구성할 때
  변경해야 하는 것과 동일한 옵션)을 선택하고 저장할 수 있는 사용자 친화적인
  화면이다. 현재 Basic View는 부분 타겟팅(fractional targeting)을 지원하지
  않는다.

- **Advanced View**: 원시 구성 JSON 파일을 불러와 브라우저 내에서 편집할 수 있는
  화면이다. 이 화면은 원시 JSON 파일을 편집하는 데 따르는 유연성을 제공하며,
  동시에 JSON이 유효하고 제공된 구성 값이 올바른지 확인하는 스키마 검사 기능도
  제공한다.

## 구현된 기능 플래그 {#implemented-feature-flags}

| 기능 플래그                         | 서비스        | 설명                                                                                                                                        |
| ----------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `adServiceFailure`                  | 광고          | `GetAds` 호출의 1/10 확률로 오류를 발생시킨다                                                                                               |
| `adServiceManualGc`                 | 광고          | 광고 서비스에서 전체 수동 가비지 컬렉션을 트리거한다                                                                                        |
| `adServiceHighCpu`                  | 광고          | 광고 서비스에서 높은 CPU 부하를 트리거한다. CPU 스로틀링을 데모하려면 CPU 리소스 제한을 설정한다                                            |
| `cartServiceFailure`                | 장바구니      | `EmptyCart`가 호출될 때마다 오류를 발생시킨다                                                                                               |
| `emailMemoryLeak`                   | 이메일        | `email` 서비스에서 메모리 누수를 시뮬레이션한다                                                                                             |
| `productCatalogFailure`             | 상품 카탈로그 | 상품 ID가 `OLJCESPC7Z`인 `GetProduct` 요청에 대해 오류를 발생시킨다                                                                         |
| `recommendationServiceCacheFailure` | 추천          | 지수적으로 증가하는 캐시로 인해 메모리 누수를 발생시킨다. 1.4배씩 증가하며, 요청의 50%가 증가를 트리거한다                                  |
| `paymentServiceFailure`             | 결제          | `charge` 메서드를 호출할 때 오류를 발생시킨다                                                                                               |
| `paymentServiceUnreachable`         | 체크아웃      | PaymentService 호출 시 잘못된 주소를 사용해 PaymentService를 사용할 수 없는 것처럼 보이게 만든다                                            |
| `loadgeneratorFloodHomepage`        | 부하 생성기   | 홈페이지에 대량의 요청을 쏟아붓기 시작하며, 상태에 대한 flagd JSON을 변경해 설정할 수 있다                                                  |
| `kafkaQueueProblems`                | Kafka         | Kafka 큐에 과부하를 일으키는 동시에 컨슈머 측 지연을 발생시켜 랙(lag) 스파이크를 유발한다                                                   |
| `imageSlowLoad`                     | 프론트엔드    | Envoy 결함 주입(fault injection)을 활용해 프론트엔드에서 상품 이미지 로딩을 지연시킨다                                                      |
| `failedReadinessProbe`              | 장바구니      | 레디니스 프로브(readiness probe)가 unhealthy 상태로 실패하도록 강제해 파드의 "NotReady" 상태를 시뮬레이션한다. 쿠버네티스 배포에만 적용된다 |

## 안내형 디버깅 시나리오 {#guided-debugging-scenario}

`recommendationServiceCacheFailure` 시나리오에는 오픈텔레메트리(OpenTelemetry)로
메모리 누수를 디버깅하는 방법을 이해하는 데 도움이 되는
[전용 안내 문서](recommendation-cache/)가 있다.

## 기능 플래그 아키텍처 {#feature-flag-architecture}

flagd가 어떻게 동작하는지 자세히 알아보려면 [flagd 문서](https://flagd.dev)를
참고하고, OpenFeature가 어떻게 동작하는지와 OpenFeature API 문서를 확인하려면
[OpenFeature](https://openfeature.dev) 웹사이트를 참고한다.
