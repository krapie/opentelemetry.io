---
title: 텔레메트리 기능
linkTitle: 텔레메트리 기능
aliases: [demo_features, features]
default_lang_commit: 4f50451b8d03165e25df580d9147560ce2af7105
---

## 오픈텔레메트리(OpenTelemetry) {#opentelemetry}

- **[오픈텔레메트리 트레이스](/docs/concepts/signals/traces/)**: 모든 서비스는
  오픈텔레메트리에서 제공하는 계측 라이브러리를 사용해 계측된다.
- **[오픈텔레메트리 메트릭](/docs/concepts/signals/metrics/)**: 일부 서비스는
  오픈텔레메트리에서 제공하는 계측 라이브러리를 사용해 계측된다. 관련 SDK가
  출시됨에 따라 더 많은 서비스가 추가될 예정이다.
- **[오픈텔레메트리 로그](/docs/concepts/signals/logs/)**: 일부 서비스는
  오픈텔레메트리에서 제공하는 계측 라이브러리를 사용해 계측된다. 관련 SDK가
  출시됨에 따라 더 많은 서비스가 추가될 예정이다.
- **[오픈텔레메트리 컬렉터](/docs/collector/)**: 모든 서비스는 계측되어 생성된
  트레이스와 메트릭을 gRPC를 통해 오픈텔레메트리 컬렉터로 전송한다. 수신된
  트레이스는 로그와 Jaeger로 내보내지며, 수신된 메트릭과 익셈플러(exemplar)는
  로그와 Prometheus로 내보내진다.
- **[OpAMP](/docs/specs/opamp/)**: 오픈텔레메트리 컬렉터는 상태, 버전, 속성,
  유효 구성(effective configuration)을 데모의 OpAMP 서버에 보고한다. 보고된
  상태는 <http://localhost:8080/opamp/>의 OpAMP UI에서 확인할 수 있다.
- **SDK 자체 옵저버빌리티(SDK Self-Observability)**: 일부 서비스는
  오픈텔레메트리 SDK 자체가 방출하는 실험적인 `otel.sdk.*` 내부 메트릭을
  선택적으로 활성화하며, 이는
  [자체 옵저버빌리티 대시보드](/docs/demo/self-observability-dashboard/)에서
  시각화된다.

## 옵저버빌리티 솔루션 {#observability-solutions}

- **[Grafana](https://github.com/grafana/grafana)**: 모든 메트릭 대시보드는
  Grafana에 저장된다.
- **[Jaeger](https://www.jaegertracing.io/)**: 생성된 모든 트레이스는 Jaeger로
  전송된다.
- **[OpenSearch](https://opensearch.org/)**: 생성된 모든 로그는 Data Prepper로
  전송된다. OpenSearch는 서비스의 로깅 데이터를 중앙화하는 데 사용된다.
- **[Prometheus](https://prometheus.io/)**: 생성된 모든 메트릭과 익셈플러는
  Prometheus가 스크래핑한다.

## 환경 {#environments}

- **[Docker](https://docs.docker.com)**: 이 포크된 샘플은 Docker로 실행할 수
  있다.
- **[쿠버네티스](https://kubernetes.io/)**: 이 애플리케이션은 Helm 차트를 사용해
  로컬 환경과 클라우드 환경 모두에서 쿠버네티스 위에 실행되도록 설계되었다.

## 프로토콜 {#protocols}

- **[gRPC](https://grpc.io/)**: 마이크로서비스는 서로 통신하기 위해 다량의 gRPC
  호출을 사용한다.
- **[HTTP](https://www.rfc-editor.org/rfc/rfc9110.html)**: 마이크로서비스는
  gRPC를 사용할 수 없거나 잘 지원되지 않는 경우 HTTP를 사용한다.

## 기타 구성 요소 {#other-components}

- **[Envoy](https://www.envoyproxy.io/)**: Envoy는 프론트엔드나 기능 플래그
  서비스와 같이 사용자를 대상으로 하는 웹 인터페이스의 리버스 프록시로 사용된다.
- **[k6](https://k6.io)**: 합성 부하 생성기(synthetic load generator)를 사용해
  웹사이트에 실제와 유사한 사용 패턴을 만들어내는 백그라운드 작업이다.
- **[OpenFeature](https://openfeature.dev)**: 애플리케이션 내 기능을
  활성화하거나 비활성화할 수 있게 해주는 기능 플래그 API 및 SDK이다.
- **[flagd](https://flagd.dev)**: 데모 애플리케이션에서 기능 플래그를 관리하는
  데 사용되는 기능 플래그 데몬(daemon)이다.
