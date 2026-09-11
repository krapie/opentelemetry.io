---
title: 컬렉터(Collector) 구성 모범 사례
linkTitle: 컬렉터 구성
weight: 112
default_lang_commit: 6a7f17450ce3edc2e4363013551ee93ba7934a5d
cSpell:ignore: exporterhelper
---

오픈텔레메트리(OpenTelemetry, OTel) 컬렉터를 구성할 때는 컬렉터 인스턴스를 더
안전하게 보호하기 위해 다음 모범 사례를 고려한다.

## 안전한 구성 만들기 {#create-secure-configurations}

컬렉터의 구성과 파이프라인을 안전하게 보호하려면 다음 지침을 따른다.

### 구성을 안전하게 저장하기 {#store-your-configuration-securely}

컬렉터의 구성에는 다음과 같은 민감한 정보가 포함될 수 있다.

- API 토큰과 같은 인증 정보.
- 개인 키를 포함한 TLS 인증서.

민감한 정보는 암호화된 파일 시스템이나 시크릿 저장소 등에 안전하게 저장해야
한다. 컬렉터는
[환경 변수 확장](/docs/collector/configuration/#environment-variables)을
지원하므로 환경 변수를 사용해 민감한 데이터와 그렇지 않은 데이터를 모두 다룰 수
있다.

### 암호화 및 인증 사용하기 {#use-encryption-and-authentication}

OTel 컬렉터 구성에는 암호화와 인증이 포함되어야 한다.

- 통신 암호화에 대해서는
  [인증서 구성하기](/docs/collector/configuration/#setting-up-certificates)를
  참고한다.
- 인증에는 [인증](/docs/collector/configuration/#authentication)에서 설명하는
  OTel 컬렉터의 인증 메커니즘을 사용한다.

### 구성 요소 수 최소화하기 {#minimize-the-number-of-components}

컬렉터 구성에 포함하는 구성 요소 집합은 필요한 것만으로 제한할 것을 권장한다.
사용하는 구성 요소 수를 최소화하면 노출되는 공격 표면도 최소화된다.

- 필요한 구성 요소만 사용하는 컬렉터 배포판을 만들려면
  [오픈텔레메트리 컬렉터 빌더(`ocb`)](/docs/collector/extend/ocb/)를 사용한다.
- 구성에서 사용하지 않는 구성 요소를 제거한다.

### 신중하게 구성하기 {#configure-with-care}

일부 구성 요소는 컬렉터 파이프라인의 보안 위험을 높일 수 있다.

- 리시버, 익스포터 및 기타 구성 요소는 안전한 채널을 통해 네트워크 연결을 맺어야
  하며, 가능하다면 인증도 거쳐야 한다.
- 리시버와 익스포터는 구성 매개변수를 통해 버퍼, 큐, 페이로드, 워커 설정을
  노출할 수 있다. 이러한 설정을 사용할 수 있는 경우, 기본 구성 값을 수정하기
  전에 주의해야 한다. 이 값을 잘못 설정하면 오픈텔레메트리 컬렉터가 추가적인
  공격 벡터에 노출될 수 있다.

## 권한을 신중하게 설정하기 {#set-permissions-carefully}

컬렉터를 루트 사용자로 실행하는 것은 피한다. 다만 일부 구성 요소는 특수한 권한을
필요로 할 수도 있다. 이런 경우에는 최소 권한 원칙을 따르고, 구성 요소가 자신의
작업을 수행하는 데 필요한 접근 권한만 갖도록 한다.

### 옵저버(Observer) {#observers}

옵저버는 익스텐션으로 구현된다. 익스텐션은 컬렉터의 주요 기능 위에 부가적인
기능을 더하는 유형의 구성 요소다. 익스텐션은 텔레메트리에 직접 접근할 필요가
없고 파이프라인의 일부도 아니지만, 특수한 권한을 필요로 하는 경우에는 여전히
보안 위험이 될 수 있다.

옵저버는
[리시버 생성기(receiver creator)](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/receivercreator/README.md)를
대신해 쿠버네티스 파드, 도커 컨테이너, 로컬 리스닝 포트와 같은 네트워크
엔드포인트를 검색한다. 서비스를 검색하려면 옵저버가 더 큰 접근 권한을 필요로 할
수 있다. 예를 들어 `k8s_observer`는 쿠버네티스에서
[역할 기반 접근 제어(role-based access control, RBAC) 권한](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/observer/k8sobserver#setting-up-rbac-permissions)을
필요로 한다.

## 특정 보안 위험 관리하기 {#manage-specific-security-risks}

이러한 보안 위협을 차단하도록 컬렉터를 구성한다.

### 서비스 거부(denial of service, DoS) 공격 방어하기 {#protect-against-denial-of-service-attacks}

서버와 유사한 리시버와 익스텐션의 경우, 이러한 구성 요소의 엔드포인트를 인가된
사용자만 접속할 수 있는 주소에 바인딩함으로써 컬렉터가 퍼블릭 인터넷이나 필요
이상으로 넓은 네트워크에 노출되지 않도록 보호할 수 있다. `0.0.0.0` 대신 파드의
IP와 같은 특정 인터페이스나 `localhost`를 사용하도록 한다. 자세한 내용은
[CWE-1327: Binding to an Unrestricted IP Address](https://cwe.mitre.org/data/definitions/1327.html)를
참고한다.

컬렉터 v0.110.0부터는 컬렉터 구성 요소 내 모든 서버의 기본 호스트가
`localhost`다. 이전 버전의 컬렉터에서는 `component.UseLocalHostAsDefaultHost`
[기능 게이트(feature gate)](https://github.com/open-telemetry/opentelemetry-collector/tree/main/featuregate)를
활성화하여 모든 구성 요소의 기본 엔드포인트를 `0.0.0.0`에서 `localhost`로
변경한다.

DNS 설정으로 인해 `localhost`가 다른 IP로 해석되는 경우, 대신 루프백 IP를
명시적으로 사용한다. IPv4에서는 `127.0.0.1`, IPv6에서는 `::1`을 사용한다. 예를
들어 gRPC 포트를 사용하는 IPv4 구성은 다음과 같다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 127.0.0.1:4317
```

IPv6 환경에서는 두 프로토콜 버전이 모두 사용되는 듀얼 스택 환경과
애플리케이션에서 네트워크가 정상적으로 동작하도록, 시스템이 IPv4와 IPv6 루프백
주소를 모두 지원하는지 확인한다.

도커나 쿠버네티스처럼 비표준적인 네트워크 구성을 갖는 환경에서 작업하는 경우에는
`localhost`가 예상대로 동작하지 않을 수 있다. 다음 예시는 OTLP 리시버 gRPC
엔드포인트에 대한 구성을 보여준다. 다른 컬렉터 구성 요소에도 유사한 구성이
필요할 수 있다.

#### 도커 {#docker}

올바른 주소에 바인딩함으로써 도커에서 컬렉터를 실행할 수 있다. 다음은 도커에서
OTLP 익스포터를 위한 `config.yaml` 구성 파일이다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: my-hostname:4317 # Use the same hostname from your docker run command
```

`docker run` 명령에서 `--hostname` 인수를 사용해 컬렉터를 `my-hostname` 주소에
바인딩한다. 해당 도커 네트워크 밖에서(예를 들어 호스트에서 실행되는 일반
프로그램에서) `127.0.0.1:4567`로 접속하여 컬렉터에 접근할 수 있다. 다음은
`docker run` 명령의 예시다.

```shell
docker run --hostname my-hostname --name container-name -p 127.0.0.1:4567:4317 otel/opentelemetry-collector:{{% param collector_vers %}}
```

#### 도커 컴포즈 {#docker-compose}

일반 도커와 마찬가지로, 올바른 주소에 바인딩함으로써 도커에서 컬렉터를 실행할 수
있다.

도커 `compose.yaml` 파일:

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:{{% param collector_vers %}}
    ports:
      - '4567:4317'
```

컬렉터 `config.yaml` 파일:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: otel-collector:4317 # Use the service name from your Docker compose file
```

같은 네트워크에서 실행 중인 다른 도커 컨테이너에서는 `otel-collector:4317`로
접속하여 이 컬렉터에 연결할 수 있다. 해당 도커 네트워크 밖에서(예를 들어
호스트에서 실행되는 일반 프로그램에서) `127.0.0.1:4567`로 접속하여 컬렉터에
접근할 수 있다.

#### 쿠버네티스 {#kubernetes}

컬렉터를 `DaemonSet`으로 실행하는 경우, 다음과 같은 구성을 사용할 수 있다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: collector
spec:
  selector:
    matchLabels:
      name: collector
  template:
    metadata:
      labels:
        name: collector
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector:{{% param collector_vers %}}
          ports:
            - containerPort: 4317
              hostPort: 4317
              protocol: TCP
              name: otlp-grpc
            - containerPort: 4318
              hostPort: 4318
              protocol: TCP
              name: otlp-http
          env:
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
```

이 예시에서는
[쿠버네티스 다운워드 API(Downward API)](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/)를
사용해 자신의 파드 IP를 가져온 다음, 해당 네트워크 인터페이스에 바인딩한다.
그리고 `hostPort` 옵션을 사용해 컬렉터가 호스트에 노출되도록 한다. 컬렉터의
구성은 다음과 같아야 한다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
```

노드의 IP 주소인 `MY_HOST_IP`를 사용해 `${MY_HOST_IP}:4317`에 접근하면 gRPC를
통해, `${MY_HOST_IP}:4318`에 접근하면 HTTP를 통해 해당 노드의 어떤 파드에서든 이
컬렉터로 OTLP 데이터를 전송할 수 있다. 이 IP는 다운워드 API에서 가져올 수 있다.

```yaml
env:
  - name: MY_HOST_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
```

### 민감한 데이터 제거하기 {#scrub-sensitive-data}

[프로세서](/docs/collector/configuration/#processors)는 리시버와 익스포터 사이에
위치하는 컬렉터 구성 요소로, 텔레메트리가 분석되기 전에 이를 처리하는 역할을
한다. 오픈텔레메트리 컬렉터의 `redaction` 프로세서를 사용하면 백엔드로 내보내기
전에 민감한 데이터를 난독화하거나 제거할 수 있다.

[`redaction` 프로세서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor)는
허용된 속성 목록과 일치하지 않는 스팬, 로그, 메트릭 데이터포인트 속성을
삭제한다. 또한 차단된 값 목록과 일치하는 속성 값을 마스킹한다. 허용 목록에 없는
속성은 값 검사가 이루어지기 전에 먼저 제거된다.

예를 들어 다음은 신용카드 번호가 포함된 값을 마스킹하는 구성이다.

```yaml
processors:
  redaction:
    allow_all_keys: false
    allowed_keys:
      - description
      - group
      - id
      - name
    ignored_keys:
      - safe_attribute
    blocked_values: # Regular expressions for blocking values of allowed span attributes
      - '4[0-9]{12}(?:[0-9]{3})?' # Visa credit card number
      - '(5[1-5][0-9]{14})' # MasterCard number
    summary: debug
```

컬렉터 구성에 `redaction` 프로세서를 추가하는 방법을 알아보려면
[문서](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor)를
참고한다.

### 리소스 사용량 보호하기 {#safeguard-resource-utilization}

[호스팅 인프라](../hosting-best-practices/)에 리소스 사용량에 대한 안전장치를
구현했다면, 오픈텔레메트리 컬렉터 구성에도 이러한 안전장치를 추가하는 것을
고려한다.

텔레메트리를 배칭(batching)하고 컬렉터가 사용할 수 있는 메모리를 제한하면 메모리
부족 오류와 사용량 급증을 방지할 수 있다. 큐 크기를 조정하여 데이터 손실을
피하면서 메모리 사용량을 관리함으로써 트래픽 급증에도 대응할 수 있다. 예를 들어
`otlp` 익스포터의 큐 크기를 관리하려면
[`exporterhelper`](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/exporterhelper/README.md)를
사용한다.

```yaml
exporters:
  otlp:
    endpoint: <ENDPOINT>
    sending_queue:
      queue_size: 800
```

원치 않는 텔레메트리를 필터링하는 것도 컬렉터의 리소스를 보호하는 또 다른
방법이다. 필터링은 컬렉터 인스턴스를 보호할 뿐만 아니라 백엔드의 부하도
줄여준다. 필요하지 않은 로그, 메트릭, 스팬을 삭제하려면
[`filter` 프로세서](/docs/collector/transforming-telemetry/#basic-filtering)를
사용한다. 예를 들어 다음은 HTTP가 아닌 스팬을 삭제하는 구성이다.

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        - attributes["http.request.method"] == nil
```

구성 요소에 적절한 타임아웃 및 재시도 제한값을 설정할 수도 있다. 이러한 제한값을
설정하면 컬렉터가 메모리에 지나치게 많은 데이터를 쌓지 않고도 장애를 처리할 수
있다. 자세한 내용은
[`exporterhelper` 문서](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/exporterhelper/README.md)를
참고한다.

마지막으로, 익스포터에 압축을 사용하면 데이터 전송 크기를 줄이고 네트워크와 CPU
리소스를 절약할 수 있다. 기본적으로
[`otlp` 익스포터](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlpexporter)는
`gzip` 압축을 사용한다.
