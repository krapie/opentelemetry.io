---
title: 리소스
weight: 70
description: 애플리케이션 환경에 대한 세부 정보를 텔레메트리에 추가하기
cSpell:ignore: myhost SIGINT uuidgen WORKDIR
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

{{% docs/languages/resources-intro %}}

아래에서 Node.js SDK로 리소스 감지(resource detection)를 설정하는 방법에 대한
소개를 확인할 수 있다.

## 설정 {#setup}

[시작하기 - Node.js][]의 안내를 따라 `package.json`, `app.js`(또는 `app.ts`),
`instrumentation.mjs`(또는 `instrumentation.ts`) 파일을 준비한다.

{{% include esm-support-note.md %}}

## 프로세스 및 환경 리소스 감지 {#process--environment-resource-detection}

기본적으로 Node.js SDK는 [프로세스 및 프로세스 런타임 리소스][]를 감지하고,
`OTEL_RESOURCE_ATTRIBUTES` 환경 변수에서 속성을 가져온다. 계측 파일에서 진단
로깅을 켜면 무엇이 감지되는지 확인할 수 있다.

```javascript
// For troubleshooting, set the log level to DiagLogLevel.DEBUG
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG);
```

`OTEL_RESOURCE_ATTRIBUTES`에 값을 설정한 상태로 애플리케이션을 실행한다. 예를
들어 [Host][]를 식별하기 위해 `host.name`을 설정한다.

```console
$ env OTEL_RESOURCE_ATTRIBUTES="host.name=localhost" \
  node --import ./instrumentation.mjs app.js
@opentelemetry/api: Registered a global for diag v1.2.0.
...
Listening for requests on http://localhost:8080
EnvDetector found resource. Resource { attributes: { 'host.name': 'localhost' } }
ProcessDetector found resource. Resource {
  attributes: {
    'process.pid': 12345,
    'process.executable.name': 'node',
    'process.command': '/app.js',
    'process.command_line': '/bin/node /app.js',
    'process.runtime.version': '16.17.0',
    'process.runtime.name': 'nodejs',
    'process.runtime.description': 'Node.js'
  }
}
...
```

## 환경 변수로 리소스 추가하기 {#adding-resources-with-environment-variables}

위 예시에서 SDK는 프로세스를 감지했을 뿐만 아니라, 환경 변수로 설정한
`host.name=localhost` 속성도 자동으로 추가했다.

아래에서 리소스가 자동으로 감지되도록 하는 방법을 안내한다. 다만 필요한 리소스에
대한 디텍터가 존재하지 않는 상황이 발생할 수 있다. 이럴 때는
`OTEL_RESOURCE_ATTRIBUTES` 환경 변수를 사용해 필요한 값을 주입한다. 또한
`OTEL_SERVICE_NAME` 환경 변수를 사용해 `service.name` 리소스 속성의 값을 설정할
수도 있다. 예를 들어 다음 스크립트는 [Service][], [Host][], [OS][] 리소스 속성을
추가한다.

```console
$ env OTEL_SERVICE_NAME="app.js" OTEL_RESOURCE_ATTRIBUTES="service.namespace=tutorial,service.version=1.0,service.instance.id=`uuidgen`,host.name=${HOSTNAME},host.type=`uname -m`,os.name=`uname -s`,os.version=`uname -r`" \
  node --import ./instrumentation.mjs app.js
...
EnvDetector found resource. Resource {
  attributes: {
    'service.name': 'app.js',
    'service.namespace': 'tutorial',
    'service.version': '1.0',
    'service.instance.id': '46D99F44-27AB-4006-9F57-3B7C9032827B',
    'host.name': 'myhost',
    'host.type': 'arm64',
    'os.name': 'linux',
    'os.version': '6.0'
  }
}
...
```

## 코드에서 리소스 추가하기 {#adding-resources-in-code}

커스텀 리소스는 코드에서도 구성할 수 있다. `NodeSDK`는 리소스를 설정할 수 있는
구성 옵션을 제공한다. 예를 들어 다음과 같이 계측 파일을 수정하면 `service.*`
속성을 설정할 수 있다.

```javascript
...
const { resourceFromAttributes } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } = require('@opentelemetry/semantic-conventions');
...
const sdk = new opentelemetry.NodeSDK({
  ...
  resource: resourceFromAttributes({
    [ ATTR_SERVICE_NAME ]: "yourServiceName",
    [ ATTR_SERVICE_VERSION ]: "1.0",
  })
  ...
});
...
```

> [!NOTE]
>
> 환경 변수와 코드 양쪽에서 리소스 속성을 설정하면, 환경 변수로 설정한 값이
> 우선한다.

## 컨테이너 리소스 감지 {#container-resource-detection}

