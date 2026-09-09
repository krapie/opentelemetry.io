---
title: 리시버 빌드하기
linkTitle: 리시버
weight: 100
aliases:
  - /docs/collector/trace-receiver
  - /docs/collector/building/receiver
# prettier-ignore
cSpell:ignore: backendsystem crand debugexporter mapstructure pcommon pdata ptrace rcvr resourcespans struct structs tailtracer telemetrygen uber
default_lang_commit: 3560c03d5cbe845c6189e6e30441434c7760eca0
---

<!-- markdownlint-disable heading-increment no-duplicate-heading -->

오픈텔레메트리(OpenTelemetry)는
[분산 트레이싱](/docs/concepts/glossary/#distributed-tracing)을 다음과 같이
정의한다.

> 애플리케이션을 구성하는 서비스들이 처리하는 단일 요청의 진행 과정을 추적하는
> 것을 트레이스(trace)라고 한다. 요청은 사용자 또는 애플리케이션에 의해 시작될
> 수 있다. 분산 트레이싱은 프로세스, 네트워크, 보안 경계를 가로지르는 트레이싱의
> 한 형태다.

분산 트레이스는 애플리케이션 중심적인 방식으로 정의되지만, 시스템을 통과하는
_모든_ 요청에 대한 타임라인으로 생각할 수 있다. 각 분산 트레이스는 요청이
시작부터 끝까지 얼마나 걸렸는지 보여주고, 이를 완료하기 위해 거친 단계를
세분화해서 보여준다.

시스템이 트레이싱 텔레메트리를 생성한다면, 해당 텔레메트리를 수신하고 변환하도록
설계된 트레이스 리시버로 [오픈텔레메트리 컬렉터](/docs/collector/)를 구성할 수
있다. 리시버는 데이터를 원래 형식에서 오픈텔레메트리 트레이스 모델로 변환해서
컬렉터가 이를 처리할 수 있게 한다.

트레이스 리시버를 구현하려면 다음이 필요하다.

- 트레이스 리시버가 컬렉터 config.yaml에서 자신의 설정을 수집하고 검증할 수
  있도록 하는 `Config` 구현체.

- 컬렉터가 트레이스 리시버 구성 요소를 제대로 인스턴스화할 수 있도록 하는
  `receiver.Factory` 구현체.

- 텔레메트리를 수집하고, 내부 트레이스 표현으로 변환한 다음, 파이프라인의 다음
  컨슈머(consumer)로 텔레메트리를 전달하는 `receiver.Traces` 구현체.

이 튜토리얼은 풀(pull) 연산을 시뮬레이션하고 그 결과로 트레이스를 생성하는
`tailtracer`라는 트레이스 리시버를 만드는 방법을 보여준다.

## 리시버 개발 및 테스트 환경 설정하기 {#setting-up-receiver-development-and-testing-environment}

먼저, [커스텀 컬렉터 빌드하기](/docs/collector/extend/ocb/) 튜토리얼을 사용해
`otelcol-dev`라는 이름의 컬렉터 인스턴스를 생성한다. 필요한 작업은
[오픈텔레메트리 컬렉터 빌더 구성하기](/docs/collector/extend/ocb/#configure-the-opentelemetry-collector-builder)에
설명된 `builder-config.yaml`을 복사하고 빌더를 실행하는 것뿐이다. 그 결과로
다음과 같은 폴더 구조가 생겨야 한다.

```text
.
├── builder-config.yaml
├── ocb
└── otelcol-dev
    ├── components.go
    ├── components_test.go
    ├── go.mod
    ├── go.sum
    ├── main.go
    ├── main_others.go
    ├── main_windows.go
    └── otelcol-dev
```

트레이스 리시버를 제대로 테스트하려면 컬렉터가 텔레메트리를 전송할 수 있는 분산
트레이싱 백엔드가 필요할 수도 있다. 여기서는
[Jaeger](https://www.jaegertracing.io/docs/latest/getting-started/)를 사용한다.
`Jaeger` 인스턴스가 실행 중이지 않다면, 다음 명령으로 Docker를 사용해 간단히
하나를 시작할 수 있다.

```sh
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 14317:4317 \
  -p 14318:4318 \
  jaegertracing/jaeger:latest
```

컨테이너가 실행되면, 다음 URL을 통해 Jaeger UI에 접근할 수 있다.
<http://localhost:16686/>

이제 컬렉터 구성 요소와 파이프라인을 설정하기 위해 `config.yaml`이라는 이름의
컬렉터 구성 파일을 만든다.

```sh
touch config.yaml
```

지금은 `otlp` 리시버와 `otlp`, `debug` 익스포터로 이루어진 기본적인 트레이스
파이프라인만 있으면 된다. `config.yaml` 파일은 다음과 같은 모습이어야 한다.

> config.yaml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  debug:
    verbosity: detailed
  otlp/jaeger:
    endpoint: localhost:14317
    tls:
      insecure: true
    sending_queue:
      batch:

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp/jaeger, debug]
  telemetry:
    logs:
      level: debug
```

> [!NOTE]
>
> 여기서는 단순화를 위해 `otlp` 익스포터 설정에서 `insecure` 플래그를 사용한다.
> 프로덕션 환경에서 컬렉터를 실행할 때는
> [이 가이드](/docs/collector/configuration/#setting-up-certificates)를 따라
> 안전한 통신을 위한 TLS 인증서나 상호 인증을 위한 mTLS를 사용해야 한다.

컬렉터가 제대로 설정되었는지 확인하려면 다음 명령을 실행한다.

```sh
./otelcol-dev/otelcol-dev --config config.yaml
```

출력은 다음과 같은 모습일 수 있다.

```log
2023-11-08T18:38:37.183+0800	info	service@v0.88.0/telemetry.go:84	Setting up own telemetry...
2023-11-08T18:38:37.185+0800	info	service@v0.88.0/telemetry.go:201	Serving Prometheus metrics	{"address": ":8888", "level": "Basic"}
2023-11-08T18:38:37.185+0800	debug	exporter@v0.88.0/exporter.go:273	Stable component.	{"kind": "exporter", "data_type": "traces", "name": "otlp/jaeger"}
2023-11-08T18:38:37.186+0800	info	exporter@v0.88.0/exporter.go:275	Development component. May change in the future.	{"kind": "exporter", "data_type": "traces", "name": "debug"}
2023-11-08T18:38:37.186+0800	debug	receiver@v0.88.0/receiver.go:294	Stable component.	{"kind": "receiver", "name": "otlp", "data_type": "traces"}
2023-11-08T18:38:37.186+0800	info	service@v0.88.0/service.go:143	Starting otelcol-dev...	{"Version": "1.0.0", "NumCPU": 10}

<OMITTED>

2023-11-08T18:38:37.189+0800	info	service@v0.88.0/service.go:169	Everything is ready. Begin running and processing data.
2023-11-08T18:38:37.189+0800	info	zapgrpc/zapgrpc.go:178	[core] [Server #3 ListenSocket #4] ListenSocket created	{"grpc_log": true}
2023-11-08T18:38:37.195+0800	info	zapgrpc/zapgrpc.go:178	[core] [Channel #1 SubChannel #2] Subchannel Connectivity change to READY	{"grpc_log": true}
2023-11-08T18:38:37.195+0800	info	zapgrpc/zapgrpc.go:178	[core] [pick-first-lb 0x140005efdd0] Received SubConn state update: 0x140005eff80, {ConnectivityState:READY ConnectionError:<nil>}	{"grpc_log": true}
2023-11-08T18:38:37.195+0800	info	zapgrpc/zapgrpc.go:178	[core] [Channel #1] Channel Connectivity change to READY	{"grpc_log": true}
```

모든 것이 잘 진행됐다면, 컬렉터 인스턴스가 정상적으로 실행 중이어야 한다.

설정을 추가로 검증하려면
[telemetrygen](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen)을
사용할 수 있다. 예를 들어 다른 콘솔을 열고 다음 명령을 실행한다.

```sh
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest

telemetrygen traces --otlp-insecure --traces 1
```

콘솔에서 상세한 로그를 확인할 수 있고, <http://localhost:16686/> URL을 통해
Jaeger UI에서도 트레이스를 확인할 수 있어야 한다.

컬렉터 콘솔에서 <kbd>Ctrl + C</kbd>를 눌러 컬렉터 인스턴스를 중지한다.

## Go 모듈 설정하기 {#setting-up-go-module}

모든 컬렉터 구성 요소는 Go 모듈로 생성해야 한다. 리시버 프로젝트를 담을
`tailtracer` 폴더를 만들고 이를 Go 모듈로 초기화한다.

```sh
mkdir tailtracer
cd tailtracer
go mod init github.com/open-telemetry/opentelemetry-tutorials/trace-receiver/tailtracer
```

> [!NOTE]
>
> 위의 모듈 경로는 예시 경로이며, 원하는 프라이빗 또는 퍼블릭 경로로 대체할 수
> 있다.
> [초기 trace-receiver 코드](https://github.com/rquedas/otel4devs/tree/main/collector/receiver/trace-receiver)를
> 참고한다.

`otelcol-dev`와 `tailtracer`, 그리고 향후 추가될 수 있는 여러 구성 요소를
관리해야 하므로 Go [Workspaces](https://go.dev/doc/tutorial/workspaces)를
활성화하는 것이 좋다.

```sh
cd ..
go work init
go work use otelcol-dev
go work use tailtracer
```

## 리시버 설정 설계 및 검증하기 {#designing-and-validating-receiver-settings}

리시버는 컬렉터 구성 파일을 통해 설정할 수 있는 구성 가능한 설정을 가질 수 있다.

`tailtracer` 리시버는 다음 설정을 갖는다.

- `interval`: 텔레메트리 풀 연산 사이의 시간 간격(분 단위)을 나타내는
  문자열이다.
- `number_of_traces`: 각 간격마다 생성되는 모의 트레이스의 개수다.

`tailtracer` 리시버 설정은 다음과 같은 모습이다.

```yaml
receivers:
  tailtracer: # this line represents the ID of your receiver
    interval: 1m
    number_of_traces: 1
```

리시버 설정을 지원하는 모든 코드를 작성할 `tailtracer` 폴더 아래에
`config.go`라는 이름의 파일을 생성한다.

```sh
touch tailtracer/config.go
```

리시버의 구성 관련 부분을 구현하려면 `Config` struct를 만들어야 한다. 다음
코드를 `config.go` 파일에 추가한다.

```go
package tailtracer

type Config struct{

}
```

리시버가 자신의 설정에 접근할 수 있게 하려면, `Config` struct는 리시버의 각
설정에 대응하는 필드를 가져야 한다.

위의 요구 사항을 구현하고 나면 `config.go` 파일은 다음과 같은 모습이 된다.

> tailtracer/config.go

```go
package tailtracer

// Config represents the receiver config settings in the Collector config.yaml
type Config struct {
   Interval    string `mapstructure:"interval"`
   NumberOfTraces int `mapstructure:"number_of_traces"`
}
```

> [!NOTE] 작업 확인
>
> - config.yaml에서 값에 제대로 접근할 수 있도록 `Interval` 필드와
>   `NumberOfTraces` 필드를 추가했다.

이제 설정에 접근할 수 있게 됐으니, 선택적
[ConfigValidator](https://github.com/open-telemetry/opentelemetry-collector/blob/677b87e3ab5c615bc3f93b8f99bb1fa5be951751/component/config.go#L28)
인터페이스에 따라 `Validate` 메서드를 구현해서 해당 값들에 필요한 검증을 제공할
수 있다.

이 경우 `interval` 값은 선택 사항이다(기본값을 생성하는 방법은 나중에 살펴본다).
하지만 값이 정의된다면 최소 1분(1m) 이상이어야 하고, `number_of_traces`는 필수
값이다. `Validate` 메서드를 구현하고 나면 config.go는 다음과 같은 모습이 된다.

> tailtracer/config.go

```go
package tailtracer

import (
	"fmt"
	"time"
)

// Config represents the receiver config settings in the Collector config.yaml
type Config struct {
	Interval       string `mapstructure:"interval"`
	NumberOfTraces int    `mapstructure:"number_of_traces"`
}

// Validate checks if the receiver configuration is valid
func (cfg *Config) Validate() error {
	interval, _ := time.ParseDuration(cfg.Interval)
	if interval.Minutes() < 1 {
		return fmt.Errorf("when defined, the interval has to be set to at least 1 minute (1m)")
	}

	if cfg.NumberOfTraces < 1 {
		return fmt.Errorf("number_of_traces must be greater or equal to 1")
	}
	return nil
}
```

> [!NOTE] 작업 확인
>
> - 에러 메시지를 제대로 포맷해서 출력할 수 있도록 `fmt` 패키지를 임포트했다.
> - `interval` 설정 값이 최소 1분(1m) 이상인지, `number_of_traces` 설정 값이 1
>   이상인지 확인하도록 `Config` struct에 `Validate` 메서드를 추가했다. 그렇지
>   않으면 컬렉터가 시작 과정에서 에러를 생성하고 그에 맞는 메시지를 표시한다.

구성 요소의 구성 관련 부분에 관여하는 struct와 인터페이스를 좀 더 자세히
살펴보고 싶다면, 컬렉터 GitHub 프로젝트 안의
[component/config.go](<https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/component/config.go>)
파일을 참고한다.

## receiver.Factory 인터페이스 구현하기 {#implementing-the-receiverfactory-interface}

`tailtracer` 리시버는 `receiver.Factory` 구현체를 제공해야 한다.
`receiver.Factory` 인터페이스는 컬렉터 프로젝트 내
[receiver/receiver.go](<https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/receiver/receiver.go#L58>)
파일에 정의되어 있지만, 이를 구현하는 올바른 방법은
`go.opentelemetry.io/collector/receiver` 패키지에서 제공하는 함수를 사용하는
것이다.

`factory.go`라는 이름의 파일을 생성한다.

```sh
touch tailtracer/factory.go
```

이제 관례를 따라 `tailtracer` 팩토리를 인스턴스화하는 역할을 담당할
`NewFactory()`라는 함수를 추가한다. 다음 코드를 `factory.go` 파일에 추가한다.

```go
package tailtracer

import (
	"go.opentelemetry.io/collector/receiver"
)

// NewFactory creates a factory for tailtracer receiver.
func NewFactory() receiver.Factory {
	return nil
}
```

`tailtracer` 리시버 팩토리를 인스턴스화하려면 `receiver` 패키지의 다음 함수를
사용한다.

```go
func NewFactory(cfgType component.Type, createDefaultConfig component.CreateDefaultConfigFunc, options ...FactoryOption) Factory
```

`receiver.NewFactory()`는 `receiver.Factory`를 인스턴스화해서 반환하며, 다음
매개변수를 필요로 한다.

- `component.Type`: 모든 컬렉터 구성 요소를 통틀어 리시버를 식별하는 고유한
  문자열 식별자.

- `component.CreateDefaultConfigFunc`: 리시버의 `component.Config` 인스턴스를
  반환하는 함수에 대한 참조.

- `...FactoryOption`: 리시버가 처리할 수 있는 시그널의 유형을 결정하는
  `receiver.FactoryOption`의 슬라이스.

이제 `receiver.NewFactory()`가 요구하는 모든 매개변수를 지원하는 코드를
구현해보자.

## 기본 설정 식별 및 제공하기 {#identifying-and-providing-default-settings}

앞서 `tailtracer` 리시버의 `interval` 설정이 선택 사항이 될 것이라고 언급했다.
이를 기본 설정의 일부로 사용할 수 있도록 기본값을 제공해야 한다.

다음 코드를 `factory.go` 파일에 추가한다.

```go
var (
	typeStr         = component.MustNewType("tailtracer")
)

const (
	defaultInterval = 1 * time.Minute
)
```

기본 설정과 관련해서는, `tailtracer` 리시버의 기본 구성을 담은
`component.Config`를 반환하는 함수만 추가하면 된다.

이를 위해 다음 코드를 `factory.go` 파일에 추가한다.

```go
func createDefaultConfig() component.Config {
	return &Config{
		Interval: string(defaultInterval),
	}
}
```

이 두 가지 변경 후에는 몇 가지 임포트가 빠져 있다는 것을 알게 될 것이다.
`factory.go` 파일은 올바른 임포트와 함께 다음과 같은 모습이어야 한다.

> tailtracer/factory.go

```go
package tailtracer

import (
	"time"

	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/receiver"
)

var (
	typeStr         = component.MustNewType("tailtracer")
)

const (
	defaultInterval = 1 * time.Minute
)

func createDefaultConfig() component.Config {
	return &Config{
		Interval: string(defaultInterval),
	}
}

// NewFactory creates a factory for tailtracer receiver.
func NewFactory() receiver.Factory {
	return nil
}
```

> [!NOTE] 작업 확인
>
> - `defaultInterval`의 `time.Duration` 타입을 지원하기 위해 `time` 패키지를
>   임포트했다.
> - `component.Config`가 선언되어 있는 `go.opentelemetry.io/collector/component`
>   패키지를 임포트했다.
> - `receiver.Factory`가 선언되어 있는 `go.opentelemetry.io/collector/receiver`
>   패키지를 임포트했다.
> - 리시버의 `Interval` 설정에 대한 기본값을 나타내는 `defaultInterval`이라는
>   이름의 `time.Duration` 상수를 추가했다. 기본값은 1분으로 설정할 것이므로
>   값으로 `1 * time.Minute`을 할당했다.
> - `component.Config` 구현체를 반환하는 역할을 하는 `createDefaultConfig`라는
>   이름의 함수를 추가했다. 이 경우에는 `tailtracer.Config` struct의 인스턴스가
>   된다.
> - `tailtracer.Config.Interval` 필드는 `defaultInterval` 상수로 초기화됐다.

## 리시버의 기능 지정하기 {#specifying-the-receivers-capabilities}

리시버 구성 요소는 트레이스, 메트릭, 로그를 처리할 수 있다. 리시버가 제공할
기능을 지정하는 것은 리시버의 팩토리가 담당한다.

이 튜토리얼의 주제가 트레이싱이므로, `tailtracer` 리시버가 트레이스만 처리할 수
있도록 활성화한다. `receiver` 패키지는 팩토리가 트레이스 처리 기능을 설명하는 데
도움이 되는 다음 함수와 타입을 제공한다.

```go
func WithTraces(createTracesReceiver CreateTracesFunc, sl component.StabilityLevel) FactoryOption
```

`receiver.WithTraces()`는 `receiver.FactoryOption`을 인스턴스화해서 반환하며,
다음 매개변수를 필요로 한다.

- `createTracesReceiver`: `receiver.CreateTracesFunc` 타입과 일치하는 함수에
  대한 참조다. `receiver.CreateTracesFunc` 타입은 `receiver.Traces` 인스턴스를
  인스턴스화해서 반환하는 역할을 하는 함수에 대한 포인터이며, 다음 매개변수를
  필요로 한다.
  - `context.Context`: 컬렉터의 `context.Context`에 대한 참조로, 트레이스
    리시버가 자신의 실행 컨텍스트를 제대로 관리할 수 있게 한다.
  - `receiver.Settings`: 리시버가 생성되는 컬렉터 설정 일부에 대한 참조다.
  - `component.Config`: 컬렉터가 팩토리에 전달하는 리시버 구성 설정에 대한
    참조로, 컬렉터 구성에서 자신의 설정을 제대로 읽을 수 있게 한다.
  - `consumer.Traces`: 파이프라인의 다음 `consumer.Traces`에 대한 참조로, 수신된
    트레이스가 전달되는 곳이다. 이는 프로세서 또는 익스포터 중 하나다.

`receiver.CreateTracesFunc` 함수 포인터를 제대로 구현하기 위한 부트스트랩 코드를
추가하는 것부터 시작한다. 다음 코드를 `factory.go` 파일에 추가한다.

```go
func createTracesReceiver(_ context.Context, params receiver.Settings, baseCfg component.Config, consumer consumer.Traces) (receiver.Traces, error) {
	return nil, nil
}
```

이제 `receiver.NewFactory` 함수를 사용해 리시버 팩토리를 성공적으로
인스턴스화하는 데 필요한 모든 구성 요소를 갖췄다. `factory.go` 파일의
`NewFactory()` 함수를 다음과 같이 업데이트한다.

```go
// NewFactory creates a factory for tailtracer receiver.
func NewFactory() receiver.Factory {
	return receiver.NewFactory(
		typeStr,
		createDefaultConfig,
		receiver.WithTraces(createTracesReceiver, component.StabilityLevelAlpha))
}
```

이 변경 후에는 몇 가지 임포트가 빠져 있다는 것을 알게 될 것이다. `factory.go`
파일은 올바른 임포트와 함께 다음과 같은 모습이어야 한다.

> tailtracer/factory.go

```go
package tailtracer

import (
	"context"
	"time"

	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/consumer"
	"go.opentelemetry.io/collector/receiver"
)

var (
	typeStr         = component.MustNewType("tailtracer")
)

const (
	defaultInterval = 1 * time.Minute
)

func createDefaultConfig() component.Config {
	return &Config{
		Interval: string(defaultInterval),
	}
}

func createTracesReceiver(_ context.Context, params receiver.Settings, baseCfg component.Config, consumer consumer.Traces) (receiver.Traces, error) {
	return nil, nil
}

// NewFactory creates a factory for tailtracer receiver.
func NewFactory() receiver.Factory {
	return receiver.NewFactory(
		typeStr,
		createDefaultConfig,
		receiver.WithTraces(createTracesReceiver, component.StabilityLevelAlpha))
}
```

> [!NOTE] 작업 확인
>
> - `createTracesReceiver` 함수에서 참조하는 `context.Context` 타입을 지원하기
>   위해 `context` 패키지를 임포트했다.
> - `createTracesReceiver` 함수에서 참조하는 `consumer.Traces` 타입을 지원하기
>   위해 `go.opentelemetry.io/collector/consumer` 패키지를 임포트했다.
> - `NewFactory()` 함수가 필요한 매개변수와 함께 `receiver.NewFactory()` 호출로
>   생성된 `receiver.Factory`를 반환하도록 업데이트했다. 생성된 리시버 팩토리는
>   `receiver.WithTraces(createTracesReceiver, component.StabilityLevelAlpha)`
>   호출을 통해 트레이스를 처리할 수 있게 된다.

## 리시버 구성 요소 구현하기 {#implementing-the-receiver-component}

모든 리시버 API는 현재 컬렉터 프로젝트의
[receiver/receiver.go](<https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/receiver/receiver.go>)
파일에 선언되어 있다. 파일을 열어서 잠시 시간을 들여 모든 인터페이스를 살펴본다.

`receiver.Traces`(그리고 그 형제 격인 `receiver.Metrics`와 `receiver.Logs`)가
현재 시점에서는 `component.Component`로부터 "상속받은" 메서드 외에 특별한
메서드를 설명하고 있지 않다는 점에 주목한다.

이상하게 느껴질 수 있지만, 컬렉터 API는 확장 가능하도록 설계되었다는 점을
기억한다. 구성 요소와 그 시그널은 저마다 다른 방식으로 발전할 수 있으므로, 이런
인터페이스들의 역할은 그것을 뒷받침하기 위해 존재하는 것이다.

`receiver.Traces`를 만들려면 `component.Component` 인터페이스가 설명하는 다음
메서드를 구현해야 한다.

```go
Start(ctx context.Context, host Host) error
Shutdown(ctx context.Context) error
```

두 메서드 모두 컬렉터가 생명 주기의 일부로서 구성 요소와 통신하는 데 사용하는
이벤트 핸들러 역할을 한다.

`Start()` 메서드는 컬렉터가 구성 요소에게 처리를 시작하라고 알리는 신호를
나타낸다. 이 이벤트의 일부로 컬렉터는 다음 정보를 전달한다.

- `context.Context`: 대부분의 경우 리시버는 장기 실행 연산을 처리하므로, 이
  컨텍스트는 무시하고 context.Background()로부터 새 컨텍스트를 생성하는 것이
  권장된다.
- `Host`: 컬렉터 호스트가 가동되어 실행 중일 때 리시버가 이와 통신할 수 있게
  하기 위한 것이다.

`Shutdown()` 메서드는 컬렉터가 구성 요소에게 서비스가 종료되고 있다는 신호를
나타내며, 이에 따라 구성 요소는 처리를 중단하고 필요한 모든 정리 작업을 수행해야
한다.

- `context.Context`: 종료 연산의 일부로 컬렉터가 전달하는 컨텍스트다.

`tailtracer` 폴더에 `trace-receiver.go`라는 이름의 새 파일을 생성하는 것으로
구현을 시작한다.

```sh
touch tailtracer/trace-receiver.go
```

그런 다음 다음과 같이 `tailtracerReceiver`라는 타입에 대한 선언을 추가한다.

```go
type tailtracerReceiver struct{

}
```

이제 `tailtracerReceiver` 타입이 생겼으니, 리시버 타입이 `receiver.Traces`
인터페이스를 준수할 수 있도록 `Start()` 메서드와 `Shutdown()` 메서드를 구현할 수
있다.

> tailtracer/trace-receiver.go

```go
package tailtracer

import (
	"context"
	"go.opentelemetry.io/collector/component"
)

type tailtracerReceiver struct {
}

func (tailtracerRcvr *tailtracerReceiver) Start(ctx context.Context, host component.Host) error {
	return nil
}

func (tailtracerRcvr *tailtracerReceiver) Shutdown(ctx context.Context) error {
	return nil
}
```

> [!NOTE] 작업 확인
>
> - `Context` 타입과 함수가 선언되어 있는 `context` 패키지를 임포트했다.
> - `Host` 타입이 선언되어 있는 `go.opentelemetry.io/collector/component`
>   패키지를 임포트했다.
> - `receiver.Traces` 인터페이스를 준수하기 위해
>   `Start(ctx context.Context, host component.Host)` 메서드의 부트스트랩 구현을
>   추가했다.
> - `receiver.Traces` 인터페이스를 준수하기 위해 `Shutdown(ctx context.Context)`
>   메서드의 부트스트랩 구현을 추가했다.

`Start()` 메서드는 리시버가 처리 연산의 일부로 사용할 수 있도록 보관해야 할 수도
있는 2개의 참조(`context.Context`와 `component.Host`)를 전달한다.

`context.Context` 참조는 리시버의 처리 연산을 지원하는 새 컨텍스트를 생성하는 데
사용해야 한다. 컨텍스트 취소를 처리할 최선의 방법을 결정해서 `Shutdown()`
메서드에서 구성 요소의 종료 과정 일부로 이를 제대로 마무리할 수 있어야 한다.

`component.Host`는 리시버의 전체 생명 주기 동안 유용할 수 있으므로 해당 참조를
`tailtracerReceiver` 타입에 보관한다.

위에서 제안한 참조를 보관하기 위한 필드를 포함하고 나면, `tailtracerReceiver`
타입 선언은 다음과 같은 모습이 된다.

```go
type tailtracerReceiver struct {
	host   component.Host
	cancel context.CancelFunc
}
```

이제 리시버가 자신의 처리 컨텍스트를 제대로 초기화하고, 취소 함수를 `cancel`
필드에 보관하고, `host` 필드 값을 초기화할 수 있도록 `Start()` 메서드를
업데이트해야 한다. 또한 `cancel` 함수를 호출해서 컨텍스트를 마무리하도록
`Stop()` 메서드도 업데이트한다.

변경 후 `trace-receiver.go` 파일은 다음과 같은 모습이 된다.

> tailtracer/trace-receiver.go

```go
package tailtracer

import (
	"context"
	"go.opentelemetry.io/collector/component"
)

type tailtracerReceiver struct {
	host   component.Host
	cancel context.CancelFunc
}

func (tailtracerRcvr *tailtracerReceiver) Start(ctx context.Context, host component.Host) error {
	tailtracerRcvr.host = host
	ctx = context.Background()
	ctx, tailtracerRcvr.cancel = context.WithCancel(ctx)

	return nil
}

func (tailtracerRcvr *tailtracerReceiver) Shutdown(ctx context.Context) error {
	if tailtracerRcvr.cancel != nil {
		tailtracerRcvr.cancel()
	}
	return nil
}
```

> [!NOTE] 작업 확인
>
> 컬렉터가 전달하는 `component.Host` 참조로 `host` 필드를 초기화하는 코드를
> 추가해서 `Start()` 메서드를 업데이트했다.
>
> - `context.Background()`로 생성된 새 컨텍스트를 기반으로 한 취소로 `cancel`
>   함수 필드를 설정했다(컬렉터 API 문서의 제안에 따른 것이다).
> - `cancel()` 컨텍스트 취소 함수 호출을 추가해서 `Shutdown()` 메서드를
>   업데이트했다.

## 리시버의 팩토리가 전달하는 정보 보관하기 {#keeping-information-passed-by-the-receivers-factory}

이제 `receiver.Traces` 인터페이스 메서드를 구현했으니, `tailtracer` 리시버 구성
요소는 팩토리에 의해 인스턴스화되어 반환될 준비가 됐다.

`tailtracer/factory.go` 파일을 열어 `createTracesReceiver()` 함수로 이동한다.
팩토리는 리시버가 제대로 동작하기 위해 필요한 참조를 `createTracesReceiver()`
함수 매개변수의 일부로 전달한다는 점에 주목한다. 여기에는 리시버의 구성
설정(`component.Config`), 생성된 트레이스를 소비할 파이프라인의 다음 `Consumer`
(`consumer.Traces`), 그리고 컬렉터 로거가 포함된다. 이는 `tailtracer` 리시버가
의미 있는 이벤트를 로거에 추가할 수 있도록 하기 위함이다(`receiver.Settings`).

이 모든 정보는 팩토리에 의해 인스턴스화되는 순간에만 리시버에게 제공되므로,
`tailtracerReceiver` 타입은 이 정보를 보관하고 생명 주기의 다른 단계에서 사용할
필드가 필요하다.

업데이트된 `tailtracerReceiver` 타입 선언과 함께 `trace-receiver.go` 파일은
다음과 같은 모습이 된다.

> tailtracer/trace-receiver.go

```go
package tailtracer

import (
	"context"
	"time"
	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/consumer"
	"go.uber.org/zap"
)

type tailtracerReceiver struct {
	host         component.Host
	cancel       context.CancelFunc
	logger       *zap.Logger
	nextConsumer consumer.Traces
	config       *Config
}

func (tailtracerRcvr *tailtracerReceiver) Start(ctx context.Context, host component.Host) error {
	tailtracerRcvr.host = host
	ctx = context.Background()
	ctx, tailtracerRcvr.cancel = context.WithCancel(ctx)

	interval, _ := time.ParseDuration(tailtracerRcvr.config.Interval)
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		for {
			select {
				case <-ticker.C:
					tailtracerRcvr.logger.Info("I should start processing traces now!")
				case <-ctx.Done():
					return
			}
		}
	}()

	return nil
}

func (tailtracerRcvr *tailtracerReceiver) Shutdown(ctx context.Context) error {
	if tailtracerRcvr.cancel != nil {
		tailtracerRcvr.cancel()
	}
	return nil
}
```

> [!NOTE] 작업 확인
>
> - 파이프라인의 컨슈머 타입과 인터페이스가 선언되어 있는
>   `go.opentelemetry.io/collector/consumer`를 임포트했다.
> - 컬렉터가 디버깅 기능에 사용하는 `go.uber.org/zap` 패키지를 임포트했다.
> - 리시버 내에서 컬렉터 로거 참조에 접근할 수 있도록 `logger`라는 이름의
>   `zap.Logger` 필드를 추가했다.
> - `tailtracer` 리시버가 생성한 트레이스를 컬렉터 파이프라인에 선언된 다음
>   컨슈머로 전달할 수 있도록 `nextConsumer`라는 이름의 `consumer.Traces` 필드를
>   추가했다.
> - 컬렉터 구성에 정의된 리시버의 구성 설정에 접근할 수 있도록 `config`라는
>   이름의 `Config` 필드를 추가했다.
> - 컬렉터 구성에 정의된 `tailtracer` 리시버의 `interval` 설정 값을 기반으로
>   `time.Duration`으로 초기화되는 `interval`이라는 이름의 변수를 추가했다.
> - `ticker`가 `interval` 변수로 지정된 시간에 도달할 때마다 리시버가 트레이스를
>   생성할 수 있도록 `ticker` 메커니즘을 구현하는 `go func()`를 추가했다.
> - 리시버가 트레이스를 생성해야 할 때마다 정보 메시지를 생성하기 위해
>   `tailtracerRcvr.logger` 필드를 사용했다.

`tailtracerReceiver` 타입은 이제 인스턴스화될 준비가 됐으며, 팩토리가 전달하는
모든 의미 있는 정보를 보관하게 된다.

`tailtracer/factory.go` 파일을 열어 `createTracesReceiver()` 함수로 이동한다.

리시버는 파이프라인에서 구성 요소로 선언된 경우에만 인스턴스화되며, 팩토리는
파이프라인의 다음 컨슈머(프로세서 또는 익스포터)가 유효한지 확인할 책임이 있다.
그렇지 않다면 에러를 생성해야 한다.

`createTracesReceiver()` 함수에는 이 검증을 수행할 가드 절(guard clause)이
필요하다.

또한 `tailtracerReceiver` 인스턴스의 `config` 필드와 `logger` 필드를 제대로
초기화하기 위한 변수도 필요하다.

업데이트된 `createTracesReceiver()` 함수와 함께 `factory.go` 파일은 다음과 같은
모습이 된다.

> tailtracer/factory.go

```go
package tailtracer

import (
	"context"
	"time"

	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/consumer"
	"go.opentelemetry.io/collector/receiver"
)

var (
	typeStr         = component.MustNewType("tailtracer")
)

const (
	defaultInterval = 1 * time.Minute
)

func createDefaultConfig() component.Config {
	return &Config{
		Interval: string(defaultInterval),
	}
}

func createTracesReceiver(_ context.Context, params receiver.Settings, baseCfg component.Config, consumer consumer.Traces) (receiver.Traces, error) {

	logger := params.Logger
	tailtracerCfg := baseCfg.(*Config)

	traceRcvr := &tailtracerReceiver{
		logger:       logger,
		nextConsumer: consumer,
		config:       tailtracerCfg,
	}

	return traceRcvr, nil
}

// NewFactory creates a factory for tailtracer receiver.
func NewFactory() receiver.Factory {
	return receiver.NewFactory(
		typeStr,
		createDefaultConfig,
		receiver.WithTraces(createTracesReceiver, component.StabilityLevelAlpha))
}
```

> [!NOTE] 작업 확인
>
> - `logger`라는 이름의 변수를 추가하고 `receiver.Settings` 참조에서
>   `Logger`라는 이름의 필드로 제공되는 컬렉터 로거로 초기화했다.
> - `tailtracerCfg`라는 이름의 변수를 추가하고 `component.Config` 참조를
>   `tailtracer` 리시버의 `Config`로 캐스팅해서 초기화했다.
> - `traceRcvr`라는 이름의 변수를 추가하고, 변수에 저장된 팩토리 정보를 사용해
>   `tailtracerReceiver` 인스턴스로 초기화했다.
> - `traceRcvr` 인스턴스를 포함하도록 반환 문을 업데이트했다.

여기까지, 리시버의 기본 골격이 완전히 구현됐다.

## 리시버를 사용한 컬렉터 초기화 과정 업데이트하기 {#updating-the-collector-initialization-process-with-the-receiver}

리시버가 컬렉터 파이프라인에 참여하려면, 모든 컬렉터 구성 요소가 등록되고
인스턴스화되는 생성된 `otelcol-dev/components.go` 파일을 일부 업데이트해야 한다.

컬렉터가 초기화 과정의 일부로 이를 제대로 로드할 수 있도록 `tailtracer` 리시버
팩토리 인스턴스를 `factories` 맵에 추가해야 한다.

이를 지원하기 위해 변경한 후 `components.go` 파일은 다음과 같은 모습이 된다.

> otelcol-dev/components.go

```go
// Code generated by "go.opentelemetry.io/collector/cmd/builder". DO NOT EDIT.

package main

import (
	"go.opentelemetry.io/collector/exporter"
	"go.opentelemetry.io/collector/extension"
	"go.opentelemetry.io/collector/otelcol"
	"go.opentelemetry.io/collector/processor"
	"go.opentelemetry.io/collector/receiver"
	debugexporter "go.opentelemetry.io/collector/exporter/debugexporter"
	otlpexporter "go.opentelemetry.io/collector/exporter/otlpexporter"
	otlpreceiver "go.opentelemetry.io/collector/receiver/otlpreceiver"
	tailtracer "github.com/open-telemetry/opentelemetry-tutorials/trace-receiver/tailtracer" // newly added line
)

func components() (otelcol.Factories, error) {
	var err error
	factories := otelcol.Factories{}

	factories.Extensions, err = otelcol.MakeFactoryMap[extension.Factory](
	)
	if err != nil {
		return otelcol.Factories{}, err
	}

	factories.Receivers, err = otelcol.MakeFactoryMap[receiver.Factory](
		otlpreceiver.NewFactory(),
		tailtracer.NewFactory(), // newly added line
	)
	if err != nil {
		return otelcol.Factories{}, err
	}

	factories.Exporters, err = otelcol.MakeFactoryMap[exporter.Factory](
		debugexporter.NewFactory(),
		otlpexporter.NewFactory(),
	)
	if err != nil {
		return otelcol.Factories{}, err
	}

	factories.Processors, err = otelcol.MakeFactoryMap[processor.Factory](
	)
	if err != nil {
		return otelcol.Factories{}, err
	}

	return factories, nil
}
```

> [!NOTE] 작업 확인
>
> - 리시버 타입과 함수가 있는
>   `github.com/open-telemetry/opentelemetry-tutorials/trace-receiver/tailtracer`
>   모듈을 임포트했다.
> - `tailtracer` 리시버 팩토리가 `factories` 맵에 제대로 추가되도록
>   `otelcol.MakeFactoryMap()` 호출의 매개변수로 `tailtracer.NewFactory()`
>   호출을 추가했다.

## 리시버 실행 및 디버깅하기 {#running-and-debugging-the-receiver}

컬렉터 `config.yaml`이 파이프라인에서 사용되는 리시버 중 하나로 `tailtracer`
리시버가 구성된 상태로 제대로 업데이트되었는지 확인한다.

> config.yaml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
  tailtracer: # this line represents the ID of your receiver
    interval: 1m
    number_of_traces: 1

exporters:
  debug:
    verbosity: detailed
  otlp/jaeger:
    endpoint: localhost:14317
    tls:
      insecure: true
    sending_queue:
      batch:

service:
  pipelines:
    traces:
      receivers: [otlp, tailtracer]
      exporters: [otlp/jaeger, debug]
  telemetry:
    logs:
      level: debug
```

`otelcol-dev/components.go` 파일에 코드 변경이 있었으므로, 이전에 생성한
`./otelcol-dev/otelcol-dev` 바이너리 파일 대신 `go run` 명령을 사용해 업데이트된
컬렉터를 시작한다.

```sh
go run ./otelcol-dev --config config.yaml
```

출력은 다음과 같은 모습이어야 한다.

```log
2023-11-08T21:38:36.621+0800	info	service@v0.88.0/telemetry.go:84	Setting up own telemetry...
2023-11-08T21:38:36.621+0800	info	service@v0.88.0/telemetry.go:201	Serving Prometheus metrics	{"address": ":8888", "level": "Basic"}
2023-11-08T21:38:36.621+0800	info	exporter@v0.88.0/exporter.go:275	Development component. May change in the future.	{"kind": "exporter", "data_type": "traces", "name": "debug"}
2023-11-08T21:38:36.621+0800	debug	exporter@v0.88.0/exporter.go:273	Stable component.	{"kind": "exporter", "data_type": "traces", "name": "otlp/jaeger"}
2023-11-08T21:38:36.621+0800	debug	receiver@v0.88.0/receiver.go:294	Stable component.	{"kind": "receiver", "name": "otlp", "data_type": "traces"}
2023-11-08T21:38:36.621+0800	debug	receiver@v0.88.0/receiver.go:294	Alpha component. May change in the future.	{"kind": "receiver", "name": "tailtracer", "data_type": "traces"}
2023-11-08T21:38:36.622+0800	info	service@v0.88.0/service.go:143	Starting otelcol-dev...	{"Version": "1.0.0", "NumCPU": 10}
2023-11-08T21:38:36.622+0800	info	extensions/extensions.go:33	Starting extensions...

<OMITTED>

2023-11-08T21:38:36.636+0800	info	zapgrpc/zapgrpc.go:178	[core] [Channel #1] Channel Connectivity change to READY	{"grpc_log": true}
2023-11-08T21:39:36.626+0800	info	tailtracer/trace-receiver.go:33	I should start processing traces now!	{"kind": "receiver", "name": "tailtracer", "data_type": "traces"}
2023-11-08T21:40:36.626+0800	info	tailtracer/trace-receiver.go:33	I should start processing traces now!	{"kind": "receiver", "name": "tailtracer", "data_type": "traces"}
...
```

로그에서 볼 수 있듯이, `tailtracer`가 성공적으로 초기화됐다. 매 분마다
`tailtracer/trace-receiver.go`의 더미 ticker에 의해 트리거되는
`I should start processing traces now!`라는 메시지가 표시된다.

> [!TIP]
>
> 프로세스를 중지하려면 컬렉터 터미널에서 <kbd>Ctrl + C</kbd>를 누른다.

또한 평소 Go 프로젝트를 디버깅하는 것처럼 원하는 IDE를 사용해 리시버를 디버깅할
수 있다. 참고용으로 [Visual Studio Code](https://code.visualstudio.com/)를 위한
간단한 `launch.json` 파일은 다음과 같다.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch otelcol-dev",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/otelcol-dev",
      "args": ["--config", "${workspaceFolder}/config.yaml"]
    }
  ]
}
```

중요한 이정표로서, 현재 폴더 구조가 어떻게 되어 있는지 살펴보자.

```console
.
├── builder-config.yaml
├── config.yaml
├── go.work
├── go.work.sum
├── ocb
├── otelcol-dev
│   ├── components.go
│   ├── components_test.go
│   ├── go.mod
│   ├── go.sum
│   ├── main.go
│   ├── main_others.go
│   ├── main_windows.go
│   └── otelcol-dev
└── tailtracer
    ├── config.go
    ├── factory.go
    ├── go.mod
    └── trace-receiver.go
```

다음 절에서는 `tailtracer` 리시버가 마침내 의미 있는 트레이스를 생성할 수 있도록
오픈텔레메트리 트레이스 데이터 모델에 대해 더 알아본다!

## 컬렉터 트레이스 데이터 모델 {#the-collector-trace-data-model}

SDK를 사용해 애플리케이션을 계측하고 Jaeger와 같은 분산 트레이싱 백엔드에서
트레이스를 관찰하고 평가해봄으로써 오픈텔레메트리 트레이스에 이미 익숙할 수도
있다.

Jaeger에서 트레이스는 다음과 같은 모습이다.

![Jaeger trace](/img/docs/tutorials/Jaeger.jpeg)

이는 Jaeger 트레이스이지만, 컬렉터의 트레이스 파이프라인에 의해 생성된 것이다.
이를 통해 OTel 트레이스 데이터 모델에 대한 몇 가지 사항을 이해할 수 있다.

- 트레이스는 의존 관계를 나타내기 위해 계층 구조로 구성된 하나 이상의 스팬으로
  이루어진다.
- 스팬은 서비스 내부 또는 서비스 간의 연산을 나타낼 수 있다.

트레이스 리시버에서 트레이스를 생성하는 것은 SDK로 하는 방식과 조금 다르므로,
먼저 상위 수준의 개념부터 살펴보자.

### 리소스 다루기 {#working-with-resources}

OTel의 세계에서 모든 텔레메트리는 `Resource`에 의해 생성된다.
[OTel 명세](/docs/specs/otel/resource/sdk)에 따른 정의는 다음과 같다.

> `Resource`는 속성(Attribute)으로서 텔레메트리를 생성하는 엔티티를 나타내는
> 불변(immutable) 표현이다. 예를 들어, 쿠버네티스(Kubernetes)의 컨테이너에서
> 실행 중인, 텔레메트리를 생성하는 프로세스는 파드(Pod) 이름을 가지고,
> 네임스페이스 안에서 실행되며, 고유한 이름을 가진 디플로이먼트(Deployment)의
> 일부일 수 있다. 이 세 가지 속성 모두 `Resource`에 포함될 수 있다.

트레이스는 가장 흔하게 서비스 요청(Jaeger 모델에서 설명하는 Services 엔티티)을
나타내는 데 사용되며, 이는 일반적으로 컴퓨팅 단위에서 실행되는 프로세스로
구현된다. 하지만 속성을 통해 `Resource`를 설명하는 OTel의 API 방식은 ATM, IoT
센서 등 필요한 모든 엔티티를 나타낼 수 있을 만큼 유연하다.

따라서 트레이스가 존재하려면 `Resource`가 이를 시작해야 한다고 말할 수 있다.

이 튜토리얼에서는 서로 다른 2개의 주(예를 들어 일리노이와 캘리포니아)에 위치한
ATM이 잔액 조회, 입금, 출금 연산을 실행하기 위해 계정 백엔드 시스템에 접근하는
것을 보여주는 텔레메트리를 갖춘 시스템을 시뮬레이션한다. 이를 위해 ATM과 백엔드
시스템을 나타내는 `Resource` 타입을 생성하는 코드를 구현한다.

`tailtracer` 폴더 안에 `model.go`라는 이름의 파일을 생성한다.

```sh
touch tailtracer/model.go
```

이제 `model.go` 파일에 다음과 같이 `Atm` 타입과 `BackendSystem` 타입에 대한
정의를 추가한다.

> tailtracer/model.go

```go
package tailtracer

type Atm struct{
	ID           int64
	Version      string
	Name         string
	StateID      string
	SerialNumber string
	ISPNetwork   string
}

type BackendSystem struct{
	Version       string
	ProcessName   string
	OSType        string
	OSVersion     string
	CloudProvider string
	CloudRegion   string
	Endpoint      string
}
```

이 타입들은 관찰 대상 시스템에 나타나는 그대로의 엔티티를 나타내기 위한 것이다.
이들은 `Resource` 정의의 일부로 트레이스에 추가하면 상당히 의미 있는 정보를 담고
있다. 이 타입들의 인스턴스를 생성하는 헬퍼 함수를 추가한다.

추가된 헬퍼 함수와 함께 `model.go` 파일은 다음과 같은 모습이 된다.

> tailtracer/model.go

```go
package tailtracer

import (
	"math/rand"
)

type Atm struct{
	ID           int64
	Version      string
	Name         string
	StateID      string
	SerialNumber string
	ISPNetwork   string
}

type BackendSystem struct{
	Version       string
	ProcessName   string
	OSType        string
	OSVersion     string
	CloudProvider string
	CloudRegion   string
	Endpoint      string
}

func generateAtm() Atm{
	i := getRandomNumber(1, 2)
	var newAtm Atm

	switch i {
		case 1:
			newAtm = Atm{
				ID: 111,
				Name: "ATM-111-IL",
				SerialNumber: "atmxph-2022-111",
				Version: "v1.0",
				ISPNetwork: "comcast-chicago",
				StateID: "IL",

			}

		case 2:
			newAtm = Atm{
				ID: 222,
				Name: "ATM-222-CA",
				SerialNumber: "atmxph-2022-222",
				Version: "v1.0",
				ISPNetwork: "comcast-sanfrancisco",
				StateID: "CA",
			}
	}

	return newAtm
}

func generateBackendSystem() BackendSystem{
	i := getRandomNumber(1, 3)

	newBackend := BackendSystem{
		ProcessName: "accounts",
		Version: "v2.5",
		OSType: "lnx",
		OSVersion: "4.16.10-300.fc28.x86_64",
		CloudProvider: "amzn",
		CloudRegion: "us-east-2",
	}

	switch i {
		case 1:
		 	newBackend.Endpoint = "api/v2.5/balance"
		case 2:
		  	newBackend.Endpoint = "api/v2.5/deposit"
		case 3:
			newBackend.Endpoint = "api/v2.5/withdrawn"

	}

	return newBackend
}

func getRandomNumber(min int, max int) int {
	i := (rand.Intn(max - min + 1) + min)
	return i
}
```

> [!NOTE] 작업 확인
>
> - `generateRandomNumber` 함수 구현을 지원하기 위해 `math/rand` 패키지를
>   임포트했다.
> - `Atm` 타입을 인스턴스화하고 `StateID` 값으로 일리노이 또는 캘리포니아 중
>   하나를 무작위로 지정하며, 그에 대응하는 `ISPNetwork` 값도 함께 지정하는
>   `generateAtm` 함수를 추가했다.
> - `BackendSystem` 타입의 인스턴스를 생성하고 `Endpoint` 필드에 서비스
>   엔드포인트 값을 무작위로 지정하는 `generateBackendSystem` 함수를 추가했다.
> - 지정된 범위 내에서 난수를 생성하는 `generateRandomNumber` 함수를 추가했다.

이제 텔레메트리를 생성하는 엔티티를 나타내는 객체 인스턴스를 생성하는 함수를
갖췄으니, OTel 컬렉터 세계에서 이 엔티티들을 나타낼 준비가 됐다.

컬렉터 API는 `pdata` 패키지 아래에 중첩된 `ptrace`라는 이름의 패키지를 제공한다.
여기에는 컬렉터 파이프라인 구성 요소에서 트레이스를 다루는 데 필요한 모든 타입,
인터페이스, 헬퍼 함수가 포함되어 있다.

`tailtracer/model.go` 파일을 열어 `import` 절에
`go.opentelemetry.io/collector/pdata/ptrace`를 추가해서 `ptrace` 패키지의 기능에
접근할 수 있게 한다.

`Resource`를 정의하기 전에, 컬렉터 파이프라인을 통해 트레이스를 전파하는 역할을
하는 `ptrace.Traces`를 생성해야 한다. 이를 인스턴스화하려면 헬퍼 함수인
`ptrace.NewTraces()`를 사용할 수 있다. 또한 트레이스에 관여하는 텔레메트리
소스를 나타낼 데이터를 갖기 위해 `Atm` 타입과 `BackendSystem` 타입의 인스턴스도
생성해야 한다.

`tailtracer/model.go` 파일을 열어 다음 함수를 추가한다.

```go
func generateTraces(numberOfTraces int) ptrace.Traces{
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++{
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()
	}

	return traces
}
```

지금까지 트레이스가 스팬으로 어떻게 구성되는지 충분히 듣고 읽었을 것이다. SDK가
제공하는 함수와 타입을 사용해 이를 생성하는 계측 코드를 직접 작성해봤을 수도
있다. 하지만 컬렉터 API에서 트레이스를 생성하는 데 관여하는 다른 유형의 "스팬"이
있다는 사실은 모를 수도 있다.

먼저 `ptrace.ResourceSpans`라는 타입부터 살펴본다. 이는 리소스와, 트레이스에
참여하는 동안 그 리소스가 시작했거나 수신한 모든 연산을 나타낸다. 그 정의는
[/pdata/ptrace/generated_resourcespans.go](<https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/pdata/ptrace/generated_resourcespans.go>)
파일에서 찾을 수 있다.

`ptrace.Traces`에는 `ResourceSpans()`라는 메서드가 있으며, 이는
`ptrace.ResourceSpansSlice`라는 헬퍼 타입의 인스턴스를 반환한다.
`ptrace.ResourceSpansSlice` 타입에는 `ptrace.ResourceSpans` 배열을 다루는 데
도움이 되는 메서드가 있다. 이 배열은 트레이스가 나타내는 요청에 참여하는
`Resource` 엔티티의 수만큼 항목을 담게 된다.

`ptrace.ResourceSpansSlice`에는 배열에 새 `ptrace.ResourceSpan`을 추가하고 그
참조를 반환하는 `AppendEmpty()`라는 메서드가 있다.

`ptrace.ResourceSpan`의 인스턴스를 얻고 나면, `Resource()`라는 이름의 메서드를
사용해 해당 `ResourceSpan`과 연관된 `pcommon.Resource` 인스턴스를 반환받을 수
있다.

다음과 같이 `generateTrace()` 함수를 변경한다.

- `ResourceSpan`을 나타내는 `resourceSpan`이라는 이름의 변수를 추가한다.
- `ResourceSpan`과 연관된 `pcommon.Resource`를 나타내는 `atmResource`라는 이름의
  변수를 추가한다.
- 위에서 언급한 메서드를 사용해 각각의 변수를 초기화한다.

변경 사항을 구현하고 나면 함수는 다음과 같은 모습이 되어야 한다.

```go
func generateTraces(numberOfTraces int) ptrace.Traces{
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++{
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
	}

	return traces
}
```

> [!NOTE] 작업 확인
>
> - `traces.ResourceSpans().AppendEmpty()` 호출이 반환하는 `ResourceSpan` 참조로
>   초기화된 `resourceSpan` 변수를 추가했다.
> - `resourceSpan.Resource()` 호출이 반환하는 `pcommon.Resource` 참조로 초기화된
>   `atmResource` 변수를 추가했다.

### 속성으로 리소스 설명하기 {#describing-resources-through-attributes}

컬렉터 API는 `pdata` 패키지 아래에 중첩된 `pcommon`이라는 이름의 패키지를
제공한다. 여기에는 `Resource`를 설명하는 데 필요한 모든 타입과 헬퍼 함수가
포함되어 있다.

컬렉터의 맥락에서 `Resource`는 `pcommon.Map` 타입으로 표현되는 키/값 쌍 형식의
속성으로 설명된다.

`pcommon.Map` 타입의 정의와, 지원되는 형식을 사용해 속성 값을 생성하는 관련 헬퍼
함수는 컬렉터 GitHub 프로젝트의
[/pdata/pcommon/map.go](<https://github.com/open-telemetry/opentelemetry-collector/blob/v{{% param vers %}}/pdata/pcommon/map.go>)
파일에서 확인할 수 있다.

키/값 쌍은 `Resource` 데이터를 모델링하는 데 많은 유연성을 제공한다. OTel 명세는
그것이 나타내야 할 수도 있는 다양한 유형의 텔레메트리 생성 엔티티 전반에 걸쳐
충돌을 정리하고 최소화하는 데 도움이 되는 몇 가지 가이드라인을 마련해 두고 있다.

이러한 가이드라인은 [리소스 시맨틱 컨벤션](/docs/specs/semconv/resource/)으로
알려져 있으며, OTel 명세에 문서화되어 있다.

자신의 텔레메트리 생성 엔티티를 나타내기 위한 속성을 직접 만들 때는 명세가
제공하는 가이드라인을 따라야 한다.

> 속성은 그것이 설명하는 개념의 유형에 따라 논리적으로 그룹화된다. 같은 그룹의
> 속성들은 마침표로 끝나는 공통 접두사를 갖는다. 예를 들어, 쿠버네티스 속성을
> 설명하는 모든 속성은 `k8s.`로 시작한다.

먼저 `tailtracer/model.go` 파일을 열어 `import` 절에
`go.opentelemetry.io/collector/pdata/pcommon`을 추가해서 `pcommon` 패키지의
기능에 접근할 수 있게 한다.

이제 `Atm` 인스턴스에서 필드 값을 읽어 `pcommon.Resource` 인스턴스에 속성으로
("atm." 접두사로 그룹화해서) 기록하는 함수를 추가한다. 함수는 다음과 같은
모습이다.

```go
func fillResourceWithAtm(resource *pcommon.Resource, atm Atm){
   atmAttrs := resource.Attributes()
   atmAttrs.PutInt("atm.id", atm.ID)
   atmAttrs.PutStr("atm.stateid", atm.StateID)
   atmAttrs.PutStr("atm.ispnetwork", atm.ISPNetwork)
   atmAttrs.PutStr("atm.serialnumber", atm.SerialNumber)
}
```

> [!NOTE] 작업 확인
>
> - `atmAttrs`라는 이름의 변수를 선언하고 `resource.Attributes()` 호출이
>   반환하는 `pcommon.Map` 참조로 초기화했다.
> - `pcommon.Map`의 `PutInt()` 메서드와 `PutStr()` 메서드를 사용해 대응하는
>   `Atm` 필드 타입에 맞는 int 속성과 string 속성을 추가했다. 이 속성들은 `Atm`
>   엔티티에만 특정적으로 해당되므로 모두 `atm.` 접두사로 그룹화되어 있다는 점에
>   주목한다.

리소스 시맨틱 컨벤션에는
[컴퓨팅 단위](/docs/specs/semconv/resource/#compute-unit),
[환경](/docs/specs/semconv/resource/#compute-unit) 등 여러 도메인에 걸쳐
공통적으로 적용 가능한 텔레메트리 생성 엔티티를 나타내는 규정된 속성 이름과 잘
알려진 값도 있다.

`BackendSystem` 엔티티의 경우 [운영체제](/docs/specs/semconv/resource/os/)와
[클라우드](/docs/specs/semconv/resource/cloud/)에 관련된 정보를 나타내는 필드를
갖고 있다. 이 정보를 해당 `Resource`에 나타내기 위해 리소스 시맨틱 컨벤션이
지정한 속성 이름과 값을 사용한다.

리소스 시맨틱 컨벤션의 키와 잘 알려진 값은 오픈텔레메트리 시맨틱 컨벤션 패키지인
[`go.opentelemetry.io/otel/semconv/v1.43.0`](https://pkg.go.dev/go.opentelemetry.io/otel/semconv/v1.43.0)에
정의되어 있다.

이제 `BackendSystem` 인스턴스에서 필드 값을 읽어 `pcommon.Resource` 인스턴스에
속성으로 기록하는 함수를 만들어보자. `tailtracer/model.go` 파일을 열어 다음
함수를 추가한다.

```go
func fillResourceWithBackendSystem(resource *pcommon.Resource, backend BackendSystem){
	backendAttrs := resource.Attributes()
	var osType, cloudProvider string

	switch {
		case backend.CloudProvider == "amzn":
			cloudProvider = semconv.CloudProviderAWS.Value.AsString()
		case backend.CloudProvider == "mcrsft":
			cloudProvider = semconv.CloudProviderAzure.Value.AsString()
		case backend.CloudProvider == "gogl":
			cloudProvider = semconv.CloudProviderGCP.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.CloudProviderKey), cloudProvider)
	backendAttrs.PutStr(string(semconv.CloudRegionKey), backend.CloudRegion)

	switch {
		case backend.OSType == "lnx":
			osType = semconv.OSTypeLinux.Value.AsString()
		case backend.OSType == "wndws":
			osType = semconv.OSTypeWindows.Value.AsString()
		case backend.OSType == "slrs":
			osType = semconv.OSTypeSolaris.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.OSTypeKey), osType)
	backendAttrs.PutStr(string(semconv.OSVersionKey), backend.OSVersion)
 }
```

`pcommon.Resource`에 `Atm` 엔티티와 `BackendSystem` 엔티티의 이름을 나타내는
"atm.name"이나 "backendsystem.name"이라는 속성을 추가하지 않았다는 점에
주목한다. 이는 OTel 트레이스 명세를 준수하는 대부분(전부라고 해도 무방할)의 분산
트레이싱 백엔드 시스템이 트레이스에 설명된 `pcommon.Resource`를 `Service`로
해석하기 때문이다. 따라서 이들은 `pcommon.Resource`가 리소스 시맨틱 컨벤션에서
규정한 `service.name`이라는 필수 속성을 가지고 있을 것으로 기대한다.

또한 `Atm`과 `BackendSystem` 두 엔티티 모두의 버전 정보를 나타내기 위해 필수는
아닌 `service.version` 속성도 사용한다.

"service." 그룹 속성을 제대로 지정하는 코드를 추가하고 나면
`tailtracer/model.go` 파일은 다음과 같은 모습이 된다.

> tailtracer/model.go

```go
package tailtracer

import (
	"math/rand"
	"time"

	"go.opentelemetry.io/collector/pdata/pcommon"
	"go.opentelemetry.io/collector/pdata/ptrace"
	"go.opentelemetry.io/otel/semconv/v1.43.0"
)

