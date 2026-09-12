---
title: 의존성 관리
description: >-
  사이트가 npm 의존성을 설치하고, 검증하고, 업데이트하는 방법이다.
weight: 5
default_lang_commit: 55cf22ecef4f7db0860404d4766bf9cc130ae82b
---

npm 의존성은 커밋된 `package-lock.json`에 고정되며, 설치는 검토된 라이프사이클
스크립트만 실행한다. 이런 통제의 위협 모델과 근거는 [공급망
보안][Supply-chain security]을 참고한다.

## 설치 계약 {#install-contracts}

CI, 데브컨테이너(devcontainer), Netlify는 잠금 파일 기준으로 정확하게, 스크립트
없이 설치한 뒤, 검토된 훅 하나만 명시적으로 다시 활성화한다: 고정된 Hugo
바이너리를 가져오는 `hugo-extended` 재빌드이다. 이 재빌드는
`scripts/rebuild-hugo-extended.mjs`를 통해 실행되며, 이 스크립트는 제한된
백오프(bounded backoff)로 가져오기를 재시도하고, `HUGO_*` 설치
재정의(override)가 하나라도 설정되어 있으면 실행을 거부한다. 설치는 선택적
의존성(optional dependency)을 유지한다: npm은 플랫폼별 바이너리(예:
`sass-embedded`의 Dart Sass 컴파일러)를 `os`/`cpu`로 선택되는 선택적 의존성으로
전달하므로, 이를 생략하면 빌드가 깨진다. 환경별로:

- **CI**: `npm run ci:min`을 실행한다; 사이트를 빌드하는 잡은 이어서
  `npm run ci:prepare`를 실행한다.
