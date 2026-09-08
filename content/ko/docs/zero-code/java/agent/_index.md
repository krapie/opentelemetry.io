---
title: Java 에이전트
linkTitle: 에이전트
aliases:
  - /docs/java/automatic_instrumentation
  - /docs/languages/java/automatic_instrumentation
redirects: [{ from: /docs/languages/java/automatic/*, to: ':splat' }]
default_lang_commit: 94b83c9010b9e61255dc840066ae23313a05f592
---

Java를 위한 제로 코드 계측(Zero-code instrumentation)은 모든 Java 8 이상
애플리케이션에 연결되는 Java 에이전트(agent) JAR를 사용한다. 이 에이전트는
바이트코드 주입(bytecode injection)을 동적으로 수행하여 다양한 인기 라이브러리
및 프레임워크로부터 텔레메트리를 캡처한다. 이는 인바운드 요청, 아웃바운드 HTTP
호출, 데이터베이스 호출 등 애플리케이션이나 서비스의 "가장자리(edge)"에서
텔레메트리 데이터를 캡처하는 데 사용할 수 있다. 서비스나 애플리케이션 코드를
수동으로 계측하는 방법을 알아보려면
[수동 계측](/docs/languages/java/instrumentation/)을 참고한다.
