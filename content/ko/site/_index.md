---
title: 이 웹사이트 소개
linkTitle: 웹사이트 문서
description: 이 사이트가 어떻게 빌드되고, 유지보수되고, 배포되는지 설명한다.
# NOTE: aliases are not currently enabled for this section.
cascade:
  type: docs
  params:
    hide_feedback: true
default_lang_commit: 53f798ca0e698ca5270ac7c63f02e2083ba9f257
---

이 섹션은 사이트 관리자와 기여자를 위한 것이다. 오픈텔레메트리(OpenTelemetry)
웹사이트가 어떻게 구성되고, 빌드되고, 유지보수되고, 배포되는지 설명한다.

<span class="badge fs-6 py-2">
{{% _param FAS person-digging " pe-2" %}} 섹션 공사 중. {{%
_param FAS person-digging " ps-2" %}}
</span>

## 콘텐츠 (계획됨) {#content}

잠정적으로 계획된 콘텐츠 구성은 다음과 같다:

- **소개(About)** — 목적, 소유권, 전체 상태를 포함해 웹사이트 프로젝트에 대한
  상위 수준(high-level) 정보이다.
- **필요 사항, 요구 사항, 기능(Needs, requirements, and features)** —
  이해관계자(stakeholder)의 니즈, 요구 사항, 기타 관련 정보를 기능 단위로
  세분화한 것이다.
- [**스킬(Skills)**](./skills/) — 사이트를 유지보수할 때 에이전트와 사람이
  사용할 수 있는 스킬이다.
- [**디자인(Design)**](./design/) — 아키텍처 설계, 정보 구조(Information
  Architecture, IA), 레이아웃, UX 선택, 테마 관련 결정, 기타 설계 수준의
  아티팩트(design-level artifact)이다.
- [**구현(Implementation)**](./implementation/) — 코드 수준의 구조와 관례,
  Hugo/Docsy 템플릿, SCSS/JS 커스터마이징, 패치, 내부 심(shim)이다.
- [**빌드(Build)**](./build/) — 도구(tooling), 로컬 개발 환경 설정, CI/CD
  워크플로, 배포 환경, 자동화 세부 사항이다.
- **배포(Deployment)** — 오픈텔레메트리 웹사이트에 특화된 배포 동작이다.
- [**테스트(Testing)**](./testing/) — 링크 검사, 접근성(accessibility) 표준,
  테스트, 리뷰 관행, 기타 품질 관련 프로세스이다.
- **로드맵(Roadmap)** — 마일스톤, 백로그, 우선순위, 기술 부채(technical debt),
  설계/구현 결정이다.

## 콘텐츠 추가 {#adding-content}

페이지는 짧고 핵심 정보 위주로 유지한다.

- 결정 사항, 근거(rationale), 제약 조건, 핵심 규칙을 기록한다.
- 긴 배경 설명 섹션보다 간결한 요약을 우선한다.
- 세부 내용은 여기에 반복하지 않고 이슈, 계획, 코드로 링크한다.
- 사이트가 어떻게, 왜 동작하는지 설명하는 데 필요한 콘텐츠만 추가한다.

## 사이트 빌드 정보 {#site-build-information}

{{% td/site-build-info/netlify "opentelemetry" %}}
