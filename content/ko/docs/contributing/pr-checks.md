---
title: 풀 리퀘스트 체크와 테스트
linkTitle: PR 체크 & 테스트
description:
  풀 리퀘스트가 모든 체크를 성공적으로 통과하게 만드는 방법을 알아본다.
weight: 40
default_lang_commit: 9a8128b38643d2c42b1d249b42c29ebe23c6c2b7
---

[opentelemetry.io 저장소](https://github.com/open-telemetry/opentelemetry.io)에
[풀 리퀘스트(pull request)](https://docs.github.com/en/get-started/learning-about-github/github-glossary#pull-request)(PR)를
등록하면 일련의 체크가 실행된다. PR 체크는 다음을 검증한다.

- [CLA](#easy-cla)에 서명했는지
- PR이 성공적으로 [Netlify를 통해 배포되는지](#netlify-deployment)
- 변경 사항이 [스타일 가이드](#checks)를 준수하는지

> [!NOTE]
>
> PR 체크 중 하나라도 실패하면, 먼저 로컬에서 `npm run fix`를 실행해
> [콘텐츠 문제를 고쳐본다](../pull-requests/#fix-issues).
>
> PR에 `/fix` 댓글을 추가할 수도 있다. 그러면 OpenTelemetry Bot이 대신 그
> 명령어를 실행하고 PR을 업데이트한다. 해당 변경 사항을 로컬로 반드시 pull
> 받는다.
>
> 문제가 계속되는 경우에만, 아래에서 각 체크가 무엇을 하는지와 실패한 상태에서
> 복구하는 방법을 읽어본다.

## `Easy CLA` {.notranslate lang=en}

이 체크는 [CLA에 서명하지](../prerequisites/#cla) 않으면 실패한다.

## Netlify 배포 {#netlify-deployment}

[Netlify](https://www.netlify.com/) 빌드가 실패하면 자세한 내용을 보려면
**Details**를 선택한다.

`"build.command" failed` 직전에 나타나는 `?? some/path` 상태 줄이 자신의 PR이
건드리지 않은 경로를 가리킨다면, 이는 자신의 변경 사항이 아니라 오래된 Netlify
빌드 캐시 때문에 빌드가 실패했을 가능성을 시사한다.

1. PR 댓글을 통해 관리자에게 캐시 없이 빌드를 재시도해달라고 요청한다. 캐시를
   지우는 절차는 [의존성 관리](/site/build/dependencies/#netlify-build-cache)를
   참고한다.
2. 캐시 없이 재시도했는데도 실패가 재현된다면, 캐시가 원인이 아니었던 것이다.
   빌드의 어떤 단계가 그 경로에 쓰고 있는 것이며, 자신의 변경 사항이 첫 번째
   용의자이다.

## GitHub PR 체크 {#checks}

기여 내용이 [스타일 가이드](../style-guide/)를 따르는지 확인하기 위해, 스타일
가이드 규칙을 검증하고 문제가 있으면 실패하는 일련의 체크를 구현해두었다.

아래 항목에서는 현재 체크와 관련 오류를 고치기 위해 할 수 있는 일을 설명한다.

> [!NOTE]
>
> 최근 블로그 게시물만 체크된다. 자세한 내용은 [오래된 블로그는 업데이트되지
> 않는다][old-blogs]를 참고한다. 특히, 오래된 게시물은 웹사이트에 렌더링되지만
> 아래에 나열된 체크는 오래된 블로그에 적용되지 않는다.

[old-blogs]: ../blog/#old-blogs-are-not-updated

### `TEXT linter` {.notranslate lang=en}

이 체크는
[오픈텔레메트리 관련 용어와 단어가 사이트 전체에서 일관되게 사용되는지](../style-guide/#opentelemetryio-word-list)
검증한다.

문제가 발견되면 PR의 `files changed` 화면에서 해당 파일에 주석이 추가된다.
체크를 통과시키려면 이를 고친다. 대안으로, 로컬에서
`npm run check:text -- --fix`를 실행해 대부분의 문제를 고칠 수 있다.
`npm run check:text`를 다시 실행하고 남은 문제를 수동으로 고친다.

### `MARKDOWN linter` {.notranslate lang=en}

이 체크는
[마크다운 파일의 표준과 일관성이 지켜지는지](../style-guide/#markdown-standards)
검증한다.

문제가 발견되면 `npm run fix:markdown`을 실행해 대부분의 문제를 고친다. 남은
문제가 있다면 `npm run check:markdown`을 실행하고 제안된 변경 사항을 수동으로
적용한다.

### `SPELLING check` {.notranslate lang=en}

이 체크는 모든 로케일에서 [모든 단어의 철자가 올바른지][spell checking]
검증한다.

체크가 실패하면 로컬에서 `npm run check:spelling`을 실행해 문제를 나열한다.
허용된 단어를 추가하거나 변경하려면 스타일 가이드의 [철자
검사][Spell checking]를 참고한다.

[Spell checking]: ../style-guide/#spell-checking

### `CSPELL` check {.notranslate lang=en}

이 체크는 프론트매터(front matter)의 cSpell `cSpell:ignore` 목록이 정규화되어
있는지와 `.cspell/*.txt` 단어 목록이 정렬되어 있는지 검증한다
(`npm run fix:dict` 참고).

이 체크가 실패하면 로컬에서 `npm run fix:dict`를 실행하고 변경 사항을 새
커밋으로 push한다.

### `FILE FORMAT` {.notranslate lang=en}

이 체크는 모든 파일이 [Prettier 형식 규칙](../style-guide/#file-format)을
준수하는지 검증한다.

이 체크가 실패하면 로컬에서 `npm run fix:format`을 실행하고 변경 사항을 새
커밋으로 push한다.

### `FILENAME check` {.notranslate lang=en}

이 체크는 다음을 검증한다.

- 모든 [파일 이름이 kebab-case인지](../style-guide/#file-names)
- 저장소에 오래된 파일이나 폴더가 존재하지 않는지(아래 목록 참고)

각 오류 주석의 안내를 따른다. 로컬에서 고치려면 `npm run fix:filenames`를
실행하고 변경 사항을 새 커밋으로 push한다.

> [!NOTE]
>
> `fix:filenames`는 오래된 파일이나 폴더를 **삭제**할 수도 있다.

#### 오래된 파일과 폴더 {#obsolete-files-and-folders}

이 체크는 다음과 같은 오래된 경로를 표시한다.

- `tools/` - [code-excerpts 도구가 npm 패키지로 이전되면서][#9638] 제거됨
- `static/refcache.json` - [Lychee로 전환되면서][#10911] 제거됨. 자신의 브랜치가
  이 파일을 되살린다면, [오래된 브랜치 업데이트 안내][#10990]를 따른다.

[#9638]: https://github.com/open-telemetry/opentelemetry.io/pull/9638
[#10911]: https://github.com/open-telemetry/opentelemetry.io/pull/10911
[#10990]: https://github.com/open-telemetry/opentelemetry.io/issues/10990

### `BUILD` and `CHECK LINKS` {.notranslate lang=en}

이 두 체크는 웹사이트를 빌드하고 모든 링크가 유효한지 검증한다.

> [!NOTE]
>
> 사이트 로컬 링크에 대한 경고와 관련한 정보는
> [사이트 로컬 링크에는 항상 경로를 사용한다](#avoid-external-site-local-links)를
> 참고한다.

#### 404 고치기 {#fix-404s}

링크 체커가 **유효하지 않다고**(HTTP 상태 **404**) 보고한 URL을 고쳐야 한다.

#### 유효한 외부 링크 처리하기 {#handling-valid-external-links}

링크 체커를 차단하는 서버 때문에 링크 체커가 200(성공) 외의 HTTP 상태를 받을
때가 있다. 이런 서버는 401, 403, 406처럼 404가 아닌 400번대 HTTP 상태를 반환하는
경우가 많으며, 이 세 가지가 가장 흔하다. LinkedIn과 같은 일부 서버는 999를
반환한다.

체커가 성공 상태를 받지 못하는 외부 링크를 수동으로 검증했다면, 자신의 URL에
다음 쿼리 파라미터를 추가해 링크 체커가 이를 무시하게 할 수 있다. 다른 쿼리
파라미터가 있다면 `?link-check=no` 또는 `&link-check=no`를 사용한다. 예를 들어
다음 URL은 무시된다.

- `https:/some-example.org?link-check=no`
- `https:/some-example.org?other-param=value&link-check=no`

`link-check=no`를 추가할 때는 `last-validated=YYYY-MM-DD` 파라미터도 함께 추가해
수동 검증 날짜를 기록한다. 예를 들면 다음과 같다.

- `https:/some-example.org?link-check=no&last-validated=2026-08-02`

### `CACHE updates committed?` {#cache-updates-committed .notranslate lang=en}

외부 링크를 추가하거나 변경하면 링크 체커가 이를 링크 캐시 (`.lycheecache`)에
기록하며, 업데이트된 캐시가 커밋될 때까지 이 체크는 실패한다.

가장 쉬운 업데이트 방법은 PR에
[`/fix:link-cache`](../pull-requests/#fixing-prs-in-github)라고 댓글을 다는
것이다. 그러면 OpenTelemetry bot이 대신 캐시를 업데이트해준다.

또는 로컬에서 `npm run check:links`를 실행해 빌드하고 링크를 체크할 수 있다. 이
명령어는 링크 캐시도 업데이트한다. 캐시에 대한 변경 사항은 새 커밋으로 push한다.

### `WARNINGS in build log?` {.notranslate lang=en}

이 체크가 실패하면 `npm run log:build` 단계 아래에 있는 `BUILD` 로그를 살펴보고
그 밖에 잠재적인 문제가 없는지 확인한다. 복구 방법을 잘 모르겠다면 관리자에게
도움을 요청한다.

#### 사이트 로컬 링크에는 항상 경로를 사용한다 {#avoid-external-site-local-links}

오픈텔레메트리 웹사이트 내의 페이지로 링크할 때는 외부 링크 대신 로컬 경로를
사용한다. 그렇게 하지 않으면 빌드에서 경고가 발생한다.

빌드 경고를 해결하려면 전체 URL에서 경로 부분만 남긴다.

| ❌ 사용하지 않는다                        | ✅ 대신 사용한다  |
| ----------------------------------------- | ----------------- |
| `https://opentelemetry.io/docs/concepts/` | `/docs/concepts/` |
| `https://www.opentelemetry.io/blog/...`   | `/blog/...`       |

로컬 경로를 사용하면 다음이 보장된다.

- 사이트 로컬 페이지가 같은 브라우저 탭에서 열린다. 외부 링크는 새 탭에서
  열리는데, 이는 사이트 로컬 내비게이션에서 원하는 동작이 아니다.
- 로컬라이제이션 링크 처리가 예상대로 작동한다. 링크에 적절한 언어 코드가
  자동으로 앞에 붙는다.
- 로컬 경로는 링크 체크가 더 쉽고 링크 캐시를 불필요하게 채우지 않는다.

<details>
<summary>관리자를 위한 참고 사항</summary>

다음 코드는 이 섹션에서 설명한 링크 요구 사항을 강제한다.

- 이 경고를 내보내는 render-link 훅:
  [`layouts/_markup/render-link.html`](https://github.com/open-telemetry/opentelemetry.io/blob/main/layouts/_markup/render-link.html)
- 전체 URL을 로컬 경로로 자동 변환하는 스크립트:
  [`scripts/content-modules/adjust-pages/`](https://github.com/open-telemetry/opentelemetry.io/tree/main/scripts/content-modules/adjust-pages)

</details>

### `LOCALIZATION` guidelines {.notranslate lang=en #localization}

이 체크는 다른 체크에서 아직 다루지 않는, 기계적으로 검증 가능한
[로컬라이제이션 가이드라인](../localization/)을 강제한다. 예를 들어
[로컬라이제이션 간에 이미지와 그 밖의 애셋을 복사하지 않는지](../localization/#images)
등이다.

이 체크가 실패하면 로컬에서 `npm run fix:l10n`을 실행하고 변경 사항을 새
커밋으로 push한다.

### `TEST (excluding test:base)` {.notranslate lang=en}

`test:*-*` NPM 스크립트로 구성된 컴파운드 스크립트(예: Netlify edge function
테스트)를 실행하는 `npm run test:compound-tests`를 실행한다. `test:base`는
실행하지 **않는다**.
