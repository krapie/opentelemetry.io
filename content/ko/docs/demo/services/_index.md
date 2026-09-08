---
title: 서비스
aliases: [service_table, service-table]
default_lang_commit: ffef14de849130bdf9ecd9d4912e75f5a8afdbfd
---

요청 흐름을 시각화하려면 [서비스 다이어그램](../architecture/)을 참고한다.

| 서비스                                | 언어       | 설명                                                                                                                                |
| ------------------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| [accounting](accounting/)             | .NET       | 들어오는 주문을 처리하고 모든 주문의 합계를 계산한다(mock/).                                                                        |
| [ad](ad/)                             | Java       | 주어진 컨텍스트 단어를 기반으로 텍스트 광고를 제공한다.                                                                             |
| [agent](agent/)                       | Python     | 내장 도구 또는 MCP가 제공하는 상점 도구를 사용하는 LangGraph 에이전트를 통해 사용자 프롬프트를 라우팅하는 AI 어시스턴트를 제공한다. |
| [cart](cart/)                         | .NET       | 사용자의 장바구니에 담긴 상품을 Valkey에 저장하고 조회한다.                                                                         |
| [chatbot](chatbot/)                   | Python     | 사용자 메시지를 에이전트 서비스로 전달하는 Gradio 기반 채팅 UI를 제공한다.                                                          |
| [checkout](checkout/)                 | Go         | 사용자의 장바구니를 조회하고 주문을 준비하며, 결제, 배송, 이메일 알림을 오케스트레이션한다.                                         |
| [currency](currency/)                 | C++        | 한 화폐 금액을 다른 통화로 변환한다. 유럽중앙은행(European Central Bank)에서 가져온 실제 값을 사용한다. QPS가 가장 높은 서비스이다. |
| [email](email/)                       | Ruby       | 사용자에게 주문 확인 이메일을 전송한다(mock/).                                                                                      |
| [flagd-ui](flagd-ui/)                 | Elixir     | 기능 플래그(feature flag)를 토글하고 편집할 수 있게 해준다.                                                                         |
| [fraud-detection](fraud-detection/)   | Kotlin     | 들어오는 주문을 분석해 사기 시도를 탐지한다(mock/).                                                                                 |
| [frontend](frontend/)                 | TypeScript | 웹사이트를 제공하는 HTTP 서버를 노출한다. 가입이나 로그인이 필요 없으며, 모든 사용자에 대해 세션 ID를 자동으로 생성한다.            |
| [load-generator](load-generator/)     | Go/k6      | 실제와 유사한 사용자 쇼핑 흐름을 모방한 요청을 프론트엔드로 지속적으로 전송한다.                                                    |
| [mcp](mcp/)                           | Python     | 에이전트와 다른 MCP 호환 클라이언트를 위해 Model Context Protocol을 통해 상점 도구를 노출한다.                                      |
| [payment](payment/)                   | JavaScript | 주어진 신용카드 정보로(mock/) 지정된 금액을 청구하고 트랜잭션 ID를 반환한다.                                                        |
| [product-catalog](product-catalog/)   | Go         | JSON 파일에서 상품 목록을 제공하며, 상품을 검색하고 개별 상품을 조회하는 기능을 제공한다.                                           |
| [quote](quote/)                       | PHP        | 배송할 상품 수를 기준으로 배송 비용을 계산한다.                                                                                     |
| [recommendation](recommendation/)     | Python     | 장바구니에 담긴 상품을 기반으로 다른 상품을 추천한다.                                                                               |
| [shipping](shipping/)                 | Rust       | 장바구니를 기반으로 배송 비용을 추정한다. 지정된 주소로 상품을 배송한다(mock/).                                                     |
| [react-native-app](react-native-app/) | TypeScript | 쇼핑 서비스 위에 UI를 제공하는 React Native 모바일 애플리케이션이다.                                                                |
