---
title: 링크 검사
weight: 12
description: 사이트의 링크가 로컬과 CI에서 어떻게 검사되는지 설명한다.
default_lang_commit: 38e36ae231c523f9e54499ad6ca05de7c49501c5
---

이 사이트는 **[Lychee][]** 로 링크를 검사하며, 커밋된 외부 링크 결과 캐시를
기반으로 한다(참고: [링크 캐시][Link cache]).

> [!NOTE] 로컬에 Lychee를 설치하는 것은 선택 사항이다
>
> CI는 모든 PR의 링크를 검사하며, 봇이 [링크 캐시](#link-cache)를 대신
> 업데이트해 줄 수 있다. 로컬에서 검사를 실행하려면 [Lychee를
> 설치한다][lychee-install]; CI는 자체적으로 고정된 버전을 설치하므로
> (`.github/actions/install-lychee` 액션 참고), 로컬 버전을 그와 어느 정도
> 가깝게 유지한다.

## 링크 검사하기 {#check-links}

로컬에서 링크를 검사하려면 다음을 실행한다:

```sh
npm run check:links
```

## 자주 쓰는 명령 {#common-commands}

| 명령                   | 검사 범위                                                                 |
| ---------------------- | ------------------------------------------------------------------------- |
| `check:links`          | 사이트 전체                                                               |
| `check:links:internal` | 사이트 전체, 오프라인(외부 링크 제외)                                     |
| `check:links:diff`     | 변경된 파일만                                                             |
| `fix:link-cache`       | `check:links`의 별칭이다; [링크 캐시][link cache]를 새로 고칠 때 사용한다 |

`check:links`와 `check:links:internal` 스크립트는 `BUILD_KIND` 빌드에 대해
실행된다; `check:links:diff`는 기존 `public/` 빌드의 파일을 검사한다. 자세한
내용은 [빌드 종류: full과 lean][Build kinds: full and lean]을 참고한다.

## 구성 {#configuration}

Lychee는 생성되고 git에서 무시되는 `lychee.toml`을 사용해 빌드된 사이트
(`public/`)에 대해 실행된다. `generate:config:links` 스크립트는
[`lychee.base.toml`][]과, 페이지 프론트매터로부터 계산되는 `exclude_path` 블록을
조합해 이 파일을 만들어내며, 이 블록에는 두 가지 출처가 있다:

- **`link_check_exclude_path`** — 링크 검사기가 건너뛰어야 하는 페이지(블로그
  페이지네이션, 오래된 블로그 게시물 등)에 대한 사이트 상대 경로 정규식
  목록이다; [`content/en/blog/_index.md`][blog-index]를 참고한다. 모든 로케일을
  포괄하려면 패턴을 `^(../)?`로 시작한다: 선택적인 `../`는 `ja/`와 같은 두 글자
  로케일 경로 세그먼트와 매칭된다.
- **`drifted_from_default`** — [드리프트된 로컬라이즈 페이지][drifted]이며,
  상태는 `true`(EN 대응 페이지가 변경됨) 또는 `file not found`(EN 대응 페이지가
  삭제됨)이다. 이런 페이지에서 나가는 링크는 오래되었을 수 있으므로 검사하지
  않지만, 해당 페이지 자체는 여전히 유효한 링크 대상이다: 프래그먼트를 포함해
  동기화된 페이지에서 들어오는 링크는 계속 검증된다.

저장된 드리프트 상태는 최근 야간 [하우스키핑][Housekeeping] 상태 동기화만큼만
최신이므로(병합 시점 기준이라 그 간격이 하루를 넘을 수도 있다), 생성기는
**드리프트 대기(drift-pending)** 페이지도 건너뛴다: 트리 전체 상태
동기화(`npm run fix:i18n`)가 `data/l10n-drift.yaml`에 기록하는 main 브랜치
커밋인 **드리프트 상태 기준선(drift-status baseline)** 이후에 변경(또는 삭제)된
영문 페이지의 로케일 사본이다. 기준선 이후 그 사본 자체가 변경되었다면 계속 검사
대상이다: 누군가 작업 중이라는 뜻이다. 기준선이 없거나 해석할 수 없으면 구성
생성이 실패한다; CI에서는 `CHECK LINKS` 잡이 먼저 얕은 클론을 기준선 커밋까지
확장한다; 로컬에서는 누락된 히스토리를 가져오거나(`git fetch upstream main`)
기준선을 재정의한다: `DRIFT_BASELINE=HEAD npm run check:links`는 오버레이를
비운다(저장된 상태 기반 건너뛰기는 계속 적용된다).

로컬에서 실행한 트리 전체 상태 동기화(`npm run fix:i18n`)는
`data/l10n-drift.yaml`을 재작성할 수 있다; 이 재작성은 커밋하지 않고 남겨둔다 —
로컬에 기록된 커밋이 업스트림에는 존재하지 않을 수 있기 때문이다.

## 링크 캐시 <a id="refcache"></a> {#link-cache}

외부 링크 검사 결과는 `.lycheecache`에 캐시되며, 이 파일은 버전 관리 대상이므로
검사는 새로운 URL이거나 캐시 항목이 만료된 URL만 가져온다. Lychee는 성공한
결과만 캐시하므로, 실패한 항목은 실행할 때마다 다시 시도된다.

이 캐시는 여러 [예약된 워크플로](#workflows)와 콘텐츠 PR에 의해 정기적으로
업데이트되므로, 동시 업데이트는 충돌로 보고되는 대신 Git의 `union` 전략(참고:
[`.gitattributes`][])으로 줄 단위 병합된다. 이런 병합은 중복되거나 오래된 항목을
남길 수 있는데, 이는 검사기 입장에서 무해하며 다음 링크 검사 실행이 캐시를
깔끔하게 다시 작성한다. PR에서는 그 재작성 결과를 커밋한다.

외부 링크를 추가하거나 변경했다면, **PR을 제출하기 전에**
`npm run check:links`를 실행하고 — 실행 시간의 대부분은 사이트 빌드가 차지한다 —
업데이트된 `.lycheecache`를 콘텐츠 변경 사항과 함께 커밋한다. 그렇지 않으면
`CACHE updates committed?` 검사가 실패한다; 복구 절차는
[`CACHE updates committed?`][pr-checks]를 참고한다.

## 캐시 새로 고침 및 하우스키핑 워크플로 {#workflows}

다음 워크플로는 매일 예약 실행되며 링크 검사 명령을 실행한다:

| 워크플로                                                       | 링크 검사 명령                           |
| -------------------------------------------------------------- | ---------------------------------------- |
| Refcache refresh                                               | `log:check:links` (full 빌드, 정리 이후) |
| [하우스키핑][Housekeeping] (`fix-and-test:all`)                | `fix:link-cache` (full 빌드)             |
| [레지스트리 버전 자동 업데이트][Auto-update registry versions] | `fix:link-cache`                         |

Refcache refresh는 가장 오래된 캐시 항목을 정리한 뒤(개수는 워크플로 입력값이다)
링크 검사를 다시 실행하며, 이를 통해 사이트에서 여전히 사용되는, 정리 대상이었던
URL의 캐시 항목을 새로 고친다.

### 실패한 링크 다시 확인하기 {#double-check}

일부 사이트는 브라우저에는 유효한 페이지를 제공하지만 Lychee 같은 일반 HTTP
클라이언트는 거부한다(봇 차단(bot wall), crates.io의 무조건적인 404, npmjs.com의
로그인 리다이렉트 등). [실패는 캐시되지 않으므로](#link-cache), 이런 사이트로의
링크는 캐시 항목이 만료될 때마다 매 실행마다 링크 검사에 실패하게 된다.

**double-check** 도구는 브라우저 수준의 프로브(probe)를 통해 Lychee가 보고한
실패를 다시 검증한다. 프로브가 정상으로 판단한 URL은 `.lycheecache`에 합성 상태
`206`("OK by analysis")으로 기록된다. Refcache refresh 워크플로는 링크 검사
이후에 이를 실행한다; 캡처된 로그에 대해 로컬에서 실행하려면:

```sh
npm run log:check:links
npm run fix:link-cache:double-check
```

옵션을 보려면 `npm run fix:link-cache:double-check -- --help`를 실행한다. 프로브
동작과 설정에 대해서는 [double-check README][]를 참고한다.

## CI에서 {#in-ci}

[`check-links.yml` 워크플로][ci]는 사이트를 한 번(lean으로) 빌드해 그 아티팩트를
`CHECK LINKS` 잡과 공유하므로, 로컬 실행과 CI가 동일한 빌드를 검사한다. 그 잡은
링크 검사가 하나라도 실패하면 실패하며, 새로 고친 캐시를
`CACHE updates committed?` 잡에 넘겨주는데, 이 잡은 실행 결과로 커밋된
`.lycheecache`가 오래된 상태로 남으면 실패한다.

<!-- prettier-ignore-start -->
[`.gitattributes`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/.gitattributes
[Auto-update registry versions]: ../scripts/#update-registry-versionssh
[blog-index]: https://github.com/open-telemetry/opentelemetry.io/blob/main/content/en/blog/_index.md
[Build kinds: full and lean]: ../#build-kinds
[ci]: ../ci-workflows/
[double-check README]: https://github.com/open-telemetry/opentelemetry.io/blob/main/scripts/lychee/double-check/README.md
[drifted]: /docs/contributing/localization/#track-changes
[Housekeeping]: ../ci-workflows/#housekeeping
[link cache]: #link-cache
[Lychee]: https://lychee.cli.rs/
[lychee-install]: https://lychee.cli.rs/guides/getting-started/
[`lychee.base.toml`]: https://github.com/open-telemetry/opentelemetry.io/blob/main/lychee.base.toml
[pr-checks]: /docs/contributing/pr-checks/#cache-updates-committed
<!-- prettier-ignore-end -->