- **데브컨테이너**: `npm run install:safe`이며, 동일한 계약을 따른다.
- **Netlify**: [Netlify][] 빌드 명령이
  [무동작 자동 설치](#inert-netlify-auto-install) 이후, 클린 작업
  트리(clean-working-tree) 검사 사이에 `npm run install:safe`를 실행한다:
  - 잠금 파일 드리프트나 그 밖의 Git에서 보이는 변경 사항이 있으면 빌드가
    실패한다.
  - 설치가 전혀 건드리지 않은 경로에서 발생한 실패는 아래
    [오래된 Netlify 빌드 캐시](#netlify-build-cache)를 참고한다.
- **로컬**: `npm run install:safe` 또는 표준 `npm install`을 사용한다. 후자는
  잠금 파일이 `package.json`과 일치하는 동안에는 잠금 파일을 따르며,
  라이프사이클 스크립트를 비활성화하는 대신
  [허용 목록(allowlist)](#lifecycle-script-allowlist)으로 걸러낸다; [로컬
  설정][local setup]을 참고한다.

중첩된 [Docsy][] 테마 설정도 동일한 계약을 따른다: `prepare` 단계는 Docsy 자체의
잠금 파일 기준, 스크립트 없는 테마 의존성 설치를 호출한다.

### 오래된 Netlify 빌드 캐시 {#netlify-build-cache}

Netlify는 [배포 컨텍스트(deploy context)][deploy context]별로 빌드 캐시를
유지한다:

- 운영(production)용 하나
- 배포 미리보기(Deploy Preview)용으로 **헤드 브랜치 이름**별 하나씩이며, 그
  이름의 첫 빌드 시점에 운영 캐시로부터 시드(seed)된다. 캐시 계보(lineage)는
  명시적으로 지워야만 사라진다: 브랜치 삭제는 Netlify에 보이지 않으므로, 같은
  이름으로 다시 만들어진 브랜치(재사용되는 봇 브랜치 이름 포함)는 이전 캐시에
  다시 연결된다.

각 캐시는 [저장소의 클론을 포함하며][includes a clone of the repository], git
서브모듈을 제거하는 커밋을 체크아웃해도 그 서브모듈의 작업 트리는 그대로 남는다.
그래서 제거된 서브모듈이 추적되지 않는 잔여물(untracked residue)로 캐시를 타고
이후 빌드까지 이어져 클린 작업 트리 검사를 실패시킬 수 있다: 배포 로그에는 해당
경로가 `??`로 시작하는 상태 줄로 표시된다.

해당 경로를 `.gitignore`에 추가하는 대신 영향을 받은 [빌드 캐시][build cache]를
지운다:

- **운영**:
  - **Deploys** > **Trigger deploy**에서 캐시를 지우고 사이트를 배포한다.
- **배포 미리보기**: 이미 빌드된 각 브랜치는 자신만의 캐시 사본을 가지며, 이후에
  운영 캐시를 지워도 영향을 받지 않는다.
  - PR의 최신 배포 페이지에서 **Retry** > **Clear cache and retry with latest
    branch commit**으로 지운다. 브랜치 전체를 한 번에 지우는 방법은 없다.

> [!IMPORTANT]
>
> git 서브모듈을 제거한 뒤에는, 그 잔여물이 브랜치별 캐시에 시드되기 전에 제거
> 작업의 일부로 운영 빌드 캐시를 지운다. 재사용되는 봇 브랜치 이름의 캐시 계보도
> 함께 지운다; 운영 캐시 지우기는 그 계보에는 영향을 미치지 않는다.

## 의존성 업데이트하기 {#updating}

정기적인 버전 상향은 [릴리스 쿨다운](#release-cooldown)의 통제를 받는
[Renovate][] PR로 들어오고, 알려진 취약점 수정은 알림 기반으로 들어온다
([보안 업데이트](#security-updates)). 나머지 경우는 수동으로 처리한다; 각 경우
모두, 재생성된 잠금 파일을 `package.json` 변경 사항과 함께 커밋한다.

### 매니페스트 변경 {#manifest-changes}

`package.json`을 직접 수정했든, `npm run update:packages`로 범위 내 모든 버전을
올렸든([릴리스 쿨다운](#release-cooldown)이 제시되는 버전에 적용된다), 변경된
매니페스트에 맞춰 잠금 파일을 정합화(reconcile)한다:

```sh
npm install --package-lock-only --ignore-scripts
```

아래의 `npm update`와 달리, 이 명령은 매니페스트 변경이 요구하는 부분만 다시
작성하고 나머지 항목은 고정된 채로 둔다. 잠금 파일에서 병합 충돌이 발생해도 처리
방법은 같다: `main`의 버전을 유지하고 명령을 다시 실행한다.

### 스크립트를 포함한 패키지 {#script-bearing-packages}

`allowScripts` 항목이 있거나 필요한 패키지를 추가하거나 업데이트할 때, 그 변경을
만드는 기여자는 다음을 수행한다:

1. 새 버전의 라이프사이클 스크립트를 검토한다.
2. 그 결과를 기록하며, 이는 의존성 변경과 함께 커밋되고 PR 리뷰에서 검증된다:
   필요한 스크립트는 정확한 버전 단위 승인으로, 불필요한 스크립트는 이름 단위
   거부(`false`, 이후 버전 상향 때 업데이트할 필요가 없다)로 기록한다.
3. 새로 승인하는 경우, [`.github/renovate.json5`][]의 Renovate 자동 병합 제외
   목록에도 해당 패키지를 추가한다: 승인된 패키지의 모든 버전 상향은 위 단계를
   거쳐야 하므로, 그 업데이트 PR은 기여자를 기다려야 한다.

### 전이 의존성 새로 고침 {#transitive-refresh}

어떤 예약 작업도 잠금 파일 전체를 다시 해석(resolve)하지 않는다 ([해석은
의도적으로 하지 않는다][deliberate]). 전이 의존성을 새로 고치려면, 저장소
루트에서 다음을 필요할 때마다 실행한다(잠금 파일은
`scripts/generate-community-data` 워크스페이스도 포괄한다):

```sh
npm update --package-lock-only --ignore-scripts
```

[릴리스 쿨다운](#release-cooldown)이 적용되며, 여기에는 까다로운 함정이 있다:
조건을 만족하는 버전이 전부 쿨다운보다 최신인 의존성(정확히 고정된 경우가
흔하다)은 그중 하나가 충분히 오래될 때까지 해석 전체를 실패시킨다(`ETARGET`). 그
최신 릴리스를 직접 검토해 신뢰할 수 있다면, 그 이름만 예외로 둔다; 나머지
트리에는 쿨다운이 계속 적용된다:

```sh
npm_config_min_release_age_exclude=PACKAGE_NAME \
  npm update --package-lock-only --ignore-scripts
```

_`PACKAGE_NAME`_ 자리에는 검증하고 신뢰할 수 있는 패키지 이름을 넣는다. 이
예외는 실행할 때마다 적용하며, [`.npmrc`][]에 고정 항목으로 넣으면 해당 이름의
쿨다운이 영구히 면제된다. 또한 메이저 버전 도약(major hop)이 없는지 새로 고쳐진
잠금 파일을 검토한다: `npm update`는 매니페스트에 선언된 범위를 그대로 따르므로,
상위 패키지가 범위를 넓히면 새로운 전이 메이저 버전이 들어올 수 있다.

### 예기치 않은 잠금 파일 변경 {#lock-drift}

의존성을 변경하지 않았는데도 잠금 파일이 바뀌었다면(설치 시 이런 일이 발생하면
`postinstall` 검사가 경고한다), 이는 드리프트의 징후이다: 그 재작성을 커밋하는
대신 잠금 파일을 복원하고 원인을 조사한다.

### 보안 업데이트 {#security-updates}

알려진 취약점 수정은 주간 업데이트 PR을 기다리지 않고 알림 기반으로 들어온다:

- **GitHub [Dependabot 보안 업데이트][Dependabot security updates]**: 저장소 측
  설정이며(`dependabot.yml` 없음), 직접 의존성과 전이 의존성을 모두 패치할 수
  있다; npm의 경우 이는 잠금 파일뿐 아니라 상위 매니페스트 항목을 다시 쓰는 것을
  의미할 수도 있다.
- **[Renovate][] 취약점 알림 PR**: 직접 의존성에 대해 즉시 열린다.

이 중복은 의도된 것이다; 가끔 중복 PR이 생기는 것도 받아들인다. 예약된 잠금 파일
재해석이 [설계상 비활성화되어 있으므로][deliberate], 이런 알림 기반 경로가 전이
의존성 수정에 대한 유일한 자동화 경로이며, 그래서 저장소 측 설정은 계속 켜둔다.
두 경로는 [릴리스 쿨다운](#release-cooldown)을 서로 다르게 처리한다:

- Dependabot 보안 업데이트는 모든 릴리스 나이 게이트(`.npmrc` 포함)를 의도적으로
  무시한다: 쿨다운보다 최신인 수정 버전도 반영될 수 있으며, 이를 검증하는 것은
  리뷰하는 관리자의 몫이다.
- Renovate의 PR은 잠금 파일을 재생성할 때 `.npmrc` 게이트의 적용을 받으므로,
  쿨다운보다 최신인 수정은 실패한 아티팩트 업데이트로 도착한다; 이를 조기에
  반영하려면 관리자가 실행하는 [범위 지정 예외](#transitive-refresh)가 필요하다.

## 공급망 통제 {#controls}

### 공급망 감사 {#audit}

공급망 감사 테스트인 [`scripts/supply-chain-audit.test.mjs`][]는 매
`test:local-tools` 실행마다 커밋된 파일만으로 아래 통제 사항을 검증하므로,
통제가 퇴행하면 사고를 기다리는 대신 테스트가 실패한다. 감사 자체의 검증 원칙에
대해서는 [설계 페이지](../../design/supply-chain-audit/)를 참고한다.

PR에서 감사가 실패하면, 어서션(assertion) 메시지가 기대되는 조건을 알려준다;
흔한 경우는 다음과 같다:

- **`allowScripts` 항목이 있는 의존성의 버전을 올렸다**:
  [스크립트를 포함한 패키지](#script-bearing-packages)를 따른다; 실패 메시지가
  항목을 옮겨야 할 버전을 알려준다.
- **설치 경로 스크립트, `.npmrc`, `netlify.toml`을 변경했다**: 그 실패가 바로
  의도된 지점이다. 감사는 설치 영역(install surface)을 고정해, 그에 대한 모든
  변경이 의도적으로 검토되도록 한다. 변경 사항과 함께 해당 어서션을
  업데이트하고, PR에 그 이유를 밝힌다.

그린 상태로 만들기 위해서 어서션을 느슨하게 풀지 않는다: 각 어서션은 이 페이지의
통제 사항을 강제하므로, 먼저 자신의 변경이 어떤 통제를 완화하는지 부터 파악한다.

감사 범위 밖:

- GitHub 워크플로 파일
- [Renovate][] 구성([`.github/renovate.json5`][]): 코드처럼 리뷰되며, 감사로
  고정되지 않는다.
- [Docsy][] 테마 자체의 의존성 설치(업스트림에서 감사됨)
- 설치 경계를 넘어서는, 빌드 쪽의 npm 스크립트

### 릴리스 쿨다운 {#release-cooldown}

버전 해석은 설정된 최소 나이보다 최신인 릴리스를 무시한다.

- **적용 방식**: [`.npmrc`][]의 `min-release-age`이다.
  `scripts/generate-community-data` 서브프로젝트는 별도의 잠금 파일을 갖는 것이
  아니라 npm 워크스페이스이므로, 루트 `.npmrc`와 잠금 파일이 이 서브프로젝트의
  해석도 지배한다.
- **범위**:
  - 해석(resolve) 작업에만 영향을 준다; 잠금 파일 기준 설치(`npm ci`)는 버전을
    해석하지 않는다.
  - npm은 프로젝트 설정을 사용자 설정보다 우선시하므로, 사용자 `.npmrc`의 더
    엄격한 쿨다운도 여기서는 프로젝트 값으로 완화된다; 특정 실행에서 자신의 값을
    유지하려면 둘 모두보다 우선하는 `npm_config_min_release_age` 환경 변수를
    설정한다.
- **[Renovate][]**: 자신이 여는 업데이트 PR에 [`.github/renovate.json5`][]의
  `minimumReleaseAge`로 설정된 자체 쿨다운을 적용한다; 사람 리뷰 없이 병합되는
  업데이트에는 더 긴 값을 적용한다. 프리셋이 제공하는 3일짜리 npm 쿨다운
  (`security:minimumReleaseAgeNpm`)은 이 나이 값들을 재정의하지 못하도록
  제외되어 있으며, 그 나이 예외도 함께 제외된다; 주의: 해당 프리셋이
  업스트림에서 이름을 바꾸면 이 제외가 조용히 무효화되어 다시 적용될 수 있다.
  Renovate가 날짜를 매길 수 없는 업데이트 유형(`pin`, `replacement`, `rollback`
  등)은 쿨다운 대상에서 제외된다: 이런 PR은 정상적으로 열리며, 기껏해야 (필수
  체크는 아닌) 영구적으로 대기 중인 안정성 상태를 표시할 뿐이므로, 일반적인
  리뷰가 그 관문이 된다.

### 라이프사이클 스크립트 허용 목록 {#lifecycle-script-allowlist}

설치는 패키지의 정확한 이름과 버전이 `allowScripts` 허용 목록에 실려 있을 때만
그 패키지의 라이프사이클 스크립트를 실행한다:

- **적용 방식**: [`package.json`][]의 `allowScripts` 맵이며, [`.npmrc`][]의
  `strict-allow-scripts`에 의해 실패 시 닫히도록 (fail-closed) 만들어져 있다.
- **거부(Denials)**:
  - `false`로 설정된 항목은 검토된 거부를 기록한다: 패키지는 설치되지만
    스크립트는 건너뛴다.
  - 거부는 아무것도 허용하지 않으므로, 버전에 상관없이 이름 기준으로 해당 패키지
    전체에 적용된다.
- **`--ignore-scripts`와의 상호작용**:
  - 허용 목록은 필터링만 한다: `ignore-scripts`가 비활성화한 스크립트를 다시
    활성화하는 일은 없으므로, 스크립트 없는 설치는 허용 목록에 있든 없든 아무
    스크립트도 실행하지 않는다.
  - 검토된 예외를 두려면 호출 지점에서 명시적으로 `--ignore-scripts=false`를
    지정해야 한다.

### npm 버전 하한 {#npm-version-floor}

활성화된 npm이 engines 하한보다 오래되었으면 설치가 실패한다: 이 하한은 위의
통제 사항을 지원하는 가장 오래된 버전이다.

- **적용 방식**:
  - [`package.json`][]의 `engines`가 하한을 설정한다.
  - [`.npmrc`][]의 `engine-strict`가 이를 실패 시 닫히도록 만든다.
- **하한 정책**:
  - npm이 통제 사항의 적용 공백을 메울 때마다 하한이 올라간다.
  - 커밋된 `.nvmrc`는 번들 npm이 이 하한을 만족하는 Node.js 릴리스에 고정되어
    있으므로, CI, Netlify, `nvm`으로 관리되는 로컬 환경은 구조적으로 이를
    통과한다; [Renovate][]가 이 고정 값을 계속 업데이트한다. (`lts/*`처럼
    떠다니는(floating) `.nvmrc`는 이를 보장할 수 없다: CI 러너는 이를 오래되었을
    수도 있는 캐시로부터 해석하기 때문이다.)
- **Netlify**:
  - Netlify의 Node 번들 기본 npm은 하한보다 오래되었을 수 있다;
    [`netlify.toml`][]의 [`NPM_VERSION`][netlify-deps]이 이를 만족하는 버전을
    고정한다.
  - 최소한 하한이 올라갈 때는 이 고정 값도 함께 올린다.

### 무동작 Netlify 자동 설치 {#inert-netlify-auto-install}

빌드 시작 시 이루어지는 Netlify의 [자동 설치][netlify-deps]는
[`netlify.toml`][]의 [`NPM_FLAGS`][netlify-deps]에 의해 무력화된다:

- `--dry-run`: npm은 설치가 무엇을 변경할지 해석하고 기록하지만, 아무것도 쓰지
  않는다.
- `--ignore-scripts`: 라이프사이클 스크립트는 dry run의 부수 효과가 아니라
  명시적인 지시에 의해 비활성화된 상태로 유지된다.

**범위**: `NPM_FLAGS`는 npm 설정이 아니라 Netlify 빌드 설정이다; 자동 설치에만
적용되며, 빌드 명령이 실행하는 npm 실행에는 결코 적용되지 않는다.

**심층 방어(defense in depth)**: [실제 설치][install contracts]는 `npm ci`이며,
이는 `node_modules`를 통째로 교체하므로, 자동 설치나 빌드 캐시가 남긴 잔여물이
있어도 `node_modules`가 클린 작업 트리 검사에는 보이지 않지만(이 검사는 Git에서
보이는 변경 사항만 확인한다) 빌드로 이어지지 않는다.

### 맨(bare) npx 금지 {#no-bare-npx}

저장소 배선(패키지 스크립트, CI, 헬퍼 스크립트, 기여자 문서)은 결코 `npx BIN`
형태로 바이너리를 호출하지 않는다: `node_modules`가 오래되었거나 없으면, `npx`는
공개 레지스트리로 폴백해 그 이름을 가진 패키지가 무엇이든 실행해 버린다. 설치
확인 프롬프트는 방어책이 되지 못한다: 비대화형 (non-interactive) 환경에서는
건너뛰어지고, 다른 환경에서는 반사적인 "예" 응답을 유도하기 때문이다. 기여자
문서의 로컬라이즈 사본은 [드리프트 추적][drift tracking]을 통해 이 규칙을 뒤늦게
따라잡는다.

- **대신**:
  - 패키지 스크립트는 의존성이 제공하는 바이너리를 직접 호출한다; npm이
    `node_modules/.bin`을 해당 스크립트의 `PATH`에 넣어 두므로, 바이너리가
    없으면 레지스트리 트래픽 없이 확실하게 실패한다.
  - 그 `PATH` 항목이 없는 환경(문서, 독립 실행 스크립트)에서는 결코 설치하지
    않는 `npm exec --no -- BIN`을 사용한다.
- **적용 방식**: 리뷰 규율에 의존하며, 자동화된 검사는 없다.

<!-- prettier-ignore-start -->
[`.github/renovate.json5`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/.github/renovate.json5
[`.npmrc`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/.npmrc
[`netlify.toml`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/netlify.toml
[`package.json`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/package.json
[`scripts/supply-chain-audit.test.mjs`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/scripts/supply-chain-audit.test.mjs
[build cache]: https://docs.netlify.com/build/configure-builds/troubleshooting-tips/
[deliberate]: ../../design/supply-chain-security/#deliberate
[Dependabot security updates]: https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates
[deploy context]: https://docs.netlify.com/deploy/deploy-overview/#deploy-contexts
[Docsy]: https://www.docsy.dev/
[drift tracking]: /docs/contributing/localization/#track-changes
[includes a clone of the repository]: https://answers.netlify.com/t/what-does-clear-cache-and-deploy-site-do-specifically/9419/2
[install contracts]: #install-contracts
[local setup]: /docs/contributing/development/#local-setup
[netlify-deps]: https://docs.netlify.com/build/configure-builds/manage-dependencies/#npm
[Netlify]: https://www.netlify.com/
[Renovate]: https://docs.renovatebot.com/
[Supply-chain security]: ../../design/supply-chain-security/
<!-- prettier-ignore-end -->
