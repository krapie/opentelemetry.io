---
title: 배포판
description: >-
  배포판(distribution)은 포크(fork)와 혼동해서는 안 되며,
  오픈텔레메트리(OpenTelemetry) 구성 요소의 커스터마이징된 버전이다.
weight: 190
default_lang_commit: 55f9c9d07ba35c241048ffc0d756d67843d68805
---

오픈텔레메트리(OpenTelemetry) 프로젝트는 여러 [시그널](../signals)을 지원하는
다수의 [구성 요소](../components)로 이루어져 있다. 오픈텔레메트리의 참조
구현체(reference implementation)는 다음과 같은 형태로 제공된다.

- [언어별 계측 라이브러리](../instrumentation)
- [컬렉터 바이너리](/docs/concepts/components/#collector)

모든 참조 구현체는 배포판(distribution)으로 커스터마이징될 수 있다.

## 배포판이란 무엇인가? {#what-is-a-distribution}

배포판은 오픈텔레메트리 구성 요소의 커스터마이징된 버전이다. 배포판은 일부
커스터마이징이 적용된 업스트림 오픈텔레메트리 저장소(repository)를 감싸는
래퍼(wrapper)이다. 배포판은 포크(fork)와 혼동해서는 안 된다.

배포판에 포함될 수 있는 커스터마이징에는 다음이 포함된다.

- 사용을 더 쉽게 하거나 특정 백엔드나 벤더에 맞게 사용을 커스터마이징하기 위한
  스크립트
- 백엔드, 벤더, 또는 최종 사용자에게 필요한 기본 설정 변경
- 벤더 또는 최종 사용자에 특화될 수 있는 추가 패키징 옵션
- 오픈텔레메트리가 제공하는 것 이상의 테스트, 성능, 보안 커버리지
- 오픈텔레메트리가 제공하는 것 이상의 추가 기능
- 오픈텔레메트리가 제공하는 것보다 적은 기능

배포판은 크게 다음과 같은 카테고리로 나뉜다.

- **"Pure":** 이러한 배포판은 업스트림과 동일한 기능을 제공하며 100% 호환된다.
  커스터마이징은 일반적으로 사용 편의성이나 패키징을 개선하는 데 사용된다.
  이러한 커스터마이징은 백엔드, 벤더, 또는 최종 사용자에 특화될 수 있다.
- **"Plus":** 이러한 배포판은 추가 구성 요소를 통해 업스트림 위에 기능을 추가로
  제공한다. 예를 들어 오픈텔레메트리 프로젝트에 업스트림되지 않은 계측
  라이브러리나 벤더 익스포터가 있다.
- **"Minus":** 이러한 배포판은 업스트림 기능의 일부만 제공한다. 예를 들어
  오픈텔레메트리 컬렉터 프로젝트에 있는 계측 라이브러리나 리시버, 프로세서,
  익스포터, 익스텐션(extension)을 제거하는 경우가 있다. 이러한 배포판은 지원
  가능성(supportability)과 보안 고려사항을 강화하기 위해 제공될 수 있다.

## 누가 배포판을 만들 수 있는가? {#who-can-create-a-distribution}

누구나 배포판을 만들 수 있다. 현재 여러 [벤더](/ecosystem/vendors/)가
[배포판](/ecosystem/distributions/)을 제공하고 있다. 또한 최종 사용자도
오픈텔레메트리 프로젝트에 업스트림되지 않은
[레지스트리(registry)](/ecosystem/registry/)의 구성 요소를 사용하고 싶다면 직접
배포판을 만드는 것을 고려할 수 있다.

## 기여인가, 배포판인가? {#contribution-or-distribution}

직접 배포판을 만드는 방법을 계속 읽어보기 전에, 오픈텔레메트리 구성 요소에
추가하려는 내용이 모두에게 유익하여 참조 구현체에 포함되어야 하는지 스스로에게
질문해본다.

- "사용 편의성"을 위한 스크립트를 일반화할 수 있는가?
- 기본 설정 변경이 모두에게 더 나은 옵션이 될 수 있는가?
- 추가 패키징 옵션이 정말로 특정한 용도에만 해당하는가?
- 테스트, 성능, 보안 커버리지가 참조 구현체에서도 동작할 수 있는가?
- 추가 기능이 표준의 일부가 될 수 있는지 커뮤니티에 확인해보았는가?

## 직접 배포판 만들기 {#creating-your-own-distribution}

### 컬렉터 {#collector}

직접 배포판을 만드는 방법에 대한 가이드는 다음 블로그 게시물에서 확인할 수 있다.
["Building your own OpenTelemetry Collector distribution"](https://medium.com/p/42337e994b63)

직접 배포판을 만들고 있다면,
[OpenTelemetry Collector Builder](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder)가
좋은 출발점이 될 수 있다.

### 언어별 계측 라이브러리 {#language-specific-instrumentation-libraries}

계측 라이브러리를 커스터마이징하기 위한 언어별 확장성(extensibility) 메커니즘이
있다.

- [Java 에이전트](/docs/zero-code/java/agent/extensions)

## 가이드라인을 따른다 {#follow-the-guidelines}

배포판에 오픈텔레메트리 프로젝트의 로고 및 이름과 같은 자료(collateral)를 사용할
때는 [기여 조직을 위한 오픈텔레메트리 마케팅 가이드라인][guidelines]을 반드시
준수해야 한다.

오픈텔레메트리 프로젝트는 현재 배포판을 인증하지 않는다. 향후 오픈텔레메트리는
쿠버네티스 프로젝트와 유사하게 배포판과 파트너를 인증할 수도 있다. 배포판을
평가할 때는 해당 배포판을 사용해도 벤더 종속(vendor lock-in)이 발생하지 않는지
확인한다.

> 배포판에 대한 지원은 오픈텔레메트리 저자가 아니라 배포판 저자로부터 제공된다.

[guidelines]:
  https://github.com/open-telemetry/community/blob/main/marketing-guidelines.md
