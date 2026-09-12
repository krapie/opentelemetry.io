---
title: NPM 스크립트
description: >-
  오픈텔레메트리(OpenTelemetry) 웹사이트를 빌드하고, 서비스를 실행하고,
  검증하고, 유지보수하기 위한 npm 스크립트이다.
weight: 20
todo: Keep table entries sorted
default_lang_commit: 16c5273feefbfbb0fdd57f00986c8daaea0ecd18
---

스크립트 정의는 저장소 루트의 [`package.json`][]에 있다. 스크립트를 실행하려면
`npm run` _`SCRIPT_NAME`_ 형태로 사용한다.

## 명명 규칙 {#nomenclature}

- **내부 스크립트(Internal scripts)**
  - `_`로 시작하는 이름은 내부 헬퍼이며 직접 실행하도록 만들어지지 않았다.
  - `NAME::pre`와 `NAME::post` 스크립트도 마찬가지로, `NAME`의 사전/사후
    단계로서 명시적으로 호출된다.
- **기본 변형과 `:all` 변형**
  - **`check`**, **`fix`**, **`test`** 스크립트는 각 작업에서 가장 흔히 필요한
    하위 스크립트를 실행한다.
  - **`*:all`** 변형인 `check:all`, `fix:all`, `test:all`은 더 넓은 범위의 하위
    스크립트 집합을 실행한다. 각 변형의 범위는 표 항목에 설명되어 있다.

## 의존성 설치 및 업데이트 {#installing-and-updating-dependencies}

| 스크립트          | 설명                                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `ci:min`          | CI용 [잠금 파일 기준 무동작 설치][lock-exact inert install]이다. 라이프사이클 스크립트가 없다.                                        |
| `ci:prepare`      | `ci:min` 이후 설정 단계이다: [고정된 Hugo 바이너리를 가져온 뒤][fetch the pinned Hugo binary] `prepare`를 실행한다.                   |
| `install:safe`    | [잠금 파일 기준 로컬 설정][lock-exact local setup]이다: 무동작 설치 후 `ci:prepare`를 실행한다.                                       |
| `prepare`         | 설치 단계이다: `get:submodule`을 실행한 뒤 Docsy의 [잠금 파일 기준 테마 의존성 설치][lock-exact theme dependency install]를 수행한다. |
| `update:hugo`     | 최신 hugo-extended를 설치한다. 버전을 올리면서 해당 [`allowScripts` 승인 항목][`allowScripts` approval]도 함께 업데이트한다.          |
| `update:packages` | npm-check-updates를 실행해 의존성 버전을 올린다. [릴리스 쿨다운(release cooldown)][release cooldown]이 적용된다.                      |

## 빌드 및 서비스 실행 {#build-and-serve}

| 스크립트           | 설명                                                                       |
| ------------------ | -------------------------------------------------------------------------- |
| `build:full`       | full 사이트를 빌드한다. 자세한 내용은 [빌드 종류][build kinds]를 참고한다. |
| `build:lean`       | 사이트를 lean 빌드한다. 자세한 내용은 [빌드 종류][build kinds]를 참고한다. |
| `build:preview`    | 축소(minification)를 적용한 full 빌드이다(예: Netlify 미리보기용).         |
| `build:production` | 축소(minification)를 적용한 운영용 Hugo 빌드이다.                          |
| `build`            | 사이트를 빌드한다. 기본값은 lean이다. [빌드 종류][build kinds]를 참고한다. |
| `clean`            | `make clean`을 실행한다.                                                   |
| `serve:hugo`       | 메모리 내(in-memory) 렌더링으로 Hugo 서버를 시작한다.                      |
| `serve`            | Hugo 개발 서버를 시작한다(기본값; full 렌더링).                            |

## 검사 {#checking}

