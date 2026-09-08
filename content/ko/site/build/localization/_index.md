---
title: 새 로컬라이제이션 설정하기
linkTitle: 로컬라이제이션 설정
description: >-
  오픈텔레메트리(OpenTelemetry) 웹사이트에 새 언어
  로컬라이제이션(localization)을 온보딩하기 위한 단계별 관리자 가이드이다.
weight: 50
default_lang_commit: b3e7218b8d2908571bbd50f19eb9a3566fc60371
---

이 가이드는 오픈텔레메트리 웹사이트 관리자가 새 언어 로컬라이제이션을 온보딩하는
데 필요한 모든 변경 사항을 단계별로 안내한다. 저장소 수준의 변경과 GitHub 조직
수준의 설정을 모두 다룬다.

기여자 대상 측면 — 번역 가이드, 드리프트(drift) 추적, 지속적인 유지보수 — 은
[Site localization][]을 참고한다.

활성 로컬라이제이션 팀과 관련 자료의 정식 레지스트리(canonical registry)는
[`projects/localization.md`][]에 있다.

## 사전 준비 사항 {#prerequisites}

시작하기 전에 다음 사항을 로케일 팀과 함께 확인한다:

- [New localizations][](신규 로컬라이제이션)의 단계에 따라 [kickoff
  issue][](킥오프 이슈)가 등록되어 있다.
- [ISO 639-1][] 언어 코드(`LANG_ID`)가 합의되어 있다.
- 멘토와 초기 기여자의 GitHub 핸들이 파악되어 있다.

이 가이드의 나머지 부분에서는 `LANG_ID`가 나올 때마다 실제 [ISO 639-1][] 코드로
바꾼다(예: 폴란드어의 경우 `pl`).

## 1단계 — Hugo 언어 설정 {#hugo-config}

### 1a단계. 언어 설정 항목 {#step-1a-language-config-entry}

`config/_default/hugo.yaml`의 `languages:` 키 아래에 새 언어에 대한 항목을
추가한다:

```yaml
LANG_ID:
  label: NativeName
  locale: LANG_ID-REGION # optional; see note below
  params:
    description: <site description translated into the new language>
```

`locale` 필드는 선택 사항이다. 예를 들어 RSS 피드 등에서 사이트가 `en-US`,
`pl-PL`, `zh-CN`과 같은 지역 언어 태그를 내보내야 할 때 사용하며, 그렇지 않으면
언어 ID가 사용된다. Google Translate는 중국어의 경우 전체 태그인 `zh-CN`을
필요로 하지만, 대부분의 다른 언어는 기본 서브태그(primary subtag)를 사용한다.

예를 들어 폴란드어 항목은 다음과 같다:

```yaml
pl:
  label: Polski
  locale: pl-PL
  params:
    description: Strona projektu OpenTelemetry
```

### 1b단계. 번역 파일 {#step-1b-translation-file}

`i18n` 디렉터리 안에 `LANG_ID.yaml`(예: `pl.yaml`)이라는 이름의 새 파일을
만든다. 이 파일에는 새 언어로 번역된 문자열이 담긴다. 이 문자열은 주 콘텐츠에
반드시 속하지는 않는 UI 요소나 기타 사이트 구성 요소, 또는 여러 페이지에서
사용되는 문자열에 쓰인다.

## 2단계 — Hugo 콘텐츠 마운트 {#hugo-mounts}

Hugo는 콘텐츠 마운트(content mount)를 사용해 로케일별 콘텐츠를 라우팅하고, 아직
번역되지 않은 섹션은 영문 페이지로 폴백(fallback)한다. 최상위 `mounts:` 섹션
아래 [`config/_default/module-template.yaml`][]에 `LANG_ID`에 대한 블록을
추가한다(이 템플릿은 `module.yaml`로 렌더링된다).

### 기본 설정 {#base-setup}

모든 로케일은 최소한 다음 마운트를 필요로 한다 — 로케일 자체 콘텐츠와, 핵심 영문
섹션에 대한 폴백이다:

