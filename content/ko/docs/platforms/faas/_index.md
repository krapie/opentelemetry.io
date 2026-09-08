---
title: 서비스형 함수(Functions as a Service)
linkTitle: FaaS
description: >-
  오픈텔레메트리(OpenTelemetry)는 여러 클라우드 벤더가 제공하는 서비스형
  함수(Function-as-a-Service)를 모니터링하는 다양한 방법을 지원한다.
redirects: [{ from: /docs/faas/*, to: ':splat' }] # cSpell:disable-line
default_lang_commit: 4b5381a2e9f129651ab8658357ab846bd4c965f2
---

서비스형 함수(Functions as a Service, FaaS)는 [클라우드 네이티브 앱][]에 있어
중요한 서버리스 컴퓨팅 플랫폼이다. 하지만 플랫폼별 특성으로 인해 이러한
애플리케이션은 쿠버네티스(Kubernetes)나 가상 머신에서 실행되는 애플리케이션과는
다소 다른 모니터링 가이드와 요구 사항을 갖는 경우가 많다.

FaaS 문서에서 초기에 다루는 벤더 범위는 Microsoft Azure, Google Cloud Platform
(GCP), Amazon Web Services (AWS)이다. AWS의 함수는 Lambda로도 불린다.

## 커뮤니티 자산 {#community-assets}

오픈텔레메트리 커뮤니티는 현재 애플리케이션을 자동 계측할 수 있는 사전 빌드된
Lambda 레이어와, 애플리케이션을 수동 또는 자동으로 계측할 때 사용할 수 있는 독립
실행형(standalone) 컬렉터 Lambda 레이어 옵션을 제공한다.

릴리스 상태는
[OpenTelemetry-Lambda 저장소](https://github.com/open-telemetry/opentelemetry-lambda)에서
추적할 수 있다.

[클라우드 네이티브 앱]: https://glossary.cncf.io/cloud-native-apps/
