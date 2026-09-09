---
title: OCB(OpenTelemetry Collector Builder)로 커스텀 컬렉터 빌드하기
linkTitle: 커스텀 컬렉터 빌드하기
description: 직접 오픈텔레메트리(OpenTelemetry) 컬렉터 배포판을 구성한다.
weight: 200
aliases: [/docs/collector/custom-collector]
params:
  providers-vers: v1.48.0
# prettier-ignore
cSpell:ignore: chipset darwin debugexporter gomod otlpexporter otlpreceiver wyrtw
default_lang_commit: 150fe5a5c5f0914cd067d4632ab26e529e7b9167
---

오픈텔레메트리(OpenTelemetry) 컬렉터에는 특정 구성 요소로 미리 구성된 다섯 가지
공식 [배포판](/docs/collector/distributions/)이 있다. 더 많은 유연성이
필요하다면, [OCB(OpenTelemetry Collector Builder)][ocb] (또는 `ocb`)를 사용해
커스텀 구성 요소, 업스트림 구성 요소, 그 밖에 공개적으로 사용 가능한 구성 요소를
포함하는 나만의 배포판 커스텀 바이너리를 생성할 수 있다.

다음 가이드는 `ocb`를 사용해 나만의 컬렉터를 빌드하는 방법을 안내한다. 이
예제에서는 커스텀 구성 요소의 개발과 테스트를 지원하는 컬렉터 배포판을 만든다.
원하는 Golang 통합 개발 환경(IDE)에서 컬렉터 구성 요소를 직접 실행하고 디버그할
수 있다. IDE의 모든 디버깅 기능(스택 트레이스는 훌륭한 스승이다!)을 활용해
컬렉터가 구성 요소 코드와 어떻게 상호작용하는지 이해한다.

## 사전 요구 사항 {#prerequisites}

