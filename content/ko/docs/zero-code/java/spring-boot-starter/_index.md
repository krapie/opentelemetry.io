---
title: Spring Boot 스타터
aliases:
  - /docs/languages/java/spring-boot
  - /docs/languages/java/automatic/spring-boot
  - /docs/zero-code/java/agent/spring-boot
  - /docs/zero-code/java/spring-boot
default_lang_commit: 2d89b60b2e09d42ba96757b0afdbc31f54a2b0e7
---

오픈텔레메트리(OpenTelemetry)로
[Spring Boot](https://spring.io/projects/spring-boot) 애플리케이션을 계측하는
방법에는 두 가지 옵션이 있다.

1. Spring Boot 애플리케이션을 계측하는 기본 선택지는 바이트코드 계측(bytecode
   instrumentation)을 사용하는 [**오픈텔레메트리 Java 에이전트**](../agent)이다.
   - 오픈텔레메트리 스타터보다 **기본으로 제공되는 계측 범위가 더 넓다**
2. **오픈텔레메트리 Spring Boot 스타터(starter)** 는 다음과 같은 경우에 도움이
   될 수 있다.
   - 오픈텔레메트리 Java 에이전트가 동작하지 않는 **Spring Boot Native 이미지**
     애플리케이션
   - 오픈텔레메트리 Java 에이전트의 **시작 오버헤드(startup overhead)** 가 요구
     사항을 초과하는 경우
   - 다른 Java 모니터링 에이전트를 이미 사용 중이어서 오픈텔레메트리 Java
     에이전트가 해당 에이전트와 함께 동작하지 않을 수 있는 경우
   - 오픈텔레메트리 Java 에이전트와는 함께 동작하지 않는 오픈텔레메트리 Spring
     Boot 스타터를 구성하기 위한 **Spring Boot 구성 파일**
     (`application.properties`, `application.yml`)
   - `application.yaml` 내부의 구조화된 YAML 형식을 사용하는
     **[선언적 구성(Declarative configuration)](declarative-configuration/)**
