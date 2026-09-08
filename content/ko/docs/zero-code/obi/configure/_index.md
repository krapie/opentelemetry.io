---
title: OBI 구성
linkTitle: 구성
description: OBI를 구성하는 방법을 알아본다.
weight: 4
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

[내보내기(export) 모드](export-modes/), 전역 속성(property), 구성 요소 옵션을
설정하여 OBI를 구성할 수 있다.

OBI가 내보내는 메트릭에 대한 정보는 [내보낸 메트릭](../metrics/) 문서를
참고한다.

낮은 카디널리티(cardinality)의 라우트 데코레이터(routes decorator)를 구성하려면
[routes decorator](routes-decorator/) 문서를 참고한다. 최적의 결과를 얻는 데
매우 중요하다.

## 구성 버전 {#configuration-versions}

OBI v0.11.0 이상에서는 Config v2를 사용한다. Config v2는 독립형(standalone) OBI
및 OBI 컬렉터 리시버(Collector receiver)에서 모두 동작하며, 공통 설정에는
오픈텔레메트리(OpenTelemetry) 선언적 구성(declarative configuration) 구조를
사용하고, OBI 관련 설정은 `extensions.obi` 아래에 배치한다.

- 새 구성을 작성하려면 [Config v2 참조](config-v2/)부터 시작한다.
- 기존 Config v1 파일을 변환하려면
  [Config v1에서 v2로 마이그레이션하는 가이드](migrate-to-config-v2/)를 따른다.

별도로 명시되지 않는 한, 이 섹션의 다른 페이지들은 Config v1을 설명한다. 각
페이지는 해당하는 Config v2 안내로 연결되는 링크를 포함한다.
