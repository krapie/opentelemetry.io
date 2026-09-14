---
title: git 서브모듈 업데이트
description: >-
  저장소의 git 서브모듈 하나 이상을 대상 버전으로 업데이트하는 방법이다.
cSpell:ignore: gitlink gitlinks
default_lang_commit: 5fe7b747f871ea7cae510db7e078ceda4a90b992
---

각 서브모듈은 [.gitmodules][]의 `*-pin` 필드를 통해 태그나 커밋에 고정되어 있다.
아래 절차를 따라 서브모듈 하나 이상을 대상 버전으로 업데이트한다.

## 인자 {#arguments}

1. 업데이트할 서브모듈 하나 이상(경로 또는 이름). 업데이트 가능한 서브모듈은
   [.gitmodules][]에 선언되어 있으며, 다음을 포함한다:
   - `themes/docsy`, Docsy 테마
   - `content-modules/*`, 오픈텔레메트리(OpenTelemetry) 서브모듈

2. 대상 버전 지정자(각 서브모듈별 또는 전체에 대해):
   - 특정 태그 또는 커밋
   - 최신 태그된 릴리스를 나타내는 `latest`
   - 기본 브랜치의 head를 나타내는 `HEAD`

## 절차 {#process}

1. `npm run update:submodule`을 실행한다.

   > 태그를 가져오고 각 서브모듈을 원격 기본 브랜치의 head로 전환하여, 다음
   > 단계에서 고정 값을 로컬에서 결정할 수 있게 한다.

2. 서브모듈마다, 대상 버전으로부터 새 고정 값을 결정한다. git 명령은 해당
   서브모듈 안에서 실행한다:
   - **특정 태그 또는 커밋**: 주어진 그대로 사용하되,
     `git rev-parse --verify <tag-or-commit>`으로 존재를 검증한 뒤 사용한다. 이
     명령은 유효할 때 해당 SHA를 출력한다.

     > 유효한 태그 목록을 보려면 `git tag --list`를 실행한다.

   - **최신 릴리스**: `git describe --tags --abbrev=0`
   - **HEAD**: `HEAD`의 SHA[^sha]

   `HEAD`를 포함한 커밋 고정의 경우:
   - [`git describe --tags`][git-describe] `<commit>`이 성공하면, 그 출력을 고정
     값으로 사용한다. 가장 가까운 릴리스를 나타내기 때문이다 -- 예를 들어
     `ca090204` 대신 `v1.23.0-11-gca090204`이다.
   - 그렇지 않으면 `<commit>`의 SHA[^sha]를 사용한다.

3. [.gitmodules][]에서 서브모듈의 `*-pin` 값을 수정한다.
4. `npm run pin:submodule`을 실행해 서브모듈을 새 고정 값으로 전환하고, 보고된
   리비전(`git submodule`로도 확인할 수 있다)이 고정 값과 일치하는지 검증한다.
5. `.gitmodules`와 서브모듈 gitlink 변경 사항을 커밋한다. 단일 모듈 업데이트라면
   "Update `<submodule>` to `<version>`"과 같은 커밋 메시지를 사용하고, 그렇지
   않으면 이와 비슷한 메시지를 사용한다. 지금 커밋해 두면, 다음 단계가
   서브모듈을 이전에 커밋된 gitlink로 되돌리는 것을 막을 수 있다.
6. `npm run prepare`를 실행해 서브모듈 의존성을 새로 고친다. 그 결과로 생긴 변경
   사항을 커밋하며, 원한다면 이전 커밋에 amend한다.

검증:

- 빠른 검증을 위해서는 `npm run build`를 실행해 오류나 (예상치 못한) 경고 없이
  성공하는지 확인한다.
- 포괄적인 검증을 위해서는 `npm run test`를 실행해 오류 없이 통과하는지
  확인한다.

[^sha]: 전체 SHA가 필요한 경우도 있지만, 대개는 짧은 SHA로 충분하다.

[.gitmodules]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.gitmodules
[git-describe]: https://git-scm.com/docs/git-describe