`ocb` 도구는 컬렉터 배포판을 빌드하기 위해 Go를 필요로 한다. 시작하기 전에
컴퓨터에
[호환되는 버전](https://github.com/open-telemetry/opentelemetry-collector/blob/main/README.md#compatibility)의
Go를 [설치](https://go.dev/doc/install)했는지 확인한다.

## OCB 설치하기 {#install-the-opentelemetry-collector-builder}

`ocb` 바이너리는 [`cmd/builder` 태그][tags]가 붙은 오픈텔레메트리 컬렉터
릴리스에서 다운로드 가능한 애셋(asset)으로 제공된다. 운영체제와 칩셋에 맞는
애셋을 찾아 다운로드한다.

{{< tabpane text=true >}}

{{% tab "Linux (AMD 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o ocb \
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_linux_amd64
chmod +x ocb
```

{{% /tab %}} {{% tab "Linux (ARM 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o ocb \
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_linux_arm64
chmod +x ocb
```

{{% /tab %}} {{% tab "Linux (ppc64le) "%}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o ocb \
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_linux_ppc64le
chmod +x ocb
```

{{% /tab %}} {{% tab "macOS (AMD 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o ocb \
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_darwin_amd64
chmod +x ocb
```

{{% /tab %}} {{% tab "macOS (ARM 64)" %}}

```sh
curl --proto '=https' --tlsv1.2 -fL -o ocb \
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_darwin_arm64
chmod +x ocb
```

{{% /tab %}} {{% tab "Windows (AMD 64)" %}}

```sh
Invoke-WebRequest -Uri "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/cmd%2Fbuilder%2F{{% version-from-registry collector-builder %}}/ocb_{{% version-from-registry collector-builder noPrefix %}}_windows_amd64.exe" -OutFile "ocb.exe"
Unblock-File -Path "ocb.exe"
```

{{% /tab %}} {{< /tabpane >}}

`ocb`가 올바르게 설치되었는지 확인하려면, 터미널에 `./ocb help`를 입력한다.
콘솔에 `help` 명령어의 출력이 표시되어야 한다.

## OCB 구성하기 {#configure-the-opentelemetry-collector-builder}

`ocb`는 YAML 매니페스트(manifest) 파일로 구성한다. 매니페스트는 두 개의 주요
섹션으로 이루어진다. 첫 번째 섹션인 `dist`는 코드 생성과 컴파일 프로세스를
구성하는 옵션을 포함한다. 두 번째 섹션은 `extensions`, `exporters`, `receivers`,
`processors`와 같은 최상위 모듈 유형을 포함한다. 각 모듈 유형은 구성 요소 목록을
받는다.

매니페스트의 `dist` 섹션은 `ocb` 커맨드 라인 `flags`와 동등한 태그를 포함한다.
다음 표는 `dist` 섹션을 구성하는 옵션을 나열한 것이다.

| 태그               | 설명                                                | 선택 여부         | 기본값                                                                            |
| ------------------ | --------------------------------------------------- | ----------------- | --------------------------------------------------------------------------------- |
| module:            | Go mod 컨벤션을 따르는 새 배포판의 모듈 이름        | 예, 그러나 권장됨 | `go.opentelemetry.io/collector/cmd/builder`                                       |
| name:              | 배포판의 바이너리 이름                              | 예                | `otelcol-custom`                                                                  |
| description:       | 애플리케이션의 긴 이름                              | 예                | `Custom OpenTelemetry Collector distribution`                                     |
| output_path:       | 출력(소스와 바이너리)을 작성할 경로                 | 예                | `/var/folders/86/s7l1czb16g124tng0d7wyrtw0000gn/T/otelcol-distribution3618633831` |
| version:           | 커스텀 오픈텔레메트리 컬렉터의 버전                 | 예                | `1.0.0`                                                                           |
| go:                | 생성된 소스를 컴파일하는 데 사용할 Go 바이너리      | 예                | PATH에서 찾은 go                                                                  |
| debug_compilation: | 결과 바이너리에 디버그 심볼(symbol)을 유지할지 여부 | 예                | False                                                                             |

모든 `dist` 태그는 선택 사항이다. 커스텀 컬렉터 배포판을 다른 사용자에게 제공할
계획인지, 아니면 `ocb`를 사용해 구성 요소 개발 및 테스트 환경을
부트스트랩(bootstrap)하려는 것인지에 따라 값을 커스터마이징하여 추가할 수 있다.

`ocb`를 구성하려면 다음 단계를 따른다.

1. 다음 내용으로 `builder-config.yaml`이라는 매니페스트 파일을 생성한다.

   ```yaml
   dist:
     name: otelcol-dev
     description: Basic OTel Collector distribution for Developers
     output_path: ./otelcol-dev
   ```

1. 이 커스텀 컬렉터 배포판에 포함하려는 구성 요소를 위한 모듈을 추가한다. 다양한
   모듈과 구성 요소를 추가하는 방법을 이해하려면
   [`ocb` 구성 문서](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder#configuration)를
   참고한다.

   이 예제 배포판에는 다음 구성 요소를 추가한다.
   - 익스포터: OTLP 및 Debug
   - 리시버: OTLP
   - 프로세서: Batch

   `builder-config.yaml` 매니페스트 파일은 다음과 같은 모습이어야 한다.

   ```yaml
   dist:
     name: otelcol-dev
     description: Basic OTel Collector distribution for Developers
     output_path: ./otelcol-dev

   exporters:
     - gomod:
         go.opentelemetry.io/collector/exporter/debugexporter {{%
         version-from-registry collector-exporter-debug %}}
     - gomod:
         go.opentelemetry.io/collector/exporter/otlpexporter {{%
         version-from-registry collector-exporter-otlp %}}

   processors:
     - gomod:
         go.opentelemetry.io/collector/processor/batchprocessor {{%
         version-from-registry collector-processor-batch %}}

   receivers:
     - gomod:
         go.opentelemetry.io/collector/receiver/otlpreceiver {{%
         version-from-registry collector-receiver-otlp %}}

   providers:
     - gomod:
         go.opentelemetry.io/collector/confmap/provider/envprovider {{% param
         providers-vers %}}
     - gomod:
         go.opentelemetry.io/collector/confmap/provider/fileprovider {{% param
         providers-vers %}}
     - gomod:
         go.opentelemetry.io/collector/confmap/provider/httpprovider {{% param
         providers-vers %}}
     - gomod:
         go.opentelemetry.io/collector/confmap/provider/httpsprovider {{% param
         providers-vers %}}
     - gomod:
         go.opentelemetry.io/collector/confmap/provider/yamlprovider {{% param
         providers-vers %}}
   ```

> [!TIP]
>
> 커스텀 컬렉터에 추가할 수 있는 구성 요소 목록은
> [오픈텔레메트리 레지스트리](/ecosystem/registry/?language=collector)를
> 참고한다. 각 레지스트리 항목에는 `builder-config.yaml`에 추가해야 하는 전체
> 이름과 버전이 포함되어 있다.

## 코드를 생성하고 컬렉터 배포판 빌드하기 {#generate-the-code-and-build-your-collector-distribution}

> [!NOTE]
>
> 이 섹션에서는 `ocb` 바이너리를 사용해 커스텀 컬렉터 배포판을 빌드하는 방법을
> 안내한다. 커스텀 컬렉터 배포판을 Kubernetes와 같은 컨테이너 오케스트레이터에
> 빌드하고 배포하려면, 이 섹션을 건너뛰고
> [컬렉터 배포판 컨테이너화하기](#containerize-your-collector-distribution)를
> 참고한다.

`ocb`를 설치하고 구성했다면, 이제 배포판을 빌드할 준비가 된 것이다.

터미널에서 다음 명령어를 입력해 `ocb`를 시작한다.

```sh
./ocb --config builder-config.yaml
```

명령어의 출력은 다음과 같은 모습이다.

```text
2025-06-13T14:25:03.037-0500	INFO	internal/command.go:85	OpenTelemetry Collector distribution builder	{"version": "{{% version-from-registry collector-builder noPrefix %}}", "date": "2025-06-03T15:05:37Z"}
2025-06-13T14:25:03.039-0500	INFO	internal/command.go:108	Using config file	{"path": "builder-config.yaml"}
2025-06-13T14:25:03.040-0500	INFO	builder/config.go:99	Using go	{"go-executable": "/usr/local/go/bin/go"}
2025-06-13T14:25:03.041-0500	INFO	builder/main.go:76	Sources created	{"path": "./otelcol-dev"}
2025-06-13T14:25:03.445-0500	INFO	builder/main.go:108	Getting go modules
2025-06-13T14:25:04.675-0500	INFO	builder/main.go:87	Compiling
2025-06-13T14:25:17.259-0500	INFO	builder/main.go:94	Compiled	{"binary": "./otelcol-dev/otelcol-dev"}
```

매니페스트의 `dist` 섹션에 정의된 대로, 이제 컬렉터 배포판의 모든 소스 코드와
바이너리가 담긴 `otelcol-dev`라는 폴더가 생성되었다.

폴더 구조는 다음과 같다.

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

생성된 코드를 사용해 구성 요소 개발 프로젝트를 부트스트랩한 다음, 해당 구성
요소로 나만의 컬렉터 배포판을 빌드하고 배포할 수 있다.

## 컬렉터 배포판 컨테이너화하기 {#containerize-your-collector-distribution}

> [!NOTE]
>
> 이 섹션에서는 `Dockerfile` 내에서 컬렉터 배포판을 빌드하는 방법을 설명한다.
> 컬렉터 배포판을 Kubernetes와 같은 컨테이너 오케스트레이터에 배포해야 하는 경우
> 이 안내를 따른다. 컨테이너화 없이 컬렉터 배포판을 빌드하고 싶다면,
> [코드를 생성하고 컬렉터 배포판 빌드하기](#generate-the-code-and-build-your-collector-distribution)를
> 참고한다.

다음 단계에 따라 커스텀 컬렉터를 컨테이너화한다.

1. 프로젝트에 두 개의 새 파일을 추가한다.
   - `Dockerfile` - 컬렉터 배포판의 컨테이너 이미지 정의
   - `collector-config.yaml` - 배포판을 테스트하기 위한 최소한의 컬렉터 설정
     YAML

   이 파일들을 추가한 후, 파일 구조는 다음과 같다.

   ```text
   .
   ├── builder-config.yaml
   ├── collector-config.yaml
   └── Dockerfile
   ```

1. `Dockerfile`에 다음 내용을 추가한다. 이 정의는 컬렉터 배포판을 제자리
   (in-place)에서 빌드하며, 결과로 생성되는 컬렉터 배포판 바이너리가 대상
   컨테이너 아키텍처(예: `linux/arm64`, `linux/amd64`)와 일치하도록 보장한다.

   ```dockerfile
   FROM alpine:3.19 AS certs
   RUN apk --update add ca-certificates

   FROM golang:1.25.0 AS build-stage
   WORKDIR /build

   COPY ./builder-config.yaml builder-config.yaml

   RUN --mount=type=cache,target=/root/.cache/go-build GO111MODULE=on go install go.opentelemetry.io/collector/cmd/builder@{{% version-from-registry collector-builder %}}
   RUN --mount=type=cache,target=/root/.cache/go-build builder --config builder-config.yaml

   FROM gcr.io/distroless/base:latest

   ARG USER_UID=10001
   USER ${USER_UID}

   COPY ./collector-config.yaml /otelcol/collector-config.yaml
   COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
   COPY --chmod=755 --from=build-stage /build/otelcol-dev /otelcol

   ENTRYPOINT ["/otelcol/otelcol-dev"]
   CMD ["--config", "/otelcol/collector-config.yaml"]

   EXPOSE 4317 4318 12001
   ```

   > [!NOTE]
   >
   > Dockerfile은 `COPY` 및 `ENTRYPOINT` 지시어(instruction)에서
   > `builder-config.yaml`의 배포판 이름 `otelcol-dev`를 참조한다.
   > `builder-config.yaml`의 `dist` 섹션에서 `name` 또는 `output_path`를
   > 변경했다면, Dockerfile에서 다음 줄도 그에 맞게 업데이트했는지 확인한다.
   >
   > - `COPY --chmod=755 --from=build-stage /build/<dist_name> /otelcol`
   > - `ENTRYPOINT ["/otelcol/<dist_name>"]`

1. `collector-config.yaml` 파일에 다음 정의를 추가한다.

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
       verbosity: detailed

   service:
     pipelines:
       traces:
         receivers: [otlp]
         exporters: [debug]
       metrics:
         receivers: [otlp]
         exporters: [debug]
       logs:
         receivers: [otlp]
         exporters: [debug]
   ```

1. `linux/amd64`와 `linux/arm64`를 대상 빌드 아키텍처로 사용하여 `ocb`의 멀티
   아키텍처 Docker 이미지를 빌드하려면 다음 명령어를 사용한다. 자세히 알아보려면
   멀티 아키텍처 빌드에 관한 이
   [블로그 게시물](https://blog.jaimyn.dev/how-to-build-multi-architecture-docker-images-on-an-m1-mac/)을
   참고한다.

   ```sh
   # Enable Docker multi-arch builds
   docker run --rm --privileged tonistiigi/binfmt --install all
   docker buildx create --name mybuilder --use

   # Build the Docker image as Linux AMD and ARM
   # and load the result to "docker images"
   docker buildx build --load \
     -t <collector_distribution_image_name>:<version> \
     --platform=linux/amd64,linux/arm64 .

   # Test the newly built image
   docker run -it --rm -p 4317:4317 -p 4318:4318 \
       --name otelcol <collector_distribution_image_name>:<version>
   ```

## 더 읽어보기 {#further-reading}

- [리시버 빌드하기](/docs/collector/extend/custom-component/receiver)
- [커넥터 빌드하기](/docs/collector/extend/custom-component/connector)

[ocb]:
  https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder
[tags]: https://github.com/open-telemetry/opentelemetry-collector-releases/tags