type Atm struct {
	ID           int64
	Version      string
	Name         string
	StateID      string
	SerialNumber string
	ISPNetwork   string
}

type BackendSystem struct {
	Version       string
	ProcessName   string
	OSType        string
	OSVersion     string
	CloudProvider string
	CloudRegion   string
	Endpoint      string
}

func generateAtm() Atm {
	i := getRandomNumber(1, 2)
	var newAtm Atm

	switch i {
	case 1:
		newAtm = Atm{
			ID:           111,
			Name:         "ATM-111-IL",
			SerialNumber: "atmxph-2022-111",
			Version:      "v1.0",
			ISPNetwork:   "comcast-chicago",
			StateID:      "IL",
		}

	case 2:
		newAtm = Atm{
			ID:           222,
			Name:         "ATM-222-CA",
			SerialNumber: "atmxph-2022-222",
			Version:      "v1.0",
			ISPNetwork:   "comcast-sanfrancisco",
			StateID:      "CA",
		}
	}

	return newAtm
}

func generateBackendSystem() BackendSystem {
	i := getRandomNumber(1, 3)

	newBackend := BackendSystem{
		ProcessName:   "accounts",
		Version:       "v2.5",
		OSType:        "lnx",
		OSVersion:     "4.16.10-300.fc28.x86_64",
		CloudProvider: "amzn",
		CloudRegion:   "us-east-2",
	}

	switch i {
	case 1:
		newBackend.Endpoint = "api/v2.5/balance"
	case 2:
		newBackend.Endpoint = "api/v2.5/deposit"
	case 3:
		newBackend.Endpoint = "api/v2.5/withdrawn"
	}

	return newBackend
}

