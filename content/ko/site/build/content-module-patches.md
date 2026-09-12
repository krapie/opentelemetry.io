---
title: 콘텐츠 모듈 패치
description: >-
  릴리스 사이에 콘텐츠 모듈에 임시 패치를 만들고 관리하는 방법이다.
weight: 15
default_lang_commit: 92dd44d5998fe30453368880929dc86dd9ba84da
---

이 사이트에 게시되는 스펙 페이지(OTel 명세(specification), OTLP, 시맨틱
컨벤션(semantic conventions), OpAMP)는 [`content-modules/`][content-modules]
아래에 git 서브모듈로 관리되는 업스트림 저장소에서 가져온다. 웹사이트가 각
서브모듈의 특정 릴리스에 고정되어 있으므로, 원본 마크다운은 스냅샷이며 더 새로운
릴리스로 올려야만 업데이트할 수 있다.

[`npm run cp:spec`](../npm-scripts/#submodules-and-content)를 실행하면,
[`cp-pages.sh`][cp-pages]가 서브모듈 콘텐츠를 `tmp/`로 복사하고 `README.md`
파일의 이름을 `_index.md`로 바꾼 뒤, 모든 마크다운 파일에 대해
[`adjust-pages`][script] 스크립트를 실행한다. Hugo는 `tmp/`를 사이트 트리에
마운트하므로, 처리된 페이지가 `/docs/specs/` 아래에 나타난다.

## 스크립트가 하는 일 {#what-the-script-does}

스펙 마크다운 파일은 GitHub 렌더링을 위해 작성된다: Hugo 프론트매터가 없고,
링크는 GitHub URL을 가리키며, 이미지 경로는 저장소 레이아웃을 전제로 한다.
[`adjust-pages`][script] 스크립트는 각 파일에 다음 변환을 적용해 이 간극을
메운다:

- **프론트매터 삽입(Front matter injection)** — 첫 번째 `# Heading`을 `title`로
  추출하고, `linkTitle`을 생성하며, Hugo 프론트매터를 만들어낸다.
  `<!--- Hugo ... --->` 주석 블록에 내장된 프론트매터도 지원한다.
- **버전 스탬핑(Version stamping)** — OTel 스펙, OTLP, semconv 랜딩 페이지의
  title과 linkTitle에 스펙 버전 번호(예: `1.54.0`)를 덧붙인다.
- **URL 재작성(URL rewriting)** — 스펙 저장소를 가리키는 절대 GitHub URL을 로컬
  `/docs/specs/...` 경로로 변환해, 사이트에서 스펙 간 링크가 동작하게 한다.
- **이미지 경로 조정(Image path adjustment)** — 상대 이미지 경로를 Hugo 페이지
  위치에서 올바르게 해석되도록 재작성한다.
- **콘텐츠 제거(Content stripping)** — 사이트에 필요 없는 `<details>` 블록과
  `<!-- toc -->` 섹션을 제거한다.
- **임시 패치(Temporary patches)** — 아직 릴리스에서 수정되지 않은 스펙 이슈에
  대해 정규식 기반 패치를 적용한다(아래 참고).

이 변환들은 스크립트의 [`index.mjs`][script]에서 순서가 있는 규칙 파이프라인으로
실행되며, 순서가 중요하다. 이를 변경할 때는 스펙 페이지를 다시 생성하고 `tmp/`의
전후 diff를 검토하며, 이에 맞춰 스크립트의 특성화 테스트(characterization test,
`index.test.mjs`)를 업데이트한다.

스펙 버전은 [`data/spec-versions.yml`][spec-versions]에 선언되어 있다 — Hugo
데이터 파일이므로 템플릿에서도 접근할 수 있다 — 그리고 버전 업데이트 워크플로에
의해 자동으로 갱신된다. `cp:spec` 실행 시점에, 스크립트는 각 버전이
[`.gitmodules`][gitmodules]의 해당 `*-pin` 항목의 기준 릴리스와 일치하는지
검증하고, 일치하지 않으면 실패한다. (핀(pin)은 정확한 릴리스가 아니라
`git describe` 식별자일 수도 있다 — 예를 들어 초안 스펙(draft-spec) 통합
브랜치에서.)

## 릴리스 사이에 스펙 패치하기 {#patching-specs}

스펙의 깨진 링크나 잘못된 콘텐츠를 고치려면 업스트림 저장소에 PR을 보내고, 새
릴리스를 내고, 이 저장소에서 서브모듈을 올려야 한다. 이 과정은 몇 주에서 몇 달이
걸릴 수 있다. 그동안 깨진 콘텐츠는 CI 실패를 일으키는데 — 가장 흔하게는 사이트의
모든 외부 링크를 검사하는 자동화된 `otelbot/refcache-refresh` PR에서 발생한다.

업스트림 릴리스를 기다리지 않고 CI를 다시 통과시키려면
[`patches.yml`][patches]에 임시 패치를 추가하면 된다 — 코드 변경이 필요 없다.
패치는 빌드 시점에 실행되는 정규식 기반 재작성이며, 버전 추적 기능이 내장되어
있다: 스펙이 패치의 버전 범위를 넘어서면 `cp:spec`이 해당 패치가 오래되어 제거할
수 있다는 경고를 출력한다.

### 1. 패치 항목 추가하기 {#1-add-a-patch-entry}

패치는 [`patches.yml`][patches]에 YAML 목록의 항목으로 선언된다. 새 항목을
추가한다(목록이 비어 있다면 `[]` 마커를 대체한다):

```yaml
- id: 2025-11-21-docker-api-versions
  module: semconv
  minVers: 1.39.0-dev
  file: ^tmp/semconv/docs/
  search: '(https://docs\.docker\.com/reference/api/engine/version)/v1\.(43|51)/(#tag/)'
  replace: '$1/v1.52/$3'
  flags: g
  notes: >-
    Replace older Docker API versions with the latest. See
    open-telemetry/semantic-conventions#3103; upstreamed fix:
    open-telemetry/semantic-conventions#3093
```

각 패치 항목의 필드는 다음과 같다:

- **`id`** — 로그 메시지에 출력되는 고유 ID이다(날짜 + 짧은 설명).
- **`module`** — `spec`, `otlp`, `semconv` 중 하나이다.
- **`minVers`** — 포함되는 하한이다. 서브모듈 버전이 이 값 이상인 동안 패치가
  적용되며, 스펙이 패치의 버전 범위를 넘어서면 오래된 패치가 된다.
- **`maxVers`** — 선택적인, 포함되지 않는 상한이다. 생략하면 `minVers`의 패치
  번호를 하나 올린 값이 기본값이 된다(예를 들어 `1.55.0`은 `maxVers = 1.55.1`을
  의미한다). 이는 기존의 접두사 매칭(prefix-match) 동작과 일치한다. 명시적으로
  설정하면, 서브모듈 버전이 `maxVers`에 도달하는 즉시 패치가 건너뛰어진다(즉,
  버전이 `maxVers` 미만인 동안에만 적용된다).
- **`file`** — 패치를 적용할 파일 경로와 매칭되는 선택적 정규 표현식이다(예:
  `^tmp/semconv/docs/`). 생략하면 모듈의 스펙/문서 트리를 기본값으로 사용한다:
  `spec`은 `^tmp/otel/specification/`, `otlp`는 `^tmp/otlp/docs/`, `semconv`는
  `^tmp/semconv/docs/`이다.
- **`context`** — 선택 사항: `body|front-matter`(기본값: `body`)이다. body
  패치는 줄 단위로 적용되고, `front-matter` 패치는 프론트매터 블록 전체에
  적용된다.
- **`search`** — 교체할 텍스트에 대한 JavaScript `RegExp` 소스이다. 백슬래시가
  그대로 유지되도록 작은따옴표로 감싼 YAML을 쓰는 것이 좋다.
- **`replace`** — JavaScript 교체 구문(`$1`, `$<name>`, `$&`)을 사용하는 대체
  문자열이다. 그룹 참조 뒤에 숫자가 오는 경우에는 이름 있는 캡처 그룹(named
  capture group)을 사용한다: `$108`은 모호하다.
- **`flags`** — 선택적 `RegExp` 플래그이며, 보통 모든 항목을 교체하려면 `g`를
  사용한다.
- **`notes`** — 선택적 자유 텍스트이다: 패치가 하는 일과 업스트림 이슈/PR 링크를
  적는다.

별도의 등록 단계는 필요 없다 — 스크립트는 빌드 중에 [`patches.yml`][patches]의
모든 항목을 적용한다.

### 2. 패치 테스트하기 {#2-test-the-patch}

스펙 복사 단계를 실행하고 패치가 적용되었는지 확인한다:

```sh
npm run cp:spec
```

성공적으로 실행되면 오류가 표시되지 않는다. 그런 다음 `tmp/` 출력에서 문제가
되었던 콘텐츠를 검색해 재작성되었는지 확인할 수 있다. 링크 관련 패치의 경우에는
다음도 실행한다:

```sh
npm run fix:link-cache  # Checks links, updating the link cache
npm test                # Full test run including link checking
```

### 3. 커밋하고 푸시하기 {#3-commit-and-push}

패치가 링크 캐시 PR을 수정하는 과정에서 만들어졌다면(예:
`otelbot/refcache-refresh` 브랜치), `patches.yml`의 변경 사항을 업데이트된
`.lycheecache`와 함께 커밋한 뒤 force-push with lease를 수행한다:

```sh
git add scripts/content-modules/adjust-pages/patches.yml .lycheecache
git commit -m "Patch content modules and refresh the link cache"
git push --force-with-lease
```

### 4. 오래된 패치 제거하기 {#4-remove-obsolete-patches}

스펙의 새 릴리스에 수정 사항이 포함되면, `cp:spec`이 경고를 출력한다:

```text
INFO: scripts/content-modules/adjust-pages/cli.mjs: patch '<id>' is probably
obsolete now that spec '<name>' is at version '<new>' >= '<target>'; if so,
remove the patch
```

이 메시지가 보이면 [`patches.yml`][patches]에서 해당 패치 항목을 삭제한다.
마지막으로 남은 패치라면, 향후 패치를 위한 참고 자료로 남겨두기 위해 삭제하는
대신 주석 처리해도 된다.

[content-modules]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/content-modules
[cp-pages]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/scripts/content-modules/cp-pages.sh
[gitmodules]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/.gitmodules
[patches]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/scripts/content-modules/adjust-pages/patches.yml
[script]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/scripts/content-modules/adjust-pages
[spec-versions]:
  https://github.com/open-telemetry/opentelemetry.io/blob/main/data/spec-versions.yml
