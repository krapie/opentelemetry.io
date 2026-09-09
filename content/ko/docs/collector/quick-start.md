---
title: 빠른 시작
description: 몇 분 만에 텔레메트리를 설정하고 수집한다!
aliases: [getting-started]
weight: 1
cSpell:ignore: docker dokey gobin okey telemetrygen
default_lang_commit: 51e300212ab0e50fc87d96726fcae3ba865b2332
---

<!-- markdownlint-disable ol-prefix blanks-around-fences -->

오픈텔레메트리(OpenTelemetry) 컬렉터는
[트레이스](/docs/concepts/signals/traces/),
[메트릭](/docs/concepts/signals/metrics/),
[로그](/docs/concepts/signals/logs/)와 같은 텔레메트리를 수신하고, 처리한 다음,
구성 요소 파이프라인을 통해 하나 이상의 옵저버빌리티 백엔드로 전달한다.

> [!NOTE]
>
> 이 빠른 시작 데모는 기본적인 로컬 설정을 생성한다. 목표는 프로덕션에 바로
> 사용할 수 있는 환경을 구축하는 것이 아니라, 컬렉터가 어떻게 작동하는지
> 보여주는 것이다.

이 가이드에서는 다음을 수행한다.

- 오픈텔레메트리 컬렉터의 로컬 인스턴스를 시작한다.
- 트레이스 데이터를 생성하여 컬렉터로 전송한다.
- 컬렉터가 데이터를 수신하고 처리하는지 확인한다.

이 가이드를 마치면 머신에서 실행되는 간단한 파이프라인을 갖추게 되며, 컬렉터가
옵저버빌리티 스택에서 어떤 역할을 하는지 더 명확하게 이해하게 된다. 시작하기
전에 더 많은 배경 지식이 필요하다면 [컬렉터](/docs/collector) 개요를 참고한다.

## 사전 요구 사항 {#prerequisites}

시작하기 전에 환경에 다음 도구가 설치되어 있는지 확인한다.

- [Docker](https://www.docker.com/) 또는 호환되는 컨테이너 런타임 — 컬렉터를
  실행하는 데 사용
- [Go](https://go.dev/), 최신 두 마이너 버전 중 하나 — 텔레메트리 생성기를
  설치하는 데 사용
- [`GOBIN` 환경 변수][gobin] 설정 — 설치된 Go 바이너리가 PATH에서 사용
  가능하도록 보장[^1]

`GOBIN`이 설정되어 있지 않다면 다음을 실행한다.

```sh
export GOBIN=${GOBIN:-$(go env GOPATH)/bin}
```

이 가이드는 `bash` 명령어를 사용한다. 다른 셸을 사용하는 경우, 명령어 구문을
조정해야 할 수 있다.

[^1]:
    자세한 내용은 [Your first program](https://go.dev/doc/code#Command)을
    참고한다.

## 환경 설정하기 {#set-up-the-environment}

1. 오픈텔레메트리 컬렉터 코어 [배포판](/docs/collector/distributions/)의 Docker
   이미지를 가져온다(pull).

   ```sh
   docker pull otel/opentelemetry-collector:{{% param vers %}}
   ```

2. 텔레메트리를 생성하는 클라이언트를 시뮬레이션하는 데 사용할
   [telemetrygen][]을 설치한다.

   ```sh
   go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
   ```

## 텔레메트리 생성 및 수집하기 {#generate-and-collect-telemetry}

3. 컬렉터를 시작한다.

   ```sh
   docker run \
     -p 127.0.0.1:4317:4317 \
     -p 127.0.0.1:4318:4318 \
     -p 127.0.0.1:55679:55679 \
     otel/opentelemetry-collector:{{% param vers %}} \
     2>&1 | tee collector-output.txt
   ```

   이전 명령어는 컬렉터를 로컬에서 실행하고 세 개의 포트를 연다.
   - `4317` — gRPC를 통한 OTLP, 대부분의 SDK에서 기본값
   - `4318` — HTTP를 통한 OTLP, gRPC를 지원하지 않는 클라이언트용
   - `55679` — ZPages, 브라우저에서 열 수 있는 내장 디버그 UI

4. 별도의 터미널에서 트레이스를 생성한다.

   ```sh
   $GOBIN/telemetrygen traces --otlp-insecure --traces 3
   ```

   트레이스가 전송되었음을 확인하는 출력이 표시된다.

   ```text
   2024-01-16T14:33:15.692-0500  INFO  traces/worker.go:99  traces generated  {"worker": 0, "traces": 3}
   2024-01-16T14:33:15.692-0500  INFO  traces/traces.go:58  stop the batch span processor
   ```

5. 컬렉터 터미널로 돌아가면, 다음과 유사한 트레이스 유입(ingest) 활동이
   표시되어야 한다.

   ```console
   $ grep -E '^Span|(ID|Name|Kind|time|Status \w+)\s+:' ./collector-output.txt
   Span #0
       Trace ID       : f30faffbde5fcf71432f89da1bf7bc14
       Parent ID      : 6f1ff7f9cf4ec1c7
       ID             : 8d1e820c1ac57337
       Name           : okey-dokey
       Kind           : Server
       Start time     : 2024-01-16 14:13:54.585877 +0000 UTC
       End time       : 2024-01-16 14:13:54.586 +0000 UTC
       Status code    : Unset
       Status message :
   Span #1
       Trace ID       : f30faffbde5fcf71432f89da1bf7bc14
       Parent ID      :
       ID             : 6f1ff7f9cf4ec1c7
       Name           : lets-go
       Kind           : Client
       Start time     : 2024-01-16 14:13:54.585877 +0000 UTC
       End time       : 2024-01-16 14:13:54.586 +0000 UTC
       Status code    : Unset
       Status message :
   ...
   ```

6. <kbd>Control-C</kbd>를 눌러 컬렉터를 중지한다.

## 다음 단계 {#next-steps}

이 시점에서, 컬렉터를 로컬에서 실행하고 텔레메트리를 처음부터 끝까지 어떻게
처리하는지 확인했다. 이제부터는 실제 환경에서 컬렉터가 어떻게 사용되는지 배워볼
수 있다.

- [구성](/docs/collector/configuration): 컬렉터의 구성 파일이 어떻게 작동하는지,
  그리고 Jaeger나 Prometheus 같은 실제 백엔드에 연결하는 방법을 알아본다.
- [배포 패턴](/docs/collector/deploy/): 컬렉터를 에이전트로 실행하는 것과
  게이트웨이로 실행하는 것의 차이를 이해한다.
- [컬렉터 설치하기](/docs/collector/install/): 바이너리와 Kubernetes를 포함해
  Docker 외의 설치 옵션을 살펴본다.
- [구성 요소 레지스트리](/ecosystem/registry/?language=collector): 파이프라인을
  확장하기 위해 사용 가능한 리시버, 프로세서, 익스포터를 살펴본다.

[gobin]: https://pkg.go.dev/cmd/go#hdr-Environment_variables
[telemetrygen]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen
