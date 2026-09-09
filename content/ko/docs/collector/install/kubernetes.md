---
title: Kubernetes로 컬렉터 설치하기
linkTitle: Kubernetes
weight: 200
default_lang_commit: 9f912d59a165ded5dec82d0e1a94c2aef54e5c57
---

다음 명령을 사용하여 오픈텔레메트리(OpenTelemetry) 컬렉터를 DaemonSet과 단일
게이트웨이(gateway) 인스턴스로 설치한다.

```sh
kubectl apply -f https://raw.githubusercontent.com/open-telemetry/opentelemetry-collector/v{{% param vers %}}/examples/k8s/otel-config.yaml
```

이 예제는 시작점으로 활용할 수 있다. 프로덕션 환경에 적합한 커스터마이징 및 설치
방법은 [오픈텔레메트리 Helm 차트][OpenTelemetry Helm Charts]를 참고한다.

또한 [오픈텔레메트리 오퍼레이터][OpenTelemetry Operator]를 사용하여
오픈텔레메트리 컬렉터 인스턴스를 프로비저닝하고 유지 관리할 수도 있다.
오퍼레이터에는 자동 업그레이드 처리, 오픈텔레메트리 구성을 기반으로 한 `Service`
구성, 배포에 대한 자동 사이드카(sidecar) 주입 등의 기능이 포함되어 있다.

Kubernetes와 함께 컬렉터를 사용하는 방법에 대한 안내는
[Kubernetes 시작하기](/docs/platforms/kubernetes/getting-started/)를 참고한다.

[opentelemetry helm charts]: /docs/platforms/kubernetes/helm/
[opentelemetry operator]: /docs/platforms/kubernetes/operator/
