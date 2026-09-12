---
title: 헬퍼 스크립트
description: >-
  레이블 관리, 링크 검사, 레지스트리 업데이트 등을 위해 CI 워크플로와 로컬 개발
  환경에서 사용되는 셸 스크립트이다.
weight: 30
default_lang_commit: 3a24363da99b1bf022d730fd21816279bd892869
---

모든 스크립트는
[`.github/scripts/`](https://github.com/open-telemetry/opentelemetry.io/tree/main/.github/scripts)
아래에 있다.

## check-i18n-helper.sh {#check-i18n-helpersh}

로컬라이제이션 페이지에 필수 `default_lang_commit` 프론트매터 필드가 포함되어
있는지 검증한다. 페이지에 이 필드가 없으면 스크립트가 수정 명령을 출력한다:

```sh
npm run fix:i18n:new
```

## pr-approval-labels.sh {#pr-approval-labelssh}

리뷰 상태와 파일 소유권을 기준으로 PR 승인 레이블을 관리한다.
[`label-manager` 워크플로](../ci-workflows/#pr-approval-labels)에서 호출된다.

**동작 방식:**

1. `gh`를 통해 PR 데이터(변경된 파일, 최신 리뷰, 현재 레이블)를 가져온다.
2. GitHub 조직 API에서 `docs-approvers` 팀 멤버를 조회한다.
3. 변경된 파일을 [`.github/component-owners.yml`][owners]과 대조해 필요한 SIG
   팀을 결정한다(`yq` 의존성 없이 YAML을 수동으로 파싱한다).
4. 필요한 각 그룹이 승인 리뷰를 보유하고 있는지 확인한다.
5. 팀 멤버십을 가져올 수 없을 때 레이블이 변경되지 않도록, 3중 상태(tri-state)
   로직(`true`/`false`/`unknown`)을 사용해 레이블을 추가하거나 제거한다.

[owners]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/component-owners.yml

**필수 환경 변수:** `REPO`, `PR`, `GITHUB_TOKEN`.

## update-registry-versions.sh {#update-registry-versionssh}

업스트림 레지스트리를 조회해 `data/registry/*.yml`의 패키지 버전을 자동으로
업데이트한다. npm, Packagist, RubyGems, Go, NuGet, Hex, Maven을 지원한다.

- CI에서는(`GITHUB_ACTIONS`가 설정된 경우) 브랜치를 생성하고 PR을 연다.
- 로컬에서는 기본적으로 **dry-run** 모드로 실행된다. 실제로 실행하려면 `-f`를
  사용한다.

업데이트 요약으로부터 SHA-1 태그를 생성해 PR 중복 생성을 방지한다.

버전이 변경되면, 레지스트리 업데이트로 인해 외부 URL이 추가되거나 제거될 수
있으므로 커밋하기 전에 스크립트가 [링크 캐시][link cache]를 새로 고친다. 이 새로
고침 과정에서 일시적인 실패가 발생하면 링크는 통과했더라도 봇 PR의
`CACHE updates committed?`가 레드로 남을 수 있다. 이 경우 PR에
[`/fix:link-cache`][]를 댓글로 남겨 복구한다. 반대로 업데이트가 실제로 깨진
URL을 유입시켰다면 봇 PR은 링크 검사 자체에서 레드로 표시된다. 이때는 캐시
수정을 다시 실행하는 대신 URL을 수정한다.

<!-- prettier-ignore-start -->
[`/fix:link-cache`]: /docs/contributing/pull-requests/#fixing-prs-in-github
[link cache]: ../link-checking/#link-cache
<!-- prettier-ignore-end -->
