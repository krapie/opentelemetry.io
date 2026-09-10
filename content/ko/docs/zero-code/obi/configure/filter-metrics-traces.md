---
title: 속성 값으로 메트릭 및 트레이스 필터링
linkTitle: 데이터 필터링
description: 속성 값으로 메트릭과 트레이스를 필터링하도록 OBI를 구성한다.
weight: 40
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

> [!NOTE]
>
> 이 페이지는 Config v1 필드 이름과 예시를 사용한다. Config v2는
> [Config v2 참조](../config-v2/)를 참고한다. 기존 파일을 변환하려면
> [마이그레이션 가이드](../migrate-to-config-v2/)를 사용한다.

보고되는 메트릭과 트레이스를 속성 값에 따라 매우 구체적인 이벤트 유형으로
제한하고 싶을 수 있다(예: 네트워크 메트릭을 TCP 트래픽만 보고하도록 필터링).

`filter` YAML 섹션을 사용하면 애플리케이션 메트릭과 네트워크 메트릭을 모두 속성
값으로 필터링할 수 있다. 구조는 다음과 같다.

```yaml
filter:
  application:
    # map of attribute matches to restrict application metrics
  network:
    # map of attribute matches to restrict network metrics
```

애플리케이션 및 네트워크 계열에 속하는 메트릭 목록과 그 속성은
[OBI 내보낸 메트릭](../../metrics/) 문서를 참고한다.

각 `application` 및 `network` 필터 섹션은 맵으로, 각 키는 속성 이름(Prometheus
또는 오픈텔레메트리(OpenTelemetry) 형식)이며 문자열 또는 숫자 매처(아래 참고)를
값으로 가진다. 문자열 매칭에는 `match` 또는 `not_match` 속성을 사용할 수 있다.
두 속성 모두 [glob과 유사한](https://github.com/gobwas/glob) 문자열(전체
값이거나 와일드카드를 포함할 수 있음)을 받는다. `match` 속성을 설정하면 OBI는
해당 속성에 대해 제공된 값과 일치하는 메트릭과 트레이스만 보고한다. `not_match`
속성은 `match`의 부정이다.

다음 예시는 UDP 프로토콜을 제외하고 대상 포트 53으로 향하는 연결의 네트워크
메트릭을 보고한다.

```yaml
filter:
  network:
    transport:
      not_match: UDP
    dst_port:
      match: '53'
```

## 숫자 필터 {#numeric-filters}

OBI v0.6.0부터 숫자 필터도 사용할 수 있다. 예를 들어 다음은 서버 포트가 8000
이상인 모든 스팬을 포함한다.

```yaml
filter:
  application:
    server.port:
      greater_equals: 8000
```

다음 매처를 사용할 수 있다.

- greater_than
- greater_equals
- equals
- not_equals
- less_equals
- less_than

숫자 필터와 문자열 매처를 조합할 수 있다.

```yaml
filter:
  network:
    transport:
      not_match: UDP
    dst_port:
      less_than: 1024
```
