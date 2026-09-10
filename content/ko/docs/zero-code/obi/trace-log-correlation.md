---
title: 트레이스-로그 상관관계
linkTitle: 트레이스-로그 상관관계
weight: 35
description:
  OBI가 더 빠른 디버깅과 문제 해결을 위해 애플리케이션 로그를 분산 트레이스와
  상관 짓는 방법을 알아본다.
cSpell:ignore: BPFFS NUL PYTHONUNBUFFERED ringbuffer
default_lang_commit: 2021ec6e35d03a3f5f99da13b908091068154a44
---

오픈텔레메트리 eBPF 계측(OpenTelemetry eBPF Instrumentation, OBI)은 JSON 및 일반
텍스트 로그를 트레이스 컨텍스트로 보강(enrichment)하여 애플리케이션 로그를 분산
트레이스와 상관관계(correlation)로 연결한다. OBI는 로그를 내보내지 않는다.
보강된 로그를 동일한 스트림에 다시 기록하며, 트레이스는 OTLP를 통해 내보낸다.

## 개요 {#overview}

트레이스-로그 상관관계는 상호 보완적인 두 옵저버빌리티(observability) 시그널을
연결한다.

- **트레이스**: 타이밍과 구조와 함께 서비스 전반의 요청 흐름을 보여준다
- **로그**: 상세한 이벤트 정보와 애플리케이션 상태를 제공한다

OBI 트레이스-로그 상관관계를 사용하면, 계측된 프로세스의 로그가 트레이스
컨텍스트로 자동 보강된다.

- **트레이스 ID(Trace ID)**: 로그 항목을 분산 트레이스에 연결한다
- **스팬 ID(Span ID)**: 로그 항목을 특정 트레이스 스팬에 연결한다

이를 통해 옵저버빌리티 백엔드는 애플리케이션 코드 변경 없이 로그를 그 로그가
발생한 트레이스와 상관 지을 수 있다.

## 동작 방식 {#how-it-works}

OBI는 eBPF를 사용하여 커널 수준에서 애플리케이션 로그에 트레이스 컨텍스트를
주입한다.

1. **트레이스 캡처**: OBI는 트레이싱된 모든 작업에 대해 트레이스
   컨텍스트(트레이스 ID와 스팬 ID)를 캡처한다
2. **로그 가로채기**: OBI는 write 시스템 콜을 가로채 애플리케이션 로그를
   캡처한다
3. **컨텍스트 주입**: OBI는 JSON 객체에 `trace_id`와 `span_id` 필드를
   주입하거나, 선택된 일반 텍스트 줄에 구성 가능한 `key=value` 필드를 추가한다
4. **트레이스 내보내기**: 로그는 기존 로깅 파이프라인을 통해 계속 흐른다
5. **백엔드 연결**: 옵저버빌리티 백엔드는 이 ID를 사용하여 로그를 트레이스에
   연결한다

### 기술적 접근 방식 {#technical-approach}

OBI는 애플리케이션 바이너리를 수정하지 않고 커널 수준에서 상관관계를 수행한다.

- 커널 eBPF 프로브를 사용하여 쓰기 작업을 가로챈다
- 성능을 위해 파일 디스크립터 캐싱을 유지한다
- JSON 또는 일반 텍스트를 기록하는 로깅 프레임워크와 함께 동작한다

OBI는 이미 존재하는, 구성된 트레이스 및 스팬 필드를 보존한다. JSON 키는 문자
그대로 일치시킨다. 일반 텍스트에서 OBI는 줄의 시작이나 공백 뒤의 `name=value`
토큰을 인식한다. 오픈텔레메트리 트레이스를 직접 내보내는 것으로 감지된 서비스의
경우, OBI는 `trace_id`만 주입한다. OBI의 eBPF가 생성한 스팬 ID로는 SDK 스팬을
식별할 수 없기 때문이다.

## 구성 {#configuration}

