---
default_lang_commit: 58e684763e8dd50a07ef5fbf428303973025428a
---

텔레메트리가 올바르게 내보내지도록 [오픈텔레메트리 컬렉터](/docs/collector/)로
텔레메트리를 전송한다. 프로덕션 환경에서 컬렉터를 사용하는 것은 모범 사례이다.
텔레메트리를 시각화하려면 [Jaeger](https://jaegertracing.io/),
[Zipkin](https://zipkin.io/), [Prometheus](https://prometheus.io/) 또는
[벤더별](/ecosystem/vendors/) 백엔드와 같은 백엔드로 내보낸다.

{{ if $name }}

## 사용 가능한 익스포터 {#available-exporters}

레지스트리에는 [{{ $name }}용 익스포터 목록][reg]이 있다.

{{ end }}

{{ if not $name }}

레지스트리에는 [언어별 익스포터 목록][reg]이 있다.

{{ end }}

익스포터 중에서도 [오픈텔레메트리 프로토콜(OTLP)][OTLP] 익스포터는
오픈텔레메트리 데이터 모델을 염두에 두고 설계되어, 정보 손실 없이 OTel 데이터를
내보낸다. 또한 텔레메트리 데이터를 다루는 많은 도구가 OTLP를
지원하므로([Prometheus][], [Jaeger][], 그리고 대부분의 [벤더][vendors] 등),
필요할 때 높은 유연성을 제공한다. OTLP에 대한 자세한 내용은 [OTLP 명세][OTLP]를
참고한다.

[Jaeger]: /blog/2022/jaeger-native-otlp/
[OTLP]: /docs/specs/otlp/
[Prometheus]:
  https://prometheus.io/docs/prometheus/2.55/feature_flags/#otlp-receiver
[reg]: </ecosystem/registry/?component=exporter&language={{ $lang }}>
[vendors]: /ecosystem/vendors/

{{ if $name }}

이 페이지에서는 주요 오픈텔레메트리 {{ $name }} 익스포터와 그 설정 방법을
다룬다.

{{ end }}

{{ if $zeroConfigPageExists }}

> [!NOTE]
>
> [제로 코드 계측](</docs/zero-code/{{ $langIdAsPath }}>)을 사용한다면,
> [설정 가이드](</docs/zero-code/{{ $langIdAsPath }}/configuration/>)에 따라
> 익스포터를 설정하는 방법을 알아볼 수 있다.

{{ end }}

{{ if $supportsOTLP }}

## OTLP {#otlp}

### 컬렉터 설정 {#collector-setup}

> [!NOTE]
>
> OTLP 컬렉터나 백엔드가 이미 설정되어 있다면, 이 섹션을 건너뛰고 애플리케이션에
> 대한 [OTLP 익스포터 의존성을 설정](#otlp-dependencies)해도 된다.

OTLP 익스포터를 시험해 보고 검증하려면, 텔레메트리를 콘솔에 직접 기록하는 도커
컨테이너로 컬렉터를 실행할 수 있다.

빈 디렉터리에 다음 내용으로 `collector-config.yaml` 파일을 생성한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
    logs:
      receivers: [otlp]
      exporters: [debug]
```

이제 도커 컨테이너로 컬렉터를 실행한다.

```shell
docker run -p 4317:4317 -p 4318:4318 --rm -v $(pwd)/collector-config.yaml:/etc/otelcol/config.yaml otel/opentelemetry-collector
```

이제 이 컬렉터는 OTLP를 통해 텔레메트리를 수신할 수 있다. 이후에는 옵저버빌리티
백엔드로 텔레메트리를 전송하도록
[컬렉터를 설정](/docs/collector/configuration)할 수 있다.

{{ end }}
