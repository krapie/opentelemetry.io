---
title: 사이트 로컬라이제이션
description: 영어가 아닌 로컬라이제이션으로 사이트 페이지를 만들고 유지보수하기.
linkTitle: 로컬라이제이션
weight: 25
cSpell:ignore: Dowair shortcodes
default_lang_commit: cd6a7aa0e28ff8eb622e9fa0c9e9a40f78c9c777
---

OTel 웹사이트는 페이지 로컬라이제이션(localization)을 지원하기 위해 Hugo의
[다국어 프레임워크][multilingual framework]를 사용한다. 영어가 기본 언어이며,
미국 영어가 기본(암묵적) 로컬라이제이션이다. 상단 내비게이션의 언어 드롭다운
메뉴에서 볼 수 있듯이, 점점 더 많은 다른 로컬라이제이션이 지원되고 있다.

## 번역 가이드 {#translation-guidance}

영어에서 웹사이트 페이지를 번역할 때는 이 섹션에서 제공하는 가이드를 따르기를
권장한다.

### 요약 {#summary}

#### ✅ 해야 할 일 {#do}

<div class="border-start border-success bg-success-subtle">

- **번역한다**:
  - 다음을 포함한 페이지 콘텐츠:
    - Mermaid [다이어그램](#images) 텍스트 필드
    - 코드 발췌의 코드 주석(선택 사항)
  - `title`, `linkTitle`, `description`에 대한 [프론트매터][Front matter] 필드
    값
  - 별도로 표시하지 않는 한 **모든** 페이지 콘텐츠와 프론트매터
- 원문의 _콘텐츠_, _의미_, _스타일_을 **보존한다**
- [작은 풀 리퀘스트](#small-prs)를 통해 작업물을 **점진적으로** 제출한다
- 의문이나 질문이 있다면 다음을 통해 [관리자][maintainers]에게 **물어본다**:
  - [Slack][]의 `#otel-docs-localization` 또는 `#otel-comms` 채널
  - [디스커션][Discussion], 이슈, 또는 PR 댓글

[Discussion]:
  https://github.com/open-telemetry/opentelemetry.io/discussions?discussions_q=is%3Aopen+label%3Ai18n

</div>

#### ❌ 하지 말아야 할 일 {#do-not}

<div class="border-start border-warning bg-warning-subtle">

- **번역한다**:
  - `TIP`, `WARNING` 등과 같은 [알림 유형](../style-guide/#alerts). 이는
    [`MARKDOWN` 린터][`MARKDOWN` linter] 규칙으로 강제된다.
  - 코드 블록과 인라인 코드(이 `인라인 코드 예시`처럼)를 포함한 코드
  - 이 저장소에 있는 리소스의 **파일 또는 디렉터리** 이름
  - [해야 할 일](#do)에 나열된 것 외의 [프론트매터][Front matter] 필드. 특히
    `aliases`는 번역하지 않는다. 확실하지 않다면 관리자에게 물어본다.
  - [링크](#links). 여기에는 [제목 ID](#headings)도 포함된다[^*]
  - `notranslate`로 표시된 마크다운 요소(보통 CSS 클래스로 표시되며, 특히
    [제목](#headings)에 대해)
- [이미지와 그 밖의 애셋](#images)의 **사본**을 만든다. 단,
  [파일 안의 텍스트를 로컬라이즈하는](#images) 경우는 예외이다.
- 다음을 추가하거나 변경한다:
  - 원래 의도된 의미와 달라지는 **콘텐츠**
  - _서식_, _레이아웃_, (예를 들어 타이포그래피, 대소문자 표기, 간격과 같은)
    디자인 **스타일**을 포함한 표현 **스타일**

[^*]: 예외가 있을 수 있는 경우는 [링크](#links)를 참고한다.

[`MARKDOWN` linter]: ../pr-checks/#markdown-linter

</div>

#### AI 도구 사용 {#ai-tools}

번역을 돕기 위해 생성형 AI 도구(ChatGPT, Gemini 등)를 사용한다면, 오픈텔레메트리
[생성형 AI 기여 정책][genai-policy]과 리눅스 재단의 [생성형 AI
정책][lf-ai-policy]을 따라야 한다. 특히 다음 사항을 지켜야 한다.

- [풀 리퀘스트 템플릿][pull request template]에서 해당 체크박스를 선택해 AI를
  사용했음을 **공개한다**.
- AI가 생성한 모든 번역의 정확성을 **리뷰하고 검증한다**. 자신이 제출한 콘텐츠에
  대한 책임은 자신에게 있다.
- 자신이 직접 리뷰하고 검증할 수 없는 AI 생성 번역은 **제출하지 않는다**(예:
  능숙하지 않은 언어로 된 제출물). 이는 상당한 리뷰 병목을 만들며, 관리자의
  시간을 지키기 위해 자신의 PR이 닫힐 수 있다.

[genai-policy]:
  https://github.com/open-telemetry/community/blob/main/policies/genai.md
[lf-ai-policy]: https://www.linuxfoundation.org/legal/generative-ai
[pull request template]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/PULL_REQUEST_TEMPLATE.md

### 제목 ID {#headings}

로컬라이제이션 전체에서 제목 앵커 대상이 일관되도록, 제목을 번역할 때는 다음과
같이 한다.

- 명시적 ID가 있다면 제목의 명시적 ID를 보존한다. [제목 ID
  구문][Heading ID syntax]은 제목 텍스트 뒤에 `{ #some-id }`와 같은 구문으로
  작성된다.
- 그렇지 않다면, 원본 영어 제목의 자동 생성된 ID에 해당하는 제목 ID를 명시적으로
  선언한다.

[Heading ID syntax]:
  https://github.com/yuin/goldmark/blob/master/README.md#headings

### 링크 {#links}

링크 참조는 번역하지 **않는다**. 이는 외부 링크와 이미지 및 그 밖의
[애셋](#images) 같은 웹사이트 페이지 및 섹션 로컬 리소스로의 경로 모두에
해당한다.

유일한 예외는 자신의 로케일에 특화된 버전이 있는 외부 페이지(예를 들어
<https://en.wikipedia.org>)로의 링크이다. 이는 대개 URL의 `en`을 자신의 로케일
언어 코드로 바꾸는 것을 의미한다.

> [!NOTE]
>
> OTel 웹사이트 저장소에는 절대 링크 경로가 문서 페이지를 가리킬 때 Hugo가 이를
> 변환하는 데 사용하는 커스텀 render-link 훅이 있다. **`/docs/some-page` 형식의
> 링크는** 렌더링될 때 경로 앞에 페이지 언어 코드가 붙어 **로케일에 특화된다**.
> 예를 들어, 앞서 든 경로 예시는 일본어 페이지에서 렌더링될 때
> `/ja/docs/some-page`가 된다.

### 링크 정의 레이블 {#link-labels}

로케일 저자는 마크다운 [링크 정의][link definitions]의 [레이블][labels]을
번역할지 말지 선택할 수 있다. 영어 레이블을 유지하기로 했다면, 이 섹션에서
제공하는 가이드를 따른다.

예를 들어 다음 마크다운을 살펴본다.

```markdown
[Hello], world! Welcome to the [OTel website][].

[hello]: https://code.org/helloworld
[OTel website]: https://opentelemetry.io
```

이는 프랑스어로 다음과 같이 번역된다.

```markdown
[Bonjour][hello], le monde! Bienvenue sur le [site OTel][OTel website].

[hello]: https://code.org/helloworld
[OTel website]: https://opentelemetry.io
```

[labels]: https://spec.commonmark.org/0.31.2/#link-label
[link definitions]:
  https://spec.commonmark.org/0.31.2/#link-reference-definitions

### 이미지와 그 밖의 애셋 {#images}

- 파일 자체의 텍스트를 로컬라이즈하는 경우가 아니라면, 이미지, 동영상, 그 밖의
  콘텐츠가 아닌 애셋 파일의 사본을 만들지 **않는다**.
  - Hugo는 사이트 로컬라이제이션 간에 공유되는 이미지 파일을 렌더링하는 방식이
    영리하다. 즉 Hugo는 _단일_ 이미지 파일을 출력해 모든 로케일에 걸쳐 공유한다.
    자세한 내용은 [페이지 번들][Page bundles]을 참고한다.
  - 이는 [`LOCALIZATION` 가이드라인][l10n-check] 체크로 강제된다.

- [Mermaid][] 다이어그램 안의 텍스트는 번역**한다**.

[l10n-check]: ../pr-checks/#localization
[Mermaid]: https://mermaid.js.org
[Page bundles]: https://gohugo.io/content-management/multilingual/#page-bundles

### 포함 파일 {#includes}

`_includes` 디렉터리 아래에 있는 페이지 조각(fragment)은 다른 어떤 페이지
콘텐츠와 마찬가지로 번역**한다**.

### 숏코드 {#shortcodes}

> [!NOTE]
>
> 2025년 2월 기준으로, 공유 페이지 콘텐츠를 지원하는 수단을 숏코드
> (shortcode)에서 [포함 파일](#includes)로 마이그레이션하는 중이다.

일부 기본 숏코드에는 로컬라이즈해야 할 수도 있는 영어 텍스트가 들어 있다. 특히
[layouts/_shortcodes/docs][]에 있는 숏코드가 그렇다.

숏코드의 로컬라이즈 버전을 만들어야 한다면, `layouts/_shortcodes/xx` 아래에
배치한다. 여기서 `xx`는 자신의 로컬라이제이션 언어 코드이다. 거기에서 원본 기본
숏코드와 같은 상대 경로를 사용한다.

[layouts/_shortcodes/docs]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/layouts/_shortcodes/docs

## 로컬라이즈된 페이지 드리프트 추적하기 {#track-changes}

로컬라이즈된 페이지를 유지보수하는 데 있어 가장 큰 어려움 중 하나는, 대응하는
영어 페이지가 언제 업데이트되었는지 파악하는 것이다. 이 섹션에서는 이를 어떻게
처리하는지 설명한다.

### `default_lang_commit` 프론트매터 필드 {#the-default_lang_commit-front-matter-field}

`content/zh/<some-path>/page.md`와 같은 로컬라이즈된 페이지가 작성될 때, 이
번역은 `content/en/<some-path>/page.md`에 있는 대응하는 영어 버전 페이지의 특정
[`main` 브랜치 커밋][main]을 기준으로 한다. 이 저장소에서는 모든 로컬라이즈된
페이지가 다음과 같이 로컬라이즈된 페이지의 프론트매터에 영어 페이지 커밋을
명시한다.

```markdown
---
title: Your localized page title
# ...
default_lang_commit: <most-recent-commit-hash-of-default-language-page>
---
```

위 프론트매터는 `content/zh/<some-path>/page.md`에 있게 된다. 커밋 해시는 `main`
브랜치에서 `content/en/<some-path>/page.md`의 최신 커밋에 대응한다.

### 영어 페이지 변경 사항 추적하기 {#tracking-changes-to-english-pages}

영어 페이지가 업데이트되면, 다음 명령어를 실행해 업데이트가 필요한 대응
로컬라이즈된 페이지를 추적할 수 있다.

```console
$ npm run check:i18n
> Drifted file: content/zh/docs/platforms/kubernetes/_index.md
...
DRIFTED files: 361 out of 990
```

다음과 같이 경로를 제공해 대상 페이지를 하나 이상의 로컬라이제이션으로 제한할 수
있다.

```sh
npm run check:i18n -- content/zh
```

### 변경 사항 세부 정보 확인하기 {#viewing-change-details}

업데이트가 필요한 로컬라이즈된 페이지에 대해, 로컬라이즈된 페이지의 경로를
제공하고 `diff` 서브 명령어를 사용해 대응하는 영어 페이지의 diff 세부 정보를
확인할 수 있다. 예를 들면 다음과 같다.

```console
$ npm run check:i18n -- diff content/zh/docs/platforms/kubernetes
# content/zh/docs/platforms/kubernetes/_index.md: drifted from 1ca30b4d
diff --git a/content/en/docs/platforms/kubernetes/_index.md b/content/en/docs/platforms/kubernetes/_index.md
index 3592df5d..c7980653 100644
--- a/content/en/docs/platforms/kubernetes/_index.md
+++ b/content/en/docs/platforms/kubernetes/_index.md
@@ -1,7 +1,7 @@
 ---
 title: OpenTelemetry with Kubernetes
 linkTitle: Kubernetes
-weight: 11
+weight: 350
 description: Using OpenTelemetry with Kubernetes
 ---
```

### 새 페이지에 `default_lang_commit` 추가하기 {#adding-default_lang_commit-to-new-pages}

자신의 로컬라이제이션을 위한 페이지를 만들 때는, `main`의 적절한 커밋 해시와
함께 페이지 프론트매터에 `default_lang_commit`을 추가하는 것을 잊지 않는다.

자신의 페이지 번역이 `<HASH>`에 있는 `main`의 영어 페이지를 기준으로 한다면,
다음 명령어를 실행해 커밋 `<HASH>`를 사용하는 `default_lang_commit`을 자신의
페이지 파일 프론트매터에 자동으로 추가한다. 자신의 페이지가 이제 `HEAD`에 있는
`main`과 동기화되었다면 인자로 `HEAD`를 지정할 수 있다. 예를 들면 다음과 같다.

```sh
npm run check:i18n -- commit 1ca30b4d --new content/ja
npm run check:i18n -- commit HEAD --new content/zh/docs/concepts
```

해시 키가 없는 로컬라이제이션 페이지 파일을 나열하려면 다음을 실행한다.

```sh
npm run check:i18n -- --new
```

### 기존 페이지의 `default_lang_commit` 업데이트하기 {#updating-default_lang_commit-for-existing-pages}

대응하는 영어 페이지에 이루어진 변경 사항에 맞춰 로컬라이즈된 페이지를
업데이트할 때는, `default_lang_commit` 커밋 해시도 함께 업데이트했는지 확인한다.

> [!TIP]
>
> 자신의 로컬라이즈된 페이지가 이제 `main`의 `HEAD`에 있는 영어 버전과
> 대응한다면, `npm run check:i18n -- commit HEAD <PATH-TO-YOUR-PAGE>`를
> 실행한다. `default_lang_commit` 해시가 새로고침되고, 같은 작업으로 페이지의
> [드리프트 상태](#drift-status)도 지워진다.

드리프트된 모든 로컬라이제이션 페이지를 일괄 업데이트했다면, `commit` 서브
명령어 뒤에 커밋 해시나 `main@HEAD`를 사용하기 위한 'HEAD'를 붙여 이 파일들의
커밋 해시를 업데이트할 수 있다.

```sh
npm run check:i18n -- commit <HASH> <PATH-TO-YOUR-UPDATED-FILES>
npm run check:i18n -- commit HEAD <PATH-TO-YOUR-UPDATED-FILES>
```

> [!IMPORTANT]
>
> 해시 지정자로 `HEAD`를 사용하면, 스크립트는 **자신의 로컬 환경**에 있는
> `main`의 HEAD 해시를 사용한다. HEAD가 GitHub의 `main`과 대응하도록 하려면
> `main`을 fetch하고 pull했는지 확인한다.

### 로컬라이즈된 페이지 패치하기 {#patched}

[빌드 수정](#keep-checks-green)은 때때로 로컬라이즈된 페이지를 대응하는 영어
페이지와 동기화하지 않고 편집해야 할 것을 요구한다. 예를 들어 공유 숏코드가
변경된 후 숏코드 호출을 복구하는 경우이다. 이렇게 고쳐진 각 로컬라이즈된
페이지는, 그 수정이 여러 로케일에 걸쳐 있는지 여부와 상관없이 **patched**로
표시한다.

- 수정에 필요한 편집만 한다. 페이지에 다른 변경 사항은 만들지 않는다.
- 페이지의 `default_lang_commit` 줄에 `# patched` YAML 주석을 추가한다.

  ```yaml
  default_lang_commit: abc4567... # patched
  ```

이 마커는 이런 기계적인 수정을 위해 예약되어 있다.
[의미적 변경 사항](#semantic-changes)에는 절대 이 마커를 사용하지 않는다. 이
마커는 페이지가 동기화되지 않은 채 수정되었음을 페이지의 로케일 팀에게 알려준다.
해시는 여전히 마지막 동기화 지점을 기록한다. 이 마커는 다음에 페이지의 해시가
[업데이트](#updating-default_lang_commit-for-existing-pages)될 때 제거된다.

### 드리프트 상태 {#drift-status}

`drifted_from_default` 프론트매터 필드는 로컬라이즈된 페이지를 드리프트된 것으로
표시한다. 이 페이지에는 "오래됨(outdated)" 배너가 표시되고, 링크 체커가 이를
건너뛰어 드리프트된 페이지의 오래된 링크가 CI를 실패시키지 않도록 한다. 링크
체커는 이 필드를 기다리지 않는다. 트리 전체 상태 동기화 이후 변경된 영어
페이지의 로케일 사본도 마찬가지로
[드리프트 대기](/site/build/link-checking/#configuration) 상태로 건너뛴다.

매일 실행되는 [하우스키핑 실행](/site/build/ci-workflows/#housekeeping)은 트리
전체에서 이 필드를 동기화된 상태로 유지한다. PR은 자신이 달리 변경하지 않는
페이지의 상태를 업데이트하지 않는다. PR이 변경하는 각 페이지는 `I18N check`이
강제하는 대로 정확한 드리프트 상태를 유지한 채로 PR을 벗어나야 한다. 즉,
페이지를 대응하는 영어 페이지와 동기화하고
[핀을 새로고침하거나](#updating-default_lang_commit-for-existing-pages) (같은
작업으로 상태가 지워진다), 아니면 `npm run fix:i18n:status -- <PATHS>`로 남은
드리프트를 기록해야 한다. 핀은 `main`의 커밋만 가리킬 수 있으므로, 같은 PR에서
만들어진 영어 변경 사항에 동기화된 페이지는 그 변경 사항이 머지될 때까지 남은
드리프트를 기록한다.

### 스크립트 도움말 {#script-help}

스크립트에 대한 더 자세한 내용은 `npm run check:i18n -- -h`를 실행한다.

## 새 로컬라이제이션 {#new-localizations}

OTel 웹사이트를 위한 새 로컬라이제이션을 시작하는 데 관심이 있는가? GitHub
디스커션이나 Slack `#otel-docs-localization` 채널 등을 통해 관리자에게 관심을
표현한다. 이 섹션에서는 새 로컬라이제이션을 시작하는 단계를 설명한다.

> [!NOTE]
>
> 새 로컬라이제이션을 시작하기 위해 오픈텔레메트리 프로젝트의 기존 기여자일
> 필요는 없다. 다만 [멤버십 가이드라인][membership guidelines]에 명시된 정식
> 멤버 및 승인자가 되기 위한 요구 사항을 충족하기 전까지는
> [오픈텔레메트리 GitHub 조직](https://github.com/open-telemetry/)의 멤버로,
> 또는 자신의 로컬라이제이션의 승인자 그룹의 멤버로 추가될 수 없다.
>
> 승인자 상태를 얻기 전에도, "LGTM"(Looks Good To Me) 댓글을 추가해
> 로컬라이제이션 PR에 대한 자신의 승인을 표시할 수 있다. 이 시작 단계 동안에는
> 관리자가 자신을 이미 승인자인 것처럼 대우해 리뷰를 처리한다.

[membership guidelines]:
  https://github.com/open-telemetry/community/blob/main/guides/contributor/membership.md

### 1. 로컬라이제이션 팀 꾸리기 {#team}

로컬라이제이션을 만드는 것은 활발하고 서로를 지지하는 커뮤니티를 키우는 일이다.
오픈텔레메트리 웹사이트의 새 로컬라이제이션을 시작하려면 다음이 필요하다.

1. [CNCF 용어집][CNCF Glossary]이나 [쿠버네티스 웹사이트][Kubernetes website]의
   [활동 중인 승인자][active approver]와 같이, 자신의 언어에 익숙한
   **로컬라이제이션 멘토**.
2. 최소 두 명의 잠재적 기여자.

[active approver]: https://github.com/cncf/glossary/blob/main/CODEOWNERS
[CNCF Glossary]: https://glossary.cncf.io/
[Kubernetes website]: https://github.com/kubernetes/website

### 2. 로컬라이제이션 킥오프: 이슈 생성하기 {#kickoff}

[로컬라이제이션 팀](#team)이 갖춰졌거나 갖춰지고 있다면, 아래에 주어진 작업
목록으로 이슈를 생성한다.

1. 추가하려는 언어의 공식 [ISO 639-1 코드][ISO 639-1 code]를 찾는다. 이 섹션의
   나머지 부분에서는 이 언어 코드를 `LANG_ID`라고 부른다. 특히 하위 지역을
   선택할 때 어떤 태그를 사용할지 확실하지 않다면 관리자에게 물어본다.

   [ISO 639-1 code]: https://en.wikipedia.org/wiki/ISO_639-1

2. [멘토와 잠재적 기여자](#team)의 GitHub 핸들을 파악한다.

3. 시작 댓글에 다음 작업 목록을 담아 [새 이슈][new issue]를 생성한다.

   ```markdown
   - [ ] Language info:
     - ISO 639-1 language code: `LANG_ID`
     - Language name: ADD_NAME_HERE
   - [ ] Locale team info:
     - [ ] Locale mentor: @GITHUB_HANDLE1, @GITHUB_HANDLE2, ...
     - [ ] Contributors: @GITHUB_HANDLE1, @GITHUB_HANDLE2, ...
   - [ ] Read through
         [Localization](https://opentelemetry.io/docs/contributing/localization/)
         and all other pages in the Contributing section
   - [ ] Localize site homepage (only) to YOUR_LANGUAGE_NAME and submit a PR.
         For details, see
         [Localize the homepage](https://opentelemetry.io/docs/contributing/localization/#homepage).
   - [ ] OTel maintainers:
     - [ ] Update Hugo config for `LANG_ID`
     - [ ] Configure cSpell and other tooling support
     - [ ] Create an issue label for `lang:LANG_ID`
     - [ ] Create org-level group for `LANG_ID` approvers
     - [ ] Update components owners for `content/LANG_ID`
   - [ ] Create an issue to track the localization of the **glossary**. Add the
         issue number here. For details, see
         [Localize the glossary](https://opentelemetry.io/docs/contributing/localization/#glossary).
   ```

### 3. 홈페이지 로컬라이즈하기 {#homepage}

[홈페이지][homepage]의 번역을 담아, `content/LANG_ID/_index.md` 파일에 _그것만_
담은 [풀 리퀘스트를 제출한다](../pull-requests/). 관리자가 로컬라이제이션
프로젝트를 시작하는 데 필요한 추가 변경 사항을 자신의 PR에 더하게 되므로,
관리자가 자신의 PR을 편집할 수 있는 권한이 있는지 확인한다.

[homepage]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/content/en/_index.md

첫 번째 PR이 머지된 후, 관리자가 이슈 레이블, 조직 수준 그룹, 컴포넌트 소유자를
설정한다.

### 4. 용어집 로컬라이즈하기 {#glossary}

두 번째로 로컬라이즈할 페이지는 [용어집](/docs/concepts/glossary/)이다. 이는
옵저버빌리티와 특히 오픈텔레메트리에서 사용되는 핵심 용어를 정의하므로,
로컬라이즈된 독자에게 **매우 중요한** 페이지이다. 자신의 언어에 그런 용어가
존재하지 않는다면 특히 더 중요하다.

가이드가 필요하다면 Write the Docs 2024에서 진행된 Ali Dowair의 발표
[영상][ali-d-youtube]을 참고한다: [The art of translation: How to localize
technical content][ali-dowair-2024].

[ali-dowair-2024]:
  https://www.writethedocs.org/conf/atlantic/2024/speakers/#speaker-ali-dowair-what-s-in-a-word-lessons-from-localizing-kubernetes-documentation-to-arabic-ali-dowair
[ali-d-youtube]: https://youtu.be/HY3LZOQqdig

### 5. 나머지 사이트 페이지를 작은 단위로 로컬라이즈하기 {#rest}

용어가 확립되었다면, 이제 나머지 사이트 페이지를 로컬라이즈할 수 있다.

> [!IMPORTANT] 작은 PR 제출하기 <a id="small-prs"></a>
>
> 로컬라이제이션 팀은 **작은 단위**로 작업물을 제출해야 한다. 즉 [PR][PRs]을
> 작게, 가급적 하나 또는 몇 개의 작은 파일로 제한한다. 더 작은 PR은 리뷰하기
> 쉬워서 대개 더 빨리 머지된다.

### OTel 관리자 체크리스트 {#otel-maintainer-checklist}

#### Hugo {#hugo}

`LANG_ID`에 대한 Hugo 설정을 업데이트한다. 다음 아래에 `LANG_ID`에 대한 적절한
항목을 추가한다.

- `config/_default/hugo.yaml`의 `languages`
- `config/_default/module-template.yaml`을 통한 `module.mounts`. 최소한
  `content`에 대해 하나의 `source`-`target` 항목을 추가한다. 로케일에 충분한
  콘텐츠가 쌓인 후에만 `en` 폴백 페이지에 대한 항목 추가를 고려한다.

#### 철자 {#spelling}

[cSpell 사전][cSpell dictionaries]이 [@cspell/dict-LANG_ID][]라는 NPM 패키지로
제공되는지 찾아본다. 자신의 방언이나 지역에 대한 사전이 없다면 가장 가까운
지역을 선택한다.

- **사전이 있는 경우**:
  - 개발 의존성으로 NPM 패키지를 추가한다. 예를 들면
    `npm install --save-dev @cspell/dict-bn`.
  - [`.cspell.yml`][]의 `import:` 아래에 패키지의 `cspell-ext.json`을 추가하고,
    `dictionaries:` 아래에 사전의 ID(예를 들어 `bn`, `es-es`, `pl_pl`)를
    추가한다.
- 해당 언어에 대한 **사전이 없는 경우**, 이에 대한 `import`를 추가하지 않는다.
  cSpell이 해당 로케일의 마크다운을 영어로 취급해 철자를 체크하지 않도록,
  [`.cspell.yml`][]의 `ignorePaths` 목록에 `content/LANG_ID`를 추가한다.

[cSpell dictionaries]: https://github.com/streetsidesoftware/cspell-dicts
[@cspell/dict-LANG_ID]: https://www.npmjs.com/search?q=%40cspell%2Fdict
[`.cspell.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.cspell.yml

#### 단어 목록 {#word-list}

**철자** 항목에 추가할 자연어 사전이 없는 경우에도, 새 로케일마다
`.cspell/LANG_ID-words.txt`를 만든다(처음에는 비어 있음).

- [`.cspell.yml`][]에서 파일을 등록하고 활성화한다.
  - `dictionaryDefinitions` 아래에 `name`(예를 들어 `LANG_ID-words`)과
    `path`(예를 들어 `.cspell/LANG_ID-words.txt`)를 가진 항목을 추가한다.
  - `dictionaries` 아래에, 위 단계와 같은 `name` 값(파일 경로가 아니라)을
    추가한다.

#### 그 밖의 도구 지원 {#other-tooling-support}

- Prettier 지원: `LANG_ID`가 Prettier에서 잘 지원되지 않는다면,
  `.prettierignore`에 무시 규칙을 추가한다.

## 승인자와 관리자를 위한 가이드 {#approver-and-maintainer-guidance}

### 로케일 전용 PR에서 auto-merge 활성화하기 {#auto-merge}

로케일 관리자 팀의 멤버는 `/auto-merge`(또는 `/auto-merge:enable`) 댓글을 달아
로케일 전용 PR에서 [GitHub auto-merge][]를 활성화할 수 있다. 끄려면
`/auto-merge:disable`을 사용한다. 이 지시어는 댓글의 첫 번째 또는 마지막 비어
있지 않은 줄로, 앞에 다른 텍스트나 공백 없이 자신만의 줄에 있어야 한다. 최대 한
번만 나타날 수 있다. 예를 들어 다음과 같이 쓸 수 있다.

```text
LGTM
/auto-merge
```

이를 통해 확립된 로컬라이제이션 팀은 문서 관리자를 기다리지 않고 자신의 PR을
머지할 수 있다. GitHub, 브랜치 보호, CODEOWNERS 규칙은 여전히 머지를 통제한다.
필요한 모든 리뷰가 이루어지고 체크를 통과해야만 PR이 머지된다.

auto-merge 댓글은 변경된 모든 파일이 자신이 관리하는 로케일 소유일 때만
적용되므로, 공유 콘텐츠나 영어 콘텐츠를 변경하는 데는 사용할 수 없다. 적격
요건과 명령어 세부 사항은 [헬퍼 README][helper README]를 참고한다.

[GitHub auto-merge]:
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request
[helper README]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/scripts/gh/locale-auto-merge

### PR은 로케일에 걸쳐 있어서는 안 된다 {#prs-should-not-span-locales}

일반적인 규칙으로, PR은 로케일에 걸쳐 있어서는 안 된다. 즉 최대 하나의 로케일
페이지만 변경해야 한다. 유일한 예외는 이 섹션에서 설명한다.

#### 의미적 변경 사항 {#semantic-changes}

승인자는 문서 페이지에 **의미적인** 변경을 가하는 [PR][PRs]이 여러 로케일에
걸치지 않도록 해야 한다. 의미적 변경이란 페이지 콘텐츠의 _의미_ — 독자가
이해하고 그에 따라 행동하는 내용 — 에 영향을 미치는 변경이다. 코드 블록, 명령어,
설정 예시도 이 콘텐츠의 일부이다. 이들은 [번역되지](#do-not) 않지만, 이들에 대한
편집도 마찬가지로 의미적 변경이다. 우리 문서의 [로컬라이제이션 프로세스](.)는
로케일 승인자가 때가 되면 영어 편집 내용을 리뷰해 그 변경이 자신의 로케일에
적절한지, 그리고 자신의 로케일에 어떻게 가장 잘 반영할지 결정하도록 보장한다.
변경이 필요하다면, 로케일 승인자가 자신의 로케일별 PR을 통해 직접 그렇게 한다.

> [!NOTE] 콘텐츠와 무관한 유지보수
>
> 로케일 범위 규칙은 페이지 **콘텐츠**를 다스린다. 관리자는 때때로 사이트 전체
> 도구, 설정, 프론트매터, 마크업 업데이트를 포함해
> [드리프트 상태](#drift-status) 정리(자동화된 PR과 수동 상태 전용 편집
> 모두)처럼, 필연적으로 여러 로케일에 걸치는 콘텐츠와 무관한 변경 사항을
> 제출한다. 이런 변경은 로컬라이즈된 페이지의 의미를 바꾸지 않는다.

#### 빌드를 그린 상태로 유지하기 {#keep-checks-green}

로컬라이즈된 페이지 **콘텐츠**를 변경하는 PR은 사이트 빌드를 그린 상태로
유지하는 데 반드시 필요한 경우에만 여러 로케일에 걸칠 수 있다. 이런 **빌드
수정**은, 예를 들어 공유 숏코드, 포함 파일, 데이터 소스가 변경된 후에
로컬라이즈된 페이지에서 발생하는 사이트 빌드 손상을 복구한다. 페이지의
[드리프트 상태](#drift-status)는 링크 체크로부터만 페이지를 보호하며, Hugo
빌드로부터는 보호하지 않는다. 이렇게 고친 모든 로컬라이즈된 페이지는
[patched](#patched)로 표시한다.

링크 체크 실패는 이런 경우에 **해당하지 않는다**. 이를 해결하는 방법은
[링크 수정과 리소스 업데이트](#link-fixes-and-resource-updates)를 참고한다.

드리프트 상태 정리에도 같은 최소 수정 규칙이 적용된다. 실패한 체크가
`drifted_from_default`를 새로고침하라고 요구할 때는, 해당 실패한 체크가 보고하는
페이지만 업데이트한다. 매일 실행되는
[하우스키핑 실행](/site/build/ci-workflows/#housekeeping)이 나머지를 완료한다.
상태 전용 편집은 [콘텐츠와 무관한 유지보수](#semantic-changes)이다.

로컬라이즈된 페이지 콘텐츠에 대한 그 밖의 변경은 해당 로케일에 대한 **의미적**
변경으로 취급한다. 여기에는 새 용어집 항목 추가와 같이, 드리프트된 페이지에 대한
대상 지정 콘텐츠 추가도 포함된다.

#### 링크 수정과 리소스 업데이트 {#link-fixes-and-resource-updates}

영어 문서에 대한 변경은 영어가 아닌 로케일에서 링크 체크 실패를 일으킬 수 있다.
이는 문서 페이지나 그 안의 섹션이 이동되거나 삭제될 때 발생한다. 이동된 외부
리소스로의 링크도 비슷하게 실패할 수 있다. 이런 실패는 영어 페이지에서만 고친다.
**링크를 고치기 위해 로컬라이즈된 페이지 콘텐츠를 절대 편집하지 않는다.**
[드리프트 추적](#track-changes)이 오래된 로컬라이즈된 사본을 해당 로케일 팀에게
알리며, 링크 수정을 포함한 조정은 각 팀에 맡겨진다.

먼저, 실패한 링크를 고쳐 영어 쪽에서 파급을 막는다. 일부 대상의 운명에는
추가적인 완화책이 있다.

- **페이지가 이동한 경우**: 이동된 영어 페이지가 이전 경로에 대한
  [별칭(alias)][aliases]을 선언하는지 확인한다. 별칭은 사이트 방문자에 한해서
  이전에 게시된 페이지 링크가 계속 작동하게 해준다. 별칭은 서버 사이드
  리디렉션으로 게시되며, 링크 체커는 빌드된 사이트의 정규 페이지 경로를 기준으로
  링크를 해석한다. 따라서 이전 경로로의 링크는 여전히 영어 페이지에서 고쳐야
  한다.
- **섹션이 페이지 내에서 이동한 경우**: 해당 섹션으로의 링크가 계속 작동하도록
  [제목 ID](#headings)를 보존한다. 별칭은 이 경우 도움이 되지 않는다. 별칭은
  프래그먼트가 아니라 페이지 경로를 리디렉션하기 때문이다.

그 밖의 운명에는 이런 완화책이 없다. 다른 페이지로 이동한 섹션, 이동된 외부
리소스, 삭제된 대상이 그렇다. 영어 링크를 고치는 것이 영어 쪽의 수정 전부이다.
삭제된 대상의 경우, 이는 각 링크를 거는 페이지에서 대체 대상을 선택하거나 참조를
삭제하는 것을 의미한다.

그런 다음 드리프트 처리가 로컬라이즈된 페이지를 다루도록 둔다. 영어 페이지를
고치면 그 로컬라이즈된 사본이 [드리프트](#drift-status)되며, 링크 체커는
드리프트된 사본을 건너뛴다. 로컬라이즈된 페이지가 여전히 링크 체크에 실패한다면,
드리프트 상태를 직접 새로고침한다.

```sh
npm run fix:i18n:status -- PATHS_TO_FAILING_LOCALIZED_PAGES
```

드물게 실패한 링크가 로컬라이즈된 페이지에만 존재하는 경우, 이를 보고하고 해당
로케일 팀과 함께 수정을 조율한다.

마지막으로 `npm run check:links`를 다시 실행해 남은 링크 실패가 없는지 확인한다.

[aliases]: https://gohugo.io/content-management/urls/#aliases
[front matter]: https://gohugo.io/content-management/front-matter/
[main]: https://github.com/open-telemetry/opentelemetry.io/commits/main/
[maintainers]: https://github.com/orgs/open-telemetry/teams/docs-maintainers
[multilingual framework]: https://gohugo.io/content-management/multilingual/
[new issue]: https://github.com/open-telemetry/opentelemetry.io/issues/new
[PRs]: ../pull-requests/
[slack]: https://slack.cncf.io/
