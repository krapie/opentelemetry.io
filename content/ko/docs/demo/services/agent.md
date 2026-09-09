---
title: 에이전트 서비스
linkTitle: 에이전트
cSpell:ignore: fastapi httpx langchain langgraph openai
default_lang_commit: ffef14de849130bdf9ecd9d4912e75f5a8afdbfd
---

이 서비스는 데모를 위한 AI 어시스턴트를 제공한다. 사용자 프롬프트를 받아
LangGraph ReAct 에이전트로 라우팅하고, 내장된 도구 또는
[MCP 서비스](../mcp/)에서 로드한 도구를 통해 상점의 API를 호출하는 FastAPI
엔드포인트를 노출한다.

[에이전트 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/agent/)

## LLM 구성 {#llm-configuration}

기본적으로 이 서비스는 실제 모델 없이도 데모가 동작하도록 기록된 LLM 응답을
재생(replay)한다. 실제 OpenAI 호환 LLM을 사용하려면 `.env.override` 파일에 다음
환경 변수를 채운다.

```text
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini
API_KEY=<replace with API key>
USE_VCR=False
```

## 계측 라이브러리 {#instrumentation-libraries}

이 서비스는 `opentelemetry-instrument` 래퍼를 통해 시작되지 않는다.
`Dockerfile`이 스크립트를 직접 실행하며, 계측은 코드에서 설정된다.

```dockerfile
CMD ["python", "run.py"]
```

`run.py`에서 [Traceloop SDK](https://www.traceloop.com/)가 오픈텔레메트리 SDK를
초기화하고, 생성형 AI(generative AI) 계측 라이브러리 번들을 활성화한다. 이후
HTTPX 계측이 명시적으로 활성화된다.

```python
Traceloop.init(
    app_name=os.getenv("OTEL_SERVICE_NAME", "agent"),
)

HTTPXClientInstrumentor().instrument()
```

FastAPI 계측은 애플리케이션 객체가 생성된 이후 `start_servers`에서 적용된다.

```python
FastAPIInstrumentor.instrument_app(agent.app)
```

이들을 함께 사용하면 수동으로 스팬을 생성하지 않고도 스팬이 생성된다.

- `opentelemetry-instrumentation-fastapi` — `POST /prompt` 요청에 대한 서버
  스팬이다.
- `opentelemetry-instrumentation-httpx` — LLM API와 상점 도구가 사용하는
  프론트엔드 API 모두에 대한 아웃바운드 호출의 클라이언트 스팬이다.
- Traceloop의 번들, 특히 `opentelemetry-instrumentation-langchain`,
  `opentelemetry-instrumentation-openai`, `opentelemetry-instrumentation-mcp` —
  LangChain 및 LangGraph 단계, LLM 호출, 그리고 `MCP_ENABLED=True`일 때의 MCP
  도구 호출에 대한 스팬이다.

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

`Traceloop.init()`은 배치 스팬 프로세서와 OTLP 익스포터를 사용하는 트레이서
프로바이더(tracer provider)를 생성하고, 이를 전역 트레이서 프로바이더로
등록한다. 따라서 위의 계측 라이브러리들은 단일한 내보내기 파이프라인을 공유한다.

내보내기 엔드포인트는 `OTEL_EXPORTER_OTLP_ENDPOINT`가 아닌
`TRACELOOP_BASE_URL`에서 가져오며, Traceloop이 여기에 `/v1/traces`를 추가한다.
Docker Compose에서는 이 값이 오픈텔레메트리 컬렉터의 OTLP/HTTP 포트를 가리킨다.
`app_name` 인수는 `service.name` 리소스 속성이 되며, 추가 리소스 속성은
`OTEL_RESOURCE_ATTRIBUTES`에서 읽어온다.

### 새 스팬 생성 {#create-new-spans}

`run_agent` 메서드는 Traceloop의 `@workflow` 데코레이터로 감싸져 있으며, 이
데코레이터는 전체 에이전트 실행에 대한 스팬을 시작한다. 계측 라이브러리가
생성하는 LLM 및 도구 스팬은 이 스팬의 자식이 된다.

```python
@workflow(name="astronomy_shop_agent_workflow")
async def run_agent(self, input_prompt, history: List[Dict] | None = None):
```

이는 `astronomy_shop_agent_workflow`라는 이름의 스팬을 생성한다. 하나의
프롬프트가 여러 차례의 추론과 도구 호출 턴을 유발할 수 있으므로, 이 스팬이
하나의 엔드투엔드(end-to-end) 에이전트 실행을 하나로 묶어주는 역할을 한다.

이 데코레이터 외에는 이 서비스가 오픈텔레메트리 트레이싱 API를 직접 사용하지
않는다. 즉, `start_as_current_span`으로 스팬을 생성하지도 않고,
`set_attribute`로 스팬에 정보를 추가(enrich)하지도 않는다.

### 프롬프트 및 완성 콘텐츠 {#prompt-and-completion-content}

번들로 제공되는 생성형 AI 계측은 오픈텔레메트리의
[생성형 AI 시맨틱 컨벤션](/docs/specs/semconv/gen-ai/)을 따르며, 프롬프트와
완성(completion) 내용을 스팬 속성 `gen_ai.input.messages` 및
`gen_ai.output.messages`로 기록한다. 내보내지는 스팬에서 프롬프트와 완성 내용을
제외하려면 `TRACELOOP_TRACE_CONTENT=false`로 설정한다.

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

`Traceloop.init()`은 `TRACELOOP_METRICS_ENABLED=false`로 설정하지 않는 한
메트릭도 구성한다. 이는 주기적으로 내보내는 메트릭 리더(metric reader)를
사용하는 미터 프로바이더(meter provider)를 생성하고 전역으로 등록하여, FastAPI
및 HTTPX 계측 라이브러리가 발생시키는 메트릭이 내보내지도록 한다.

### 커스텀 메트릭 {#custom-metrics}

이 서비스는 커스텀 메트릭을 정의하지 않는다. 자체적으로 미터(meter)를 가져오거나
계측기(instrument)를 생성하지 않는다.

## 로그 {#logs}

이 서비스는 파이썬 표준 라이브러리 로거만 구성한다.

```python
logging.basicConfig(level=logging.INFO)
```

Traceloop의 로그 내보내기는 기본적으로 비활성화되어 있으며, 이 서비스는
`LoggerProvider`나 `LoggingHandler`를 설정하지 않는다. 로그 레코드는 OTLP로
내보내지는 대신 표준 출력(stdout)에 기록되어 컨테이너 런타임이 수집하므로,
트레이스와 상관관계가 연결되지 않는다.
[로그 기능 매트릭스](../../telemetry-features/log-coverage/)를 참고한다.

환경 변수의 전체 목록과 문제 해결 방법은
[서비스 README](https://github.com/open-telemetry/opentelemetry-demo/tree/main/src/agent#readme)를
참고한다.
