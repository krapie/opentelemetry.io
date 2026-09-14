---
title: 오래된 블로그 무시 범위 업데이트
description: >-
  사이트 점검과 수정 대상에서 제외되는 오래된 블로그 게시물의 연도 범위를
  갱신하는 방법이다.
cSpell:ignore: textlintignore
default_lang_commit: 89fe70663d19f375b1566b3688ef25257233cafc
---

오래된 블로그 게시물은 [역사적 기록이며 업데이트되지 않으므로][old-blogs],
린트/포맷 점검과 수정 스크립트 대상에서 제외된다. 1년에 한 번(또는 필요할 때)
아래 나열된 각 도구의 구성에서 연도 범위를 갱신한다.

[old-blogs]: /docs/contributing/blog/#old-blogs-are-not-updated

## 갱신할 구성 {#configuration}

각 항목은 동일한 정책을 인코딩한다: 현재 2019와 `202[0-4]`를 무시한다. 링크
검사기는 해당 연도에 게시물이 없으므로 2020년을 생략한다(glob 기반 도구는 이를
포함해도 무해하다). 필요에 따라 각 도구의 연도 무시 glob/패턴을 조정한다:

| 도구                                          | 구성                                                    |
| --------------------------------------------- | ------------------------------------------------------- |
| cspell                                        | `.cspell.yml` → `ignorePaths`                           |
| markdownlint                                  | `.markdownlint-cli2.yaml` → `ignores`                   |
| prettier                                      | `.prettierignore`                                       |
| textlint                                      | `.textlintignore`(참고: `**` glob 접미사가 필요하다)    |
| `fix:dict`, 트레일링 스페이스(trailing space) | `package.json` → `__find:md:not-old-blog` 스크립트      |
| 링크 검사기(Lychee)                           | `content/en/blog/_index.md` → `link_check_exclude_path` |

## 검증 {#verify}

이 정책은 `scripts/old-blog-lint-ignores.test.mjs`가 지켜준다. 이 테스트는
오래된 블로그 폴더와 최근 블로그 폴더에 위반 사항을 심어두고, 각 도구가 전자는
건너뛰고 후자는 표시하는지 어서션한다. 다음으로 실행한다:

```sh
npm run test:local-tools
```

새로 무시 대상이 된 연도에 여전히 도구가 건너뛰는 린트 부채(lint debt)가 남아
있다면, 이는 예상된 것이다 — 오래된 게시물은 그대로 둔다.
