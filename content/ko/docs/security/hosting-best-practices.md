---
title: 컬렉터(Collector) 호스팅 모범 사례
linkTitle: 컬렉터 호스팅
weight: 115
default_lang_commit: d96ef10c0bf452f6e01b9d9b596355693638e0d9
---

오픈텔레메트리(OpenTelemetry, OTel) 컬렉터의 호스팅을 설정할 때는 호스팅
인스턴스를 더 안전하게 보호하기 위해 다음 모범 사례를 고려한다.

## 데이터를 안전하게 저장하기 {#store-data-securely}

컬렉터 구성 파일에는 인증 토큰이나 TLS 인증서 등 민감한 데이터가 포함될 수 있다.
구성을 안전하게 보호하는 방법은
[안전한 구성 만들기](../config-best-practices/#create-secure-configurations)를
참고한다.

처리를 위해 텔레메트리를 저장하는 경우, 원본 데이터가 변조되지 않도록 해당
디렉터리에 대한 접근을 제한해야 한다.

## 시크릿(Secret) 안전하게 보관하기 {#keep-your-secrets-safe}

쿠버네티스 [시크릿](https://kubernetes.io/docs/concepts/configuration/secret/)은
기밀 데이터를 담고 있는 자격 증명으로, 권한이 있는 접근을 인증하고 승인하는
역할을 한다. 컬렉터를 쿠버네티스에 배포하는 경우, 클러스터의 보안을 강화하기
위해 다음
[권장 사례](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)를
따르도록 한다.

## 최소 권한 원칙 적용하기 {#apply-the-principle-of-least-privilege}

컬렉터가 수집하는 데이터가 권한이 필요한 위치에 있는 경우가 아니라면, 컬렉터는
권한이 있는 접근을 필요로 해서는 안 된다. 예를 들어 쿠버네티스 배포에서 시스템
로그, 애플리케이션 로그, 컨테이너 런타임 로그는 접근에 특수 권한이 필요한 노드
볼륨에 저장되는 경우가 많다. 컬렉터가 노드에서 `daemonset`으로 실행되고 있다면,
이러한 로그에 접근하는 데 필요한 특정 볼륨 마운트 권한만 부여하고 그 이상은
부여하지 않도록 한다. 역할 기반 접근 제어(Role-Based Access Control, RBAC)를
사용해 권한이 있는 접근을 구성할 수 있다. 자세한 내용은
[RBAC 권장 사례](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)를
참고한다.

## 서버와 유사한 구성 요소에 대한 접근 제어하기 {#control-access-to-server-like-components}

리시버나 익스포터와 같은 일부 컬렉터 구성 요소는 서버처럼 동작할 수 있다. 인가된
사용자로 접근을 제한하려면 다음을 수행해야 한다.

- 예를 들어 베어러 토큰 인증 익스텐션과 기본 인증 익스텐션을 사용해 인증을
  활성화한다.
- 컬렉터가 실행되는 IP를 제한한다.

## 리소스 사용량 보호하기 {#safeguard-resource-utilization}

컬렉터 자체의 [내부 텔레메트리](/docs/collector/internal-telemetry/)를 사용해
성능을 모니터링한다. 컬렉터의 CPU, 메모리, 처리량 사용에 대한 메트릭을 수집하고,
리소스 고갈에 대한 경고를 설정한다.

리소스 한도에 도달한 경우, 로드 밸런싱된 구성으로 여러 인스턴스를 배포해
[컬렉터를 수평으로 스케일링](/docs/collector/scaling/)하는 것을 고려한다.
컬렉터를 스케일링하면 리소스 수요가 분산되어 병목 현상을 방지할 수 있다.

배포에서 리소스 사용량에 대한 보안을 확보했다면, 컬렉터 인스턴스도
[구성 내 안전장치](../config-best-practices/#safeguard-resource-utilization)를
사용하도록 한다.
