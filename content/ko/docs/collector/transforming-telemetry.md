---
title: 텔레메트리 변환하기
weight: 26
# prettier-ignore
cSpell:ignore: accountid clustername k8sattributes metricstransform OTTL resourcedetection
default_lang_commit: 801233d066e99c97408663e5dbc6971fa38e94d3
---

오픈텔레메트리(OpenTelemetry) 컬렉터는 벤더나 다른 시스템으로 데이터를 전송하기
전에 데이터를 변환하기에 편리한 위치이다. 이는 데이터 품질, 거버넌스, 비용, 보안
등의 이유로 자주 수행된다.

[컬렉터 Contrib 저장소](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor)에서
제공하는 프로세서는 메트릭, 스팬, 로그 데이터에 대한 수십 가지 변환을 지원한다.
다음 섹션에서는 자주 사용되는 몇 가지 프로세서로 시작하는 방법에 대한 기본적인
예제를 제공한다.

프로세서의 구성, 특히 고급 변환은 컬렉터 성능에 상당한 영향을 미칠 수 있다.

## 기본 필터링 {#basic-filtering}

**프로세서**:
[필터 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)

필터 프로세서는 사용자가
[OTTL](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/pkg/ottl/README.md)을
사용하여 텔레메트리를 필터링할 수 있게 해준다. 조건과 일치하는 텔레메트리는
삭제된다.

예를 들어, app1, app2, app3 서비스의 스팬 데이터만 허용하고 다른 모든 서비스의
데이터는 삭제하려면 다음과 같이 구성한다.

```yaml
processors:
  filter/ottl:
    error_mode: ignore
    traces:
      span:
        - |
        resource.attributes["service.name"] != "app1" and
        resource.attributes["service.name"] != "app2" and
        resource.attributes["service.name"] != "app3"
```

`service1`이라는 서비스의 스팬만 삭제하고 다른 모든 스팬은 유지하려면 다음과
같이 구성한다.

```yaml
processors:
  filter/ottl:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["service.name"] == "service1"
```

[필터 프로세서 문서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)에는
로그 및 메트릭 필터링을 포함한 더 많은 예제가 있다.

## 속성 추가 또는 삭제 {#adding-or-deleting-attributes}

**프로세서**:
[속성 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor)
또는
[리소스 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourceprocessor)

속성 프로세서는 메트릭이나 트레이스에 있는 기존 속성을 업데이트, 삽입, 삭제 또는
교체하는 데 사용할 수 있다. 예를 들어, 다음은 모든 스팬에 account_id라는 속성을
추가하는 구성이다.

```yaml
processors:
  attributes/accountid:
    actions:
      - key: account_id
        value: 2245
        action: insert
```

리소스 프로세서는 동일한 구성을 가지지만
[리소스 속성](/docs/specs/semconv/resource/)에만 적용된다. 텔레메트리와 관련된
인프라 메타데이터를 수정하려면 리소스 프로세서를 사용한다. 예를 들어, 다음은
Kubernetes 클러스터 이름을 삽입하는 예이다.

```yaml
processors:
  resource/k8s:
    attributes:
      - key: k8s.cluster.name
        from_attribute: k8s-cluster
        action: insert
```

## 메트릭 또는 메트릭 레이블 이름 변경하기 {#renaming-metrics-or-metric-labels}

**프로세서:**
[메트릭 변환 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/metricstransformprocessor)

[메트릭 변환 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/metricstransformprocessor)는
[속성 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor)와
일부 기능을 공유하지만, 이름 변경 및 기타 메트릭 전용 기능도 지원한다.

```yaml
processors:
  metricstransform/rename:
    transforms:
      - include: system.cpu.usage
        action: update
        new_name: system.cpu.usage_time
```

[메트릭 변환 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/metricstransformprocessor)는
정규 표현식을 지원하여 여러 메트릭 이름이나 메트릭 레이블에 동시에 변환 규칙을
적용할 수도 있다. 다음 예제는 모든 메트릭에서 cluster_name을 cluster-name으로
이름을 변경한다.

```yaml
processors:
  metricstransform/clustername:
    transforms:
      - include: ^.*$
        match_type: regexp
        action: update
        operations:
          - action: update_label
            label: cluster_name
            new_label: cluster-name
```

## 리소스 속성으로 텔레메트리 보강하기 {#enriching-telemetry-with-resource-attributes}

**프로세서**:
[리소스 감지 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourcedetectionprocessor)
및
[k8sattributes 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor)

이 프로세서들은 팀이 기저 인프라가 서비스의 상태나 성능에 영향을 미치는 시점을
빠르게 파악할 수 있도록, 관련 인프라 메타데이터로 텔레메트리를 보강하는 데
사용할 수 있다.

리소스 감지 프로세서는 관련된 클라우드 또는 호스트 수준 정보를 텔레메트리에
추가한다.

```yaml
processors:
  resourcedetection/system:
    # Modify the list of detectors to match the cloud environment
    detectors: [env, system, gcp, ec2, azure]
    timeout: 2s
    override: false
```

마찬가지로, K8s 프로세서는 파드 이름, 노드 이름, 워크로드 이름과 같은 관련
Kubernetes 메타데이터로 텔레메트리를 보강한다. 컬렉터 파드는
[특정 Kubernetes RBAC API에 대한 읽기 접근 권한](https://pkg.go.dev/github.com/open-telemetry/opentelemetry-collector-contrib/processor/k8sattributesprocessor#readme-role-based-access-control)을
갖도록 구성되어야 한다. 기본 옵션을 사용하려면 빈 블록으로 구성할 수 있다.

```yaml
processors:
  k8sattributes/default:
```

## 스팬 상태 설정하기 {#setting-a-span-status}

**프로세서**:
[변환 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor)

변환 프로세서를 사용하여 스팬의 상태를 설정한다. 다음 예제는
`http.request.status_code` 속성이 400일 때 스팬 상태를 `Ok`로 설정한다.

<!-- prettier-ignore-start -->

```yaml
transform:
  error_mode: ignore
  trace_statements:
    - set(span.status.code, STATUS_CODE_OK) where span.attributes["http.request.status_code"] == 400
```

<!-- prettier-ignore-end -->

또한 변환 프로세서를 사용하여 속성을 기반으로 스팬 이름을 수정하거나 스팬
이름에서 스팬 속성을 추출할 수도 있다. 예제는 변환 프로세서의
[구성 파일](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/9b28f76c02c18f7479d10e4b6a95a21467fd85d6/processor/transformprocessor/testdata/config.yaml)
예시를 참고한다.

## 고급 변환 {#advanced-transformations}

더 고급 속성 변환은
[변환 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor)에서도
사용할 수 있다. 변환 프로세서는
[오픈텔레메트리 변환 언어(OpenTelemetry Transformation Language)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/ottl)를
사용하여 최종 사용자가 메트릭, 로그, 트레이스에 대한 변환을 지정할 수 있게
해준다.
