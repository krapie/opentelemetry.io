---
title: 벤더
description: 오픈텔레메트리(OpenTelemetry)를 네이티브로 지원하는 벤더
aliases: [/vendors]
default_lang_commit: 42ef3b8c965480f4d58b173ed95fcb05fbc7d429
---

{{% include freeze-notice.md %}}

옵저버빌리티 백엔드, 옵저버빌리티 파이프라인과 같이 [OTLP](/docs/specs/otlp/)를
통해 오픈텔레메트리를 네이티브로 소비하는 솔루션을 제공하는 조직의 목록이며,
전체 목록은 아니다.

일부 조직은 (커스터마이징한 오픈텔레메트리 구성 요소로 이루어진)
[배포판](/ecosystem/distributions/)을 제공하여 추가 기능을 제공하거나 사용
편의성을 개선한다.

오픈소스(OSS)는 [오픈소스](https://opensource.org/osd) 옵저버빌리티 제품을
보유한 벤더를 의미한다. 이러한 벤더도 고객을 위해 오픈소스 제품을 호스팅하는
SaaS 제공처럼, 클로즈드 소스인 다른 제품을 보유할 수 있다.

{{% ecosystem/vendor-table %}}

## 조직 추가하기 {#how-to-add}

조직을 목록에 등록하려면 [벤더 목록][vendors list]에 항목을 추가한 [PR을
제출][submit a PR]한다. 항목에는 다음 내용이 포함되어야 한다.

- 제품이 [OTLP](/docs/specs/otlp/)를 통해 오픈텔레메트리를 네이티브로 소비하는
  방식을 자세히 설명하는 문서 링크
- 배포판이 있는 경우, 배포판 링크
- 제품이 오픈소스임을 증명하는 링크(해당하는 경우). 오픈소스 배포판이 있다고
  해서 제품이 "오픈소스"로 표시될 자격을 얻는 것은 아니다.
- 문의 시 연락할 수 있도록 담당자의 GitHub 핸들 또는 이메일 주소

이 목록은 오픈텔레메트리를 소비하고 [최종 사용자](/community/end-user/)에게
옵저버빌리티를 제공하는 조직을 위한 것임에 유의한다.

[최종 사용자 조직](https://www.cncf.io/enduser/)으로서 옵저버빌리티를 위해
오픈텔레메트리를 도입했을 뿐 오픈텔레메트리 관련 서비스를 전혀 제공하지
않는다면, [도입 기업](/ecosystem/adopters/)을 참고한다.

오픈텔레메트리를 통해 옵저버빌리티가 가능해진 라이브러리, 서비스, 앱을
제공한다면, [통합](/ecosystem/integrations/)을 참고한다.

[submit a PR]: /docs/contributing/pull-requests/

{{% include keep-up-to-date.md 벤더 %}}

[vendors list]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/data/ecosystem/vendors.yaml
