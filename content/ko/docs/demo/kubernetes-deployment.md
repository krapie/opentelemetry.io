---
title: 쿠버네티스 배포
linkTitle: 쿠버네티스
aliases: [kubernetes_deployment]
cSpell:ignore: otlphttp spanmetrics
default_lang_commit: c060ef7682b152a285d3a2f0c6a84c93ff877070
---

기존 쿠버네티스(Kubernetes) 클러스터에 데모를 배포하는 데 도움이 되는
[오픈텔레메트리 데모 Helm 차트](/docs/platforms/kubernetes/helm/demo/)를
제공한다.

차트를 사용하려면 [Helm](https://helm.sh)이 설치되어 있어야 한다. 시작하려면
Helm의 [문서](https://helm.sh/docs/)를 참고한다.

## 사전 요구 사항 {#prerequisites}

- 쿠버네티스 1.24+
- 애플리케이션을 위한 여유 RAM 6GB
- Helm 3.14+(Helm 설치 방법을 사용하는 경우에만 필요)

## Helm을 사용한 설치 {#install-using-helm}

오픈텔레메트리 Helm 저장소를 추가한다:

```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

릴리스 이름 my-otel-demo로 차트를 설치하려면 다음 명령어를 실행한다:

```shell
helm install my-otel-demo open-telemetry/opentelemetry-demo
```

<!-- markdownlint-disable no-blanks-blockquote -->

> [!NOTE]
>
> 오픈텔레메트리 데모 Helm 차트는 한 버전에서 다른 버전으로 업그레이드하는 것을
> 지원하지 않는다. 차트를 업그레이드해야 하는 경우, 먼저 기존 릴리스를 삭제한
> 다음 새 버전을 설치해야 한다.

> [!NOTE]
>
> 아래에 언급된 모든 사용 방법을 수행하려면 오픈텔레메트리 데모 Helm 차트 버전
> 0.11.0 이상이 필요하다.

### Helm을 사용하여 쿠버네티스 매니페스트 생성 {#use-helm-to-generate-a-kubernetes-manifests}

다음 명령어는 필요한 모든 리소스에 대한 정의를 포함하는 단일 쿠버네티스
매니페스트 파일을 생성한다. 생성한 후에는
`kubectl apply -f opentelemetry-demo.yaml`을 사용하여 이 매니페스트를 적용할 수
있다.

```shell
helm template opentelemetry-demo open-telemetry/opentelemetry-demo --namespace otel-demo > opentelemetry-demo.yaml
```

> [!NOTE]
>
> 오픈텔레메트리 데모 쿠버네티스 매니페스트는 한 버전에서 다른 버전으로
> 업그레이드하는 것을 지원하지 않는다. 데모를 업그레이드해야 하는 경우, 먼저
> 기존 리소스를 삭제한 다음 새 버전을 설치해야 한다.

## 데모 사용 {#use-the-demo}

데모 애플리케이션의 서비스를 사용하려면 쿠버네티스 클러스터 외부로 노출해야
한다. `kubectl port-forward` 명령어를 사용하거나, 서비스 타입(예:
`LoadBalancer`)을 구성하고 선택적으로 인그레스 리소스를 배포하여 서비스를 로컬
시스템에 노출할 수 있다.

### kubectl port-forward를 사용한 서비스 노출 {#expose-services-using-kubectl-port-forward}

frontend-proxy 서비스를 노출하려면 다음 명령어를 사용한다(`default`를 Helm 차트
릴리스의 네임스페이스에 맞게 교체한다):

```shell
kubectl --namespace default port-forward svc/frontend-proxy 8080:8080
```

> [!NOTE]
>
> `kubectl port-forward`는 프로세스가 종료될 때까지 포트를 프록시한다.
> `kubectl port-forward`를 사용할 때마다 별도의 터미널 세션을 생성해야 할 수
> 있으며, 작업이 끝나면 <kbd>Ctrl-C</kbd>를 사용하여 프로세스를 종료한다.

frontend-proxy port-forward를 설정하면 다음에 접속할 수 있다:

- 웹 스토어: <http://localhost:8080/>
- Grafana: <http://localhost:8080/grafana/>
- Jaeger UI: <http://localhost:8080/jaeger/ui/>
- Flagd 구성 도구 UI: <http://localhost:8080/feature>

### 서비스 또는 인그레스 구성을 사용한 데모 구성 요소 노출 {#expose-demo-components-using-service-or-ingress-configurations}

> [!NOTE]
>
> 추가 구성 옵션을 지정하려면 Helm 차트를 설치할 때 values 파일을 사용하는 것을
> 권장한다.

#### 인그레스 리소스 구성 {#configure-ingress-resources}

> [!NOTE]
>
> 쿠버네티스 클러스터에는 `LoadBalancer` 서비스 타입이나 인그레스 리소스를
> 활성화하는 데 필요한 인프라 구성 요소가 없을 수 있다. 이러한 구성 옵션을
> 사용하기 전에 클러스터가 이를 제대로 지원하는지 확인한다.

각 데모 구성 요소(예: frontend-proxy)는 자체 쿠버네티스 서비스 타입을 구성할 수
있는 방법을 제공한다. 기본적으로 이는 생성되지 않지만, 각 구성 요소의 `ingress`
속성을 통해 활성화하고 구성할 수 있다.

frontend-proxy 구성 요소가 인그레스 리소스를 사용하도록 구성하려면 values 파일에
다음을 지정한다:

```yaml
components:
  frontend-proxy:
    ingress:
      enabled: true
      annotations: {}
      hosts:
        - host: otel-demo.my-domain.com
          paths:
            - path: /
              pathType: Prefix
              port: 8080
```

일부 인그레스 컨트롤러는 특별한 애노테이션이나 서비스 타입을 필요로 한다. 자세한
내용은 사용 중인 인그레스 컨트롤러의 문서를 참고한다.

#### 서비스 타입 구성 {#configure-service-types}

각 데모 구성 요소(예: frontend-proxy)는 자체 쿠버네티스 서비스 타입을 구성할 수
있는 방법을 제공한다. 기본적으로 이는 `ClusterIP`이지만, 각 구성 요소의
`service.type` 속성을 사용하여 변경할 수 있다.

frontend-proxy 구성 요소가 `LoadBalancer` 서비스 타입을 사용하도록 구성하려면
values 파일에 다음을 지정한다:

```yaml
components:
  frontend-proxy:
    service:
      type: LoadBalancer
```

#### 브라우저 텔레메트리 구성 {#configure-browser-telemetry}

브라우저에서 발생하는 스팬이 올바르게 수집(collect)되려면 오픈텔레메트리
컬렉터가 노출되는 위치도 지정해야 한다. frontend-proxy는 `/otlp-http` 경로
접두사(path prefix)로 컬렉터에 대한 라우트를 정의한다. frontend 구성 요소에 다음
환경 변수를 설정하여 컬렉터 엔드포인트를 구성할 수 있다:

```yaml
components:
  frontend:
    envOverrides:
      - name: PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
        value: http://otel-demo.my-domain.com/otlp-http/v1/traces
```

## 자체 백엔드 사용하기 {#bring-your-own-backend}

이미 보유하고 있는 옵저버빌리티 백엔드(예: 기존 Jaeger나 Zipkin 인스턴스, 또는
원하는 [벤더](/ecosystem/vendors/) 중 하나)를 위한 데모 애플리케이션으로 웹
스토어를 사용하고 싶을 수 있다.

오픈텔레메트리 컬렉터의 구성은 Helm 차트에 노출되어 있다. 추가하는 내용은 기본
구성에 병합된다.

사용자 지정 파일(예: `my-values-file.yaml`)을 생성하여 원하는 파이프라인에 자체
익스포터를 추가하는 데 사용할 수 있다:

```yaml
opentelemetry-collector:
  config:
    exporters:
      otlphttp/example:
        endpoint: <your-endpoint-url>

    service:
      pipelines:
        traces:
          exporters: [spanmetrics, otlphttp/example]
```

> [!NOTE]
>
> Helm에서 YAML 값을 병합할 때 객체는 병합되고 배열은 교체된다. `traces`
> 파이프라인을 재정의하는 경우 익스포터 배열에 `spanmetrics` 익스포터를 반드시
> 포함해야 한다. 이 익스포터를 포함하지 않으면 오류가 발생한다.

벤더 백엔드는 인증(authentication)을 위한 추가 파라미터를 요구할 수 있으므로
해당 문서를 확인한다. 일부 백엔드는 다른 익스포터를 필요로 하며, 이러한
익스포터와 관련 문서는
[opentelemetry-collector-contrib/exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter)에서
찾을 수 있다.

사용자 지정 `my-values-file.yaml` values 파일로 Helm 차트를 설치하려면 다음을
사용한다:

```shell
helm install my-otel-demo open-telemetry/opentelemetry-demo --values my-values-file.yaml
```
