---
title: 컬렉터 설치하기
linkTitle: 설치
aliases: [installation]
weight: 2
default_lang_commit: 9f912d59a165ded5dec82d0e1a94c2aef54e5c57
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 다양한 운영체제와 아키텍처에 배포할 수
있다. 다음 안내에서는 사용 중인 환경에 맞는 최신 안정 버전의 컬렉터를
다운로드하고 설치하는 방법을 설명한다.

시작하기 전에 [배포 패턴][deployment patterns], [구성 요소][components],
[구성][configuration]을 포함한 컬렉터의 기본 개념을 이해하고 있는지 확인한다.

## 소스에서 빌드하기 {#build-from-source}

로컬 운영체제를 기반으로 다음 명령을 사용하여 최신 버전의 컬렉터를 빌드할 수
있다.

```sh
git clone https://github.com/open-telemetry/opentelemetry-collector.git
cd opentelemetry-collector
make install-tools
make otelcorecol
```

[deployment patterns]: /docs/collector/deploy/
[components]: /docs/collector/components/
[configuration]: /docs/collector/configuration/
