---
title: 계측 구성
linkTitle: 계측 구성
weight: 100
cSpell:ignore: enduser hset serverlessapis
default_lang_commit: 5b82e8f9c057d4d4961d41091a4bc75fc9b5b37c
---

이 페이지에서는 여러 계측에 동시에 적용되는 공통 설정을 설명한다.

## 피어 서비스 이름(Peer service name) {#peer-service-name}

[피어 서비스 이름(peer service name)](/docs/specs/semconv/general/attributes/#general-remote-service-attributes)은
연결이 이루어지는 원격 서비스의 이름이다. 이는 로컬 서비스의
[리소스](/docs/specs/semconv/resource/#service)에 있는 `service.name`에
해당한다.

{{% config_option name="otel.instrumentation.common.peer-service-mapping" %}}

호스트 이름 또는 IP 주소를 피어 서비스에 매핑하는 데 사용되며,
`<host_or_ip>=<user_assigned_name>` 쌍을 쉼표로 구분한 목록 형태로 지정한다.
호스트 또는 IP 주소가 매핑과 일치하는 스팬에는 피어 서비스가 속성으로 추가된다.

예를 들어 다음과 같이 설정하면:

```text
1.2.3.4=cats-service,dogs-abcdef123.serverlessapis.com=dogs-api
```

`1.2.3.4`로의 요청은 `peer.service` 속성이 `cats-service`가 되고,
`dogs-abcdef123.serverlessapis.com`으로의 요청은 속성이 `dogs-api`가 된다.

Java 에이전트 버전 `1.31.0`부터는 `peer.service`를 정의할 때 포트와 경로를
지정할 수 있다.

예를 들어 다음과 같이 설정하면:

```text
1.2.3.4:443=cats-service,dogs-abcdef123.serverlessapis.com:80/api=dogs-api
```

`1.2.3.4`로의 요청은 `peer.service` 속성이 재정의되지 않으며, `1.2.3.4:443`은
`peer.service`가 `cats-service`가 되고,
`dogs-abcdef123.serverlessapis.com:80/api/v1`으로의 요청은 속성이 `dogs-api`가
된다.

{{% /config_option %}}

## DB 문(statement) 정제 {#db-statement-sanitization}

에이전트는 `db.statement` 시맨틱 속성을 설정하기 전에 모든 데이터베이스
쿼리/문장을 정제(sanitize)한다. 쿼리 문자열 내의 모든 값(문자열, 숫자)은
물음표(`?`)로 대체된다.

참고: JDBC 바인드 매개변수(bind parameter)는 `db.statement`에 캡처되지 않는다.
바인드 매개변수를 캡처하고 싶다면
[관련 이슈](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/7413)를
참고한다.

예시:

- SQL 쿼리 `SELECT a from b where password="secret"`는 내보내진 스팬에서
  `SELECT a from b where password=?`로 나타난다.
- Redis 명령 `HSET map password "secret"`는 내보내진 스팬에서
  `HSET map password ?`로 나타난다.

이 동작은 모든 데이터베이스 계측에 대해 기본적으로 켜져 있다. 비활성화하려면
다음 속성을 사용한다.

{{% config_option
name="otel.instrumentation.common.db-statement-sanitizer.enabled"
default=true
%}} DB 문 정제(sanitization)를 활성화한다. {{% /config_option %}}

## 메시징 계측에서 컨슈머 메시지 수신 텔레메트리 캡처하기 {#capturing-consumer-message-receive-telemetry-in-messaging-instrumentations}

메시징 계측에서 컨슈머 메시지 수신 텔레메트리를 캡처하도록 에이전트를 구성할 수
있다. 활성화하려면 다음 속성을 사용한다.

{{% config_option
name="otel.instrumentation.messaging.experimental.receive-telemetry.enabled"
default=false
%}} 컨슈머 메시지 수신 텔레메트리를 활성화한다. {{% /config_option %}}

이렇게 하면 컨슈머 측에서 새로운 트레이스가 시작되며, 프로듀서 트레이스와는 스팬
링크(span link)로만 연결된다는 점에 유의한다.

> **참고**: 표에 나열된 속성/환경 변수 이름은 아직 실험적(experimental)이므로
> 변경될 수 있다.

## 최종 사용자(enduser) 속성 캡처하기 {#capturing-enduser-attributes}

[JavaEE/JakartaEE Servlet](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/servlet)
및
[Spring Security](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/spring/spring-security-config-6.0)와
같은 계측 라이브러리로부터
[일반 아이덴티티(identity) 속성](/docs/specs/semconv/registry/attributes/enduser/)
(`enduser.id`, `enduser.role`, `enduser.scope`)을 캡처하도록 에이전트를 구성할
수 있다.

> **참고**: 관련된 데이터의 민감한 특성을 고려하여, 이 기능은 기본적으로 꺼져
> 있으며 개별 속성에 대해 선택적으로 활성화할 수 있다. 데이터 수집을 활성화하기
> 전에 각 속성의 개인정보 보호(privacy) 영향을 신중하게 평가해야 한다.

{{% config_option
name="otel.instrumentation.common.enduser.id.enabled"
default=false
%}} `enduser.id` 시맨틱 속성을 캡처할지 여부를 결정한다. {{% /config_option %}}

{{% config_option
name="otel.instrumentation.common.enduser.role.enabled"
default=false
%}} `enduser.role` 시맨틱 속성을 캡처할지 여부를 결정한다.
{{% /config_option %}}

{{% config_option
name="otel.instrumentation.common.enduser.scope.enabled"
default=false
%}} `enduser.scope` 시맨틱 속성을 캡처할지 여부를 결정한다.
{{% /config_option %}}

### Spring Security {#spring-security}

커스텀
[권한 부여 접두사(granted authority prefix)](https://docs.spring.io/spring-security/reference/servlet/authorization/architecture.html#authz-authorities)를
사용하는 Spring Security 사용자는, 실제 역할(role) 및 스코프(scope) 이름을 더 잘
나타내도록 `enduser.*` 속성 값에서 해당 접두사를 제거하기 위해 다음 속성을
사용할 수 있다.

{{% config_option
name="otel.instrumentation.spring-security.enduser.role.granted-authority-prefix"
default=ROLE_
%}} `enduser.role` 시맨틱 속성에 캡처할 역할(role)을 식별하는 권한 부여(granted
authority)의 접두사이다. {{% /config_option %}}

{{% config_option
name="otel.instrumentation.spring-security.enduser.scope.granted-authority-prefix"
default=SCOPE_
%}} `enduser.scopes` 시맨틱 속성에 캡처할 스코프(scope)를 식별하는 권한
부여(granted authority)의 접두사이다. {{% /config_option %}}
