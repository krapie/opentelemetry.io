---
title: 'Mastodon: 소규모 팀으로 프로덕션에서 오픈텔레메트리 컬렉터 운영하기'
linkTitle: Mastodon
cSpell:ignore: otelbin Sidekiq Sloughter Öjeling
default_lang_commit: 11753c0e99bbc1b62606d4c819736c777dfb0e98
---

글: [Juliano Costa](https://github.com/julianocosta89) (Datadog),
[Tristan Sloughter](https://github.com/tsloughter) (커뮤니티),
[Johanna Öjeling](https://github.com/johannaojeling) (Grafana Labs),
[Damien Mathieu](https://github.com/dmathieu) (Elastic),
[Tim Campbell](https://github.com/timetinytim) (Mastodon) | 2026년 3월 18일

이 레퍼런스 구현은 놀라울 정도로 작은 팀으로 전 세계적인 규모에서 운영하는
비영리 조직인 Mastodon이 프로덕션에서 오픈텔레메트리 컬렉터를 어떻게 운영하는지
설명한다.

## 한눈에 보는 Mastodon {#mastodon-at-a-glance}

[Mastodon](https://joinmastodon.org)은 비영리 조직이 운영하는 무료, 오픈 소스,
탈중앙화 소셜 미디어 플랫폼이다.

여기서 탈중앙화는 마케팅 용어가 아니라 핵심적인 아키텍처 원칙이다. 누구나
[자신의 Mastodon 서버를 운영](https://docs.joinmastodon.org/user/run-your-own/)
할 수 있고, 이렇게 독립적으로 운영되는 서버들은 ActivityPub과 같은 표준화된
프로토콜을 사용해 서로 통신하는 독립 소셜 플랫폼들의 연합 네트워크인
_Fediverse_라 불리는 것의 일부로서 개방형 프로토콜을 사용해 상호 운용된다.
이메일과 마찬가지로, 사용자는 누가 운영하든 상관없이 인스턴스 간에 소통할 수
있다.

이러한 철학은 Mastodon의 기능 관련 결정뿐 아니라 옵저버빌리티에 대한 접근 방식도
형성한다.

### 조직 구조 {#organizational-structure}

Mastodon 조직 전체는 약 20명으로 구성되어 있으며, (오픈텔레메트리 컬렉터를
포함한) 옵저버빌리티 인프라는 단 한 명의 엔지니어가 관리한다.

작은 팀 규모에도 불구하고, Mastodon은 두 개의 대형 프로덕션 Mastodon 인스턴스를
운영한다.

- [mastodon.social](https://mastodon.social)

  9~~15개 노드(각 16코어, 64GB RAM) 사이에서 오토스케일링되는 쿠버네티스 위에서
  실행된다. 웹 프런트엔드는 5~~20개 파드 사이에서 스케일링되고, 다양한 Sidekiq
  워커 풀은 10~~40개 파드 사이에서 스케일링된다. 평균적으로 mastodon.social은
  언제든 70~~80개의 파드를 실행하고 있다. 이 플랫폼은 하루 최대 **30만
  명(300,000)의 활성 사용자**와 분당 약 1,000만 건의 요청을 처리한다.

- [mastodon.online](https://mastodon.online)

  3~~6개 노드(각 8코어, 32GB RAM) 사이에서 오토스케일링되는 쿠버네티스 위에서
  실행된다. 웹 프런트엔드는 3~~10개 파드 사이에서 스케일링되고, Sidekiq 풀은
  5~~15개 파드 사이에서 스케일링되어 평균 20~~30개의 파드가 실행된다. 이
  인스턴스는 더 작지만 여전히 상당한 규모에서 운영된다.

이렇게 제한된 운영 여력을 고려하면 단순함과 신뢰성은 타협할 수 없는 조건이다.

### 오픈텔레메트리 도입: 설계상의 선택의 자유 {#opentelemetry-adoption-freedom-of-choice-by-design}

Mastodon은 오픈 소스이며 다른 사람들이 직접 운영하도록 설계되었기 때문에, 팀은
운영자의 자유를 지켜주는 텔레메트리 솔루션을 원했다.

오픈텔레메트리는 각 Mastodon 서버 운영자가 텔레메트리를 어떻게, 또는 수집할지
여부를 직접 결정할 수 있게 해주기 때문에 기본값이 되었다.

간단한 [환경 변수 구성](https://docs.joinmastodon.org/admin/config/#otel)을
사용해, 운영자는 다음을 선택할 수 있다.

- 텔레메트리를 옵저버빌리티 백엔드로 직접 전송(Ruby SDK 구성만 사용)
- 오픈텔레메트리 컬렉터를 통해 텔레메트리 라우팅
- 텔레메트리를 완전히 비활성화

Mastodon의 핵심 조직은 외부 인스턴스가 옵저버빌리티를 어떻게 처리하는지 추적하지
않는다. 중요한 것은 내보내지는 텔레메트리가
**[오픈텔레메트리 시맨틱 컨벤션](/docs/specs/semconv/)** 을 엄격히 준수해,
어디서나 사용할 수 있게 하는 것이다.

이러한 접근 방식은 벤더 특화 데이터 모델을 피하고, Mastodon이 자체 컨벤션을 유지
관리할 필요 없이 더 넓은 오픈텔레메트리 생태계와의 호환성을 보장한다.

## 컬렉터 아키텍처: 네임스페이스당 하나, 그 이상은 없다 {#collector-architecture-one-per-namespace-no-more}

Mastodon의 컬렉터 아키텍처는 의도적으로 최소화되어 있다.

쿠버네티스 네임스페이스당 하나의 오픈텔레메트리 컬렉터가 트레이스, 메트릭, 로그
등 모든 텔레메트리 시그널을 처리한다. 별도의 게이트웨이와 에이전트 계층도 없고,
복잡한 라우팅 계층도, 커스텀 배포 툴링도 없다.

![Mastodon 노드 아키텍처 다이어그램](mastodon-nodes.png)

규모와 트래픽을 고려하면, 이것으로 충분함이 입증되었다.

Mastodon의 소프트웨어 엔지니어인
[Tim Campbell](https://github.com/timetinytim)은 컬렉터를 운영해온 약 2년 동안
단 한 번도 문제를 겪은 적이 없다고 밝혔다.

> "놀랍게도, 정말 기분 좋게 놀랍게도 단 한 번도 문제를 겪은 적이 없다.
> 쿠버네티스 오퍼레이터를 사용하고 있기 때문에, 혹시 문제가 생기더라도 자동으로
> 재시작된다. 적어도 Datadog에 남는 실제 트레이스와 로그를 보면 어떤 공백도
> 발견하지 못했다. 메모리와 프로세스 측면에서도 우리가 설정한 한도 안에서 완전히
> 안정적으로 유지되었다."

## 배포와 라이프사이클 관리 {#deployment-and-lifecycle-management}

운영 부담을 최대한 낮게 유지하기 위해 Mastodon은 다음에 의존한다.

- 쿠버네티스를 위한
  [오픈텔레메트리 오퍼레이터(Operator)](/docs/platforms/kubernetes/operator/)
- Git 기반 배포와 승격을 위한 Argo CD

각 컬렉터는 `OpenTelemetryCollector` 커스텀 리소스로 정의된다. 이후로는
쿠버네티스가 조정(reconciliation), 재시작, 라이프사이클 관리를 자동으로
처리한다.

> "기본적으로 우리가 만들어야 할 각 `OpenTelemetryCollector` 객체마다 yaml
> 파일만 만들면 되고, 필요한 것을 Argo가 자동으로 배포/업데이트해 준다."

이 모델은 다음을 제공한다.

- 선언적 구성
- 장애 발생 시 자동 복구
- Git 히스토리를 통한 명확한 감사 가능성(auditability)

특기할 점으로, Mastodon은 컬렉터 파드에 엄격한 CPU나 메모리 제한을 강제하지
않는다. 실제로 리소스 사용량은 플랫폼의 나머지 부분에 비해 무시할 만한 수준으로
유지되어 왔다.

## 샘플링을 통한 트래픽 관리 {#traffic-management-through-sampling}

리소스 제한에 의존하기보다, Mastodon은 주로 테일 기반(tail-based) 샘플링을 통해
옵저버빌리티 오버헤드를 통제한다.

- mastodon.social에서는 성공한 트레이스를 대략 0.1%만 샘플링해, 극도로 높은
  트래픽에도 불구하고 분당 수십 건의 트레이스만 생성된다.
- mastodon.online에서는 샘플링이 조금 더 완화되어 있지만 동일한 원칙을 따른다.
- 모든 오류 트레이스는 항상 수집되어, 장애에 대한 완전한 가시성을 보장한다.

이러한 접근 방식은 고가치 진단 데이터를 보존하면서도 데이터 양을 예측 가능하게
유지한다.

## 구성: 주관이 뚜렷하지만 최소한으로 {#configuration-opinionated-but-minimal}

Mastodon은 오픈텔레메트리 컬렉터 Contrib 배포판을 사용하는데, 주로 편의성
때문이다. 커스텀 빌드 없이도 필요한 모든 것이 포함되어 있다.

구성은 다음에 초점을 맞춘다.

- 모든 시그널을 위한 OTLP 인제스트
- 쿠버네티스 메타데이터 보강
- 리소스 감지
- 테일 기반 샘플링
- 백엔드 호환성을 위한 변환

전체 프로덕션 구성은 참고용으로 아래에 포함되어 있다([otelbin][otelbin-mastodon]
에서도 확인할 수 있다).

<details><summary>Mastodon의 컬렉터 구성</summary>

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: mastodon-social
  namespace: mastodon-social
spec:
  nodeSelector:
    joinmastodon.org/property: mastodon.social
  env:
    - name: DD_API_KEY
      valueFrom:
        secretKeyRef:
          name: datadog-secret
          key: api-key
    - name: DD_SITE
      valueFrom:
        secretKeyRef:
          name: datadog-secret
          key: site
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
            cors:
              allowed_origins:
                - 'http://*'
                - 'https://*'

    processors:
      batch: {}
      resource:
        attributes:
          - key: deployment.environment.name
            value: 'production'
            action: upsert
          - key: property
            value: 'mastodon.social'
            action: upsert
          - key: git.commit.sha
            from_attribute: vcs.repository.ref.revision
            action: insert
          - key: git.repository_url
            from_attribute: vcs.repository.url.full
            action: insert
      k8sattributes:
        auth_type: 'serviceAccount'
        passthrough: false
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.pod.start_time
            - k8s.pod.uid
            - k8s.deployment.name
            - k8s.node.name
          labels:
            - tag_name: app.label.component
              key: app.kubernetes.io/component
              from: pod
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
          - sources:
              - from: resource_attribute
                name: k8s.pod.uid
          - sources:
              - from: connection
      resourcedetection:
        detectors: [system]
        system:
          resource_attributes:
            os.description:
              enabled: true
            host.arch:
              enabled: true
            host.cpu.vendor.id:
              enabled: true
            host.cpu.family:
              enabled: true
            host.cpu.model.id:
              enabled: true
            host.cpu.model.name:
              enabled: true
            host.cpu.stepping:
              enabled: true
            host.cpu.cache.l2.size:
              enabled: true
      transform:
        error_mode: ignore

        # Proper code function naming
        trace_statements:
          - context: span
            conditions:
              - attributes["code.namespace"] != nil
            statements:
              - set(attributes["resource.name"],
                Concat([attributes["code.namespace"],
                attributes["code.function"]], "#"))

          # Proper kubernetes hostname
          - context: resource
            conditions:
              - attributes["k8s.node.name"] != nil
            statements:
              - set (attributes["k8s.node.name"],
                Concat([attributes["k8s.node.name"], "k8s-1"], "-"))
        metric_statements:
          - context: resource
            conditions:
              - attributes["k8s.node.name"] != nil
            statements:
              - set (attributes["k8s.node.name"],
                Concat([attributes["k8s.node.name"], "k8s-1"], "-"))
        log_statements:
          - context: resource
            conditions:
              - attributes["k8s.node.name"] != nil
            statements:
              - set (attributes["k8s.node.name"],
                Concat([attributes["k8s.node.name"], "k8s-1"], "-"))
      attributes/sidekiq:
        include:
          match_type: strict
          attributes:
            - key: messaging.sidekiq.job_class
        actions:
          - key: resource.name
            from_attribute: messaging.sidekiq.job_class
            action: upsert
      tail_sampling:
        policies:
          [
            {
              name: errors-policy,
              type: status_code,
              status_code: { status_codes: [ERROR] },
            },
            {
              name: randomized-policy,
              type: probabilistic,
              probabilistic: { sampling_percentage: 0.1 },
            },
          ]

    connectors:
      datadog/connector:
        traces:
          compute_stats_by_span_kind: true

    exporters:
      datadog:
        api:
          site: ${DD_SITE}
          key: ${DD_API_KEY}
        traces:
          compute_stats_by_span_kind: true
          trace_buffer: 500

    service:
      pipelines:
        traces/all:
          receivers: [otlp]
          processors:
            [
              resource,
              k8sattributes,
              resourcedetection,
              transform,
              attributes/sidekiq,
              batch,
            ]
          exporters: [datadog/connector]
        traces/sample:
          receivers: [datadog/connector]
          processors: [tail_sampling, batch]
          exporters: [datadog]
        metrics:
          receivers: [datadog/connector, otlp]
          processors:
            [resource, k8sattributes, resourcedetection, transform, batch]
          exporters: [datadog]
        logs:
          receivers: [otlp]
          processors:
            [
              resource,
              k8sattributes,
              resourcedetection,
              transform,
              attributes/sidekiq,
              batch,
            ]
          exporters: [datadog]
```

</details>

### 최신 상태 유지하기 {#staying-up-to-date}

Mastodon은 일반적으로 각 릴리스가 나온 지 하루 이틀 안에 오픈텔레메트리 컬렉터를
업그레이드한다.

> "모든 것이 문서화되어 있고, 모든 호환성이 깨지는 변경 사항이 제대로 상세히
> 설명되어 있다"라고 Tim은 릴리스 노트의 명확함을 칭찬하며 말했다.

잦은 릴리스가 때때로 호환성이 깨지는 변경을 가져오지만, 팀은 이를 계속 최신
상태를 유지하는 한 건강하고 활발한 개발의 신호로 본다.

### 배운 점과 어려웠던 점 {#lessons-and-pain-points}

여정에서 가장 어려웠던 부분은 단순히 시작하는 것이었다. 컬렉터의 컴포넌트들이
어떻게 맞물리는지 이해하는 데는 시간이 걸렸으며, 특히 전담 옵저버빌리티 전문가가
없는 팀에게는 더욱 그랬다. 더 최근에는 백엔드별 네이밍 요구사항에 맞춰 스팬
속성을 조정할 때 특히, transform 프로세서를 고급으로 활용하는 데서 가장 큰
복잡성이 나왔다.

```yaml
transform:
  error_mode: ignore

  # Proper code function naming
  trace_statements:
    - context: span
      conditions:
        - attributes["code.namespace"] != nil
      statements:
        - set(attributes["resource.name"], Concat([attributes["code.namespace"],
          attributes["code.function"]], "#"))
```

위의 transform 프로세서 규칙에서, 팀은 (Datadog 특화 속성인) `resource.name`을
`code.namespace#code.function`의 값으로 설정하는 조건을 구성했다. 이를 설정하고
나면, 스팬이 백엔드에 도달할 때마다 팀이 정의한 이름으로 매핑될 수 있었다. 이런
학습 곡선에도 불구하고 전반적인 경험은 기대를 뛰어넘었다.

> "원하는 건 기본적으로 뭐든 할 수 있다. 내 기대를 뛰어넘었다. 모든 게 꽤 잘
> 작동한다."

그러한 신뢰성과 유연성이 Mastodon이 프로덕션에서 계속 오픈텔레메트리 컬렉터를
사용하는 이유다.

## 소규모 팀을 위한 조언 {#advice-for-small-teams}

Mastodon의 경험을 바탕으로 몇 가지 교훈이 두드러진다.

- **아키텍처를 단순하게 유지한다**: 컬렉터 하나로도 많은 것을 해낼 수 있다
- 라이프사이클 관리를 위해 **쿠버네티스 오퍼레이터에 의존한다**
- 비용을 통제하기 위해 **샘플링을 사용한다**
- 장기적인 락인을 피하기 위해 **시맨틱 컨벤션을 고수한다**
- 호환성이 깨지는 변경으로 인한 고통을 줄이기 위해 **자주 업그레이드한다**

## 핵심 요약 {#takeaways}

Mastodon의 사례는 매우 작은 팀도 큰 운영 부담 없이 전 세계적인 규모에서
프로덕션에서 오픈텔레메트리 컬렉터를 성공적으로 운영할 수 있음을 보여준다.

[otelbin-mastodon]:
  https://www.otelbin.io/?#config=receivers%3A*N__otlp%3A*N____protocols%3A*N______grpc%3A*N________endpoint%3A_0.0.0.0%3A4317*N______http%3A*N________endpoint%3A_0.0.0.0%3A4318*N________cors%3A*N__________allowed*_origins%3A*N____________-_*%22http%3A%2F%2F***%22*N____________-_*%22https%3A%2F%2F***%22*N*Nprocessors%3A*N__batch%3A_%7B%7D*N__resource%3A*N____attributes%3A*N______-_key%3A_deployment.environment.name*N________value%3A_*%22production*%22*N________action%3A_upsert*N______-_key%3A_property*N________value%3A_*%22mastodon.social*%22*N________action%3A_upsert*N______-_key%3A_git.commit.sha*N________from*_attribute%3A_vcs.repository.ref.revision*N________action%3A_insert*N______-_key%3A_git.repository*_url*N________from*_attribute%3A_vcs.repository.url.full*N________action%3A_insert*N__k8sattributes%3A*N____auth*_type%3A_*%22serviceAccount*%22*N____passthrough%3A_false*N____extract%3A*N______metadata%3A*N________-_k8s.namespace.name*N________-_k8s.pod.name*N________-_k8s.pod.start*_time*N________-_k8s.pod.uid*N________-_k8s.deployment.name*N________-_k8s.node.name*N______labels%3A*N________-_tag*_name%3A_app.label.component*N__________key%3A_app.kubernetes.io%2Fcomponent*N__________from%3A_pod*N____pod*_association%3A*N______-_sources%3A*N__________-_from%3A_resource*_attribute*N____________name%3A_k8s.pod.ip*N______-_sources%3A*N__________-_from%3A_resource*_attribute*N____________name%3A_k8s.pod.uid*N______-_sources%3A*N__________-_from%3A_connection*N__resourcedetection%3A*N____detectors%3A_%5Bsystem%5D*N____system%3A*N______resource*_attributes%3A*N________os.description%3A*N__________enabled%3A_true*N________host.arch%3A*N__________enabled%3A_true*N________host.cpu.vendor.id%3A*N__________enabled%3A_true*N________host.cpu.family%3A*N__________enabled%3A_true*N________host.cpu.model.id%3A*N__________enabled%3A_true*N________host.cpu.model.name%3A*N__________enabled%3A_true*N________host.cpu.stepping%3A*N__________enabled%3A_true*N________host.cpu.cache.l2.size%3A*N__________enabled%3A_true*N__transform%3A*N____error*_mode%3A_ignore*N*N____*H_Proper_code_function_naming*N____trace*_statements%3A*N______-_context%3A_span*N________conditions%3A*N__________-_attributes%5B%22code.namespace%22%5D_%21*E_nil*N________statements%3A*N__________-_set*Cattributes%5B%22resource.name%22%5D%2C*N____________Concat*C%5Battributes%5B%22code.namespace%22%5D%2C*N____________attributes%5B%22code.function%22%5D%5D%2C_%22*H%22*D*D*N*N______*H_Proper_kubernetes_hostname*N______-_context%3A_resource*N________conditions%3A*N__________-_attributes%5B%22k8s.node.name%22%5D_%21*E_nil*N________statements%3A*N__________-_set_*Cattributes%5B%22k8s.node.name%22%5D%2C*N____________Concat*C%5Battributes%5B%22k8s.node.name%22%5D%2C_%22k8s-1%22%5D%2C_%22-%22*D*D*N____metric*_statements%3A*N______-_context%3A_resource*N________conditions%3A*N__________-_attributes%5B%22k8s.node.name%22%5D_%21*E_nil*N________statements%3A*N__________-_set_*Cattributes%5B%22k8s.node.name%22%5D%2C*N____________Concat*C%5Battributes%5B%22k8s.node.name%22%5D%2C_%22k8s-1%22%5D%2C_%22-%22*D*D*N____log*_statements%3A*N______-_context%3A_resource*N________conditions%3A*N__________-_attributes%5B%22k8s.node.name%22%5D_%21*E_nil*N________statements%3A*N__________-_set_*Cattributes%5B%22k8s.node.name%22%5D%2C*N____________Concat*C%5Battributes%5B%22k8s.node.name%22%5D%2C_%22k8s-1%22%5D%2C_%22-%22*D*D*N__attributes%2Fsidekiq%3A*N____include%3A*N______match*_type%3A_strict*N______attributes%3A*N________-_key%3A_messaging.sidekiq.job*_class*N____actions%3A*N______-_key%3A_resource.name*N________from*_attribute%3A_messaging.sidekiq.job*_class*N________action%3A_upsert*N__tail*_sampling%3A*N____policies%3A*N______%5B*N________%7B*N__________name%3A_errors-policy%2C*N__________type%3A_status*_code%2C*N__________status*_code%3A_%7B_status*_codes%3A_%5BERROR%5D_%7D%2C*N________%7D%2C*N________%7B*N__________name%3A_randomized-policy%2C*N__________type%3A_probabilistic%2C*N__________probabilistic%3A_%7B_sampling*_percentage%3A_0.1_%7D%2C*N________%7D%2C*N______%5D*N*Nconnectors%3A*N__datadog%2Fconnector%3A*N____traces%3A*N______compute*_stats*_by*_span*_kind%3A_true*N*Nexporters%3A*N__datadog%3A*N____api%3A*N______site%3A_*S%7BDD*_SITE%7D*N______key%3A_*S%7BDD*_API*_KEY%7D*N____traces%3A*N______compute*_stats*_by*_span*_kind%3A_true*N______trace*_buffer%3A_500*N*Nservice%3A*N__pipelines%3A*N____traces%2Fall%3A*N______receivers%3A_%5Botlp%5D*N______processors%3A*N________%5B*N__________resource%2C*N__________k8sattributes%2C*N__________resourcedetection%2C*N__________transform%2C*N__________attributes%2Fsidekiq%2C*N__________batch%2C*N________%5D*N______exporters%3A_%5Bdatadog%2Fconnector%5D*N____traces%2Fsample%3A*N______receivers%3A_%5Bdatadog%2Fconnector%5D*N______processors%3A_%5Btail*_sampling%2C_batch%5D*N______exporters%3A_%5Bdatadog%5D*N____metrics%3A*N______receivers%3A_%5Bdatadog%2Fconnector%2C_otlp%5D*N______processors%3A*N________%5Bresource%2C_k8sattributes%2C_resourcedetection%2C_transform%2C_batch%5D*N______exporters%3A_%5Bdatadog%5D*N____logs%3A*N______receivers%3A_%5Botlp%5D*N______processors%3A*N________%5B*N__________resource%2C*N__________k8sattributes%2C*N__________resourcedetection%2C*N__________transform%2C*N__________attributes%2Fsidekiq%2C*N__________batch%2C*N________%5D*N______exporters%3A_%5Bdatadog%5D%7E