| 스크립트               | 설명                                                                           |
| ---------------------- | ------------------------------------------------------------------------------ |
| `check:all`            | 모든 검사 스크립트를 순서대로 실행한다.                                        |
| `check:code-excerpts`  | 코드 발췌(excerpt)를 검사하고, 업데이트가 필요하면 실패시킨다.                 |
| `check:codeowners`     | CODEOWNERS 로케일 섹션이 레지스트리와 일치하는지 검증한다.                     |
| `check:collector-sync` | collector-sync 검사를 실행한다.                                                |
| `check:expired`        | 만료된 콘텐츠를 (프론트매터 기준으로) 나열한다.                                |
| `check:filenames`      | [파일 이름 규칙을 검증하고 오래된 파일/폴더를 감지한다][fn].                   |
| `check:format`         | Prettier 및 prose-wrap 검사이다.                                               |
| `check:i18n`           | 로컬라이제이션 프론트매터(`default_lang_commit`)를 검증한다.                   |
| `check:l10n`           | 로컬라이제이션 검사를 실행한다.                                                |
| `check:links:diff`     | 변경된 파일만 대상으로 Lychee 링크 검사를 실행한다.                            |
| `check:links:internal` | 오프라인 링크 검사이다(내부 링크만 대상); 먼저 lean 빌드를 수행한다.           |
| `check:links`          | Lychee로 사이트 전체를 [링크 검사][link check]한다; 먼저 lean 빌드를 수행한다. |
| `check:markdown:specs` | `tmp/`에 있는 스펙 프래그먼트에 대한 Markdown 린트이다.                        |
| `check:markdown`       | Markdown 린트이다(콘텐츠와 프로젝트 대상).                                     |
| `check:registry`       | `data/registry/` 아래의 레지스트리 YAML을 검증한다.                            |
| `check:spelling`       | 콘텐츠, 데이터, 레이아웃 Markdown에 대해 cspell을 실행한다.                    |
| `check:text`           | 콘텐츠와 데이터에 대해 textlint를 실행한다.                                    |
| `check`                | 가장 흔히 필요한 검사 스크립트를 순서대로 실행한다.                            |

## 수정 {#fixing}

| 스크립트                      | 설명                                                                 |
| ----------------------------- | -------------------------------------------------------------------- |
| `fix`                         | 가장 흔히 필요한 수정 스크립트를 실행한다.                           |
| `fix:code-excerpts`           | 코드 발췌(excerpt)를 새로 고친다.                                    |
| `fix:codeowners`              | 레지스트리로부터 CODEOWNERS 로케일 섹션을 재생성한다.                |
| `fix:all`                     | 모든 수정 스크립트를 실행한다.                                       |
| `fix:format`                  | Prettier를 적용하고 후행 공백을 정리한다.                            |
| `fix:format:staged`           | 스테이징된 파일만 포맷한다.                                          |
| `fix:i18n`                    | i18n 프론트매터를 추가/수정한다(`fix:i18n:new`, `fix:i18n:status`).  |
| `fix:l10n`                    | 로컬라이제이션 수정 사항을 적용한다.                                 |
| `fix:link-cache`              | 링크를 검사하며 커밋된 [`.lycheecache`][]를 업데이트한다.            |
| `fix:link-cache:double-check` | [브라우저 프로브로 실패한 링크를 다시 검증한다][dc].                 |
| `fix:link-cache:refresh`      | 가장 오래된 캐시 항목을 정리한 뒤 `fix:link-cache`를 실행한다.       |
| `fix:markdown`                | Markdown 린트 문제와 후행 공백을 수정한다.                           |
| `fix:submodule`               | 서브모듈 리비전을 업데이트하고, 재고정(re-pin)한 뒤 목록을 표시한다. |
| `fix:filenames`               | [파일 이름을 변경하고 오래된 파일/폴더를 제거한다][fn].              |
| `fix:dict`                    | cspell 단어 목록을 정렬하고 프론트매터를 정규화한다.                 |
| `fix:expired`                 | `check:expired`가 보고한 파일을 삭제한다.                            |
| `fix:text`                    | `--fix` 옵션으로 textlint를 실행한다.                                |
| `fix:collector-sync:lint`     | collector-sync에서 `--fix` 옵션으로 ruff를 실행한다.                 |
| `format`                      | Prettier write의 별칭이다(콘텐츠 및 no-wrap 경로 대상).              |

## 서브모듈과 콘텐츠 {#submodules-and-content}

| 스크립트           | 설명                                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `code-excerpts`    | 코드 발췌(excerpt)를 새로 고친다. 지원 중단(DEPRECATED): `fix:code-excerpts` 또는 `check:code-excerpts`를 사용한다. |
| `cp:spec`          | 스펙 콘텐츠를 복사한다(content-modules).                                                                            |
| `get:submodule`    | git 서브모듈을 초기화/업데이트한다(건너뛰려면 `GET=no`로 설정).                                                     |
| `pin:submodule`    | 서브모듈 리비전을 고정한다(선택적으로 `PIN_SKIP`).                                                                  |
| `schemas:update`   | 오픈텔레메트리 스펙 서브모듈과 콘텐츠를 업데이트한다.                                                               |
| `update:submodule` | 서브모듈을 최신 원격 상태로 업데이트하고 태그를 가져온다.                                                           |

## 테스트와 CI {#test-and-ci}

