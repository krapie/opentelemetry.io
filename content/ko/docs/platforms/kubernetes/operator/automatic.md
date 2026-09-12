---
title: 자동 계측 주입
linkTitle: 자동 계측
weight: 11
description:
  오픈텔레메트리(OpenTelemetry) 오퍼레이터를 사용한 자동 계측 구현체이다.
# prettier-ignore
cSpell:ignore: Dockerfiles GRPCNETCLIENT k8sattributesprocessor otelinst otlpreceiver REDISCALA replicaset statefulset
default_lang_commit: 48d3ff356dc39a3b1323637f3163d435dc751228
---

오픈텔레메트리 오퍼레이터는 .NET, Java, Node.js, Python, Go 서비스를 위한 자동
계측 라이브러리의 주입과 구성을 지원한다.

## 설치 {#installation}

먼저
[오픈텔레메트리 오퍼레이터](https://github.com/open-telemetry/opentelemetry-operator)를
클러스터에 설치한다.

[오퍼레이터 릴리스 매니페스트](https://github.com/open-telemetry/opentelemetry-operator#getting-started),
[오퍼레이터 Helm 차트](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator#opentelemetry-operator-helm-chart),
또는 [Operator Hub](https://operatorhub.io/operator/opentelemetry-operator)를
사용해 설치할 수 있다.

대부분의 경우 [cert-manager](https://cert-manager.io/docs/installation/)를
설치해야 한다. Helm 차트를 사용하는 경우 자체 서명된 인증서를 대신 생성하는
옵션이 있다.

> Go 자동 계측을 사용하려면 기능 게이트(feature gate)를 활성화해야 한다. 자세한
> 내용은 [계측 기능 제어](#controlling-instrumentation-capabilities)를 참고한다.

## 오픈텔레메트리 컬렉터 생성(선택 사항) {#create-an-opentelemetry-collector-optional}

컨테이너의 텔레메트리를 백엔드로 직접 보내는 대신
[오픈텔레메트리 컬렉터](/docs/platforms/kubernetes/collector/)로 보내는 것이
모범 사례이다. 컬렉터는 시크릿 관리를 단순화하는 데 도움을 주고, (재시도가
필요한 경우와 같은) 데이터 내보내기 문제를 애플리케이션으로부터 분리해주며,
[k8sattributesprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor)
컴포넌트와 같은 기능으로 텔레메트리에 추가 데이터를 덧붙일 수 있게 해준다.
컬렉터를 사용하지 않기로 했다면 다음 섹션으로 건너뛰어도 된다.

오퍼레이터는
[오픈텔레메트리 컬렉터를 위한 CRD(Custom Resource Definition)](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/opentelemetrycollectors.md)를
제공하며, 이를 사용해 오퍼레이터가 관리하는 컬렉터 인스턴스를 생성한다. 다음
예제는 컬렉터를 (기본값인) 디플로이먼트로 배포하지만, 이 밖에도 사용할 수 있는
다른
[배포 모드](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/collector/deployment-modes.md)가
있다.

`Deployment` 모드를 사용하면 오퍼레이터는 컬렉터와 상호 작용하는 데 사용할 수
있는 서비스도 함께 생성한다. 이 서비스의 이름은 `OpenTelemetryCollector`
리소스의 이름 뒤에 `-collector`를 붙인 것이다. 이 예제에서는 `demo-collector`가
된다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: demo
spec:
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
    exporters:
      debug:
        verbosity: basic

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
        logs:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
EOF
```

위 명령을 실행하면 컬렉터가 배포되며, 파드의 자동 계측을 위한 엔드포인트로
사용할 수 있다.

## 자동 계측 구성 {#configure-automatic-instrumentation}

자동 계측을 관리할 수 있으려면, 어떤 파드를 계측할지와 해당 파드에 어떤 자동
계측을 사용할지를 오퍼레이터가 알 수 있도록 구성해야 한다. 이는
[Instrumentation CRD](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/instrumentations.md)를
통해 이루어진다.

`Instrumentation` 리소스를 올바르게 생성하는 것은 자동 계측이 동작하도록 만드는
데 매우 중요하다. 모든 엔드포인트와 환경 변수가 올바른지 확인해야 자동 계측이
제대로 동작한다.

### .NET {#net}

다음 명령은 .NET 서비스를 계측하도록 특별히 구성된 기본 `Instrumentation`
리소스를 생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

.NET 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로
`http/protobuf` 프로토콜을 사용하는 `otlp`를 사용한다. 즉, 구성된 엔드포인트는
`http/protobuf`를 통해 OTLP를 수신할 수 있어야 한다. 그래서 이 예제에서는
`http://demo-collector:4318`을 사용하며, 이는 이전 단계에서 생성한 컬렉터의
`otlpreceiver`의 `http` 포트로 연결된다.

#### 자동 계측 제외 {#dotnet-excluding-auto-instrumentation}

기본적으로 .NET 자동 계측에는
[많은 계측 라이브러리](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/config.md#instrumentations)가
함께 제공된다. 덕분에 계측이 쉬워지지만, 그만큼 너무 많거나 원하지 않는 데이터가
발생할 수 있다. 사용하고 싶지 않은 라이브러리가 있다면
`OTEL_DOTNET_AUTO_[SIGNAL]_[NAME]_INSTRUMENTATION_ENABLED=false`를 설정할 수
있다. 여기서 `[SIGNAL]`은 시그널의 유형이고 `[NAME]`은 대소문자를 구분하는
라이브러리 이름이다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  dotnet:
    env:
      - name: OTEL_DOTNET_AUTO_TRACES_GRPCNETCLIENT_INSTRUMENTATION_ENABLED
        value: false
      - name: OTEL_DOTNET_AUTO_METRICS_PROCESS_INSTRUMENTATION_ENABLED
        value: false
```

.NET 자동 계측은 .NET
[런타임 식별자(Runtime Identifier, RID)](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog)를
설정하기 위한 런타임 어노테이션도 지원한다. 현재 `linux-x64`(기본값)와
`linux-musl-x64`가 지원된다.

```bash
instrumentation.opentelemetry.io/inject-dotnet: "true"
instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-x64"   # default, can be omitted
instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-musl-x64"  # for musl-based images
```

> **참고:** 기본적으로 오퍼레이터는
> `OTEL_DOTNET_AUTO_TRACES_ENABLED_INSTRUMENTATIONS`를 사용 중인
> `opentelemetry-dotnet-instrumentation` 릴리스가 지원하는 모든 계측으로
> 설정한다(예: `AspNet,HttpClient,SqlClient`). 이 값은 환경 변수를 직접 구성해
> 재정의할 수 있다.

#### 더 알아보기 {#dotnet-learn-more}

자세한 내용은 [.NET 자동 계측 문서](/docs/zero-code/dotnet/)를 참고한다.

### Deno {#deno}

다음 명령은 [Deno](https://deno.com) 서비스를 계측하도록 구성된 기본
`Instrumentation` 리소스를 생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  env:
    - name: OTEL_DENO
      value: 'true'
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
EOF
```

Deno 프로세스는 `OTEL_DENO=true` 환경 변수와 함께 시작되면 구성된 엔드포인트로
텔레메트리 데이터를 자동으로 내보낸다. 그래서 이 예제에서는 `Instrumentation`
리소스의 `env` 필드에 이 환경 변수를 지정하며, 이렇게 하면 이 `Instrumentation`
리소스로 환경 변수가 주입되는 모든 서비스에 이 값이 설정된다.

Deno 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로 `http/proto`
프로토콜을 사용하는 `otlp`를 사용한다. 즉, 구성된 엔드포인트는 `http/proto`를
통해 OTLP를 수신할 수 있어야 한다. 그래서 이 예제에서는
`http://demo-collector:4318`을 사용하며, 이는 이전 단계에서 생성한 컬렉터의
`otlpreceiver`의 `http/proto` 포트로 연결된다.

> [!NOTE]
>
> [Deno의 오픈텔레메트리 통합][deno-docs]은 아직 안정적이지 않다. 그 결과 Deno로
> 계측하려는 모든 워크로드는 Deno 프로세스를 시작할 때 `--unstable-otel`
> 플래그를 설정해야 한다.
>
> [deno-docs]: https://docs.deno.com/runtime/fundamentals/open_telemetry/

#### 구성 옵션 {#deno-configuration-options}

기본적으로 Deno 오픈텔레메트리 통합은 `console.log()` 출력을
[로그](/docs/concepts/signals/logs/)로 내보내는 동시에, stdout/stderr로도 계속
출력한다. 다음과 같은 대체 동작을 구성할 수 있다.

- `OTEL_DENO_CONSOLE=replace`: `console.log()` 출력만 로그로 내보내고,
  stdout/stderr로는 출력하지 않는다.
- `OTEL_DENO_CONSOLE=ignore`: `console.log()` 출력을 로그로 내보내지 않고,
  stdout/stderr로는 출력한다.

#### 더 알아보기 {#deno-learn-more}

자세한 내용은 Deno의 [오픈텔레메트리 통합][deno-otel-docs] 문서를 참고한다.

[deno-otel-docs]: https://docs.deno.com/runtime/fundamentals/open_telemetry/

### Go {#go}

다음 명령은 Go 서비스를 계측하도록 특별히 구성된 기본 `Instrumentation` 리소스를
생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Go 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로 `http/protobuf`
프로토콜을 사용하는 `otlp`를 사용한다. 즉, 구성된 엔드포인트는 `http/protobuf`를
통해 OTLP를 수신할 수 있어야 한다. 그래서 이 예제에서는
`http://demo-collector:4318`을 사용하며, 이는 이전 단계에서 생성한 컬렉터의
`http/protobuf` 포트로 연결된다.

Go 자동 계측은 특정 계측을 비활성화하는 기능을 지원하지 않는다.
[자세한 내용은 Go 자동 계측 저장소를 참고한다.](https://github.com/open-telemetry/opentelemetry-go-instrumentation)

### Java {#java}

다음 명령은 Java 서비스를 계측하도록 구성된 기본 `Instrumentation` 리소스를
생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Java 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로
`http/protobuf` 프로토콜을 사용하는 `otlp`를 사용한다. 즉, 구성된 엔드포인트는
`protobuf` 페이로드를 통해 `http`로 OTLP를 수신할 수 있어야 한다. 그래서 이
예제에서는 `http://demo-collector:4318`을 사용하며, 이는 이전 단계에서 생성한
컬렉터의 otlpreceiver의 `http` 포트로 연결된다.

#### 자동 계측 제외 {#java-excluding-auto-instrumentation}

기본적으로 Java 자동 계측에는
[많은 계측 라이브러리](/docs/zero-code/java/agent/getting-started/#supported-libraries-frameworks-application-services-and-jvms)가
함께 제공된다. 덕분에 계측이 쉬워지지만, 그만큼 너무 많거나 원하지 않는 데이터가
발생할 수 있다. 사용하고 싶지 않은 라이브러리가 있다면
`OTEL_INSTRUMENTATION_[NAME]_ENABLED=false`를 설정할 수 있다. 여기서 `[NAME]`은
라이브러리의 이름이다. 사용하려는 라이브러리를 정확히 알고 있다면,
`OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false`를 설정해 기본 라이브러리를
모두 비활성화한 다음 `OTEL_INSTRUMENTATION_[NAME]_ENABLED=true`를 설정할 수
있다. 여기서 `[NAME]`은 라이브러리의 이름이다. 자세한 내용은
[특정 계측 억제하기](/docs/zero-code/java/agent/disable/)를 참고한다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  java:
    env:
      - name: OTEL_INSTRUMENTATION_KAFKA_ENABLED
        value: false
      - name: OTEL_INSTRUMENTATION_REDISCALA_ENABLED
        value: false
```

#### 더 알아보기 {#java-learn-more}

자세한 내용은 [Java 에이전트 구성](/docs/zero-code/java/agent/configuration/)을
참고한다.

### Node.js {#nodejs}

다음 명령은 Node.js 서비스를 계측하도록 구성된 기본 `Instrumentation` 리소스를
생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Node.js 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로 `grpc`
프로토콜을 사용하는 `otlp`를 사용한다. 즉, 구성된 엔드포인트는 `grpc`를 통해
OTLP를 수신할 수 있어야 한다. 그래서 이 예제에서는
`http://demo-collector:4317`을 사용하며, 이는 이전 단계에서 생성한 컬렉터의
`otlpreceiver`의 `grpc` 포트로 연결된다.

#### 계측 라이브러리 제외 {#js-excluding-instrumentation-libraries}

기본적으로 Node.js 제로 코드 계측은 모든 계측 라이브러리가 활성화되어 있다.

특정 계측 라이브러리만 활성화하려면
[Node.js 제로 코드 계측 문서](/docs/zero-code/js/configuration/#excluding-instrumentation-libraries)에
설명된 대로 `OTEL_NODE_ENABLED_INSTRUMENTATIONS` 환경 변수를 사용할 수 있다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
# ... other fields skipped from this example
spec:
  # ... other fields skipped from this example
  nodejs:
    env:
      - name: OTEL_NODE_ENABLED_INSTRUMENTATIONS
        value: http,nestjs-core # comma-separated list of the instrumentation package names without the `@opentelemetry/instrumentation-` prefix.
```

기본 라이브러리는 모두 유지하고 특정 계측 라이브러리만 비활성화하려면
`OTEL_NODE_DISABLED_INSTRUMENTATIONS` 환경 변수를 사용할 수 있다. 자세한 내용은
[계측 라이브러리 제외](/docs/zero-code/js/configuration/#excluding-instrumentation-libraries)를
참고한다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
# ... other fields skipped from this example
spec:
  # ... other fields skipped from this example
  nodejs:
    env:
      - name: OTEL_NODE_DISABLED_INSTRUMENTATIONS
        value: fs,grpc # comma-separated list of the instrumentation package names without the `@opentelemetry/instrumentation-` prefix.
```

> [!NOTE]
>
> 두 환경 변수가 모두 설정되어 있으면 `OTEL_NODE_ENABLED_INSTRUMENTATIONS`가
> 먼저 적용된 다음, 그 목록에 `OTEL_NODE_DISABLED_INSTRUMENTATIONS`가 적용된다.
> 따라서 같은 계측이 두 목록에 모두 포함되어 있으면 해당 계측은 비활성화된다.

#### 더 알아보기 {#js-learn-more}

자세한 내용은 [Node.js 자동 계측](/docs/languages/js/libraries/#registration)을
참고한다.

### Python {#python}

다음 명령은 Python 서비스를 계측하도록 특별히 구성된 기본 `Instrumentation`
리소스를 생성한다.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Python 서비스를 자동 계측하는 `Instrumentation` 리소스는 기본적으로
`http/protobuf` 프로토콜을 사용하는 `otlp`를 사용한다(현재 gRPC는 지원하지
않는다). 즉, 구성된 엔드포인트는 `http/protobuf`를 통해 OTLP를 수신할 수 있어야
한다. 그래서 이 예제에서는 `http://demo-collector:4318`을 사용하며, 이는 이전
단계에서 생성한 컬렉터의 `otlpreceiver`의 `http` 포트로 연결된다.

> 오퍼레이터 v0.108.0부터는 `Instrumentation` 리소스가 Python 서비스에 대해
> `OTEL_EXPORTER_OTLP_PROTOCOL`을 `http/protobuf`로 자동으로 설정한다. 그보다
> 이전 버전의 오퍼레이터를 사용한다면 이 환경 변수를 `http/protobuf`로 반드시
> 설정해야 하며, 그렇지 않으면 Python 자동 계측이 동작하지 않는다.

#### Python 로그 자동 계측 {#auto-instrumenting-python-logs}

기본적으로 Python 로그 자동 계측은 비활성화되어 있다. 이 기능을 활성화하려면
`OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED` 환경 변수를 다음과 같이
설정해야 한다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
  namespace: application
spec:
  exporter:
    endpoint: http://demo-collector:4318
  env:
  propagators:
    - tracecontext
    - baggage
  python:
    env:
      - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
        value: 'true'
```

> 오퍼레이터 v0.111.0부터는 `OTEL_LOGS_EXPORTER`를 `otlp`로 설정할 필요가 더
> 이상 없다.

#### 자동 계측 제외 {#python-excluding-auto-instrumentation}

기본적으로 Python 자동 계측에는
[많은 계측 라이브러리](https://github.com/open-telemetry/opentelemetry-operator/blob/main/autoinstrumentation/python/requirements.txt)가
함께 제공된다. 덕분에 계측이 쉬워지지만, 그만큼 너무 많거나 원하지 않는 데이터가
발생할 수 있다. 계측하고 싶지 않은 패키지가 있다면
`OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` 환경 변수를 설정할 수 있다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  python:
    env:
      - name: OTEL_PYTHON_DISABLED_INSTRUMENTATIONS
        value:
          <comma-separated list of package names to exclude from
          instrumentation>
```

자세한 내용은
[Python 에이전트 구성 문서](/docs/zero-code/python/configuration/#disabling-specific-instrumentations)를
참고한다.

#### 더 알아보기 {#python-learn-more}

Python 관련 특이 사항은
[Python 오픈텔레메트리 오퍼레이터 문서](/docs/zero-code/python/operator/#python-specific-topics)와
[Python 에이전트 구성 문서](/docs/zero-code/python/configuration/)를 참고한다.

---

이제 `Instrumentation` 오브젝트가 생성되었으므로, 클러스터는 서비스를 자동
계측하고 엔드포인트로 데이터를 보낼 수 있게 되었다. 하지만 오픈텔레메트리
오퍼레이터를 사용한 자동 계측은 옵트인(opt-in) 모델을 따른다. 자동 계측을
활성화하려면 디플로이먼트에 어노테이션을 추가해야 한다.

## 기존 디플로이먼트에 어노테이션 추가 {#add-annotations-to-existing-deployments}

마지막 단계는 서비스를 자동 계측에 옵트인시키는 것이다. 이는 서비스의
`spec.template.metadata.annotations`를 업데이트하여 언어별 어노테이션을
포함시킴으로써 이루어진다.

- .NET: `instrumentation.opentelemetry.io/inject-dotnet: "true"`
- Deno: `instrumentation.opentelemetry.io/inject-sdk: "true"`
- Go: `instrumentation.opentelemetry.io/inject-go: "true"`
- Java: `instrumentation.opentelemetry.io/inject-java: "true"`
- Node.js: `instrumentation.opentelemetry.io/inject-nodejs: "true"`
- Python: `instrumentation.opentelemetry.io/inject-python: "true"`

어노테이션에 사용할 수 있는 값은 다음과 같다.

- `"true"` - 현재 네임스페이스에서 기본 이름을 가진 `Instrumentation` 리소스를
  주입한다.
- `"my-instrumentation"` - 현재 네임스페이스에서 이름이 `"my-instrumentation"`인
  `Instrumentation` CR 인스턴스를 주입한다.
- `"my-other-namespace/my-instrumentation"` - 다른 네임스페이스인
  `"my-other-namespace"`에서 이름이 `"my-instrumentation"`인 `Instrumentation`
  CR 인스턴스를 주입한다.
- `"false"` - 주입하지 않는다.

또는 네임스페이스에 어노테이션을 추가할 수도 있으며, 이 경우 해당 네임스페이스의
모든 서비스가 자동 계측에 옵트인하게 된다. 자세한 내용은
[오퍼레이터 자동 계측 문서](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/auto-instrumentation/README.md)를
참고한다.

### Go 서비스 옵트인 {#opt-in-a-go-service}

다른 언어의 자동 계측과 달리, Go는 사이드카(sidecar)로 실행되는 eBPF 에이전트를
통해 동작한다. 옵트인하면 오퍼레이터가 이 사이드카를 파드에 주입한다. 위에서
언급한 `instrumentation.opentelemetry.io/inject-go` 어노테이션 외에도,
[`OTEL_GO_AUTO_TARGET_EXE` 환경 변수](https://github.com/open-telemetry/opentelemetry-go-instrumentation/blob/main/docs/how-it-works.md)에
값을 지정해야 한다.

이 환경 변수는 `instrumentation.opentelemetry.io/otel-go-auto-target-exe`
어노테이션을 통해 설정할 수 있다.

```yaml
instrumentation.opentelemetry.io/inject-go: 'true'
instrumentation.opentelemetry.io/otel-go-auto-target-exe: '/path/to/container/executable'
```

이 환경 변수는 `Instrumentation` 리소스를 통해서도 설정할 수 있으며, 이 경우
어노테이션이 우선한다. Go 자동 계측은 `OTEL_GO_AUTO_TARGET_EXE`가 설정되어
있어야 하므로, 어노테이션이나 `Instrumentation` 리소스를 통해 유효한 실행 파일
경로를 반드시 지정해야 한다. 이 값을 설정하지 않으면 계측 주입이 중단되고, 원래
파드는 변경되지 않은 상태로 남는다.

Go 자동 계측은 eBPF를 사용하므로 높은 권한도 필요하다. 옵트인하면 오퍼레이터가
주입하는 사이드카는 다음과 같은 권한을 필요로 한다.

```yaml
securityContext:
  privileged: true
  runAsUser: 0
```

### Python musl 기반 컨테이너 자동 계측 {#annotations-python-musl}

오퍼레이터 v0.113.0부터 Python 자동 계측은 glibc가 아닌 다른 C 라이브러리를
사용하는 이미지에서 실행할 수 있게 해주는 어노테이션도 지원한다.

```sh
# for Linux glibc based images, this is the default value and can be omitted
instrumentation.opentelemetry.io/otel-python-platform: "glibc"
# for Linux musl based images
instrumentation.opentelemetry.io/otel-python-platform: "musl"
```

## 다중 컨테이너 파드 {#multi-container-pods}

### 단일 계측 {#single-instrumentation}

별도로 지정하지 않으면, 계측은 파드 스펙에서 (init 컨테이너가 아니라
`.spec.containers`에 있는) 첫 번째 컨테이너에 대해 수행된다. 예를 들어 Istio
사이드카가 주입된 경우처럼, 어떤 컨테이너에 주입해야 하는지 지정해야 하는 경우도
있다.

`instrumentation.opentelemetry.io/container-names` 어노테이션을 사용하면
(`.spec.containers.name`이나 `.spec.initContainers.name`에 있는) 주입 대상
컨테이너 이름을 하나 이상 지정할 수 있다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multiple-containers
spec:
  selector:
    matchLabels:
      app: my-pod-with-multiple-containers
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multiple-containers
      annotations:
        instrumentation.opentelemetry.io/inject-java: 'true'
        instrumentation.opentelemetry.io/container-names: 'myapp,myapp2'
    spec:
      containers:
        - name: myapp
          image: myImage1
        - name: myapp2
          image: myImage2
        - name: myapp3
          image: myImage3
```

위 경우 `myapp`과 `myapp2` 컨테이너는 계측되지만 `myapp3`은 계측되지 않는다.

> **참고**: Go 자동 계측은 다중 컨테이너 파드를 지원하지 **않는다**. Go 자동
> 계측을 주입할 때는 첫 번째 컨테이너만 계측 대상이어야 한다.

### init 컨테이너 계측 {#instrumenting-init-containers}

init 컨테이너는 `container-names` 어노테이션에 이름을 포함시켜 계측할 수 있다.
init 컨테이너가 계측 대상이 되면, 오퍼레이터는 파드의 init 컨테이너 시퀀스에서
대상 init 컨테이너 **앞에** 계측용 init 컨테이너를 자동으로 삽입한다. 이렇게
하면 대상 init 컨테이너가 실행될 때 계측 에이전트 파일을 사용할 수 있게 된다.

init 컨테이너에 대해 지원되는 계측: Java, Python, Node.js, .NET, SDK 전용 주입.

init 컨테이너에 대해 지원되지 않는 항목: Go(다중 컨테이너 파드를 지원하지 않음),
Apache HTTPD, NGINX.

> **참고**: 쿠버네티스는 파드 스펙 내에서 `initContainers`와 `containers` 목록
> 전체에 걸쳐 컨테이너 이름이 고유하도록 보장하므로, 오퍼레이터는 컨테이너
> 이름이 init 컨테이너를 가리키는지 일반 컨테이너를 가리키는지 명확히 판단할 수
> 있다.

init 컨테이너와 일반 컨테이너를 모두 계측하는 예제:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-init-container
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        instrumentation.opentelemetry.io/inject-python: 'true'
        instrumentation.opentelemetry.io/container-names: 'my-init-job,myapp'
    spec:
      initContainers:
        - name: my-init-job
          image: my-python-init-image
      containers:
        - name: myapp
          image: my-python-app-image
```

이 예제에서는 (init 컨테이너인) `my-init-job`과 (일반 컨테이너인) `myapp` 모두
Python 자동 계측으로 계측된다.

### 다중 계측 {#multiple-instrumentations}

다중 계측은 `enable-multi-instrumentation` 기능 플래그가 `true`로 설정된
경우에만 동작한다. 활성화되면, 언어별 컨테이너 이름 어노테이션을 사용해 어떤
컨테이너가 어떤 계측을 받을지 지정한다.

언어 계측별 컨테이너 이름이 지정되지 않으면, (단일 계측 주입이 구성된 경우에
한해) 계측은 파드 스펙에서 첫 번째 일반 컨테이너에 대해 수행된다.

같은 파드의 컨테이너들이 서로 다른 기술을 사용하는 경우도 있다. 이런 경우 언어별
컨테이너 이름 어노테이션을 사용하여 (`.spec.containers.name`이나
`.spec.initContainers.name`에 있는) 주입 대상 컨테이너 이름을 하나 이상
지정한다.

| 언어         | 어노테이션                                                      |
| ------------ | --------------------------------------------------------------- |
| Java         | `instrumentation.opentelemetry.io/java-container-names`         |
| Node.js      | `instrumentation.opentelemetry.io/nodejs-container-names`       |
| Python       | `instrumentation.opentelemetry.io/python-container-names`       |
| .NET         | `instrumentation.opentelemetry.io/dotnet-container-names`       |
| Go           | `instrumentation.opentelemetry.io/go-container-names`           |
| Apache HTTPD | `instrumentation.opentelemetry.io/apache-httpd-container-names` |
| NGINX        | `instrumentation.opentelemetry.io/nginx-container-names`        |
| SDK only     | `instrumentation.opentelemetry.io/sdk-container-names`          |

Java와 Python이 서로 다른 컨테이너에서 실행되는 예제:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multi-containers-multi-instrumentations
spec:
  selector:
    matchLabels:
      app: my-pod-with-multi-containers-multi-instrumentations
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multi-containers-multi-instrumentations
      annotations:
        instrumentation.opentelemetry.io/inject-java: 'true'
        instrumentation.opentelemetry.io/java-container-names: 'myapp,myapp2'
        instrumentation.opentelemetry.io/inject-python: 'true'
        instrumentation.opentelemetry.io/python-container-names: 'myapp3'
    spec:
      containers:
        - name: myapp
          image: myImage1
        - name: myapp2
          image: myImage2
        - name: myapp3
          image: myImage3
```

위 경우 `myapp`과 `myapp2`는 Java로, `myapp3`는 Python 계측으로 계측된다.

> **참고**: Go 자동 계측은 다중 컨테이너 파드를 지원하지 **않는다**. **참고**:
> 하나의 컨테이너는 여러 언어 계측으로 동시에 계측될 수 없다. **참고**:
> `instrumentation.opentelemetry.io/container-names` 어노테이션은 이 기능에는
> 사용되지 않는다.

## 커스터마이징된 계측 또는 벤더 계측 사용 {#using-customized-or-vendor-instrumentation}

오퍼레이터는 기본적으로 업스트림 자동 계측 라이브러리를 사용한다.
`Instrumentation` CR에서 `image` 필드를 재정의해 커스텀 자동 계측 이미지를
구성할 수 있다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  java:
    image: your-customized-auto-instrumentation-image:java
  nodejs:
    image: your-customized-auto-instrumentation-image:nodejs
  python:
    image: your-customized-auto-instrumentation-image:python
  dotnet:
    image: your-customized-auto-instrumentation-image:dotnet
  go:
    image: your-customized-auto-instrumentation-image:go
  apacheHttpd:
    image: your-customized-auto-instrumentation-image:apache-httpd
  nginx:
    image: your-customized-auto-instrumentation-image:nginx
```

자동 계측을 위한 Dockerfile은
[autoinstrumentation 디렉터리](https://github.com/open-telemetry/opentelemetry-operator/tree/main/autoinstrumentation)에서
찾을 수 있다. 커스텀 컨테이너 이미지를 빌드하는 방법은 Dockerfile에 있는 안내를
따른다.

## Apache HTTPD 자동 계측 사용 {#using-apache-httpd-auto-instrumentation}

Apache HTTPD 자동 계측의 경우, 오퍼레이터는 기본적으로 (공식 `httpd` 이미지에서
사용되는) HTTPD 버전 2.4와 구성 디렉터리 `/usr/local/apache2/conf`를 가정한다.
버전 2.2를 사용하거나, 다른 구성 디렉터리를 사용하거나, 커스텀 에이전트 속성이
필요하다면 다음 예제를 사용한다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  apacheHttpd:
    image: your-customized-auto-instrumentation-image:apache-httpd
    version: '2.2'
    configPath: /your-custom-config-path
    attrs:
      - name: ApacheModuleOtelMaxQueueSize
        value: '4096'
```

사용 가능한 속성의 전체 목록은
[otel-webserver-module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module)에서
확인할 수 있다.

## NGINX 자동 계측 사용 {#using-nginx-auto-instrumentation}

NGINX 자동 계측은 NGINX 버전 1.22.0, 1.23.0, 1.23.1을 지원한다. NGINX 구성
파일은 기본적으로 `/etc/nginx/nginx.conf`일 것으로 예상된다. 계측은 또한 구성
파일과 같은 디렉터리에 `conf.d` 디렉터리가 있고, `http { ... }` 섹션에
`include <config-file-dir-path>/conf.d/*.conf;` 지시문이 있을 것으로 예상한다.
오픈텔레메트리 SDK 속성도 조정할 수 있다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  nginx:
    image: your-customized-auto-instrumentation-image:nginx
    configFile: /my/custom-dir/custom-nginx.conf
    attrs:
      - name: NginxModuleOtelMaxQueueSize
        value: '4096'
```

사용 가능한 속성의 전체 목록은
[otel-webserver-module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module)에서
확인할 수 있다.

## 오픈텔레메트리 SDK 환경 변수만 주입 {#inject-opentelemetry-sdk-environment-variables-only}

현재 자동 계측을 적용할 수 없는 애플리케이션의 경우, `inject-python`이나
`inject-java` 대신 `inject-sdk`를 사용해 오픈텔레메트리 SDK를 구성할 수 있다.
이렇게 하면 `Instrumentation` 리소스에서 구성한 `OTEL_RESOURCE_ATTRIBUTES`,
`OTEL_TRACES_SAMPLER`, `OTEL_EXPORTER_OTLP_ENDPOINT`와 같은 환경 변수는
주입되지만, SDK 자체는 주입되지 않는다.

```bash
instrumentation.opentelemetry.io/inject-sdk: "true"
```

## 계측 기능 제어 {#controlling-instrumentation-capabilities}

오퍼레이터는 기능 플래그를 통해 `Instrumentation` 리소스가 계측할 수 있는 언어를
지정할 수 있게 해준다. 기본적으로 활성화된 언어는 비활성화할 때만 해당 게이트를
지정하면 된다. 언어 지원은 플래그 값을 `false`로 전달해 비활성화할 수 있다.

| 언어         | 게이트                                | 기본값  |
| ------------ | ------------------------------------- | ------- |
| Java         | `enable-java-instrumentation`         | `true`  |
| Node.js      | `enable-nodejs-instrumentation`       | `true`  |
| Python       | `enable-python-instrumentation`       | `true`  |
| .NET         | `enable-dotnet-instrumentation`       | `true`  |
| Apache HTTPD | `enable-apache-httpd-instrumentation` | `true`  |
| Go           | `enable-go-instrumentation`           | `false` |
| NGINX        | `enable-nginx-instrumentation`        | `false` |

다중 계측(같은 파드에서 여러 언어)은 `enable-multi-instrumentation` 플래그로
활성화할 수 있으며, 기본값은 `false`이다. 다중 계측 기능에 대한 자세한 내용은
[다중 계측이 적용된 다중 컨테이너 파드](#multiple-instrumentations)를 참고한다.

## 리소스 속성 구성 {#configure-resource-attributes}

오픈텔레메트리 오퍼레이터는
[오픈텔레메트리 시맨틱 컨벤션](/docs/specs/semconv/non-normative/k8s-attributes/)에
정의된 리소스 속성을 자동으로 설정할 수 있다.

### 어노테이션으로 리소스 속성 구성 {#configure-resource-attributes-with-annotations}

`resource.opentelemetry.io/` 어노테이션 접두사를 사용해 오픈텔레메트리 계측이
생성하는 데이터에 리소스 속성을 추가할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    resource.opentelemetry.io/service.name: 'my-service'
    resource.opentelemetry.io/service.version: '1.0.0'
    resource.opentelemetry.io/deployment.environment.name: 'production'
spec:
  containers:
    - name: main-container
      image: your-image:tag
```

### 레이블로 리소스 속성 구성 {#configure-resource-attributes-with-labels}

일반적인 쿠버네티스 레이블을 사용해 리소스 속성을 설정할 수도 있다(첫 번째
항목이 우선한다). 다음 레이블이 지원된다.

- `app.kubernetes.io/instance` → `service.name`
- `app.kubernetes.io/name` → `service.name`
- `app.kubernetes.io/version` → `service.version`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  labels:
    app.kubernetes.io/name: 'my-service'
    app.kubernetes.io/version: '1.0.0'
    app.kubernetes.io/part-of: 'shop'
spec:
  containers:
    - name: main-container
      image: your-image:tag
```

이 기능을 사용하려면 `Instrumentation` CR에서 명시적으로 옵트인해야 한다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  defaults:
    useLabelsForResourceAttributes: true
```

### 리소스 속성 설정 우선순위 {#priority-for-setting-resource-attributes}

리소스 속성을 설정하는 우선순위는 다음과 같다(먼저 발견되는 값이 우선한다).

1. `OTEL_RESOURCE_ATTRIBUTES` 및 `OTEL_SERVICE_NAME` 환경 변수
2. `resource.opentelemetry.io/` 접두사가 붙은 어노테이션
3. `defaults.useLabelsForResourceAttributes=true`일 때의 레이블(예:
   `app.kubernetes.io/name`)
4. 파드의 메타데이터로부터 계산된 리소스 속성(예: `k8s.pod.name`)
5. `spec.resource.resourceAttributes` 아래에서 `Instrumentation` CR에 설정된
   리소스 속성

이 우선순위는 속성 하나하나에 개별적으로 적용되므로, 일부 속성은 어노테이션으로,
다른 속성은 레이블로 설정하는 것도 가능하다.

### 파드 메타데이터로부터 리소스 속성을 계산하는 방법 {#how-resource-attributes-are-calculated-from-pod-metadata}

#### service.name 계산 방법 {#how-servicename-is-calculated}

다음 순서로 처음 발견되는 값이 사용된다.

1. `pod.annotation[resource.opentelemetry.io/service.name]`
2. `pod.label[app.kubernetes.io/name]`(`useLabelsForResourceAttributes=true`인
   경우)
3. `k8s.deployment.name`
4. `k8s.replicaset.name`
5. `k8s.statefulset.name`
6. `k8s.daemonset.name`
7. `k8s.cronjob.name`
8. `k8s.job.name`
9. `k8s.pod.name`
10. `k8s.container.name`

#### service.version 계산 방법 {#how-serviceversion-is-calculated}

다음 순서로 처음 발견되는 값이 사용된다.

1. `pod.annotation[resource.opentelemetry.io/service.version]`
2. `pod.label[app.kubernetes.io/version]`(`useLabelsForResourceAttributes=true`인
   경우)
3. 컨테이너 Docker 이미지 태그(태그에 `/`가 포함되어 있지 않은 경우에 한함)

#### service.instance.id 계산 방법 {#how-serviceinstanceid-is-calculated}

다음 순서로 처음 발견되는 값이 사용된다.

1. `pod.annotation[resource.opentelemetry.io/service.instance.id]`
2. `k8s.namespace.name`, `k8s.pod.name`, `k8s.container.name`을 `.`으로 연결한
   값

#### service.namespace 계산 방법 {#how-servicenamespace-is-calculated}

다음 순서로 처음 발견되는 값이 사용된다.

1. `pod.annotation[resource.opentelemetry.io/service.namespace]`
2. `k8s.namespace.name`

## 문제 해결 {#troubleshooting}

코드를 자동 계측하려다 문제가 발생했다면, 시도해볼 수 있는 몇 가지 방법이 있다.

### Instrumentation 리소스가 설치되었는가? {#did-the-instrumentation-resource-install}

`Instrumentation` 리소스를 설치한 후, 다음 명령을 실행해 올바르게 설치되었는지
확인한다. 여기서 `<namespace>`는 `Instrumentation` 리소스가 배포된
네임스페이스이다.

```sh
kubectl describe otelinst -n <namespace>
```

출력 예시:

```yaml
Name:         python-instrumentation
Namespace:    application
Labels:       app.kubernetes.io/managed-by=opentelemetry-operator
Annotations:  instrumentation.opentelemetry.io/default-auto-instrumentation-apache-httpd-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-apache-httpd:1.0.3
             instrumentation.opentelemetry.io/default-auto-instrumentation-dotnet-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:0.7.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-go-image:
               ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.2.1-alpha
             instrumentation.opentelemetry.io/default-auto-instrumentation-java-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.26.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-nodejs-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.40.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-python-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
API Version:  opentelemetry.io/v1alpha1
Kind:         Instrumentation
Metadata:
 Creation Timestamp:  2023-07-28T03:42:12Z
 Generation:          1
 Resource Version:    3385
 UID:                 646661d5-a8fc-4b64-80b7-8587c9865f53
Spec:
...
 Exporter:
   Endpoint:  http://demo-collector.opentelemetry.svc.cluster.local:4318
...
 Propagators:
   tracecontext
   baggage
 Python:
   Image:  ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
   Resource Requirements:
     Limits:
       Cpu:     500m
       Memory:  32Mi
     Requests:
       Cpu:     50m
       Memory:  32Mi
 Resource:
 Sampler:
Events:  <none>
```

### OTel 오퍼레이터 로그에 자동 계측 오류가 표시되는가? {#do-the-otel-operator-logs-show-any-auto-instrumentation-errors}

다음 명령을 실행해 OTel 오퍼레이터 로그에서 자동 계측과 관련된 오류가 있는지
확인한다.

```sh
kubectl logs -l app.kubernetes.io/name=opentelemetry-operator --container manager -n opentelemetry-operator-system --follow
```

### 리소스가 올바른 순서로 배포되었는가? {#were-the-resources-deployed-in-the-right-order}

순서가 중요하다! `Instrumentation` 리소스는 애플리케이션을 배포하기 전에 먼저
배포되어야 하며, 그렇지 않으면 자동 계측이 동작하지 않는다.

자동 계측 어노테이션을 다시 살펴본다.

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-python: 'true'
```

파드가 시작될 때 위 어노테이션은 OTel 오퍼레이터에게 파드의 네임스페이스에서
`Instrumentation` 오브젝트를 찾으라고 알려준다. 또한 오퍼레이터에게 파드에
Python 자동 계측을 주입하라고 알려준다.

이는 애플리케이션의 파드에
[init-container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)를
추가하며, `opentelemetry-auto-instrumentation`이라고 불리는 이 컨테이너는 앱
컨테이너에 자동 계측을 주입하는 데 사용된다.

`Instrumentation` 리소스가 애플리케이션이 배포되는 시점까지 존재하지 않으면
init-container를 생성할 수 없다. 따라서 `Instrumentation` 리소스를 배포하기
_전에_ 애플리케이션이 배포되면 자동 계측은 실패한다.

`opentelemetry-auto-instrumentation` init-container가 올바르게 시작되었는지(또는
아예 시작이라도 되었는지) 확인하려면 다음 명령을 실행한다.

```sh
kubectl get events -n <your_app_namespace>
```

다음과 같은 출력이 나와야 한다.

```text
53s         Normal   Created             pod/py-otel-server-7f54bf4cbc-p8wmj    Created container opentelemetry-auto-instrumentation
53s         Normal   Started             pod/py-otel-server-7f54bf4cbc-p8wmj    Started container opentelemetry-auto-instrumentation
```

출력에서 `opentelemetry-auto-instrumentation`에 대한 `Created`나 `Started`
항목이 빠져 있다면 자동 계측에 문제가 있다는 뜻이다. 다음 중 하나가 원인일 수
있다.

- `Instrumentation` 리소스가 설치되지 않았거나(또는 올바르게 설치되지 않았다).
- `Instrumentation` 리소스가 애플리케이션이 배포된 _후에_ 설치되었다.
- 자동 계측 어노테이션에 오류가 있거나, 어노테이션이 잘못된 위치에 있다. 아래
  4번을 참고한다.

`kubectl get events`의 출력에서 오류가 있는지 확인해보면 문제를 파악하는 데
도움이 될 수 있다.

### 자동 계측 어노테이션이 올바른가? {#is-the-auto-instrumentation-annotation-correct}

자동 계측 어노테이션의 오류로 인해 자동 계측이 실패하는 경우가 있다.

다음 사항을 확인한다.

- **올바른 언어에 대한 자동 계측인가?**
  - 예를 들어 Python 애플리케이션을 계측할 때, 어노테이션이 실수로
    `instrumentation.opentelemetry.io/inject-java: "true"`로 잘못 지정되어 있지
    않은지 확인한다.
  - **Deno**의 경우, `deno`라는 문자열이 포함된 어노테이션이 아니라
    `instrumentation.opentelemetry.io/inject-sdk: "true"` 어노테이션을 사용하고
    있는지 확인한다.
- **자동 계측 어노테이션이 올바른 위치에 있는가?** `Deployment`를 정의할 때
  어노테이션을 추가할 수 있는 위치는 `spec.metadata.annotations`와
  `spec.template.metadata.annotations` 두 곳이다. 자동 계측 어노테이션은
  `spec.template.metadata.annotations`에 추가해야 하며, 그렇지 않으면 동작하지
  않는다.

### 자동 계측 엔드포인트가 올바르게 구성되었는가? {#was-the-auto-instrumentation-endpoint-configured-correctly}

`Instrumentation` 리소스의 `spec.exporter.endpoint` 속성은 데이터를 어디로
보낼지 정의한다. 이는 [OTel 컬렉터](/docs/collector/)나 다른 OTLP 엔드포인트일
수 있다. 이 속성을 생략하면 기본값인 `http://localhost:4317`이 사용되며,
대부분의 경우 이 주소로는 텔레메트리 데이터가 어디로도 전송되지 않는다.

같은 쿠버네티스 클러스터에 있는 OTel 컬렉터로 텔레메트리를 보낼 때는,
`spec.exporter.endpoint`가 OTel 컬렉터
[`Service`](https://kubernetes.io/docs/concepts/services-networking/service/)의
이름을 참조해야 한다.

예를 들면 다음과 같다.

```yaml
spec:
  exporter:
    endpoint: http://demo-collector.opentelemetry.svc.cluster.local:4317
```

여기서 컬렉터 엔드포인트는
`http://demo-collector.opentelemetry.svc.cluster.local:4317`로 설정되어 있으며,
`demo-collector`는 OTel 컬렉터 쿠버네티스 `Service`의 이름이다. 위 예제에서
컬렉터는 애플리케이션과 다른 네임스페이스에서 실행 중이므로, 컬렉터의 서비스
이름 뒤에 `opentelemetry.svc.cluster.local`을 붙여야 한다. 여기서
`opentelemetry`는 컬렉터가 위치한 네임스페이스이다.
