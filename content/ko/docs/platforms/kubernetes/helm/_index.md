---
title: 오픈텔레메트리 Helm 차트
linkTitle: Helm 차트
default_lang_commit: 99a39c5e4e51daba968bfbb3eb078be4a14ad363
---

## 소개 {#introduction}

[Helm](https://helm.sh/)은 쿠버네티스(Kubernetes) 애플리케이션을 관리하기 위한
CLI 솔루션이다.

Helm을 사용하기로 했다면,
[오픈텔레메트리(OpenTelemetry) Helm 차트](https://github.com/open-telemetry/opentelemetry-helm-charts)를
사용해 [오픈텔레메트리 컬렉터](/docs/collector),
[오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator),
[오픈텔레메트리 데모](/docs/demo)의 설치를 관리할 수 있다.

다음 명령으로 오픈텔레메트리 Helm 저장소를 추가한다.

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```
