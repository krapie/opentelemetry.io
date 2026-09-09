---
title: 결제 서비스
linkTitle: 결제
aliases: [paymentservice]
default_lang_commit: faacefc5407b59c9193caeb839616406cb8a5708
---

이 서비스는 주문에 대한 신용카드 결제를 처리하는 역할을 담당한다. 신용카드가
유효하지 않거나 결제를 처리할 수 없는 경우 오류를 반환한다.

[결제 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/payment/)

## 제로 코드 계측 {#zero-code-instrumentation}

Node.js 기반인 이 서비스는 시작 시
`@opentelemetry/auto-instrumentations-node/register` 모듈을 require 함으로써
오픈텔레메트리(OpenTelemetry) Node.js 제로 코드 계측(Zero-code
Instrumentation)을 활용한다. 내보내기 엔드포인트, 리소스 속성, 서비스 이름은
환경 변수를 기반으로 자동으로 설정된다. 이는 서비스의 `package.json` 시작
스크립트나 `NODE_OPTIONS`를 통해 수행할 수 있다.

```json
"scripts": {
  "start": "node --require @opentelemetry/auto-instrumentations-node/register index.js"
}
```

## 트레이스 {#traces}

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬을 가져올 수 있다.

```javascript
const span = opentelemetry.trace.getActiveSpan();
```

스팬에 속성을 추가하려면 스팬 객체의 `setAttributes`를 사용한다.
`chargeServiceHandler` 함수에서는 속성 키/값 쌍을 담은 익명 객체(맵)로 스팬에
속성이 추가된다.

```javascript
span?.setAttributes({
  'demo.payment.amount': parseFloat(`${amount.units}.${amount.nanos}`).toFixed(
    2,
  ),
});
```

### 스팬 예외와 상태 {#span-exceptions-and-status}

스팬 객체의 `recordException` 함수를 사용하면 처리된 오류의 전체 스택 트레이스를
담은 스팬 이벤트를 생성할 수 있다. 예외를 기록할 때는 그에 맞게 스팬 상태도
설정해야 한다. 이는 `charge.js`의 `charge` 함수에서 확인할 수 있다.

```javascript
span.recordException(err);
span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
```

## 메트릭 {#metrics}

### 미터와 계측기 생성하기 {#creating-meters-and-instruments}

`@opentelemetry/api` 패키지를 사용해 미터를 생성할 수 있다. 아래와 같이 미터를
생성한 다음, 생성한 미터를 사용해 계측기(instrument)를 생성할 수 있다.

```javascript
const { metrics } = require('@opentelemetry/api');

const meter = metrics.getMeter('payment');
const transactionsCounter = meter.createCounter('demo.payment.transactions');
```

미터와 계측기는 계속 유지되어야 한다. 즉, 가능하다면 미터나 계측기는 한 번만
얻은 뒤 필요에 따라 재사용해야 한다.

## 로그 {#logs}

TBD

## 배기지 {#baggage}

이 서비스에서는 요청이(부하 생성기로부터 온) 합성(synthetic) 요청인지 확인하기
위해 오픈텔레메트리 배기지(Baggage)를 활용한다. 합성 요청에는 요금이 청구되지
않으며, 이는 스팬 속성으로 표시된다. 실제 결제 처리를 수행하는 `charge.js`
파일에는 배기지를 확인하는 로직이 있다.

```javascript
// check baggage for synthetic_request=true, and add charged attribute accordingly
const baggage = propagation.getBaggage(context.active());
if (
  baggage &&
  baggage.getEntry('synthetic_request') &&
  baggage.getEntry('synthetic_request').value === 'true'
) {
  span.setAttribute('demo.payment.charged', false);
} else {
  span.setAttribute('demo.payment.charged', true);
}
```
