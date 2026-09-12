---
title: 홈페이지 공지사항
description: 홈페이지 공지사항 페이지의 프론트매터 필드와 렌더링 동작이다.
weight: 60
default_lang_commit: 212fd3ead19d20d10975e270e840066902e82233
---

홈페이지 공지사항에는 `content/<lang>/announcements/*.md`를 사용한다.

이 페이지는 공지사항 페이지 필드의 의미와 사이트가 공지사항을 렌더링하는 방식에
대한 진실 공급원(source of truth)이다.

공유되는 공지사항 매개변수(param)는 `content/en/announcements/_index.md`에서
`cascade`를 통해 설정되며, 이는 모든 로케일에 걸쳐 공유된다. `en`이 아닌
로케일은 캐스케이드된 매개변수를 중복 정의해서는 안 된다.

## 프론트매터 필드 {#front-matter-fields}

- `title`: 공지사항 텍스트에 사용되는 공지사항 제목이다.
- `linkTitle`: 공지사항 색인 페이지에 사용되는 선택적 짧은 제목이다.
- `date`: 공지사항이 표시되기 시작해야 하는 날짜이다.
- `expiryDate`: 이후로는 공지사항이 더 이상 표시되지 않아야 하는 날짜이다.
  - 이 값을 이벤트 종료일로 설정한다(예: `2026-06-06`).
  - 공지사항을 나중에 재사용할 수도 있다면, 공지사항 파일이 삭제되지 않도록 이
    줄 끝에 `# keep`을 덧붙인다.
- `weight`: 필수 항목이다. 이 값을 이벤트 종료일을 나타내는 `yyyymmdd` 형식의
  정수로 설정한다(예: `20260606`). 공지사항은 `weight` 오름차순으로 나열되므로,
  가장 먼저 종료되는 이벤트가 맨 위에 표시된다.
- `params`:
  - `eventUrl`: 페이지 링크에 사용되는 기본 이벤트 URL이다. 리눅스
    파운데이션(Linux Foundation) 이벤트의 경우 흔히
    `https://events.linuxfoundation.org/event-name/` 형태이다.
  - `utmParam`: 이벤트 URL에 덧붙는 UTM 매개변수이다. 섹션 색인
    페이지(`content/en/announcements/_index.md`)에서 설정되므로, 재정의하고 싶은
    경우가 아니라면 따로 정의할 필요가 없다.
  - `blogPostURL`: 공지사항 CTA 링크에 사용되는 선택적 사이트 블로그 게시물
    URL이다.

## 배너 텍스트 {#banner-text}

배너 텍스트는 짧고 간결하게 유지한다. 흔히 쓰이는 본문 템플릿은 다음과 같다:

```markdown
[**{{%/* param title */%}}**][LF], **<span class="text-nowrap">March
23–26,</span> Amsterdam**. <span class="d-none d-md-inline"><br></span> Come
[collaborate, learn, and share][blog]<span class="d-none d-sm-inline"> with the
Cloud Native community</span>!

[blog]: <{{%/* param blogPostURL */%}}>
[LF]: <{{%/* param eventUrl */%}}register/?{{%/* _param utmParam */%}}>
```

설계 참고 사항:

- 화면 크기에 따라 배너 텍스트의 표시 여부를 제어하기 위해 `d-none`,
  `d-*-inline`과 같은 클래스를 사용한다. 이를 통해 작은 화면에서 배너 텍스트를
  더 간결하게 만들 수 있다.
- `utmParam`에 접근할 때는 `_param`을 사용하는데, 이는 값이 안전하다고 가정하기
  때문이다. 이는 쿼리 매개변수의 `&`를 `&amp;`로 이스케이프하는 `param`과
  대조되며, 이러한 이스케이프는 원치 않는 동작이다.

## 렌더링 동작 {#rendering-behavior}

- 홈페이지 배너 템플릿: `layouts/_partials/banner.html`
- 커뮤니티 이벤트 목록 숏코드: `layouts/_shortcodes/community-events.md`.

두 경우 모두 `.RegularPages`를 사용해 공지사항을 렌더링한다:

- Hugo는 `expiryDate`를 기준으로 만료된 페이지를 자동으로 제외한다.
- 페이지는 Hugo의 [기본 페이지 순서][default page order]에 따라 나열되며, 이는
  먼저 `weight` 오름차순으로 정렬한다. `weight`가 (위에서 설명한 대로)
  `yyyymmdd` 형식의 종료일 정수로 설정되므로, 가장 먼저 종료되는 공지사항이 맨
  위에 표시된다.

[default page order]:
  https://gohugo.io/quick-reference/glossary/#default-sort-order
