---
title: 민감한 데이터 처리
description:
  오픈텔레메트리(OpenTelemetry)에서 민감한 데이터를 처리하는 모범 사례와 가이드
weight: 100
default_lang_commit: 4edfbfc2ff38123678ca63eca95de94ede457623
cSpell:ignore: anonymization
---

오픈텔레메트리를 구현할 때는 민감한 데이터 처리에 유의하는 것이 중요하다.
텔레메트리 데이터를 수집하다 보면 각종 프라이버시 규정 및 컴플라이언스
요구사항의 적용을 받을 수 있는 민감하거나 개인적인 정보를 의도치 않게 수집하게
될 위험이 항상 존재한다.

## 사용자의 책임 {#your-responsibility}

오픈텔레메트리는 텔레메트리 데이터를 수집하지만, 특정 상황에서 어떤 데이터가
민감한지는 스스로 판단할 수 없다. 구현자로서 사용자는 다음에 대한 책임을 진다.

- 적용 가능한 프라이버시 관련 법률 및 규정을 준수하는 것.
- 텔레메트리 데이터 내 민감한 정보를 보호하는 것.
- 데이터 수집에 필요한 동의를 확보하는 것.
- 적절한 데이터 처리 및 저장 관행을 구현하는 것.

또한 사용자는 자신이 사용하는 계측 라이브러리(instrumentation library)가
내보내는 텔레메트리 데이터를 이해하고 검토할 책임도 진다. 이러한 라이브러리 역시
민감한 정보를 수집하고 노출할 수 있기 때문이다.

## 민감한 데이터 관련 고려 사항 {#sensitive-data-considerations}

어떤 데이터가 민감한지는 상황마다 다르다. 예를 들면 다음과 같다.

- 개인 식별 정보(Personally Identifiable Information, PII)
- 인증 자격 증명
- 세션 토큰
- 금융 정보
- 건강 관련 데이터
- 사용자 행동 데이터

## 데이터 최소화 {#data-minimization}

텔레메트리를 통해 민감할 수 있는 데이터를 수집할 때는
[데이터 최소화(data minimization)](https://en.wikipedia.org/wiki/Data_minimization)
원칙을 따른다. 이는 다음을 의미한다.

- 옵저버빌리티 목적에 부합하는 데이터만 수집한다.
- 반드시 필요한 경우가 아니라면 개인 정보 수집을 피한다.
- 집계된 데이터나 익명화된 데이터로도 같은 목적을 달성할 수 있는지 검토한다.
- 수집한 속성이 계속 필요한지 정기적으로 검토한다.

## 민감한 데이터 보호 {#protecting-sensitive-data}

앞 절에서 설명했듯이, 민감한 데이터 수집을 방지하는 가장 좋은 방법은 민감할 수
있는 데이터를 아예 수집하지 않는 것이다. 하지만 특정 상황에서는 이러한 데이터를
수집해야 하거나, 수집되는 데이터를 완전히 통제할 수 없어 후처리 과정에서
데이터를 긁어내야 할 수도 있다. 다음 제안이 이런 경우에 도움이 될 수 있다.

[오픈텔레메트리 컬렉터](/docs/collector)는 민감한 데이터를 관리하는 데 도움이
되는 여러 프로세서를 제공한다.

- [`attribute` 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor):
  특정 속성을 제거하거나 수정한다.
- [`filter` 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor):
  민감한 데이터가 포함된 스팬이나 메트릭 전체를 필터링한다.
- [`redaction` 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor):
  허용된 속성 목록과 일치하지 않는 스팬, 로그, 메트릭 데이터포인트 속성을
  삭제한다.
- [`transform` 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor):
  정규 표현식을 사용해 데이터를 변환한다.

### 사용자 정보 삭제 및 해싱 {#deleting-and-hashing-user-information}

다음은 민감한 [`user`](/docs/specs/semconv/registry/attributes/user/#user-hash)
정보 중 `user.email`을 해싱하고 `user.full_name`을 삭제하는 `attribute` 프로세서
구성이다.

```yaml
processors:
  attributes/example:
    actions:
      - key: user.email
        action: hash
      - key: user.full_name
        action: delete
```

### `user.id`를 `user.hash`로 교체하기 {#replacing-userid-with-userhash}

다음은 `transform` 프로세서 구성으로, `user.id`를 제거하고 이를 `user.hash`로
교체하는 데 사용할 수 있다.

```yaml
transform:
  trace_statements:
    - context: span
      statements:
        - set(attributes["user.hash"], SHA256(attributes["user.id"]))
        - delete_key(attributes, "user.id")
```

> [!WARNING] 익명화를 위한 해싱의 위험과 한계
>
> 사용자의 ID나 이름을 해싱하더라도 필요한 수준의 익명화가 보장되지 않을 수
> 있다. 입력값의 범위가 작고 예측 가능한 경우(예: 숫자로 된 사용자 ID) 해시는
> 실제로 역산이 가능하기 때문이다.

### IP 주소 잘라내기 {#truncating-ip-addresses}

해싱의 대안으로 데이터를 잘라내거나 공통된 접두사나 접미사로 그룹화할 수 있다.
예를 들면 다음과 같다.

- 날짜: 일자는 버리고 연도만, 또는 연도와 월만 남긴다.
- 이메일 주소: 로컬 파트는 버리고 도메인만 남긴다.
- IP 주소: IPv4의 마지막 옥텟(octet)이나 IPv6의 마지막 80비트를 버린다.

다음은 `transform` 프로세서 구성으로, `client.address` 속성의 마지막 옥텟을
버린다.

```yaml
transform:
  trace_statements:
    - context: span
      statements:
        - replace_pattern(attributes["client.address"], "\\.\\d+$", ".0")
```

### redaction 프로세서로 속성 삭제하기 {#delete-attributes-with-redaction-processor}

마지막으로, `redaction` 프로세서를 사용해 특정 속성을 삭제하는 예시는 컬렉터
구성을 위한 보안 모범 사례 페이지의
["민감한 데이터 제거하기"](/docs/security/config-best-practices/#scrub-sensitive-data)
절에서 확인할 수 있다.
