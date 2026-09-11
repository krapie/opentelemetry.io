---
title: 공지사항
description: 특별한 이벤트를 위한 공지나 배너를 만든다.
weight: 50
default_lang_commit: 3c96d90486204d2ebebb2d88f146fb650a0b5fab
---

공지(announcement)는 로케일의 `announcements` 섹션 아래에 있는 _일반 Hugo
페이지(regular Hugo page)_ 이다. 즉, 페이지 날짜(예정 또는 만료) 처리와 국제화
등 Hugo의 내장 기능을 활용하여, 빌드 날짜에 따라 배너를 자동으로 표시하거나
숨기고, 배너 순서를 결정하며, 영어 배너로의 폴백을 처리하는 등의 작업을
수행한다는 뜻이다.

> 공지는 현재 배너로만 사용된다. 향후에는 조금 더 일반적인 형태의 공지도 지원할
> _수도_ 있다.

## 공지 만들기 {#creating-an-announcement}

새 공지를 추가하려면 다음 명령어를 사용해 자신의 로컬라이제이션의
`announcements` 폴더 아래에 공지 마크다운 파일을 만든다.

```sh
hugo new --kind announcement content/YOUR-LOCALE/announcements/announcement-file-name.md
```

원하는 로케일과 파일 이름에 맞게 조정한다. 공지 텍스트는 페이지 본문으로
추가한다.

> 배너의 경우, 공지 본문은 짧은 문구여야 한다.

<!-- markdownlint-disable no-blanks-blockquote -->

> [!NOTE] 로컬라이제이션 관련
>
> **로케일별 공지 오버라이드**를 만드는 경우, 영어 공지와 **동일한 파일 이름**을
> 사용해야 한다.

## 공지 목록 {#announcement-list}

특정 공지는 빌드 날짜가 해당 공지의 `date` 필드와 `expiryDate` 필드 사이에 있을
때 사이트 빌드에 나타난다. 이 필드가 없으면 각각 "지금"과 "영원히"로 간주된다.

공지는 Hugo의 [Regular pages](https://gohugo.io/methods/site/regularpages/)
함수로 결정되는 표준 페이지 순서로 나타난다. 즉, (`weight` 기준으로) "가장
가벼운" 공지가 먼저 나타나고, weight가 같거나 지정되지 않은 경우에는 (`date`
기준으로) 가장 최근 공지가 먼저 나타나는 식이다.

따라서 공지를 맨 위로 강제로 올리고 싶다면 프론트매터(front matter)에서 음수
`weight` 값을 사용한다.

이 저장소의 콘텐츠에서 버그나 문제를 발견했거나 개선을 요청하고 싶다면 [이슈를
생성한다][new-issue].

보안 문제를 발견한 경우, 이슈를 열기 전에
[보안 정책](https://github.com/open-telemetry/opentelemetry.io/security/policy)을
읽어본다.

새 이슈를 보고하기 전에,
[이슈 목록](https://github.com/open-telemetry/opentelemetry.io/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc)을
검색해 해당 이슈가 이미 보고되었거나 해결되지 않았는지 확인한다.

새 이슈를 생성할 때는 짧고 의미 있는 제목과 명확한 설명을 포함한다. 가능한 한
많은 관련 정보를 추가하고, 가능하다면 테스트 케이스도 추가한다.

[new-issue]:
  https://github.com/open-telemetry/opentelemetry.io/issues/new/choose
