---
title: 배포판
weight: 25
default_lang_commit: 6a7f17450ce3edc2e4363013551ee93ba7934a5d
---

오픈텔레메트리(OpenTelemetry) 프로젝트는 현재 컬렉터의 미리 빌드된
[배포판][distributions]을 제공한다. 각 배포판에 포함된 구성 요소는 해당 배포판의
`manifest.yaml`에서 확인할 수 있다.

[distributions]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions

{{% ecosystem/distributions-table filter="first-party-collector" %}}

## 커스텀 배포판 {#custom-distributions}

오픈텔레메트리 프로젝트에서 제공하는 기존 배포판이 필요에 맞지 않을 수도 있다.
예를 들어, 더 작은 바이너리가 필요하거나
[인증자(authenticator) 익스텐션](/docs/collector/extend/custom-component/extension/authenticator),
[리시버](/docs/collector/extend/custom-component/receiver), 프로세서, 익스포터
또는 [커넥터](/docs/collector/extend/custom-component/connector)와 같은 커스텀
기능을 구현해야 할 수도 있다. 배포판을 빌드하는 데 사용되는 도구인
[ocb](/docs/collector/extend/ocb/)(오픈텔레메트리 컬렉터 빌더, OpenTelemetry
Collector Builder)를 사용하면 나만의 배포판을 빌드할 수 있다.

## 서드파티(third-party) 배포판 {#third-party-distributions}

일부 조직은 추가 기능을 제공하거나 사용 편의성을 개선하기 위해 자체 컬렉터
배포판을 제공한다. 다음은 서드파티에서 유지 관리하는 컬렉터 배포판 목록이다.

{{% ecosystem/distributions-table filter="third-party-collector" %}}

## 컬렉터 배포판 추가하기 {#how-to-add}

컬렉터 배포판을 목록에 올리려면, [배포판 목록][distributions list]에 항목을
추가하여 [PR을 제출한다][submit a PR]. 항목에는 다음 내용이 포함되어야 한다.

- 배포판의 메인 페이지 링크
- 배포판 사용 방법을 설명하는 문서 링크
- 질문이 있을 경우 연락할 수 있는 GitHub 핸들 또는 이메일 주소

[submit a PR]: /docs/contributing/pull-requests/
[distributions list]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/data/ecosystem/distributions.yaml
