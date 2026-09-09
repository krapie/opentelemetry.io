---
title: 광고 서비스
linkTitle: 광고
aliases: [adservice]
default_lang_commit: 1af9529134b418c716977d11d6e04644dfb55ac9
---

이 서비스는 컨텍스트 키(context key)를 기반으로 사용자에게 제공할 적절한 광고를
결정한다. 광고는 매장에서 판매 중인 상품에 대한 것이다.

[광고 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/ad/)

## 자동 계측 {#auto-instrumentation}

이 서비스는 오픈텔레메트리 자바 에이전트에 의존하여 gRPC와 같은 라이브러리를
자동으로 계측하고 오픈텔레메트리 SDK를 구성한다. 이 에이전트는 `-javaagent`
커맨드 라인 인수를 통해 프로세스에 전달된다. 커맨드 라인 인수는 `Dockerfile`의
`JAVA_TOOL_OPTIONS`를 통해 추가되며, 자동으로 생성된 Gradle 시작 스크립트에서
활용된다.

```dockerfile
ENV JAVA_TOOL_OPTIONS=-javaagent:/app/opentelemetry-javaagent.jar
```

## 트레이스 {#traces}

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬을 가져올 수 있다.

```java
Span span = Span.current();
```

스팬에 속성을 추가하려면 스팬 객체의 `setAttribute`를 사용한다. `getAds`
함수에서는 스팬에 여러 속성이 추가된다.

```java
span.setAttribute("app.ads.contextKeys", req.getContextKeysList().toString());
span.setAttribute("app.ads.contextKeys.count", req.getContextKeysCount());
```

### 스팬 이벤트 추가 {#add-span-events}

스팬에 이벤트를 추가하려면 스팬 객체의 `addEvent`를 사용한다. `getAds`
함수에서는 예외가 발생(catch)했을 때 속성이 포함된 이벤트가 추가된다.

```java
span.addEvent("Error", Attributes.of(AttributeKey.stringKey("exception.message"), e.getMessage()));
```

### 스팬 상태 설정 {#setting-span-status}

작업 결과가 오류인 경우, 스팬 객체의 `setStatus`를 사용하여 그에 맞게 스팬
상태를 설정해야 한다. `getAds` 함수에서는 예외가 발생했을 때 스팬 상태가
설정된다.

```java
span.setStatus(StatusCode.ERROR);
```

### 새 스팬 생성 {#create-new-spans}

새로운 스팬은 `Tracer.spanBuilder("spanName").startSpan()`을 사용하여 생성하고
시작할 수 있다. 새로 생성된 스팬은 `Span.makeCurrent()`를 사용하여 컨텍스트에
설정해야 한다. `getRandomAds` 함수는 새 스팬을 생성하고, 이를 컨텍스트에 설정한
뒤, 작업을 수행하고, 마지막으로 스팬을 종료한다.

```java
// create and start a new span manually
Tracer tracer = GlobalOpenTelemetry.getTracer("ad");
Span span = tracer.spanBuilder("getRandomAds").startSpan();

// put the span into context, so if any child span is started the parent will be set properly
try (Scope ignored = span.makeCurrent()) {

  Collection<Ad> allAds = adsMap.values();
  for (int i = 0; i < MAX_ADS_TO_SERVE; i++) {
    ads.add(Iterables.get(allAds, random.nextInt(allAds.size())));
  }
  span.setAttribute("app.ads.count", ads.size());

} finally {
  span.end();
}
```

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

스팬을 생성하는 것과 마찬가지로, 메트릭을 생성하는 첫 단계는 `Meter` 인스턴스를
초기화하는 것이다. 예를 들어 `GlobalOpenTelemetry.getMeter("ad")`와 같다. 이후
`Meter` 인스턴스에서 제공하는 다양한 빌더 메서드를 사용하여 원하는 메트릭
계측기(instrument)를 생성한다. 예를 들면 다음과 같다.

```java
meter
  .counterBuilder("app.ads.ad_requests")
  .setDescription("Counts ad requests by request and response type")
  .build();
```

### OTel이 아닌 커스텀 메트릭 연동(Prometheus 클라이언트 라이브러리) {#bridging-non-otel-custom-metrics-prometheus-client-library}

