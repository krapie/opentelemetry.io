---
title: 오픈텔레메트리 데모 문서
linkTitle: 데모
cascade:
  repo: https://github.com/open-telemetry/opentelemetry-demo
weight: 180
default_lang_commit: ffef14de849130bdf9ecd9d4912e75f5a8afdbfd
---

오픈텔레메트리(OpenTelemetry)의 데모인 [오픈텔레메트리 데모](/ecosystem/demo/)
문서에 오신 것을 환영한다. 이 문서는 데모를 설치하고 실행하는 방법과,
오픈텔레메트리가 실제로 동작하는 모습을 확인할 수 있는 몇 가지 시나리오를
다룬다.

## 데모 실행 {#running-the-demo}

데모를 배포하고 실제로 동작하는 모습을 보고 싶다면 여기서 시작한다.

- [Docker](docker-deployment/)
- [쿠버네티스](kubernetes-deployment/)

## 언어별 기능 참조 {#language-feature-reference}

특정 언어의 계측(instrumentation)이 어떻게 동작하는지 알고 싶다면 여기서
시작한다.

| 언어       | 자동 계측                                                                                                                                  | 계측 라이브러리                                                                          | 수동 계측                                                                                |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| .NET       | [회계 서비스](services/accounting/)                                                                                                        | [장바구니 서비스](services/cart/)                                                        | [장바구니 서비스](services/cart/)                                                        |
| C++        |                                                                                                                                            |                                                                                          | [통화 서비스](services/currency/)                                                        |
| Elixir     |                                                                                                                                            | [Flagd-UI 서비스](services/flagd-ui/)                                                    |                                                                                          |
| Go         |                                                                                                                                            | [체크아웃 서비스](services/checkout/), [상품 카탈로그 서비스](services/product-catalog/) | [체크아웃 서비스](services/checkout/), [상품 카탈로그 서비스](services/product-catalog/) |
| Java       | [광고 서비스](services/ad/)                                                                                                                |                                                                                          | [광고 서비스](services/ad/)                                                              |
| JavaScript | [결제 서비스](services/payment/)                                                                                                           |                                                                                          | [결제 서비스](services/payment/)                                                         |
| TypeScript |                                                                                                                                            | [프론트엔드](services/frontend/), [React Native 앱](services/react-native-app/)          | [프론트엔드](services/frontend/)                                                         |
| Kotlin     |                                                                                                                                            | [사기 탐지 서비스](services/fraud-detection/)                                            |                                                                                          |
| PHP        |                                                                                                                                            | [견적 서비스](services/quote/)                                                           | [견적 서비스](services/quote/)                                                           |
| Python     | [추천 서비스](services/recommendation/), [에이전트 서비스](services/agent/), [챗봇 서비스](services/chatbot/), [MCP 서비스](services/mcp/) |                                                                                          | [추천 서비스](services/recommendation/)                                                  |
| Ruby       |                                                                                                                                            | [이메일 서비스](services/email/)                                                         | [이메일 서비스](services/email/)                                                         |
| Rust       |                                                                                                                                            | [배송 서비스](services/shipping/)                                                        | [배송 서비스](services/shipping/)                                                        |

## 서비스 문서 {#service-documentation}

각 서비스에 오픈텔레메트리가 어떻게 배포되는지에 대한 구체적인 정보는 여기서
확인할 수 있다.

- [회계 서비스](services/accounting/)
- [광고 서비스](services/ad/)
- [에이전트 서비스](services/agent/)
- [장바구니 서비스](services/cart/)
- [챗봇 서비스](services/chatbot/)
- [체크아웃 서비스](services/checkout/)
- [이메일 서비스](services/email/)
- [프론트엔드](services/frontend/)
- [부하 생성기](services/load-generator/)
- [MCP 서비스](services/mcp/)
- [결제 서비스](services/payment/)
- [상품 카탈로그 서비스](services/product-catalog/)
- [견적 서비스](services/quote/)
- [추천 서비스](services/recommendation/)
- [배송 서비스](services/shipping/)
- [이미지 제공 서비스](services/image-provider/)
- [React Native 앱](services/react-native-app/)

## 기능 플래그 시나리오 {#feature-flag-scenarios}

오픈텔레메트리로 문제를 어떻게 해결할 수 있을까? 이
[기능 플래그(feature flag) 시나리오](feature-flags/)는 사전 구성된 몇 가지
문제를 단계별로 안내하며, 오픈텔레메트리 데이터를 해석해 문제를 해결하는 방법을
보여준다.

## 참조 {#reference}

요구사항 및 기능 매트릭스와 같은 프로젝트 참조 문서이다.

- [아키텍처](architecture/)
- [개발](development/)
- [기능 플래그 참조](feature-flags/)
- [메트릭 기능 매트릭스](telemetry-features/metric-coverage/)
- [요구사항](./requirements/)
- [스크린샷](screenshots/)
- [서비스](services/)
- [스팬 속성 참조](telemetry-features/manual-span-attributes/)
- [테스트](tests/)
- [트레이스 기능 매트릭스](telemetry-features/trace-coverage/)
