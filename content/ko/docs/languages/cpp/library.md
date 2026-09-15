---
title: 계측 라이브러리 사용하기
linkTitle: 라이브러리
weight: 40
default_lang_commit: 825010e3cfece195ae4dfd019eff080ef8eb6365
---

{{% docs/languages/libraries-intro cpp %}}

## 계측 라이브러리 사용하기 {#using-instrumentation-libraries}

앱을 개발할 때는 작업 속도를 높이기 위해 서드파티 라이브러리와 프레임워크를
사용하는 경우가 많다. 이후 오픈텔레메트리(OpenTelemetry)를 사용해 앱을 계측할
때, 사용 중인 서드파티 라이브러리와 프레임워크에 트레이스, 로그, 메트릭을
수동으로 추가하는 데 추가 시간을 들이고 싶지 않을 수 있다.

많은 라이브러리와 프레임워크는 이미 오픈텔레메트리를 지원하거나, 오픈텔레메트리
[계측(instrumentation)](/docs/concepts/instrumentation/libraries/)을 통해
지원되므로, 옵저버빌리티(observability) 백엔드로 내보낼 수 있는 텔레메트리를
생성할 수 있다.

서드파티 라이브러리나 프레임워크를 사용하는 앱이나 서비스를 계측하는 중이라면,
아래 안내를 따라 네이티브(native)로 계측된 라이브러리와, 의존성을 위한 계측
라이브러리(instrumentation library)를 사용하는 방법을 알아본다.

## 네이티브로 계측된 라이브러리 사용하기 {#use-natively-instrumented-libraries-1}

라이브러리가 기본적으로 오픈텔레메트리를 지원한다면, 앱에 오픈텔레메트리 SDK를
추가하고 설정하는 것만으로 해당 라이브러리에서 방출되는 트레이스, 메트릭, 로그를
얻을 수 있다.

라이브러리에 따라 계측을 위한 추가 구성(configuration)이 필요할 수 있다. 자세한
내용은 해당 라이브러리의 문서를 참고한다.

라이브러리가 오픈텔레메트리를 지원하지 않는다면,
[계측 라이브러리](/docs/specs/otel/glossary/#instrumentation-library)를 사용해
해당 라이브러리나 프레임워크의 텔레메트리 데이터를 생성할 수 있다.

## 설정 {#setup}

계측 라이브러리를 설정하는 방법은
[otel-cpp-contrib](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation)를
참고한다.

## 사용 가능한 패키지 {#available-packages}

사용 가능한 계측 라이브러리의 전체 목록은
[오픈텔레메트리 레지스트리](/ecosystem/registry/?language=cpp&component=instrumentation)에서
찾을 수 있다.

## 다음 단계 {#next-steps}

계측 라이브러리를 설정한 후에는, 커스텀 텔레메트리 데이터를 수집하기 위해
[추가 계측](/docs/languages/cpp/instrumentation/)을 더하고 싶을 수 있다.

또한 텔레메트리 백엔드 한 곳 이상으로
[텔레메트리 데이터를 내보내기](/docs/languages/cpp/exporters/) 위해 적절한
익스포터를 구성하고 싶을 수도 있다.
