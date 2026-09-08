---
title: 빌드
description: >-
  CI/CD 워크플로, 이 사이트를 빌드하는 방법, 다양한 사이트 유지보수 활동을
  수행하는 방법이다.
weight: 40
default_lang_commit: 89fe70663d19f375b1566b3688ef25257233cafc
---

이 섹션은 오픈텔레메트리(OpenTelemetry) 웹사이트의 빌드, 배포, 유지보수
프로세스를 구동하는 CI/CD 워크플로, npm 스크립트, 헬퍼 스크립트를 문서화한다.

## 빌드 종류: full과 lean {#build-kinds}

정규(regular) Hugo 웹사이트 빌드인 **full** 빌드 외에도, Docsy는 **lean** 빌드를
지원하며, 이를 통해 전체 검사 범위를 유지하면서도 링크 검사 속도를 크게 높일 수
있다. 자세한 내용은 Docsy 문서의 [Chrome build modes][]를 참고한다.

일부 빌드 npm 스크립트는 항상 같은 종류(full 또는 lean)의 사이트를 빌드한다. 그
외에는 `BUILD_KIND` 환경 변수 값을 사용하며, 설정되지 않은 경우 기본값으로
`lean`을 사용한다.

| 스크립트                   | 빌드 종류    | 초안/예약 게시물 | 축소(Minify) |
| -------------------------- | ------------ | ---------------- | ------------ |
| `build`                    | `BUILD_KIND` | 예               | 아니오       |
| `build:full`               | full         | 예               | 아니오       |
| `build:lean`               | lean         | 예               | 아니오       |
| `build:preview`            | full         | 예               | 예           |
| `build:production`         | full         | 아니오           | 예           |
| `log:build` (CI 아티팩트)  | `BUILD_KIND` | 예               | 아니오       |
| `netlify-build:preview`    | full         | 예               | 예           |
| `netlify-build:production` | full         | 아니오           | 예           |
| 그 외 대부분의 명령        | `BUILD_KIND` | 예               | 아니오       |

링크 검사(link-checking)처럼 먼저 새로 빌드를 강제하는 대부분의 검사는
`BUILD_KIND`를 사용한다. 링크 검사 스크립트에 대한 자세한 내용은
[링크 검사](./link-checking/)를 참고한다.

<!-- prettier-ignore-start -->
[Chrome build modes]: https://github.com/google/docsy/blob/main/docsy.dev/content/en/docs/deployment/chrome.md
<!-- prettier-ignore-end -->