로그를 트레이스와 상관 지으려면, 트레이스를 내보내고 선택된 워크로드의 로그에
트레이스 컨텍스트를 추가하도록 OBI를 구성한다. 구성 필드는 Config v1과 Config v2
간에 다르다. Config v2를 사용한다면 [Config v2 참조](../configure/config-v2/)를
참고한다. 기존 Config v1 파일을 변환하려면
[마이그레이션 가이드](../configure/migrate-to-config-v2/)를 따른다.

### Config v1 {#config-v1}

```yaml
# Enable trace export
otel_traces_export:
  endpoint: http://otel-collector:4318/v1/traces

# Select services to instrument
discovery:
  instrument:
    - open_ports: '8380'

# Enable log enrichment for the same services
ebpf:
  log_enricher:
    services:
      - service:
          - open_ports: '8380'
```

로그 보강 동작은 `ebpf.log_enricher` 아래에서 추가로 구성할 수 있다.

- `cache_ttl`: 캐시된 파일 디스크립터의 TTL(time-to-live)
- `cache_size`: 캐시된 파일 디스크립터의 최대 개수
- `async_writer_workers`: 비동기 라이터(writer) 샤드 수
- `async_writer_channel_len`: 샤드당 큐 크기
- `field_names`: 트레이스 및 스팬 ID를 인식하고 주입하는 데 사용되는 필드 이름
- `plain_text.enabled`: JSON이 아닌 로그에 주석을 달지 여부. 기본값은 `true`
- `plain_text.placement`: 필드를 `prefix` 또는 `suffix`로 추가
- `plain_text.multiline`: 가로챈 각 쓰기에서 `first_line`, `last_line`, 또는
  `each_line`에 주석을 단다

예를 들면 다음과 같다.

```yaml
ebpf:
  log_enricher:
    field_names:
      trace_id: trace_id
      span_id: span_id
    plain_text:
      enabled: true
      placement: suffix
      multiline: first_line
```

v0.11.0에서는 선택된 서비스에 대해 일반 텍스트 보강이 기본적으로 활성화된다.
JSON이 아닌 쓰기가 이전의 통과(pass-through) 동작을 유지해야 한다면, 업그레이드
전에 `plain_text.enabled: false`로 설정한다. 필드 이름은 JSON 및 일반 텍스트
출력에 적용되며, 비어 있지 않고 고유해야 하며 공백, `=`, 제어 문자를 포함하지
않아야 한다.

#### 서비스 선택 {#service-selection}

OBI는 `ebpf.log_enricher.services` 아래에 나열된 서비스에 대해 JSON 및 일반
텍스트 로그를 보강한다. 보강이 동일한 프로세스를 추적하도록 서비스 선택기를
`discovery.instrument`와 일치시킨다.

### Config v2 {#config-v2}

Config v2에서는 OBI를 독립형(standalone) 프로세스로 실행할 때만 로그 트레이스
주석을 사용할 수 있다. 기본적으로 비활성화되어 있다.
`extensions.obi.correlation.log_trace_annotation` 아래에서 구성한다.

```yaml
extensions:
  obi:
    correlation:
      log_trace_annotation:
        enabled: true
        field_names:
          trace_id: trace_id
          span_id: span_id
        plain_text:
          enabled: true
          placement: suffix
          multiline: first_line
```

Config v2 캡처 선택은 어떤 워크로드가 로그 주석 대상이 되는지 결정한다.
`log_trace_annotation.filter` 필드는 v0.11.0에서 예약되어 있으며 비어 있는
상태로 유지해야 한다.

## 요구 사항 {#requirements}

### 1. 지원되는 로그 형식 {#1-supported-log-format}

JSON 형식의 로그의 경우, OBI는 JSON 객체에 `trace_id`와 `span_id` 필드를
주입한다.

**OBI 적용 전**:

```json
{ "level": "info", "message": "Request processed", "duration_ms": 125 }
```

**OBI 보강 후**:

