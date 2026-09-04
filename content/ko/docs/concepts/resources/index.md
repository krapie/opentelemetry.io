---
title: 리소스
weight: 70
default_lang_commit: 8ada5a9b285dec6ce4cfa790c577ef1523920cd0
---

## 소개 {#introduction}

{{% docs/languages/resources-intro %}}

[Jaeger](https://www.jaegertracing.io/)를 옵저버빌리티 백엔드로 사용하는 경우,
리소스 속성은 **Process** 탭 아래에 그룹화된다.

![트레이스와 연관된 리소스 속성의 출력 예시를 보여주는 Jaeger 스크린샷](screenshot-jaeger-resources.png)

리소스는 초기화 중 생성되는 `TracerProvider` 또는 `MetricProvider`에 추가된다.
이 연관 관계는 이후에 변경할 수 없다. 리소스가 추가되고 나면, 해당
프로바이더로부터 생성된 `Tracer`나 `Meter`가 만들어내는 모든 스팬과 메트릭에는
그 리소스가 연관된다.

## SDK가 기본값을 제공하는 시맨틱 속성(Semantic Attributes) {#semantic-attributes-with-sdk-provided-default-value}

오픈텔레메트리 SDK가 제공하는 속성들이 있다. 그중 하나가 `service.name`이며,
이는 서비스의 논리적 이름을 나타낸다. SDK는 기본적으로 이 값에
`unknown_service`를 할당하므로, 코드에서 직접 설정하거나 환경 변수
`OTEL_SERVICE_NAME`을 설정하여 명시적으로 지정하는 것이 좋다.

또한 SDK는 자신을 식별하기 위해 `telemetry.sdk.name`, `telemetry.sdk.language`,
`telemetry.sdk.version`과 같은 리소스 속성도 제공한다.

## 리소스 감지기(Resource Detectors) {#resource-detectors}

대부분의 언어별 SDK는 환경으로부터 리소스 정보를 자동으로 감지하는 데 사용할 수
있는 일련의 리소스 감지기(resource detector)를 제공한다. 일반적인 리소스
감지기에는 다음이 포함된다.

- [운영체제](/docs/specs/semconv/resource/os/)
- [호스트](/docs/specs/semconv/resource/host/)
- [프로세스 및 프로세스 런타임](/docs/specs/semconv/resource/process/)
- [컨테이너](/docs/specs/semconv/resource/container/)
- [쿠버네티스](/docs/specs/semconv/resource/k8s/)
- [클라우드 제공업체별 속성](/docs/specs/semconv/resource/#cloud-provider-specific-attributes)
- [기타](/docs/specs/semconv/resource/)

## 커스텀 리소스 {#custom-resources}

직접 리소스 속성을 제공할 수도 있다. 코드에서 직접 제공하거나 환경 변수
`OTEL_RESOURCE_ATTRIBUTES`를 채워서 제공할 수 있다. 해당되는 경우
[리소스 속성에 대한 시맨틱 컨벤션](/docs/specs/semconv/resource)을 사용한다.
예를 들어 `deployment.environment.name`을 사용해
[배포 환경](/docs/specs/semconv/resource/deployment-environment/)의 이름을
제공할 수 있다.

```shell
env OTEL_RESOURCE_ATTRIBUTES=deployment.environment.name=production yourApp
```
