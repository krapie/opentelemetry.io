---
title: 챗봇 서비스
linkTitle: 챗봇
cSpell:ignore: gradio httpx
default_lang_commit: ffef14de849130bdf9ecd9d4912e75f5a8afdbfd
---

이 서비스는 데모의 AI 어시스턴트를 위한 채팅 인터페이스를 제공한다.
[Gradio](https://www.gradio.app/) 웹 UI를 제공하고, 사용자 메시지를
[에이전트 서비스](../agent/)로 HTTP를 통해 전달하며, 응답을 렌더링한다.
프론트엔드 프록시를 통해 `/chatbot`으로 노출된다.

[챗봇 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/chatbot/)

## 계측 라이브러리 {#instrumentation-libraries}

이 서비스는 `opentelemetry-instrument` 래퍼를 통해 시작되지 않는다.
`Dockerfile`은 스크립트를 직접 실행하며, 계측은 코드 내에서 설정된다.

```dockerfile
CMD ["python", "run.py"]
```

[에이전트](../agent/) 및 [MCP](../mcp/) 서비스와 달리, 이 서비스는 Traceloop
SDK가 아니라 오픈텔레메트리(OpenTelemetry) SDK를 직접 사용한다. 두 개의 HTTP
클라이언트 계측 라이브러리가 활성화되어 있다.

```python
RequestsInstrumentor().instrument()
HTTPXClientInstrumentor().instrument()
```

에이전트로의 호출은 `requests`로 이루어지므로,
`opentelemetry-instrumentation-requests`가 이에 대한 클라이언트 스팬을 생성하고
나가는 요청에 트레이스 컨텍스트를 주입한다. 이것이 바로 채팅 인터페이스를
에이전트와, 나아가 에이전트가 만드는 LLM 및 도구 호출과 연결하는 지점이다.

Gradio 서버 자체는 계측되어 있지 않으므로, 들어오는 브라우저 요청은 서버 스팬을
생성하지 않는다. 이 서비스의 트레이스는 에이전트로의 아웃바운드 호출에서
시작된다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

트레이싱은 `run.py`가 임포트 시점에 호출하는 `_configure_tracing`에서 명시적으로
구성된다. 이 코드는 트레이서 프로바이더를 생성하고, OTLP 익스포터를 갖춘 배치
스팬 프로세서를 추가한 뒤, 계측 라이브러리가 이를 사용하도록 전역으로 등록한다.

```python
def _configure_tracing() -> None:
    provider = TracerProvider()
    provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
    trace.set_tracer_provider(provider)

    RequestsInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()
```

이 익스포터는 `opentelemetry.exporter.otlp.proto.http.trace_exporter`에서
임포트되므로, 이 서비스는 OTLP/HTTP로 내보낸다. Docker Compose는 이 서비스에
대해 `OTEL_EXPORTER_OTLP_ENDPOINT`를 오픈텔레메트리 컬렉터의 OTLP/HTTP 포트로
설정하는 반면, 대부분의 다른 데모 서비스는 gRPC로 내보낸다. 내보내기 엔드포인트,
리소스 속성, 서비스 이름은 모두 표준 오픈텔레메트리 환경 변수에서 가져온다.

### 새 스팬 생성 {#create-new-spans}

이 서비스는 자체적으로 스팬을 생성하지 않는다. 트레이서를 얻지 않으며,
`start_as_current_span`을 호출하지 않고, `set_attribute`를 사용해 스팬을
보강(enrich)하지도 않는다. 이 서비스의 모든 스팬은 `requests`와 HTTPX 계측
라이브러리에서 나온다.

## 메트릭 {#metrics}

미터 프로바이더는 구성되어 있지 않다. 이 서비스는 트레이스 익스포터만 임포트하며
`metrics.set_meter_provider`를 절대 호출하지 않으므로, `requests`와 HTTPX 계측
라이브러리가 발생시킬 수 있는 메트릭은 갈 곳이 없어 내보내지지 않는다.
[메트릭 커버리지 매트릭스](../../telemetry-features/metric-coverage/)를
참고한다.

## 로그 {#logs}

이 서비스는 Python 표준 라이브러리 로거만 구성한다.

```python
logging.basicConfig(level=logging.INFO)
```

에이전트로의 요청과 오류는 `chat_with_agent`에서 이를 통해 로그로 남는다.

```python
logging.info(f"Sending request {payload} to Agent")
```

`LoggerProvider`나 `LoggingHandler`가 설정되어 있지 않으므로, 이 레코드는 OTLP로
내보내지는 대신 표준 출력(stdout)으로 전달되어 컨테이너 런타임이 수집하며,
트레이스와 상관관계(correlation)를 갖지 않는다.
[로그 커버리지 매트릭스](../../telemetry-features/log-coverage/)를 참고한다.

전체 환경 변수 목록과 문제 해결 단계는
[서비스 README](https://github.com/open-telemetry/opentelemetry-demo/tree/main/src/chatbot#readme)를
참고한다.
