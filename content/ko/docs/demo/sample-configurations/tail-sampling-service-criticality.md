---
title: service.criticality를 사용한 테일 기반 샘플링(Tail-Based Sampling)
linkTitle: 테일 샘플링
default_lang_commit: 8387f1584794f10784684cff32d0a4b8769ce50b
---

이 예시는 오픈텔레메트리(OpenTelemetry) 컬렉터에서 지능적인 테일 기반
샘플링(tail-based sampling) 결정을 내리기 위해
[`service.criticality`](/docs/specs/semconv/resource/service/#service) 리소스
속성을 사용하는 방법을 보여준다.

데모 애플리케이션은 각 서비스에 `service.criticality` 값을 할당하여, 운영상
중요도에 따라 서비스를 분류한다.

| 중요도     | 샘플링 비율 | 서비스                                                                                     |
| ---------- | ----------- | ------------------------------------------------------------------------------------------ |
| `critical` | 100%        | payment, checkout, frontend, frontend-proxy                                                |
| `high`     | 50%         | cart, product-catalog, currency, shipping                                                  |
| `medium`   | 10%         | recommendation, ad, email                                                                  |
| `low`      | 1%          | accounting, fraud-detection, image-provider, load-generator, quote, flagd, flagd-ui, Kafka |

## 컬렉터 구성 {#collector-configuration}

테일 기반 샘플링을 활성화하려면 `otelcol-config-extras.yml`에 다음을 추가한다.

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      # Policy 1: Always sample critical services (100%)
      - name: critical-services-always-sample
        type: string_attribute
        string_attribute:
          key: service.criticality
          values:
            - critical
          enabled_regex_matching: false
          invert_match: false

      # Policy 2: Sample 50% of high-criticality services
      - name: high-criticality-probabilistic
        type: and
        and:
          and_sub_policy:
            - name: is-high-criticality
              type: string_attribute
              string_attribute:
                key: service.criticality
                values:
                  - high
            - name: probabilistic-50
              type: probabilistic
              probabilistic:
                sampling_percentage: 50

      # Policy 3: Sample 10% of medium-criticality services
      - name: medium-criticality-probabilistic
        type: and
        and:
          and_sub_policy:
            - name: is-medium-criticality
              type: string_attribute
              string_attribute:
                key: service.criticality
                values:
                  - medium
            - name: probabilistic-10
              type: probabilistic
              probabilistic:
                sampling_percentage: 10

      # Policy 4: Sample 1% of low-criticality services
      - name: low-criticality-probabilistic
        type: and
        and:
          and_sub_policy:
            - name: is-low-criticality
              type: string_attribute
              string_attribute:
                key: service.criticality
                values:
                  - low
            - name: probabilistic-1
              type: probabilistic
              probabilistic:
                sampling_percentage: 1

      # Policy 5: Always sample error traces regardless of criticality
      - name: errors-always-sample
        type: status_code
        status_code:
          status_codes:
            - ERROR

      # Policy 6: Always sample slow traces from critical/high services
      - name: slow-critical-traces
        type: and
        and:
          and_sub_policy:
            - name: is-critical-or-high
              type: string_attribute
              string_attribute:
                key: service.criticality
                values:
                  - critical
                  - high
            - name: is-slow
              type: latency
              latency:
                threshold_ms: 5000

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resourcedetection, memory_limiter, transform, tail_sampling]
      exporters: [otlp, debug, spanmetrics]
```

## 동작 방식 {#how-it-works}

테일 샘플링 프로세서는 완료된 트레이스를 구성된 정책과 비교하여 평가한다.
트레이스는 **하나라도** 일치하는 정책이 있으면 샘플링된다.

- **critical 서비스**는 결제 흐름, 체크아웃, 사용자 대면(user-facing) 서비스에
  대한 완전한 가시성을 보장하기 위해 항상 샘플링된다.
- **high 중요도 서비스**는 옵저버빌리티와 데이터 양의 균형을 맞추기 위해 50%로
  샘플링된다.
- **medium 및 low 중요도 서비스**는 덜 중요한 경로에서 발생하는 노이즈를 줄이기
  위해 점진적으로 더 낮은 비율로 샘플링된다.
- **오류는 서비스 중요도와 관계없이 항상 캡처되어**, 어떤 문제도 놓치지 않도록
  한다.
- **느린 트레이스**(5초 초과)는 critical 및 high 중요도 서비스에서 발생한 경우,
  성능 병목 지점을 파악하는 데 도움이 되도록 항상 샘플링된다.
