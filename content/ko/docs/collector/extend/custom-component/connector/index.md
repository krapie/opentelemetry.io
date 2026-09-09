---
title: 커넥터 빌드하기
linkTitle: 커넥터
aliases:
  - /docs/collector/build-connector
  - /docs/collector/building/connector
weight: 200
# prettier-ignore
cSpell:ignore: debugexporter exampleconnector gomod gord Jaglowski mapstructure otlpreceiver pdata pmetric ptrace servicegraph spanmetrics struct uber
default_lang_commit: e151bde15ac8bfd079726454edcf5bd91516a33a
---

## 오픈텔레메트리의 커넥터 {#connectors-in-opentelemetry}

이 페이지의 내용은 이미 어떤 형태로든 트레이싱 텔레메트리 데이터를 생성하는
계측된(instrumented) 애플리케이션을 가지고 있고,
[오픈텔레메트리(OpenTelemetry) 컬렉터](/docs/collector)에 대한 이해가 있는
경우에 가장 적합하다.

## 커넥터란 무엇인가? {#what-is-a-connector}

커넥터(Connector)는 서로 다른 컬렉터 파이프라인을 연결하여 그 사이에서
텔레메트리 데이터를 전송하는 수단 역할을 한다. 커넥터는 한 파이프라인에는
익스포터(Exporter)로, 다른 파이프라인에는 리시버(Receiver)로 동작한다.
오픈텔레메트리 컬렉터의 각 파이프라인은 한 가지 유형의 텔레메트리 데이터만
다룬다. 한 형태의 텔레메트리 데이터를 다른 형태로 처리해야 할 필요가 있을 수
있지만, 이때는 해당 데이터를 알맞은 컬렉터 파이프라인으로 라우팅(route)해야
한다.

## 커넥터를 사용하는 이유는 무엇인가? {#why-use-a-connector}

커넥터는 데이터 스트림을 병합(merge), 라우팅, 복제(replicate)하는 데 유용하다.
파이프라인을 서로 연결하는 순차적 파이프라이닝(sequential pipelining)과 더불어,
커넥터 구성 요소(component)는 조건부 데이터 흐름(conditional data flow)과 생성된
데이터 스트림(generated data streams)도 지원할 수 있다. 조건부 데이터 흐름이란
데이터를 우선순위가 가장 높은 파이프라인으로 보내고, 필요할 경우 대체
파이프라인으로 라우팅하는 오류 감지 기능을 갖추는 것을 의미한다. 생성된 데이터
스트림이란 구성 요소가 수신한 데이터를 바탕으로 자체적으로 데이터를 생성하고
내보내는 것을 의미한다. 이 튜토리얼(tutorial)은 파이프라인을 연결하는 커넥터의
기능에 중점을 둔다.

오픈텔레메트리에는 한 유형의 텔레메트리 데이터를 다른 유형으로 변환하는
프로세서(Processor)가 있다. 몇 가지 예로 spanmetrics 프로세서와 servicegraph
프로세서가 있다. spanmetrics 프로세서는 스팬 데이터로부터 집계된 요청, 오류,
지속 시간(duration) 메트릭을 생성한다. servicegraph 프로세서는 트레이스 데이터를
분석하여 서비스 간의 관계를 설명하는 메트릭을 생성한다. 이 두 프로세서는 모두
트레이스 데이터를 받아들여 메트릭 데이터로 변환한다. 오픈텔레메트리 컬렉터의
파이프라인은 한 가지 유형의 데이터만 다루므로, 프로세서에서 나온 트레이스
데이터를 트레이스 파이프라인에서 변환하여 메트릭 파이프라인으로 보내야 한다.
과거에는 일부 프로세서가 처리 후 데이터를 직접 내보내는 나쁜 관행을 따르는 우회
방법(workaround)을 사용해 데이터를 전달했다. 커넥터 구성 요소는 이런 우회 방법의
필요성을 해소했고, 이 우회 방법을 사용하던 프로세서들은 지원
중단(deprecated)되었다. 같은 맥락에서, 앞서 언급한 프로세서들도 최근 릴리스에서
지원 중단되었고 커넥터로 대체되었다.

