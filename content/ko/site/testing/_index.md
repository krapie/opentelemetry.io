---
title: 테스트
description:
  점검(checking) 및 테스트 전략, 프로세스, 테스트 페이지에 대한 설명이다.
default_lang_commit: e3c96ee03d3e6ac852d630334b6a921376f9f55e
---

이 섹션에는 웹사이트 테스트와 배포된 라이브 점검(live check)에서 사용하는 점검
및 테스트 전략, 프로세스, 테스트 페이지가 포함되어 있다.

> 이 섹션은 공사 중이다.

## 테스트 범주 {#test-categories}

[`package.json`](../build/npm-scripts/)의 테스트 스크립트는 자동
검색(auto-discovery)도 함께 좌우하는 명명 규칙(naming convention)에 따라
그룹화되어 있다:

- **`test:base`** 는 핵심 점검을 실행한다(`check`와 동일하다). CI에서는 단일
  작업이 아니라 전용 점검 워크플로가 이를 담당한다.
- **복합(compound) `test:<word>-<word>`** 스크립트는
  `test:compound-tests`(따라서 `test:all`)에 의해 자동으로 검색되어 함께
  실행된다. 스크립트에 복합 이름을 붙이는 것이, 단일 점검(예:
  `test:local-tools`)이더라도 해당 그룹에 포함시키는 방법이다.
- **`test:public`** 은 `tests/public/` 아래의 점검(해당 폴더의 모든
  `*.test.mjs`)을 실행한다. 이 점검은 _빌드된_ `public/` 사이트를 읽으므로
  사전에 `npm run build`가 필요하며, `public/`이 없으면 건너뛴다. 해당 폴더에
  `*.test.mjs`를 추가하는 방식으로 점검을 추가할 수 있으며, `public/`이 없을 때
  건너뛰는 관례를 따른다. 이 점검은 (빌드를 수행하지 않는)
  `test:compound-tests`에서 의도적으로 제외되어 있으며, 대신 기존 빌드
  아티팩트를 재사용하는 CI 작업에서 실행된다.
- **`test:*:live`** 스크립트는 배포된 라이브 사이트를 대상으로 하는 선택적
  점검이다.

복합 테스트(compound-test) 스위트 중 `test:local-tools`에는
[공급망 감사(supply-chain audit)](../build/dependencies/#audit)도 포함되어
있으며, 커밋된 설치 강화(install-hardening) 제어가 퇴행(regress)하면 실패한다.
이 감사가 발동했을 때 해야 할 일은 해당 페이지에서 다룬다.

## 테스트 어서션(assertion) {#test-assertions}

### 목표 {#goal}

테스트가 실패했을 때, 출력 결과는 무엇을 점검했는지 명확히 드러내고 실제 값과
기대 값 사이의 차이(diff)를 뚜렷하게 보여주어야 하며, 장황한 수작성 메시지에
의존하지 않아야 한다.

예를 들어, 다음과 같은 방식은 피한다:

```js
assert.ok(a === b, `expected ${a} to be ${b}`);
```

대신 다음과 같은 방식을 선호한다:

```js
assert.strictEqual(status, expectedStatus, 'HTTP status');
```

### 가이드라인 {#guidance}

아래 항목은 여러 테스트 스위트에서 사용하는 Node의 내장 `node:test` 러너와
`assert` API를 기준으로 작성했다. 같은 원칙은 다른 테스트 프레임워크에도
적용된다. 즉, 간결한 diff를 만들어내는 어서션을 우선하고, 실패 컨텍스트를 짧고
구체적으로 유지하며, 동일한 점검이 반복될 때는 공유 헬퍼로 추출한다.

1. 엄격함과 diff 품질이 중요한 원시값(primitive) 점검에서는 `assert.equal`보다
   `assert.strictEqual`을 선호한다.
2. 짧은 세 번째 인자를 컨텍스트로 추가한다. 예를 들어 `HTTP status`,
   `Content-Type`, `Location`, `Request body` 등이다.
3. 정규 표현식(regular expression)이 `includes`나 연쇄된 `ok` 로직보다 의도를 더
   명확하게 표현할 수 있다면 `assert.match`를 사용한다.
4. 공유 어서션 헬퍼는 여러 파일에 복사-붙여넣기 하지 말고, 이를 사용하는 테스트
   스위트가 임포트하는 모듈에 두어 해당 테스트와 함께 배치한다. 그 모듈이 초점을
   잃지 않는 한, 다른 소규모 테스트 전용 유틸리티도 같은 모듈에 둘 수 있다(예:
   `netlify/edge-functions/lib/test-helpers.ts`의 `assertVaryIncludesAccept`).