```yaml
## LANG_ID
- source: content/LANG_ID # locale-specific pages
  target: content
  sites: &LANG_ID-matrix
    matrix: { languages: [LANG_ID] }
# fallback pages (serve English content where no translation exists yet)
- source: content/en/_includes
  target: content/_includes
  sites: *LANG_ID-matrix
- source: content/en/announcements
  target: content/announcements
  sites: *LANG_ID-matrix
- source: content/en/docs
  target: content/docs
  files: ['! specs/**'] # exclude spec fragments (too large to fall back)
  sites: *LANG_ID-matrix
```

로컬라이제이션이 성숙해짐에 따라 (`ecosystem`과 같은) 추가 섹션을 넣을 수 있다.
예를 들어 `pt` 블록에는 `ecosystem` 폴백이 포함되어 있다:

```yaml
## pt
- source: content/pt
  target: content
  sites: &pt-matrix
    matrix: { languages: [pt] }
# fallback pages
- source: content/en/_includes
  target: content/_includes
  sites: *pt-matrix
- source: content/en/announcements
  target: content/announcements
  sites: *pt-matrix
- source: content/en/docs
  target: content/docs
  files: ['! specs/**']
  sites: *pt-matrix
- source: content/en/ecosystem
  target: content/ecosystem
  sites: *pt-matrix
```

`config/_default/module-template.yaml`에서 기존 로케일 블록과 나란히 새 블록을
삽입하되, 해당 파일에서 사용 중인 현재 정렬 관례를 따른다.

## 3단계 — 맞춤법 검사 {#cspell}

### 3a. cspell 사전 확인 {#3a-check-for-a-cspell-dictionary}

해당 언어에 대한 기존 cspell 사전이 있는지 npm에서 검색한다:

```sh
npm search @cspell/dict
```

