---
title: 테스트
cSpell:ignore: Tracetest
default_lang_commit: b588b7136fb0f6fb7cf569e65479238e3e2eefc8
---

현재 이 저장소에는 프론트엔드와 백엔드 서비스 모두를 위한 E2E 테스트가 포함되어
있다. 프론트엔드의 경우 [Cypress](https://www.cypress.io/)를 사용하여 웹
스토어의 다양한 플로우를 실행한다. 반면 백엔드 서비스는 통합 테스트를 위한 주요
테스트 프레임워크로 [AVA](https://avajs.dev)를, 트레이스 기반 테스트를 위해
[Tracetest](https://tracetest.io/)를 사용한다.

모든 테스트를 실행하려면 루트 디렉터리에서 `make run-tests`를 실행한다.

특정 테스트 모음만 실행하고 싶다면 각 테스트 유형에 맞는 명령어를 실행할 수
있다[^1]:

- **프론트엔드 테스트**: `docker compose run frontendTests`
- **백엔드 테스트**:
  - 통합: `docker compose run integrationTests`
  - 트레이스 기반: `docker compose run traceBasedTests`

이러한 테스트에 대해 자세히 알아보려면
[Service Testing](https://github.com/open-telemetry/opentelemetry-demo/tree/main/test)을
참고한다.

[^1]: {{% param notes.docker-compose-v2 %}}
