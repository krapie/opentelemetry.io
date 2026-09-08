---
title: 기여하기
aliases: [/docs/contribution-guidelines]
sidebar_root_for: self
weight: 980
cascade:
  chooseAnIssueAtYourLevel: |
    오픈텔레메트리에 대한 **경험**과 **이해도** 수준에 맞는
    [이슈 선택하기][choose an issue]를 확인한다. 자신의 역량을 넘어서는
    이슈는 피한다.
  _issues: https://github.com/open-telemetry/opentelemetry.io/issues
  _issue: https://github.com/open-telemetry/opentelemetry.io/issues?q=state%3Aopen%20label%3A
default_lang_commit: f09627656c918ddb572c6c876beed29bb415f5ef
---

> [!TIP] 관심을 가져주어 감사하다!
>
> 오픈텔레메트리(OpenTelemetry) 문서와 웹사이트에 기여하는 데 관심을 가져주어
> 감사하다.

## <i class='far fa-exclamation-triangle text-warning '></i> 처음 기여하는가? {#first-time-contributing}

- 다음 레이블이 붙은 **[이슈 선택하기][choose an issue]**:
  - [Good first issue](<{{% param _issue %}}%22good%20first%20issue%22>)
  - [Help wanted](<{{% param _issue %}}%22help%20wanted%22>)

  > [!WARNING] 이슈를 배정하지 않는다
  >
  > 확정된 멘토십 또는 온보딩 프로세스의 일부가 아닌 한, [오픈텔레메트리
  > 조직][org]에 아직 기여한 적이 없는 사람에게는 이슈를 **배정하지 _않는다_**.
  >
  > [org]: https://github.com/open-telemetry

- {{% param chooseAnIssueAtYourLevel %}}

- [생성형 AI 기여 정책](pull-requests#using-ai)을 읽어본다

- 다른 이슈나 더 큰 변경 작업을 하고 싶은가? [먼저 관리자와
  상의한다][discuss it with maintainers first].

[discuss it with maintainers first]: issues/#fixing-an-existing-issue

## 바로 시작하기! {#jump-right-in}

무엇을 하고 싶은가?

- **오타나 그 밖의 간단한 수정**을 하려면
  [GitHub를 이용한 콘텐츠 제출](pull-requests/#changes-using-github)을 참고한다
- 홈페이지 공지를 추가하거나 업데이트하려면
  [홈페이지 공지](/site/build/announcements/)를 참고한다
- 더 중요한 기여를 하고 싶다면 다음부터 시작해 이 섹션의 페이지를 읽어본다.
  - [사전 준비 사항][Prerequisites]
  - [이슈][Issues]
  - [콘텐츠 제출][Submitting content]

[Prerequisites]: prerequisites/
[Submitting content]: pull-requests/

## 무엇에 기여할 수 있는가? {#what-can-i-contribute-to}

오픈텔레메트리 문서 기여자는 다음과 같은 활동을 할 수 있다.

- 기존 콘텐츠 개선 또는 새 콘텐츠 작성
- [블로그 게시물](blog/) 또는 사례 연구 제출
- [오픈텔레메트리 레지스트리(Registry)](/ecosystem/registry/)에 항목 추가 또는
  업데이트
- 사이트를 빌드하는 코드 개선

이 섹션의 페이지는 오픈텔레메트리 **문서**에 기여하는 방법을 설명한다.

오픈텔레메트리 프로젝트 전반에 기여하는 방법에 대한 안내는 커뮤니티의
[오픈텔레메트리 신규 기여자 가이드][OpenTelemetry New Contributor Guide]를
참고한다. 각 언어 구현체, 컬렉터(Collector), 컨벤션을 위한 모든 [OTel
저장소][org]에는 각자의 프로젝트별 기여 가이드가 있다.

[choose an issue]: issues/#fixing-an-existing-issue
[issues]: issues/
[OpenTelemetry New Contributor Guide]:
  https://github.com/open-telemetry/community/blob/main/guides/contributor
