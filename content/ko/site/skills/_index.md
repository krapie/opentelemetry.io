---
title: 에이전트와 관리자를 위한 스킬
linkTitle: 스킬
description: 사이트를 유지보수할 때 에이전트와 관리자가 사용하는 스킬이다.
weight: 22
cSpell:ignore: agentskills
default_lang_commit: 92dd44d5998fe30453368880929dc86dd9ba84da
---

이 섹션은 사이트를 유지보수할 때 에이전트와 관리자가 사용하는 스킬과 절차를
설명한다.

재사용 가능한 동작으로서 [agentskills.io][]를 준수해 작성되어 에이전트가
호출하거나 관리자가 수동으로 따를 수 있는 것을 **에이전트 스킬(agent skill)**
이라는 용어로 지칭한다. 특정 작업을 완료하기 위해 에이전트나 관리자가 따를 수
있는 일련의 단계를 **_관리자_ 스킬(maintainer skill)**(또는 관리자 절차)이라고
부른다. 에이전트 스킬은 [`.claude/skills/`][]에 정의되어 있다. 관리자 절차는 이
섹션에 정의되어 있다.

스킬과 절차의 단계에서, 본문은 각 동작의 의도를 서술한다. 괄호 안에 주어진
명령은 그 단계를 수행하는 하나의 제안된 방법일 뿐, 유일하게 유효한 방법은
아니다. 이 관례가 도입되기 전에 작성된 스킬은 아직 이를 따르지 않을 수 있다.

## 에이전트 스킬 {#agent-skills}

위에서 언급했듯이, 스킬은 [`.claude/skills/`][]에 정의되어 있으며, 다음과 같다:

- [`/approve-registry-update [pr-number-or-url]`][approve-registry-update]:
  리뷰어가 otelbot 레지스트리 버전 업데이트(version-bump) PR을 병합할지
  판단하도록 돕는다. 클린 업데이트(clean bump)인지 확인하고, 확인되면 승인한 뒤
  병합 대기열(merge queue)에 추가한다. 인자가 없으면 열려 있는 레지스트리 자동
  업데이트 PR을 모두 처리한다.
- [`/draft-issue <issue-description>`][draft-issue]: 이슈 템플릿, 기여
  가이드라인, 레이블 분류 체계(taxonomy)를 따라 `opentelemetry.io` 저장소에
  GitHub 이슈 초안을 작성한다.
- [`/refresh-link-cache-pr-fix`][refresh-link-cache-pr-fix]: otelbot PR에서
  2XX가 아닌 URL을 가져와 검토하고 수정을 시도한다(기본적으로 링크 검사에 실패한
  모든 열려 있는 `otelbot/*` PR을 대상으로 하며, 지시가 있으면 특정 브랜치를
  대상으로 한다).
- [`/resolve-link-cache-conflicts <optional-pr-number>`][resolve-link-cache-conflicts]:
  `.lycheecache` 병합/리베이스 충돌을 해결한다.
- [`/review-blog-post <blog-post-path-or-pr-number>`][review-blog-post]:
  프론트매터(front matter) 준수 여부, 콘텐츠 관례, GitHub 링크 안정성
  (`gh-url-hash`), 맞춤법, OTel 용어 사용을 기준으로
  오픈텔레메트리(OpenTelemetry) 블로그 게시물을 검토한다.
- [`/review-pull-request <pr-number-or-url>`][review-pull-request]: CI 점검의
  의미, CLA 및 승인 레이블(approval-label) 워크플로, 링크 캐시 처리, 로케일
  규칙, 콘텐츠 품질을 기준으로 풀 리퀘스트(pull request)를 검토한다.
- [`/setup-new-localization <kickoff-issue | lang-code>`][setup-new-localization]:
  새 웹사이트 로컬라이제이션(localization)을 처음부터 끝까지 설정한다 — Hugo
  언어 블록, 콘텐츠 마운트, cSpell 단어 목록, `lang:<lang>` 레이블러(labeler)
  설정 및 레이블, CODEOWNERS 재생성을 포함한 `locale-teams.yaml`,
  `localization.md` 항목이다.
- [`/update-i18n-drift-status [--locale locale,...] [--create-pr]`][update-i18n-drift-status]:
  로컬라이즈된 콘텐츠의 `drifted_from_default` 프론트매터 필드를 업데이트한다.
  선택적 인자로 처리할 로케일을 제한하거나 PR을 자동으로 열지 여부를 지정할 수
  있다.
- [`/update-old-blog-ignores`][update-old-blog-ignores]: 린트/포맷 점검과 수정
  스크립트에서 제외되는 오래된 블로그 게시물의 연도 범위를 갱신한다.
- [`/update-git-submodule <submodule>... <version|latest|HEAD>`][update-git-submodule]:
  하나 이상의 git 서브모듈(submodule)을 대상 버전으로 업데이트한다.

일부 에이전트 채팅에서는 `/`를 입력한 뒤 스킬 이름을 입력하는 방식으로 스킬을
호출할 수 있다.

## 훅(Hooks) {#hooks}

위의 에이전트 스킬과 더불어, [hooks][]는 특정 도구 이벤트에서 자동으로 실행된다.
설정은 [`.claude/hooks/hooks.json`][hooks-json]에 있고, 훅 소스 코드는
[`scripts/validate/`][validate] 아래에 있다.

- **블로그 프론트매터(front matter) 점검**: `content/en/blog/**/*.md`에 대한
  변경 사항 중 프론트매터에 필수 필드가 빠져 있거나, 날짜 형식이 잘못되었거나,
  H1 헤딩이 추가된 경우 이를 차단하는, `Write`와 `Edit`에 대한 `PreToolUse`
  훅이다. 리뷰를 기다리지 않고 작성 시점(write-time)에
  [`/review-blog-post`](#agent-skills)와 동일한 관례를 적용한다. 소스:
  [`scripts/validate/front-matter-check/`][frontmatter-check]. 순수 로직은
  `index.mjs`에 있으며, 같은 폴더의 `index.test.mjs`가 이를 커버한다
  (`npm run test:local-tools`로 실행할 수 있다).

## 관리자 스킬 {#maintainer-skills}

아래 섹션 색인을 참고한다.

[`.claude/skills/`]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/.claude/skills
[agentskills.io]: https://agentskills.io
[approve-registry-update]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/approve-registry-update/SKILL.md
[draft-issue]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/draft-issue/SKILL.md
[refresh-link-cache-pr-fix]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/refresh-link-cache-pr-fix/SKILL.md
[resolve-link-cache-conflicts]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/resolve-link-cache-conflicts/SKILL.md
[review-blog-post]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/review-blog-post/SKILL.md
[review-pull-request]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/review-pull-request/SKILL.md
[setup-new-localization]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/setup-new-localization/SKILL.md
[update-i18n-drift-status]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/update-i18n-drift-status/SKILL.md
[update-old-blog-ignores]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/update-old-blog-ignores/SKILL.md
[update-git-submodule]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/skills/update-git-submodule/SKILL.md
[hooks]: https://docs.claude.com/en/docs/claude-code/hooks
[hooks-json]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.claude/hooks/hooks.json
[validate]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/scripts/validate
[frontmatter-check]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/scripts/validate/front-matter-check
