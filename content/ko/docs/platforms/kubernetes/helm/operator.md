---
title: 오픈텔레메트리 오퍼레이터 차트
linkTitle: 오퍼레이터 차트
default_lang_commit: 924424b3aad888ee3ddb745eaed063021a5ef8d9
---

## 소개 {#introduction}

[오픈텔레메트리 오퍼레이터](/docs/platforms/kubernetes/operator)는
[오픈텔레메트리 컬렉터](/docs/collector)와 워크로드의 자동 계측을 관리하는
쿠버네티스 오퍼레이터이다. 오픈텔레메트리 오퍼레이터를 설치하는 방법 중 하나는
[오픈텔레메트리 오퍼레이터 Helm 차트](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator)를
사용하는 것이다.

오픈텔레메트리 오퍼레이터의 자세한 사용법은
[문서](/docs/platforms/kubernetes/operator)를 참고한다.

### 차트 설치 {#installing-the-chart}

릴리스 이름 `my-opentelemetry-operator`로 차트를 설치하려면 다음 명령을
실행한다.

```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

```shell
helm install my-opentelemetry-operator open-telemetry/opentelemetry-operator \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true
```

이렇게 하면 자체 서명된(self-signed) 인증서와 시크릿을 사용하는 오픈텔레메트리
오퍼레이터가 설치된다.

### 구성 {#configuration}

오퍼레이터 Helm 차트의 기본 `values.yaml`은 바로 설치할 수 있는 상태이지만,
클러스터에 Cert Manager가 이미 설치되어 있어야 한다.

쿠버네티스에서 API 서버가 웹훅(webhook) 컴포넌트와 통신하려면, 웹훅에는 API
서버가 신뢰하도록 구성된 TLS 인증서가 필요하다. 필요한 TLS 인증서를
생성/구성하는 방법에는 몇 가지가 있다.

- 가장 쉽고 기본적인 방법은 [cert-manager](https://cert-manager.io/docs/)를
  설치하고 `admissionWebhooks.certManager.enabled`를 `true`로 설정하는 것이다.
  이렇게 하면 cert-manager가 자체 서명된 인증서를 생성한다. 자세한 내용은
  [cert-manager 설치](https://cert-manager.io/docs/installation/kubernetes/)를
  참고한다.
- `admissionWebhooks.certManager.issuerRef` 값을 구성해 직접 Issuer를 제공할
  수도 있다. `kind`(Issuer 또는 ClusterIssuer)와 `name`을 지정해야 한다. 이
  방법도 cert-manager 설치가 필요하다는 점에 유의한다.
- `admissionWebhooks.certManager.enabled`를 `false`로,
  `admissionWebhooks.autoGenerateCert.enabled`를 `true`로 설정하면 자동으로
  생성된 자체 서명 인증서를 사용할 수 있다. Helm이 자체 서명된 인증서와 시크릿을
  생성해준다.
- `admissionWebhooks.certManager.enabled`와
  `admissionWebhooks.autoGenerateCert.enabled`를 모두 `false`로 설정하면 직접
  생성한 자체 서명 인증서를 사용할 수 있다. `admissionWebhooks.cert_file`,
  `admissionWebhooks.key_file`, `admissionWebhooks.ca_file`에 필요한 값을
  제공해야 한다.
- `.Values.admissionWebhooks.create`와 `admissionWebhooks.certManager.enabled`를
  비활성화하고 `admissionWebhooks.secretName`에 커스텀 인증서 시크릿 이름을
  설정하면 커스텀 웹훅과 인증서를 사이드로드(side-load)할 수 있다.
- `.Values.admissionWebhooks.create`를 비활성화하고 환경 변수
  `.Values.manager.env.ENABLE_WEBHOOKS`를 `false`로 설정하면 웹훅을 모두
  비활성화할 수 있다.

차트에서 사용 가능한 모든 구성 옵션(주석 포함)은
[values.yaml 파일](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-operator/values.yaml)에서
확인할 수 있다.
