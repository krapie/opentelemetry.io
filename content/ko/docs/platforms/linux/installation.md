---
title: 설치
weight: 10
description:
  오픈텔레메트리(OpenTelemetry) 패키지 저장소를 추가하고 Debian, Ubuntu, Fedora,
  RHEL 및 그 파생 배포판에 시스템 패키지를 설치한다.
cSpell:ignore: metapackage
default_lang_commit: edb244ceebdcbbb33c640eaac8d218dbc480e4c0
---

오픈텔레메트리 시스템 패키지는 Debian 계열 배포판을 위한 APT 저장소와 RPM 계열
배포판을 위한 YUM 저장소에 게시된다. 이 페이지에서는 저장소를 추가하고
`opentelemetry` 메타패키지(metapackage)를 설치하는 방법을 다룬다. 이
메타패키지는
[오픈텔레메트리 인젝터(Injector)](https://github.com/open-telemetry/opentelemetry-injector)와
Java, .NET, Node.js, Python용 자동 계측을 함께 설치한다.

> [!WARNING]
>
> 이 패키지는 아직 초기 단계이며 프로덕션 워크로드에 사용하도록 의도되지 않았다.
> 저장소는 GitHub Pages에서 호스팅되고 있으며 패키지는 아직 서명되지 않았으므로,
> 아래 안내에서는 서명 검증을 비활성화한다.
> [상태 및 제약 사항](../#status-and-limitations)을 참고한다.

## Debian, Ubuntu 및 파생 배포판 {#apt}

APT 저장소를 추가하고 패키지를 설치한다.

```sh
echo "deb [trusted=yes] https://open-telemetry.github.io/opentelemetry-packaging/debian stable main" |
  sudo tee /etc/apt/sources.list.d/opentelemetry.list
sudo apt update
sudo apt install opentelemetry
```

## Fedora, RHEL 및 파생 배포판 {#yum}

YUM 저장소를 추가하고 패키지를 설치한다.

```sh
cat <<EOF | sudo tee /etc/yum.repos.d/opentelemetry.repo
[opentelemetry]
name=OpenTelemetry Auto-Instrumentation System Packages
baseurl=https://open-telemetry.github.io/opentelemetry-packaging/rpm/packages
enabled=1
gpgcheck=0
EOF
sudo dnf install opentelemetry
```

## 설치 확인 {#verify-the-installation}

지원되는 언어로 작성된 애플리케이션을 재시작하거나 새로 시작하고, 구성한
대상으로 텔레메트리를 내보내는지 확인한다. [대상을 구성](../configuration/)하기
전까지는 텔레메트리가 OTLP를 사용해 `localhost`의 `4317`(gRPC) 및 `4318`(HTTP)
포트로 전송되므로, 데이터를 확인하려면 해당 포트에서 대기 중인
[컬렉터](/docs/collector/)나 다른 OTLP 리시버가 있어야 한다.

## 개별 언어 설치 {#install-individual-languages}

`opentelemetry` 메타패키지는 인젝터와 지원되는 모든 언어의 자동 계측을 함께
설치한다. 일부만 필요하다면 언어별 패키지를 개별적으로 설치할 수 있다.

- `opentelemetry-java`
- `opentelemetry-nodejs`
- `opentelemetry-dotnet`
- `opentelemetry-python`

## 다음 단계 {#next-steps}

- [구성](../configuration/): 컬렉터나 백엔드로 텔레메트리를 전송하고 계측 대상을
  조정한다.
