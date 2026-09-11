---
title: 블로그
description: 블로그 게시물을 제출하는 방법을 알아본다.
weight: 30
default_lang_commit: 9a8128b38643d2c42b1d249b42c29ebe23c6c2b7
---

[오픈텔레메트리 블로그](/blog/)는 새로운 기능, 커뮤니티 보고서, 그리고
오픈텔레메트리 커뮤니티와 관련 있을 만한 소식을 전한다. 여기에는 최종 사용자와
개발자가 모두 포함된다. 누구나 블로그 게시물을 작성할 수 있으며, 아래에서 요구
사항이 무엇인지 읽어본다.

## 문서인가, 블로그 게시물인가? {#documentation-or-blog-post}

블로그 게시물을 작성하기 전에, 자신의 콘텐츠가 문서에 추가해도 좋은 내용은
아닌지 자문해본다. 답이 "예"라면 콘텐츠를 문서에 추가할 수 있도록 새 이슈나 풀
리퀘스트(PR)를 생성한다.

오픈텔레메트리 웹사이트의 관리자와 승인자는 프로젝트 문서를 개선하는 데 초점을
맞추고 있으므로, 블로그 게시물은 리뷰 우선순위가 낮다는 점에 유의한다.

## 소셜 미디어 콘텐츠 요청 {#social-media-content-request}

블로그 대신 오픈텔레메트리의 소셜 미디어 채널에 게시를 요청하려면 [소셜 미디어
요청을 제출한다][submit a social media request].

[submit a social media request]:
  https://github.com/open-telemetry/community/issues/new?template=social-media-request.yml

## 블로그 게시물 제출 전에 {#before-submitting-a-blog-post}

블로그 게시물은 상업적인 성격이어서는 안 되며, 오픈텔레메트리 커뮤니티에 폭넓게
적용되는 독창적인 콘텐츠로 구성되어야 한다. 블로그 게시물은 [소셜 미디어
가이드][Social Media Guide]에 명시된 정책을 따라야 한다.

[Social Media Guide]:
  https://github.com/open-telemetry/community/blob/main/social-media-guide.md

### GitHub 저장소로 링크 걸기 {#linking-to-github-repositories}

블로그 게시물은 불안정한 GitHub `blob`/`tree` 링크를 방지하기 위해
[markdownlint로 체크된다][checked by markdownlint](`gh-url-hash`).

체크에서 문제가 보고되면 다음과 같이 한다.

- 기본 브랜치 참조(예: `main`/`master`)를 태그/릴리스나 커밋 해시로 교체한다.
- 전체 40자 커밋 해시를 사용한다(짧은 해시는 문제로 표시된다).
- `npm run fix:markdown`을 실행해 자동으로 고칠 수 있는 부분을 고친 다음, 나머지
  보고된 링크를 수동으로 고친다.

의도한 콘텐츠가 오픈텔레메트리 커뮤니티에 폭넓게 적용되는지 확인한다. 적절한
콘텐츠에는 다음이 포함된다.

