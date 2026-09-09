---
title: 수동 스팬 속성
aliases: [manual_span_attributes, ../manual-span-attributes]
default_lang_commit: 8387f1584794f10784684cff32d0a4b8769ce50b
---

이 페이지는 데모 전반에서 사용되는 수동 스팬 속성을 나열한다:

## 광고 {#ad}

| 이름                        | 타입   | 설명                                   |
| --------------------------- | ------ | -------------------------------------- |
| `app.ads.category`          | string | 반환된 광고의 카테고리                 |
| `app.ads.contextKeys`       | string | 관련 광고를 찾는 데 사용된 컨텍스트 키 |
| `app.ads.contextKeys.count` | number | 사용된 고유 컨텍스트 키의 개수         |
| `app.ads.count`             | number | 사용자에게 반환된 광고의 개수          |
| `app.ads.ad_request_type`   | string | `targeted` 또는 `not_targeted` 중 하나 |
| `app.ads.ad_response_type`  | string | `targeted` 또는 `random` 중 하나       |

## 장바구니 {#cart}

| 이름                   | 타입   | 설명                             |
| ---------------------- | ------ | -------------------------------- |
| `app.cart.items.count` | number | 장바구니에 있는 고유 항목의 개수 |
| `app.product.id`       | string | 장바구니 항목의 상품 ID          |
| `app.product.quantity` | string | 장바구니 항목의 수량             |
| `app.user.id`          | string | 사용자 ID                        |

## 체크아웃 {#checkout}

| 이름                         | 타입   | 설명                             |
| ---------------------------- | ------ | -------------------------------- |
| `app.cart.items.count`       | number | 장바구니에 있는 전체 항목의 개수 |
| `app.order.amount`           | number | 주문 금액                        |
| `app.order.id`               | string | 주문 ID                          |
| `app.order.items.count`      | number | 주문에 있는 고유 항목의 개수     |
| `app.payment.transaction.id` | string | 결제 트랜잭션 ID                 |
| `app.shipping.amount`        | number | 배송 금액                        |
| `app.shipping.tracking.id`   | string | 배송 추적 ID                     |
| `app.user.currency`          | string | 사용자 통화                      |
| `app.user.id`                | string | 사용자 ID                        |

## 통화 {#currency}

| 이름                           | 타입   | 설명              |
| ------------------------------ | ------ | ----------------- |
| `app.currency.conversion.from` | string | 변환 전 통화 코드 |
| `app.currency.conversion.to`   | string | 변환 후 통화 코드 |

## 이메일 {#email}

| 이름                  | 타입   | 설명                      |
| --------------------- | ------ | ------------------------- |
| `app.email.recipient` | string | 주문 확인에 사용된 이메일 |
| `app.order.id`        | string | 주문 ID                   |

## 프론트엔드 {#frontend}

| 이름                     | 타입   | 설명                             |
| ------------------------ | ------ | -------------------------------- |
| `app.cart.size`          | number | 장바구니에 있는 전체 항목의 개수 |
| `app.cart.items.count`   | number | 장바구니에 있는 고유 항목의 개수 |
| `app.cart.shipping.cost` | number | 장바구니 배송비                  |
| `app.cart.total.price`   | number | 장바구니 총 가격                 |
| `app.currency`           | string | 사용자 통화                      |
| `app.currency.new`       | string | 설정할 새 통화                   |
| `app.order.total`        | number | 주문 총 비용                     |
| `app.product.id`         | string | 상품 ID                          |
| `app.product.quantity`   | number | 상품 수량                        |
| `app.products.count`     | number | 표시된 전체 상품 수              |
| `app.request.id`         | string | 요청 ID                          |
| `app.session.id`         | string | 세션 ID                          |
| `app.user.id`            | string | 사용자 ID                        |

## 부하 생성기 {#load-generator}

| 이름      | 타입 | 설명 |
| --------- | ---- | ---- |
| 아직 없음 |      |      |

## 결제 {#payment}

| 이름                     | 타입    | 설명                                      |
| ------------------------ | ------- | ----------------------------------------- |
| `app.payment.amount`     | number  | 총 결제 금액                              |
| `app.payment.card_type`  | string  | 결제에 사용된 카드 유형                   |
| `app.payment.card_valid` | boolean | 사용된 카드의 유효 여부                   |
| `app.payment.charged`    | boolean | 결제 성공 여부(부하 생성기 사용 시 false) |

## 상품 카탈로그 {#product-catalog}

| 이름                        | 타입   | 설명                         |
| --------------------------- | ------ | ---------------------------- |
| `app.product.id`            | string | 상품 ID                      |
| `app.product.name`          | string | 상품 이름                    |
| `app.products.count`        | number | 카탈로그에 있는 상품 개수    |
| `app.products_search.count` | number | 검색 결과로 반환된 상품 개수 |

## 견적 {#quote}

| 이름                    | 타입   | 설명                |
| ----------------------- | ------ | ------------------- |
| `app.quote.items.count` | number | 배송할 전체 항목 수 |
| `app.quote.cost.total`  | number | 총 배송 견적        |

## 추천 {#recommendation}

| 이름                             | 타입    | 설명                      |
| -------------------------------- | ------- | ------------------------- |
| `app.filtered_products.count`    | number  | 반환된 필터링된 상품 개수 |
| `app.products.count`             | number  | 카탈로그에 있는 상품 개수 |
| `app.products_recommended.count` | number  | 반환된 추천 상품 개수     |
| `app.cache_hit`                  | boolean | 캐시 접근 여부            |

## 배송 {#shipping}

| 이름                      | 타입   | 설명         |
| ------------------------- | ------ | ------------ |
| `app.shipping.cost.total` | number | 총 배송 비용 |