func getRandomNumber(min int, max int) int {
	i := (rand.Intn(max-min+1) + min)
	return i
}

func generateTraces(numberOfTraces int) ptrace.Traces {
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++ {
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
		fillResourceWithAtm(&atmResource, newAtm)

		resourceSpan = traces.ResourceSpans().AppendEmpty()
		backendResource := resourceSpan.Resource()
		fillResourceWithBackendSystem(&backendResource, newBackendSystem)
	}

	return traces
}

func fillResourceWithAtm(resource *pcommon.Resource, atm Atm) {
	atmAttrs := resource.Attributes()
	atmAttrs.PutInt("atm.id", atm.ID)
	atmAttrs.PutStr("atm.stateid", atm.StateID)
	atmAttrs.PutStr("atm.ispnetwork", atm.ISPNetwork)
	atmAttrs.PutStr("atm.serialnumber", atm.SerialNumber)
	atmAttrs.PutStr(string(semconv.ServiceNameKey), atm.Name)
	atmAttrs.PutStr(string(semconv.ServiceVersionKey), atm.Version)

}

func fillResourceWithBackendSystem(resource *pcommon.Resource, backend BackendSystem) {
	backendAttrs := resource.Attributes()
	var osType, cloudProvider string

	switch {
	case backend.CloudProvider == "amzn":
		cloudProvider = semconv.CloudProviderAWS.Value.AsString()
	case backend.CloudProvider == "mcrsft":
		cloudProvider = semconv.CloudProviderAzure.Value.AsString()
	case backend.CloudProvider == "gogl":
		cloudProvider = semconv.CloudProviderGCP.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.CloudProviderKey), cloudProvider)
	backendAttrs.PutStr(string(semconv.CloudRegionKey), backend.CloudRegion)

	switch {
	case backend.OSType == "lnx":
		osType = semconv.OSTypeLinux.Value.AsString()
	case backend.OSType == "wndws":
		osType = semconv.OSTypeWindows.Value.AsString()
	case backend.OSType == "slrs":
		osType = semconv.OSTypeSolaris.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.OSTypeKey), osType)
	backendAttrs.PutStr(string(semconv.OSVersionKey), backend.OSVersion)

	backendAttrs.PutStr(string(semconv.ServiceNameKey), backend.ProcessName)
	backendAttrs.PutStr(string(semconv.ServiceVersionKey), backend.Version)
}
```

> [!NOTE] 작업 확인
>
> - `fillResourceWithAtm()` 함수를 업데이트해서 `Atm` 엔티티를 나타내는
>   `pcommon.Resource`에 "service.name" 속성과 "service.version" 속성을 제대로
>   지정하는 줄을 추가했다.
> - `fillResourceWithBackendSystem()` 함수를 업데이트해서 `BackendSystem`
>   엔티티를 나타내는 `pcommon.Resource`에 "service.name" 속성과
>   "service.version" 속성을 제대로 지정하는 줄을 추가했다.
> - `generateTraces` 함수를 업데이트해서 `pcommon.Resource`를 인스턴스화하고
>   `fillResourceWithAtm()` 함수와 `fillResourceWithBackendSystem()` 함수를
>   사용해 `Atm` 엔티티와 `BackendSystem` 엔티티 모두에 대한 속성 정보를 제대로
>   채우는 줄을 추가했다.

### 스팬으로 연산 나타내기 {#representing-operations-with-spans}

이제 `Atm` 엔티티와 `BackendSystem` 엔티티를 나타내는 속성으로 제대로 채워진
`Resource`를 가진 `ResourceSpan` 인스턴스가 생겼다. 이제 각 `Resource`가
트레이스의 일부로 실행하는 연산을 `ResourceSpan`에 나타낼 준비가 됐다.

OTel의 세계에서 시스템이 텔레메트리를 생성하려면 수동 또는 계측 라이브러리를
통한 자동 방식 중 하나로 계측되어야 한다.

계측 라이브러리는 스코프(계측 스코프라고도 함)를 설정하는 역할을 담당하며, 그
안에서 트레이스에 참여하는 연산이 발생하고, 이러한 연산을 트레이스의 맥락에서
스팬으로 설명한다.

`pdata.ResourceSpans`에는 `ScopeSpans()`라는 메서드가 있으며, 이는
`ptrace.ScopeSpansSlice`라는 헬퍼 타입의 인스턴스를 반환한다.
`ptrace.ScopeSpansSlice` 타입에는 `ptrace.ScopeSpans` 배열을 다루는 데 도움이
되는 메서드가 있다. 이 배열은 트레이스의 맥락에서 서로 다른 계측 스코프의
수만큼, 그리고 그것이 생성한 스팬의 수만큼 항목을 담게 된다.

`ptrace.ScopeSpansSlice`에는 배열에 새 `ptrace.ScopeSpans`를 추가하고 그 참조를
반환하는 `AppendEmpty()`라는 메서드가 있다.

ATM 시스템의 계측 스코프와 그 스팬을 나타내는 `ptrace.ScopeSpans`를
인스턴스화하는 함수를 만들어보자. `tailtracer/model.go` 파일을 열어 다음 함수를
추가한다.

```go
func appendAtmSystemInstrScopeSpans(resourceSpans *ptrace.ResourceSpans) ptrace.ScopeSpans {
	scopeSpans := resourceSpans.ScopeSpans().AppendEmpty()

	return scopeSpans
}
```

`ptrace.ScopeSpans`에는 스팬을 생성한 계측 스코프를 나타내는
`pcommon.InstrumentationScope` 인스턴스에 대한 참조를 반환하는 `Scope()`라는
메서드가 있다.

`pcommon.InstrumentationScope`에는 계측 스코프를 설명하는 다음 메서드가 있다.

- `SetName(v string)`은 계측 라이브러리의 이름을 설정한다.

- `SetVersion(v string)`은 계측 라이브러리의 버전을 설정한다.

- `Name() string`은 계측 라이브러리와 연관된 이름을 반환한다.

- `Version() string`은 계측 라이브러리와 연관된 버전을 반환한다.

새 `ptrace.ScopeSpans`의 계측 스코프의 이름과 버전을 설정할 수 있도록
`appendAtmSystemInstrScopeSpans` 함수를 업데이트해보자. 업데이트 후
`appendAtmSystemInstrScopeSpans`는 다음과 같은 모습이 된다.

```go
func appendAtmSystemInstrScopeSpans(resourceSpans *ptrace.ResourceSpans) ptrace.ScopeSpans {
	scopeSpans := resourceSpans.ScopeSpans().AppendEmpty()
	scopeSpans.Scope().SetName("atm-system")
	scopeSpans.Scope().SetVersion("v1.0")
	return scopeSpans
}
```

이제 `generateTraces` 함수를 업데이트하고, `appendAtmSystemInstrScopeSpans()`로
초기화하여 `Atm`과 `BackendSystem` 두 엔티티 모두가 사용하는 계측 스코프를
나타내는 변수를 추가할 수 있다. 업데이트 후 `generateTraces()`는 다음과 같은
모습이 된다.

```go
func generateTraces(numberOfTraces int) ptrace.Traces{
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++{
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
		fillResourceWithAtm(&atmResource, newAtm)

		atmInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		resourceSpan = traces.ResourceSpans().AppendEmpty()
		backendResource := resourceSpan.Resource()
		fillResourceWithBackendSystem(&backendResource, newBackendSystem)

		backendInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)
	}

	return traces
}
```

이 시점에서 시스템 내 텔레메트리 생성 엔티티를 나타내는 데 필요한 모든 것과,
연산을 식별하고 시스템을 위한 트레이스를 생성하는 역할을 담당하는 계측
스코프까지 갖췄다. 다음 단계는 주어진 계측 스코프가 트레이스의 일부로 생성한
연산을 나타내는 스팬을 만드는 것이다.

`ptrace.ScopeSpans`에는 `ptrace.SpanSlice`라는 헬퍼 타입의 인스턴스를 반환하는
`Spans()`라는 메서드가 있다. `ptrace.SpanSlice` 타입에는 `ptrace.Span` 배열을
다루는 데 도움이 되는 메서드가 있다. 이 배열은 계측 스코프가 트레이스의 일부로
식별하고 설명할 수 있었던 연산의 수만큼 항목을 담게 된다.

`ptrace.SpanSlice`에는 배열에 새 `ptrace.Span`을 추가하고 그 참조를 반환하는
`AppendEmpty()`라는 메서드가 있다.

`ptrace.Span`에는 연산을 설명하는 다음 메서드가 있다.

- `SetTraceID(v pcommon.TraceID)`는 이 스팬이 연관된 트레이스를 고유하게
  식별하는 `pcommon.TraceID`를 설정한다.

- `SetSpanID(v pcommon.SpanID)`는 이 스팬이 연관된 트레이스의 맥락에서 이 스팬을
  고유하게 식별하는 `pcommon.SpanID`를 설정한다.

- `SetParentSpanID(v pcommon.SpanID)`는 이 스팬이 나타내는 연산이 부모(중첩)
  연산의 일부로 실행되는 경우, 부모 스팬/연산의 `pcommon.SpanID`를 설정한다.

- `SetName(v string)`은 스팬의 연산 이름을 설정한다.

- `SetKind(v ptrace.SpanKind)`는 스팬이 나타내는 연산의 종류를 정의하는
  `ptrace.SpanKind`를 설정한다.

- `SetStartTimestamp(v pcommon.Timestamp)`는 스팬과 연관된 연산이 시작된 일시를
  나타내는 `pcommon.Timestamp`를 설정한다.

- `SetEndTimestamp(v pcommon.Timestamp)`는 스팬과 연관된 연산이 끝난 일시를
  나타내는 `pcommon.Timestamp`를 설정한다.

위 메서드에서 알 수 있듯이, `ptrace.Span`은 2개의 필수 ID로 고유하게 식별된다.
하나는 `pcommon.SpanID` 타입으로 표현되는 자기 자신의 고유 ID이고, 다른 하나는
`pcommon.TraceID` 타입으로 표현되는, 스팬이 연관된 트레이스의 ID다.

`pcommon.TraceID`는 16바이트 배열로 표현되는 전역적으로 고유한 ID를 가져야 하며,
[W3C 트레이스 컨텍스트 명세](https://www.w3.org/TR/trace-context/#trace-id)를
따라야 한다. `pcommon.SpanID`는 연관된 트레이스의 맥락에서 고유한 ID이며 8바이트
배열로 표현된다.

`pcommon` 패키지는 스팬 ID를 생성하기 위한 다음 타입을 제공한다.

- `type TraceID [16]byte`

- `type SpanID [8]byte`

이 튜토리얼에서는 `pcommon.TraceID`를 위해 `github.com/google/uuid` 패키지의
함수를, `pcommon.SpanID`를 무작위로 생성하기 위해 `crypto/rand` 패키지의 함수를
사용해 ID를 생성한다. 먼저 `tailtracer/model.go` 파일을 열어 두 패키지 모두를
`import` 문에 추가한다. 그런 다음 두 ID를 생성하는 데 도움이 되는 다음 함수를
추가한다.

```go
import (
	crand "crypto/rand"
	"math/rand"
  	...
)

