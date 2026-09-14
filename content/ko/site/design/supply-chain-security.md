---
title: 공급망 보안
description: >-
  사이트의 npm 의존성 통제 뒤에 있는 위협 모델(threat model)과 근거이다.
weight: 20
cSpell:ignore: cooldowns repoint unreviewed
default_lang_commit: 55cf22ecef4f7db0860404d4766bf9cc130ae82b
---

통제 자체와 일상적인 절차는 [의존성 관리](../../build/dependencies/)를 참고한다.
인접한 보안 주제는 각자의 자리가 따로 있다: 이 통제를 검증하는 감사의 설계는
[공급망 감사 설계](../supply-chain-audit/)에, 워크플로 트리거와 토큰 권한은
[CI 워크플로](../../build/ci-workflows/#security-model)에, 취약점 신고는 [보안
정책][security policy]에 있다.

## 위협 모델 {#threat-model}

2026년 8월 npm 웜([보안 공지][security notice])이 현재의 태세(posture)를
결정했다: 인기 있는 npm 패키지의 악성 버전이 탈취된 관리자(maintainer)
계정으로부터 게시되었다. 그 패키지의 설치 시점 [라이프사이클
스크립트][lifecycle scripts]가 페이로드(payload)를 실행하고, 탈취한 자격
증명(credential)을 이용해 이를 전파했다. 이 저장소의 PR 브랜치 여러 개가
봉쇄(containment) 이전에 영향을 받았다; `main`이나 운영(production)에는 도달하지
않았다.

이 저장소에 중요한 공격 경로(attack path)는 다음과 같다:

- **버전 해석(version resolution)**: 버전 범위를 해석하는 설치는 무엇이든 방금
  게시된 악성 릴리스를 끌어올 수 있다.
- **라이프사이클 스크립트**: 설치 시점 스크립트 실행은 악성 패키지가 기여자의
  호스트, CI 러너, 빌드 이미지를 탈취하게 만든다.
- **무인(unattended) 설치**: CI 잡과 [Netlify][] 빌드 이미지는 사람이 지켜보지
  않는 상태로 설치하며, 에이전트 세션도 마찬가지다.
- **이름 해석(name resolution)**: 이름으로 레지스트리에 접근할 수 있는 도구
  호출(`npx`)은 로컬 설치가 오래되었거나 없을 때 그 이름을 가진 패키지가
  무엇이든 실행해 버린다.

## 설계 결정 {#design-decisions}

되풀이되는 주제는 **실패 시 닫힘(fail closed)** 이다: [통제][control]를 강제할
수 없을 때, 설치는 그것 없이 진행되는 대신 실패한다.

각 결정은 하나의 **공격 경로**에 답하며, 대략 작동 시점 순서로 나열되어 있다.
마지막 표는 결정을 그 강제 방식에 매핑한다.

- _직접적이든 전이적이든, 모든 의존성은 공격자가 도달할 수 있는 표면이다._
  - **의존성을 최소화한다**: <a id="minimize"></a> 사용하지 않는 의존성과
    편의상의 의존성은 유지하는 대신 제거한다.
