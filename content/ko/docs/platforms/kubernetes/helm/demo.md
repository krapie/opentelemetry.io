---
title: 오픈텔레메트리 데모 차트
linkTitle: 데모 차트
default_lang_commit: c060ef7682b152a285d3a2f0c6a84c93ff877070
---

[오픈텔레메트리 데모](/docs/demo/)는 실제와 유사한 환경에서 오픈텔레메트리의
구현을 보여주기 위한 마이크로서비스 기반의 분산 시스템이다. 이 노력의 일환으로,
오픈텔레메트리 커뮤니티는 쿠버네티스에 쉽게 설치할 수 있도록
[오픈텔레메트리 데모 Helm 차트](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-demo)를
만들었다.

## 구성 {#configuration}

데모 Helm 차트의 기본 `values.yaml`은 바로 설치할 수 있는 상태이다. 모든
컴포넌트는 성능을 최적화하기 위해 메모리 제한이 조정되어 있으며, 클러스터가
충분히 크지 않으면 문제가 발생할 수 있다. 전체 설치는 약 4기가바이트의 메모리로
제한되지만, 그보다 적게 사용할 수도 있다.

차트에서 사용 가능한 모든 구성 옵션(주석 포함)은
[`values.yaml` 파일](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-demo/values.yaml)에서
확인할 수 있으며, 자세한 설명은
[차트의 README](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-demo#chart-parameters)에서
찾을 수 있다.

## 설치 {#installation}

오픈텔레메트리 Helm 저장소를 추가한다.

```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

릴리스 이름 `my-otel-demo`로 차트를 설치하려면 다음 명령을 실행한다.

```sh
helm install my-otel-demo open-telemetry/opentelemetry-demo
```

설치가 완료되면, 다음 명령을 실행해 Frontend 프록시(<http://localhost:8080>)를
통해 모든 서비스를 사용할 수 있다.

```sh
kubectl port-forward svc/my-otel-demo-frontendproxy 8080:8080
```

프록시가 노출되면 다음 경로에도 접속할 수 있다.

| 컴포넌트           | 경로                              |
| ------------------ | --------------------------------- |
| 웹 스토어          | <http://localhost:8080>           |
| Grafana            | <http://localhost:8080/grafana>   |
| Flagd 구성 도구 UI | <http://localhost:8080/feature>   |
| Jaeger UI          | <http://localhost:8080/jaeger/ui> |

웹 스토어의 스팬을 수집하려면 오픈텔레메트리 컬렉터의 OTLP/HTTP 리시버를
노출해야 한다.

```sh
kubectl port-forward svc/my-otel-demo-otelcol 4318:4318
```

쿠버네티스에서 데모를 사용하는 방법에 대한 자세한 내용은
[쿠버네티스 배포](/docs/demo/kubernetes-deployment/)를 참고한다.