func NewTraceID() pcommon.TraceID {
	return pcommon.TraceID(uuid.New())
}

func NewSpanID() pcommon.SpanID {
	var rngSeed int64
	_ = binary.Read(crand.Reader, binary.LittleEndian, &rngSeed)
	randSource := rand.New(rand.NewSource(rngSeed))

	var sid [8]byte
	randSource.Read(sid[:])
	spanID := pcommon.SpanID(sid)

	return spanID
}
```

> [!NOTE] 작업 확인
>
> - `math/rand`와의 충돌을 피하기 위해 `crypto/rand`를 `crand`로 임포트했다.
> - 각각 트레이스 ID와 스팬 ID를 생성하는 새 함수 `NewTraceID()`와
>   `NewSpanID()`를 추가했다.

이제 스팬을 제대로 식별할 방법이 생겼으니, 시스템 내 엔티티 안팎의 연산을
나타내는 스팬을 생성할 수 있다.

`generateBackendSystem()` 함수의 일부로, `BackEndSystem` 엔티티가 시스템에
서비스로 제공할 수 있는 연산을 무작위로 지정했다. 이제 `tailtracer/model.go`
파일을 열어 트레이스를 생성하고 `BackendSystem` 연산을 나타내는 스팬을 추가하는
역할을 하게 될 `appendTraceSpans()`라는 함수를 살펴본다. `appendTraceSpans()`
함수의 초기 구현은 다음과 같다.

```go
func appendTraceSpans(backend *BackendSystem, backendScopeSpans *ptrace.ScopeSpans, atmScopeSpans *ptrace.ScopeSpans) {
	traceId := NewTraceID()
	backendSpanId := NewSpanID()

	backendDuration, _ := time.ParseDuration("1s")
	backendSpanStartTime := time.Now()
	backendSpanFinishTime := backendSpanStartTime.Add(backendDuration)

	backendSpan := backendScopeSpans.Spans().AppendEmpty()
	backendSpan.SetTraceID(traceId)
	backendSpan.SetSpanID(backendSpanId)
	backendSpan.SetName(backend.Endpoint)
	backendSpan.SetKind(ptrace.SpanKindServer)
	backendSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(backendSpanStartTime))
	backendSpan.SetEndTimestamp(pcommon.NewTimestampFromTime(backendSpanFinishTime))
}
```

> [!NOTE] 작업 확인
>
> - 각각 트레이스 ID와 스팬 ID를 나타내는 `traceId` 변수와 `backendSpanId`
>   변수를 추가하고, 이전에 만든 헬퍼 함수로 초기화했다.
> - 연산의 시작 시간과 종료 시간을 나타내는 `backendSpanStartTime` 변수와
>   `backendSpanFinishTime` 변수를 추가했다. 이 튜토리얼에서는 모든
>   `BackendSystem` 연산이 1초가 걸린다고 가정한다.
> - 이 연산을 나타내는 `ptrace.Span` 인스턴스를 담을 `backendSpan`이라는 이름의
>   변수를 추가했다.
> - `BackendSystem` 인스턴스의 `Endpoint` 필드 값으로 스팬의 `Name`을 설정했다.
> - 스팬의 `Kind`를 `ptrace.SpanKindServer`로 설정했다. SpanKind를 제대로
>   정의하는 방법을 이해하려면 트레이스 명세의
>   [SpanKind 절](/docs/specs/otel/trace/api/#spankind)을 참고한다.
> - 위에서 언급한 모든 메서드를 사용해 `BackendSystem` 연산을 나타내는 데 필요한
>   값으로 `ptrace.Span`을 채웠다.

`appendTraceSpans()` 함수의 매개변수로 `ptrace.ScopeSpans`에 대한 2개의 참조가
있다는 점을 눈치챘을 수도 있지만, 그중 하나만 사용했다. 지금은 신경 쓰지 않아도
된다. 나중에 다시 다룰 것이다.

다음으로, `appendTraceSpans()` 함수를 호출해서 트레이스를 생성할 수 있도록
`generateTraces()` 함수를 업데이트한다. 업데이트 후 `generateTraces()` 함수는
다음과 같은 모습이 된다.

```go
func generateTraces(numberOfTraces int) ptrace.Traces {
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++ {
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
		fillResourceWithAtm(&atmResource, newAtm)

		atmInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		resourceSpan = traces.ResourceSpans().AppendEmpty()
		backendResource := resourceSpan.Resource()
		fillResourceWithBackendSystem(&backendResource, newBackendSystem)

		backendInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		appendTraceSpans(&newBackendSystem, &backendInstScope, &atmInstScope)
	}

	return traces
}
```

이제 `BackendSystem` 엔티티와 그 연산이 제대로 된 트레이스 맥락 안에서 스팬으로
표현됐다! 다음으로, 생성된 트레이스를 파이프라인을 통해 전달해서 다음
컨슈머(프로세서 또는 익스포터)가 이를 수신하고 처리할 수 있게 해야 한다.

`tailtracer/model.go` 파일은 다음과 같은 모습이다.

> tailtracer/model.go

```go
package tailtracer

