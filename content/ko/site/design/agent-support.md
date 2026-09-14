---
title: 에이전트 지원
description: >-
  오픈텔레메트리(OpenTelemetry) 웹사이트 콘텐츠를 에이전트가 더 쉽게 소비할 수
  있게 만들기 위한 설계 노트이다.
weight: 10
default_lang_commit: 2fcaa100635bf0eb5916e0e26fcefd58e21fad9b
---

더 넓은 [에이전트 친화적인 콘텐츠 전달](/site/features/) 기능에 대한 설계
노트이다.

## 마크다운 콘텐츠 협상 {#markdown-content-negotiation}

요청이 `text/markdown`을 명시적으로 요구하거나 선호할 때, Netlify Edge
Function을 사용해 Hugo가 미리 빌드한 `index.md` 출력을 제공한다.

### 근거 {#rationale}

- 모든 HTML 페이지에 마크다운(Markdown) 상응물이 있어야 하는 것은 아니다.
- HTTP 협상(negotiation)은 전달 계층(delivery layer)의 몫이다.
- 마크다운 아티팩트(artifact)가 없으면 그 함수는 일반 HTML로 대체될 수 있다.

### 규칙 {#rules}

- `GET`과 `HEAD`만 고려 대상이다.
- `.md` 요청과 그 밖의 비페이지(non-page) 리소스 요청은 협상을 건너뛴다.
- 페이지형(page-like) 요청에는 다음이 포함된다:
  - 슬래시 경로
  - 확장자 없는 경로
  - `.../index.html` 경로
- `text/markdown`이 `q`값 0보다 크게 허용되고, 그 `q`값이 `text/html` /
  `application/xhtml+xml`의 최댓값보다 **크거나 같을 때** 마크다운이
  제공된다(가중치가 같으면 마크다운을 선택한다).
- `*/*`와 같은 와일드카드(wildcard)는 의도적으로 무시된다: 명시적인
  마크다운/HTML 미디어 타입만 `q`값에 반영된다. 이는 나중에 재검토될 수 있는
  보수적인 선택이다.
- 마크다운이 없으면 일반 HTML 응답으로 대체된다.
- 협상된 응답은 `Vary: Accept`를 설정한다.
- `/search/`는 HTML만 내보내므로 항상 HTML로 대체된다.

다음은 경로 매핑(path mapping)에 대한 참고 사항이다:

- `/docs/`와 같은 프리티 URL(pretty URL)은 Hugo의 `/docs/index.md` 출력에
  매핑된다;
- `index.html`은 형제(sibling) `.md` 파일에 매핑된다(예: `/docs/index.html` →
  `/docs/index.md`).
- 그 밖의 `.html` 경로는 Netlify의 일반적인 리다이렉트와 라우팅에 맡긴다: 예를
  들어 Netlify는 `/docs.html`을 `/docs/`로 리다이렉트한다.

### 관련 구현 {#related-implementation}

- `config/_default/hugo.yaml`은 이 사이트의 마크다운 출력을 활성화한다.
- `content/en/search.md`는 `outputs: [HTML]`로 검색 페이지를 제외시킨다.
- `netlify.toml`은 다른 라우트 처리보다 앞서 Edge Function을 연결한다.
- `netlify/edge-functions/markdown-negotiation/index.ts`가 협상을 구현하며,
  `netlify/edge-functions/markdown-negotiation.ts`는 이를 다시 내보내는
  (re-export) Netlify 진입점 스텁(entry stub)이다.