| 스크립트                   | 설명                                                                                |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `diff:check`               | 작업 트리에 커밋되지 않은 변경 사항이 있으면 경고한다.                              |
| `diff:fail`                | 작업 트리에 변경 사항이 있으면 실패시킨다(예: 빌드 이후).                           |
| `fix-and-test:all`         | i18n을 포함한 모든 수정을 실행한 뒤 검사를 수행한다; 링크는 한 번만 검사한다.[^fat] |
| `is:clean`                 | Git 작업 트리에 변경 사항이 있으면(추적되지 않는 파일 포함) 실패시킨다.             |
| `netlify-build:preview`    | Netlify 배포 미리보기를 빌드한다.                                                   |
| `netlify-build:production` | Netlify 운영 사이트를 빌드한다.                                                     |
| `test-and-fix`             | 수정 스크립트를 실행한 뒤(i18n/link-cache/submodule 제외) 검사를 수행한다.          |
| `test:all`                 | `test:base`를 실행한 뒤 `test:compound-tests`를 실행한다.                           |
| `test:base`                | 기본 테스트이다(`check`와 동일하다).                                                |
| `test:collector-sync`      | collector-sync 테스트이다.                                                          |
| `test:compound-tests`      | 복합적인 `test:*-*` 스크립트를 실행한다.[^categories]                               |
| `test:double-check:live`   | [double-check 프로브][dc]의 라이브 스모크(smoke) 테스트이다.                        |
| `test:edge-functions:live` | 선택적 `node:test` 라이브 테스트 스위트이다; `--help`를 지원한다.                   |
| `test:edge-functions`      | `netlify/edge-functions/**/*.test.ts`에 대한 Node 테스트 러너이다.                  |
| `test:local-tools`         | `scripts/**/*.test.mjs`에 대한 Node 테스트 러너이다.[^categories]                   |
| `test:local-tools:lychee`  | `test:local-tools`의 Lychee 바이너리 슬라이스이다; 바이너리가 없으면 건너뛴다.      |
| `test:public`              | 빌드된 사이트에 대해 `tests/public/` 검사를 실행한다.[^categories]                  |
| `test`                     | 가장 흔히 필요한 테스트를 실행한다.                                                 |

[^categories]:
    이 스크립트들은 테스트 스크립트 명명 규칙을 따른다. 자세한 내용은
    [테스트 카테고리](../../testing/#test-categories)를 참고한다.

[^fat]:
    하우스키핑(housekeeping) 기본값이다: 콘텐츠 수정 이후 `fix:link-cache`(링크
    검사 및 링크 캐시 새로 고침)를 실행하며, 모든 수정 사항이 반영되도록 계속
    진행하는(keep-going) `all` 러너를 사용한다. 검사 단계는
    `check:links`(`fix:link-cache`가 이를 대신 처리한다)와
    `check:i18n`(`fix:i18n` 이후 드리프트 상태가 기록되므로 중복이다)을
    제외한다. [하우스키핑](../ci-workflows/#housekeeping)을 참고한다.

## 유틸리티 {#utilities}

| 스크립트                       | 설명                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------ |
| `all`                          | 일부가 실패하더라도 주어진 모든 스크립트를 실행한다; 하나라도 실패하면 0이 아닌 값으로 종료한다. |
| `generate:config:links`        | `lychee.base.toml`과 페이지 프론트매터로부터 git-ignore된 `lychee.toml`을 생성한다.              |
| `locale-auto-merge`            | [로케일 자동 병합 헬퍼 CLI][locale-auto-merge]이다(`--help`).                                    |
| `log:build`, `log:check:links` | 해당 스크립트를 실행하고, 출력을 `tmp/`로 tee하며, 스크립트의 종료 코드를 그대로 전달한다.       |
| `seq`                          | 주어진 스크립트 이름을 순서대로 실행한다; 처음 실패하면 종료한다.                                |

<!-- prettier-ignore-start -->
[`allowScripts` approval]: ../dependencies/#script-bearing-packages
[`.lycheecache`]: ../link-checking/#link-cache
[`package.json`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/package.json
[build kinds]: ../#build-kinds
[dc]: ../link-checking/#double-check
[fetch the pinned Hugo binary]: ../dependencies/#install-contracts
[fn]: /docs/contributing/pr-checks/#filename-check
[link check]: ../link-checking/
[locale-auto-merge]: ../ci-workflows/#locale-auto-merge
[lock-exact inert install]: ../dependencies/#install-contracts
[lock-exact local setup]: ../dependencies/#install-contracts
[lock-exact theme dependency install]: ../dependencies/#install-contracts
[release cooldown]: ../dependencies/#release-cooldown
<!-- prettier-ignore-end -->
