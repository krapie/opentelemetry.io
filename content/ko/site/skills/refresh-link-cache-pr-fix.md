---
title: 링크 캐시 새로 고침 PR 수정
aliases: [refresh-refcache-pr-fix]
description: >-
  otelbot PR에서 실패한 링크 검사를 해결하는 방법이다.
default_lang_commit: 0aff0aab149003b5592edfe186c193047e40c766
---

[대상 otelbot PR](#target-prs)에서 실패한 링크 검사를 해결하려면 다음 단계를
따른다. 이 과정은 사이트에서 끊어진 링크(dead link)를 업데이트하거나 제거한 뒤,
실패가 남지 않을 때까지 링크 검사를 다시 실행하는 것을 포함할 수 있다.

## 대상 PR {#target-prs}

기본적으로 head 브랜치가 `otelbot/*`와 일치하는 열려 있는 모든 otelbot PR을
훑는다. 지시가 있으면 훑는 범위를 지정된 브랜치나 브랜치 그룹으로 좁힌다(예를
들어 `otelbot/refcache-refresh`, 또는 spec/semconv 통합 브랜치); 지시가 모호하면
질문한다. 이 스킬은 PR을 대상으로 동작한다: 지정된 브랜치에 열려 있는 PR이
없으면 그 사실을 보고하고 멈춘다.

1. 열려 있는 otelbot PR을 나열한다:

   ```sh
   gh pr list --search head:otelbot/ --json number,title,headRefName,isDraft
   ```

2. 그중 어느 것이 실패한 링크 검사를 가지고 있는지 확인한다 -- `Links`
   워크플로의 점검(`gh pr checks <num>`)이다.
3. **PR을 처리하기 전에** 훑기 평가를 보고한다: PR마다 한 줄씩 -- 번호, head
   브랜치, 초안(draft) 상태, 처리 여부(건너뛸 경우 그 이유 포함)이다.
4. 해당하는 각 PR을 차례로 처리하며, 아래 섹션을 따르고, 시작할 때 그 PR의
   이름을 밝힌다. 이 단계들에서 _`TARGET_BRANCH`_ 는 처리 중인 PR의 head
   브랜치이다.

## 준비 {#preparation}

`upstream` 원격이 메인 저장소를 가리키는 로컬 클론의 루트에서 이 단계들을
실행한다.

1. PR 브랜치를 체크아웃한다: `gh pr checkout <num>`. 로컬 _`TARGET_BRANCH`_ 가
   달라져(diverge) 이 명령이 실패하면, 로컬 전용 커밋을 백업하거나(또는 멈추고)
   다음으로 재정렬한다:

   ```sh
   git fetch upstream
   git checkout TARGET_BRANCH
   git reset --hard upstream/TARGET_BRANCH
   ```

2. 콘텐츠 모듈이 오래되었다면 `npm run get:submodule`을 실행한다.

## 5XX 응답 처리하기 {#handling-5xx-responses}

5XX 상태 응답은 대개 일시적이다. 링크 검사가 어떤 URL에 대해 5XX 상태를
보고하면, 일시적일 가능성이 크다고 간주한다(오리진 다운, 게이트웨이 오류,
과부하). 5XX를 우회하려고 사이트 콘텐츠나 링크를 변경하지 **않는다**; 대신
나중에 `npm run log:check:links`를 다시 실행하는 편을 선호한다. 시간이 지나며
여러 번 실행해도 **계속** 실패하고, 그 URL이 다른 방식으로도 정상이 아님을
확인한 경우에만 5XX를 실제 결함처럼 조사한다.

## 실패한 링크 해결하기 {#resolve-failing-links}

1. 사이트를 빌드하고 링크를 검사한다: `npm run log:check:links`. 이 명령은
   `.lycheecache`도 업데이트하고, 아래 다시 확인(double-check) 단계가 읽는 점검
   로그를 캡처한다. 아래 LinkedIn 참고 사항을 확인한다.
2. 점검이 통과하면 [PR 마무리](#wrap-up)로 넘어간다.
3. **그렇지 않다면**, 점검 출력에서 실패한 URL과 그 상태를 나열한다(CI 실행에
   대해서는 PR의 실패한 `CHECK LINKS` 잡 로그를 참고한다).

   > [!NOTE] LinkedIn URL
   >
   > `LinkedIn.com`의 응답은 종종 신뢰할 수 없다(프로필이 존재해도 에이전트와
   > 봇에는 403, 404, 999가 보일 수 있다). 이런 상태만으로 LinkedIn 링크를
   > 제거하거나 수정하지 **않는다**; 대신 관리자가 수동으로 검증하게 한다.

4. 끊어진 링크가 아니라 봇 차단(bot-blocking)처럼 보이는 실패(예를 들어
   브라우저에서는 정상적으로 열리는 사이트에서의 403/429/999)는 분석하기 전에
   실제 브라우저로 다시 검증한다:

   ```sh
   npm run fix:link-cache:double-check
   ```

   이 프로브는 1단계에서 캡처한 로그를 읽는다. 프로브가 정상으로 판단한 URL은
   캐시에 저장된다; 1단계부터 반복하고 남아 있는 실패만 분석한다. 자세한 내용은
   [실패한 링크 다시 확인하기](/site/build/link-checking/#double-check)를
   참고한다.

5. **분석하고 권고한다.** 실패한 URL마다 다음을 보고한다:
   - URL과 HTTP 상태.
   - 그 링크가 어디서 비롯되었는지: 파일이나 페이지로의 링크를 제공한다.
   - 권장하는 수정 또는 후속 조치;
     [실패한 URL에 대한 수정 권고하기](#recommending-a-fix-for-failing-urls)를
     참고한다.
   - 그 수정이 어디에 속해야 하는지, 예를 들어:
     - 해당 브랜치 안에서
     - 같은 끊어진 링크가 `main`이나 여러 대상 PR에도 영향을 준다면, `main`을
       대상으로 하는 별도의 PR에서
     - 통합 브랜치의 경우, 소스 저장소 업스트림에서

   멈추고 리뷰어의 승인을 기다린다 -- 권고 사항을 스스로 승인하지 않는다.

6. **승인된 수정을 적용한다.** 관리자가 승인한 수정과 후속 조치만, 오직 그것만
   수행한다. `content/` 아래의 페이지 콘텐츠에 대해서는 영문 페이지만 수정한다:
   **로컬라이즈된 페이지 콘텐츠는 절대 수정하지 않는다**.

7. 그 소스 링크 변경 이후 `npm run log:check:links`를 실행해 링크를 다시
   검사하고 `.lycheecache`를 새로 고친다. 로컬라이즈된 페이지가 여전히 실패하면,
   수정하는 대신 그 [드리프트 상태][drift status]를 새로 고친다:

   ```sh
   npm run fix:i18n:status -- PATHS_TO_FAILING_LOCALIZED_PAGES
   ```

   드물게 실패한 링크가 로컬라이즈된 페이지에만 존재하는 경우, 이를 보고하고
   해당 로케일 팀과 함께 수정을 조율한다. 자세한 내용은 [링크 수정과 리소스
   업데이트][Link fixes and resource updates]를 참고한다. 이 섹션의
   단계를(2단계부터) 점검이 통과할 때까지 반복한다.

## PR 마무리 {#wrap-up}

처리 중인 PR에서 링크 검사가 통과하면:

1. 응답에 링크 검사 요약을 공유한다(다시 검사했거나 수정한 URL, 그리고
   표시된다면 최종 상태 개수).
2. `.lycheecache`가 변경되었다면, 업스트림 _`TARGET_BRANCH`_ 에 커밋하고
   push한다. 링크 검사 요약을 커밋 메시지 본문으로 사용한다(일반 텍스트; URL
   목록이 길면 개수만 포함한다): 이는 스쿼시(squash) 병합 이후에도 PR의 커밋
   히스토리에 계속 남는다.
3. 스킬 호출이 댓글을 달지 말라고 요청하지 않는 한(예를 들어 "no comment"나
   "silent"가 포함된 경우), PR에 다음으로 구성된 댓글을
   추가한다(`gh pr comment <num> --body '…'`):
   - 인라인 코드로 표시한 스킬 호출 -- 최소한의 형태로 재구성한다(스킬 이름과
     대상 선택만). 비공개이거나 무관한 맥락을 담고 있을 수 있는 주변 대화는 절대
     인용하지 않는다.
   - 실행에 대한 한두 줄짜리 간결한 요약.

   예를 들면:

   ```text
   Link-cache update done using: `/refresh-link-cache-pr-fix for the collector-docs branch`

   Re-checked the failing URLs; all now resolve -- the link check passes.
   ```

4. PR이 초안이 **아니고** 링크 검사가 유일하게 실패한 점검이었다면, 자동 병합을
   활성화하고(`gh pr merge <num> --auto --squash`), 자동 병합이 완료될 수 있도록
   **관리자에게 승인을 상기시킨다**; PR 링크를 포함한다. 그렇지 않다면 PR을
   그대로 둔 이유를 보고한다: 초안 상태(예를 들어, 자체 워크플로가 릴리스 시점에
   마무리하는 통합 PR)이거나, 다른 실패한 점검이다.

이어서 다음 대상 PR이 있다면 계속 진행한다.

## 실패한 URL에 대한 수정 권고하기 {#recommending-a-fix-for-failing-urls}

모든 권고를 증거에 근거하게 하고, 수정을 상황에 맞춘다:

- **링크된 페이지가 이동함**: 대체 대상이 일치한다는
  [증거](#evidence-for-a-replacement-url)와 함께 링크를 업데이트한다.
- **항목의 대상이 사라짐**: 링크가 레지스트리나 에코시스템(ecosystem) 목록 항목
  -- 도입 사례(adopter), 배포판, 통합(integration), 벤더 -- 에서 비롯되었고, 그
  항목 뒤의 컴포넌트, 제품, 회사가 없어졌거나, 흡수되었거나, 더 이상
  오픈텔레메트리(OpenTelemetry)를 적극적으로 지원하지 않는다면, 링크를
  업데이트하는 대신 [항목을 폐기한다](#retiring-an-entry).
- **링크된 페이지가 사라졌고 대체할 것이 없음**: 에이전트는 이 수정을 적용해서는
  안 된다; 관리자에게 맡긴다. 최후의 수단으로, **관리자**가 링크를 제거하고 주변
  서술을 다듬을 수 있다. 상황에 맞게 다음의 GitHub 핸들을 참조(Cc)한다:
  - 해당 콘텐츠를 도입한 PR의 작성자
  - SIG docs 승인자, 이들의 GitHub 팀 핸들을 통해

### 대체 URL에 대한 증거 {#evidence-for-a-replacement-url}

가져온 페이지가 링크된 리소스를 명명하거나 그와 일치함을 보인다:

- 2XX 상태만으로는 아무것도 증명하지 못한다: SPA catch-all과 로그인 페이지는
  어떤 경로에 대해서도 200을 반환한다.
- github.com으로의 링크는, 해당 리소스를 담고 있는 마지막 커밋을 근거로
  대체한다.
- [Wayback Machine](https://web.archive.org/)은 끊어진 URL이 예전에 무엇을
  제공했는지, 또는 어디로 옮겨갔는지 보여줄 수 있다; 간단히 확인해 보되,
  요청받지 않았다면 깊이 파고들지 않는다 -- 아카이브는 느리다.

### 항목 폐기하기 {#retiring-an-entry}

- 항목을 제거한다
- [레지스트리와 목록 정보를 최신으로 유지하기](/ecosystem/registry/updating/)에
  따라, 항목을 처음 제출한 사람과/또는 항목을 업데이트한 작성자의 GitHub 핸들을
  PR 댓글에서 참조(Cc)한다.

<!-- prettier-ignore-start -->
[drift status]: /docs/contributing/localization/#drift-status
[link fixes and resource updates]:
  /docs/contributing/localization/#link-fixes-and-resource-updates
<!-- prettier-ignore-end -->
