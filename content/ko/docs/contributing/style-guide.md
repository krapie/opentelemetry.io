---
title: 문서 스타일 가이드
description: 오픈텔레메트리 문서를 작성할 때의 용어와 스타일.
linkTitle: 스타일 가이드
weight: 20
params:
  alertExamples: |
    > [!TIP]
    >
    > If you are writing new content, generally prefer using this blockquote alert
    > syntax instead of the Docsy
    > [alert shortcode](https://www.docsy.dev/docs/content/shortcodes/#alert).

    > [!WARNING] :warning: Blank line required!
    >
    > This site uses the [Prettier] formatter, and it requires an empty line
    > separating the alert tag/title from the alert body.
cSpell:ignore: postgre
default_lang_commit: 4aa05a4d65591e780952439f45415c597f3047dd
---

아직 공식적인 스타일 가이드는 없지만, 현재 오픈텔레메트리 문서 스타일은 다음
스타일 가이드에서 영감을 받았다.

- [Google Developer Documentation Style Guide](https://developers.google.com/style)
- [Kubernetes Style Guide](https://kubernetes.io/docs/contribute/style/style-guide/)

다음 섹션에서는 오픈텔레메트리 프로젝트에 특화된 가이드를 다룬다.

> [!NOTE]
>
> 스타일 가이드의 많은 요구 사항은 자동화 실행으로 강제할 수 있다. [풀
> 리퀘스트][pull request](PR)를 제출하기 전에, 로컬 컴퓨터에서
> `npm run fix:all`을 실행하고 변경 사항을 커밋한다.
>
> 오류가 발생하거나 [PR 체크가 실패한다면](../pr-checks), 스타일 가이드를 읽고
> 특정 일반적인 문제를 고칠 방법을 알아본다.

[pull request]:
  https://docs.github.com/en/get-started/learning-about-github/github-glossary#pull-request

## OpenTelemetry.io 단어 목록 {#opentelemetryio-word-list}

사이트 전체에서 일관되게 사용해야 할 오픈텔레메트리 특화 용어와 단어 목록:

- [오픈텔레메트리(OpenTelemetry)](/docs/concepts/glossary/#opentelemetry)와
  [OTel](/docs/concepts/glossary/#otel)
- [컬렉터(Collector)](/docs/concepts/glossary/#collector)
- [OTEP](/docs/concepts/glossary/#otep)
- [OpAMP](/docs/concepts/glossary/#opamp)

오픈텔레메트리 용어와 그 정의의 전체 목록은 [용어집](/docs/concepts/glossary/)을
참고한다.

다른 CNCF 프로젝트나 서드파티 도구와 같은 고유 명사는 원래의 대소문자 표기법을
사용해 올바르게 표기해야 한다. 예를 들어 "postgre"가 아니라 "PostgreSQL"이라고
쓴다. 전체 목록은
[`.textlintrc.yml`](https://github.com/open-telemetry/opentelemetry.io/blob/main/.textlintrc.yml)
파일에서 확인한다.

## 마크다운 {#markdown}

사이트 페이지는 [Goldmark][] 마크다운 렌더러가 지원하는 마크다운 구문으로
작성된다. 지원되는 마크다운 확장 기능의 전체 목록은 [Goldmark][]를 참고한다.

다음 마크다운 확장 기능도 사용할 수 있다.

- [알림(Alert)](#alerts)
- [이모지][Emojis]: 사용 가능한 전체 이모지 목록은 Hugo 문서의
  [이모지][Emojis]를 참고한다.

[Emojis]: https://gohugo.io/quick-reference/emojis/

### 알림 {#alerts}

다음 확장 구문을 사용해 알림을 작성할 수 있다.

- [GitHub-flavored Markdown][GFM] (GFM)의 [알림][gfm-alerts]
- 사용자 지정 알림 제목을 위한 [Obsidian callout][] 구문

각각의 예시는 다음과 같다.

```markdown
{{% _param alertExamples %}}
```

이는 다음과 같이 렌더링된다.

{{% _param alertExamples %}}

블록쿼트 알림 구문에 대한 세부 사항은 Docsy 문서의 [알림][docsy-alerts]을
참고한다.

[gfm-alerts]:
  https://docs.github.com/en/contributing/style-guide-and-content-model/style-guide#alerts
[GFM]: https://github.github.com/gfm/
[Goldmark]: https://gohugo.io/configuration/markup/#goldmark
[docsy-alerts]: https://www.docsy.dev/docs/content/adding-content/#alerts
[Obsidian callout]: https://help.obsidian.md/callouts

### 링크 참조 {#link-references}

마크다운 [참조 링크][reference links]를 사용할 때는, _shortcut_ 형식
`[text]`보다 _collapsed_ 형식 `[text][]`를 선호한다. 둘 다 유효한
[CommonMark][]이지만, shortcut 형식은 모든 마크다운 도구에서 일관되게 인식되지는
않는다. 특히 `[example]`이라고 쓰고 정의를 빠뜨리면, [markdownlint][] 린터는
이를 경고하지 않고[^md052] 텍스트가 그대로 링크가 아닌 리터럴 `[example]`로
렌더링된다. collapsed 형식 `[example][]`을 쓰면 린터가 누락된 정의를 즉시
잡아낸다.

[^md052]:
    구체적으로, 내장된 [MD052][] 규칙(`reference-links-images`)은 기본적으로
    collapsed 형식과 full 참조 형식만 체크한다. 이 규칙의 `shortcut_syntax`
    옵션으로 shortcut 참조를 포함시킬 수는 있지만, 실제로는 잘 작동하지 않는다.

[MD052]: https://github.com/DavidAnson/markdownlint/blob/main/doc/md052.md

이는 `no-shortcut-ref-link` 커스텀 규칙으로 강제된다. shortcut 참조를 자동으로
변환하려면 `npm run fix:markdown`을 실행한다.

[CommonMark]: https://spec.commonmark.org/0.31.2/#reference-link
[reference links]: https://spec.commonmark.org/0.31.2/#reference-link

### 마크다운 체크 {#markdown-standards}

마크다운 파일의 표준과 일관성을 강제하기 위해, 모든 파일은 [markdownlint][]로
강제되는 특정 규칙을 따라야 한다. 전체 목록은 [.markdownlint.yaml][]과
[.markdownlint-cli2.yaml][] 파일에서 확인한다.

규칙에 대한 정당한 예외가 있는 경우, `markdownlint-disable` 지시어를 사용해 규칙
경고를 억제한다. 세부 사항은
[markdownlint 문서](https://github.com/DavidAnson/markdownlint#configuration)를
참고한다.

또한 마크다운 [파일 형식](#file-format)을 강제하고 파일 끝의 공백을 제거한다.
이는 [줄바꿈 구문][line break syntax]에서 2개 이상의 공백을 사용하는 방식을
배제한다. 대신 `<br>`을 사용하거나 텍스트를 다시 구성한다.

## 철자 검사 {#spell-checking}

모든 텍스트의 철자가 올바른지 확인하려면
[CSpell](https://github.com/streetsidesoftware/cspell)을 사용한다.

`cspell`이 "Unknown word"를 보고하면, 단어를 올바르게 작성했는지 확인한다.
올바르다면 다음 위치 중 한 곳에 단어를 추가한다.

- 페이지 프론트매터의 페이지 로컬 `cSpell:ignore` 목록. 세부 사항은 아래를
  참고한다.
- 자신의 로케일별 단어 목록 파일
- 일반 [all-words.txt][] 단어 목록

[all-words.txt]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.cspell/all-words.txt

### 페이지 로컬 `cSpell:ignore` 목록 {#page-local-cspellignore-list}

알 수 없는 단어가 한 페이지나 몇몇 페이지에만 나타난다면, 페이지 프론트매터의
페이지 로컬 `cSpell:ignore` 목록에 추가한다.

```markdown
---
title: PageTitle
cSpell:ignore: <word>
---
```

마크다운이 아닌 파일의 경우, 해당 파일에 적절한 주석 줄에
`cSpell:ignore <word>`를 추가한다. 예를 들어 [레지스트리](/ecosystem/registry/)
항목 YAML 파일에서는 다음과 같은 형태가 된다.

```yaml
# cSpell:ignore <word>
title: registryEntryTitle
```

### 단어 목록 파일 {#word-list-files}

알 수 없는 단어가 여러 페이지에 나타나거나 기술 용어라면, 자신의 로케일별 단어
목록 파일에 추가한다. 단어 목록 파일은 [.cspell/][] 디렉터리에 있다.

`opamp`처럼 모든 로케일에서 철자가 올바른 단어라면 [all-words.txt][] 파일에
추가한다.

[.cspell/]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.cspell/

## 파일 형식 {#file-format}

파일 서식을 강제하기 위해 [Prettier][]를 사용한다. 다음 명령어로 실행한다.

- 모든 파일 서식을 지정하려면 `npm run fix:format`
- 마지막 커밋 이후 변경된 파일만 서식을 지정하려면 `npm run fix:format:diff`
- 다음 커밋을 위해 스테이징된 파일만 서식을 지정하려면
  `npm run fix:format:staged`

## 파일 이름 {#file-names}

모든 파일 이름은
[kebab case](https://en.wikipedia.org/wiki/Letter_case#Kebab_case)로 작성해야
한다.

## 검증 문제 고치기 {#fixing-validation-issues}

검증 문제를 고치는 방법을 알아보려면 [풀 리퀘스트 체크](../pr-checks)를
참고한다.

[.markdownlint.yaml]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.markdownlint.yaml
[.markdownlint-cli2.yaml]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.markdownlint-cli2.yaml
[line break syntax]: https://www.markdownguide.org/basic-syntax/#line-breaks
[markdownlint]: https://github.com/DavidAnson/markdownlint
[Prettier]: https://prettier.io
