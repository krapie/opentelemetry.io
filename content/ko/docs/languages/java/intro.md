---
title: 오픈텔레메트리(OpenTelemetry) Java 소개
description: 오픈텔레메트리(OpenTelemetry) Java 생태계 소개
weight: 9
default_lang_commit: 4edfbfc2ff38123678ca63eca95de94ede457623
---

오픈텔레메트리(OpenTelemetry) Java는 Java 생태계를 위한 오픈텔레메트리
옵저버빌리티(observability) 도구 모음이다. 상위 수준에서 이는 API, SDK,
계측(instrumentation)으로 구성된다.

이 페이지는 개념적 [개요](#overview), [문서 탐색](#navigating-the-docs) 안내,
릴리스와 아티팩트에 대한 핵심 세부 정보를 담은 [저장소](#repositories) 목록과
함께 생태계를 소개한다.

## 개요 {#overview}

API는 핵심 옵저버빌리티 시그널 전반에 걸쳐 텔레메트리를 기록하기 위한 클래스와
인터페이스의 집합이다. 여러 구현체를 지원하도록 설계되었으며, 기본으로
오버헤드가 낮은 최소한의 무동작(no-op) 구현체와 SDK 참조 구현체를 제공한다.
계측을 추가하려는 라이브러리, 프레임워크, 애플리케이션 소유자가 직접적인
의존성으로 가져다 쓰도록 설계되었다. 강력한 하위 호환성 보장, 전이 의존성
없음(zero transitive dependencies)을 제공하며, Java 8 이상을 지원한다.

SDK는 API의 내장 참조 구현체로, 계측 API 호출로 생성된 텔레메트리를 처리하고
내보낸다. SDK를 적절하게 처리하고 내보내도록 구성하는 것은 오픈텔레메트리를
애플리케이션에 통합하는 데 필수적인 단계이다. SDK는
자동구성(autoconfiguration)과 프로그래밍 방식 구성 옵션을 제공한다.

계측은 API를 사용해 텔레메트리를 기록한다. 제로 코드 Java 에이전트, 제로 코드
Spring Boot 스타터, 라이브러리, 네이티브, 수동, 심(shim) 등 다양한 범주의 계측이
있다.

언어에 종속되지 않는 개요는 [오픈텔레메트리 개념](/docs/concepts/)을 참고한다.

## 문서 탐색 {#navigating-the-docs}

오픈텔레메트리 Java 문서는 다음과 같이 구성된다.

- [예제로 시작하기](../getting-started/): 간단한 웹 애플리케이션에
  오픈텔레메트리 Java 에이전트를 통합하는 것을 보여주는, 오픈텔레메트리 Java로
  빠르게 시작할 수 있는 간단한 예제이다.
- [계측 생태계](../instrumentation/): 오픈텔레메트리 Java 계측 생태계에 대한
  안내이다. 애플리케이션에 오픈텔레메트리 Java를 통합하려는 애플리케이션
  작성자에게 핵심 리소스이다. 다양한 범주의 계측에 대해 배우고, 어떤 것이
  적합한지 결정한다.
- [API로 텔레메트리 기록하기](../api/): 동작하는 코드 예제와 함께 API의 모든
  핵심 측면을 탐구하는, 오픈텔레메트리 API에 대한 기술 참조이다. 대부분의
  사용자는 처음부터 끝까지 읽기보다는, 필요할 때 절의 색인을 참고하며
  백과사전처럼 이 페이지를 사용한다.
- [SDK로 텔레메트리 관리하기](../sdk/) 동작하는 코드 예제와 함께 모든 SDK
  플러그인 확장 지점과 프로그래밍 방식 구성 API를 탐구하는, 오픈텔레메트리 SDK에
  대한 기술 참조이다. 대부분의 사용자는 처음부터 끝까지 읽기보다는, 필요할 때
  절의 색인을 참고하며 백과사전처럼 이 페이지를 사용한다.
- [SDK 구성하기](../configuration/): 제로 코드 자동구성에 초점을 맞춘, SDK
  구성에 대한 기술 참조이다. SDK를 구성하기 위해 지원되는 모든 환경 변수와
  시스템 속성의 참조를 포함한다. 동작하는 코드 예제와 함께 모든 프로그래밍 방식
  커스터마이징 지점을 탐구한다. 대부분의 사용자는 처음부터 끝까지 읽기보다는,
  필요할 때 절의 색인을 참고하며 백과사전처럼 이 페이지를 사용한다.
- **더 알아보기**: 엔드투엔드 [예제](../examples/), [Javadoc](../api/), 구성
  요소 [레지스트리](../registry/),
  [성능 참조](/docs/zero-code/java/agent/performance/)를 포함한 보충 리소스이다.

## 저장소 {#repositories}

오픈텔레메트리 Java 소스 코드는 여러 저장소로 구성된다.

| 저장소                                                                                                     | 설명                                                                         | Group ID                           | 현재 버전                            | 릴리스 주기                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| [opentelemetry-java](https://github.com/open-telemetry/opentelemetry-java)                                 | 핵심 API와 SDK 구성 요소                                                     | `io.opentelemetry`                 | `{{% param vers.otel %}}`            | [매월 첫 번째 월요일 다음 금요일](https://github.com/open-telemetry/opentelemetry-java/blob/main/RELEASING.md#release-cadence)                 |
| [opentelemetry-java-instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation) | 오픈텔레메트리 Java 에이전트를 포함하여, 오픈텔레메트리가 유지 관리하는 계측 | `io.opentelemetry.instrumentation` | `{{% param vers.instrumentation %}}` | [매월 두 번째 월요일 다음 수요일](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/RELEASING.md#release-cadence) |
| [opentelemetry-java-contrib](https://github.com/open-telemetry/opentelemetry-java-contrib)                 | 다른 저장소의 명시적 범위에 맞지 않는, 커뮤니티가 유지 관리하는 구성 요소    | `io.opentelemetry.contrib`         | `{{% param vers.contrib %}}`         | [매월 두 번째 월요일 다음 금요일](https://github.com/open-telemetry/opentelemetry-java-contrib/blob/main/RELEASING.md#release-cadence)         |
| [semantic-conventions-java](https://github.com/open-telemetry/semantic-conventions-java)                   | 시맨틱 컨벤션을 위해 생성된 코드                                             | `io.opentelemetry.semconv`         | `{{% param vers.semconv %}}`         | [semantic-conventions](https://github.com/open-telemetry/semantic-conventions)의 릴리스를 뒤따름                                               |
| [opentelemetry-proto-java](https://github.com/open-telemetry/opentelemetry-proto-java)                     | OTLP를 위해 생성된 바인딩                                                    | `io.opentelemetry.proto`           | `1.3.2-alpha`                        | [opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto)의 릴리스를 뒤따름                                                 |
| [opentelemetry-java-examples](https://github.com/open-telemetry/opentelemetry-java-examples)               | API, SDK, 계측을 사용하는 다양한 패턴을 보여주는 엔드투엔드 코드 예제        | 없음                               | 없음                                 | 없음                                                                                                                                           |

`opentelemetry-java`, `opentelemetry-java-instrumentation`,
`opentelemetry-java-contrib`는 각각 방대한 아티팩트 카탈로그를 게시한다. 자세한
내용은 각 저장소를 참고하거나, 관리 대상 의존성의 전체 목록을 보려면
[Bill of Materials](#dependencies-and-boms) 표의 "관리 대상 의존성" 열을
참고한다.

일반적으로 동일한 저장소에서 게시된 아티팩트는 동일한 버전을 가진다. 예외는
`opentelemetry-java-contrib`로, 공유 도구를 활용하기 위해 동일한 저장소에 함께
위치한 독립 프로젝트들의 모음으로 볼 수 있다. 현재
`opentelemetry-java-contrib`의 아티팩트들은 버전이 일치하지만, 이는 우연이며
향후 변경될 수 있다.

저장소들은 상위 수준의 의존성 구조를 반영하는 릴리스 주기를 가진다.

- `opentelemetry-java`는 핵심이며 매월 가장 먼저 릴리스된다.
- `opentelemetry-java-instrumentation`은 `opentelemetry-java`에 의존하며,
  다음으로 게시된다.
- `opentelemetry-java-contrib`는 `opentelemetry-java-instrumentation`과
  `opentelemetry-java`에 의존하며, 마지막으로 게시된다.
- `semantic-conventions-java`는 `opentelemetry-java-instrumentation`의
  의존성이지만, 독립적인 릴리스 일정을 가진 독립적인 아티팩트이다.

## 의존성과 BOM {#dependencies-and-boms}

[bill of materials](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Bill_of_Materials_.28BOM.29_POMs)(약칭
BOM)는 관련 의존성들의 버전을 일치시키는 데 도움을 주는 아티팩트이다.
오픈텔레메트리 Java는 다양한 사용 사례를 위한 여러 BOM을 게시하며, 아래에 범위가
넓어지는 순서로 나열되어 있다. BOM을 사용할 것을 강력히 권장한다.

> [!NOTE]
>
> BOM은 계층적이므로, 여러 BOM에 대한 의존성을 추가하는 것은 권장하지 않는다.
> 이는 중복이며 직관적이지 않은 의존성 버전 해석으로 이어질 수 있다.

BOM이 관리하는 아티팩트 목록을 보려면 "관리 대상 의존성" 열의 링크를 클릭한다.

| 설명                                                                           | 저장소                               | Group ID                           | Artifact ID                               | 현재 버전                                  | 관리 대상 의존성                                        |
| ------------------------------------------------------------------------------ | ------------------------------------ | ---------------------------------- | ----------------------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| 안정적인 핵심 API와 SDK 아티팩트                                               | `opentelemetry-java`                 | `io.opentelemetry`                 | `opentelemetry-bom`                       | `{{% param vers.otel %}}`                  | [최신 pom.xml][opentelemetry-bom]                       |
| `opentelemetry-bom`의 모든 것을 포함한, 실험적인 핵심 API와 SDK 아티팩트       | `opentelemetry-java`                 | `io.opentelemetry`                 | `opentelemetry-bom-alpha`                 | `{{% param vers.otel %}}-alpha`            | [최신 pom.xml][opentelemetry-bom-alpha]                 |
| `opentelemetry-bom`의 모든 것을 포함한, 안정적인 계측 아티팩트                 | `opentelemetry-java-instrumentation` | `io.opentelemetry.instrumentation` | `opentelemetry-instrumentation-bom`       | `{{% param vers.instrumentation %}}`       | [최신 pom.xml][opentelemetry-instrumentation-bom]       |
| `opentelemetry-instrumentation-bom`의 모든 것을 포함한, 실험적인 계측 아티팩트 | `opentelemetry-java-instrumentation` | `io.opentelemetry.instrumentation` | `opentelemetry-instrumentation-bom-alpha` | `{{% param vers.instrumentation %}}-alpha` | [최신 pom.xml][opentelemetry-instrumentation-alpha-bom] |

다음 코드 스니펫은 BOM 의존성을 추가하는 방법을 보여주며, `{{bomGroupId}}`,
`{{bomArtifactId}}`, `{{bomVersion}}`은 각각 표의 "Group ID", "Artifact ID",
"현재 버전" 열을 가리킨다.

{{< tabpane text=true >}} {{% tab "Gradle" %}}

```kotlin
dependencies {
  implementation(platform("{{bomGroupId}}:{{bomArtifactId}}:{{bomVersion}}"))
  // Add a dependency on an artifact whose version is managed by the bom
  implementation("io.opentelemetry:opentelemetry-api")
}
```

{{% /tab %}} {{% tab Maven %}}

```xml
<project>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>{{bomGroupId}}</groupId>
        <artifactId>{{bomArtifactId}}</artifactId>
        <version>{{bomVersion}}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <!-- Add a dependency on an artifact whose version is managed by the bom -->
  <dependencies>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-api</artifactId>
    </dependency>
  </dependencies>
</project>
```

{{% /tab %}} {{< /tabpane >}}

[opentelemetry-bom]:
  <https://repo1.maven.org/maven2/io/opentelemetry/opentelemetry-bom/{{% param vers.otel %}}/opentelemetry-bom-{{% param vers.otel %}}.pom>
[opentelemetry-bom-alpha]:
  <https://repo1.maven.org/maven2/io/opentelemetry/opentelemetry-bom-alpha/{{% param vers.otel %}}-alpha/opentelemetry-bom-alpha-{{% param vers.otel %}}-alpha.pom>
[opentelemetry-instrumentation-bom]:
  <https://repo1.maven.org/maven2/io/opentelemetry/instrumentation/opentelemetry-instrumentation-bom/{{% param vers.instrumentation %}}/opentelemetry-instrumentation-bom-{{% param vers.instrumentation %}}.pom>
[opentelemetry-instrumentation-alpha-bom]:
  <https://repo1.maven.org/maven2/io/opentelemetry/instrumentation/opentelemetry-instrumentation-bom-alpha/{{% param vers.instrumentation %}}-alpha/opentelemetry-instrumentation-bom-alpha-{{% param vers.instrumentation %}}-alpha.pom>
