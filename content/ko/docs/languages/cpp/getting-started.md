---
title: 예제로 시작하기
description: 5분 안에 앱의 텔레메트리를 확보해 보자!
weight: 10
cSpell:ignore: oatpp rolldice
default_lang_commit: 76da9fb03296c7f2de8c9ba8bf61d5becb0731f1
---

이 페이지는 C++에서 오픈텔레메트리(OpenTelemetry)를 시작하는 방법을 보여준다.

간단한 C++ 애플리케이션을 계측하여 [트레이스](/docs/concepts/signals/traces/)가
터미널로 방출되도록 만드는 방법을 배운다.

## 사전 요구 사항 {#prerequisites}

다음이 로컬에 설치되어 있는지 확인한다.

- Git
- C++ 버전 14 이상을 지원하는 C++ 컴파일러
- Make
- CMake 버전 3.25 이상

## 예제 애플리케이션 {#example-application}

다음 예제는 기본적인 [Oat++](https://oatpp.io/) 애플리케이션을 사용한다. Oat++를
사용하지 않아도 괜찮다. OpenTelemetry C++는 다른 어떤 웹 프레임워크와도 함께
사용할 수 있다.

## 설정 {#setup}

- `otel-cpp-starter`라는 이름의 폴더를 생성한다.
- 새로 생성한 폴더로 이동한다. 이 폴더가 작업 디렉터리 역할을 한다.
- 의존성을 설정하고 나면, 디렉터리 구조는 다음과 비슷해야 한다.

  ```plaintext
  otel-cpp-starter
  │
  ├── oatpp
  ├── opentelemetry-cpp
  └── roll-dice
  ```

## 의존성 {#dependencies}

먼저 [소스 코드](https://github.com/oatpp)와 `make`를 사용해 Oat++를 로컬에
설치한다. 다음 단계를 따른다.

1. [oatpp/oatpp](https://github.com/oatpp/oatpp) GitHub 저장소를 클론하여 Oat++
   소스 코드를 받는다.

   ```bash
   git clone https://github.com/oatpp/oatpp.git
   ```

2. `oatpp` 디렉터리로 이동하고, 우선 1.3.0 버전으로 전환한다.

   ```bash
   cd oatpp
   git checkout 1.3.0-latest
   ```

3. `build` 하위 디렉터리를 생성하고 그 안으로 이동한다.

   ```bash
   mkdir build
   cd build
   ```

4. `cmake`와 `make` 명령을 사용해 oatpp를 빌드한다. 이 명령은 oatpp 소스 코드에
   포함된 `CMakeLists.txt`에 지정된 빌드 프로세스를 실행한다.

   ```bash
   cmake ..
   make
   ```

5. oatpp를 로컬 프리픽스에 설치한다. 이 명령은 빌드된 oatpp 라이브러리, 헤더,
   CMake 패키지 구성을 `install` 디렉터리에 설치하여 개발에서 사용할 수 있도록
   만든다.

   ```bash
   cmake --install . --prefix ../../install
   ```

다음으로, CMake를 사용해
[OpenTelemetry C++](https://github.com/open-telemetry/opentelemetry-cpp)를
로컬에 설치하고 빌드한다. 다음 단계를 따른다.

1. 터미널에서 `otel-cpp-starter` 디렉터리로 다시 이동한다. 그런 다음
   OpenTelemetry C++ GitHub 저장소를 로컬 머신에 클론한다.

   ```bash
   git clone https://github.com/open-telemetry/opentelemetry-cpp.git
   ```

2. 작업 디렉터리를 OpenTelemetry C++ SDK 디렉터리로 변경한다.

   ```bash
   cd opentelemetry-cpp
   ```

3. build 디렉터리를 생성하고 그 안으로 이동한다.

   ```bash
   mkdir build
   cd build
   ```

4. `build` 디렉터리에서 CMake를 실행하여, 테스트를 활성화하지 않은 채로 빌드
   시스템을 구성하고 생성한다.

   ```bash
   cmake -DBUILD_TESTING=OFF ..
   ```

   또는 `cmake --build`가 실패한다면 다음도 시도해 볼 수 있다.

   ```bash
   cmake -DBUILD_TESTING=OFF -DWITH_ABSEIL=ON ..
   ```

5. 빌드 프로세스를 실행한다.

   ```bash
   cmake --build .
   ```

6. OpenTelemetry C++를 oatpp와 동일한 로컬 프리픽스에 설치한다.

   ```bash
   cmake --install . --prefix ../../install
   ```

Oat++와 OpenTelemetry C++가 준비되었으니, 이제 계측하고자 하는 HTTP 서버를
만드는 단계로 넘어갈 수 있다.

## HTTP 서버 생성 및 실행 {#create-and-launch-an-http-server}

`otel-cpp-starter` 폴더 안에 `roll-dice` 하위 폴더를 생성한다. 이 폴더에서는
oatpp 헤더를 참조하고 프로젝트를 컴파일할 때 링크함으로써 Oat++ 라이브러리를
사용한다.

`roll-dice` 안에 `CMakeLists.txt` 파일을 만들어, Oat++ 라이브러리 디렉터리와
include 경로를 정의하고, 컴파일 과정에서 Oat++를 링크하도록 한다.

```cmake
cmake_minimum_required(VERSION 3.25)
project(RollDiceServer)
# Set C++ standard (e.g., C++17)
set(CMAKE_CXX_STANDARD 17)
set(project_name roll-dice-server)

# Define your project's source files
set(SOURCES
    main.cpp  # Add your source files here
)

# Create an executable target
add_executable(dice-server ${SOURCES})

find_package(oatpp REQUIRED)

target_link_libraries(dice-server PRIVATE oatpp::oatpp)
```

다음으로, 샘플 HTTP 서버 소스 코드가 필요하다. 이 코드는 다음을 수행한다.

- HTTP 라우터를 초기화하고, `/rolldice` 엔드포인트로 GET 요청이 들어오면
  응답으로 임의의 숫자를 생성하는 요청 핸들러를 설정한다.
- 이어서 커넥션 핸들러와 커넥션 프로바이더를 생성하고,
  <http://localhost:8080>에서 서버를 시작한다.
- 마지막으로, main 함수 안에서 애플리케이션을 초기화하고 실행한다.

`roll-dice` 폴더에 `main.cpp` 파일을 만들고 다음 코드를 추가한다.

```cpp
#include "oatpp/web/server/HttpConnectionHandler.hpp"
#include "oatpp/network/Server.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"
#include <cstdlib>
#include <ctime>
#include <string>

using namespace std;

class Handler : public oatpp::web::server::HttpRequestHandler {
public:
  shared_ptr<OutgoingResponse> handle(const shared_ptr<IncomingRequest>& request) override {
    int low = 1;
    int high = 7;
    int random = rand() % (high - low) + low;
    // Convert a std::string to oatpp::String
    const string response = to_string(random);
    return ResponseFactory::createResponse(Status::CODE_200, response.c_str());
  }
};

void run() {
  auto router = oatpp::web::server::HttpRouter::createShared();
  router->route("GET", "/rolldice", std::make_shared<Handler>());
  auto connectionHandler = oatpp::web::server::HttpConnectionHandler::createShared(router);
  auto connectionProvider = oatpp::network::tcp::server::ConnectionProvider::createShared({"localhost", 8080, oatpp::network::Address::IP_4});
  oatpp::network::Server server(connectionProvider, connectionHandler);
  OATPP_LOGI("Dice Server", "Server running on port %s", static_cast<const char*>(connectionProvider->getProperty("port").getData()));
  server.run();
}

int main() {
  oatpp::base::Environment::init();
  srand((int)time(0));
  run();
  oatpp::base::Environment::destroy();
  return 0;
}
```

다음 CMake 명령으로 애플리케이션을 빌드하고 실행한다.

```bash
mkdir build
cd build
cmake .. -DCMAKE_PREFIX_PATH=$(pwd)/../../install
cmake --build .
```

프로젝트 빌드에 성공하면, 생성된 실행 파일을 실행할 수 있다.

```bash
./dice-server
```

그런 다음 브라우저에서 <http://localhost:8080/rolldice>를 열어 제대로 동작하는지
확인한다.

## 계측 {#instrumentation}

애플리케이션에 오픈텔레메트리를 추가하려면, `CMakeLists.txt` 파일을 다음과 같이
추가 의존성으로 업데이트한다.

```cmake
cmake_minimum_required(VERSION 3.25)
project(RollDiceServer)
# Set C++ standard (e.g., C++17)
set(CMAKE_CXX_STANDARD 17)
set(project_name roll-dice-server)

# Define your project's source files
set(SOURCES
    main.cpp  # Add your source files here
)

# Create an executable target
add_executable(dice-server ${SOURCES})

find_package(oatpp REQUIRED)
find_package(opentelemetry-cpp CONFIG REQUIRED)

target_link_libraries(dice-server PRIVATE
                      oatpp::oatpp
                      ${OPENTELEMETRY_CPP_LIBRARIES})
```

`/rolldice` 요청 핸들러가 호출될 때 트레이서를 초기화하고 스팬을 방출하도록,
`main.cpp` 파일을 다음 코드로 업데이트한다.

```cpp
#include "oatpp/web/server/HttpConnectionHandler.hpp"
#include "oatpp/network/Server.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"

#include "opentelemetry/exporters/ostream/span_exporter_factory.h"
#include "opentelemetry/sdk/trace/exporter.h"
#include "opentelemetry/sdk/trace/processor.h"
#include "opentelemetry/sdk/trace/simple_processor_factory.h"
#include "opentelemetry/sdk/trace/tracer_provider_factory.h"
#include "opentelemetry/trace/provider.h"

#include <cstdlib>
#include <ctime>
#include <string>

using namespace std;
namespace trace_api = opentelemetry::trace;
namespace trace_sdk = opentelemetry::sdk::trace;
namespace trace_exporter = opentelemetry::exporter::trace;

namespace {
  void InitTracer() {
    auto exporter  = trace_exporter::OStreamSpanExporterFactory::Create();
    auto processor = trace_sdk::SimpleSpanProcessorFactory::Create(std::move(exporter));
    std::shared_ptr<opentelemetry::trace::TracerProvider> provider =
      trace_sdk::TracerProviderFactory::Create(std::move(processor));
    //set the global trace provider
    trace_api::Provider::SetTracerProvider(provider);
  }
  void CleanupTracer() {
    std::shared_ptr<opentelemetry::trace::TracerProvider> none;
    trace_api::Provider::SetTracerProvider(none);
  }

}

class Handler : public oatpp::web::server::HttpRequestHandler {
public:
  shared_ptr<OutgoingResponse> handle(const shared_ptr<IncomingRequest>& request) override {
    auto tracer = opentelemetry::trace::Provider::GetTracerProvider()->GetTracer("my-app-tracer");
    auto span = tracer->StartSpan("RollDiceServer");
    int low = 1;
    int high = 7;
    int random = rand() % (high - low) + low;
    // Convert a std::string to oatpp::String
    const string response = to_string(random);
    span->End();
    return ResponseFactory::createResponse(Status::CODE_200, response.c_str());
  }
};

void run() {
  auto router = oatpp::web::server::HttpRouter::createShared();
  router->route("GET", "/rolldice", std::make_shared<Handler>());
  auto connectionHandler = oatpp::web::server::HttpConnectionHandler::createShared(router);
  auto connectionProvider = oatpp::network::tcp::server::ConnectionProvider::createShared({"localhost", 8080, oatpp::network::Address::IP_4});
  oatpp::network::Server server(connectionProvider, connectionHandler);
  OATPP_LOGI("Dice Server", "Server running on port %s", static_cast<const char*>(connectionProvider->getProperty("port").getData()));
  server.run();
}

int main() {
  oatpp::base::Environment::init();
  InitTracer();
  srand((int)time(0));
  run();
  oatpp::base::Environment::destroy();
  CleanupTracer();
  return 0;
}
```

프로젝트를 다시 빌드한다.

```bash
cd build
cmake .. -DCMAKE_PREFIX_PATH=$(pwd)/../../install
cmake --build .
```

프로젝트 빌드에 성공하면, 생성된 실행 파일을 실행할 수 있다.

```bash
./dice-server
```

<http://localhost:8080/rolldice>로 요청을 보내면, 터미널에 스팬이 방출되는 것을
볼 수 있다.

```json
{
  "name" : "RollDiceServer",
  "trace_id": "f47bea385dc55e4d17470d51f9d3130b",
  "span_id": "deed994b51f970fa",
  "tracestate" : ,
  "parent_span_id": "0000000000000000",
  "start": 1698991818716461000,
  "duration": 64697,
  "span kind": "Internal",
  "status": "Unset",
  "service.name": "unknown_service",
  "telemetry.sdk.language": "cpp",
  "telemetry.sdk.name": "opentelemetry",
  "telemetry.sdk.version": "1.11.0",
  "instr-lib": "my-app-tracer"
}
```

## 다음 단계 {#next-steps}

코드 계측에 대해 더 자세히 알아보려면
[계측](/docs/languages/cpp/instrumentation) 문서를 참고한다.

또한 텔레메트리 백엔드 한 곳 이상으로
[텔레메트리 데이터를 내보내기](/docs/languages/cpp/exporters/) 위해 적절한
익스포터를 구성해야 할 것이다.
