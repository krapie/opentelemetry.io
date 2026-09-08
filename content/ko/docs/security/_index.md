---
title: 보안
cascade:
  collector_vers: 0.159.0
weight: 970
default_lang_commit: de42414bea5449f67be2211e96836fbc70660548
---

이 섹션에서는 오픈텔레메트리(OpenTelemetry) 프로젝트가 취약점을 어떻게 공개하고
사고에 어떻게 대응하는지 알아보고, 옵저버빌리티(observability) 데이터를 안전하게
수집하고 전송하기 위해 무엇을 할 수 있는지 살펴본다.

## 일반 취약점 및 노출(CVE) {#common-vulnerabilities-and-exposures-cves}

모든 저장소에 걸친 CVE 목록은 [일반 취약점 및 노출](cve/)을 참고한다.

## 사고 대응 {#incident-response}

취약점을 보고하는 방법이나 사고 대응이 어떻게 처리되는지 알아보려면
[커뮤니티 사고 대응 가이드라인](security-response/)을 참고한다.

## 컬렉터(Collector) 보안 {#collector-security}

오픈텔레메트리 컬렉터를 설정할 때는 호스팅 인프라와 컬렉터 구성 모두에서 보안
모범 사례를 적용하는 것을 고려한다. 안전한 컬렉터를 운영하면 다음과 같은 이점이
있다.

- 개인 식별 정보(personally identifiable information, PII), 애플리케이션별
  데이터, 네트워크 트래픽 패턴 등 포함되어서는 안 되지만 포함될 수도 있는 민감한
  정보로부터 텔레메트리를 보호할 수 있다.
- 텔레메트리를 신뢰할 수 없게 만들고 사고 대응을 방해하는 데이터 변조를 방지할
  수 있다.
- 데이터 프라이버시 및 보안 규정을 준수할 수 있다.
- 서비스 거부(denial of service, DoS) 공격을 방어할 수 있다.

컬렉터의 인프라를 안전하게 만드는 방법을 알아보려면
[호스팅 모범 사례](hosting-best-practices/)를 참고한다.

컬렉터를 안전하게 구성하는 방법을 알아보려면
[구성 모범 사례](config-best-practices/)를 참고한다.

컬렉터 구성 요소 개발자는
[보안 모범 사례](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md)를
참고한다.
