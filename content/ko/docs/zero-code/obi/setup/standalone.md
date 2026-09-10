---
title: 독립형 프로세스로 OBI 실행하기
linkTitle: 독립형
description: 독립형 Linux 프로세스로 OBI를 설정하고 실행하는 방법을 알아본다.
weight: 5
cSpell:ignore: cyclonedx
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

OBI는 실행 중인 다른 프로세스를 검사할 수 있는 상승된 권한(elevated
privileges)을 가진 독립형 Linux OS 프로세스로 실행할 수 있다.

## 다운로드 및 검증 {#download-and-verify}

OBI는 Linux(amd64 및 arm64)용 사전 빌드된 바이너리를 제공한다. 최신 릴리스는
[릴리스 페이지](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases)에서
다운로드한다. 각 릴리스에는 다음이 포함된다.

- `obi-v<version>-linux-amd64.tar.gz` - Linux AMD64/x86_64 아카이브
- `obi-v<version>-linux-arm64.tar.gz` - Linux ARM64 아카이브
- `obi-v<version>-linux-amd64.cyclonedx.json` - AMD64 아카이브용 CycloneDX SBOM
- `obi-v<version>-linux-arm64.cyclonedx.json` - ARM64 아카이브용 CycloneDX SBOM
- `obi-v<version>-source-generated.cyclonedx.json` - 소스 생성(source-generated)
  아카이브용 CycloneDX SBOM
- `obi-java-agent-v<version>.cyclonedx.json` - 내장된 Java 에이전트 및 그 Java
  의존성에 대한 CycloneDX SBOM
- `SHA256SUMS` - 릴리스 아카이브 및 SBOM 자산 검증용 체크섬

동일한 릴리스의 컨테이너 이미지도 게시된다. 이미지 풀(pull) 및 서명 검증 방법은
[Docker 컨테이너로 OBI 실행하기](../docker/)를 참고한다.

원하는 버전과 아키텍처를 설정한다.

```sh
# Set your desired version (find latest at
# https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases)
VERSION=0.12.1

# Determine your architecture
# For Intel/AMD 64-bit: amd64
# For ARM 64-bit: arm64
ARCH=amd64

# Download the archive for your architecture
wget https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases/download/v${VERSION}/obi-v${VERSION}-linux-${ARCH}.tar.gz

# Download checksums
wget https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases/download/v${VERSION}/SHA256SUMS

# Verify the archive
sha256sum -c SHA256SUMS --ignore-missing

# Extract the archive
tar -xzf obi-v${VERSION}-linux-${ARCH}.tar.gz
```

검증에 성공하면 다운로드한 각 파일에 대해 `OK` 결과가 출력된다.

```text
obi-v${VERSION}-linux-${ARCH}.tar.gz: OK
```

검증에 실패하면 `sha256sum`은 `FAILED`를 보고한다. 이 경우 다음을 수행한다.

- `VERSION`이 다운로드한 아카이브 및 `SHA256SUMS`와 일치하는지 확인한다
- 부분적으로 다운로드된 파일을 삭제하고 다시 받는다
- 해당 릴리스에서 실제로 다운로드한 파일만 검증한다

아카이브에는 다음이 포함된다.

- `obi` - 메인 OBI 바이너리
- `LICENSE` - 프로젝트 라이선스
- `NOTICE` - 법적 고지
- `NOTICES/` - 서드파티 라이선스 및 저작자 표시

> [!IMPORTANT]
>
> OBI v0.6.0부터 Java 에이전트가 `obi` 바이너리에 내장된다. 별도의
> `obi-java-agent.jar` 파일은 필요하지 않다. 런타임에 OBI는 내장된 Java
> 에이전트를 추출하여 `$XDG_CACHE_HOME/obi/java`(또는 `~/.cache/obi/java`)에
> 캐시한다.
>
> 캐시 디렉터리는 `obi`를 실행하는 사용자 계정에 따라 결정된다. `sudo`를
> 사용하면, 재정의하지 않는 한 캐시는 일반적으로 root 사용자의 캐시 디렉터리(예:
> `/root/.cache/obi/java`) 아래에 생성된다. 시스템 또는 서비스 배포의 경우,
> `XDG_CACHE_HOME`을 적절한 위치로 설정하거나(예:
> `XDG_CACHE_HOME=/var/cache/obi sudo -E obi ...`) 환경에 맞게 명시적인 캐시
> 경로를 구성한다.

## SBOM {#sboms}

CycloneDX SBOM 파일은 공급망(supply-chain) 검토 및 자동화를 위한 선택적
메타데이터이다. OBI를 설치하거나 실행하는 데는 필요하지 않다.

게시된 SBOM은 바이너리 아카이브와 내장 구성 요소의 내용을
[CycloneDX JSON 형식](https://cyclonedx.org/)으로 설명한다. 표준 SBOM 도구와
함께 사용하여 바이너리를 실행하지 않고도 의존성, 라이선스, 구성 요소를 검사할 수
있다.

검사하려는 SBOM을 다운로드한다.

```sh
# SBOM for the binary archive you downloaded
wget https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases/download/v${VERSION}/obi-v${VERSION}-linux-${ARCH}.cyclonedx.json

# SBOM for the embedded Java agent and its Java dependencies
wget https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/releases/download/v${VERSION}/obi-java-agent-v${VERSION}.cyclonedx.json

# Optional: verify the downloaded SBOM files against SHA256SUMS too
sha256sum -c SHA256SUMS --ignore-missing
```

검사 명령 예시:

```sh
# List component names and versions from the archive SBOM
jq '.components[] | {name, version}' obi-v${VERSION}-linux-${ARCH}.cyclonedx.json

# Scan the SBOM with Grype
grype sbom:obi-v${VERSION}-linux-${ARCH}.cyclonedx.json

# Inspect the Java agent dependency graph
jq '.components[] | {name, version}' obi-java-agent-v${VERSION}.cyclonedx.json
```

## 시스템에 설치하기 {#install-to-system}

아카이브를 추출한 후, 어느 디렉터리에서나 사용할 수 있도록 바이너리를 PATH에
있는 위치에 설치할 수 있다.

다음 예시는 대부분의 Linux 배포판에서 표준 위치인 `/usr/local/bin`에 설치한다.
PATH에 있는 다른 디렉터리에 설치해도 된다.

```bash
# Move binaries to a directory in your PATH
sudo cp obi /usr/local/bin/

# Verify installation
obi --version
```

## OBI 설정하기 {#set-up-obi}

1. [구성 옵션](../../configure/options/) 문서에 따라 구성 파일을 생성한다.
   [OBI 구성 YAML 예시](../../configure/example/)로 시작할 수 있다.

2. OBI를 특권(privileged) 프로세스로 실행한다.

   ```bash
   sudo obi --config=<path to config file>
   ```

   OBI를 PATH에 설치하지 않았다면, 추출한 디렉터리에서 실행할 수 있다.

   ```bash
   sudo ./obi --config=<path to config file>
   ```

## 권한 {#permissions}

OBI가 제대로 작동하려면 상승된 권한이 필요하다. 필요한 구체적인 capability에
대한 자세한 내용은 [보안 문서](../../security/)를 참고한다.
