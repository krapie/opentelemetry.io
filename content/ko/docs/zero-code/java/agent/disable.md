---
title: 특정 계측 억제하기
linkTitle: 계측 억제하기
weight: 12
# prettier-ignore
cSpell:ignore: activej akka armeria avaje clickhouse couchbase datasource dbcp Dotel dropwizard dubbo elasticjob finatra helidon hikari hikaricp httpasyncclient httpclient hystrix javalin jaxrs jaxws jedis jfinal jodd kotlinx ktor logmanager mojarra mybatis myfaces nats okhttp openai oshi payara pekko powerjob rabbitmq ratpack rediscala redisson restlet rocketmq shenyu spymemcached twilio vaadin vertx vibur webflux webmvc
default_lang_commit: 368f811f81c27798a031b4c92024ecdd65cddc19
---

## 에이전트 전체 비활성화하기 {#disabling-the-agent-entirely}

{{% config_option name="otel.javaagent.enabled" %}}

값을 `false`로 설정하면 에이전트가 전체적으로 비활성화된다.

{{% /config_option %}}

## 특정 계측만 활성화하기 {#enable-only-specific-instrumentation}

모든 기본 자동 계측을 비활성화하고 개별 계측을 선택적으로 다시 활성화할 수 있다.
이는 시작 오버헤드를 줄이거나 어떤 계측이 적용될지를 더 세밀하게 제어하고 싶을
때 유용할 수 있다.

{{% config_option name="otel.instrumentation.common.default-enabled" %}}
`false`로 설정하면 에이전트의 모든 계측이 비활성화된다. {{% /config_option %}}

{{% config_option name="otel.instrumentation.[name].enabled" %}} `true`로
설정하면 원하는 각 계측을 개별적으로 활성화할 수 있다. {{% /config_option %}}

> [!WARNING]
>
> 일부 계측은 제대로 동작하기 위해 다른 계측에 의존한다. 계측을 선택적으로
> 활성화할 때는 전이 의존성도 함께 활성화해야 한다. 이 의존 관계를 파악하는 일은
> 사용자의 몫으로 남겨둔다. 이는 고급 사용법으로 간주되며 대부분의 사용자에게는
> 권장되지 않는다.

## 수동 계측만 활성화하기 {#enable-manual-instrumentation-only}

`-Dotel.instrumentation.common.default-enabled=false -Dotel.instrumentation.opentelemetry-api.enabled=true -Dotel.instrumentation.opentelemetry-instrumentation-annotations.enabled=true`를
사용하면, 모든 자동 계측을 억제하면서도 `@WithSpan`을 통한 수동 계측과 일반적인
API 상호작용은 지원되도록 할 수 있다.

## 특정 에이전트 계측 억제하기 {#suppressing-specific-agent-instrumentation}

특정 라이브러리에 대한 에이전트 계측을 억제할 수 있다.

{{% config_option name="otel.instrumentation.[name].enabled" %}} `false`로
설정하면 특정 라이브러리에 대한 에이전트 계측을 억제한다. 이때 `[name]`은 해당
계측 이름이다. {{% /config_option %}}

