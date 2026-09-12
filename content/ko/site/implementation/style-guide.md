---
title: 스타일 가이드
description: Hugo 템플릿의 코드 수준 스타일 관례이다.
default_lang_commit: 53f798ca0e698ca5270ac7c63f02e2083ba9f257
---

이 페이지는 오픈텔레메트리(OpenTelemetry) 웹사이트의 코드 수준 스타일 관례를
기록하며, 현재는 Hugo 템플릿을 다룬다. 글과 마크다운 _콘텐츠_ 스타일에 대해서는
대신 [문서 스타일 가이드](/docs/contributing/style-guide/)를 참고한다.

## Hugo 템플릿 {#hugo-templates}

### 들여쓰기 {#indentation}

코드 가독성을 위해 `{{ if }}`, `{{ range }}`, `{{ with }}` 액션(action) 내부를
포함한 중첩 코드 블록에 적절한 들여쓰기를 사용한다.

### 공백 제어 {#whitespace-control}

생성된 출력에 불필요한 빈 줄과 들여쓰기가 남지 않도록 하면서도 템플릿 소스는
읽기 쉽게 유지하기 위해, Hugo의 [공백 제어(whitespace
control)][whitespace control] 트림 마커(trim marker)인 `{{- ... }}`와
`{{ ... -}}`를 사용한다.

- **기본적으로 오른쪽 트림(right-trim)만 사용한다.** `{{ ... -}}`를 작성하면
  대체로 생성된 출력을 깔끔하게 유지하는 데 충분하다.
- 주변의 빈 줄을 없애는 등, 타당한 이유가 있을 때만 **왼쪽 트림(left
  trim)**(`{{- ... }}`) 또는 **양쪽 트림(double-sided trim)**(`{{- ... -}}`)을
  사용한다.
- **마크업 내부의 인라인 출력**에는 트림을 적용하지 않는다. 예를 들어
  `class="{{ $class }}"`처럼 속성 값을 출력하는 액션에는 트림 마커가 필요 없다.

### `range` 블록 안의 줄바꿈 {#newlines-in-range-blocks}

`range` 블록 안에서는, 생성된 출력에 선행/후행 빈 줄이 생기지 않도록 하면서 반복
사이의 줄바꿈을 제어하기 위해 `{{ "\n" -}}` 또는 `{{- "\n" -}}` 사용을 고려한다.

[whitespace control]: https://gohugo.io/templates/introduction/#whitespace