```json
{
  "level": "info",
  "message": "Request processed",
  "duration_ms": 125,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

일반 텍스트 로그의 경우, OBI는 소문자 고정 폭 ID를 공백으로 구분된 `key=value`
필드로 추가한다. 배치와 멀티라인 선택은 구성할 수 있다.

```text
request processed trace_id=4bf92f3577b34da6a3ce929d0e0e4736 span_id=00f067aa0ba902b7
```

줄바꿈으로 구분된 JSON(NDJSON)은 구조화된 JSON으로 처리된다. OBI는 각 객체
레코드를 독립적으로 보강하며 유효한 NDJSON에는 일반 텍스트 주석을 적용하지
않는다. 멀티라인 선택은 하나의 가로챈 쓰기 내에서 비어 있지 않은 물리적 줄에
대해 동작한다. OBI는 별도의 쓰기에 걸친 논리적 이벤트를 재구성하지 않는다.

#### 런타임 버퍼링 제한 사항 {#runtime-buffering-limitations}

로그 보강기(enricher)는 로그 쓰기가 요청 처리 스레드에서 발생할 때만 트레이스
컨텍스트를 볼 수 있다. stdout을 비동기적으로 버퍼링하는 런타임은 이 가정을
깨뜨릴 수 있다.

- Docker의 Python은 일반적으로 `PYTHONUNBUFFERED=1`이 필요하다
- stdout이 파이프일 때 .NET `Console.Out`은 기본적으로 버퍼링된다.
  `AutoFlush = true`인 `StreamWriter`를 사용한다
- ASP.NET Core의 기본 `Microsoft.Extensions.Logging.AddConsole()` 파이프라인은
  백그라운드 스레드에서 기록하므로 호환되지 않는다
- 캐리어 커널 스레드가 여러 가상 스레드의 작업을 실행할 수 있으므로 Java 가상
  스레드 로그는 보강되지 않는다. 플랫폼 스레드 보강은 영향을 받지 않는다

### 2. 트레이스 내보내기 및 로그 보강 활성화 {#2-trace-export-and-log-enrichment-enabled}

트레이스-로그 상관관계에는 트레이스 내보내기와 로그 보강이 모두 필요하다. Config
v1의 경우:

```yaml
otel_traces_export:
  endpoint: http://collector:4318/v1/traces # Required

ebpf:
  log_enricher:
    services:
      - service:
          - open_ports: '8380' # Required
```

### 3. Linux 커널 {#3-linux-kernel}

트레이스-로그 상관관계에는 특정 커널 기능이 있는 Linux가 필요하다.

- **Linux 커널 6.0+**(트레이스-로그 상관관계에 필요)
- 지원 아키텍처: x86_64, ARM64
- **BPFFS 마운트**: 커널에 BPF 파일시스템이 `/sys/fs/bpf`에 마운트되어 있어야
  한다
- **보안 lockdown이 아닌 커널**: 보안 lockdown 모드로 실행되지 않는 커널이
  필요하다(대부분의 프로덕션 배포판에서 일반적임)

### 4. 지원되는 로그를 내보내는 프레임워크 {#4-framework-that-emits-supported-logs}

애플리케이션은 JSON 또는 일반 텍스트를 출력하도록 구성된 로깅 프레임워크를
사용할 수 있다. 다음 JSON 예시는 구조화된 필드를 생성한다.

{{< tabpane text=true persist=lang >}} {{% tab header="Python" lang=python %}}

```python
import json
import logging

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
        }
        return json.dumps(log_entry)

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
```

{{% /tab %}} {{% tab header="Go (zap 사용)" lang=go %}}

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction() // Outputs JSON by default
defer logger.Sync()
logger.Info("Request processed", zap.Duration("duration", 125*time.Millisecond))
```

{{% /tab %}} {{% tab header="Java (Logback 사용)" lang=java %}}

