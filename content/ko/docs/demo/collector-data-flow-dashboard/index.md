---
title: 컬렉터 데이터 흐름 대시보드
default_lang_commit: b98ab730de1f866d89a065fdac22b0ae123ec10c
---

오픈텔레메트리 컬렉터를 통과하는 데이터 흐름을 모니터링하는 것은 여러 이유로
매우 중요하다. 샘플 수와 카디널리티 같은 유입 데이터에 대한 매크로 수준의 관점을
확보하는 것은 컬렉터의 내부 동작을 이해하는 데 필수적이다. 하지만 세부적으로
파고들면 구성 요소 간의 연결이 복잡해질 수 있다. 컬렉터 데이터 흐름 대시보드는
오픈텔레메트리 데모 애플리케이션의 기능을 보여주어, 사용자가 이를 바탕으로
확장해 나갈 수 있는 견고한 토대를 제공하는 것을 목표로 한다. 컬렉터 데이터 흐름
대시보드는 어떤 메트릭을 모니터링해야 하는지에 대한 유용한 지침을 제공한다.
사용자는 memory_delimiter 프로세서나 다른 데이터 흐름 지표와 같이 자신의 사용
사례에 특화된 메트릭을 추가하여 자신만의 대시보드 변형을 만들 수 있다. 이 데모
대시보드는 시작점 역할을 하여, 사용자가 다양한 사용 시나리오를 탐색하고 자신의
고유한 모니터링 요구사항에 맞게 이 도구를 조정할 수 있도록 한다.

## 데이터 흐름 개요 {#data-flow-overview}

아래 다이어그램은 시스템 구성 요소에 대한 개요를 제공하며, 오픈텔레메트리 데모
애플리케이션이 사용하는 오픈텔레메트리 컬렉터(otelcol) 구성 파일에서 도출된
구성을 보여준다. 또한 시스템 내에서 옵저버빌리티 데이터(트레이스와 메트릭)가
흐르는 과정을 강조해서 보여준다.

![오픈텔레메트리 컬렉터 개요](otelcol-data-flow-overview.png)

## 유입/유출 메트릭 {#ingressegress-metrics}

아래 다이어그램에 표시된 메트릭은 유입 및 유출 데이터 흐름을 모두 모니터링하는
데 사용된다. 이 메트릭은 otelcol 프로세스에 의해 생성되어 8888 포트로
내보내지며, 이후 Prometheus가 스크레이핑한다. 이 메트릭과 연관된 네임스페이스는
"otelcol"이며, 작업(job) 이름은 `otel`로 표시된다.

![오픈텔레메트리 컬렉터 유입 및 유출 메트릭](otelcol-data-flow-metrics.png)

레이블(label)은 특정 메트릭 집합(예: exporter, receiver, job)을 식별하는 데
유용한 도구로 작용하여, 전체 네임스페이스 내에서 메트릭 집합을 구분할 수 있게
해준다. 메모리 딜리미터(memory delimiter) 프로세서에 정의된 메모리 제한을
초과했을 때만 거부된(refused) 메트릭이 나타난다는 점에 유의해야 한다.

### 유입 트레이스 파이프라인 {#ingress-traces-pipeline}

- `otelcol_receiver_accepted_spans`
- `otelcol_receiver_refused_spans`
- `by (receiver,transport)`

### 유입 메트릭 파이프라인 {#ingress-metrics-pipeline}

- `otelcol_receiver_accepted_metric_points`
- `otelcol_receiver_refused_metric_points`
- `by (receiver,transport)`

### 프로세서 {#processor}

현재 데모 애플리케이션에 존재하는 유일한 프로세서는 배치(batch) 프로세서이며,
이는 트레이스와 메트릭 파이프라인 모두에서 사용된다.

- `otelcol_processor_batch_batch_send_size_sum`

### 유출 트레이스 파이프라인 {#egress-traces-pipeline}

- `otelcol_exporter_sent_spans`
- `otelcol_exporter_send_failed_spans`
- `by (exporter)`

### 유출 메트릭 파이프라인 {#egress-metrics-pipeline}

- `otelcol_exporter_sent_metric_points`
- `otelcol_exporter_send_failed_metric_points`
- `by (exporter)`

### Prometheus 스크레이핑 {#prometheus-scraping}

- `scrape_samples_scraped`
- `by (job)`

## 대시보드 {#dashboard}

화면 왼쪽의 browse 아이콘 아래에서 **OpenTelemetry Collector** 대시보드를
선택하여 Grafana UI로 이동하면 대시보드에 접근할 수 있다.

![오픈텔레메트리 컬렉터 대시보드](otelcol-data-flow-dashboard.png)

대시보드는 네 개의 주요 섹션으로 구성된다.

1. 프로세스 메트릭
2. 트레이스 파이프라인
3. 메트릭 파이프라인
4. Prometheus 스크레이핑

2, 3, 4번 섹션은 위에서 언급한 메트릭을 사용하여 전체 데이터 흐름을 나타낸다.
또한 데이터 흐름을 이해하기 위해 각 파이프라인에 대한 내보내기 비율(export
ratio)이 계산된다.

### 내보내기 비율 {#export-ratio}

내보내기 비율은 기본적으로 리시버와 익스포터 메트릭 간의 비율이다. 위의 대시보드
스크린샷에서 메트릭에 대한 내보내기 비율이 수신된 메트릭보다 훨씬 높다는 것을
확인할 수 있다. 이는 데모 애플리케이션이 스팬 메트릭(span metrics)을 생성하도록
구성되어 있기 때문이며, 이는 개요 다이어그램에서 볼 수 있듯이 컬렉터 내부에서
스팬으로부터 메트릭을 생성하는 프로세서이다.

### 프로세스 메트릭 {#process-metrics}

매우 제한적이지만 유용한 프로세스 메트릭이 대시보드에 추가되어 있다. 예를 들어
재시작 등의 상황에서 시스템에 둘 이상의 otelcol 인스턴스가 실행 중인 것을 관찰할
수도 있다. 이는 데이터 흐름의 급증을 이해하는 데 유용할 수 있다.

![오픈텔레메트리 컬렉터 프로세스 메트릭](otelcol-dashboard-process-metrics.png)