광고 서비스는 오픈텔레메트리 SDK 대신
[Prometheus 자바 클라이언트 라이브러리](https://github.com/prometheus/client_java)를
사용하여 소규모의 커스텀 메트릭 집합도 노출한다. 이러한 메트릭은 별도의 HTTP
엔드포인트(`AD_PROMETHEUS_PORT`의 `/metrics`, 기본값 `9465`)에서 노출되며,
오픈텔레메트리 컬렉터의
[`prometheus` 리시버](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver)가
스크레이핑(scraping)하여 OTel SDK 메트릭과 동일한 파이프라인으로 전달한다.

```java
private static final Counter adsServedCounter =
    Counter.builder()
        .name("demo_ad_served_total")
        .help("Total number of ads served, labeled by category")
        .labelNames("category")
        .register();

HTTPServer prometheusServer =
    HTTPServer.builder().port(prometheusPort).buildAndStart();
```

> [!NOTE]
>
> 이 예시는 **OTel 도입 과정에서 흔히 나타나는 패턴**을 보여주기 위해 의도적으로
> 포함되었다. 조직은 라이브러리, 서드파티 익스포터, 또는 레거시 서비스에서 이미
> 상당한 규모의 Prometheus 계측을 보유하고 있는 경우가 많으며, 이를 처음부터
> 전부 다시 작성하지 않고도 오픈텔레메트리 네이티브 파이프라인으로 유입하고자
> 한다. 컬렉터의 `prometheus` 리시버가 바로 이를 가능하게 하는 다리 역할을 한다.

이를 연결하는 컬렉터 구성은 다음과 같다.

```yaml
receivers:
  prometheus/ad:
    config:
      scrape_configs:
        - job_name: ad
          scrape_interval: 10s
          static_configs:
            - targets: ['ad:${env:AD_PROMETHEUS_PORT}']
```

> [!TIP]
>
> **권장 사항**: 이를 _과도기적(transitional)_ 패턴으로 취급한다. 새로운 커스텀
> 메트릭에는 오픈텔레메트리 SDK를 직접 사용한다. 기존 Prometheus 클라이언트
> 메트릭은 주변 코드를 수정할 때 점진적으로 마이그레이션하거나, 별도의 집중적인
> 리팩터링을 통해 마이그레이션한다.
>
> 오픈텔레메트리와 Prometheus 텔레메트리를 혼용할 때 흔히 발생하는 어려움은
> 다음과 같다.
>
> - **아이덴티티 불일치**: `service.name`과 `service.instance.id`가 두
>   파이프라인 간에 일치하지 않을 수 있다.
> - **이중 사고 모델**: Prometheus와 OTel은 서로 다른 개념(레이블 대 속성, 서로
>   다른 시맨틱 컨벤션)과 별도의 API, 유입 파이프라인, 그리고 서로 다른
>   보강(enrichment) 규칙을 사용할 수 있다.
> - **일관성 없는 코드**: 이전 메트릭에는 Prometheus 클라이언트 호출을, 새
>   메트릭에는 OTel API 호출을 혼용하면 코드베이스가 하나의 관용적인(idiomatic)
>   스타일을 갖지 못하게 된다.

### 현재 생성되는 메트릭 {#current-metrics-produced}

아래 메트릭 이름은 모두 Prometheus/Grafana에서 `.` 문자가 `_`로 변환되어
표시된다는 점에 유의한다.

#### 커스텀 메트릭 {#custom-metrics}

현재 사용 가능한 커스텀 메트릭은 다음과 같다.

- `app.ads.ad_requests`(오픈텔레메트리 SDK): 요청이 컨텍스트 키로 타겟팅되었는지
  여부와, 응답이 타겟팅된 광고인지 랜덤 광고인지를 나타내는 차원을 갖는 광고
  요청 카운터(counter)이다.
- `demo_ad_served_total`(Prometheus 클라이언트 라이브러리, 컬렉터가 스크레이핑):
  `category`(예: `telescopes`, `binoculars`, `random`)로 레이블이 지정된, 제공된
  광고 수를 나타내는 카운터이다. 위의
  [OTel이 아닌 커스텀 메트릭 연동](#bridging-non-otel-custom-metrics-prometheus-client-library)을
  참고한다.

#### 자동 계측된 메트릭 {#auto-instrumented-metrics}

애플리케이션에서 사용 가능한 자동 계측된 메트릭은 다음과 같다.

- [JVM 런타임 메트릭](/docs/specs/semconv/runtime/jvm-metrics/)
- [RPC 지연 시간 메트릭](/docs/specs/semconv/rpc/rpc-metrics/#rpc-server)

## 로그 {#logs}

광고 서비스는 Log4J를 사용하며, 이는 OTel 자바 에이전트에 의해 자동으로
구성된다.

이는 로그 레코드에 트레이스 컨텍스트를 포함하여, 로그와 트레이스 간의
상관관계(correlation) 분석을 가능하게 한다.
