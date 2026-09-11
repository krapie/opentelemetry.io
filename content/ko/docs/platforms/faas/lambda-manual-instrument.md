---
title: Lambda 수동 계측
weight: 11
description: 오픈텔레메트리(OpenTelemetry)로 Lambda를 수동으로 계측한다.
default_lang_commit: f49ec57e5a0ec766b07c7c8e8974c83531620af3
---

Lambda 자동 계측 문서에서 다루지 않는 언어의 경우, 커뮤니티가 제공하는 독립
실행형 계측 레이어가 없다.

사용자는 선택한 언어의 일반적인 계측 가이드를 따라야 하며, 데이터를 제출하기
위해 컬렉터 Lambda 레이어를 추가해야 한다.

## OTel 컬렉터 Lambda 레이어의 ARN 추가 {#add-the-arn-of-the-otel-collector-lambda-layer}

애플리케이션에 레이어를 추가하고 컬렉터를 구성하려면
[컬렉터 Lambda 레이어 가이드](../lambda-collector/)를 참고한다. 이 작업을 먼저
진행하는 것을 권장한다.

## OTel로 Lambda 계측하기 {#instrument-the-lambda-with-otel}

애플리케이션을 수동으로 계측하는 방법은 [언어별 계측 가이드](/docs/languages/)를
참고한다.

## Lambda 게시 {#publish-your-lambda}

새로운 변경 사항과 계측을 배포하려면 Lambda의 새 버전을 게시한다.
