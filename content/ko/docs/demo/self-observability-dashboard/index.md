---
title: 셀프 옵저버빌리티 대시보드
default_lang_commit: 4f50451b8d03165e25df580d9147560ce2af7105
---

오픈텔레메트리(OpenTelemetry) SDK는(실험적인
[`otel.sdk.*` 시맨틱 컨벤션](/docs/specs/semconv/otel/sdk-metrics/)을 사용해)
서비스가 텔레메트리를 누락하고 있는지, 내보내기(export)에 얼마나 걸리는지,
프로세서 큐가 가득 차고 있는지 등 SDK의 동작 방식을 설명하는 자체 내부 메트릭을
발생시킬 수 있다. 데모의 **셀프 옵저버빌리티(Self-Observability)** 대시보드는
스팬, 로그, 메트릭 파이프라인 전반에서 이러한 메트릭을 시각화한다.

## SDK 셀프 옵저버빌리티 활성화하기 {#enabling-sdk-self-observability}

SDK 셀프 옵저버빌리티는 옵트인(opt-in) 방식이며 아직 실험적인 단계로, 서비스별로
SDK 구성을 통해 활성화된다. 이 데모에서는 `ad`, `fraud-detection`, `kafka`
서비스가 이를 옵트인한다. 대시보드는 `Service` 템플릿 변수에 의해 구동되므로,
이를 옵트인하는 서비스가 추가되면 자동으로 나타난다.

## 대시보드 접근하기 {#accessing-the-dashboard}

데모가 실행 중이면 <http://localhost:8080/grafana/d/self-observability>에서
대시보드에 직접 접근하거나, Grafana 대시보드 목록("Self-Observability")에서
이동할 수 있다.
