---
title: Docker 컨테이너로 OBI 실행하기
linkTitle: Docker
description:
  다른 컨테이너를 계측하는 독립형 Docker 컨테이너로 OBI를 설정하고 실행하는
  방법을 알아본다.
weight: 3
cSpell:ignore: goblog
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

OBI는 다른 컨테이너에서 실행 중인 프로세스를 계측할 수 있는 독립형 Docker
컨테이너로 실행할 수 있다.

OBI 컨테이너 이미지는 두 레지스트리 모두에 게시된다.

- [Docker Hub](https://hub.docker.com/r/otel/ebpf-instrument):
  `otel/ebpf-instrument:v<version>`
- [GHCR](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/pkgs/container/opentelemetry-ebpf-instrumentation%2Febpf-instrument):
  `ghcr.io/open-telemetry/opentelemetry-ebpf-instrumentation/ebpf-instrument:v<version>`

개발 태그도 Docker Hub에 다음과 같이 게시된다.

```text
otel/ebpf-instrument:main
```

OBI 컨테이너는 다음과 같이 구성해야 한다.

- **특권(privileged)** 컨테이너로 실행하거나, `SYS_ADMIN` capability를 가진
  컨테이너로 실행한다(단, 후자의 방법은 일부 컨테이너 환경에서 동작하지 않을 수
  있다)
- 다른 컨테이너의 프로세스에 접근할 수 있도록 `host` PID 네임스페이스를
  사용한다.

## 이미지 서명 및 검증 {#image-signing-and-verification}

OBI 컨테이너 이미지는 GitHub Actions에서 OIDC(OpenID Connect) 프로토콜로
인증되는 임시 키(ephemeral key)를 사용하여
[Cosign](https://docs.sigstore.dev/cosign/signing/overview/)으로 서명된다. 이를
통해 오픈텔레메트리(OpenTelemetry) 프로젝트가 게시한 컨테이너의 진위성과
무결성이 보장된다.

다음 명령을 사용하여 컨테이너 이미지의 서명을 검증할 수 있다.

```sh
export VERSION=v0.12.1

# Verify a release image from Docker Hub
cosign verify --certificate-identity-regexp 'https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/' --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' otel/ebpf-instrument:${VERSION}

# Verify the same release from GHCR
cosign verify --certificate-identity-regexp 'https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/' --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' ghcr.io/open-telemetry/opentelemetry-ebpf-instrumentation/ebpf-instrument:${VERSION}
```

다음은 예시 출력이다.

```log
Verification for index.docker.io/otel/ebpf-instrument:main --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The code-signing certificate was verified using trusted certificate authority certificates

[{"critical":{"identity":{"docker-reference":"index.docker.io/otel/ebpf-instrument:main"},"image":{"docker-manifest-digest":"sha256:55426a2bbb8003573a961697888aa770a1f5f67fcda2276dc2187d1faf7181fe"},"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]
```

검증에 성공하면 Cosign 클레임이 검증되었다고 보고하고 서명된 이미지 다이제스트를
표시한다. 검증에 실패하면 다음을 수행한다.

- 조회한 레지스트리에 해당 태그가 존재하는지 확인한다
- `main`이 아니라 게시된 릴리스 태그를 검증하고 있는지 확인한다
- 위에 표시된 GitHub OIDC 발급자(issuer)와 아이덴티티 정규 표현식을 사용하고
  있는지 확인한다

## Docker CLI 예시 {#docker-cli-example}

이 예시를 위해서는 HTTP/S 또는 gRPC 서비스를 실행하는 컨테이너가 필요하다.
없다면 이 [Go로 작성된 간단한 블로그 엔진 서비스](https://macias.info)를 사용할
수 있다.

```sh
export VERSION=v0.12.1
docker run -p 18443:8443 --name goblog mariomac/goblog:dev
```

위 명령은 간단한 HTTPS 애플리케이션을 실행한다. 이 프로세스는 컨테이너의 내부
포트 `8443`을 여는데, 이는 호스트 수준에서 포트 `18443`으로 노출된다.

OBI가 stdout으로 출력하고 실행 파일을 검사하기 위해 (컨테이너의) 포트를 수신
대기하도록 구성하는 환경 변수를 설정한다.

```sh
export OTEL_EBPF_TRACE_PRINTER=text
export OTEL_EBPF_OPEN_PORT=8443
```

OBI는 다음 설정으로 실행해야 한다.

- `--privileged` 모드로 실행하거나 `SYS_ADMIN` capability와 함께 실행한다(단,
  일부 컨테이너 환경에서는 `SYS_ADMIN`만으로는 권한이 충분하지 않을 수 있다)
- `--pid=host` 옵션으로 호스트 PID 네임스페이스를 사용한다.

```sh
docker run --rm \
  -e OTEL_EBPF_OPEN_PORT=8443 \
  -e OTEL_EBPF_TRACE_PRINTER=text \
  --pid=host \
  --privileged \
  otel/ebpf-instrument:${VERSION}
```

OBI가 실행되면 브라우저에서 `https://localhost:18443`을 열고, 앱을 사용하여
테스트 데이터를 생성한 뒤, OBI가 다음과 유사하게 트레이스 요청을 stdout에
출력하는지 확인한다.

```sh
time=2023-05-22T14:03:42.402Z level=INFO msg="creating instrumentation pipeline"
time=2023-05-22T14:03:42.526Z level=INFO msg="Starting main node"
2023-05-22 14:03:53.5222353 (19.066625ms[942.583µs]) 200 GET / [172.17.0.1]->[localhost:18443] size:0B
2023-05-22 14:03:53.5222353 (355.792µs[321.75µs]) 200 GET /static/style.css [172.17.0.1]->[localhost:18443] size:0B
2023-05-22 14:03:53.5222353 (170.958µs[142.916µs]) 200 GET /static/img.png [172.17.0.1]->[localhost:18443] size:0B
2023-05-22 14:13:47.52221347 (7.243667ms[295.292µs]) 200 GET /entry/201710281345_instructions.md [172.17.0.1]->[localhost:18443] size:0B
2023-05-22 14:13:47.52221347 (115µs[75.625µs]) 200 GET /static/style.css [172.17.0.1]->[localhost:18443] size:0B
```

이제 OBI가 대상 HTTP 서비스를 트레이싱하고 있으므로, 메트릭과 트레이스를
오픈텔레메트리 엔드포인트로 전송하거나 Prometheus가 메트릭을 스크레이프하도록
구성한다.

트레이스와 메트릭을 내보내는 방법에 대한 정보는
[구성 옵션](../../configure/options/) 문서를 참고한다.

## Docker Compose 예시 {#docker-compose-example}

다음 Docker Compose 파일은 Docker CLI 예시와 동일한 기능을 재현한다.

```yaml
version: '3.8'

services:
  # Service to instrument. Change it to any
  # other container that you want to instrument.
  goblog:
    image: mariomac/goblog:dev
    ports:
      # Exposes port 18843, forwarding it to container port 8443
      - '18443:8443'

  autoinstrumenter:
    image: otel/ebpf-instrument:main
    pid: 'host'
    privileged: true
    environment:
      OTEL_EBPF_TRACE_PRINTER: text
      OTEL_EBPF_OPEN_PORT: 8443
```

다음 명령으로 Docker Compose 파일을 실행하고 앱을 사용하여 트레이스를 생성한다.

```sh
docker compose -f compose-example.yml up
```