| 라이브러리/프레임워크                            | 계측 이름                                   |
| ------------------------------------------------ | ------------------------------------------- |
| Additional methods tracing                       | `methods`                                   |
| Additional tracing annotations                   | `external-annotations`                      |
| Activej HTTP                                     | `activej-http`                              |
| Avaje Jex                                        | `avaje-jex`                                 |
| Akka Actor                                       | `akka-actor`                                |
| Akka HTTP                                        | `akka-http`                                 |
| Alibaba Druid                                    | `alibaba-druid`                             |
| Apache Axis2                                     | `axis2`                                     |
| Apache Camel                                     | `camel`                                     |
| Apache Cassandra                                 | `cassandra`                                 |
| Apache CXF                                       | `cxf`                                       |
| Apache DBCP                                      | `apache-dbcp`                               |
| Apache Dubbo                                     | `apache-dubbo`                              |
| Apache ElasticJob                                | `apache-elasticjob`                         |
| Apache Geode                                     | `geode`                                     |
| Apache HttpAsyncClient                           | `apache-httpasyncclient`                    |
| Apache HttpClient                                | `apache-httpclient`                         |
| Apache Iceberg                                   | `iceberg`                                   |
| Apache Kafka                                     | `kafka`                                     |
| Apache MyFaces                                   | `jsf-myfaces`                               |
| Apache Pekko Actor                               | `pekko-actor`                               |
| Apache Pekko HTTP                                | `pekko-http`                                |
| Apache Pulsar                                    | `pulsar`                                    |
| Apache RocketMQ                                  | `rocketmq-client`                           |
| Apache Shenyu                                    | `apache-shenyu`                             |
| Apache Struts 2                                  | `struts`                                    |
| Apache Tapestry                                  | `tapestry`                                  |
| Apache Tomcat                                    | `tomcat`                                    |
| Apache Wicket                                    | `wicket`                                    |
| Armeria                                          | `armeria`                                   |
| AsyncHttpClient (AHC)                            | `async-http-client`                         |
| AWS Lambda                                       | `aws-lambda`                                |
| AWS SDK                                          | `aws-sdk`                                   |
| Azure SDK                                        | `azure-core`                                |
| Clickhouse Client                                | `clickhouse`                                |
| Couchbase                                        | `couchbase`                                 |
| C3P0                                             | `c3p0`                                      |
| Dropwizard Views                                 | `dropwizard-views`                          |
| Dropwizard Metrics                               | `dropwizard-metrics`                        |
| Eclipse Grizzly                                  | `grizzly`                                   |
| Eclipse Jersey                                   | `jersey`                                    |
| Eclipse Jetty                                    | `jetty`                                     |
| Eclipse Jetty HTTP Client                        | `jetty-httpclient`                          |
| Eclipse Metro                                    | `metro`                                     |
| Eclipse Mojarra                                  | `jsf-mojarra`                               |
| Eclipse Vert.x HttpClient                        | `vertx-http-client`                         |
| Eclipse Vert.x Kafka Client                      | `vertx-kafka-client`                        |
| Eclipse Vert.x Redis Client                      | `vertx-redis-client`                        |
| Eclipse Vert.x RxJava                            | `vertx-rx-java`                             |
| Eclipse Vert.x SQL Client                        | `vertx-sql-client`                          |
| Eclipse Vert.x Web                               | `vertx-web`                                 |
| Elasticsearch API client                         | `elasticsearch-api-client`                  |
| Elasticsearch client                             | `elasticsearch-transport`                   |
| Elasticsearch REST client                        | `elasticsearch-rest`                        |
| Failsafe                                         | `failsafe`                                  |
| Finagle                                          | `finagle-http`                              |
| Google Guava                                     | `guava`                                     |
| Google HTTP client                               | `google-http-client`                        |
| Google Web Toolkit                               | `gwt`                                       |
| Grails                                           | `grails`                                    |
| GraphQL Java                                     | `graphql-java`                              |
| GRPC                                             | `grpc`                                      |
| Helidon                                          | `helidon`                                   |
| Hibernate                                        | `hibernate`                                 |
| Hibernate Reactive                               | `hibernate-reactive`                        |
| HikariCP                                         | `hikaricp`                                  |
| InfluxDB                                         | `influxdb`                                  |
| Java HTTP Client                                 | `java-http-client`                          |
| Java HTTP Server                                 | `java-http-server`                          |
| Java `HttpURLConnection`                         | `http-url-connection`                       |
| Java JDBC                                        | `jdbc`                                      |
| Java JDBC `DataSource`                           | `jdbc-datasource`                           |
| Java RMI                                         | `rmi`                                       |
| Java Runtime                                     | `runtime-telemetry`                         |
| Java Servlet                                     | `servlet`                                   |
| java.util.concurrent                             | `executors`                                 |
| java.util.logging                                | `java-util-logging`                         |
| Javalin                                          | `javalin`                                   |
| JAX-RS (Client)                                  | `jaxrs-client`                              |
| JAX-RS (Server)                                  | `jaxrs`                                     |
| JAX-WS                                           | `jaxws`                                     |
| JBoss Logging Appender                           | `jboss-logmanager-appender`                 |
| JBoss Logging MDC                                | `jboss-logmanager-mdc`                      |
| JFinal                                           | `jfinal`                                    |
| JMS                                              | `jms`                                       |
| Jodd HTTP                                        | `jodd-http`                                 |
| JSP                                              | `jsp`                                       |
| K8s Client                                       | `kubernetes-client`                         |
| Ktor                                             | `ktor`                                      |
| kotlinx.coroutines                               | `kotlinx-coroutines`                        |
| Log4j Appender                                   | `log4j-appender`                            |
| Log4j MDC (1.x)                                  | `log4j-mdc`                                 |
| Log4j Context Data (2.x)                         | `log4j-context-data`                        |
| Logback Appender                                 | `logback-appender`                          |
| Logback MDC                                      | `logback-mdc`                               |
| Micrometer                                       | `micrometer`                                |
| MongoDB                                          | `mongo`                                     |
| MyBatis                                          | `mybatis`                                   |
| NATS Client                                      | `nats`                                      |
| Netflix Hystrix                                  | `hystrix`                                   |
| Netty                                            | `netty`                                     |
| OkHttp                                           | `okhttp`                                    |
| OpenLiberty                                      | `liberty`                                   |
| OpenAI                                           | `openai`                                    |
| OpenSearch Java                                  | `opensearch-java`                           |
| OpenSearch REST                                  | `opensearch-rest`                           |
| OpenTelemetry Extension Annotations              | `opentelemetry-extension-annotations`       |
| OpenTelemetry Instrumentation Annotations        | `opentelemetry-instrumentation-annotations` |
| OpenTelemetry API                                | `opentelemetry-api`                         |
| Oracle UCP                                       | `oracle-ucp`                                |
| OSHI (Operating System and Hardware Information) | `oshi`                                      |
| Payara                                           | `payara`                                    |
| Play Framework                                   | `play`                                      |
| Play WS HTTP Client                              | `play-ws`                                   |
| Powerjob                                         | `powerjob`                                  |
| Quarkus                                          | `quarkus`                                   |
| Quartz                                           | `quartz`                                    |
| R2DBC                                            | `r2dbc`                                     |
| RabbitMQ Client                                  | `rabbitmq`                                  |
| Ratpack                                          | `ratpack`                                   |
| ReactiveX RxJava                                 | `rxjava`                                    |
| Reactor                                          | `reactor`                                   |
| Reactor Kafka                                    | `reactor-kafka`                             |
| Reactor Netty                                    | `reactor-netty`                             |
| Redis Jedis                                      | `jedis`                                     |
| Redis Lettuce                                    | `lettuce`                                   |
| Rediscala                                        | `rediscala`                                 |
| Redisson                                         | `redisson`                                  |
| Restlet                                          | `restlet`                                   |
| Scala ForkJoinPool                               | `scala-fork-join`                           |
| Spark Web Framework                              | `spark`                                     |
| Spring Batch                                     | `spring-batch`                              |
| Spring Boot Actuator Autoconfigure               | `spring-boot-actuator-autoconfigure`        |
| Spring Cloud AWS                                 | `spring-cloud-aws`                          |
| Spring Cloud Gateway                             | `spring-cloud-gateway`                      |
| Spring Core                                      | `spring-core`                               |
| Spring Data                                      | `spring-data`                               |
| Spring JMS                                       | `spring-jms`                                |
| Spring Integration                               | `spring-integration`                        |
| Spring Kafka                                     | `spring-kafka`                              |
| Spring Pulsar                                    | `spring-pulsar`                             |
| Spring RabbitMQ                                  | `spring-rabbit`                             |
| Spring RMI                                       | `spring-rmi`                                |
| Spring Scheduling                                | `spring-scheduling`                         |
| Spring Security Config                           | `spring-security-config`                    |
| Spring Web                                       | `spring-web`                                |
| Spring WebFlux                                   | `spring-webflux`                            |
| Spring Web MVC                                   | `spring-webmvc`                             |
| Spring Web Services                              | `spring-ws`                                 |
| Spymemcached                                     | `spymemcached`                              |
| Tomcat JDBC                                      | `tomcat-jdbc`                               |
| Twilio SDK                                       | `twilio`                                    |
| Twitter Finatra                                  | `finatra`                                   |
| Undertow                                         | `undertow`                                  |
| Vaadin                                           | `vaadin`                                    |
| Vibur DBCP                                       | `vibur-dbcp`                                |
| XXL-JOB                                          | `xxl-job`                                   |
| ZIO                                              | `zio`                                       |

