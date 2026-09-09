---
title: Kafka
cSpell:ignore: Dotel
default_lang_commit: 36a8a53c4a20a6d7a706539e9f3b4887327be781
---

이는 체크아웃 서비스를 회계 서비스 및 사기 탐지 서비스와 연결하는 메시지 큐
서비스로 사용된다.

[Kafka 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/kafka/)

## 자동 계측 {#auto-instrumentation}

이 서비스는 오픈텔레메트리(OpenTelemetry) Java 에이전트와, 내장된
[JMX Metric Insight Module](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/jmx-metrics/javaagent)에
의존하여
[Kafka 브로커 메트릭](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/library/kafka-broker.md)을
수집하고 이를 OTLP를 통해 컬렉터로 전송한다.

이 에이전트는 `-javaagent` 커맨드 라인 인수를 통해 프로세스에 전달된다. 커맨드
라인 인수는 `Dockerfile`의 `KAFKA_OPTS`를 통해 추가된다.

```dockerfile
ENV KAFKA_OPTS="-javaagent:/tmp/opentelemetry-javaagent.jar -Dotel.jmx.target.system=kafka-broker"
```