import (
	crand "crypto/rand"
	"encoding/binary"
	"math/rand"
	"time"

	"github.com/google/uuid"
	"go.opentelemetry.io/collector/pdata/pcommon"
	"go.opentelemetry.io/collector/pdata/ptrace"
	"go.opentelemetry.io/otel/semconv/v1.43.0"
)

type Atm struct {
	ID           int64
	Version      string
	Name         string
	StateID      string
	SerialNumber string
	ISPNetwork   string
}

type BackendSystem struct {
	Version       string
	ProcessName   string
	OSType        string
	OSVersion     string
	CloudProvider string
	CloudRegion   string
	Endpoint      string
}

func generateAtm() Atm {
	i := getRandomNumber(1, 2)
	var newAtm Atm

	switch i {
	case 1:
		newAtm = Atm{
			ID:           111,
			Name:         "ATM-111-IL",
			SerialNumber: "atmxph-2022-111",
			Version:      "v1.0",
			ISPNetwork:   "comcast-chicago",
			StateID:      "IL",
		}

	case 2:
		newAtm = Atm{
			ID:           222,
			Name:         "ATM-222-CA",
			SerialNumber: "atmxph-2022-222",
			Version:      "v1.0",
			ISPNetwork:   "comcast-sanfrancisco",
			StateID:      "CA",
		}
	}

	return newAtm
}

