---
title: MCP 서비스
linkTitle: MCP
cSpell:ignore: fastmcp httpx
default_lang_commit: ffef14de849130bdf9ecd9d4912e75f5a8afdbfd
---

이 서비스는 상점의 작업을
[Model Context Protocol](https://modelcontextprotocol.io/)을 통해 도구(tool)로
노출하여, [에이전트 서비스](../agent/)와 그 밖의 MCP 호환 클라이언트가 이를
호출할 수 있게 한다. 각 도구는 프론트엔드 API를 HTTP로 호출하는 얇은
래퍼(wrapper)이다.

[MCP 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/mcp/)

## 계측 라이브러리 {#instrumentation-libraries}

이 서비스는 `opentelemetry-instrument` 래퍼를 통해 시작되지 않는다.
`Dockerfile`은 스크립트를 직접 실행하며, 계측은 코드 내에서 설정된다.

```dockerfile
CMD ["python", "run.py"]
```

`run.py`에서 [Traceloop SDK](https://www.traceloop.com/)는
오픈텔레메트리(OpenTelemetry) SDK를 초기화하고,
`opentelemetry-instrumentation-mcp`를 포함한 자체 계측 라이브러리 번들을
활성화한다. 이후 HTTPX 계측이 명시적으로 활성화된다.

```python
Traceloop.init(
    app_name=os.getenv("OTEL_SERVICE_NAME", "mcp"),
)

HTTPXClientInstrumentor().instrument()
```

이 조합은 수동으로 스팬을 생성하지 않고도 서비스의 양쪽 측면을 모두 다룬다.

- `opentelemetry-instrumentation-mcp` — FastMCP 서버가 처리하는 인바운드 MCP
  도구 호출에 대한 스팬이다.
- `opentelemetry-instrumentation-httpx` — 각 도구가 프론트엔드 API로 보내는
  아웃바운드 HTTP 호출에 대한 클라이언트 스팬이다.

에이전트와 이 서비스가 동일한 MCP 계측으로 계측되어 있기 때문에, 컨텍스트는 MCP
전송(transport) 전반에 걸쳐 전파되며, 에이전트가 수행한 도구 호출은 이 서비스가
수행한 작업과 동일한 트레이스에 나타난다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

`Traceloop.init()`은 배치 스팬 프로세서와 OTLP 익스포터를 갖춘 트레이서
프로바이더를 생성하고, 이를 전역 트레이서 프로바이더로 등록한다. 이를 통해 위의
계측 라이브러리들이 하나의 내보내기(export) 파이프라인을 공유한다.

내보내기 엔드포인트는 `OTEL_EXPORTER_OTLP_ENDPOINT`가 아니라
`TRACELOOP_BASE_URL`에서 가져오며, Traceloop는 여기에 `/v1/traces`를 덧붙인다.
Docker Compose에서 이는 오픈텔레메트리 컬렉터의 OTLP/HTTP 포트를 가리킨다.
`app_name` 인수는 `service.name` 리소스 속성이 되며, 그 밖의 리소스 속성은
`OTEL_RESOURCE_ATTRIBUTES`에서 읽어온다.

### 새 스팬 생성 {#create-new-spans}

이 서비스는 자체적으로 스팬을 생성하지 않는다. 도구는 FastMCP 서버에 등록되며
별도로 래핑되지 않으므로, 모든 스팬은 계측 라이브러리에서 나온다.

```python
self.mcp.tool("add_to_cart")(tools.add_to_cart)
```

이 서비스는 오픈텔레메트리 트레이싱 API를 직접 사용하지 않는다.
`start_as_current_span`을 호출하지 않으며, `set_attribute`를 사용해 스팬을
보강(enrich)하지도 않는다.

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

`Traceloop.init()`은 `TRACELOOP_METRICS_ENABLED=false`가 설정되지 않은 한
메트릭도 구성한다. 이는 주기적으로 내보내는(periodic exporting) 메트릭 리더를
갖춘 미터 프로바이더를 생성하고 전역으로 등록하여, HTTPX 계측 라이브러리가
발생시키는 메트릭이 내보내지도록 한다.

### 커스텀 메트릭 {#custom-metrics}

이 서비스는 커스텀 메트릭을 정의하지 않는다. 미터를 얻거나 자체
계측기(instrument)를 생성하지 않는다.

## 로그 {#logs}

이 서비스는 Python 표준 라이브러리 로거만 구성한다.

```python
logging.basicConfig(level=logging.INFO)
```

Traceloop의 로그 내보내기는 기본적으로 비활성화되어 있으며, 이 서비스는
`LoggerProvider`나 `LoggingHandler`를 설정하지 않는다. 로그 레코드는 OTLP로
내보내지는 대신 표준 출력(stdout)에 기록되어 컨테이너 런타임이 수집하므로,
트레이스와 상관관계(correlation)를 갖지 않는다.
[로그 커버리지 매트릭스](../../telemetry-features/log-coverage/)를 참고한다.

전체 환경 변수 목록과 문제 해결 단계는
[서비스 README](https://github.com/open-telemetry/opentelemetry-demo/tree/main/src/mcp#readme)를
참고한다.
