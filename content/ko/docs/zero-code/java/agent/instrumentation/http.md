---
title: HTTP 계측 구성
linkTitle: HTTP
weight: 110
default_lang_commit: 4dda37feaa73c216918aab12f16de959d1977766
---

## HTTP 요청 및 응답 헤더 캡처하기 {#capturing-http-request-and-response-headers}

[시맨틱 컨벤션](/docs/specs/semconv/http/http-spans/)에 따라, 미리 정의된 HTTP
헤더를 스팬 속성으로 캡처하도록 에이전트를 구성할 수 있다. 캡처할 HTTP 헤더를
정의하려면 다음 속성을 사용한다.

{{% config_option name="otel.instrumentation.http.client.capture-request-headers" %}}
HTTP 헤더 이름을 쉼표로 구분한 목록이다. HTTP 클라이언트 계측은 구성된 모든 헤더
이름에 대해 HTTP 요청 헤더 값을 캡처한다. {{% /config_option %}}

{{% config_option name="otel.instrumentation.http.client.capture-response-headers" %}}
HTTP 헤더 이름을 쉼표로 구분한 목록이다. HTTP 클라이언트 계측은 구성된 모든 헤더
이름에 대해 HTTP 응답 헤더 값을 캡처한다. {{% /config_option %}}

{{% config_option name="otel.instrumentation.http.server.capture-request-headers" %}}
HTTP 헤더 이름을 쉼표로 구분한 목록이다. HTTP 서버 계측은 구성된 모든 헤더
이름에 대해 HTTP 요청 헤더 값을 캡처한다. {{% /config_option %}}

{{% config_option name="otel.instrumentation.http.server.capture-response-headers" %}}
HTTP 헤더 이름을 쉼표로 구분한 목록이다. HTTP 서버 계측은 구성된 모든 헤더
이름에 대해 HTTP 응답 헤더 값을 캡처한다. {{% /config_option %}}

이 구성 옵션은 모든 HTTP 클라이언트 및 서버 계측에서 지원된다.

> **참고**: 표에 나열된 속성/환경 변수 이름은 아직 실험적(experimental)이므로
> 변경될 수 있다.

## 서블릿 요청 매개변수 캡처하기 {#capturing-servlet-request-parameters}

Servlet API가 처리하는 요청에 대해, 미리 정의된 HTTP 요청 매개변수를 스팬
속성으로 캡처하도록 에이전트를 구성할 수 있다. 캡처할 서블릿 요청 매개변수를
정의하려면 다음 속성을 사용한다.

{{% config_option name="otel.instrumentation.servlet.experimental.capture-request-parameters" %}}
요청 매개변수 이름을 쉼표로 구분한 목록이다. {{% /config_option %}}

> **참고**: 표에 나열된 속성/환경 변수 이름은 아직 실험적(experimental)이므로
> 변경될 수 있다.

## 알려진 HTTP 메서드 구성하기 {#configuring-known-http-methods}

계측이 대체 HTTP 요청 메서드 집합을 인식하도록 구성한다. 그 밖의 모든 메서드는
`_OTHER`로 취급된다.

{{% config_option
name="otel.instrumentation.http.known-methods"
default="CONNECT,DELETE,GET,HEAD,OPTIONS,PATCH,POST,PUT,TRACE"
%}} 알려진 HTTP 메서드를 쉼표로 구분한 목록이다. {{% /config_option %}}

## 실험적 HTTP 텔레메트리 활성화하기 {#enabling-experimental-http-telemetry}

추가적인 실험적 HTTP 텔레메트리 데이터를 캡처하도록 에이전트를 구성할 수 있다.

{{% config_option
name="otel.instrumentation.http.client.emit-experimental-telemetry"
default=false
%}} 실험적 HTTP 클라이언트 텔레메트리를 활성화한다. {{% /config_option %}}

{{% config_option name="otel.instrumentation.http.server.emit-experimental-telemetry"
default=false
%}}
실험적 HTTP 서버 텔레메트리를 활성화한다. {{% /config_option %}}

클라이언트 및 서버 스팬에 대해 다음 속성이 추가된다.

- `http.request.body.size` 및 `http.response.body.size`: 각각 요청 본문과 응답
  본문의 크기이다.

클라이언트 메트릭에 대해 다음 메트릭이 생성된다.

- [http.client.request.body.size](/docs/specs/semconv/http/http-metrics/#metric-httpclientrequestbodysize)
- [http.client.response.body.size](/docs/specs/semconv/http/http-metrics/#metric-httpclientresponsebodysize)

서버 메트릭에 대해 다음 메트릭이 생성된다.

- [http.server.active_requests](/docs/specs/semconv/http/http-metrics/#metric-httpserveractive_requests)
- [http.server.request.body.size](/docs/specs/semconv/http/http-metrics/#metric-httpserverrequestbodysize)
- [http.server.response.body.size](/docs/specs/semconv/http/http-metrics/#metric-httpserverresponsebodysize)