```xml
<appender name="FILE" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

{{% /tab %}} {{% tab header="Node.js (pino 사용)" lang=javascript %}}

```javascript
const pino = require('pino');
const logger = pino();
logger.info({ duration_ms: 125 }, 'Request processed');
```

{{% /tab %}} {{< /tabpane >}}

### 5. 로그 전달 파이프라인 {#5-log-shipping-pipeline}

OBI는 로그를 제자리에서 보강한다. 기존 로그 포워더나 컬렉터를 사용하여 로그를
백엔드로 전달한다.

OBI가 원본 줄을 억제하면, 컨테이너 로그 파일에는 그 자리에 NUL 바이트로 이루어진
줄이 포함된다. 최대 8KiB 쓰기의 경우, 다운스트림에서 `^[\x00\s]*$`로 이러한
플레이스홀더 줄을 필터링한다. 예를 들어 오픈텔레메트리 컬렉터 `filelog` 리시버를
사용하는 경우:

```yaml
receivers:
  filelog:
    include:
      - /var/log/pods/*/*/*.log
    start_at: end
    operators:
      - type: container
      - type: filter
        expr: 'body matches "^[\\x00\\s]*$"'
```

CRI 및 Docker JSON 로그 엔벨로프는 NUL을 `\u0000`으로 인코딩한다. `container`
연산자는 필터가 실행되기 전에 본문을 디코딩한다.

## 성능 고려 사항 {#performance-considerations}

- **최소한의 오버헤드**: 상관관계는 효율적인 파일 디스크립터 캐싱과 함께 eBPF
  커널 프로브를 사용한다
- **캐시 제한**: 파일 디스크립터 캐시에는 무제한 메모리 사용을 방지하기 위한
  크기 및 TTL 제한이 있다
- **비동기 처리**: 로그 보강은 커널 링버퍼(ringbuffer) 오버플로를 방지하기 위해
  비동기 워커를 사용한다

## 알려진 제한 사항 {#known-limitations}

- **쓰기별 멀티라인 선택**: OBI는 별도의 쓰기에 걸친 논리적 멀티라인 이벤트를
  재구성하지 않는다
- **파일 디스크립터 캐시**: 성능을 위해 캐시되며, 구성 가능한 TTL(기본값:
  30분)을 가진다
- **스팬 정렬만 지원**: 로그는 스팬이 활성 상태인 동안에만 보강된다. 스팬 범위
  밖의 로그는 보강되지 않는다.
- **쓰기당 8KiB 제한**: OBI는 단일 `write()` 또는 `writev()`의 최대 처음 8KiB만
  보강하고 억제한다. 남은 바이트는 보강되지 않은 채 통과하며 플레이스홀더 줄
  필터와 일치하지 않는다.
- **Java 가상 스레드**: 가상 스레드에서 기록된 로그는 보강되지 않는다.

## 문제 해결 {#troubleshooting}

### 로그에 트레이스 컨텍스트가 나타나지 않음 {#trace-context-not-appearing-in-logs}

1. **구성된 형식 확인**: JSON 로그의 경우, 애플리케이션이 유효한 JSON을
   출력하는지 확인한다. 일반 텍스트의 경우, `plain_text.enabled`가 `true`인지
   확인하고 배치 및 멀티라인 설정을 검사한다.

   ```bash
   # Check for malformed JSON
   cat app.log | jq empty && echo "Valid JSON" || echo "Invalid JSON"
   ```

2. **트레이스 내보내기 및 로그 보강 확인**:

   ```yaml
   otel_traces_export:
     endpoint: http://collector:4318/v1/traces

   ebpf:
     log_enricher:
       services:
         - service:
             - open_ports: '8380'
   ```

3. **Linux 커널 확인**: 트레이스-로그 상관관계에는 Linux가 필요하다

   ```bash
   uname -s  # Must return "Linux"
   ```

4. **로그 파이프라인 확인**: 로그 포워더가 로그를 백엔드로 전달하고 있는지
   확인한다

## 다음 단계 {#whats-next}

- 트레이스와 메트릭을 위한
  [내보내기 대상](/docs/zero-code/obi/configure/export-data/)을 설정한다
- 중앙 집중식 처리를 위해
  [컬렉터 리시버로서의](/docs/zero-code/obi/configure/collector-receiver/) OBI를
  살펴본다
