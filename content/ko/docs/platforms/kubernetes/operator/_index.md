---
title: 쿠버네티스(Kubernetes)를 위한 오픈텔레메트리 오퍼레이터
linkTitle: 쿠버네티스 오퍼레이터
description:
  오픈텔레메트리(OpenTelemetry) 계측 라이브러리를 사용하여 컬렉터와 워크로드의
  자동 계측을 관리하는 쿠버네티스 오퍼레이터(Operator) 구현체이다.
aliases:
  - /docs/operator
  - /docs/k8s-operator
  - /docs/platforms/kubernetes-operator
redirects:
  - { from: /docs/operator/*, to: ':splat' }
  - { from: /docs/k8s-operator/*, to: ':splat' }
  - { from: /docs/platforms/kubernetes-operator/*, to: ':splat' }
default_lang_commit: 48d3ff356dc39a3b1323637f3163d435dc751228
---

## 소개 {#introduction}

[오픈텔레메트리(OpenTelemetry) 오퍼레이터](https://github.com/open-telemetry/opentelemetry-operator)는
[쿠버네티스 오퍼레이터](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
구현체이다.

오퍼레이터는 다음을 관리한다.

- [오픈텔레메트리 컬렉터](https://github.com/open-telemetry/opentelemetry-collector)
- [오픈텔레메트리 계측 라이브러리를 사용한 워크로드의 자동 계측](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/auto-instrumentation/README.md)

## 시작하기 {#getting-started}

기존 클러스터에 오퍼레이터를 설치하려면
[`cert-manager`](https://cert-manager.io/docs/installation/)가 설치되어 있는지
확인한 다음 다음을 실행한다.

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

`opentelemetry-operator` 디플로이먼트(deployment)가 준비되면, 다음과 같이
오픈텔레메트리 컬렉터(otelcol) 인스턴스를 생성한다.

```console
$ kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: simplest
spec:
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15

    exporters:
      # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
      debug: {}

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
EOF
```

> [!NOTE]
>
> 기본적으로 `opentelemetry-operator`는
> [`opentelemetry-collector` 이미지](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector)를
> 사용한다. [Helm 차트](/docs/platforms/kubernetes/helm/)를 사용해 오퍼레이터를
> 설치하면
> [`opentelemetry-collector-k8s` 이미지](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-k8s)가
> 사용된다. 이 릴리스에서 찾을 수 없는 구성 요소가 필요하다면
> [자체 컬렉터를 빌드](/docs/collector/extend/ocb/)해야 할 수도 있다.

오픈텔레메트리 계측 라이브러리를 사용한 워크로드의 자동 계측 주입을 설정하는
방법을 비롯한 더 많은 설정 옵션은
[쿠버네티스를 위한 오픈텔레메트리 오퍼레이터](https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md)를
참고한다.
