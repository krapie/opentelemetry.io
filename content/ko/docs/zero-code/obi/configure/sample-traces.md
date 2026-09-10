---
title: OBI 오픈텔레메트리 트레이스 샘플링 구성
linkTitle: 트레이스 샘플링
description:
  오픈텔레메트리(OpenTelemetry) 트레이스를 샘플링하는 방법을 구성한다.
weight: 70
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2에서는 최상위
> `tracer_provider.sampler` 필드로 샘플링을 구성한다. 자세한 내용은
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

OBI는 트레이스의 샘플링 비율을 구성하기 위해 표준 오픈텔레메트리(OpenTelemetry)
환경 변수를 받는다.

YAML 섹션: `otel_traces_export.sampler`

YAML 구성의 `otel_traces_export.sampler` 섹션이나 환경 변수로 이 구성 요소를
구성할 수 있다.

```yaml
otel_traces_export:
  sampler:
    name: 'traceidratio'
    arg: '0.1'
```

| YAML<p>환경 변수</p>                  | 설명                                                                                                                                                                                                | 유형   | 기본값                  |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ----------------------- |
| `name`<p>`OTEL_TRACES_SAMPLER`</p>    | 샘플러의 이름을 지정한다. [오픈텔레메트리 명세](/docs/languages/sdk-configuration/general/#otel_traces_sampler)의 표준 샘플러 이름을 받는다. 자세한 내용은 [샘플러 이름](#sampler-name)을 참고한다. | string | `parentbased_always_on` |
| `arg`<p>`OTEL_TRACES_SAMPLER_ARG`</p> | 선택한 샘플러의 인수를 지정한다. `traceidratio`와 `parentbased_traceidratio`만 인수가 필요하다. 자세한 내용은 [샘플러 인수](#sampler-argument)를 참고한다.                                          | string | (unset)                 |

## 샘플러 이름 {#sampler-name}

`name` 속성은 다음 표준 샘플러 이름을 받는다.

- `always_on`: 모든 트레이스를 샘플링한다. 트래픽이 많은 애플리케이션에서 이
  샘플러를 사용할 때는 주의한다. 모든 요청마다 새 트레이스가 시작되어 내보내진다
- `always_off`: 어떤 트레이스도 샘플링하지 않는다
- `traceidratio`: 트레이스의 주어진 비율(`arg` 속성으로 지정)을 샘플링한다.
  비율은 0과 1 사이의 실수 값이어야 한다. 예를 들어 값이 `"0.5"`이면 트레이스의
  50%를 샘플링한다. 비율이 1 이상이면 항상 샘플링한다. 비율이 0보다 작으면 0으로
  처리된다. 부모 트레이스의 샘플링 구성을 존중하려면 `parentbased_traceidratio`
  샘플러를 사용한다
- `parentbased_always_on`(기본값): `always_on` 샘플러의 부모 기반(parent-based)
  버전
- `parentbased_always_off`: `always_off` 샘플러의 부모 기반 버전
- `parentbased_traceidratio`: `traceidratio` 샘플러의 부모 기반 버전

부모 기반 샘플러는 트레이스된 스팬의 부모에 따라 다르게 동작하는 복합
샘플러이다. 스팬에 부모가 없으면 루트 샘플러를 사용하여 샘플링 결정을 내린다.
스팬에 부모가 있으면 샘플링 구성은 샘플링 부모에 따라 달라진다.

## 샘플러 인수 {#sampler-argument}

`arg` 속성은 선택한 샘플러의 인수를 지정한다. `traceidratio`와
`parentbased_traceidratio`만 인수가 필요하다.

YAML에서는 이 값을 반드시 문자열로 제공해야 한다. 값이 숫자이더라도 YAML
파일에서 반드시 따옴표로 감싼다(예: `arg: "0.25"`).