- 새로운 오픈텔레메트리 기능
- 오픈텔레메트리 프로젝트 업데이트
- 특수 관심 그룹(Special Interest Group, SIG)의 업데이트
- 튜토리얼과 안내
- 오픈텔레메트리 통합(Integration)
- [기여자 모집](#call-for-contributors)

부적절한 콘텐츠에는 다음이 포함된다.

- 벤더 제품 홍보

자신의 블로그 게시물이 적절한 콘텐츠 목록에 해당한다면, **반드시** 먼저 다음
세부 사항을 담아
[이슈를 등록해야 한다](https://github.com/open-telemetry/opentelemetry.io/issues/new?template=BLOG_POST.yml).

- 블로그 게시물의 제목
- 블로그 게시물의 짧은 설명과 개요
- 해당된다면, 블로그 게시물에서 사용한 기술을 나열한다. 모두 오픈 소스인지
  확인하고, CNCF 프로젝트가 아닌 것보다는 CNCF 프로젝트를 우선한다(예: 트레이스
  시각화에는 Jaeger, 메트릭 시각화에는 Prometheus를 사용한다).
- 자신의 블로그 게시물을 후원할
  [SIG](https://github.com/open-telemetry/community/)의 이름. **SIG 후원자가
  필요하다.**
- 해당 SIG에서 후원자(관리자 또는 승인자)의 이름을 적는다. 이 후원자는 Comms
  SIG가 게시물을 리뷰하기 전에 첫 번째 라운드의 리뷰를 진행한다. 이 후원자는
  저자와 **다른** 회사 소속이어야 **한다**.

**풀 리퀘스트를 제출하기 전에 이슈를 여는 것은 필수이다.** 승인된 이슈가 먼저
없었던 블로그 게시물의 풀 리퀘스트는 추가 리뷰 없이 닫힐 수 있다.

SIG Communication의 관리자는 블로그 게시물이 승인을 위한 모든 요구 사항을
충족하는지 검증한다. Comms SIG가 게시물을 살펴보기 전에 SIG 후원자가 먼저 리뷰를
완료해야 한다. 블로그 게시물의 저자는 후원자를 찾을 책임이 있다.

이슈에 필요한 모든 내용이 담겨 있다면, 관리자가 블로그 게시물을 제출해도 좋다고
검증해준다.

### 기여자 모집 {#call-for-contributors}

새 프로젝트나 SIG의 생성을 제안하거나, 오픈텔레메트리 프로젝트에 기부를 제안하는
경우, 제안을 성공시키려면 추가 기여자가 필요하다. 이를 돕기 위해 "기여자
모집(Call for Contributors, CfC)" 블로그 게시물을 제안할 수 있다.

이를 위해서는
[새 프로젝트](https://github.com/open-telemetry/community/blob/main/project-management.md)와
[기부](https://github.com/open-telemetry/community/blob/main/guides/contributor/donations.md)를
위한 절차를 따라야 한다.

## 블로그 게시물 제출하기 {#submit-a-blog-post}

이 저장소를 포크해 로컬에서 작성하거나 GitHub UI를 사용해 블로그 게시물을 제출할
수 있다. 두 경우 모두
[블로그 게시물 템플릿](https://github.com/open-telemetry/opentelemetry.io/tree/main/archetypes/blog.md)에서
제공하는 안내를 따라야 한다.

### 포크한 다음 로컬에서 작성하기 {#fork-and-write-locally}

로컬 포크를 설정한 후에는 템플릿을 사용해 블로그 게시물을 만들 수 있다. 다음
단계를 따라 템플릿에서 게시물을 생성한다.

1. 저장소 루트에서 다음 명령어를 실행한다.

   ```sh
   npm exec --no -- hugo new content/en/blog/$(date +%Y)/short-name-for-post.md
   ```

   게시물에 이미지나 그 밖의 애셋이 있다면 다음 명령어를 실행한다.

   ```sh
   npm exec --no -- hugo new content/en/blog/$(date +%Y)/short-name-for-post/index.md
   ```

2. 이전 명령어에서 제공한 경로에 있는 마크다운 파일을 편집한다. 이 파일은
   [archetypes](https://github.com/open-telemetry/opentelemetry.io/tree/main/archetypes/)의
   블로그 게시물 스타터에서 초기화된다.

3. 이미지나 그 밖의 파일과 같은 애셋을 만들어둔 폴더에 넣는다.

4. 게시물이 준비되면 풀 리퀘스트를 통해 제출한다.

### GitHub UI 사용하기 {#use-the-github-ui}

로컬 포크를 만들고 싶지 않다면 GitHub UI를 사용해 새 게시물을 만들 수 있다. 다음
단계를 따라 UI로 게시물을 추가한다.

1.  [블로그 게시물 템플릿](https://github.com/open-telemetry/opentelemetry.io/tree/main/archetypes/blog.md)으로
    이동해 메뉴 오른쪽 위에 있는 **Copy raw content**를 클릭한다.

1.  [Create a new file](https://github.com/open-telemetry/opentelemetry.io/new/main)을
    선택한다.

1.  첫 번째 단계에서 복사한 템플릿의 내용을 붙여넣는다.

1.  파일 이름을 정한다. 예를 들어(`YYYY`는 현재 연도이다):

    `content/en/blog/YYYY/short-name-for-your-blog-post/index.md`.

1.  GitHub에서 마크다운 파일을 편집한다.

1.  게시물이 준비되면 **Propose changes**를 선택하고 안내를 따른다.

## 게시 일정 {#publication-timelines}

오픈텔레메트리 블로그는 엄격한 게시 일정을 따르지 않는다. 이는 다음을 의미한다.

- 블로그 게시물은 필요한 모든 승인을 받으면 게시된다.
- 필요하다면 게시가 연기될 수 있지만, 관리자는 특정 날짜에 또는 그 전에
  게시된다고 보장할 수 없다.
- 특정 블로그 게시물(주요 발표)은 우선순위가 높아 자신의 블로그 게시물보다 먼저
  게시될 수 있다.

## 블로그 콘텐츠 교차 게시하기 {#cross-posting-blog-content}

자신의 오픈텔레메트리 블로그 게시물을 다른 플랫폼에서 공유하고 싶다면 그렇게
해도 좋다. 다음 사항만 유의한다.

- 어떤 버전을 정본(canonical post)으로 할지 정한다(일반적으로 원본
  오픈텔레메트리 블로그 게시물).
- 게시물의 다른 버전은 다음과 같이 해야 한다.
  - 원본 게시물이 오픈텔레메트리 블로그에 게시되었음을 명확히 언급한다.
  - 페이지 위쪽이나 아래쪽에 원본으로 돌아가는 링크를 포함한다.
  - 플랫폼이 지원한다면 오픈텔레메트리 블로그 게시물을 가리키는 정본
    URL(canonical URL) 태그를 설정한다.

이는 적절한 출처 표기를 보장하고, SEO 모범 사례를 지원하며, 콘텐츠 중복을
방지하는 데 도움이 된다.

## 오래된 블로그는 업데이트되지 않는다 {#old-blogs-are-not-updated}

블로그 게시물은 역사적인 기록으로 간주되며, 1년 정도가 지나면 업데이트되지
않는다(사이트 빌드를 보장하기 위한 필수적인 변경은 예외). 오래된 블로그
게시물에는 콘텐츠가 오래되었을 수 있고 일부 링크가 유효하지 않을 수 있음을
독자에게 경고하는 배너가 상단에 표시된다.

또한 오래된 게시물은 [린트도, 링크 체크도 되지 않는다][pr-checks].

[pr-checks]: ../pr-checks/#checks
[checked by markdownlint]: ../pr-checks/#markdown-linter
