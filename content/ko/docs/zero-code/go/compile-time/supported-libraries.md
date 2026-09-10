---
title: 지원되는 라이브러리
description: 기본적으로 계측되는 라이브러리 및 프레임워크.
weight: 10
# prettier-ignore
cSpell:ignore: anthropics gonic logrus openai runtimemetrics segmentio sirupsen
default_lang_commit: f796f16d521045c1a607f9e06daa162403c36d24
---

이 도구는 다음 라이브러리 및 프레임워크에 대한 계측 패키지를 제공한다.
애플리케이션이나 그 의존성이 이 중 하나를 사용하면, 빌드 타임에 해당 계측이
자동으로 주입된다.

| 라이브러리 또는 프레임워크 | 임포트 경로                              | 계측되는 오퍼레이션     |
| -------------------------- | ---------------------------------------- | ----------------------- |
| HTTP(표준 라이브러리)      | `net/http`                               | 클라이언트 및 서버 요청 |
| gRPC                       | `google.golang.org/grpc`                 | 클라이언트 및 서버 호출 |
| SQL 데이터베이스           | `database/sql`                           | 데이터베이스 호출       |
| Gin                        | `github.com/gin-gonic/gin`               | 서버 요청               |
| Redis                      | `github.com/redis/go-redis/v9`           | 클라이언트 명령         |
| MongoDB                    | `go.mongodb.org/mongo-driver/mongo`      | 클라이언트 명령         |
| Kafka                      | `github.com/segmentio/kafka-go`          | 생산 및 소비되는 메시지 |
| OpenAI                     | `github.com/openai/openai-go` (v1 – v3)  | 클라이언트 호출         |
| Anthropic                  | `github.com/anthropics/anthropic-sdk-go` | 클라이언트 호출         |
| Kubernetes 클라이언트      | `k8s.io/client-go/tools/cache`           | 인포머 캐시 오퍼레이션  |
| slog(표준 라이브러리)      | `log/slog`                               | 로그 레코드             |
| Logrus                     | `github.com/sirupsen/logrus`             | 로그 레코드             |

HTTP 및 gRPC 계측은 서비스 간 자동
[컨텍스트 전파](/docs/concepts/context-propagation/)를 포함하여 스팬과 메트릭을
생성한다. 계측은 각 라이브러리에 대해 오픈텔레메트리(OpenTelemetry)
[시맨틱 컨벤션](/docs/specs/semconv/)을 따른다. Go 런타임 메트릭은 기본적으로
수집되며, `OTEL_GO_DISABLED_INSTRUMENTATIONS`에 `runtimemetrics`를 추가하여 끌
수 있다.

지원되는 라이브러리 버전 집합은 각 계측의 규칙에 의해 선언된다. 신뢰할 수 있는
최신 목록은 저장소의
[계측 패키지](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/tree/main/instrumentation)를
참고한다.

## 라이브러리 요청하기 {#requesting-a-library}

의존하는 라이브러리가 아직 계측되지 않았다면
[기능 요청](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/issues)을
연다. 직접 계측을 추가할 수도 있다. 저장소의
[계측 가이드](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/blob/main/docs/instrument-guide.md)는
새 라이브러리에 대한 규칙 정의와 후크(hook) 구현을 단계별로 안내한다.
