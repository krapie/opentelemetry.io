---
title: 소스에서 빌드하기
description:
  소스에서 오픈텔레메트리(OpenTelemetry) 컬렉터를 빌드하는 방법을 알아본다
weight: 100
default_lang_commit: 6a7f17450ce3edc2e4363013551ee93ba7934a5d
---

로컬 운영체제를 기반으로 다음 명령을 사용하여 최신 버전의 컬렉터를 빌드할 수
있다.

```sh
git clone https://github.com/open-telemetry/opentelemetry-collector.git
cd opentelemetry-collector
make install-tools
make otelcorecol
```
