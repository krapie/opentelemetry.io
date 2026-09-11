---
title: 승인자와 관리자를 위한 SIG 관행
linkTitle: SIG 관행
description: 승인자와 관리자가 이슈와 기여를 관리하는 방법을 알아본다.
weight: 999
# prettier-ignore
cSpell:ignore: Comms contribfest hotfixes inactivitiy onboarded triager triagers
default_lang_commit: 4d367048a304df51f35b33ca51cfb0d3703de230
---

이 페이지는 승인자와 관리자가 사용하는 가이드라인과 일부 일반적인 관행을 담고
있다.

## 온보딩 {#onboarding}

기여자가 문서에 대해 더 큰 책임을 지는 역할(승인자, 관리자)을 맡기로 하면, 기존
승인자와 관리자가 다음과 같이 온보딩(onboarding)한다.

- `docs-approvers`(또는 `docs-maintainers`) 그룹에 추가된다.
- `#otel-comms`와 `#otel-maintainers`, 그리고 비공개 팀 내 slack 채널에
  추가된다.
- [SIG Comms 미팅](https://groups.google.com/a/opentelemetry.io/g/calendar-comms)과
  [관리자 미팅](https://groups.google.com/a/opentelemetry.io/g/calendar-maintainer-meeting)에
  대한 캘린더 초대에 등록하도록 요청받는다.
- 현재 SIG Comms 미팅 시간이 자신에게 맞는지 확인하고, 맞지 않는다면 기존
  승인자, 관리자와 협력해 모두에게 맞는 시간을 찾도록 요청받는다.
- 기여자가 이용할 수 있는 다양한 리소스를 검토하도록 요청받는다.
  - [커뮤니티 리소스](https://github.com/open-telemetry/community/), 특히
    [커뮤니티 멤버십](https://github.com/open-telemetry/community/blob/main/community-membership.md)과
    [소셜 미디어 가이드](https://github.com/open-telemetry/community/blob/main/social-media-guide.md)에
    대한 문서.
  - [기여 가이드라인](/docs/contributing). 이 과정의 일환으로, 이 문서를
    검토하고 이슈나 풀 리퀘스트를 통해 개선을 위한 피드백을 제공한다.

그 밖에 검토하면 유용한 리소스는 다음과 같다.

- [Hugo 문서](https://gohugo.io/documentation/)
- [Docsy 문서](https://www.docsy.dev/docs/)
- 리눅스 재단의 브랜딩과
  [상표 사용 가이드라인](https://www.linuxfoundation.org/legal/trademark-usage)을
  포함한 [마케팅 가이드라인](/community/marketing-guidelines/). 이는 특히
  레지스트리, 통합(Integration), 벤더, 채택 사례, 배포판에 대한 항목을 리뷰할 때
  유용하다.

## 협업 {#collaboration}

- 승인자와 관리자는 각자 다른 작업 일정과 사정을 가지고 있다. 그렇기 때문에 모든
  커뮤니케이션은 비동기적이라고 가정하며, 자신의 정상적인 일정 밖에서 응답해야
  한다는 의무감을 느낄 필요는 없다.
- 승인자나 관리자가 장기간(며칠 또는 일주일 이상) 기여할 수 없거나 그 기간 동안
  자리를 비운다면,
  [#otel-comms](https://cloud-native.slack.com/archives/C02UN96HZH6) 채널을
  이용하고 GitHub 상태를 업데이트해 이를 알려야 한다.
- 승인자와 관리자는
  [OTel 행동 강령](https://github.com/open-telemetry/community/?tab=coc-ov-file#opentelemetry-community-code-of-conduct)과
  [커뮤니티 가치](/community/mission/#community-values)를 준수한다. 이들은
  기여자에게 친절하고 도움이 되는 태도를 취한다. 승인자/관리자가 불편함을 느끼게
  하는 갈등, 오해, 그 밖의 어떤 상황이 생기면 대화나 이슈, PR에서 한 발 물러나
  다른 승인자/관리자에게 대신 나서달라고 요청할 수 있다.

## 트리아지 {#triage}

### 이슈 {#issues}

- 들어오는 이슈는 `@open-telemetry/docs-triagers` 팀이 트리아지(triage)한다.
- 첫 단계로 트리아저(triager)는 이슈 제목과 설명을 읽고 다음과 같이 레이블을
  붙인다.
  - 필수: 이슈의 (공동) 소유권을 정하기 위한 `sig:*`, `lang:*`, `docs:*` 중
    하나:
    - 이슈가 SIG가 공동 소유하는 콘텐츠나 질문과 관련 있다면 `sig:*` 레이블(예:
      컬렉터 관련 질문에는 `sig:collector` 레이블을 붙인다).
    - 이슈가 특정 로컬라이제이션과 관련된 콘텐츠나 질문이라면 `lang:*` 레이블.
    - 이슈가 문서 팀(SIG Comms)만 소유하는 콘텐츠나 질문이라면 `docs:*` 레이블:
      - `docs`
      - `docs:admin`
      - `docs:accessibility`
      - `docs:analytics-and-seo`
      - `docs:IA`
      - `docs:blog`
      - `docs:cleanup/refactoring`
      - `docs:upstream`, `docs:upstream/docsy`
      - `docs:javascript`
      - `docs:mobile`
      - `docs:registry`
      - `docs:ux`
  - 필수: `triage:*` 레이블:
    - `triage:accepted`, `triage:accepted:needs-pr`
    - `triage:deciding`, `triage:deciding:blocked`, `triage:deciding:needs-info`
    - `triage:rejected`, `triage:rejected:duplicate`, `triage:rejected:invalid`,
      `triage:rejected:wontfix`
  - 필수: 다음과 같이 이슈의 "type"을 설정한다.
    - 버그에는 이슈 type `bug`
    - 기능 요청에는 이슈 type `enhancement`
    - 질문에는 `type:question` 레이블
    - 카피 편집(copyedit)에는 `type:copyedit` 레이블
    - 작업 가능하지 않은 열린 형태의 대화로 보인다면 이슈를 "discussions"로
      옮긴다
  - 선택: 해당한다면 예상 소요 시간 레이블:
    - `e0-minutes`
    - ...
    - `e4-months`
  - 선택(관리자만 설정): 우선순위 레이블:
    - `p0-critical`
    - `p1-high`
    - `p2-medium`
    - `p3-low`
  - 선택: 다음 특수 태그 중 하나:
    - `good first issue`
    - `help wanted`
    - `contribfest`
    - `maintainers only`
    - `forever`
    - `stale`
- 자동화는 이슈에 14일 동안 활동이 없으면 재트리아지를 위해 `triage:deciding`
  상태의 이슈에 `triage:followup`을 표시한다. `triage:followup` 레이블은 7일
  이내에 제거해야 한다. 참여자에게 알리고 레이블을 제거하는 것으로 충분한
  활동으로 간주된다.

### PR {#prs}

- PR은 다음 예외를 제외하고 `triage:accepted` 레이블이 붙은 연결된 이슈가 있어야
  한다.
  - 자동화된 PR
  - 관리자/승인자에 의한 hotfix
- 자동화는 PR이 해당 (공동)소유 SIG나 로컬라이제이션 팀에
  [레이블이 붙고](https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/component-label-map.yml)
  [배정되도록](https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/component-owners.yml)
  보장한다.
- PR은 이슈와 같은 공동 소유권 레이블을 가져야 한다.
- PR이 SIG의 공동 소유라면, 해당 그룹이 콘텐츠가 기술적으로 올바른지 확인하는 첫
  번째 리뷰를 담당한다.
- PR이 언어 팀의 공동 소유라면, 해당 그룹이 콘텐츠 번역이 올바른지 확인하는 것을
  담당한다.
- 문서 팀의 주요 책임은 PR이 프로젝트의 전반적인 목표에 부합하고, 구조상 올바른
  위치에 놓여 있으며, 프로젝트의 스타일 가이드와 콘텐츠 가이드를 따르는지
  확인하는 것이다.
- 머지되기 위해 무언가 빠진 PR에는 그에 맞게 레이블을 붙여야 한다.
  - `missing:cla`
  - `missing:docs-approval`
  - `missing:sig-approval`
  - `blocked`
- 자동화는 21일 동안 활동이 없으면 재리뷰를 요청하기 위해 PR에 `stale` 레이블을
  붙인다. `stale` 레이블은 14일 이내에 제거해야 한다. 참여자에게 알리고 레이블을
  제거하는 것으로 충분한 활동으로 간주된다.
- PR은 절대 자동으로 닫히지 않는다.

## 코드 리뷰 {#code-reviews}

### 일반 {#general}

- PR 브랜치가
  `기준 브랜치보다 오래된 상태(out-of-date with the base branch)`라면, 계속
  업데이트할 필요는 없다. 업데이트할 때마다 모든 PR CI 체크가 다시 실행되기
  때문이다! 머지하기 전에만 업데이트해도 충분한 경우가 많다.
- 관리자가 아닌 사람의 PR은 git 서브모듈을 업데이트해서는 **절대** 안 된다. 이런
  일은 가끔 실수로 일어난다. PR 작성자에게 걱정하지 않아도 된다고 알려준다.
  머지하기 전에 우리가 이를 고치겠지만, 앞으로는 최신 상태의 포크에서 작업하도록
  해야 한다고 알려준다.
- 기여자가 CLA에 서명하는 데 어려움을 겪거나 실수로 커밋 중 하나에 잘못된
  이메일을 사용했다면, 문제를 고치거나 풀 리퀘스트를 rebase하도록 요청한다.
  최악의 경우, 새 CLA 체크를 트리거하기 위해 PR을 닫았다가 다시 연다.
- cSpell에 알려지지 않은 단어는 다음 세 위치 중 한 곳의 무시 목록에 추가할 수
  있다.
  - **페이지 프론트매터**: 다른 곳에서는 나타날 가능성이 낮은 일회성 단어에
    일반적으로 선호된다. [철자 검사][Spell Checking]를 참고한다.
  - **로케일 사전**: `.cspell/en-words.txt`처럼 같은 언어의 여러 페이지에 걸쳐
    사용될 가능성이 있는 단어에 선호된다.
  - **공유(모든 로케일) 단어 목록**: `.cspell/all-words.txt`. 제품 이름이나 사람
    이름처럼 모든 언어에서 철자가 유효한 용어에 선호된다. [철자
    검사][Spell Checking]를 참고한다.

  리뷰어와 승인자는 리뷰 과정에서 배치가 적절한지 판단할 수 있다.

[Spell Checking]: ../style-guide/#spell-checking

### 공동 소유 PR {#co-owned-prs}

컬렉터, 데모, 언어별 문서 등 SIG가 공동 소유하는 문서에 대한 변경 사항이 있는
PR은 문서 승인자 한 명과 SIG 승인자 한 명, 이렇게 두 건의 승인을 목표로 해야
한다.

- 문서 승인자는 이런 PR에 `sig:<name>` 레이블을 붙이고 해당 PR에서 SIG의
  `-approvers` 그룹을 태그한다.
- 문서 승인자가 PR을 리뷰하고 승인한 후에는
  [`sig-approval-missing`](https://github.com/open-telemetry/opentelemetry.io/labels/sig-approval-missing)
  레이블을 추가할 수 있다. 이는 SIG에게 해당 PR을 처리해야 한다는 신호를 보낸다.
- 일정 유예 기간(일반적으로 2주이지만 긴급한 경우에는 더 짧을 수 있다) 내에 SIG
  승인이 이루어지지 않으면, 문서 관리자는 자신의 판단으로 해당 PR을 머지할 수
  있다.

### bot의 PR {#prs-from-bots}

bot이 만든 PR은 다음 관행에 따라 머지될 수 있다.

- 레지스트리의 버전을 자동으로 업데이트하는 PR은 즉시 고치고, 승인하고, 머지할
  수 있다.
- SDK, 제로 코드 계측, 컬렉터의 버전을 자동으로 업데이트하는 PR은 해당 SIG가
  머지를 연기해야 한다고 알리지 않는 한 승인하고 머지할 수 있다.
- 명세(specification)의 버전을 자동으로 업데이트하는 PR은 CI 체크를 통과시키기
  위해 스크립트 업데이트가 필요한 경우가 많다. 이 경우
  [@chalin](https://github.com/chalin/)이 해당 PR을 처리한다. 그렇지 않다면 이
  PR도 해당 SIG가 머지를 연기해야 한다고 알리지 않는 한 승인하고 머지할 수 있다.

### 번역 PR {#translation-prs}

번역에 대한 변경 사항이 있는 PR은 문서 승인자 한 명과 번역 승인자 한 명, 이렇게
두 건의 승인을 목표로 해야 한다. 공동 소유 PR에 제안된 것과 비슷한 관행이
적용된다.

## PR 머지하기 {#merging-prs}

관리자는 PR을 머지할 때 다음 워크플로를 적용할 수 있다.

- PR에 모든 승인이 있고 모든 CI 체크가 통과했는지 확인한다.
- 브랜치가 오래된 상태라면 GitHub UI를 통해 rebase 업데이트한다.
- 업데이트하면 모든 CI 체크가 다시 실행된다. 통과할 때까지 기다리거나 다음과
  같은 스크립트를 실행해 백그라운드에서 처리되게 한다.

  ```shell
  export PR=<ID OF THE PR>; gh pr checks ${PR} --watch && gh pr merge ${PR} --squash
  ```

## 명세 PR과 통합 브랜치 {#spec-integration-branches}

웹사이트는 [opentelemetry-specification][]과 [semantic-conventions][] 저장소에서
아직 릴리스되지 않은 변경 사항을 지속적으로 통합한다. 예약된 두 워크플로([자세한
내용][ci-section])가 매일 실행되어 draft "통합" PR을 다음 개발 버전으로
최신화한다.

- 브랜치 패턴: `otelbot/spec-integration-vX.Y.Z-dev`와
  `otelbot/semconv-integration-vX.Y.Z-dev`.
- 현재 존재하는 브랜치 목록: [spec][spec-branches] ·
  [semconv][semconv-branches].

[opentelemetry-specification]:
  https://github.com/open-telemetry/opentelemetry-specification
[semantic-conventions]: https://github.com/open-telemetry/semantic-conventions
[ci-section]: /site/build/ci-workflows/#spec-integration-branches
[spec-branches]:
  https://github.com/open-telemetry/opentelemetry.io/branches/all?query=spec-integration
[semconv-branches]:
  https://github.com/open-telemetry/opentelemetry.io/branches/all?query=semconv-integration

### 명세/시맨틱 컨벤션 SIG 관리자 {#spec--semconv-sig-maintainers}

릴리스를 만들기 직전에 다음과 같이 한다.

1. 이전 섹션에 나온 링크에서 자신의 명세에 대한 최신 통합 브랜치를 찾는다 (예:
   `otelbot/spec-integration-v1.56.0-dev`).
2. 연결된 PR을 연다(브랜치 페이지에서 링크됨).
3. [specs-integration.yml][] 워크플로를 새로 실행해 최신 변경 사항을 반영한다.
4. PR 체크가 성공하면 해당 명세는 릴리스해도 안전하다. 그렇지 않다면 릴리스가
   나가기 전에 문제를 해결할 수 있도록 `@open-telemetry/docs-maintainers`에게
   알린다.

### Comms SIG 관리자 {#comms-sig-maintainers}

한 달 내내 통합 PR을 정기적으로 확인하고, CI 체크를 통과 상태로 유지하기 위해
점진적인 수정 사항을 커밋한다. 업스트림 변경 사항이 아직 작을 때 초기에 문제를
발견하는 것이, 릴리스 당일에 불 끄듯 급하게 대응하는 것보다 훨씬 쉽다.

[specs-integration.yml]:
  https://github.com/open-telemetry/opentelemetry.io/actions/workflows/specs-integration.yml
