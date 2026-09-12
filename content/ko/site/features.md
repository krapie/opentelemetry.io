---
title: 기능
description: >-
  대표적인 사이트 기능에 대한 간략한 요약과 주요 참고 자료 링크이다.
weight: 20
default_lang_commit: 4251379e16ba6263a45644c86bbbc9be17d58a06
---

## 에이전트 친화적인 콘텐츠 전달 {#agent-friendly-content-delivery}

에이전트가 사이트 콘텐츠를 더 쉽게 찾아내고 소비할 수 있게 한다. 현재 진행 중인
작업은 콘텐츠 페이지용 마크다운(Markdown) 출력과 `Accept: text/markdown`에 대한
HTTP 협상(negotiation)을 추가한다.

- 상태: 진행 중
- 설계: [에이전트 지원](../design/agent-support/)
- 구현: `netlify/edge-functions/markdown-negotiation.ts` 아래에 로직과 테스트용
  폴더가 있다.
- 참고 자료:
  [opentelemetry.io#9449](https://github.com/open-telemetry/opentelemetry.io/issues/9449),
  [docsy#2596](https://github.com/google/docsy/issues/2596)