`@cspell/dict-LANG_ID`와 일치하는 패키지나 가장 가까운 지역 변형(예: 폴란드어의
경우 `@cspell/dict-pl_pl`)을 찾는다. 사용 가능한 사전의 전체 목록은
[cspell-dicts repository](https://github.com/streetsidesoftware/cspell-dicts#natural-language-dictionaries)에서도
확인할 수 있다.

### 3b. 사전 설치(가능한 경우) {#3b-install-the-dictionary-if-available}

```sh
npm install --save-dev @cspell/dict-LANG_ID
```

이 명령은 패키지를 `package.json`에 추가한다. 업데이트된 `package.json`과
`package-lock.json`을 커밋한다.

### 3c. 커스텀 단어 목록 만들기 {#3c-create-the-custom-word-list}

사이트 로컬 기술 용어를 위한 빈 파일을 만든다:

```sh
touch .cspell/LANG_ID-words.txt
```

빈 파일을 커밋한다. 기여자들이 시간이 지나면서 로케일별 기술 용어를 여기에
추가하게 된다.

### 3d. `.cspell.yml` 업데이트 {#3d-update-cspellyml}

새 언어의 맞춤법 검사를 활성화하려면 [`.cspell.yml`][]에 세 개의 항목을
추가한다:

1. `import:` 아래 — 해당 로케일의 cspell 사전을 임포트한다:

   ```yaml
   - '@cspell/dict-CSPELL_DICT_ID/cspell-ext.json'
   ```

2. `dictionaryDefinitions:` 아래 — 커스텀 단어 목록을 등록한다:

   ```yaml
   - name: LANG_ID-words
     path: .cspell/LANG_ID-words.txt
   ```

3. `dictionaries:` 아래 — 임포트한 사전과 커스텀 단어 목록을 모두 활성화한다:

   ```yaml
   - CSPELL_DICT_ID # the @cspell/dict-CSPELL_DICT_ID package
   - LANG_ID-words # the .cspell/LANG_ID-words.txt list
   ```

각 섹션의 항목은 언어 코드 기준 알파벳순으로 유지한다.

> [!NOTE]
>
> 해당 언어에 대한 cspell 사전 패키지가 없다면 3b 단계와 `import`,
> `dictionaries` 항목은 건너뛴다. 커스텀 단어 목록만 만들고(3c 단계)
> `dictionaryDefinitions` 아래에 등록한다.
>
> 또한 cspell이 검증할 수 없는 콘텐츠를 맞춤법 검사하지 않도록,
> [`.cspell.yml`][]의 `ignorePaths` 목록에 로케일 경로를 추가한다:
>
> ```yaml
> ignorePaths:
>   - content/LANG_ID
> ```

## 4단계 — Prettier(조건부) {#prettier}

Prettier가 해당 언어를 잘 처리하지 못하는 경우 — 예를 들어 오른쪽에서 왼쪽으로
쓰는 문자(right-to-left script)나 비라틴 문자를 사용하는 경우 —
[`.prettierignore`][]에 무시 항목을 추가한다:

```sh
content/LANG_ID/**
```

[`.prettierignore`][]의 기존 무시 항목을 확인해 비슷한 문자 체계를 쓰는 다른
로케일이 이미 제외되어 있는지 살펴보고, 같은 패턴을 따른다. 이 단계는 선택
사항이며, Prettier가 해당 언어에 대해 잘못된 포맷을 만든다고 알려진 경우에만
수행한다.

## 5단계 — GitHub 저장소 자동화 {#gh-repo}

### 컴포넌트 레이블 맵 {#component-label-map}

[`.github/component-label-map.yml`][]에서, `content/LANG_ID/`를 건드리는 모든
PR에 `lang:LANG_ID` 레이블을 트리거하는 항목을 추가한다:

```yaml
lang:LANG_ID:
  - changed-files:
      - any-glob-to-any-file:
          - content/LANG_ID/**
```

항목은 알파벳순으로 유지한다.

### 코드 소유자 {#code-owners}

- [`.github/CODEOWNERS`][]에서, 로케일 팀에 해당 파일의 **단독(sole)** 소유권을
  부여한다. 파일에 제시된 가이드를 따른다.
- 항목은 알파벳순으로 유지한다.
- [`.github/component-owners.yml`][]에는 어떤 항목도 추가하지 **않는다**. 2026년
  6월부터 이 파일에 대한 변경은 더 이상 필요하지 않다.

## 6단계 — GitHub 조직 수준 설정 {#gh-org}

이 단계들은 저장소 밖에서 이루어지며 `open-telemetry` GitHub 조직에 대한 관리자
수준 접근 권한이 필요하다.

팀 생성은 (비공개) [`open-telemetry/admin`][] 저장소에 풀 리퀘스트(pull
request)를 여는 방식으로 이루어진다. 예상되는 형식의 예시는
[이 PR](https://github.com/open-telemetry/admin/pull/588?link-check=no)을
참고한다.

> [!NOTE]
>
> [팀원은 수동으로 추가해야 한다](https://github.com/open-telemetry/admin/issues/58?link-check=no).
> 현재 이 저장소에서 팀원을 관리하고 있지 않기 때문이다.

## 7단계 — Slack 채널 {#slack}

[CNCF Slack workspace][]에 해당 로케일의 채널을 만들되,
`#otel-localization-LANG_ID`(예: 폴란드어의 경우 `#otel-localization-pl`) 명명
규칙을 사용한다. 채널을 만든 뒤
[OpenTelemetry Admin](https://cloud-native.slack.com/team/U07DR07KAEQ)을 채널
관리자로 추가한다.

## 8단계 — 프로젝트 추적 {#projects}

[`projects/localization.md`][]를 새 로케일 정보로 업데이트한다:

1. 언어 코드 기준 알파벳순으로, 상단의 지원 언어 목록에 언어를 추가한다:

   ```markdown
   - [NativeName - EnglishName (LANG_ID)][LANG_ID]

   [LANG_ID]: https://opentelemetry.io/LANG_ID/
   ```

2. 기존 항목과 동일한 구조를 따라 **Current language teams** 아래에 팀 항목을
   추가한다:

   ```markdown
   **EnglishName**:

   - Website: <https://opentelemetry.io/LANG_ID/>
   - Slack channel:
     [`#otel-localization-LANG_ID`](https://cloud-native.slack.com/archives/XXXXXXXXXXX)
   - Maintainers: `@open-telemetry/docs-LANG_ID-maintainers`
   - Approvers: `@open-telemetry/docs-LANG_ID-approvers`
   ```

3. **Labels** 섹션에 `lang:LANG_ID` 레이블을 추가한다:

   ```markdown
   - [`lang:LANG_ID`][issues-lang-LANG_ID] - EnglishName localization
   ```

   그리고 그에 대응하는 링크 정의를 추가한다:

   ```markdown
   [issues-lang-LANG_ID]:
     https://github.com/open-telemetry/opentelemetry.io/issues?q=is%3Aissue%20state%3Aopen%20label%3Alang%3ALANG_ID
   ```

4. Slack 채널 링크 정의를 추가한다:

   ```markdown
   [otel-localization-LANG_ID]:
     https://cloud-native.slack.com/archives/CHANNEL_ID
   ```

## 검증 {#verification}

### 설정 체크리스트 {#setup-checklist}

리뷰를 요청하기 전에 이 체크리스트를 사용해 모든 설정 단계가 완료되었는지
확인한다:

- [ ] **1단계** — `config/_default/hugo.yaml`에 언어 항목 추가
- [ ] **2단계** — `config/_default/module-template.yaml`에 콘텐츠 마운트 추가
- [ ] **3단계** — cSpell 설정 완료: 사전 설치(또는 `ignorePaths`에 로케일 추가),
      `.cspell/LANG_ID-words.txt`에 커스텀 단어 목록 생성, `.cspell.yml`
      업데이트
- [ ] **4단계** — `.prettierignore` 업데이트(문자 체계에 해당하는 경우)
- [ ] **5단계** — 해당 로케일에 대해 `.github/component-label-map.yml`(레이블
      항목)과 `.github/CODEOWNERS`(단독 소유권 블록) 업데이트
- [ ] **6단계** — `open-telemetry/admin`에 팀 PR 오픈, 팀원 수동 추가
- [ ] **7단계** — Slack 채널 `#otel-localization-LANG_ID` 생성, OpenTelemetry
      Admin을 채널 관리자로 추가
- [ ] **8단계** — `projects/localization.md`에 언어 항목, 팀 항목, 레이블, Slack
      채널 링크 업데이트

### 자동화된 점검 {#automated-checks}

모든 PR이 병합된 뒤, 다음을 실행해 설정이 올바른지 확인한다:

- **`npm run build`** — Hugo가 오류 없이 새 언어를 인식하는지 확인한다.
- **`npm run check:spelling`** — cspell 설정이 유효하고 새 사전 항목으로 인해
  오류가 발생하지 않는지 확인한다.
- **GitHub 레이블 자동화** — `content/LANG_ID/` 아래의 파일을 건드리는 테스트
  PR을 열어 `lang:LANG_ID` 레이블이 자동으로 적용되는지 확인한다.

[`.cspell.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.cspell.yml
[`.prettierignore`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.prettierignore
[`.github/component-label-map.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/component-label-map.yml
[`.github/CODEOWNERS`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/CODEOWNERS
[`.github/component-owners.yml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/component-owners.yml
[`projects/localization.md`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/projects/localization.md
[`config/_default/module-template.yaml`]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/config/_default/module-template.yaml
[`open-telemetry/admin`]: https://github.com/open-telemetry/admin?link-check=no
[kickoff issue]: /docs/contributing/localization/#kickoff
[New localizations]: /docs/contributing/localization/#new-localizations
[Site localization]: /docs/contributing/localization/
[ISO 639-1]: https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes
[CNCF Slack workspace]: https://cloud-native.slack.com