func generateBackendSystem() BackendSystem {
	i := getRandomNumber(1, 3)

	newBackend := BackendSystem{
		ProcessName:   "accounts",
		Version:       "v2.5",
		OSType:        "lnx",
		OSVersion:     "4.16.10-300.fc28.x86_64",
		CloudProvider: "amzn",
		CloudRegion:   "us-east-2",
	}

	switch i {
	case 1:
		newBackend.Endpoint = "api/v2.5/balance"
	case 2:
		newBackend.Endpoint = "api/v2.5/deposit"
	case 3:
		newBackend.Endpoint = "api/v2.5/withdrawn"
	}

	return newBackend
}

func getRandomNumber(min int, max int) int {
	i := (rand.Intn(max-min+1) + min)
	return i
}

func generateTraces(numberOfTraces int) ptrace.Traces {
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++ {
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
		fillResourceWithAtm(&atmResource, newAtm)

		atmInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		resourceSpan = traces.ResourceSpans().AppendEmpty()
		backendResource := resourceSpan.Resource()
		fillResourceWithBackendSystem(&backendResource, newBackendSystem)

		backendInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		appendTraceSpans(&newBackendSystem, &backendInstScope, &atmInstScope)
	}

	return traces
}

func fillResourceWithAtm(resource *pcommon.Resource, atm Atm) {
	atmAttrs := resource.Attributes()
	atmAttrs.PutInt("atm.id", atm.ID)
	atmAttrs.PutStr("atm.stateid", atm.StateID)
	atmAttrs.PutStr("atm.ispnetwork", atm.ISPNetwork)
	atmAttrs.PutStr("atm.serialnumber", atm.SerialNumber)
	atmAttrs.PutStr(string(semconv.ServiceNameKey), atm.Name)
	atmAttrs.PutStr(string(semconv.ServiceVersionKey), atm.Version)

}

func fillResourceWithBackendSystem(resource *pcommon.Resource, backend BackendSystem) {
	backendAttrs := resource.Attributes()
	var osType, cloudProvider string

	switch {
	case backend.CloudProvider == "amzn":
		cloudProvider = semconv.CloudProviderAWS.Value.AsString()
	case backend.CloudProvider == "mcrsft":
		cloudProvider = semconv.CloudProviderAzure.Value.AsString()
	case backend.CloudProvider == "gogl":
		cloudProvider = semconv.CloudProviderGCP.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.CloudProviderKey), cloudProvider)
	backendAttrs.PutStr(string(semconv.CloudRegionKey), backend.CloudRegion)

	switch {
	case backend.OSType == "lnx":
		osType = semconv.OSTypeLinux.Value.AsString()
	case backend.OSType == "wndws":
		osType = semconv.OSTypeWindows.Value.AsString()
	case backend.OSType == "slrs":
		osType = semconv.OSTypeSolaris.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.OSTypeKey), osType)
	backendAttrs.PutStr(string(semconv.OSVersionKey), backend.OSVersion)

	backendAttrs.PutStr(string(semconv.ServiceNameKey), backend.ProcessName)
	backendAttrs.PutStr(string(semconv.ServiceVersionKey), backend.Version)
}

func appendAtmSystemInstrScopeSpans(resourceSpans *ptrace.ResourceSpans) ptrace.ScopeSpans {
	scopeSpans := resourceSpans.ScopeSpans().AppendEmpty()
	scopeSpans.Scope().SetName("atm-system")
	scopeSpans.Scope().SetVersion("v1.0")
	return scopeSpans
}

func NewTraceID() pcommon.TraceID {
	return pcommon.TraceID(uuid.New())
}

func NewSpanID() pcommon.SpanID {
	var rngSeed int64
	_ = binary.Read(crand.Reader, binary.LittleEndian, &rngSeed)
	randSource := rand.New(rand.NewSource(rngSeed))

	var sid [8]byte
	randSource.Read(sid[:])
	spanID := pcommon.SpanID(sid)

	return spanID
}

func appendTraceSpans(backend *BackendSystem, backendScopeSpans *ptrace.ScopeSpans, atmScopeSpans *ptrace.ScopeSpans) {
	traceId := NewTraceID()
	backendSpanId := NewSpanID()

	backendDuration, _ := time.ParseDuration("1s")
	backendSpanStartTime := time.Now()
	backendSpanFinishTime := backendSpanStartTime.Add(backendDuration)

	backendSpan := backendScopeSpans.Spans().AppendEmpty()
	backendSpan.SetTraceID(traceId)
	backendSpan.SetSpanID(backendSpanId)
	backendSpan.SetName(backend.Endpoint)
	backendSpan.SetKind(ptrace.SpanKindServer)
	backendSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(backendSpanStartTime))
	backendSpan.SetEndTimestamp(pcommon.NewTimestampFromTime(backendSpanFinishTime))
}
```

`consumer.Traces`에는 생성된 트레이스를 파이프라인의 다음 컨슈머로 전달하는
역할을 하는 `ConsumeTraces()`라는 메서드가 있다. `tailtracerReceiver` 타입의
`Start()` 메서드를 업데이트해서 이를 사용하는 코드를 추가해야 한다.

`tailtracer/trace-receiver.go` 파일을 열어 다음과 같이 `Start()` 메서드를
업데이트한다.

```go
func (tailtracerRcvr *tailtracerReceiver) Start(ctx context.Context, host component.Host) error {
	tailtracerRcvr.host = host
	ctx = context.Background()
	ctx, tailtracerRcvr.cancel = context.WithCancel(ctx)

	interval, _ := time.ParseDuration(tailtracerRcvr.config.Interval)
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
				case <-ticker.C:
					tailtracerRcvr.logger.Info("I should start processing traces now!")
					tailtracerRcvr.nextConsumer.ConsumeTraces(ctx, generateTraces(tailtracerRcvr.config.NumberOfTraces)) // new line added
				case <-ctx.Done():
					return
			}
		}
	}()

	return nil
}
```

> [!NOTE] 작업 확인
>
> - `case <=ticker.C` 조건 아래에 `tailtracerRcvr.nextConsumer.ConsumeTraces()`
>   메서드를 호출하는 줄을 추가하고, `Start()` 메서드에서 생성한 새 컨텍스트
>   (`ctx`)와 `generateTraces()` 함수 호출을 전달해서, 생성된 트레이스가
>   파이프라인의 다음 컨슈머로 전달될 수 있게 했다.

이제 `otelcol-dev`를 다시 실행해보자.

```sh
go run ./otelcol-dev --config config.yaml
```

몇 분 후 다음과 같은 출력이 보여야 한다.

```log
2023-11-09T11:38:19.890+0800	info	service@v0.88.0/telemetry.go:84	Setting up own telemetry...
2023-11-09T11:38:19.890+0800	info	service@v0.88.0/telemetry.go:201	Serving Prometheus metrics	{"address": ":8888", "level": "Basic"}
2023-11-09T11:38:19.890+0800	debug	exporter@v0.88.0/exporter.go:273	Stable component.	{"kind": "exporter", "data_type": "traces", "name": "otlp/jaeger"}
2023-11-09T11:38:19.890+0800	info	exporter@v0.88.0/exporter.go:275	Development component. May change in the future.	{"kind": "exporter", "data_type": "traces", "name": "debug"}
2023-11-09T11:38:19.891+0800	debug	receiver@v0.88.0/receiver.go:294	Stable component.	{"kind": "receiver", "name": "otlp", "data_type": "traces"}
2023-11-09T11:38:19.891+0800	debug	receiver@v0.88.0/receiver.go:294	Alpha component. May change in the future.	{"kind": "receiver", "name": "tailtracer", "data_type": "traces"}
2023-11-09T11:38:19.891+0800	info	service@v0.88.0/service.go:143	Starting otelcol-dev...	{"Version": "1.0.0", "NumCPU": 10}
2023-11-09T11:38:19.891+0800	info	extensions/extensions.go:33	Starting extensions...

<OMITTED>