**참고:** 환경 변수를 사용할 때는 대시(`-`)를 언더스코어(`_`)로 변환해야 한다.
예를 들어 `akka-actor` 라이브러리의 트레이스를 억제하려면
`OTEL_INSTRUMENTATION_AKKA_ACTOR_ENABLED`를 `false`로 설정한다.

## 컨트롤러 및/또는 뷰 스팬 억제하기 {#suppressing-controller-andor-view-spans}

일부 계측(예: Spring Web MVC 계측)은 컨트롤러 및/또는 뷰 실행을 캡처하기 위해
[SpanKind.Internal](/docs/specs/otel/trace/api/#spankind) 스팬을 생성한다. 계측
전체를 억제하면 `http.route` 캡처와 부모
[SpanKind.Server](/docs/specs/otel/trace/api/#spankind) 스팬의 연관된 스팬
이름도 비활성화되는데, 아래 구성 설정을 사용하면 그렇게 하지 않고도 이 스팬들을
억제할 수 있다.

{{% config_option
name="otel.instrumentation.common.experimental.controller-telemetry.enabled"
default=false
%}} `true`로 설정하면 컨트롤러 텔레메트리가 활성화된다. {{% /config_option %}}

{{% config_option
name="otel.instrumentation.common.experimental.view-telemetry.enabled"
default=false
%}} `true`로 설정하면 뷰 텔레메트리가 활성화된다. {{% /config_option %}}

## 계측 스팬 억제 동작 {#instrumentation-span-suppression-behavior}

이 에이전트가 계측하는 일부 라이브러리는 다시 저수준 라이브러리를 사용하며, 그
저수준 라이브러리도 계측된다. 이는 보통 중복된 텔레메트리 데이터를 담은 중첩된
스팬으로 이어진다. 예를 들어:

- Reactor Netty HTTP 클라이언트 계측이 생성한 스팬에는 Netty 계측이 생성한
  중복된 HTTP 클라이언트 스팬이 포함된다.
- AWS SDK 계측이 생성한 Dynamo DB 스팬에는 내부 HTTP 클라이언트 라이브러리(이
  역시 계측됨)가 생성한 자식 HTTP 클라이언트 스팬이 포함된다.
- Tomcat 계측이 생성한 스팬에는 일반 Servlet API 계측이 생성한 중복된 HTTP 서버
  스팬이 포함된다.

Java 에이전트는 텔레메트리 데이터를 중복시키는 중첩된 스팬을 감지하고 억제하여
이러한 상황을 방지한다. 억제 동작은 다음 구성 옵션으로 구성할 수 있다.

{{% config_option name="otel.instrumentation.experimental.span-suppression-strategy" %}}

Java 에이전트 스팬 억제 전략이다. 다음 3가지 전략이 지원된다.

- `semconv`: 에이전트가 중복된 시맨틱 컨벤션을 억제한다. 이것이 Java 에이전트의
  기본 동작이다.
- `span-kind`: 에이전트가 같은 종류(`INTERNAL` 제외)의 스팬을 억제한다.
- `none`: 에이전트가 아무것도 억제하지 않는다. **이 옵션은 중복된 텔레메트리
  데이터를 많이 생성하므로 디버그 목적 외에는 사용하지 않기를 권장한다**.

{{% /config_option %}}

예를 들어 내부적으로 Reactor Netty HTTP 클라이언트를 사용하는 데이터베이스
클라이언트를 계측하고, 그 Reactor Netty가 다시 Netty를 사용한다고 가정하자.

기본 `semconv` 억제 전략을 사용하면 중첩된 `CLIENT` 스팬 2개가 생긴다.

- 데이터베이스 클라이언트 계측이 방출한, 데이터베이스 클라이언트 시맨틱 속성을
  가진 `CLIENT` 스팬
- Reactor Netty 계측이 방출한, HTTP 클라이언트 시맨틱 속성을 가진 `CLIENT` 스팬

Netty 계측은 Reactor Netty HTTP 클라이언트 계측과 중복되므로 억제된다.

억제 전략 `span-kind`를 사용하면 스팬이 하나만 생긴다.

- 데이터베이스 클라이언트 계측이 방출한, 데이터베이스 클라이언트 시맨틱 속성을
  가진 `CLIENT` 스팬

Reactor Netty와 Netty 계측은 둘 다 `CLIENT` 스팬을 방출하므로 모두 억제된다.

마지막으로, 억제 전략 `none`을 사용하면 스팬 3개가 생긴다.

- 데이터베이스 클라이언트 계측이 방출한, 데이터베이스 클라이언트 시맨틱 속성을
  가진 `CLIENT` 스팬
- Reactor Netty 계측이 방출한, HTTP 클라이언트 시맨틱 속성을 가진 `CLIENT` 스팬
- Netty 계측이 방출한, HTTP 클라이언트 시맨틱 속성을 가진 `CLIENT` 스팬
