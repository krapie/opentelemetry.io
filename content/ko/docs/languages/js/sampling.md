---
title: 샘플링
weight: 80
description: 생성되는 텔레메트리의 양 줄이기
default_lang_commit: 06837fe15457a584f6a9e09579be0f0400593d57
---

[샘플링](/docs/concepts/sampling/)은 시스템이 생성하는 트레이스의 양을 제한하는
프로세스이다. JavaScript SDK는 여러
[헤드 샘플러](/docs/concepts/sampling#head-sampling)를 제공한다.

## 기본 동작 {#default-behavior}

기본적으로 모든 스팬이 샘플링되며, 따라서 트레이스의 100%가 샘플링된다. 데이터
양을 관리할 필요가 없다면, 샘플러를 별도로 설정하지 않아도 된다.

## TraceIDRatioBasedSampler {#traceidratiobasedsampler}

샘플링할 때 가장 흔히 사용하는 헤드 샘플러는 TraceIdRatioBasedSampler이다. 이는
매개변수로 전달한 비율에 따라 결정론적으로 트레이스를 샘플링한다.

### 환경 변수 {#environment-variables}

TraceIdRatioBasedSampler는 환경 변수로도 구성할 수 있다.

```shell
export OTEL_TRACES_SAMPLER="traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"
```

이렇게 하면 SDK는 트레이스의 10%만 생성되도록 스팬을 샘플링한다.

### Node.js {#nodejs}

TraceIdRatioBasedSampler는 코드에서도 구성할 수 있다. 다음은 Node.js의 예시이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import { TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-node';

const samplePercentage = 0.1;

const sdk = new NodeSDK({
  // Other SDK configuration parameters go here
  sampler: new TraceIdRatioBasedSampler(samplePercentage),
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const { TraceIdRatioBasedSampler } = require('@opentelemetry/sdk-trace-node');

const samplePercentage = 0.1;

const sdk = new NodeSDK({
  // Other SDK configuration parameters go here
  sampler: new TraceIdRatioBasedSampler(samplePercentage),
});
```

{{% /tab %}} {{< /tabpane >}}

### 브라우저 {#browser}

TraceIdRatioBasedSampler는 코드에서도 구성할 수 있다. 다음은 브라우저
애플리케이션의 예시이다.

{{< tabpane text=true >}} {{% tab TypeScript %}}

```ts
import {
  WebTracerProvider,
  TraceIdRatioBasedSampler,
} from '@opentelemetry/sdk-trace-web';

const samplePercentage = 0.1;

const provider = new WebTracerProvider({
  sampler: new TraceIdRatioBasedSampler(samplePercentage),
});
```

{{% /tab %}} {{% tab JavaScript %}}

```js
const {
  WebTracerProvider,
  TraceIdRatioBasedSampler,
} = require('@opentelemetry/sdk-trace-web');

const samplePercentage = 0.1;

const provider = new WebTracerProvider({
  sampler: new TraceIdRatioBasedSampler(samplePercentage),
});
```

{{% /tab %}} {{< /tabpane >}}
