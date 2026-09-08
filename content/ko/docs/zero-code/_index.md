---
title: 제로 코드 계측
redirects: [{ from: 'net/*', to: 'dotnet/:splat' }]
weight: 265
default_lang_commit: 4b5381a2e9f129651ab8658357ab846bd4c965f2
---

오픈텔레메트리(OpenTelemetry) [제로 코드 계측][zero-code instrumentation]은 아래
섹션 색인에 나열된 언어에 대해 지원된다.

쿠버네티스(Kubernetes)를 사용하는 경우, [쿠버네티스용 오픈텔레메트리
오퍼레이터(Operator)][otel-op]를 사용하여 .NET, Java, Node.js, Python 또는 Go
애플리케이션에 [제로 코드 계측을 주입][inject zero-code instrumentation]할 수
있다.

[inject zero-code instrumentation]:
  /docs/platforms/kubernetes/operator/automatic/
[zero-code instrumentation]: /docs/concepts/instrumentation/zero-code/
[otel-op]: /docs/platforms/kubernetes/operator/
