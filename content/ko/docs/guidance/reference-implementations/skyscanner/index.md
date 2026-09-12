---
title: 'Skyscanner: 24개의 프로덕션 클러스터에 걸친 오픈텔레메트리 컬렉터 관리'
linkTitle: Skyscanner
# prettier-ignore
cSpell:ignore: Fordyce kube kubelet rollouts Skyscanner Sloughter unsets Öjeling
default_lang_commit: 11753c0e99bbc1b62606d4c819736c777dfb0e98
---

글: [Johanna Öjeling](https://github.com/johannaojeling) (Grafana Labs),
[Juliano Costa](https://github.com/julianocosta89) (Datadog),
[Tristan Sloughter](https://github.com/tsloughter) (커뮤니티),
[Neil Fordyce](https://github.com/neilfordyce) (Skyscanner) | 2026년 4월 21일

이 레퍼런스 구현은 스코틀랜드 에든버러에 본사를 둔 글로벌 여행 검색 플랫폼인
[Skyscanner](https://www.skyscanner.net/)가 어떻게 대규모로 오픈텔레메트리를
운영하는지 설명한다.

전 세계 1,400명의 직원이 24개의 프로덕션 쿠버네티스 클러스터에 걸쳐 1,000개가
넘는 마이크로서비스를 운영하는 가운데, Skyscanner의 오픈텔레메트리 여정은 규모
있게 운영하는 조직에게 값진 교훈을 준다.

## 조직 구조 {#organizational-structure}

6명의 플랫폼 엔지니어로 구성된 Hubble 팀이 Skyscanner의 대부분의 컬렉터를
관리한다. 더 넓은 플랫폼 엔지니어링 조직의 일부로서, 이들은 Skyscanner의 주로
자바 기반인 마이크로서비스 아키텍처를 실행하는 컴퓨트 플랫폼을 담당한다.

서비스 팀 자신은 배포 및 텔레메트리 수집 인프라로부터 추상화되어 있다. 자바
서비스의 경우, 팀은 사전 구성된 오픈텔레메트리 자바 에이전트가 포함된 기본
Docker 이미지를 물려받는다. Python과 Node.js 서비스의 경우, 플랫폼 팀은 환경과
리소스 속성을 기반으로 합리적인 기본값을 설정하는 래퍼 라이브러리를 제공한다.
이러한 접근 방식은 상용구(boilerplate) 설정을 최소화하고, 서비스 팀이 깊은
오픈텔레메트리 지식 없이도 즉시 옵저버빌리티를 확보할 수 있게 해준다.

## 오픈텔레메트리 도입 {#opentelemetry-adoption}

Skyscanner의 오픈텔레메트리 여정은 2021년에 시작되었다. 회사는 내부적으로 구축한
오픈 소스 스택에서 상용 벤더로 마이그레이션하고 있었다. 하지만 벤더 락인은
피하고 싶었다.

> "우리는 벤더 중립적인 방식으로 벤더로 이전하고 싶었다"라고 Skyscanner의 Hubble
> 플랫폼 팀 소프트웨어 엔지니어인
> [Neil Fordyce](https://github.com/neilfordyce)는 설명했다.

이러한 벤더 중립적 접근 방식은 팀이 오픈텔레메트리 컬렉터를 텔레메트리 인프라의
중심축으로 채택하도록 이끌었다.

## 아키텍처: 중앙 집중식 라우팅, 분산 수집 {#architecture-centralized-routing-distributed-collection}

Skyscanner의 컬렉터 아키텍처는 Istio 기반의 지능형 라우팅을 갖춘 중앙 DNS
엔드포인트를 특징으로 한다. 서비스가 전 세계 어디서 실행되든, 어느 클러스터에
있든 상관없이 이 단일 주소로 텔레메트리를 전송한다. Istio는 가장 가까운 사용
가능한 컬렉터로 요청을 라우팅하는 것을 처리한다.

배포는 두 가지 뚜렷한 컬렉터 패턴으로 구성된다.

**게이트웨이 컬렉터(레플리카 세트)**: 대부분의 서비스로부터 대량의 OTLP
트래픽(트레이스와 메트릭)을 처리하며, 대부분의 처리가 여기서 이루어진다.

**에이전트 컬렉터(DaemonSet)**: 아직 OTLP를 네이티브로 지원하지 않는 오픈 소스
및 플랫폼 서비스로부터 Prometheus 엔드포인트를 스크레이핑한다.

![Skyscanner 아키텍처 다이어그램](skyscanner-architecture.png)

## 구성: 단순하게 시작해서 점진적으로 발전시키기 {#configuration-start-simple-evolve-gradually}

Skyscanner가 2021년 처음 컬렉터를 배포했을 때, 구성은 최소한이었다. 메모리
제한기, 배치 프로세서, 트레이스를 위한 OTLP 익스포터뿐이었다.

시간이 지나면서 구성은 유기적으로 발전했다. 메트릭 파이프라인 추가, Istio 스팬
인제스트 통합, 스팬-투-메트릭(span-to-metrics) 변환 구현, 노이즈를 줄이고 비용을
통제하기 위한 필터 프로세서 추가 등이 이루어졌다.

### Istio 서비스 메시 스팬을 플랫폼 메트릭으로 전환하기 {#turning-istio-service-mesh-spans-into-platform-metrics}

Skyscanner가 컬렉터를 사용한 가장 혁신적인 방식 중 하나는 Istio 서비스 메시
스팬으로부터 메트릭을 생성하는 것이다.

Istio의 네이티브 메트릭은 Prometheus 배포를 압도할 수 있는 카디널리티
폭발(cardinality explosion) 문제를 겪고 있었다. 또한 Skyscanner는 코드를
소유하지 않으면서도 일관된 메트릭이 필요한 다수의 기성(off-the-shelf) 서비스를
운영한다.

이들의 해결책은 다음과 같다. Istio가 (원래는 Zipkin 형식이었지만, 현재 Istio는
OTLP도 지원한다) 스팬을 내보내도록 구성하고, Zipkin 리시버로 컬렉터를 통해 이를
인제스트하고, 시맨틱 컨벤션을 충족하도록 변환한 다음,
[스팬 메트릭 커넥터](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/e8a502371ea1d2c3534235d623c1b1eb3b6b4b58/connector/spanmetricsconnector?from_branch=main)
를 사용해 애플리케이션 계측 없이도 일관된 메트릭을 생성한다.

> "애플리케이션 소유자가 코드를 전혀 계측하지 않고도 플랫폼 수준에서 그렇게 할
> 수 있다"라고 Neil은 언급했다.

스팬 메트릭 커넥터 구성은 스팬에서 핵심 차원(dimension)을 추출한다.

```yaml
connectors:
  spanmetrics:
    aggregation_temporality: AGGREGATION_TEMPORALITY_DELTA
    dimensions:
      - name: http.status_code
      - name: grpc.status_code
      - name: rpc.service
      - name: rpc.method
      - name: prot
      - name: flag
      - name: k8s.deployment.name
      - name: k8s.replicaset.name
      - name: destination_subset
    dimensions_cache_size: 15000000
    histogram:
      exponential:
        max_size: 160
      unit: ms
    metrics_flush_interval: 30s
```

이후 컬렉터는 이러한 메트릭을 `http.client.duration`, `http.server.duration`
같은 시맨틱 컨벤션 이름으로 변환하고, 클러스터, 서비스 이름, HTTP 상태 코드별로
집계한다. 이는 코드 변경 없이 모든 서비스에 대해 플랫폼 수준의 HTTP 메트릭을
제공하며, 시맨틱 컨벤션을 준수하는 일관된 네이밍과 네이티브 Istio 메트릭보다
낮은 카디널리티를 제공한다.

### 404 오류 문제 {#the-404-error-challenge}

컬렉터 구성과 관련한 한 가지 주목할 만한 과제는, 캐시에 항목이 존재하지 않음을
나타내기 위해 HTTP 404를 반환하는 캐시 서비스와 관련이 있었다. 컬렉터는 이러한
404를 오류로 취급해, 실제로는 정상적인 고빈도 동작에 대해 100% 트레이스 샘플링을
유발했다.

해결책은 이러한 특정 404 응답에 대해 오류 상태를 해제(unset)하는 필터 프로세서를
추가하는 것이었다.

```yaml
processors:
  span/unset_cache_client_404:
    include:
      attributes:
        - key: http.response.status_code
          value: ^404$
        - key: server.address
          value: ^(service-x\.skyscanner\.net|service-y\.skyscanner\.net|service-z\.skyscanner\.net|service-z-\w{2}-\w+-\d\.int\.\w{2}-\w+-\d\.skyscanner\.com)$
      match_type: regexp
      regexp:
        cacheenabled: true
        cachemaxnumentries: 1000
    status:
      code: Unset
```

이 프로세서는 특정 캐시 서비스로부터 온 404 상태 코드를 가진 스팬을 매칭해 오류
상태를 해제함으로써, 오류 기반 샘플링을 유발하지 않도록 한다.

> "처음부터 이 필터 프로세서가 있었다면 더 고품질이고 더 사용하기 쉬운
> 트레이스를 얻었을 것이다"라고 Neil은 회고했다.

다만 Neil은 최근 도입된 오픈텔레메트리 SDK
[선언적 구성(declarative configuration)](/docs/languages/sdk-configuration/declarative-configuration/)
덕분에, 이제는 중앙 컬렉터 구성을 변경할 필요 없이 서비스 팀 스스로가 분산된
방식으로 이러한 필터링을 구성할 수 있다고 언급했다.

### 구성 심층 분석 {#configuration-deep-dive}

Skyscanner는 다른 사람들이 이러한 패턴을 실제로 이해하도록 돕기 위해 자신들의
프로덕션 컬렉터 구성을 공개했다.

#### 게이트웨이 컬렉터 {#gateway-collector}

[게이트웨이 컬렉터][gateway-otelbin]는 대부분의 처리를 담당한다.

- 서비스로부터 OTLP 메트릭과 트레이스를, Istio로부터 Zipkin 스팬을 수신한다
- 스팬 메트릭 커넥터를 사용해 Istio 스팬으로부터 메트릭을 생성한다
- Istio 속성을 시맨틱 컨벤션에 매핑하기 위해 방대한 변환 프로세서를 사용한다
- 캐시 서비스를 위한 404 필터링 로직을 구현한다
- 메트릭과 트레이스를 OTLP를 통해 옵저버빌리티 벤더로 내보낸다

아래 다이어그램은 OTLP 메트릭과 트레이스, 그리고 Istio 스팬이 이러한 게이트웨이
컬렉터에 어떻게 도달하는지 보여준다.

![Skyscanner 아키텍처(게이트웨이 컬렉터) 다이어그램](skyscanner-architecture-gateway.png)

#### 에이전트 컬렉터 {#agent-collector}

[에이전트 컬렉터][agent-otelbin]는 각 노드로부터 인프라 및 플랫폼 수준 메트릭을
수집하는 데 집중한다.

- 다양한 소스(node exporter, kube-state-metrics, kubelet)로부터 Prometheus
  엔드포인트를 스크레이핑한다
- 최소한의 처리(메모리 제한, 배칭, 속성 정리)를 수행한다
- 메트릭을 OTLP를 통해 옵저버빌리티 벤더로 내보낸다

## 계측 전략 {#instrumentation-strategy}

Skyscanner의 자바 중심 환경은 오픈텔레메트리의 자동 계측 기능으로부터 상당한
이점을 얻는다. 기본 Docker 이미지에 사전 구성된 자바 에이전트는 즉시 HTTP와 gRPC
스팬 생성을 제공한다.

### 주관이 뚜렷한 자동 계측 {#opinionated-auto-instrumentation}

팀은 자동 계측에 대해 의도적으로 주관이 뚜렷한 접근 방식을 취한다. 모든 것을
기본으로 활성화하는 대신, 정반대 방향에서 출발한다. 공유 기본 Docker
이미지에서는 모든 계측이 비활성화되어 있고, 엄선된 일부만 명시적으로 활성화된다.

> "말하자면 반대 방향이다. 모두 비활성화한 다음, 필요한 것만 활성화한다"라고
> Neil은 설명했다.

기본 이미지에서 환경 변수를 사용해, Skyscanner는 런타임, HTTP, gRPC 관련 계측 중
엄선된 일부를 기본으로 활성화한다. 여기에는 JAX-RS, gRPC, Jetty, 일반적인 HTTP
클라이언트, 실행자(executor) 계측, 로깅 컨텍스트 전파가 포함된다. 서비스 팀은
이러한 기본값을 자동으로 물려받지만, 필요하다면 자신의 서비스 정의에서 이를
오버라이드하거나 추가 계측을 활성화할 자유가 있다.

이 모델은 가장자리에서의 유연성을 허용하면서도 수백 개의 서비스 전반에서
일관성을 보장한다.

### 자바 에이전트 설정하기 {#setting-up-the-java-agent}

아래 스니펫은 공유 자바 기본 이미지를 보여주는 예시이다. 오픈텔레메트리 자바
에이전트를 이미지에 번들링하고, 조직 전체의 기본값을 설정하며, 공통 런처
스크립트를 설치한다.

```Dockerfile base image
# Image used as source for the OpenTelemetry Java agent
FROM ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:2.25.0 AS otel

# Define a common base image for all Java microservices to extend
FROM image/registry/public-java-image:x.y.z

# Copy OpenTelemetry Java agent from OTel image
COPY --from=otel /javaagent.jar $OPEN_TELEMETRY_DIRECTORY/opentelemetry-javaagent.jar
ENV OTEL_AGENT=$OPEN_TELEMETRY_DIRECTORY/opentelemetry-javaagent.jar

# Pick sensible defaults for everyone in the org
ENV OTEL_METRICS_EXPORTER="otlp"
ENV OTEL_TRACES_EXPORTER="otlp"
ENV OTEL_LOGS_EXPORTER="none"
ENV OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE="DELTA"
ENV OTEL_EXPERIMENTAL_METRICS_VIEW_CONFIG="otel-view.yaml"
ENV OTEL_EXPORTER_OTLP_ENDPOINT="http://otel.skyscanner.net"
ENV OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED="false"
ENV OTEL_INSTRUMENTATION_RUNTIME_TELEMETRY_ENABLED="true"
ENV OTEL_INSTRUMENTATION_ASYNC_HTTP_CLIENT_ENABLED="true"
ENV OTEL_INSTRUMENTATION_APACHE_HTTPCLIENT_ENABLED="true"

COPY run.sh /usr/bin/run.sh
```

런처 스크립트 `run.sh`는 배포 시스템이 제공하는 환경 변수로부터 `-javaagent`
플래그와 `otel.resource.attributes`를 구성한다.

```bash run.sh
# We use this to setup OTel resource attributes
# for things we can detect from environment variables on service launch
# These vars are set automatically by our deployment system
# Some env vars have been omitted to avoid repetition
setup_otel_agent() {
    if [[ -n "$AWS_REGION" ]]; then CLOUD_REGION="cloud.region=${AWS_REGION},"; else CLOUD_REGION=""; fi
    if [[ -n "$AWS_ACCOUNT" ]]; then CLOUD_ACCOUNT_ID="cloud.account.id=${AWS_ACCOUNT},"; else CLOUD_ACCOUNT_ID=""; fi
    if [[ -n "$CLUSTER_NAME" ]]; then K8S_CLUSTER_NAME="k8s.cluster.name=${CLUSTER_NAME},"; else K8S_CLUSTER_NAME=""; fi
    if [[ -n "$SERVICE" ]]; then SERVICE_NAME="service.name=${SERVICE}"; else SERVICE_NAME=""; fi
    echo -n "-javaagent:$OTEL_AGENT" \
        "-Dotel.resource.attributes=${CLOUD_REGION}${CLOUD_ACCOUNT_ID}${K8S_CLUSTER_NAME}${SERVICE_NAME}"
}

JAVA_OPTS="-D64 -server -showversion $(setup_otel_agent) ${ADDITIONAL_JAVA_OPTS:-}"

exec java $JAVA_OPTS "$@"
```

마지막으로, 개별 서비스 Dockerfile은 동일한 기본 이미지를 확장하고 해당 서비스에
필요한 추가 계측만 더한다.

```Dockerfile my-service
FROM image/registry/skyscanner-java-base:x.y.z

COPY my-service.jar

# It's easy to extend if my-service wants to enable some other non-default instrumentation
ENV OTEL_INSTRUMENTATION_OPENAI_ENABLED=true
ENV OTEL_INSTRUMENTATION_OKHTTP_ENABLED=true

CMD exec /usr/bin/run.sh -jar my-service.jar server
```

### 스팬은 예, 메트릭은 아니오(기본값 기준) {#spans-yes-metrics-no-by-default}

Skyscanner 전략에서 특히 흥미로운 측면은 메트릭과 트레이스를 다루는 방식이다.
HTTP와 gRPC 계측이 활성화되어 있음에도, 팀은 의도적으로 대부분의 SDK 생성 HTTP
및 RPC 메트릭을 드롭한다. 앞서 설명했듯이 이미 Istio 서비스 메시 스팬으로부터
일관되고 카디널리티가 낮은 플랫폼 메트릭을 얻고 있기 때문이다.

계측을 완전히 비활성화하면 스팬까지 제거되므로, 대신 트레이싱은 유지하면서
메트릭 집계만 드롭하기 위해 오픈텔레메트리 SDK 뷰(view)를 사용한다.

- HTTP와 RPC 메트릭은 전역적으로 드롭된다
- 스팬은 평소처럼 계속 내보내진다
- 서비스 팀은 Istio가 제공하는 것보다 더 세밀한 정보가 필요하다면 특정
  메트릭(예: 서버 측 지연 시간)을 선택적으로 다시 활성화할 수 있다

팀이 SDK 메트릭을 다시 활성화하기로 할 때는, 기존 Istio 파생 메트릭과의 충돌이나
이중 집계를 피하기 위해 이름을 바꾸는 경우가 많다.

앞서 나온 [자바 기본 이미지](#setting-up-the-java-agent)에서
`OTEL_EXPERIMENTAL_METRICS_VIEW_CONFIG`는
[뷰 파일 구성](https://github.com/open-telemetry/opentelemetry-java/blob/65f7412a986cb474314b093c1bbba77955b52031/sdk-extensions/incubator/README.md#view-file-configuration)을
사용하는 Skyscanner의 기본 `otel-view.yaml`을 가리킨다.

```yaml
# Default Skyscanner metrics view config
# Stored in a file which OTEL_EXPERIMENTAL_METRICS_VIEW_CONFIG points to
# Drop http and rpc metrics, because we have metrics from Istio already
# We still want tracing to work so we wouldn't just disable the instrumentation
- selector:
    instrument_name: http.*
  view:
    aggregation: drop
- selector:
    instrument_name: rpc.*
  view:
    aggregation: drop
```

동일한 파일은 서비스가 특정 메트릭을 유지해야 할 때 확장될 수 있다. 대표적인
사용 사례는 `http.route`별로 요청을 세분화하는 것이다.

```yaml
# This dropping behaviour can be altered by extending the list to add more views
# to explicitly select the metrics to be kept.
# e.g. to keep http.server.request.duration metrics,
# but continue to drop http.client.* metrics
- selector:
    instrument_name: http.server.request.duration
  view:
    # renamed because we already have Istio metrics named http.server.request.duration,
    # so don't want to clash and double count
    name: app.http.server.request.duration
    attribute_keys:
      - http.request.method
      - http.route
      - http.response.status_code
```

이 접근 방식은 서비스 소유자가 오픈텔레메트리 내부 구조를 깊이 이해할 필요
없이도, Skyscanner가 높은 가치를 지닌 분산 트레이스를 유지하고, 메트릭 중복을
피하고, 카디널리티를 통제하고, 인제스트 비용을 줄일 수 있게 해준다.

전반적으로 이 전략은 강력한 플랫폼 사고방식을 반영한다. 규모에 맞게 작동하는
합리적인 기본값을 제공하고, 노이즈를 최소화하며, "올바른 일"을 쉬운 일로
만들면서도, 고급 요구가 있는 팀이 더 나아갈 수 있는 여지를 남겨둔다.

## 배포와 릴리스 관리 {#deployment-and-release-management}

Skyscanner는 필요한 모든 것이 포함되어 있다는 이유로 채택한 오픈텔레메트리
컬렉터 Contrib 배포판을 사용한다. 팀은 Contrib이 프로덕션 사용에 권장되지
않는다는 사실을 알게 되었고, 필요한 컴포넌트만 포함하는 커스텀 컬렉터 이미지를
빌드하는 방안을 검토할 계획이다.

Skyscanner는 대략 6개월마다 컬렉터를 업데이트하지만, 특정 기능이나 중요한 수정
사항을 추적하는 경우 더 자주 업그레이드한다. 릴리스 정보를 파악하기 위해 RSS
피드와 CNCF Slack 채널을 팔로우한다.

이들의 롤아웃 전략은 클러스터 계층 전반에 걸친 점진적 승격을 사용한다. 개발
클러스터, 이어서 3개의 알파 프로덕션 클러스터, 그다음 8개의 베타 프로덕션
클러스터, 마지막으로 나머지 13개의 프로덕션 클러스터 순이다. 배포에 Argo CD를
사용하며, 변경 사항은 풀 리퀘스트를 통해 계층 간에 승격된다.

> "개발 테스트 클러스터에서 확실히 문제를 일으킨 적이 있고, 이를 더 진행하기
> 전에 고친 적도 있다"라고 Neil은 말했다.

이러한 점진적인 접근 방식은 구성 문제가 프로덕션에 도달하기 전에 이를 포착해
왔다. 아직 오픈텔레메트리 컬렉터 배포를 위한 자동화된 테스트와 롤백 기능은
갖추지 못했지만, 이러한 개선 사항은 곧 도입될 예정이다.

## 잘 작동하는 것들 {#what-works-well}

프로덕션에서 오픈텔레메트리를 도입한 이후, 팀의 경험은 매우 긍정적이었다.

> "정말이지 큰 고통 없이 진행됐다"라고 Neil은 회고했다.

유연성이 컬렉터의 가장 큰 강점으로 두드러진다.

> "우리가 하고자 했던 모든 것을 실제로 실현할 수 있었다. 그것이 얼마나
> 유연한지를 말해준다고 생각한다"라고 Neil은 설명했다.

그 밖의 성과로는 OTLP 프로토콜이 간단한 구성만으로 벤더 독립성을 제공한다는 점,
명확하고 잘 정리된 릴리스 노트, 팀원들이 컬렉터 컴포넌트의 메모리 누수를
발견하고 수정 사항을 기여했을 때의 커뮤니티의 신속한 대응이 있다.

## 배운 점과 어려웠던 점 {#lessons-and-pain-points}

Skyscanner는 일부 파이프라인에서 여전히 오래되고 불안정한 HTTP 시맨틱 컨벤션을
사용하고 있다. 업그레이드하려면 Istio 속성을 시맨틱 컨벤션 이름에 매핑하는 여러
변환 프로세서 규칙을 업데이트해야 하는데, 이는 문서를 수작업으로 교차 참조하고
구성 문자열을 채워 넣는 작업을 수반한다.

팀은 시맨틱 컨벤션 관리를 위한
[Weaver](https://github.com/open-telemetry/weaver) 를 알고 있지만 아직 자신들의
워크플로에 통합하지는 않았다.

6개월마다 업그레이드한다는 것은 한 번에 여러 호환성이 깨지는 변경을 마주하게
된다는 것을 의미한다. 릴리스 노트가 잘 작성되어 변경 사항을 명확하게 문서화하고
있지만, 6개월치 업데이트를 한 번에 검토하는 것은 릴리스마다 꾸준히 따라가는
것보다 더 큰 마찰을 더한다.

## 다른 조직을 위한 조언 {#advice-for-others}

프로덕션 경험을 바탕으로 Skyscanner 팀은 다음과 같은 조언을 제시한다.

- **단순하게 시작한다**: 메모리 제한기, 배치 프로세서, 기본 익스포터만으로
  시작한다. 필요할 때만 복잡성을 추가한다.
- **첫날부터 메모리 제한기를 설정한다**: 규모가 커짐에 따라 메모리 문제를
  방지하기 위해 이를 즉시 설정한다.
- **필터 프로세서를 일찍 고려한다**: 애플리케이션의 상태 코드 시맨틱을 이해하고,
  비용을 통제하기 위해 대용량의 "가양성(false positive)"을 필터링한다.
- **복원력을 과도하게 설계하지 않는다**: 텔레메트리 데이터에는 단순한 인메모리
  배칭으로 충분한 경우가 많다.
- **점진적 롤아웃이 문제를 포착한다**: 환경 계층 전반의 점진적 승격은 값진
  검증을 제공한다.

## 핵심 요약 {#takeaways}

Skyscanner의 사례는 적당한 규모의 플랫폼 엔지니어링 팀도 비교적 낮은 운영
부담으로 상당한 규모의 오픈텔레메트리 컬렉터 인프라를 성공적으로 관리할 수
있음을 보여준다.

[gateway-otelbin]:
  https://www.otelbin.io/#config=connectors%3A*N__spanmetrics%3A*N____aggregation*_temporality%3A_AGGREGATION*_TEMPORALITY*_DELTA*N____dimensions%3A*N____-_name%3A_http.status*_code*N____-_name%3A_grpc.status*_code*N____-_name%3A_rpc.service*N____-_name%3A_rpc.method*N____-_name%3A_prot*N____-_name%3A_flag*N____-_name%3A_k8s.deployment.name*N____-_name%3A_k8s.replicaset.name*N____-_name%3A_destination*_subset*N____dimensions*_cache*_size%3A_15000000*N____histogram%3A*N______exponential%3A*N________max*_size%3A_160*N______unit%3A_ms*N____metrics*_flush*_interval%3A_30s*Nexporters%3A*N__debug%3A*N____verbosity%3A_normal*N__debug%2Fbasic%3A*N____verbosity%3A_basic*N__debug%2Fdetailed%3A*N____verbosity%3A_detailed*N__otlphttp%3A*N____endpoint%3A_https%3A%2F%2Fotlp-trace-sampler.vendor.com*N__otlphttp%2Fmetrics%3A*N____endpoint%3A_https%3A%2F%2Fotlp.vendor.com*N__otlphttp%2Fspanmetrics%3A*N____endpoint%3A_https%3A%2F%2Fotlp.vendor.com*Nextensions%3A*N__health*_check%3A*N____endpoint%3A_*S%7Benv%3AMY*_POD*_IP%7D%3A13133*N__pprof%3A*N____endpoint%3A_%3A1888*Nprocessors%3A*N__attributes%2Fspanmetrics-prep%3A*N____actions%3A*N____-_action%3A_extract*N______key%3A_response*_flags*N______pattern%3A_*C*QP*Lflag*G.***D*N____-_action%3A_extract*N______key%3A_dest*N______pattern%3A_%5E*C*QP*Lip*G*C*QP*Lip*_8*G*Bd*P*D*B.*C*QP*Lip*_16*G*Bd*P*D*B.*Bd*P*B.*Bd*P*D*S*N____-_action%3A_extract*N______key%3A_grpc.path*N______pattern%3A_%5E*B%2F*C*QP*Lrpc*_service*G%5B%5E*B%2F%5D*P*D*B%2F*C*QP*Lrpc*_method*G%5B%5E*B%2F%5D*P*D*S*N____-_action%3A_upsert*N______from*_attribute%3A_rpc*_service*N______key%3A_rpc.service*N____-_action%3A_delete*N______key%3A_rpc*_service*N____-_action%3A_upsert*N______from*_attribute%3A_rpc*_method*N______key%3A_rpc.method*N____-_action%3A_delete*N______key%3A_rpc*_method*N____-_action%3A_extract*N______key%3A_upstream*_cluster*N______pattern%3A_%5Eoutbound*B%7C*Bd*P*B%7C*C*QP*Ldestination*_subset*G%5B%5E%7C%5D***D*B%7C.***S*N__attributes%2Fspanmetrics-split-service-name%3A*N____actions%3A*N____-_action%3A_extract*N______key%3A_service.name*N______pattern%3A_%5E*C*QP*Lsvc*G%5B%5E*B.%5D*P*D*C*QP*Lns*_suffix*G*B.*C*QP*Lns*G%5B%5E*B.%5D*P*D*D*Q*S*N__batch%3A*N____send*_batch*_max*_size%3A_500*N____send*_batch*_size%3A_500*N__batch%2Fspanmetrics-export%3A*N____send*_batch*_max*_size%3A_512*N____send*_batch*_size%3A_512*N__batch%2Fspanmetrics-prep%3A_%7B%7D*N__filter%2Fspanmetrics%3A*N____metrics%3A*N______include%3A*N________match*_type%3A_strict*N________metric*_names%3A*N________-_traces.span.metrics.duration*N__filter%2Fspanmetrics-duration%3A*N____metrics%3A*N______include%3A*N________match*_type%3A_regexp*N________metric*_names%3A*N________-_*Crpc%7Chttp*D*B.*Cclient%7Cserver*D*B.duration*N__filter%2Fspanmetrics-prep%3A*N____spans%3A*N______include%3A*N________attributes%3A*N________-_key%3A_component*N__________value%3A_proxy*N________match*_type%3A_strict*N__groupbyattrs%2Fgroup-by-ip%3A*N____keys%3A*N____-_net.host.ip*N__k8sattributes%2Ffrom-pod-ip%3A*N____auth*_type%3A_serviceAccount*N____extract%3A*N______metadata%3A*N______-_k8s.deployment.name*N______-_k8s.replicaset.name*N____passthrough%3A_false*N____pod*_association%3A*N____-_sources%3A*N______-_from%3A_resource*_attribute*N________name%3A_k8s.pod.ip*N__memory*_limiter%3A*N____check*_interval%3A_1s*N____limit*_percentage%3A_100*N____spike*_limit*_percentage%3A_1*N__memory*_limiter%2Fmetrics%3A*N____check*_interval%3A_1s*N____limit*_percentage%3A_85*N____spike*_limit*_percentage%3A_10*N__memory*_limiter%2Fspanmetrics%3A*N____check*_interval%3A_1s*N____limit*_percentage%3A_75*N____spike*_limit*_percentage%3A_1*N__metricstransform%2Fspanmetrics-semantic-naming%3A*N____transforms%3A*N____-_action%3A_insert*N______experimental*_match*_labels%3A*N________prot%3A_http*N________span.kind%3A_SPAN*_KIND*_CLIENT*N______include%3A_traces.span.metrics.duration*N______match*_type%3A_strict*N______new*_name%3A_http.client.duration*N______operations%3A*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.name*N________new*_value%3A_actual*_cluster*_name*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.internal*_name*N________new*_value%3A_internal*_cluster*_name*N______-_action%3A_update*_label*N________label%3A_flag*N________new*_label%3A_istio.response*_flags*N______-_action%3A_update*_label*N________label%3A_svc*N________new*_label%3A_service.name*N______-_action%3A_update*_label*N________label%3A_ns*N________new*_label%3A_service.namespace*N______-_action%3A_update*_label*N________label%3A_span.name*N________new*_label%3A_net.peer.name*N______-_action%3A_aggregate*_labels*N________aggregation*_type%3A_sum*N________label*_set%3A*N________-_k8s.cluster.name*N________-_k8s.cluster.internal*_name*N________-_istio.response*_flags*N________-_service.name*N________-_service.namespace*N________-_net.peer.name*N________-_http.status*_code*N________-_destination*_subset*N____-_action%3A_insert*N______experimental*_match*_labels%3A*N________prot%3A_http*N________span.kind%3A_SPAN*_KIND*_SERVER*N______include%3A_traces.span.metrics.duration*N______match*_type%3A_strict*N______new*_name%3A_http.server.duration*N______operations%3A*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.name*N________new*_value%3A_actual*_cluster*_name*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.internal*_name*N________new*_value%3A_internal*_cluster*_name*N______-_action%3A_update*_label*N________label%3A_flag*N________new*_label%3A_istio.response*_flags*N______-_action%3A_update*_label*N________label%3A_svc*N________new*_label%3A_service.name*N______-_action%3A_update*_label*N________label%3A_ns*N________new*_label%3A_service.namespace*N______-_action%3A_aggregate*_labels*N________aggregation*_type%3A_sum*N________label*_set%3A*N________-_k8s.cluster.name*N________-_k8s.cluster.internal*_name*N________-_istio.response*_flags*N________-_service.name*N________-_service.namespace*N________-_http.status*_code*N____-_action%3A_insert*N______experimental*_match*_labels%3A*N________prot%3A_grpc*N________span.kind%3A_SPAN*_KIND*_CLIENT*N______include%3A_traces.span.metrics.duration*N______match*_type%3A_strict*N______new*_name%3A_rpc.client.duration*N______operations%3A*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.name*N________new*_value%3A_actual*_cluster*_name*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.internal*_name*N________new*_value%3A_internal*_cluster*_name*N______-_action%3A_update*_label*N________label%3A_flag*N________new*_label%3A_istio.response*_flags*N______-_action%3A_update*_label*N________label%3A_svc*N________new*_label%3A_service.name*N______-_action%3A_update*_label*N________label%3A_ns*N________new*_label%3A_service.namespace*N______-_action%3A_update*_label*N________label%3A_span.name*N________new*_label%3A_net.peer.name*N______-_action%3A_add*_label*N________new*_label%3A_rpc.system*N________new*_value%3A_grpc*N______-_action%3A_update*_label*N________label%3A_grpc.status*_code*N________new*_label%3A_rpc.grpc.status*_code*N______-_action%3A_aggregate*_labels*N________aggregation*_type%3A_sum*N________label*_set%3A*N________-_k8s.cluster.name*N________-_k8s.cluster.internal*_name*N________-_istio.response*_flags*N________-_service.name*N________-_service.namespace*N________-_net.peer.name*N________-_rpc.system*N________-_rpc.grpc.status*_code*N________-_rpc.service*N________-_rpc.method*N________-_destination*_subset*N____-_action%3A_insert*N______experimental*_match*_labels%3A*N________prot%3A_grpc*N________span.kind%3A_SPAN*_KIND*_SERVER*N______include%3A_traces.span.metrics.duration*N______match*_type%3A_strict*N______new*_name%3A_rpc.server.duration*N______operations%3A*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.name*N________new*_value%3A_actual*_cluster*_name*N______-_action%3A_add*_label*N________new*_label%3A_k8s.cluster.internal*_name*N________new*_value%3A_internal*_cluster*_name*N______-_action%3A_update*_label*N________label%3A_flag*N________new*_label%3A_istio.response*_flags*N______-_action%3A_update*_label*N________label%3A_svc*N________new*_label%3A_service.name*N______-_action%3A_update*_label*N________label%3A_ns*N________new*_label%3A_service.namespace*N______-_action%3A_add*_label*N________new*_label%3A_rpc.system*N________new*_value%3A_grpc*N______-_action%3A_update*_label*N________label%3A_grpc.status*_code*N________new*_label%3A_rpc.grpc.status*_code*N______-_action%3A_aggregate*_labels*N________aggregation*_type%3A_sum*N________label*_set%3A*N________-_k8s.cluster.name*N________-_k8s.cluster.internal*_name*N________-_istio.response*_flags*N________-_service.name*N________-_service.namespace*N________-_rpc.system*N________-_rpc.grpc.status*_code*N________-_rpc.service*N________-_rpc.method*N__resource%2Fcommon%3A*N____attributes%3A*N____-_action%3A_delete*N______key%3A_process.command*N____-_action%3A_delete*N______key%3A_process.command*_line*N____-_action%3A_delete*N______key%3A_process.command*_args*N____-_action%3A_delete*N______key%3A_process.executable.name*N____-_action%3A_delete*N______key%3A_process.executable.path*N____-_action%3A_delete*N______key%3A_process.pid*N____-_action%3A_delete*N______key%3A_process.runtime.description*N____-_action%3A_delete*N______key%3A_process.runtime.name*N____-_action%3A_delete*N______key%3A_process.runtime.version*N____-_action%3A_delete*N______key%3A_os.description*N____-_action%3A_delete*N______key%3A_os.type*N____-_action%3A_delete*N______key%3A_env.aws.account*N____-_action%3A_delete*N______key%3A_env.aws.project.name*N____-_action%3A_delete*N______key%3A_env.aws.region*N____-_action%3A_delete*N______key%3A_env.platform*N____-_action%3A_delete*N______key%3A_env.service.name*N____-_action%3A_delete*N______key%3A_env.version*N__resource%2Fpod-ip-sem-conv%3A*N____attributes%3A*N____-_action%3A_insert*N______from*_attribute%3A_ipv4*N______key%3A_k8s.pod.ip*N____-_action%3A_insert*N______from*_attribute%3A_net.host.ip*N______key%3A_k8s.pod.ip*N____-_action%3A_delete*N______key%3A_ipv4*N____-_action%3A_delete*N______key%3A_net.host.ip*N__resource%2Fremove-k8s-pod-ip%3A*N____attributes%3A*N____-_action%3A_delete*N______key%3A_k8s.pod.ip*N__resource%2Fspanmetrics-remove-unused%3A*N____attributes%3A*N____-_action%3A_delete*N______key%3A_http.scheme*N____-_action%3A_delete*N______key%3A_net.host.port*N____-_action%3A_delete*N______key%3A_service.instance.id*N____-_action%3A_delete*N______key%3A_service.name*N__span%2Fspanmetrics-destination-service%3A*N____name%3A*N______to*_attributes%3A*N________rules%3A*N________-_%5E*C*QP*Ldest*G%5B%5E%3A*B%2F%5D*P*D.***S*N__span%2Fspanmetrics-fake-name%3A*N____name%3A*N______from*_attributes%3A*N______-_dest*N__span%2Funset*_cache*_client*_404%3A*N____include%3A*N______attributes%3A*N______-_key%3A_http.response.status*_code*N________value%3A_%5E404*S*N______-_key%3A_server.address*N________value%3A_%5E*Cservice-x*B.skyscanner*B.net%7Cservice-y*B.skyscanner*B.net%7Cservice-z*B.skyscanner*B.net%7Cservice-z-*Bw%7B2%7D-*Bw*P-*Bd*B.int*B.*Bw%7B2%7D-*Bw*P-*Bd*B.skyscanner*B.com*D*S*N______match*_type%3A_regexp*N______regexp%3A*N________cacheenabled%3A_true*N________cachemaxnumentries%3A_1000*N____status%3A*N______code%3A_Unset*N__span%2Funset*_cache*_client*_404*_legacy%3A*N____include%3A*N______attributes%3A*N______-_key%3A_http.status*_code*N________value%3A_%5E404*S*N______-_key%3A_net.peer.name*N________value%3A_%5E*Cservice-x*B.skyscanner*B.net%7Cservice-y*B.skyscanner*B.net%7Cservice-z*B.skyscanner*B.net%7Cservice-z-*Bw%7B2%7D-*Bw*P-*Bd*B.int*B.*Bw%7B2%7D-*Bw*P-*Bd*B.skyscanner*B.com*D*S*N______match*_type%3A_regexp*N______regexp%3A*N________cacheenabled%3A_true*N________cachemaxnumentries%3A_1000*N____status%3A*N______code%3A_Unset*N__span%2Funset*_cache*_client*_404*_url%3A*N____include%3A*N______attributes%3A*N______-_key%3A_http.response.status*_code*N________value%3A_%5E404*S*N______-_key%3A_http.url*N________value%3A_%5E*Cservice-1*B.skyscanner*B.net*B%2Fapi*B%2Fv3*B%2Fflights%7Cservice-2*B.skyscanner*B.net*B%2Fapi*B%2Fv2*B%2Fhotels*B%2F.*P*D*S*N______match*_type%3A_regexp*N______regexp%3A*N________cacheenabled%3A_true*N________cachemaxnumentries%3A_1000*N____status%3A*N______code%3A_Unset*N__span%2Funset*_cache*_client*_404*_url*_legacy%3A*N____include%3A*N______attributes%3A*N______-_key%3A_http.status*_code*N________value%3A_%5E404*S*N______-_key%3A_http.url*N________value%3A_%5E*Cservice-1*B.skyscanner*B.net*B%2Fapi*B%2Fv3*B%2Fflights%7Cservice-2*B.skyscanner*B.net*B%2Fapi*B%2Fv2*B%2Fhotels*B%2F.*P*D*S*N______match*_type%3A_regexp*N______regexp%3A*N________cacheenabled%3A_true*N________cachemaxnumentries%3A_1000*N____status%3A*N______code%3A_Unset*N__span%2Funset*_status*_jaxrs*_common%3A*N____include%3A*N______libraries%3A*N______-_name%3A_io.opentelemetry.jaxrs-2.0-common*N______match*_type%3A_strict*N____status%3A*N______code%3A_Unset*N__transform%2Fspanmetrics-prep%3A*N____trace*_statements%3A*N____-_context%3A_span*N______statements%3A*N______-_set*Cattributes%5B%22prot%22%5D%2C_%22http%22*D_where_attributes%5B%22grpc.status*_code%22%5D_*E*E_nil*N______-_set*Cattributes%5B%22prot%22%5D%2C_%22grpc%22*D_where_attributes%5B%22grpc.status*_code%22%5D_%21*E_nil*N______-_set*Cattributes%5B%22dest%22%5D%2C_%22unknown-ip%22*D_where_attributes%5B%22ip%22%5D_%21*E_nil_and_attributes%5B%22ip%22%5D_%21*E_%22%22*N______-_set*Cattributes%5B%22dest%22%5D%2C_%22aws-metadata-service%22*D_where_attributes%5B%22ip%22%5D_*E*E_%22169.254.169.254%22*N______-_set*Cattributes%5B%22dest%22%5D%2C_%22vpc-ip%22*D_where_attributes%5B%22ip*_8%22%5D_*E*E_%22172%22_and_Int*Cattributes%5B%22ip*_16%22%5D*D_*G*E_16_and_Int*Cattributes%5B%22ip*_16%22%5D*D_*L*E_20*N__transform%2Ftruncate*_all%3A*N____metric*_statements%3A*N____-_context%3A_resource*N______statements%3A*N______-_truncate*_all*Cattributes%2C_4095*D*N____-_context%3A_datapoint*N______statements%3A*N______-_truncate*_all*Cattributes%2C_4095*D*N____trace*_statements%3A*N____-_context%3A_resource*N______statements%3A*N______-_truncate*_all*Cattributes%2C_4095*D*N____-_context%3A_span*N______statements%3A*N______-_truncate*_all*Cattributes%2C_4095*D*N____-_context%3A_spanevent*N______statements%3A*N______-_truncate*_all*Cattributes%2C_4095*D*Nreceivers%3A*N__otlp%3A*N____protocols%3A*N______grpc%3A*N________endpoint%3A_0.0.0.0%3A50051*N__prometheus%3A*N____config%3A*N______scrape*_configs%3A*N______-_job*_name%3A_opentelemetry-collector*N________scrape*_interval%3A_10s*N________static*_configs%3A*N________-_targets%3A*N__________-_*S%7Benv%3AMY*_POD*_IP%7D%3A8888*N__zipkin%3A*N____endpoint%3A_0.0.0.0%3A9411*Nservice%3A*N__extensions%3A*N__-_health*_check*N__-_pprof*N__pipelines%3A*N____metrics%3A*N______exporters%3A*N______-_otlphttp%2Fmetrics*N______processors%3A*N______-_memory*_limiter%2Fmetrics*N______-_batch*N______-_resource%2Fcommon*N______receivers%3A*N______-_otlp*N____metrics%2Fspanmetrics-export%3A*N______exporters%3A*N______-_otlphttp%2Fspanmetrics*N______processors%3A*N______-_filter%2Fspanmetrics*N______-_attributes%2Fspanmetrics-split-service-name*N______-_metricstransform%2Fspanmetrics-semantic-naming*N______-_filter%2Fspanmetrics-duration*N______-_resource%2Fspanmetrics-remove-unused*N______-_batch%2Fspanmetrics-export*N______receivers%3A*N______-_spanmetrics*N____traces%3A*N______exporters%3A*N______-_otlphttp*N______processors%3A*N______-_memory*_limiter*N______-_resource%2Fcommon*N______-_transform%2Ftruncate*_all*N______-_span%2Funset*_cache*_client*_404*_legacy*N______-_span%2Funset*_cache*_client*_404*N______-_span%2Funset*_cache*_client*_404*_url*_legacy*N______-_span%2Funset*_cache*_client*_404*_url*N______-_span%2Funset*_status*_jaxrs*_common*N______-_batch*N______receivers%3A*N______-_otlp*N____traces%2Fspanmetrics%3A*N______exporters%3A*N______-_otlphttp*N______-_spanmetrics*N______processors%3A*N______-_memory*_limiter%2Fspanmetrics*N______-_filter%2Fspanmetrics-prep*N______-_groupbyattrs%2Fgroup-by-ip*N______-_resource%2Fpod-ip-sem-conv*N______-_k8sattributes%2Ffrom-pod-ip*N______-_resource%2Fremove-k8s-pod-ip*N______-_span%2Fspanmetrics-destination-service*N______-_attributes%2Fspanmetrics-prep*N______-_transform%2Fspanmetrics-prep*N______-_span%2Fspanmetrics-fake-name*N______-_batch%2Fspanmetrics-prep*N______receivers%3A*N______-_zipkin*N__telemetry%3A*N____metrics%3A*N______readers%3A*N______-_pull%3A*N__________exporter%3A*N____________prometheus%3A*N______________host%3A_0.0.0.0*N______________port%3A_8888*N______________without*_type*_suffix%3A_true%7E
[agent-otelbin]:
  https://www.otelbin.io/#config=exporters%3A*N__debug%3A*N____verbosity%3A_normal*N__otlphttp%3A*N____endpoint%3A_https%3A%2F%2Fotlp.vendor.com*Nextensions%3A*N__health*_check%3A*N____endpoint%3A_*S%7Benv%3AMY*_POD*_IP%7D%3A13133*Nprocessors%3A*N__attributes%2Fconventions%3A*N____actions%3A*N____-_action%3A_delete*N______key%3A_service*_name*N____-_action%3A_delete*N______key%3A_service*_namespace*N____-_action%3A_delete*N______key%3A_service*_instance*_id*N____-_action%3A_delete*N______key%3A_service*_version*N__attributes%2Fkube-state-metrics%3A*N____actions%3A*N____-_action%3A_upsert*N______from*_attribute%3A_namespace*N______key%3A_k8s.namespace.name*N____-_action%3A_upsert*N______from*_attribute%3A_horizontalpodautoscaler*N______key%3A_k8s.hpa.name*N____-_action%3A_upsert*N______from*_attribute%3A_resourcequota*N______key%3A_k8s.resourcequota.name*N____-_action%3A_delete*N______key%3A_namespace*N____-_action%3A_delete*N______key%3A_horizontalpodautoscaler*N____-_action%3A_delete*N______key%3A_resourcequota*N__batch%3A*N____send*_batch*_max*_size%3A_512*N____send*_batch*_size%3A_512*N__cumulativetodelta%3A_null*N__filter%2Fprom*_scrape*_metrics%3A*N____metrics%3A*N______metric%3A*N______-_IsMatch*Cname%2C_%22scrape*_.**%22*D*N__filter%2Fzero*_value*_counts%3A*N____metrics%3A*N______datapoint%3A*N______-_value*_double_*E*E_0.0*N__groupbyattrs%2Fkube-state-metrics%3A*N____keys%3A*N____-_k8s.namespace.name*N____-_k8s.cluster.name*N____-_k8s.cluster.internal*_name*N____-_k8s.hpa.name*N____-_k8s.resourcequota.name*N__memory*_limiter%3A*N____check*_interval%3A_10s*N____limit*_percentage%3A_50*N____spike*_limit*_percentage%3A_1*N__metricstransform%2Fenvoy*_metrics%3A*N____transforms%3A*N____-_action%3A_update*N______include%3A_%5E.***Cejections*_active*D*S*N______match*_type%3A_regexp*N______new*_name%3A_envoy.cluster.outlier*_detection.ejections.active*N__resource%3A*N____attributes%3A*N____-_action%3A_insert*N______key%3A_k8s.cluster.name*N______value%3A_actual*_cluster*_name*N____-_action%3A_insert*N______key%3A_k8s.cluster.internal*_name*N______value%3A_internal*_cluster*_name*N__resource%2Fkube-state-metrics%3A*N____attributes%3A*N____-_action%3A_delete*N______key%3A_k8s.container.name*N____-_action%3A_delete*N______key%3A_k8s.namespace.name*N____-_action%3A_delete*N______key%3A_k8s.node.name*N____-_action%3A_delete*N______key%3A_k8s.pod.name*N____-_action%3A_delete*N______key%3A_k8s.pod.uid*N____-_action%3A_delete*N______key%3A_k8s.replicaset.name*N____-_action%3A_delete*N______key%3A_http.scheme*N____-_action%3A_delete*N______key%3A_net.host.name*N____-_action%3A_delete*N______key%3A_net.host.name*N____-_action%3A_delete*N______key%3A_net.host.port*N__transform%2Fnode*_ethtool*_convert*_gauge*_to*_sum%3A*N____metric*_statements%3A*N____-_context%3A_metric*N______statements%3A*N______-_convert*_gauge*_to*_sum*C%22cumulative%22%2C_true*D_where_IsMatch*Cname%2C_%22node*_ethtool*_.*P*_allowance*_exceeded%22*D*N________*E*E_true*N______-_convert*_gauge*_to*_sum*C%22cumulative%22%2C_true*D_where_IsMatch*Cname%2C_%22awscni*_.*P*_req*_count*S%22*D*N________*E*E_true*Nreceivers%3A*N__opencensus%3A_null*N__prometheus%3A*N____config%3A*N______scrape*_configs%3A*N______-_honor*_timestamps%3A_false*N________job*_name%3A_k8s*_by*_annotations*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_*C*Cotelcol%7Ckarpenter*D*_.*P*D*D*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________-_action%3A_drop*N__________regex%3A_*C*Ckarpenter*_build%7Ckarpenter*_scheduler*_queue*_depth*D*C*_.***D*Q*D*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_%22true%22*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*N________-_action%3A_keep*N__________regex%3A_.*P*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*_port*N________-_action%3A_keep*N__________regex%3A_.*P*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_drop*N__________regex%3A_kube-state-metrics*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_replace*N__________regex%3A_*C%5B%5E%3A%5D*P*D*C*Q%3A%3A*Bd*P*D*Q%3B*C*Bd*P*D*N__________replacement%3A_*S*S1%3A*S*S2*N__________source*_labels%3A*N__________-_*_*_address*_*_*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*_port*N__________target*_label%3A_*_*_address*_*_*N________-_action%3A_replace*N__________regex%3A_*Chttps*D*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scheme*N__________target*_label%3A_*_*_scheme*_*_*N________-_action%3A_replace*N__________regex%3A_*C.*P*D*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*_path*N__________target*_label%3A_*_*_metrics*_path*_*_*N________-_action%3A_replace*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N__________target*_label%3A_job*N________-_action%3A_drop*N__________regex%3A_.**-envoy-prom*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_container*_port*_name*N________scrape*_interval%3A_30s*N________tls*_config%3A*N__________insecure*_skip*_verify%3A_true*N______-_honor*_timestamps%3A_false*N________job*_name%3A_k8s*_node*_exporter*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_node*_ethtool*_.*P*_allowance*_exceeded*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_prometheus-node-exporter*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_replace*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_jobLabel*N__________target*_label%3A_job*N________-_action%3A_drop*N__________regex%3A_.**-envoy-prom*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_container*_port*_name*N________-_action%3A_replace*N__________regex%3A_*C.***D*N__________replacement%3A_*S*S1*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_kubernetes*_io*_arch*N__________target*_label%3A_host*_arch*N________scrape*_interval%3A_30s*N______-_honor*_timestamps%3A_false*N________job*_name%3A_aws*_vpc*_cni*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_*Cotelcol*D*_.*P*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_aws-node*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_replace*N__________regex%3A_*C%5B%5E%3A%5D*P*D*C*Q%3A%3A*Bd*P*D*Q*N__________replacement%3A_*S*S1%3A61678*N__________source*_labels%3A*N__________-_*_*_address*_*_*N__________target*_label%3A_*_*_address*_*_*N________scrape*_interval%3A_30s*N______-_honor*_timestamps%3A_false*N________job*_name%3A_argocd*_metrics*_scraper*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_*Cgo%7Cprocess%7Crest*D*_.*P*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_%22true%22*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*N________-_action%3A_keep*N__________regex%3A_.*P*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*_port*N________-_action%3A_keep*N__________regex%3A_*Cargo-rollouts%7Cargocd-.***D*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_replace*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N__________target*_label%3A_job*N________scrape*_interval%3A_30s*N________tls*_config%3A*N__________insecure*_skip*_verify%3A_true*N__prometheus%2Fenvoy-metrics%3A*N____config%3A*N______scrape*_configs%3A*N______-_job*_name%3A_envoy-stats*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_*Cenvoy*_cluster*D.***Coutlier*_detection*_ejections*_active*D*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________metrics*_path%3A_%2Fstats%2Fprometheus*N________params%3A*N__________filter%3A*N__________-_.**ejections*_active.***N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_.**-envoy-prom*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_container*_port*_name*N________-_action%3A_replace*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N__________target*_label%3A_job*N______-_job*_name%3A_envoy-stats-default-egress-gateways*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_istio*_tcp.***N__________source*_labels%3A*N__________-_*_*_name*_*_*N________metrics*_path%3A_%2Fstats%2Fprometheus*N________params%3A*N__________filter%3A*N__________-_istio*_tcp.***N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_default-egressgateway*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_type*N________-_action%3A_keep*N__________regex%3A_.**-envoy-prom*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_container*_port*_name*N________-_action%3A_replace*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*N__________target*_label%3A_job*N__prometheus%2Fkube-state-metrics%3A*N____config%3A*N______scrape*_configs%3A*N______-_honor*_timestamps%3A_false*N________job*_name%3A_kube-state-metrics*N________kubernetes*_sd*_configs%3A*N________-_role%3A_pod*N__________selectors%3A*N__________-_field%3A_spec.nodeName*E*S%7Benv%3ANODE*_NAME%7D*N____________role%3A_pod*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_kube*_resourcequota%7Ckube*_pod*_container*_status*_last*_terminated*_reason%7Ckube*_node*_status*_condition*N__________source*_labels%3A*N__________-_*_*_name*_*_*N________-_action%3A_drop*N__________regex%3A_%5E*C%5B%5EO%5D%7CO%5B%5EO%5D%7COO%5B%5EM%5D%7COOM%5B%5EK%5D%7COOMK%5B%5Ei%5D%7COOMKi%5B%5El%5D%7COOMKil%5B%5El%5D%7COOMKill%5B%5Ee%5D%7COOMKille%5B%5Ed%5D*S*D.***N__________source*_labels%3A*N__________-_reason*N________relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_service*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_instance*N________-_action%3A_keep*N__________regex%3A_kube-state-metrics*N__________source*_labels%3A*N__________-_*_*_meta*_kubernetes*_pod*_label*_app*_kubernetes*_io*_name*N________-_action%3A_replace*N__________regex%3A_*C%5B%5E%3A%5D*P*D*C*Q%3A%3A*Bd*P*D*Q%3B*C*Bd*P*D*N__________replacement%3A_*S*S1%3A*S*S2*N__________source*_labels%3A*N__________-_*_*_address*_*_*N__________-_*_*_meta*_kubernetes*_pod*_annotation*_prometheus*_io*_scrape*_port*N__________target*_label%3A_*_*_address*_*_*N________scrape*_interval%3A_30s*N__prometheus%2Fkubelet%3A*N____config%3A*N______scrape*_configs%3A*N______-_bearer*_token*_file%3A_%2Fvar%2Frun%2Fsecrets%2Fkubernetes.io%2Fserviceaccount%2Ftoken*N________honor*_timestamps%3A_false*N________job*_name%3A_kubelet*N________metric*_relabel*_configs%3A*N________-_action%3A_keep*N__________regex%3A_kubelet*_*Cpod*_start%7Cevictions%7Cimage*_pull*_duration%7Cnode*_startup*_duration%7Cpleg*_relist%7Cpreemptions%7Crunning*_containers%7Crunning*_pods*D.***N__________source*_labels%3A*N__________-_*_*_name*_*_*N________metrics*_path%3A_%2Fapi%2Fv1%2Fnodes%2F*S%7Benv%3ANODE*_NAME%7D%2Fproxy%2Fmetrics*N________scheme%3A_https*N________static*_configs%3A*N________-_targets%3A*N__________-_kubernetes.default.svc*N__zipkin%3A*N____endpoint%3A_*S%7Benv%3AMY*_POD*_IP%7D%3A9411*Nservice%3A*N__extensions%3A*N__-_health*_check*N__pipelines%3A*N____metrics%3A*N______exporters%3A*N______-_otlphttp*N______processors%3A*N______-_memory*_limiter*N______-_filter%2Fprom*_scrape*_metrics*N______-_batch*N______-_transform%2Fnode*_ethtool*_convert*_gauge*_to*_sum*N______-_attributes%2Fconventions*N______-_resource*N______receivers%3A*N______-_prometheus*N____metrics%2Fenvoy-metrics%3A*N______exporters%3A*N______-_otlphttp*N______processors%3A*N______-_memory*_limiter*N______-_filter%2Fzero*_value*_counts*N______-_filter%2Fprom*_scrape*_metrics*N______-_batch*N______-_metricstransform%2Fenvoy*_metrics*N______-_attributes%2Fconventions*N______-_resource*N______receivers%3A*N______-_prometheus%2Fenvoy-metrics*N____metrics%2Fkube-state-metrics%3A*N______exporters%3A*N______-_otlphttp*N______processors%3A*N______-_memory*_limiter*N______-_filter%2Fprom*_scrape*_metrics*N______-_batch*N______-_resource*N______-_resource%2Fkube-state-metrics*N______-_attributes%2Fkube-state-metrics*N______-_groupbyattrs%2Fkube-state-metrics*N______receivers%3A*N______-_prometheus%2Fkube-state-metrics*N____metrics%2Fkubelet%3A*N______exporters%3A*N______-_otlphttp*N______processors%3A*N______-_memory*_limiter*N______-_batch*N______-_attributes%2Fconventions*N______-_resource*N______receivers%3A*N______-_prometheus%2Fkubelet*N__telemetry%3A*N____logs%3A*N______level%3A_info*N____metrics%3A*N______readers%3A*N______-_pull%3A*N__________exporter%3A*N____________prometheus%3A*N______________host%3A_*S%7Benv%3AMY*_POD*_IP%7D*N______________port%3A_8888*N______________without*_type*_suffix%3A_true%7E
