---
title: 통합
description:
  오픈텔레메트리(OpenTelemetry)를 퍼스트파티로 지원하는 라이브러리, 서비스, 앱.
aliases: [/integrations]
default_lang_commit: 42ef3b8c965480f4d58b173ed95fcb05fbc7d429
---

{{% include freeze-notice.md %}}

오픈텔레메트리(OpenTelemetry)의 미션은
[고품질의 이식 가능한(portable) 텔레메트리를 널리 보급하여 효과적인 옵저버빌리티(observability)를 실현하는 것](/community/mission/)이다.
다시 말해, 옵저버빌리티는 개발하는 소프트웨어에 내장되어야 한다.

[제로 코드 계측(zero-code instrumentation) 솔루션](/docs/concepts/instrumentation/zero-code)과
[계측 라이브러리(instrumentation library)](/docs/specs/otel/overview/#instrumentation-libraries)를
통한 외부 계측은 애플리케이션에 옵저버빌리티를 부여하는 편리한 방법을
제공하지만, 궁극적으로는 모든 애플리케이션이 네이티브 텔레메트리를 위해
오픈텔레메트리 API와 SDK를 직접 통합하거나, 해당 소프트웨어의 에코시스템에 맞는
퍼스트파티 플러그인을 제공해야 한다고 생각한다.

이 페이지에는 네이티브 계측 또는 퍼스트클래스 플러그인을 제공하는 라이브러리,
서비스, 앱의 예시가 담겨 있다.

## 라이브러리 {#libraries}

오픈텔레메트리를 통한 네이티브 라이브러리 계측은 사용자에게 더 나은
옵저버빌리티와 개발자 경험을 제공하며, 라이브러리가 훅(hook)을 노출하고
문서화해야 할 필요성을 없애준다. 아래는 오픈텔레메트리 API를 사용하여 기본
제공(out of the box) 옵저버빌리티를 제공하는 라이브러리 목록이다.

{{% ecosystem/integrations-table "native libraries" %}}

## 애플리케이션 및 서비스 {#applications-and-services}

다음 목록에는 네이티브 텔레메트리를 위해 오픈텔레메트리 API와 SDK를 직접
통합했거나, 자체 확장성(extensibility) 에코시스템에 맞는 퍼스트파티 플러그인을
제공하는 라이브러리, 서비스, 앱의 예시가 담겨 있다.

오픈소스(OSS) 프로젝트가 목록 앞부분에, 상용 프로젝트가 그 뒤에 나열된다.
[CNCF](https://www.cncf.io/)에 속한 프로젝트는 이름 옆에 CNCF 로고가 표시된다.

{{% ecosystem/integrations-table "application integrations" %}}

## 통합 추가하기 {#how-to-add}

라이브러리, 서비스, 앱을 목록에 등록하려면
[레지스트리](/ecosystem/registry/adding)에 항목을 추가한 [PR을
제출][submit a PR]한다. 항목에는 다음 내용이 포함되어야 한다.

- 라이브러리, 서비스, 앱의 메인 페이지 링크
- 오픈텔레메트리를 사용하여 옵저버빌리티를 활성화하는 방법을 설명하는 문서 링크

> [!NOTE]
>
> 라이브러리, 서비스, 앱 등 어떤 종류든 오픈텔레메트리의 외부 통합을 제공한다면,
> [레지스트리에 추가하는 것을 고려](/ecosystem/registry/adding)한다.
>
> 최종 사용자로서 옵저버빌리티를 위해 오픈텔레메트리를 도입했을 뿐
> 오픈텔레메트리 관련 서비스를 전혀 제공하지 않는다면,
> [도입 기업](/ecosystem/adopters)을 참고한다.
>
> 최종 사용자에게 옵저버빌리티를 제공하기 위해 오픈텔레메트리를 활용하는
> 솔루션을 제공한다면, [벤더](/ecosystem/vendors)를 참고한다.

[submit a PR]: /docs/contributing/pull-requests/

{{% include keep-up-to-date.md 통합 %}}