커넥터의 전체 기능에 대한 자세한 내용은 다음 링크에서 확인할 수 있다:
[What are Connectors in OpenTelemetry?](https://bindplane.com/blog/what-are-connectors-in-opentelemetry),
[오픈텔레메트리 커넥터 설정](/docs/collector/configuration/#connectors)

### 기존 아키텍처: {#the-old-architecture}

![프로세서가 데이터를 다른 파이프라인의 익스포터로 직접 내보내던 방식을 보여주는 이전 그림](./otel-collector-before-connector.svg)

### 커넥터를 사용하는 새 아키텍처: {#new-architecture-using-a-connector}

![커넥터 구성 요소를 사용했을 때 파이프라인이 동작해야 하는 방식](./otel-collector-after-connector.svg)

## 예제 커넥터 빌드하기 {#building-example-connector}

이 튜토리얼에서는 오픈텔레메트리의 커넥터 구성 요소가 어떻게 동작하는지 보여주는
기본 예제로, 트레이스를 받아 메트릭으로 변환하는 예제 커넥터를 작성한다. 이 기본
커넥터의 기능은 특정 속성(attribute) 이름을 포함한 트레이스 안의 스팬 개수를
단순히 세는 것이다. 이렇게 센 횟수는 커넥터 안에 저장된다.

## 설정 {#configurations}

### 컬렉터 설정 구성하기: {#setting-up-collector-config}

오픈텔레메트리 컬렉터에 사용할 설정을 `config.yaml` 파일에 구성한다. 이 파일은
데이터가 어떻게 라우팅되고, 처리되고, 내보내지는지를 정의한다. 이 파일에 정의된
설정은 데이터 파이프라인이 어떻게 동작하기를 원하는지를 상세히 기술한다. 구성
요소와 정의한 파이프라인을 따라 데이터가 처음부터 끝까지 어떻게 이동하는지를
정의할 수 있다. 컬렉터를 설정하는 방법에 대한 자세한 내용은
[컬렉터 설정](/docs/collector/configuration)을 참고한다.

다음은 우리가 빌드할 예제 커넥터에 사용할 코드이다. 이 코드는 유효한 기본
오픈텔레메트리 컬렉터 설정 파일의 예시이다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:

connectors:
  example:

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [example]
    metrics:
      receivers: [example]
      exporters: [debug]
```

위 예제 코드의 `connectors` 섹션에서 파이프라인에 사용할 커넥터의 이름을
선언해야 한다. 여기서 `example`은 이 튜토리얼에서 만들 커넥터의 이름이다.

## 구현 {#implementation}

1.  예제 커넥터를 위한 폴더를 만든다. 이 튜토리얼에서는 `exampleconnector`라는
    폴더를 만든다.
2.  해당 폴더로 이동하여 다음을 실행한다.

    ```sh
    go mod init github.com/gord02/exampleconnector
    ```

3.  `go mod tidy`를 실행한다.

    이 명령은 `go.mod`와 `go.sum` 파일을 생성한다.

4.  폴더에 다음 파일을 생성한다.
    - `config.go` - 커넥터의 설정을 정의하는 파일
    - `factory.go` - 커넥터의 인스턴스를 생성하는 파일

### config.go에 커넥터 설정 만들기 {#create-your-connector-settings-in-configgo}

인스턴스화되어 파이프라인에 참여하려면, 컬렉터가 커넥터를 식별하고 설정 파일에서
해당 설정을 올바르게 불러올 수 있어야 한다.

커넥터가 자신의 설정에 접근할 수 있도록 하려면 `Config` 구조체(struct)를 만든다.
이 구조체는 커넥터의 각 설정마다 내보내진(exported) 필드를 가져야 한다. 추가한
매개변수 필드는 config.yaml 파일에서 접근할 수 있다. 설정 파일에서의 이름은
구조체 태그(struct tag)를 통해 지정된다. 구조체를 만들고 매개변수를 추가한다.
필요하다면 커넥터 인스턴스에 주어진 기본값이 유효한지 확인하는 검증
함수(validator function)를 추가할 수도 있다.

`config.go` 파일은 다음과 같은 모습이어야 한다.

> exampleconnector/config.go

```go
package exampleconnector

import "fmt"

// Config represents the connector config settings within the collector's config.yaml
type Config struct {
    AttributeName string `mapstructure:"attribute_name"`
}

func (c *Config) Validate() error {
    if c.AttributeName == "" {
        return fmt.Errorf("attribute_name must not be empty")
    }
    return nil
}
```

mapstructure에 대한 자세한 내용은
[Go mapstructure](https://pkg.go.dev/github.com/mitchellh/mapstructure)에서
확인할 수 있다.

## 팩토리 구현하기 {#implement-the-factory}

객체를 인스턴스화하려면 각 구성 요소에 연관된 `NewFactory` 함수를 사용해야 한다.
여기서는 `connector.NewFactory` 함수를 사용한다. `connector.NewFactory` 함수는
`connector.Factory`를 인스턴스화하여 반환하며, 다음 매개변수가 필요하다.

- `component.Type`: 동일한 유형의 모든 컬렉터 구성 요소 사이에서 커넥터를
  식별하는 고유한 문자열 식별자이다. 이 문자열은 커넥터를 가리키는 이름 역할도
  한다.
- `component.CreateDefaultConfigFunc`: 커넥터의 기본 `component.Config`
  인스턴스를 반환하는 함수에 대한 참조이다.
- `...FactoryOption`: `connector.FactoryOptions`의 슬라이스(slice)로, 커넥터가
  처리할 수 있는 시그널 유형을 결정한다.

1.  factory.go 파일을 만들고, 커넥터를 식별할 고유한 문자열을 전역 상수(global
    constant)로 정의한다.

    ```go
    const defaultVal string = "request.n"

    // Type is the component type name for this connector
    var Type = component.MustNewType("example")
    ```

2.  기본 설정 함수를 만든다. 이는 커넥터 객체를 기본값으로 초기화하는 방식이다.

    ```go
    func createDefaultConfig() component.Config {
        return &Config{
            AttributeName: defaultVal,
        }
    }
    ```

3.  작업할 커넥터의 유형을 정의한다. 이는 팩토리 옵션(factory option)으로
    전달된다. 커넥터는 서로 다르거나 비슷한 유형의 파이프라인을 연결할 수 있다.
    커넥터의 익스포터 쪽 끝과 리시버 쪽 끝의 유형을 각각 정의해야 한다.
    트레이스를 내보내고 메트릭을 받는 커넥터는 커넥터 구성 요소의 서로 다른 한
    가지 설정일 뿐이며, 이를 정의하는 순서가 중요하다. 트레이스를 내보내고
    메트릭을 받는 커넥터는 메트릭을 내보내고 트레이스를 받는 커넥터와 같지 않다.

    ```go
    // createTracesToMetricsConnector defines the consumer type of the connector
    // We want to consume traces and export metrics, therefore, define nextConsumer as metrics, since consumer is the next component in the pipeline
    func createTracesToMetricsConnector(ctx context.Context, params connector.Settings, cfg component.Config, nextConsumer consumer.Metrics) (connector.Traces, error) {
        return newConnector(params.Logger, cfg, nextConsumer)
    }
    ```

    `createTracesToMetricsConnector`는 컨슈머(consumer) 구성 요소, 즉 커넥터가
    데이터를 전달한 뒤 그 데이터를 받아들이는 다음 구성 요소를 정의하여 커넥터
    구성 요소를 추가로 초기화하는 함수이다. 커넥터가 여기서처럼 하나의 정해진
    유형 조합에만 국한되지 않는다는 점에 유의한다. 예를 들어 count 커넥터는
    트레이스에서 메트릭으로, 로그에서 메트릭으로, 메트릭에서 메트릭으로 변환하는
    이런 함수를 여러 개 정의한다.

    `createTracesToMetricsConnector`의 매개변수:
    - `context.Context`: 트레이스 리시버가 실행 컨텍스트(execution context)를
      올바르게 관리할 수 있도록 하는 컬렉터의 `context.Context`에 대한 참조이다.
    - `connector.CreateSettings`: 리시버가 생성되는 컬렉터의 일부 설정에 대한
      참조이다.
    - `component.Config`: 팩토리가 컬렉터 설정에서 자신의 설정을 올바르게 읽을
      수 있도록, 컬렉터가 팩토리에 전달하는 리시버 설정에 대한 참조이다.
    - `consumer.Metrics`: 파이프라인에서 다음 컨슈머 유형, 즉 수신된 트레이스가
      향할 곳에 대한 참조이다. 이는 프로세서, 익스포터 또는 다른 커넥터일 수
      있다.

4.  커넥터(구성 요소)를 위한 커스텀 팩토리를 인스턴스화하는 `NewFactory` 함수를
    작성한다.

    ```go
    // NewFactory creates a factory for example connector.
    func NewFactory() connector.Factory {
        // OpenTelemetry connector factory to make a factory for connectors
        return connector.NewFactory(
            Type,
            createDefaultConfig,
            connector.WithTracesToMetrics(createTracesToMetricsConnector, component.StabilityLevelAlpha))
    }
    ```

    커넥터는 여러 개의 정해진 데이터 유형 조합을 지원할 수 있다는 점에 유의한다.

완성되면 `factory.go`는 다음과 같다.

```go
package exampleconnector

import (
	"context"

	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/connector"
	"go.opentelemetry.io/collector/consumer"
)

const defaultVal string = "request.n"

// Type is the component type name for this connector
var Type = component.MustNewType("example")

// NewFactory creates a factory for example connector.
func NewFactory() connector.Factory {
	// OpenTelemetry connector factory to make a factory for connectors
	return connector.NewFactory(
		Type,
		createDefaultConfig,
		connector.WithTracesToMetrics(createTracesToMetricsConnector, component.StabilityLevelAlpha))
}

func createDefaultConfig() component.Config {
	return &Config{
		AttributeName: defaultVal,
	}
}

// createTracesToMetricsConnector defines the consumer type of the connector
// We want to consume traces and export metrics, therefore, define nextConsumer as metrics, since consumer is the next component in the pipeline
func createTracesToMetricsConnector(ctx context.Context, params connector.Settings, cfg component.Config, nextConsumer consumer.Metrics) (connector.Traces, error) {
	return newConnector(params.Logger, cfg, nextConsumer)
}
```

## 트레이스 커넥터 구현하기 {#implementing-the-trace-connector}

`connector.go` 파일에서 구성 요소의 유형에 특화된 인터페이스(interface)의
메서드(method)를 구현한다. 이 튜토리얼에서는 Traces 커넥터를 구현할 것이므로
`baseConsumer`, `Traces`, `component.Component` 인터페이스를 구현해야 한다.

1.  원하는 매개변수로 커넥터 구조체를 정의한다.

    ```go
    // schema for connector
    type connectorImp struct {
        config          Config
        metricsConsumer consumer.Metrics
        logger          *zap.Logger
        // Include these parameters if a specific implementation for the Start and Shutdown function are not needed
        component.StartFunc
        component.ShutdownFunc
    }
    ```

2.  커넥터를 생성하는 `newConnector` 함수를 정의한다.

    ```go
    // newConnector is a function to create a new connector
    func newConnector(logger *zap.Logger, config component.Config, nextConsumer consumer.Metrics) (*connectorImp, error) {
        logger.Info("Building exampleconnector connector")
        cfg := config.(*Config)

        return &connectorImp{
            config:          *cfg,
            logger:          logger,
            metricsConsumer: nextConsumer,
        }, nil
    }
    ```

    `newConnector` 함수는 커넥터의 인스턴스를 생성하는 팩토리 함수(factory
    function)이다.

3.  인터페이스를 올바르게 구현하기 위해 `Capabilities` 메서드를 구현한다.

    ```go
    // Capabilities implements the consumer interface.
    func (c *connectorImp) Capabilities() consumer.Capabilities {
        return consumer.Capabilities{MutatesData: false}
    }
    ```

    `Capabilities` 메서드를 구현하여 커넥터가 컨슈머(consumer) 유형임을
    보장한다. 이 메서드는 구성 요소가 데이터를 변경(mutate)할 수 있는지 여부 등
    구성 요소의 기능을 정의한다. `MutatesData`가 true로 설정되어 있으면, 이는
    커넥터가 전달받은 데이터 구조를 변경한다는 것을 나타낸다.

4.  텔레메트리 데이터를 소비(consume)하기 위해 `Consumer` 메서드를 구현한다.

    ```go
    // ConsumeTraces method is called for each instance of a trace sent to the connector
    func (c *connectorImp) ConsumeTraces(ctx context.Context, td ptrace.Traces) error {
        // loop through the levels of spans of the one trace consumed
        for i := 0; i < td.ResourceSpans().Len(); i++ {
            resourceSpan := td.ResourceSpans().At(i)

            for j := 0; j < resourceSpan.ScopeSpans().Len(); j++ {
                scopeSpan := resourceSpan.ScopeSpans().At(j)

                for k := 0; k < scopeSpan.Spans().Len(); k++ {
                    span := scopeSpan.Spans().At(k)
                    attrs := span.Attributes()
                    if _, ok := attrs.Get(c.config.AttributeName); ok {
                        // create metric only if span of trace had the specific attribute
                        metrics := pmetric.NewMetrics()
                        return c.metricsConsumer.ConsumeMetrics(ctx, metrics)
                    }
                }
            }
        }
        return nil
    }
    ```

5.  선택 사항: 특정 구현이 필요한 경우에만 인터페이스를 올바르게 구현하기 위해
    `Start`와 `Shutdown` 메서드를 구현한다. 그렇지 않다면 정의한 커넥터 구조체에
    `component.StartFunc`와 `component.ShutdownFunc`를 포함하는 것으로 충분하다.

완성된 커넥터 파일은 다음과 같다.

> exampleconnector/connector.go

```go
package exampleconnector

import (
	"context"

	"go.uber.org/zap"

	"go.opentelemetry.io/collector/component"
	"go.opentelemetry.io/collector/consumer"
	"go.opentelemetry.io/collector/pdata/pmetric"
	"go.opentelemetry.io/collector/pdata/ptrace"
)

// schema for connector
type connectorImp struct {
	config          Config
	metricsConsumer consumer.Metrics
	logger          *zap.Logger
	// Include these parameters if a specific implementation for the Start and Shutdown function are not needed
	component.StartFunc
	component.ShutdownFunc
}

// newConnector is a function to create a new connector
func newConnector(logger *zap.Logger, config component.Config, nextConsumer consumer.Metrics) (*connectorImp, error) {
	logger.Info("Building exampleconnector connector")
	cfg := config.(*Config)

	return &connectorImp{
		config:          *cfg,
		logger:          logger,
		metricsConsumer: nextConsumer,
	}, nil
}

// Capabilities implements the consumer interface.
func (c *connectorImp) Capabilities() consumer.Capabilities {
	return consumer.Capabilities{MutatesData: false}
}

// ConsumeTraces method is called for each instance of a trace sent to the connector
func (c *connectorImp) ConsumeTraces(ctx context.Context, td ptrace.Traces) error {
	// loop through the levels of spans of the one trace consumed
	for i := 0; i < td.ResourceSpans().Len(); i++ {
		resourceSpan := td.ResourceSpans().At(i)

		for j := 0; j < resourceSpan.ScopeSpans().Len(); j++ {
			scopeSpan := resourceSpan.ScopeSpans().At(j)

			for k := 0; k < scopeSpan.Spans().Len(); k++ {
				span := scopeSpan.Spans().At(k)
				attrs := span.Attributes()
				if _, ok := attrs.Get(c.config.AttributeName); ok {
					// create metric only if span of trace had the specific attribute
					metrics := pmetric.NewMetrics()
					return c.metricsConsumer.ConsumeMetrics(ctx, metrics)
				}
			}
		}
	}
	return nil
}
```

## 구성 요소 사용하기 {#using-the-component}

### OpenTelemetry Collector Builder 사용 요약: {#summary-of-using-opentelemetry-collector-builder}

[OpenTelemetry Collector Builder](/docs/collector/extend/ocb/)를 사용해 코드를
빌드하고 실행할 수 있다. 컬렉터 빌더(builder)는 자신만의 오픈텔레메트리 컬렉터
바이너리(binary)를 빌드할 수 있게 해주는 도구이다. 필요에 따라 구성 요소(리시버,
프로세서, 커넥터, 익스포터)를 추가하거나 제거할 수 있다.

1.  OpenTelemetry Collector Builder의 [설치 안내](/docs/collector/extend/ocb/)를
    따른다.

2.  설정 파일 작성하기:

    설치가 끝나면, 다음 단계는 `builder-config.yaml` 설정 파일을 만드는 것이다.
    이 파일은 커스텀 바이너리에 포함할 컬렉터 구성 요소를 정의한다.

    다음은 새로운 커넥터 구성 요소를 포함한, 사용할 수 있는 설정 파일의
    예시이다.

    ```yaml
    dist:
      name: otelcol-dev-bin
      description: Basic OpenTelemetry collector distribution for Developers
      output_path: ./otelcol-dev

    exporters:
      - gomod: go.opentelemetry.io/collector/exporter/debugexporter v0.129.0

    receivers:
      - gomod: go.opentelemetry.io/collector/receiver/otlpreceiver v0.129.0

    # Not used in this tutorial, but can be added if needed for your use case
    # processors:

    connectors:
      - gomod: github.com/gord02/exampleconnector v0.129.0

    replaces:
      # a list of "replaces" directives that will be part of the resulting go.mod

      # This replace statement is necessary since the newly added component is not found/published to GitHub yet. Replace references to GitHub path with the local path
      - github.com/gord02/exampleconnector =>
        [PATH-TO-COMPONENT-CODE]/exampleconnector
    ```

    새로 만든 구성 요소가 아직 GitHub에 게시되지 않았으므로 replace
    문(statement)을 포함해야 한다. 구성 요소의 GitHub 경로에 대한 참조는 코드의
    로컬 경로로 대체해야 한다.

    Go에서의 replace에 대한 자세한 내용은
    [Go mod file Replace](https://go.dev/ref/mod#go-mod-file-replace)에서 확인할
    수 있다.

3.  컬렉터 바이너리 빌드하기:

    포함된 커넥터 구성 요소가 상세히 기술된 빌더 설정 파일을 전달하면서 빌더를
    실행하면 커스텀 컬렉터 바이너리가 빌드된다.

    ```sh
    ./ocb --config [PATH-TO-CONFIG]/builder-config.yaml
    ```

    이렇게 하면 설정 파일에 지정된 출력 경로(output path) 디렉터리에 컬렉터
    바이너리가 생성된다.

    빌드가 성공하면 다음과 비슷한 출력이 표시된다.

    ```sh
    ./ocb --config builder-config.yaml
    2025-07-15T22:10:10.351+0900    INFO    internal/command.go:99  OpenTelemetry Collector Builder {"version": "0.129.0"}
    2025-07-15T22:10:10.352+0900    INFO    internal/command.go:104 Using config file       {"path": "builder-config.yaml"}
    2025-07-15T22:10:10.353+0900    INFO    builder/config.go:160   Using go        {"go-executable": "/opt/homebrew/Cellar/go@1.23/1.23.6/bin/go"}
    2025-07-15T22:10:10.354+0900    INFO    builder/main.go:99      Sources created {"path": "./otelcol-dev"}
    2025-07-15T22:10:10.516+0900    INFO    builder/main.go:201     Getting go modules
    2025-07-15T22:10:10.554+0900    INFO    builder/main.go:110     Compiling
    2025-07-15T22:10:13.369+0900    INFO    builder/main.go:140     Compiled        {"binary": "./otelcol-dev/otelcol-dev-bin"}
    ```

4.  컬렉터 바이너리 실행하기:

    이제 3단계 출력의 바이너리 경로(예:
    `{"binary": "./otelcol-dev/otelcol-dev-bin"}`)를 사용해 커스텀 컬렉터
    바이너리를 실행할 수 있다.

    ```sh
    ./otelcol-dev/otelcol-dev-bin --config [PATH-TO-CONFIG]/config.yaml
    ```

    출력 경로 이름과 dist의 이름은 `build-config.yaml`에 상세히 기술되어 있다.

## 커넥터 테스트하기 {#testing-your-connector}

예제 커넥터를 빌드했으니, 이제 단위 테스트(unit test)로 그 기능을 검증해보자. Go
단위 테스트는 더 나은 커버리지(coverage)를 제공하며 유지 관리하기도 더 쉽다.

### 단위 테스트 작성하기 {#writing-unit-tests}

커넥터 디렉터리에 테스트 파일 `connector_test.go`를 만든다.

> exampleconnector/connector_test.go

```go
package exampleconnector

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/vibeus/opentelemetry-collector/confmap/xconfmap"
	"go.opentelemetry.io/collector/consumer/consumertest"
	"go.opentelemetry.io/collector/pdata/ptrace"
	"go.uber.org/zap"
)

func TestConsumeTraces(t *testing.T) {
	// Create a test consumer that captures metrics
	metricsConsumer := &consumertest.MetricsSink{}

	// Create connector with test configuration
	cfg := &Config{
		AttributeName: "request.n",
	}

	connector, err := newConnector(zap.NewNop(), cfg, metricsConsumer)
	require.NoError(t, err)

	ctx := context.Background()

	t.Run("span with target attribute generates metric", func(t *testing.T) {
		// Reset the consumer
		metricsConsumer.Reset()

		// Create trace data with target attribute
		traces := ptrace.NewTraces()
		resourceSpan := traces.ResourceSpans().AppendEmpty()
		scopeSpan := resourceSpan.ScopeSpans().AppendEmpty()
		span := scopeSpan.Spans().AppendEmpty()

		// Add the target attribute
		span.Attributes().PutStr("request.n", "test-value")
		span.Attributes().PutStr("http.method", "GET")

		// Consume the traces
		err := connector.ConsumeTraces(ctx, traces)
		require.NoError(t, err)

		// Verify metric was generated
		assert.Equal(t, 1, len(metricsConsumer.AllMetrics()))
	})

	t.Run("span without target attribute does not generate metric", func(t *testing.T) {
		// Reset the consumer
		metricsConsumer.Reset()

		// Create trace data without target attribute
		traces := ptrace.NewTraces()
		resourceSpan := traces.ResourceSpans().AppendEmpty()
		scopeSpan := resourceSpan.ScopeSpans().AppendEmpty()
		span := scopeSpan.Spans().AppendEmpty()

		// Add other attributes but not the target one
		span.Attributes().PutStr("http.method", "POST")
		span.Attributes().PutStr("user.id", "12345")

		// Consume the traces
		err := connector.ConsumeTraces(ctx, traces)
		require.NoError(t, err)

		// Verify no metric was generated
		assert.Equal(t, 0, len(metricsConsumer.AllMetrics()))
	})

	t.Run("multiple spans with mixed attributes", func(t *testing.T) {
		// Reset the consumer
		metricsConsumer.Reset()

		// Create trace data with multiple spans
		traces := ptrace.NewTraces()
		resourceSpan := traces.ResourceSpans().AppendEmpty()
		scopeSpan := resourceSpan.ScopeSpans().AppendEmpty()

		// First span with target attribute
		span1 := scopeSpan.Spans().AppendEmpty()
		span1.Attributes().PutStr("request.n", "value1")

		// Second span without target attribute
		span2 := scopeSpan.Spans().AppendEmpty()
		span2.Attributes().PutStr("other.attr", "value2")

		// Consume the traces
		err := connector.ConsumeTraces(ctx, traces)
		require.NoError(t, err)

		// Should generate exactly one metric (from first span only)
		assert.Equal(t, 1, len(metricsConsumer.AllMetrics()))
	})
}

func TestConnectorCapabilities(t *testing.T) {
	connector := &connectorImp{}
	capabilities := connector.Capabilities()
	assert.False(t, capabilities.MutatesData)
}

func TestCreateDefaultConfig(t *testing.T) {
	cfg := createDefaultConfig()
	assert.NotNil(t, cfg)

	exampleConfig := cfg.(*Config)
	assert.Equal(t, "request.n", exampleConfig.AttributeName)
}

func TestConfigValidation(t *testing.T) {
	t.Run("valid config", func(t *testing.T) {
		cfg := &Config{
			AttributeName: "test.attribute",
		}
		err := xconfmap.Validate(cfg)
		assert.NoError(t, err)
	})

	t.Run("invalid config - empty attribute name", func(t *testing.T) {
		cfg := &Config{
			AttributeName: "",
		}
		err := xconfmap.Validate(cfg)
		assert.Error(t, err)
		assert.Contains(t, err.Error(), "attribute_name must not be empty")
	})
}
```

### 테스트 실행하기 {#running-the-tests}

1. **`go.mod`에 테스트 의존성 추가하기:**

   ```sh
   go mod tidy
   ```

2. **테스트 실행하기:**

   ```sh
   go test -cover -v ./...
   ```

### 예상되는 테스트 출력 {#expected-test-output}

테스트가 성공적으로 실행되면 다음과 비슷한 출력이 표시된다.

```sh
go test -cover -v ./...
=== RUN   TestConsumeTraces
=== RUN   TestConsumeTraces/span_with_target_attribute_generates_metric
=== RUN   TestConsumeTraces/span_without_target_attribute_does_not_generate_metric
=== RUN   TestConsumeTraces/multiple_spans_with_mixed_attributes
--- PASS: TestConsumeTraces (0.00s)
    --- PASS: TestConsumeTraces/span_with_target_attribute_generates_metric (0.00s)
    --- PASS: TestConsumeTraces/span_without_target_attribute_does_not_generate_metric (0.00s)
    --- PASS: TestConsumeTraces/multiple_spans_with_mixed_attributes (0.00s)
=== RUN   TestConnectorCapabilities
--- PASS: TestConnectorCapabilities (0.00s)
=== RUN   TestCreateDefaultConfig
--- PASS: TestCreateDefaultConfig (0.00s)
=== RUN   TestConfigValidation
=== RUN   TestConfigValidation/valid_config
=== RUN   TestConfigValidation/invalid_config_-_empty_attribute_name
--- PASS: TestConfigValidation (0.00s)
    --- PASS: TestConfigValidation/valid_config (0.00s)
    --- PASS: TestConfigValidation/invalid_config_-_empty_attribute_name (0.00s)
PASS
coverage: 90.5% of statements
ok      github.com/gord02/exampleconnector      0.501s  coverage: 90.5% of statements
```

이 단위 테스트는 커넥터 기능에 대한 포괄적인 커버리지를 제공하며, 오픈텔레메트리
컬렉터 생태계(ecosystem)에서 구성 요소의 동작을 검증하는 데 권장되는 접근
방식이다.

OpenTelemetry Collector Builder에 대한 추가 자료:

- [커스텀 컬렉터 빌드하기](/docs/collector/extend/ocb/)
- [OpenTelemetry Collector Builder README](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder)
- [Connected Observability Pipelines in the OpenTelemetry Collector by Dan Jaglowski](https://www.youtube.com/watch?v=uPpZ23iu6kI)
- [Connector README](https://github.com/open-telemetry/opentelemetry-collector/blob/main/connector/README.md)
