---
title: Go 계측 Auto SDK
linkTitle: Auto SDK
description: Auto SDK로 수동 스팬을 제로 코드 eBPF 스팬과 통합한다.
weight: 16
default_lang_commit: 4edfbfc2ff38123678ca63eca95de94ede457623
---

[OBI](/docs/zero-code/obi)와 같은 도구가 사용하는 오픈텔레메트리(OpenTelemetry)
Go eBPF 계측 프레임워크는 Auto SDK를 통해 수동으로 계측된 오픈텔레메트리 스팬과
통합하는 기능을 제공한다.

## Auto SDK란? {#what-is-the-auto-sdk}

Auto SDK는 Go eBPF 자동 계측과의 호환성을 위해 설계된, 완전히 구현된 사용자 정의
오픈텔레메트리 Go SDK다. 이를 통해 자동으로 계측된 패키지(예: `net/http`)가 수동
스팬과의 컨텍스트 전파를 지원할 수 있다.

## 언제 사용해야 하는가? {#when-should-i-use-it}

오픈텔레메트리 Go eBPF 계측은 현재 제한된 수의 패키지만 지원한다. 이 계측을
확장하여 코드 내에서 사용자 정의 스팬을 만들고 싶을 수 있다. Auto SDK는 사용자
정의 스팬을, 자동 스팬에서도 사용되는 공유 트레이스 컨텍스트로 계측함으로써 이를
가능하게 한다.

## 어떻게 사용하는가? {#how-do-i-use-it}

[OpenTelemetry Go v1.36.0](https://github.com/open-telemetry/opentelemetry-go/releases/tag/v1.36.0)
릴리스부터, Auto SDK는 표준 Go API와 함께 간접 의존성으로 자동으로 임포트된다.
`go.mod`에서 `go.opentelemetry.io/auto/sdk`를 확인하면 프로젝트에 Auto SDK가
있는지 알 수 있다.

Auto SDK로 수동 스팬을 만드는 것은 표준
[Go 계측](/docs/languages/go/instrumentation/)으로 스팬을 만드는 것과 본질적으로
동일하다.

Auto SDK를 사용할 수 있으면, `tracer.Start()`로 수동 스팬을 만드는 것만큼
간단하게 사용할 수 있다.

```go
package main

import (
	"log"
	"net/http"

	"go.opentelemetry.io/otel"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// Get tracer
		tracer := otel.Tracer("example-server")

		// Start a manual span
		_, span := tracer.Start(r.Context(), "manual-span")
		defer span.End()

		// Add an attribute for demonstration
		span.SetAttributes()
		span.AddEvent("Request handled")
	})

	log.Println("Server running at :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

이 예제에서 eBPF 프레임워크는 들어오는 HTTP 요청을 자동으로 계측한 다음, 수동
스팬을 HTTP 라이브러리에서 계측된 것과 동일한 트레이스에 연결한다. 이 샘플에는
초기화된 TracerProvider가 없다는 점에 유의한다. Auto SDK는 SDK를 활성화하는 데
핵심적인 자체 TracerProvider를 등록한다.

본질적으로, Go 제로 코드 에이전트로 계측된 애플리케이션에서 수동 스팬을 만드는
것 외에 Auto SDK를 활성화하기 위해 해야 할 일은 없다. 전역 TracerProvider를 직접
등록하지 않는 한, Auto SDK가 자동으로 활성화된다.

> [!WARNING]
>
> 전역 TracerProvider를 직접 설정하면 Auto SDK와 충돌하여 수동 스팬이 eBPF 기반
> 스팬과 제대로 상관되지 못하게 막는다. eBPF로도 계측되는 Go 애플리케이션에서
> 수동 스팬을 만드는 경우, 자체 전역 TracerProvider를 초기화하지 않는다.

### Auto SDK TracerProvider {#auto-sdk-tracerprovider}

대부분의 사용 사례에서는 Auto SDK의 내장 TracerProvider와 직접 상호작용할 필요가
없다. 그러나 특정 고급 사용 사례에서는 Auto SDK의 TracerProvider를 직접 구성하고
싶을 수 있다.
[`auto.TracerProvider()`](https://pkg.go.dev/go.opentelemetry.io/auto/sdk)
함수로 이에 접근할 수 있다.

```go
import (
	"go.opentelemetry.io/otel"
    autosdk "go.opentelemetry.io/auto/sdk"
)

func main() {
	tp := autosdk.TracerProvider()
	otel.SetTracerProvider(tp)
}
```

## Auto SDK는 어떻게 동작하는가? {#how-does-the-auto-sdk-work}

애플리케이션이 오픈텔레메트리 eBPF로 계측되면, eBPF 프로그램은 애플리케이션에
`go.opentelemetry.io/auto/sdk` 의존성이 있는지 확인한다(이 의존성은
`go.opentelemetry.io/otel`에 기본으로 포함되어 있으며, 명시적으로 임포트할
필요가 없다는 점을 기억한다). 발견되면, eBPF 프로그램은 전역 오픈텔레메트리
SDK의 불리언 값을 활성화하여 오픈텔레메트리가 Auto SDK TracerProvider를
사용하도록 지시한다.

그러면 Auto SDK는 다른 SDK와 매우 유사하게 동작하며, 명세에서 요구하는 모든
기능을 구현한다. 주요 차이점은 Auto SDK 역시 eBPF로 자동 계측되어 다른 eBPF 계측
라이브러리와 컨텍스트 전파를 통합한다는 것이다.

본질적으로 Auto SDK는, 오픈텔레메트리 eBPF가 다른 패키지에 하는 것과 마찬가지로
오픈텔레메트리 함수 심볼을 계측함으로써, 표준 오픈텔레메트리 API와의 컨텍스트
전파를 식별하고 조율하는 방식이다.
