---
title: OBI 라우트 데코레이터 구성
linkTitle: 라우트 데코레이터
description:
  OBI가 파이프라인의 다음 단계로 데이터를 보내기 전에 라우트
  데코레이터(routes decorator) 구성 요소를 구성하는 방법을 알아본다.
weight: 50
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

YAML 섹션: `routes`

YAML 구성의 `routes` 섹션이나 환경 변수로 이 구성 요소를 구성할 수 있다.

이 섹션은 YAML 파일에서 반드시 구성해야 한다. `routes` 섹션을 제공하지 않으면
OBI는 기본 라우트 파이프라인 단계를 생성하고 `heuristic` 라우트 데코레이터를
사용한다.

예를 들면 다음과 같다.

```yaml
routes:
  patterns:
    - /basic/:rnd
  unmatched: path
  ignored_patterns:
    - /metrics
  ignore_mode: traces
```

| YAML                           | 설명                                                                                                                                         | 유형            | 기본값    |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------------- | --------- |
| `patterns`                     | `http.route` 속성을 매칭하고 설정할 URL 경로 패턴 목록. [패턴](#patterns)을 참고한다.                                                          | list of strings | (unset)   |
| `ignored_patterns`             | 무시할 URL 경로 패턴 목록. 일치하면 트레이스/메트릭 이벤트를 버린다. [무시 패턴](#ignored-patterns)을 참고한다.                                | list of strings | (unset)   |
| `ignore_mode`                  | `ignored_patterns`를 사용할 때 어떤 유형의 이벤트를 무시할지 세분화한다. [무시 모드](#ignore-mode)를 참고한다.                                 | string          | all       |
| `unmatched`                    | 트레이스 HTTP 경로가 어떤 `patterns` 항목과도 일치하지 않을 때 수행할 동작을 지정한다. [Unmatched](#unmatched)를 참고한다.                     | string          | heuristic |
| `wildcard_char`                | `heuristic` 모드로 대체되는 경로 세그먼트에 사용할 문자. [와일드카드 문자](#wildcard-char)를 참고한다.                                         | string          | `*`       |
| `max_path_segment_cardinality` | `low-cardinality` 모드에서 경로 세그먼트 및 서비스당 고유 값의 최대 개수. `0`은 제한을 비활성화한다. [low-cardinality 모드](#low-cardinality-mode)를 참고한다. | integer         | `10`      |

## 패턴 {#patterns}

OBI는 제공된 URL 경로 패턴을 매칭하고 `http.route` 트레이스/메트릭 속성을
설정한다. 생성되는 메트릭의 카디널리티(cardinality)를 줄이기 위해 가능하면 항상
`routes` 속성을 사용한다.

각 라우트 패턴은 경로 세그먼트를 그룹화하는 태그가 있는 URL 경로이다. 매처
태그(matcher tag)에는 `:name`이나 `{name}` 형식을 사용할 수 있다.

예를 들어 다음 패턴을 정의하면:

```yaml
routes:
  patterns:
    - /user/{id}
    - /user/{id}/basket/{product}
```

다음 HTTP 경로를 가진 트레이스는 동일한 `http.route='/user/{id}'` 속성을
포함한다.

```text
/user/123
/user/456
```

다음 HTTP 경로를 가진 트레이스는 동일한
`http.route='/user/{id}'/basket/{product}` 속성을 포함한다.

```text
/user/123/basket/1
/user/456/basket/3
```

라우트 매처는 경로 접두사와 일치하는 와일드카드 문자 `*`도 지원한다. 예를 들어
다음 패턴을 정의하면:

```yaml
routes:
  patterns:
    - /user/*
```

`/user`로 시작하는(그리고 `/user` 자체를 포함한) HTTP 경로를 가진 모든 트레이스는
라우트 `/user/*`와 일치한다. 다음 경로는 모두 `/user/*`로 일치한다.

```text
/user
/user/123
/user/123/basket/1
/user/456/basket/3
```

## 무시 패턴 {#ignored-patterns}

OBI는 제공된 URL 경로를 정의된 패턴과 비교하여, `ignored_patterns` 중 하나라도
일치하면 트레이스 및/또는 메트릭 이벤트를 버린다. `ignored_patterns` 필드의
형식은 `patterns` 필드와 동일하다. 무시 패턴은 와일드카드 옵션을 사용하거나
사용하지 않고 정의할 수 있다. 예를 들어 다음 무시 패턴을 정의하면:

```yaml
routes:
  ignored_patterns:
    - /health
    - /v1/*
```

`/v1` 접두사를 가지거나 `/health`와 동일한 이벤트 경로는 모두 무시된다.

이 옵션은 개발이나 서비스 상태 모니터링에 사용되는 특정 경로가 트레이스나
메트릭으로 기록되는 것을 방지하려는 경우에 유용하다.

## 무시 모드 {#ignore-mode}

`ignored_patterns` 속성과 함께 이 속성을 사용하여 어떤 유형의 이벤트가
무시되는지 세분화한다.

`ignore_mode` 속성에 사용할 수 있는 값은 다음과 같다.

- `all`은 `ignored_patterns`와 일치하는 **메트릭**과 **트레이스**를 모두 버린다
- `traces`는 `ignored_patterns`와 일치하는 **트레이스**만 버리며, 메트릭 이벤트는
  무시하지 않는다
- `metrics`는 `ignored_patterns`와 일치하는 **메트릭**만 버리며, 트레이스
  이벤트는 무시하지 않는다

특정 유형의 이벤트를 무시하고 싶을 수 있다. 예를 들어 헬스 체크 API의 성능
메트릭은 알고 싶지만, 해당 트레이스 레코드가 트레이스 데이터베이스에 쌓이는
오버헤드는 원하지 않을 수 있다. 이 경우 `ignore_mode` 속성을 `traces`로
설정하면 `ignored_patterns`와 일치하는 트레이스만 버려지고 메트릭은 계속
기록된다.

## Unmatched {#unmatched}

이 속성은 트레이스 HTTP 경로가 어떤 `patterns` 항목과도 일치하지 않을 때 수행할
동작을 지정한다.

`unmatched` 속성에 사용할 수 있는 값은 다음과 같다.

- `unset`은 `http.route` 속성을 설정하지 않은 상태로 둔다
- `path`는 `http.route` 필드 속성에 경로 값을 복사한다. 이 옵션은 유입(ingestion)
  측에서 카디널리티 폭발(cardinality explosion)을 유발할 수 있다
- `wildcard`는 `http.route` 필드 속성을 별표 기반의 일반적인 `/**` 값으로
  설정한다
- `low-cardinality`는 `heuristic` 규칙을 적용하고, 추가로
  `max_path_segment_cardinality`를 사용하여 서비스당 고유 경로 세그먼트 값의
  개수를 제한한다
- `heuristic`은 다음 규칙에 따라 경로 값에서 `http.route` 필드 속성을 자동으로
  도출한다.
  - 숫자를 포함하거나 ASCII 알파벳(또는 `-`와 `_`) 이외의 문자를 포함하는 경로
    세그먼트는 `wildcard_char`로 대체된다
  - 단어처럼 보이지 않는 알파벳 세그먼트는 `wildcard_char`로 대체된다

`unmatched: ''`를 명시적으로 설정하면 OBI는 빈 값을 `wildcard`로 처리한다.
`unmatched`를 생략하면 OBI의 기본 구성은 `heuristic`을 사용한다.

## 와일드카드 문자 {#wildcard-char}

`unmatched: heuristic`과 함께 이 속성을 사용하여 `heuristic` 모드가 식별한 경로
세그먼트를 어떤 문자로 대체할지 선택한다. 기본적으로 OBI는 별표 `'*'`를
사용한다. 값은 따옴표로 감싸야 하며 반드시 단일 문자여야 한다.

## heuristic 라우트 데코레이터 모드 {#heuristic-route-decorator-mode}

`heuristic` 데코레이터는 최대한 노력하는(best effort) 라우트 데코레이터로, 일부
시나리오에서는 여전히 카디널리티 폭발로 이어질 수 있다. 예를 들어 GitHub URL
경로는 `heuristic` 라우트 데코레이터가 동작하지 않는 좋은 예시인데, URL 경로가
디렉터리 트리처럼 구성되기 때문이다. 이 시나리오에서는 모든 경로가 고유하게
유지되어 카디널리티 폭발로 이어진다.

URL 경로 패턴이 특정 구조를 따르고 고유 ID가 숫자나 무작위 문자로 구성되어
있다면, `heuristic` 데코레이터는 적은 노력으로 사용 사례에 맞게 동작하는 구성
옵션이 될 수 있다. 예를 들어 다음의 가상 Google Docs URL은 낮은 카디널리티
버전으로 올바르게 축소된다.

아래 두 URL 경로는

```text
document/d/CfMkAGbE_aivhFydEpaRafPuGWbmHfG/edit (no numbers in the ID)
document/d/C2fMkAGb3E_aivhFyd5EpaRafP123uGWbmHfG/edit
```

(기본 `wildcard_char`를 사용하여) 낮은 카디널리티 라우트로 변환된다.

```text
document/d/*/edit
```

## low-cardinality 모드 {#low-cardinality-mode}

`unmatched: low-cardinality`는 먼저 `heuristic` 분류기를 적용한 다음, 서비스별로
각 경로 세그먼트의 고유 값을 추적한다. 세그먼트가 `max_path_segment_cardinality`에
도달하면 OBI는 새로운 고유 값을 `wildcard_char`로 대체한다. 이 카디널리티 상한을
비활성화하려면 제한을 `0`으로 설정한다.
