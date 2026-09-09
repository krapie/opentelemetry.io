---
title: 개발
cSpell:ignore: grpcio intellij libcurl libprotobuf nlohmann openssl protoc
default_lang_commit: 7c112ea4147756b8234012f9316132a65a77593c
---

[오픈텔레메트리 데모 GitHub 저장소](https://github.com/open-telemetry/opentelemetry-demo)

이 데모를 개발하려면 여러 프로그래밍 언어의 도구가 필요하다. 가능한 경우 최소
요구 버전을 명시하지만, 모든 도구를 최신 버전으로 업데이트하는 것을 권장한다.
오픈텔레메트리 데모 팀은 가능한 한 이 저장소의 서비스들을 의존성(dependency) 및
도구의 최신 버전으로 유지하기 위해 노력한다.

## protobuf 파일 생성 {#generate-protobuf-files}

모든 서비스의 protobuf 파일을 생성하기 위해 `make generate-protobuf` 명령어가
제공된다. 이 명령어는 (Docker 없이) 로컬에서 코드를 컴파일하고 IntelliJ나 VS
Code와 같은 IDE에서 힌트를 받는 데 사용할 수 있다. 파일을 생성하기 전에 frontend
소스 폴더 내에서 `npm install`을 실행해야 할 수도 있다.

## 개발 도구 요구 사항 {#development-tooling-requirements}

### .NET {#net}

- .NET 8.0+

### C++ {#c}

- build-essential
- cmake
- libcurl4-openssl-dev
- libprotobuf-dev
- nlohmann-json3-dev
- pkg-config
- protobuf-compiler

### Go {#go}

- Go 1.19+
- protoc-gen-go
- protoc-gen-go-grpc

### Java {#java}

- JDK 17+
- Gradle 7+

### JavaScript {#javascript}

- Node.js 16+

### PHP {#php}

- PHP 8.1+
- Composer 2.4+

### Python {#python}

- Python 3.10
- grpcio-tools 1.48+

### Ruby {#ruby}

- Ruby 3.1+

### Rust {#rust}

- Rust 1.61+
- protoc 3.21+
- protobuf-dev
