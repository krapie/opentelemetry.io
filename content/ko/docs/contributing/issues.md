---
title: 이슈
description:
  기존 이슈를 수정하거나, 버그, 보안 위험, 개선 가능성을 보고하는 방법
weight: 10
_issues: https://github.com/open-telemetry/opentelemetry.io/issues
cSpell:ignore: prepopulated
default_lang_commit: bcf039912a6db8a279c3629dfe057434b59e5475
---

## 기존 이슈 수정하기 {#fixing-an-existing-issue}

문서를 개선하는 가장 좋은 방법 중 하나는 기존 이슈를 수정하는 것이다.

1. [이슈]({{% param _issues %}}) 목록을 살펴본다.
2. 작업하고 싶은 이슈를 선택한다.

   > [!WARNING] 중요!
   >
   > {{% param chooseAnIssueAtYourLevel %}}
   > [`CI/infra` 레이블이 붙은 이슈는 관리자 전용이다](#ci-infra).

3. 이슈에 댓글이 있다면 읽어본다.
4. 이 이슈가 아직 유효한지 관리자에게 물어보고, 이슈에 댓글을 달아 필요한 질문을
   한다.
5. 이슈에 댓글을 추가해 해당 이슈를 작업할 의사가 있음을 알린다.
6. 이슈 수정 작업을 진행한다. 문제가 생기면 관리자에게 알린다.
7. 준비가 되면
   [풀 리퀘스트(pull request, PR)를 통해 작업물을 제출한다](../pull-requests).

## CI 및 인프라 이슈 {#ci-infra}

[CI/infra][] 레이블이 붙은 이슈는 **관리자 전용**이다. 이 레이블은 저장소 자체의
워크플로, 필수 체크, 도구, 설정에 관한 이슈에 붙는다. 콘텐츠 및 로컬라이제이션
작업과 달리 이 영역은 다음과 같은 특징이 있다.

- 이 영역의 변경 사항은 비관리자가 의미 있게 테스트할 수 없다. 예를 들어 장애
  양상은 업스트림 CI와 머지 시점의 동작에서 드러난다.
- 이 작업은 이슈 본문에 드러나지 않는 관리자의 로드맵과 맥락에 좌우되는 경우가
  많다.
- 일부 맥락은 의도적으로 비공개로 유지된다. 사이트의 보안 상태에 영향을 미치는
  작업(강화 및 완화 조치)은 머지되기 전에는 공개적으로 논의되지 않는다.

이런 이슈에 붙는 `triage:accepted`는 문제가 확인되었다는 뜻일 뿐, 해당 작업을
아무나 맡아도 된다는 뜻은 아니다.

## 이슈 보고하기 {#reporting-an-issue}

오류를 발견했거나 기존 콘텐츠에 대한 개선을 제안하고 싶다면 이슈를 연다.

1. 아무 문서에서나 **Create documentation issue** 링크를 클릭한다. 그러면 일부
   헤더가 미리 채워진 GitHub 이슈 페이지로 리디렉션된다.
2. 이슈나 개선 제안을 설명한다. 가능한 한 자세한 내용을 제공한다.
3. **Create**를 클릭한다.

제출한 후에는 가끔 이슈를 확인하거나 GitHub 알림을 켜둔다. 관리자와 승인자가
응답하기까지 며칠이 걸릴 수 있다. 리뷰어와 다른 커뮤니티 구성원이 조치를 취하기
전에 질문을 할 수도 있다.

## 새 콘텐츠나 기능 제안하기 {#suggesting-new-content-or-features}

새 콘텐츠나 기능에 대한 아이디어가 있지만 어디에 두어야 할지 확실하지 않다면,
그래도 이슈를 등록할 수 있다. 버그와 보안 취약점도 보고할 수 있다.

1. GitHub로 이동해 **Issues** 탭에서 **[New issue][]** 항목을 선택한다.
2. 자신의 요청이나 궁금한 점에 가장 잘 맞는 이슈 유형을 선택한다.
3. 템플릿을 작성한다.
4. 이슈를 제출한다.

### 좋은 이슈를 작성하는 방법 {#how-to-file-great-issues}

이슈를 등록할 때는 다음 사항을 유념한다.

- 명확한 이슈 설명을 제공한다. 구체적으로 무엇이 빠졌는지, 오래되었는지,
  잘못되었는지, 개선이 필요한지 설명한다.
- 이 이슈가 사용자에게 미치는 구체적인 영향을 설명한다.
- 특정 이슈의 범위를 합리적인 작업 단위로 제한한다. 범위가 큰 문제는 더 작은
  이슈로 나눈다. 예를 들어 "보안 문서를 고쳐라"는 너무 광범위하지만, "'네트워크
  접근 제한' 항목에 세부 내용 추가하기"는 실행 가능할 만큼 구체적이다.
- 새 이슈와 관련되거나 유사한 것이 있는지 기존 이슈를 검색한다.
- 새 이슈가 다른 이슈나 풀 리퀘스트와 관련이 있다면 전체 URL로 참조하거나 `#`
  문자를 붙인 이슈 또는 풀 리퀘스트 번호로 참조한다. 예를 들어
  `Introduced by #987654`와 같이 작성한다.
- [행동 강령][Code of Conduct]을 따른다. 동료 기여자를 존중한다. 예를 들어 "이
  문서는 형편없다"는 도움이 되지도 않고 예의 바르지도 않은 피드백이다.

<!-- prettier-ignore-start -->
[CI/infra]: https://github.com/open-telemetry/opentelemetry.io/labels/CI%2Finfra
[Code of Conduct]: https://github.com/open-telemetry/community/blob/main/code-of-conduct.md
[new issue]: https://github.com/open-telemetry/opentelemetry.io/issues/new/
<!-- prettier-ignore-end -->
<!-- markdownlint-disable link-image-reference-definitions -->

[choose an issue]: <{{% param _issues %}}>