2023-11-09T11:38:19.903+0800	info	zapgrpc/zapgrpc.go:178	[core] [Channel #1] Channel Connectivity change to READY	{"grpc_log": true}
2023-11-09T11:39:19.894+0800	info	tailtracer/trace-receiver.go:33	I should start processing traces now!	{"kind": "receiver", "name": "tailtracer", "data_type": "traces"}
2023-11-09T11:39:19.913+0800	info	TracesExporter	{"kind": "exporter", "data_type": "traces", "name": "debug", "resource spans": 4, "spans": 2}
2023-11-09T11:39:19.913+0800	info	ResourceSpans #0
Resource SchemaURL:
Resource attributes:
     -> atm.id: Int(222)
     -> atm.stateid: Str(CA)
     -> atm.ispnetwork: Str(comcast-sanfrancisco)
     -> atm.serialnumber: Str(atmxph-2022-222)
     -> service.name: Str(ATM-222-CA)
     -> service.version: Str(v1.0)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope
ResourceSpans #1
Resource SchemaURL:
Resource attributes:
     -> cloud.provider: Str(aws)
     -> cloud.region: Str(us-east-2)
     -> os.type: Str(linux)
     -> os.version: Str(4.16.10-300.fc28.x86_64)
     -> service.name: Str(accounts)
     -> service.version: Str(v2.5)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope
Span #0
    Trace ID       : bbcb00aead044a138cf96c0bf4a4ba83
    Parent ID      :
    ID             : 5056fe4e9adf621c
    Name           : api/v2.5/withdrawn
    Kind           : Server
    Start time     : 2023-11-09 03:39:19.894881 +0000 UTC
    End time       : 2023-11-09 03:39:20.894881 +0000 UTC
    Status code    : Unset
    Status message :
ResourceSpans #2
Resource SchemaURL:
Resource attributes:
     -> atm.id: Int(111)
     -> atm.stateid: Str(IL)
     -> atm.ispnetwork: Str(comcast-chicago)
     -> atm.serialnumber: Str(atmxph-2022-111)
     -> service.name: Str(ATM-111-IL)
     -> service.version: Str(v1.0)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope
ResourceSpans #3
Resource SchemaURL:
Resource attributes:
     -> cloud.provider: Str(aws)
     -> cloud.region: Str(us-east-2)
     -> os.type: Str(linux)
     -> os.version: Str(4.16.10-300.fc28.x86_64)
     -> service.name: Str(accounts)
     -> service.version: Str(v2.5)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope
Span #0
    Trace ID       : ba013b8223ec4d29806ae493ecd1a5e4
    Parent ID      :
    ID             : 4feb47b55c9c4129
    Name           : api/v2.5/withdrawn
    Kind           : Server
    Start time     : 2023-11-09 03:39:19.894953 +0000 UTC
    End time       : 2023-11-09 03:39:20.894953 +0000 UTC
    Status code    : Unset
    Status message :
	{"kind": "exporter", "data_type": "traces", "name": "debug"}
...
```

Jaeger에서 생성된 트레이스는 다음과 같은 모습이다.
![Jaeger trace](/img/docs/tutorials/Jaeger-BackendSystem-Trace.png)

지금 Jaeger에서 보이는 것은 OTel SDK로 계측되지 않은 외부 엔티티로부터 요청을
수신하는 서비스를 나타낸다. 그 결과, 이는 트레이스의 시작/기원으로 식별될 수
없다. `ptrace.Span`이 같은 트레이스 맥락에서 `Resource` 내부 또는
외부(중첩/자식)에서 시작된 다른 연산의 결과로 실행된 연산을 나타낸다는 것을
이해하려면 다음을 수행해야 한다.

- 부모/호출자 `ptrace.Span`의 `pcommon.TraceID`를 매개변수로 전달해서
  `SetTraceID()` 메서드를 호출함으로써 호출자 연산과 같은 트레이스 맥락을
  설정한다.
- 부모/호출자 `ptrace.Span`의 `pcommon.SpanID`를 매개변수로 전달해서
  `SetParentId()` 메서드를 호출함으로써 트레이스 맥락에서 호출자 연산을
  정의한다.

이제 `Atm` 엔티티 연산을 나타내는 `ptrace.Span`을 만들고 이를 `BackendSystem`
스팬의 부모로 설정한다. `tailtracer/model.go` 파일을 열어 다음과 같이
`appendTraceSpans()` 함수를 업데이트한다.

```go
func appendTraceSpans(backend *BackendSystem, backendScopeSpans *ptrace.ScopeSpans, atmScopeSpans *ptrace.ScopeSpans) {
	traceId := NewTraceID()

	var atmOperationName string

	switch {
		case strings.Contains(backend.Endpoint, "balance"):
			atmOperationName = "Check Balance"
		case strings.Contains(backend.Endpoint, "deposit"):
			atmOperationName = "Make Deposit"
		case strings.Contains(backend.Endpoint, "withdraw"):
			atmOperationName = "Fast Cash"
		}

	atmSpanId := NewSpanID()
	atmSpanStartTime := time.Now()
	atmDuration, _ := time.ParseDuration("4s")
	atmSpanFinishTime := atmSpanStartTime.Add(atmDuration)

	atmSpan := atmScopeSpans.Spans().AppendEmpty()
	atmSpan.SetTraceID(traceId)
	atmSpan.SetSpanID(atmSpanId)
	atmSpan.SetName(atmOperationName)
	atmSpan.SetKind(ptrace.SpanKindClient)
	atmSpan.Status().SetCode(ptrace.StatusCodeOk)
	atmSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(atmSpanStartTime))
	atmSpan.SetEndTimestamp(pcommon.NewTimestampFromTime(atmSpanFinishTime))

	backendSpanId := NewSpanID()

	backendDuration, _ := time.ParseDuration("2s")
	backendSpanStartTime := atmSpanStartTime.Add(backendDuration)

	backendSpan := backendScopeSpans.Spans().AppendEmpty()
	backendSpan.SetTraceID(atmSpan.TraceID())
	backendSpan.SetSpanID(backendSpanId)
	backendSpan.SetParentSpanID(atmSpan.SpanID())
	backendSpan.SetName(backend.Endpoint)
	backendSpan.SetKind(ptrace.SpanKindServer)
	backendSpan.Status().SetCode(ptrace.StatusCodeOk)
	backendSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(backendSpanStartTime))
	backendSpan.SetEndTimestamp(atmSpan.EndTimestamp())
}
```

`tailtracer/model.go` 파일의 최종 모습은 다음과 같다.

> tailtracer/model.go

```go
package tailtracer

import (
	crand "crypto/rand"
	"encoding/binary"
	"math/rand"
	"strings"
	"time"

	"github.com/google/uuid"
	"go.opentelemetry.io/collector/pdata/pcommon"
	"go.opentelemetry.io/collector/pdata/ptrace"
	 "go.opentelemetry.io/otel/semconv/v1.43.0"
)

type Atm struct {
	ID           int64
	Version      string
	Name         string
	StateID      string
	SerialNumber string
	ISPNetwork   string
}

type BackendSystem struct {
	Version       string
	ProcessName   string
	OSType        string
	OSVersion     string
	CloudProvider string
	CloudRegion   string
	Endpoint      string
}

func generateAtm() Atm {
	i := getRandomNumber(1, 2)
	var newAtm Atm

	switch i {
	case 1:
		newAtm = Atm{
			ID:           111,
			Name:         "ATM-111-IL",
			SerialNumber: "atmxph-2022-111",
			Version:      "v1.0",
			ISPNetwork:   "comcast-chicago",
			StateID:      "IL",
		}

	case 2:
		newAtm = Atm{
			ID:           222,
			Name:         "ATM-222-CA",
			SerialNumber: "atmxph-2022-222",
			Version:      "v1.0",
			ISPNetwork:   "comcast-sanfrancisco",
			StateID:      "CA",
		}
	}

	return newAtm
}

func generateBackendSystem() BackendSystem {
	i := getRandomNumber(1, 3)

	newBackend := BackendSystem{
		ProcessName:   "accounts",
		Version:       "v2.5",
		OSType:        "lnx",
		OSVersion:     "4.16.10-300.fc28.x86_64",
		CloudProvider: "amzn",
		CloudRegion:   "us-east-2",
	}

	switch i {
	case 1:
		newBackend.Endpoint = "api/v2.5/balance"
	case 2:
		newBackend.Endpoint = "api/v2.5/deposit"
	case 3:
		newBackend.Endpoint = "api/v2.5/withdrawn"
	}

	return newBackend
}

func getRandomNumber(min int, max int) int {
	i := (rand.Intn(max-min+1) + min)
	return i
}

func generateTraces(numberOfTraces int) ptrace.Traces {
	traces := ptrace.NewTraces()

	for i := 0; i < numberOfTraces; i++ {
		newAtm := generateAtm()
		newBackendSystem := generateBackendSystem()

		resourceSpan := traces.ResourceSpans().AppendEmpty()
		atmResource := resourceSpan.Resource()
		fillResourceWithAtm(&atmResource, newAtm)

		atmInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		resourceSpan = traces.ResourceSpans().AppendEmpty()
		backendResource := resourceSpan.Resource()
		fillResourceWithBackendSystem(&backendResource, newBackendSystem)

		backendInstScope := appendAtmSystemInstrScopeSpans(&resourceSpan)

		appendTraceSpans(&newBackendSystem, &backendInstScope, &atmInstScope)
	}

	return traces
}

func fillResourceWithAtm(resource *pcommon.Resource, atm Atm) {
	atmAttrs := resource.Attributes()
	atmAttrs.PutInt("atm.id", atm.ID)
	atmAttrs.PutStr("atm.stateid", atm.StateID)
	atmAttrs.PutStr("atm.ispnetwork", atm.ISPNetwork)
	atmAttrs.PutStr("atm.serialnumber", atm.SerialNumber)
	atmAttrs.PutStr(string(semconv.ServiceNameKey), atm.Name)
	atmAttrs.PutStr(string(semconv.ServiceVersionKey), atm.Version)

}

func fillResourceWithBackendSystem(resource *pcommon.Resource, backend BackendSystem) {
	backendAttrs := resource.Attributes()
	var osType, cloudProvider string

	switch {
	case backend.CloudProvider == "amzn":
		cloudProvider = semconv.CloudProviderAWS.Value.AsString()
	case backend.CloudProvider == "mcrsft":
		cloudProvider = semconv.CloudProviderAzure.Value.AsString()
	case backend.CloudProvider == "gogl":
		cloudProvider = semconv.CloudProviderGCP.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.CloudProviderKey), cloudProvider)
	backendAttrs.PutStr(string(semconv.CloudRegionKey), backend.CloudRegion)

	switch {
	case backend.OSType == "lnx":
		osType = semconv.OSTypeLinux.Value.AsString()
	case backend.OSType == "wndws":
		osType = semconv.OSTypeWindows.Value.AsString()
	case backend.OSType == "slrs":
		osType = semconv.OSTypeSolaris.Value.AsString()
	}

	backendAttrs.PutStr(string(semconv.OSTypeKey), osType)
	backendAttrs.PutStr(string(semconv.OSVersionKey), backend.OSVersion)

	backendAttrs.PutStr(string(semconv.ServiceNameKey), backend.ProcessName)
	backendAttrs.PutStr(string(semconv.ServiceVersionKey), backend.Version)
}

func appendAtmSystemInstrScopeSpans(resourceSpans *ptrace.ResourceSpans) ptrace.ScopeSpans {
	scopeSpans := resourceSpans.ScopeSpans().AppendEmpty()
	scopeSpans.Scope().SetName("atm-system")
	scopeSpans.Scope().SetVersion("v1.0")
	return scopeSpans
}

func NewTraceID() pcommon.TraceID {
	return pcommon.TraceID(uuid.New())
}

func NewSpanID() pcommon.SpanID {
	var rngSeed int64
	_ = binary.Read(crand.Reader, binary.LittleEndian, &rngSeed)
	randSource := rand.New(rand.NewSource(rngSeed))

	var sid [8]byte
	randSource.Read(sid[:])
	spanID := pcommon.SpanID(sid)

	return spanID
}

func appendTraceSpans(backend *BackendSystem, backendScopeSpans *ptrace.ScopeSpans, atmScopeSpans *ptrace.ScopeSpans) {
	traceId := NewTraceID()

	var atmOperationName string

	switch {
	case strings.Contains(backend.Endpoint, "balance"):
		atmOperationName = "Check Balance"
	case strings.Contains(backend.Endpoint, "deposit"):
		atmOperationName = "Make Deposit"
	case strings.Contains(backend.Endpoint, "withdraw"):
		atmOperationName = "Fast Cash"
	}

	atmSpanId := NewSpanID()
	atmSpanStartTime := time.Now()
	atmDuration, _ := time.ParseDuration("4s")
	atmSpanFinishTime := atmSpanStartTime.Add(atmDuration)

	atmSpan := atmScopeSpans.Spans().AppendEmpty()
	atmSpan.SetTraceID(traceId)
	atmSpan.SetSpanID(atmSpanId)
	atmSpan.SetName(atmOperationName)
	atmSpan.SetKind(ptrace.SpanKindClient)
	atmSpan.Status().SetCode(ptrace.StatusCodeOk)
	atmSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(atmSpanStartTime))
	atmSpan.SetEndTimestamp(pcommon.NewTimestampFromTime(atmSpanFinishTime))

	backendSpanId := NewSpanID()

	backendDuration, _ := time.ParseDuration("2s")
	backendSpanStartTime := atmSpanStartTime.Add(backendDuration)

	backendSpan := backendScopeSpans.Spans().AppendEmpty()
	backendSpan.SetTraceID(atmSpan.TraceID())
	backendSpan.SetSpanID(backendSpanId)
	backendSpan.SetParentSpanID(atmSpan.SpanID())
	backendSpan.SetName(backend.Endpoint)
	backendSpan.SetKind(ptrace.SpanKindServer)
	backendSpan.Status().SetCode(ptrace.StatusCodeOk)
	backendSpan.SetStartTimestamp(pcommon.NewTimestampFromTime(backendSpanStartTime))
	backendSpan.SetEndTimestamp(atmSpan.EndTimestamp())
}
```

`otelcol-dev`를 다시 실행한다.

```sh
go run ./otelcol-dev --config config.yaml
```

약 2분 후, 다음과 같은 트레이스가 Jaeger에 표시되기 시작해야 한다.
![Jaeger trace](/img/docs/tutorials/Jaeger-FullSystem-Traces-List.png)

이제 시스템에서 `Atm`과 `BackendSystem` 텔레메트리 생성 엔티티를 모두 나타내는
서비스가 생겼다. 두 엔티티가 어떻게 사용되고 사용자가 실행하는 연산의 성능에
어떻게 기여하는지 완전히 이해하게 됐다.

다음은 그 트레이스 중 하나에 대한 상세 보기다.
![Jaeger trace](/img/docs/tutorials/Jaeger-FullSystem-Trace-Details.png)

이것으로 끝이다! 이제 이 튜토리얼의 끝에 도달했고 트레이스 리시버를 성공적으로
구현했다. 축하한다!
