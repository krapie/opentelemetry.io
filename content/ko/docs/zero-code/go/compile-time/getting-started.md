---
title: 시작하기
description:
  계측 코드를 전혀 작성하지 않고 Go 애플리케이션에서 텔레메트리를 캡처한다.
weight: 5
cSpell:ignore: GOFLAGS otelc toolexec
default_lang_commit: 5234e6b790d14d8db1a3c75e70ac64417fbfe531
---

이 페이지에서는 컴파일 타임 계측(compile-time instrumentation)으로 Go
애플리케이션을 빌드하고 그것이 생성하는 텔레메트리를 확인하는 방법을 보여준다.

## 사전 요구 사항 {#prerequisites}

- [Go](https://go.dev/) 1.25 이상

## otelc 설치 {#install-otelc}

이 프로젝트는 표준 Go 툴체인(toolchain)을 감싸는 `otelc`라는 명령줄 도구를
제공한다. `go install`로 설치한다.

```sh
go install go.opentelemetry.io/otelc/tool/cmd/otelc@latest
```

이렇게 하면 `otelc` 바이너리가 Go bin 디렉터리(기본값은
`$(go env GOPATH)/bin`)에 놓인다. 이후 단계는 `otelc`가 `PATH`에 있다고
가정한다.

또는, 예를 들어 릴리스되지 않은 변경 사항을 시험해 보기 위해 소스에서 도구를
빌드할 수도 있다. 이 경우 `git`과 `make`가 필요하다.

```sh
git clone https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation.git
cd opentelemetry-go-compile-instrumentation
make build
```

이렇게 하면 저장소 루트에 `otelc` 바이너리가 생성되며, 이를 `PATH`에 추가할 수
있다.

```sh
export PATH=$PATH:$(pwd)
```

## 애플리케이션 계측하기 {#instrument-your-application}

빌드에 대한 변경은 한 줄뿐이다. `go build`를 실행하던 곳에서 `otelc go build`를
실행하면 된다. 애플리케이션의 모듈 디렉터리에서 다음을 실행한다.

```sh
otelc go build -o myapp .
```

`go` 뒤의 모든 것은 툴체인으로 전달되므로, 나머지 빌드는 그대로 유지된다. 이
도구는 빌드를 가로채서 애플리케이션과 그 의존성에 매칭되는 계측 규칙을 적용하고,
계측된 바이너리를 생성한다. 플래그, 패키지 인자, 출력 경로 등 빌드의 다른 모든
것은 일반 `go build`와 정확히 동일하게 동작한다.

기본적으로 `otelc`는 모듈에서 지원되는 라이브러리를 찾아내어, 구성 없이 그리고
코드 변경 없이 자동으로 계측한다.

### go build 계속 사용하기 {#keep-using-go-build}

빌드 명령을 바꾸고 싶지 않다면, `otelc setup`을 한 번 실행하여 모듈을 준비한
다음, `GOFLAGS`를 통해 Go 툴체인이 `otelc`를 가리키도록 하고 평소처럼
`go build`를 계속 실행한다.

```sh
otelc setup
export GOFLAGS="${GOFLAGS} '-toolexec=otelc toolexec'"
go build -o myapp .
```

이는 `go build` 명령이 바꾸고 싶지 않은 기존 빌드 시스템이나 스크립트에 의해
고정되어 있을 때 적합하다.

## 컨테이너 빌드 계측하기 {#instrument-a-container-build}

동일한 교체가 컨테이너 빌드에서도 동작한다. 빌드 스테이지에 `otelc`를 설치하고
`Dockerfile`의 `go build` 줄을 `otelc go build`로 바꾼다.

```dockerfile
# Build stage
FROM golang:1.25 AS build
WORKDIR /src
COPY . .
RUN go install go.opentelemetry.io/otelc/tool/cmd/otelc@latest
RUN otelc go build -o /out/myapp .

# Runtime stage
FROM gcr.io/distroless/base-debian12
COPY --from=build /out/myapp /myapp
ENTRYPOINT ["/myapp"]
```

계측은 바이너리로 컴파일되므로, 런타임 스테이지에는 별도로 필요한 것이 없다.
연결할 에이전트도, 추가 시작 단계도 없다.

## 애플리케이션 실행 및 텔레메트리 내보내기 {#run-the-application-and-export-telemetry}

계측된 애플리케이션은 표준 오픈텔레메트리(OpenTelemetry) 환경 변수로 구성한다.
예를 들어, 텔레메트리를 로컬 [컬렉터(Collector)](/docs/collector/)로 OTLP를 통해
전송하려면 다음과 같이 한다.

```sh
export OTEL_SERVICE_NAME=myapp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
./myapp
```

계측이 인식하는 환경 변수의 전체 목록은 [구성](../configuration)을 참고한다.

## 데모 사용해 보기 {#try-the-demo}

저장소는 데모 애플리케이션과 완전한 옵저버빌리티 스택(컬렉터, Jaeger,
Prometheus, Grafana)을 제공하므로, 생성된 텔레메트리를 처음부터 끝까지 확인할 수
있다. 이 스택은 [Docker](https://www.docker.com/)에서 실행되므로, 먼저 Docker를
사용할 수 있는지 확인한다. 저장소 루트에서 다음을 실행한다.

```sh
cd demo/infrastructure/docker-compose
make start
```

데모 애플리케이션과 인프라에 대한 자세한 내용은
[demo 디렉터리](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/tree/main/demo)를
참고한다.

## 다음 단계 {#next-steps}

- 기본적으로 [어떤 라이브러리가 계측되는지](../supported-libraries) 확인한다.
- 도구와 그것이 생성하는 텔레메트리를 [구성](../configuration)하는 방법을
  알아본다.