- _버전 범위를 해석하는 설치는 방금 게시된 악성 릴리스를 끌어올 수 있다._
  - **잠금 파일 기준으로 설치한다**: <a id="lock"></a> 설치는 [잠금 파일 기준
    정확히(lock-exact)][install contracts] 이루어지며, 커밋되고 검토된
    [`package-lock.json`][]을 그대로 재현한다. 유일한 예외: 로컬 `npm install`은
    어긋나는 잠금 파일을 다시 쓸 수 있다; [검증](#verify)이 이런 재작성을
    잡아낸다.
  - **의도적으로만 해석한다**: <a id="deliberate"></a> 버전 해석은 오직
    [의도적인 의존성 업데이트][dep-updates]에서만 일어나며, 설치의 부수 효과로는
    결코 일어나지 않는다.
    - Renovate의 예약된 잠금 파일 전체 재해석([`lockFileMaintenance`][])은
      설계상 비활성화되어 있다: 트리 전체에 걸친 상시 레지스트리 조회는 일상적인
      전이 최신성만 얻을 뿐이며, 이는 이미 [알림 기반 수정][security updates]이
      다루고 있다.
  - **쿨다운을 마친 릴리스만 해석한다**: <a id="cooldown-releases"></a> 의도적인
    해석조차 [쿨다운 기간][cooldown]보다 어린 릴리스는 무시한다; 악성 릴리스에
    대한 레지스트리 측 회수(takedown)에는 며칠이 걸리기 때문이다.
    - 설계된 예외 하나: [Dependabot 보안 업데이트][security updates]는 알려진
      취약점 수정을 즉시 반영한다.
    - 쿨다운은 레지스트리에서 해석되는 패키지에 적용된다; Node 툴체인 고정 값은
      대신 [하한 정책][npm engines floor]을 따르는데, 서명된 프로젝트 빌드는
      레지스트리의 회수 지연 위험을 공유하지 않기 때문이다.
- _패키지의 설치 시점 스크립트는 기여자 호스트와 빌드 머신에서 공격자 코드를
  실행한다: 웜의 페이로드 경로이다._
  - **검토된 라이프사이클 스크립트만 실행한다**: <a id="scripts"></a>
    [라이프사이클 스크립트][lifecycle scripts]는 [기본
    거부(default-deny)][allowlist]이다.
    - 승인은 버전 단위로 정확하므로, 탈취된 패치 릴리스가 그 이전 버전의 승인을
      물려받을 수 없다.
    - 리뷰는 거부도 함께 기록하므로, 침묵은 항상 미검토를 의미한다.
    - 예외는 사용 지점에서 이름 붙여 인라인으로 다시 활성화하며, 기본 태세를
      약화시키는 방식으로는 결코 처리하지 않는다.
- _다시 활성화된 유일한 훅은 고정된 Hugo 바이너리를 가져온다; 설치 프로그램은 그
  가져오기를 다른 곳으로 돌리거나 고정을 풀 수 있는 환경 변수 재정의를 그대로
  따른다._
  - **Hugo 설치 프로그램 재정의를 거부한다**: <a id="hugo-env"></a> [재빌드
    래퍼(rebuild wrapper)][install contracts]는 이런 재정의가 하나라도 설정되어
    있으면 실행을 거부한다.
- _Netlify [자체 npm 설치][netlify-deps]는 이 저장소가 통제하는 스크립트 밖에서
  무인으로 실행되며, 비활성화할 수 없다._
  - **자동 설치를 무력화한다**: <a id="auto-install"></a> 구성이 [이를
    무력화하며][inert auto-install], 빌드 명령이 [실제
    설치][install contracts]를 수행한다.
- _레지스트리에서 이름으로 해석 가능한 방식으로 호출되는 bin은, 로컬 설치가
  오래되었을 때 그 이름을 차지한 자가 누구든 그것을 실행한다; 등록되지 않은 bin
  이름에 대한 스쿼팅(squat)이 6월에 이를 입증했다._
  - **이름이 아니라 bin을 호출한다**: <a id="no-bare-npx"></a> 저장소
    배선(wiring)은 [결코 맨(bare) `npx`를 사용하지 않는다][no bare npx]; bin은
    설치된 의존성 트리에서 오거나, 그렇지 않으면 요란하게(loudly) 실패한다.
- _조용히 강제되지 않게 된 통제는 없는 것보다 나쁘다._
  - **오래된 npm에서는 실패 시 닫힌다**: <a id="old-npm"></a> 활성화된 npm이
    `.npmrc` 설정을 강제하기에 너무 오래되었으면, 설치는 그대로 진행되는 대신
    실패한다.
  - **신뢰하지 말고 검증한다**: <a id="verify"></a> 무동작(inert)과 잠금 파일
    기준 정확함(lock-exact)은 가정이 아니라 검증된 주장이다.

한눈에 보는 강제 방식:

| 결정                                                                       | 강제 방식                                                                                                               |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| [의존성 최소화][Minimize dependencies]                                     | 의존성 리뷰에서의 관리자 판단; 기계적인 통제는 없다                                                                     |
| [잠금 파일 기준으로 설치][Install from the lock]                           | 모든 [설치 계약][install contracts]에서의 `npm ci`                                                                      |
| [의도적으로만 해석][Resolve deliberately]                                  | [`renovate.json5`][]에서 비활성화된 [`lockFileMaintenance`][]와 [관례][dep-updates]                                     |
| [쿨다운을 마친 릴리스만 해석][Resolve only cooled-down releases]           | npm과 [Renovate][`renovate.json5`]의 [쿨다운][Cooldown]                                                                 |
| [검토된 라이프사이클 스크립트만 실행][Run only reviewed lifecycle scripts] | strict 모드에서의 [허용 목록][allowlist]; 미검토는 설치를 실패시킨다                                                    |
| [Hugo 설치 프로그램 재정의 거부][Refuse Hugo installer overrides]          | 재빌드를 시도하기 전에 이루어지는 [재빌드 래퍼][install contracts]의 환경 변수 검사                                     |
| [자동 설치 무력화][Neutralize the auto-install]                            | [무동작 자동 설치][inert auto-install] 통제                                                                             |
| [이름이 아니라 bin 호출][Invoke bins, not names]                           | [맨(bare) npx 금지][no bare npx] 규칙; 리뷰 규율에 의존하며 기계적인 통제는 없다                                        |
| [오래된 npm에서는 실패 시 닫힘][Fail closed on old npm]                    | 엄격한 엔진 점검을 갖춘 [npm 버전 하한][npm engines floor]                                                              |
| [신뢰하지 말고 검증][Verify, don't trust]                                  | [공급망 감사][Supply-chain audit], [클린 작업 트리 검사][install contracts], 잠금 파일 재작성에 대한 `postinstall` 경고 |

## 선행 사례 {#prior-art}

- 기본 거부(default-deny) 라이프사이클 스크립트는 생태계가 향하는 방향이다:
  - [pnpm][]과 [Yarn][]은 기본적으로 의존성 스크립트를 차단한다.
  - npm에서 승인된 [RFC #54][]는 `allowScripts`를 통해 동일한 모델을 npm에
    도입하며, 버전 단위 정확한 항목도 포함한다.
- 릴리스 쿨다운은 이미 자리 잡은 관행이다:
  - [pnpm은 기본적으로][pnpm defers] 하루보다 어린 릴리스를 지연시킨다.
  - Renovate의 npm [`minimumReleaseAge`][renovate] 보안 프리셋은 npm의 72시간
    언퍼블리시(unpublish) 기간에 맞춰 3일로 설정되어 있다; 여기서 사용하는 더 긴
    값은 생태계 다른 곳에서 채택된 쿨다운과 궤를 같이한다.
- 이 통제 집합은 이미 확립된 프레임워크 가이드에 대응된다:
  - [TUF의 공격 분류 체계(attack taxonomy)][tuf]: 임의 소프트웨어 설치,
    뒤섞기(mix-and-match), 불필요한 의존성 공격이다.
  - [OpenSSF npm 가이드][openssf]: 잠금 파일 기준 정확한 CI 설치이다.

<!-- prettier-ignore-start -->
[`lockFileMaintenance`]: https://docs.renovatebot.com/configuration-options/#lockfilemaintenance
[`package-lock.json`]: https://docs.npmjs.com/cli/configuring-npm/package-lock-json
[`renovate.json5`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/renovate.json5
[allowlist]: ../../build/dependencies/#lifecycle-script-allowlist
[control]: ../../build/dependencies/#controls
[cooldown]: ../../build/dependencies/#release-cooldown
[dep-updates]: ../../build/dependencies/#updating
[Fail closed on old npm]: #old-npm
[inert auto-install]: ../../build/dependencies/#inert-netlify-auto-install
[install contracts]: ../../build/dependencies/#install-contracts
[Install from the lock]: #lock
[Invoke bins, not names]: #no-bare-npx
[lifecycle scripts]: https://docs.npmjs.com/cli/using-npm/scripts
[Minimize dependencies]: #minimize
[netlify-deps]: https://docs.netlify.com/build/configure-builds/manage-dependencies/#npm
[Netlify]: https://www.netlify.com/
[Neutralize the auto-install]: #auto-install
[no bare npx]: ../../build/dependencies/#no-bare-npx
[npm engines floor]: ../../build/dependencies/#npm-version-floor
[openssf]: https://github.com/ossf/package-manager-best-practices/blob/main/published/npm.md
[pnpm defers]: https://pnpm.io/settings/dependency-resolution
[pnpm]: https://pnpm.io/settings/build
[Refuse Hugo installer overrides]: #hugo-env
[renovate]: https://docs.renovatebot.com/presets-security/#securityminimumreleaseagenpm
[Resolve deliberately]: #deliberate
[Resolve only cooled-down releases]: #cooldown-releases
[RFC #54]: https://github.com/npm/rfcs/blob/main/accepted/0054-make-scripts-install-opt-in.md
[Run only reviewed lifecycle scripts]: #scripts
[security notice]: https://github.com/open-telemetry/opentelemetry.io/issues/11210
[security policy]: https://github.com/open-telemetry/opentelemetry.io/security/policy
[security updates]: ../../build/dependencies/#security-updates
[Supply-chain audit]: ../../build/dependencies/#audit
[tuf]: https://theupdateframework.io/docs/security/
[Verify, don't trust]: #verify
[Yarn]: https://yarnpkg.com/advanced/lifecycle-scripts
<!-- prettier-ignore-end -->
