---
title: Docker 배포
linkTitle: Docker
aliases: [docker_deployment]
cSpell:ignore: Firepit otlphttp span_metrics
default_lang_commit: d9d900a1c06ef7e2289f86887b68fdf6c150c52f
---

<!-- markdownlint-disable code-block-style ol-prefix -->

## 사전 요구 사항 {#prerequisites}

- Docker
- [Docker Compose](https://docs.docker.com/compose/install/) v2.0.0+
- Make(선택 사항)
- 애플리케이션 실행을 위한 RAM 6GB(또는 [최소 모드](#deployment-modes) 사용 시
  약 3GB)
- 디스크 공간 14GB

## 데모 받기 및 실행 {#get-and-run-the-demo}

1.  데모 저장소(repository)를 클론(clone)한다:

    ```shell
    git clone https://github.com/open-telemetry/opentelemetry-demo.git
    ```

2.  데모 폴더로 이동한다:

    ```shell
    cd opentelemetry-demo/
    ```

3.  데모를 시작한다[^1]:

    {{< tabpane text=true >}} {{% tab Make %}}

```shell
make start
```

    {{% /tab %}} {{% tab Docker %}}

```shell
docker compose --env-file .env --env-file .env.override \
  -f compose.yaml -f compose.full.yaml \
  -f compose.observability.yaml -f compose.extras.yaml \
  up --force-recreate --remove-orphans --detach
```

    {{% /tab %}} {{< /tabpane >}}

    ### 배포 모드 {#deployment-modes}

    데모는 여러 배포 모드를 지원한다. 기본값인 `make start`는 모든 서비스와
    옵저버빌리티(observability) 스택을 포함한 전체 데모를 실행한다. 다른
    모드를 사용하면 리소스 사용량을 줄이거나 특정 구성 요소를 제외할 수 있다:

    | 모드 | Make 타겟 | 설명 |
    | --- | --- | --- |
    | 전체 | `make start` | 모든 서비스와 옵저버빌리티 백엔드(기본값) |
    | 최소 | `make start-minimal` | Kafka와 이에 종속된 서비스(`accounting`, `fraud-detection`, `kafka`)를 제외하여 메모리 사용량을 약 3GB로 줄인다 |
    | 옵저버빌리티 없음 | `make start-no-o11y` | 옵저버빌리티 백엔드(Jaeger, Grafana, Prometheus, OpenSearch) 없이 모든 서비스를 실행한다 |
    | 최소, 옵저버빌리티 없음 | `make start-minimal-no-o11y` | 옵저버빌리티 백엔드 없이 최소 서비스만 실행한다 |
    | 프로파일링 | `make start-profiling` | 프로파일링 데이터를 위한 eBPF 프로파일러와 [Firepit](https://github.com/florianl/firepit) UI를 포함한 전체 모드 |
    | 에이전틱 | `make start-agentic` | 데모와 상호작용할 수 있는 AI 에이전트, MCP 서버, 챗봇을 포함한 전체 모드 |

    예를 들어, 최소 모드로 데모를 시작하려면 다음과 같이 실행한다:

    {{< tabpane text=true >}} {{% tab Make %}}

```shell
make start-minimal
```

    {{% /tab %}} {{% tab Docker %}}

```shell
docker compose --env-file .env --env-file .env.override \
  -f compose.yaml -f compose.observability.yaml -f compose.extras.yaml \
  up --force-recreate --remove-orphans --detach
```

    {{% /tab %}} {{< /tabpane >}}

4. (선택 사항) 텔레메트리 정합성 테스트(telemetry sanity tests)를 실행한다:

    데모에는 각 서비스가 트레이스, 메트릭, 로그를 생성하고 있는지, 그리고
    이러한 데이터가 예상되는 백엔드(Jaeger, Prometheus, OpenSearch)에
    도달하는지를 검증하는 텔레메트리 정합성 테스트 모음이 포함되어 있다.
    자세한 내용은
    [test/telemetry/README.md](https://github.com/open-telemetry/opentelemetry-demo/blob/main/test/telemetry/README.md)를
    참고한다.

    | 테스트 범위 | Make 타겟 | 실행 대상 |
    | --- | --- | --- |
    | 전체 | `make run-telemetry-tests` | 전체 배포(`make start`) |
    | 최소 | `make run-telemetry-tests-minimal` | 최소 배포(`make start-minimal`) |
    | 에이전틱 | `make run-telemetry-tests-agentic` | 에이전틱 배포(에이전트, MCP, 챗봇 포함) |

    각 타겟은 `./test/telemetry`에서 테스트 이미지를 빌드하고, 해당 배포를
    시작하며, 테스트를 실행한 다음 데모를 종료한다.

    {{< tabpane text=true >}} {{% tab Make %}}

```shell
make run-telemetry-tests
```

    {{% /tab %}} {{% tab Docker %}}

```shell
# The demo must be running before you start the tests.
docker build -t opentelemetry-demo-telemetry-tests ./test/telemetry
docker run --rm --network opentelemetry-demo \
  --env-file .env --env-file .env.override \
  -e TEST_SCOPE=full \
  opentelemetry-demo-telemetry-tests
```

    {{% /tab %}} {{< /tabpane >}}

## 웹 스토어 및 텔레메트리 확인 {#verify-the-web-store-and-telemetry}

이미지가 빌드되고 컨테이너가 시작되면 다음에 접속할 수 있다:

- 웹 스토어: <http://localhost:8080/>
- Flagd 구성 도구 UI: <http://localhost:8080/feature>
- 텔레메트리 문서(Weaver로 생성됨): <http://localhost:8080/telemetry/>

옵저버빌리티 스택이 실행 중일 때(즉, `*-no-o11y` 모드가 아닐 때) 다음을 사용할
수 있다:

- Grafana: <http://localhost:8080/grafana/>
- Jaeger UI: <http://localhost:8080/jaeger/ui/>
- OpAMP UI: <http://localhost:8080/opamp/>

다음은 특정 배포 모드에서만 사용할 수 있다:

- Firepit UI(프로파일링 모드): <http://localhost:8080/profiles/>
- 챗봇(에이전틱 모드): <http://localhost:8080/chatbot/>

## 데모의 기본 포트 번호 변경 {#changing-the-demos-primary-port-number}

기본적으로 데모 애플리케이션은 포트 8080에 바인딩된 모든 브라우저 트래픽을 위한
프록시를 시작한다. 포트 번호를 변경하려면 데모를 시작하기 전에 `ENVOY_PORT` 환경
변수를 설정한다.

- 예를 들어, 포트 8081을 사용하려면[^1]:

  {{< tabpane text=true >}} {{% tab Make %}}

```shell
ENVOY_PORT=8081 make start
```

    {{% /tab %}} {{% tab Docker %}}

```shell
ENVOY_PORT=8081 docker compose --env-file .env --env-file .env.override \
  -f compose.yaml -f compose.full.yaml \
  -f compose.observability.yaml -f compose.extras.yaml \
  up --force-recreate --remove-orphans --detach
```

    {{% /tab %}} {{< /tabpane >}}

## 자체 백엔드 사용하기 {#bring-your-own-backend}

이미 보유하고 있는 옵저버빌리티 백엔드(예: 기존 Jaeger나 Zipkin 인스턴스, 또는
원하는 [벤더](/ecosystem/vendors/) 중 하나)를 위한 데모 애플리케이션으로 웹
스토어를 사용하고 싶을 수 있다.

오픈텔레메트리 컬렉터(OpenTelemetry Collector)를 사용하여 텔레메트리 데이터를
여러 백엔드로 내보낼 수 있다. 기본적으로 데모 애플리케이션의 컬렉터는 다음
파일의 구성을 순서대로 병합한다:

- `otelcol-config.yml` — 기본 리시버, 프로세서, 파이프라인
- `otelcol-config-full.yml` — Kafka 및 PostgreSQL 메트릭 리시버(전체 모드)
- `otelcol-config-observability.yml` — Jaeger, Prometheus, OpenSearch
  익스포터(옵저버빌리티 스택 사용 시)
- `otelcol-config-extras.yml` — 커스터마이징을 위한 빈 스텁으로, 항상 마지막에
  로드된다

백엔드를 추가하려면 편집기로
[src/otel-collector/otelcol-config-extras.yml](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/otel-collector/otelcol-config-extras.yml)
파일을 연다.

- 새 익스포터를 추가하는 것부터 시작한다. 예를 들어 백엔드가 HTTP를 통한 OTLP를
  지원하는 경우 다음을 추가한다:

  ```yaml
  exporters:
    otlphttp/example:
      endpoint: <your-endpoint-url>
  ```

- 그런 다음 백엔드에 사용하려는 텔레메트리 파이프라인의 `exporters`를
  재정의한다.

  ```yaml
  service:
    pipelines:
      traces:
        exporters: [debug, otlp_grpc/jaeger, span_metrics, otlphttp/example]
  ```

> [!NOTE]
>
> 컬렉터에서 YAML 값을 병합할 때 객체는 병합되고 배열은 교체된다. `span_metrics`
> 커넥터(connector)는 트레이스를 메트릭에 연결하므로, 해당 파이프라인을 재정의할
> 때 traces 파이프라인의 `exporters`와 metrics 파이프라인의 `receivers`에 반드시
> 유지해야 한다 — 이를 생략하면 컬렉터가 다운된다. 그 외의 익스포터는 모두 선택
> 사항이며, 생략하면 해당 백엔드로 데이터가 전송되지 않을 뿐이다. 업스트림
> 익스포터 이름은 다음과 같다:
>
> - **traces**: `debug`, `otlp_grpc/jaeger`, `span_metrics` _(필수)_
> - **metrics**: `debug`, `otlp_http/prometheus`
> - **logs**: `debug`, `opensearch`

벤더 백엔드는 인증(authentication)을 위한 추가 파라미터를 요구할 수 있으므로
해당 문서를 확인한다. 일부 백엔드는 다른 익스포터를 필요로 하며, 이러한
익스포터와 관련 문서는
[opentelemetry-collector-contrib/exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter)에서
찾을 수 있다.

`otelcol-config-extras.yml`을 업데이트한 후 `make start`를 실행하여 데모를
시작한다. 잠시 후 백엔드로도 트레이스가 유입되는 것을 확인할 수 있다.

[^1]: {{% param notes.docker-compose-v2 %}}
