---
title: 구성
weight: 20
description:
  오픈텔레메트리 인젝터(OpenTelemetry Injector)가 컬렉터나 백엔드를 가리키도록
  지정하고, 리눅스 호스트에서 계측 대상을 조정한다.
cSpell:ignore: metapackage
default_lang_commit: edb244ceebdcbbb33c640eaac8d218dbc480e4c0
---

[설치](../installation/)가 끝나면
[오픈텔레메트리 인젝터(OpenTelemetry Injector)](https://github.com/open-telemetry/opentelemetry-injector)가
지원되는 애플리케이션을 계측하고, 기본적으로 OTLP를 사용해 `localhost`의
`4317`(gRPC) 및 `4318`(HTTP) 포트로 텔레메트리를 내보낸다. 이 페이지에서는
텔레메트리를 보낼 위치를 변경하는 방법과 주입된 구성을 조정하는 방법을 설명한다.

## 내보내기 대상 설정하기 {#set-the-export-destination}

권장하는 구성은 호스트에서 로컬 [오픈텔레메트리 컬렉터](/docs/collector/)를
실행해 이 텔레메트리를 수신하고 백엔드로 전달하는 것이다.

텔레메트리를 다른 곳으로 보내려면 구성 파일을 사용하는 방법을 권장한다. 처음부터
직접 작성해야 하는 경우는 거의 없다. 각 언어 패키지는
`/etc/opentelemetry/<language>/otel-config.yaml`에 바로 사용할 수 있는 참조
파일을 함께 제공하며, 이 파일은 환경 변수 보간(interpolation)을 통해 익스포터
엔드포인트, 헤더, 서비스 이름을 연결해준다. 이 파일 중 하나를
[선언적 구성(declarative configuration)](/docs/languages/sdk-configuration/declarative-configuration/)에
맞게 복사하거나 수정한 다음, `/etc/opentelemetry/injector/default_env.conf`에
있는 인젝터 환경 파일에서 `OTEL_CONFIG_FILE`을 설정해 활성화한다.

```conf
OTEL_CONFIG_FILE=/etc/opentelemetry/config.yaml
```

> [!NOTE]
>
> .NET의 경우 파일 기반 구성을 사용하려면
> `OTEL_EXPERIMENTAL_FILE_BASED_CONFIGURATION_ENABLED=true`도 설정해야 한다.
> 설정하지 않으면 구성 파일이 조용히 무시된다.

구성 변경 사항을 적용하려면 애플리케이션을 재시작한다.

## 환경 변수로 구성하기 {#configure-with-environment-variables}

구성 파일을 사용하고 싶지 않다면,
`/etc/opentelemetry/injector/default_env.conf`에 있는 인젝터 환경 파일에 표준
오픈텔레메트리 환경 변수를 직접 설정할 수 있다. 여기에 설정한 변수는 계측되는
모든 프로세스에 적용된다. 예를 들어 API 키가 필요한 OTLP 엔드포인트로 직접
내보내려면 다음과 같이 설정한다.

```conf
OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.example.com
OTEL_EXPORTER_OTLP_HEADERS=api-key=REPLACE_ME
```

`default_env.conf`는 표준 오픈텔레메트리 환경 변수를 사용하므로,
`OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`나 다양한 샘플러 및 익스포터
설정과 같이 다른 SDK 동작도 같은 방식으로 구성할 수 있다. 전체 목록은
[SDK 환경 변수](/docs/languages/sdk-configuration/)를 참고한다.

구성 변경 사항을 적용하려면 애플리케이션을 재시작한다.

## 로컬 컬렉터 실행하기 {#run-a-local-collector}

호스트에서 [컬렉터](/docs/collector/)를 실행하면 애플리케이션의 내보내기 구성을
단순하게 유지할 수 있다. 애플리케이션은 `localhost`로 OTLP를 보내기만 하면 되고,
배칭, 재시도, 하나 이상의 백엔드로의 라우팅은 컬렉터가 처리한다. 지금은 컬렉터를
별도로 설치하고 실행해야 한다. 아직 기본 `opentelemetry` 메타패키지에 포함되어
있지 않기 때문이다.

## 다음 단계 {#next-steps}

- [오픈텔레메트리 컬렉터](/docs/collector/)에 대해 더 알아본다.
- 사용 가능한 [SDK 환경 변수](/docs/languages/sdk-configuration/)를 확인한다.
