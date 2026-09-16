---
title: 샘플링
weight: 80
default_lang_commit: 4251379e16ba6263a45644c86bbbc9be17d58a06
---

[샘플링](/docs/concepts/sampling/)은 시스템이 생성하는 스팬의 양을 제한하는
프로세스이다. 사용해야 할 정확한 샘플러(sampler)는 구체적인 필요에 따라
다르지만, 일반적으로 트레이스가 시작될 때 결정을 내리고 그 샘플링 결정이 다른
서비스로 전파되도록 해야 한다.

[`Sampler`][]는 다음과 같이
[`WithSampler`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#WithSampler)
옵션을 사용해 트레이서 프로바이더에 설정할 수 있다.

```go
provider := trace.NewTracerProvider(
    trace.WithSampler(trace.AlwaysSample()),
)
```

[`AlwaysSample`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#AlwaysSample)와
[`NeverSample`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#NeverSample)는
이름 그대로의 값이다. `AlwaysSample`은 모든 스팬이 샘플링됨을 의미하고,
`NeverSample`은 어떤 스팬도 샘플링되지 않음을 의미한다. 처음 시작할 때나 개발
환경에서는 `AlwaysSample`을 사용한다.

그 밖의 샘플러는 다음과 같다.

- [`TraceIDRatioBased`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#TraceIDRatioBased):
  샘플러에 주어진 비율에 기반하여 스팬의 일부를 샘플링한다. .5로 설정하면 전체
  스팬의 절반이 샘플링된다.
- [`ParentBased`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#ParentBased):
  스팬의 부모에 따라 다르게 동작하는 샘플러 데코레이터(decorator)이다. 스팬에
  부모가 없으면, 데코레이팅된 샘플러가 샘플링 결정을 내리는 데 사용된다.
  기본적으로 `ParentBased`는 부모가 샘플링된 스팬은 샘플링하고, 부모가
  샘플링되지 않은 스팬은 샘플링하지 않는다.

기본적으로 트레이서 프로바이더는 `AlwaysSample` 샘플러를 사용하는 `ParentBased`
샘플러를 사용한다.

프로덕션 환경에서는 `TraceIDRatioBased` 샘플러를 사용하는 `ParentBased` 샘플러를
사용하는 것을 고려한다.

## 커스텀 샘플러 {#custom-samplers}

내장 샘플러가 필요에 맞지 않는다면, [`Sampler`][] 인터페이스를 구현하여 커스텀
샘플러를 만들 수 있다. 커스텀 샘플러는 다음 두 메서드를 구현해야 한다.

- `ShouldSample(parameters SamplingParameters) SamplingResult`: 제공된
  매개변수를 기반으로 샘플링 결정을 내린다.
- `Description() string`: 샘플러에 대한 설명을 반환한다.

> [!IMPORTANT] 부모 tracestate 보존
>
> `ShouldSample`에서는 `SamplingResult`에 부모의 tracestate를 반드시 보존해야
> 한다. 그렇게 하지 않으면 벤더별(vendor-specific) 또는 애플리케이션별 트레이스
> 데이터를 전달하기 위해 tracestate에 의존하는 분산 시스템에서 컨텍스트 전파가
> 깨진다.
>
> 부모 스팬 컨텍스트에서 tracestate를 추출한다.
>
> ```go
> psc := trace.SpanContextFromContext(parameters.ParentContext)
> ```
>
> `SamplingResult`를 만들 때 `psc.TraceState()`를 그대로 전달한다.

### 예제 {#example}

다음 예제는 tracestate를 올바르게 보존하면서 속성 값을 기반으로 스팬을
샘플링하는 커스텀 샘플러를 보여준다.

```go
package main

import (
    "context"

    "go.opentelemetry.io/otel/attribute"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/trace"
)

// AttributeBasedSampler samples spans based on an attribute value.
// It always samples spans with the "high.priority" attribute set to true.
type AttributeBasedSampler struct {
    fallback sdktrace.Sampler
}

// NewAttributeBasedSampler creates a new AttributeBasedSampler.
func NewAttributeBasedSampler(fallback sdktrace.Sampler) *AttributeBasedSampler {
    return &AttributeBasedSampler{fallback: fallback}
}

func (s *AttributeBasedSampler) ShouldSample(p sdktrace.SamplingParameters) sdktrace.SamplingResult {
    // Always extract the parent span context to get the tracestate.
    psc := trace.SpanContextFromContext(p.ParentContext)

    // Check if any attribute indicates high priority.
    for _, attr := range p.Attributes {
        if attr.Key == "high.priority" && attr.Value.AsBool() {
            return sdktrace.SamplingResult{
                Decision:   sdktrace.RecordAndSample,
                Attributes: p.Attributes,
                // Critical: preserve the parent's tracestate
                Tracestate: psc.TraceState(),
            }
        }
    }

    // Fall back to the default sampler for other spans.
    result := s.fallback.ShouldSample(p)

    // Ensure tracestate is preserved even when using fallback.
    // Built-in samplers already handle this, but it's good practice to verify.
    return sdktrace.SamplingResult{
        Decision:   result.Decision,
        Attributes: result.Attributes,
        Tracestate: psc.TraceState(),
    }
}

func (s *AttributeBasedSampler) Description() string {
    return "AttributeBasedSampler"
}
```

### 커스텀 샘플러 사용하기 {#using-your-custom-sampler}

커스텀 샘플러를 트레이서 프로바이더와 함께 사용할 수 있다.

```go
sampler := NewAttributeBasedSampler(sdktrace.TraceIDRatioBased(0.1))

provider := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sampler),
)
```

`ParentBased` 샘플러와 조합할 수도 있다.

```go
provider := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(
        sdktrace.ParentBased(
            NewAttributeBasedSampler(sdktrace.TraceIDRatioBased(0.1)),
        ),
    ),
)
```

### 추가 고려 사항 {#additional-considerations}

커스텀 샘플러를 구현할 때는 다음 사항을 유의한다.

1. **부모 샘플링 결정 무시하기**: 부모의 샘플링 결정을 존중하고 싶다면, 샘플러를
   `ParentBased`로 감싸거나 `psc.IsSampled()`를 수동으로 확인한다.

2. **ShouldSample에서의 무거운 연산**: `ShouldSample` 함수는 스팬이 생성될
   때마다 동기적으로 호출된다. 네트워크 호출이나 복잡한 연산처럼 성능에 영향을
   줄 수 있는 비용이 큰 작업은 피한다.

[`Sampler`]: https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#Sampler
