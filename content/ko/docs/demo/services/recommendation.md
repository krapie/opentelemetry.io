---
title: 추천 서비스
linkTitle: 추천
aliases: [recommendationservice]
cSpell:ignore: cpython instrumentor NOTSET
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

이 서비스는 사용자가 조회 중인 기존 상품 ID를 기반으로 사용자에게 추천할 상품
목록을 얻어오는 역할을 담당한다.

[추천 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/recommendation/)

## 자동 계측 {#auto-instrumentation}

Python 기반인 이 서비스는 `opentelemetry-instrument` Python 래퍼를 활용해
스크립트를 실행함으로써 Python용 오픈텔레메트리(OpenTelemetry) 자동 계측기를
사용한다. 이는 서비스 `Dockerfile`의 `ENTRYPOINT` 명령에서 수행할 수 있다.

```dockerfile
ENTRYPOINT [ "opentelemetry-instrument", "python", "recommendation_server.py" ]
```

## 트레이스 {#traces}

### 트레이싱 초기화 {#initializing-tracing}

오픈텔레메트리 SDK는 `__main__` 코드 블록에서 초기화된다. 이 코드는 트레이서
프로바이더를 생성하고, 사용할 스팬 프로세서를 설정한다. 내보내기 엔드포인트,
리소스 속성, 서비스 이름은 환경 변수를 기반으로 오픈텔레메트리 자동 계측기가
자동으로 설정한다.

```python
tracer = trace.get_tracer_provider().get_tracer("recommendation")
```

### 자동 계측된 스팬에 속성 추가 {#add-attributes-to-auto-instrumented-spans}

자동으로 계측된 코드가 실행되는 동안 컨텍스트에서 현재 스팬을 가져올 수 있다.

```python
span = trace.get_current_span()
```

스팬에 속성을 추가하려면 스팬 객체의 `set_attribute`를 사용한다.
`ListRecommendations` 함수에서는 스팬에 속성이 추가된다.

```python
span.set_attribute("app.products_recommended.count", len(prod_list))
```

### 새 스팬 생성 {#create-new-spans}

새로운 스팬은 오픈텔레메트리 `Tracer` 객체의 `start_as_current_span`을 사용해
생성하고 활성 컨텍스트에 배치할 수 있다. 이를 `with` 블록과 함께 사용하면,
블록의 실행이 끝날 때 스팬이 자동으로 종료된다. 이는 `get_product_list` 함수에서
수행된다.

```python
with tracer.start_as_current_span("get_product_list") as span:
```

## 메트릭 {#metrics}

### 메트릭 초기화 {#initializing-metrics}

오픈텔레메트리 SDK는 `__main__` 코드 블록에서 초기화된다. 이 코드는 미터
프로바이더를 생성한다. 내보내기 엔드포인트, 리소스 속성, 서비스 이름은 환경
변수를 기반으로 오픈텔레메트리 자동 계측기가 자동으로 설정한다.

```python
meter = metrics.get_meter_provider().get_meter("recommendation")
```

### 커스텀 메트릭 {#custom-metrics}

현재 사용 가능한 커스텀 메트릭은 다음과 같다.

- `app_recommendations_counter`: 서비스 호출당 추천된 상품 수의 누적 카운트

### 자동 계측된 메트릭 {#auto-instrumented-metrics}

`opentelemetry-instrumentation-system-metrics` 덕분에 자동 계측을 통해 다음
메트릭을 사용할 수 있다. 이 라이브러리는 추천 서비스 Docker 이미지를 빌드할 때
`opentelemetry-bootstrap`의 일부로 설치된다.

- `runtime.cpython.cpu_time`
- `runtime.cpython.memory`
- `runtime.cpython.gc_count`

## 로그 {#logs}

### 로그 초기화 {#initializing-logs}

오픈텔레메트리 SDK는 `__main__` 코드 블록에서 초기화된다. 다음 코드는 배치
프로세서, OTLP 로그 익스포터, 로깅 핸들러를 갖춘 로거 프로바이더를 생성한다.
마지막으로 애플리케이션 전반에서 사용할 로거를 생성한다.

```python
logger_provider = LoggerProvider(
    resource=Resource.create(
        {
            'service.name': service_name,
        }
    ),
)
set_logger_provider(logger_provider)
log_exporter = OTLPLogExporter(insecure=True)
logger_provider.add_log_record_processor(BatchLogRecordProcessor(log_exporter))
handler = LoggingHandler(level=logging.NOTSET, logger_provider=logger_provider)

logger = logging.getLogger('main')
logger.addHandler(handler)
```

### 로그 레코드 생성 {#create-log-records}

로거를 사용해 로그를 생성한다. 예시는 `ListRecommendations` 함수와
`get_product_list` 함수에서 찾을 수 있다.

```python
logger.info(f"Receive ListRecommendations for product ids:{prod_list}")
```

보다시피, 초기화 이후에는 표준 Python과 동일한 방식으로 로그 레코드를 생성할 수
있다. 오픈텔레메트리 라이브러리는 각 로그 레코드에 트레이스 ID와 스팬 ID를
자동으로 추가하며, 이를 통해 로그와 트레이스를 연관시킬(correlate) 수 있다.

### 참고 사항 {#notes}

Python용 로그는 아직 실험적(experimental) 단계이며, 일부 변경이 있을 수 있다. 이
서비스의 구현은
[Python 로그 예제](https://github.com/open-telemetry/opentelemetry-python/blob/stable/docs/examples/logs/example.py)를
따른다.