디버깅을 켠 상태로 동일한 설정(`package.json`, `app.js`,
`instrumentation.mjs`)을 사용하고, 같은 디렉터리에 다음 내용의 `Dockerfile`을
준비한다.

```Dockerfile
FROM node:latest
WORKDIR /usr/src/app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD [ "node", "--import", "./instrumentation.mjs", "app.js" ]
```

<kbd>Ctrl + C</kbd>(`SIGINT`)로 도커 컨테이너를 중지할 수 있도록 `app.js` 맨
아래에 다음 내용을 추가한다.

```javascript
process.on('SIGINT', function () {
  process.exit();
});
```

컨테이너의 ID를 자동으로 감지하려면, 다음 추가 의존성을 설치한다.

```sh
npm install @opentelemetry/resource-detector-container
```

이제 `instrumentation.mjs`를 다음과 같이 업데이트한다.

```javascript
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');
const { diag, DiagConsoleLogger, DiagLogLevel } = require('@opentelemetry/api');
const {
  containerDetector,
} = require('@opentelemetry/resource-detector-container');

// For troubleshooting, set the log level to DiagLogLevel.DEBUG
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG);

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new opentelemetry.tracing.ConsoleSpanExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
  resourceDetectors: [containerDetector],
});

sdk.start();
```

도커 이미지를 빌드한다.

```sh
docker build . -t nodejs-otel-getting-started
```

도커 컨테이너를 실행한다.

```sh
$ docker run --rm -p 8080:8080 nodejs-otel-getting-started
@opentelemetry/api: Registered a global for diag v1.2.0.
...
Listening for requests on http://localhost:8080
DockerCGroupV1Detector found resource. Resource {
  attributes: {
    'container.id': 'fffbeaf682f32ef86916f306ff9a7f88cc58048ab78f7de464da3c3201db5c54'
  }
}
```

디텍터가 `container.id`를 추출한 것을 확인할 수 있다. 하지만 이 예시에서는
프로세스 속성과 환경 변수로 설정한 속성이 빠져 있다는 점을 눈치챘을 것이다. 이를
해결하려면, `resourceDetectors` 목록을 설정할 때 `envDetector`와
`processDetector` 디텍터도 함께 지정해야 한다.

```javascript
const opentelemetry = require('@opentelemetry/sdk-node');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');
const { diag, DiagConsoleLogger, DiagLogLevel } = require('@opentelemetry/api');
const {
  containerDetector,
} = require('@opentelemetry/resource-detector-container');
const { envDetector, processDetector } = require('@opentelemetry/resources');

// For troubleshooting, set the log level to DiagLogLevel.DEBUG
diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG);

const sdk = new opentelemetry.NodeSDK({
  traceExporter: new opentelemetry.tracing.ConsoleSpanExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
  // Make sure to add all detectors you need here!
  resourceDetectors: [envDetector, processDetector, containerDetector],
});

sdk.start();
```

이미지를 다시 빌드하고 컨테이너를 다시 실행한다.

```shell
docker run --rm -p 8080:8080 nodejs-otel-getting-started
@opentelemetry/api: Registered a global for diag v1.2.0.
...
Listening for requests on http://localhost:8080
EnvDetector found resource. Resource { attributes: {} }
ProcessDetector found resource. Resource {
  attributes: {
    'process.pid': 1,
    'process.executable.name': 'node',
    'process.command': '/usr/src/app/app.js',
    'process.command_line': '/usr/local/bin/node /usr/src/app/app.js',
    'process.runtime.version': '18.9.0',
    'process.runtime.name': 'nodejs',
    'process.runtime.description': 'Node.js'
  }
}
DockerCGroupV1Detector found resource. Resource {
  attributes: {
    'container.id': '654d0670317b9a2d3fc70cbe021c80ea15339c4711fb8e8b3aa674143148d84e'
  }
}
...
```

## 다음 단계 {#next-steps}

구성에 추가할 수 있는 리소스 디텍터가 더 있다. 예를 들어 [Cloud][]나
[Deployment][] 환경에 대한 세부 정보를 얻을 수 있다. 더 자세한 내용은
[opentelemetry-js-contrib 저장소의 `resource-detector-*` 패키지들](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages)을
참고한다.

[시작하기 - Node.js]: /docs/languages/js/getting-started/nodejs/
[프로세스 및 프로세스 런타임 리소스]: /docs/specs/semconv/resource/process/
[host]: /docs/specs/semconv/resource/host/
[cloud]: /docs/specs/semconv/resource/cloud/
[deployment]: /docs/specs/semconv/resource/deployment-environment/
[service]: /docs/specs/semconv/resource/#service
[os]: /docs/specs/semconv/resource/os/
