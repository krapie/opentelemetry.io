---
title: 서버리스
weight: 100
description: OpenTelemetry JavaScript로 서버리스 함수 계측하기
cSpell:ignore: otelwrapper
default_lang_commit: 2b99811a310f2749a5b6389f5e4d654a4f2e2f8e
---

이 가이드는 오픈텔레메트리(OpenTelemetry) 계측 라이브러리를 사용해 서버리스
함수의 트레이싱을 시작하는 방법을 보여준다.

> [!NOTE]
>
> 오픈텔레메트리 문서는 컴파일된 애플리케이션이
> [CommonJS](https://nodejs.org/api/modules.html#modules-commonjs-modules)로
> 실행된다고 가정한다.

## AWS Lambda {#aws-lambda}

> [!NOTE]
>
> [커뮤니티에서 제공하는 Lambda 레이어](/docs/platforms/faas/lambda-auto-instrument/)를
> 사용해 AWS Lambda 함수를 자동으로 계측할 수도 있다.

다음은 Lambda 래퍼를 오픈텔레메트리와 함께 사용해 AWS Lambda 함수를 수동으로
계측하고 트레이스를 구성된 백엔드로 보내는 방법을 보여준다.

플러그 앤 플레이 방식의 사용 경험에 관심이 있다면,
[오픈텔레메트리 Lambda 레이어](https://github.com/open-telemetry/opentelemetry-lambda)를
참고한다.

### 의존성 {#dependencies}

먼저, 빈 `package.json`을 생성한다.

```sh
npm init -y
```

그런 다음 필요한 의존성을 설치한다.

```sh
npm install \
  @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/instrumentation \
  @opentelemetry/sdk-trace-base \
  @opentelemetry/sdk-trace-node
```

### AWS Lambda 래퍼 코드 {#aws-lambda-wrapper-code}

이 파일에는 트레이싱을 활성화하는 모든 오픈텔레메트리 로직이 담겨 있다. 다음
코드를 `lambda-wrapper.js`로 저장한다.

```javascript
/* lambda-wrapper.js */

const api = require('@opentelemetry/api');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const {
  OTLPTraceExporter,
} = require('@opentelemetry/exporter-trace-otlp-http');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');

api.diag.setLogger(new api.DiagConsoleLogger(), api.DiagLogLevel.ALL);

const spanProcessor = new BatchSpanProcessor(
  new OTLPTraceExporter({
    url: '<backend_url>',
  }),
);

const provider = new NodeTracerProvider({
  spanProcessors: [spanProcessor],
});

provider.register();

registerInstrumentations({
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-aws-lambda': {
        disableAwsContextPropagation: true,
      },
    }),
  ],
});
```

`<backend_url>`을 원하는 백엔드의 URL로 바꿔서 모든 트레이스를 그곳으로
내보낸다. 아직 준비된 백엔드가 없다면, [Jaeger](https://www.jaegertracing.io/)나
[Zipkin](https://zipkin.io/)을 확인해본다.

`disableAwsContextPropagation`이 true로 설정되어 있다는 점에 유의한다. 그 이유는
Lambda 계측이 기본적으로 X-Ray 컨텍스트 헤더를 사용하려고 시도하기 때문인데, 이
함수에 대해 액티브 트레이싱이 활성화되어 있지 않으면 이는 샘플링되지 않은
컨텍스트로 이어지고, `NonRecordingSpan`을 생성하게 된다.

더 자세한 내용은 계측
[문서](https://www.npmjs.com/package/@opentelemetry/instrumentation-aws-lambda)에서
확인할 수 있다.

### AWS Lambda 함수 핸들러 {#aws-lambda-function-handler}

이제 Lambda 래퍼가 준비되었으니, Lambda 함수 역할을 하는 간단한 핸들러를 만든다.
다음 코드를 `handler.js`로 저장한다.

```javascript
/* handler.js */

'use strict';

const https = require('https');

function getRequest() {
  const url = 'https://opentelemetry.io/';

  return new Promise((resolve, reject) => {
    const req = https.get(url, (res) => {
      resolve(res.statusCode);
    });

    req.on('error', (err) => {
      reject(new Error(err));
    });
  });
}

exports.handler = async (event) => {
  try {
    const result = await getRequest();
    return {
      statusCode: result,
    };
  } catch (error) {
    return {
      statusCode: 400,
      body: error.message,
    };
  }
};
```

### 배포 {#deployment}

Lambda 함수를 배포하는 방법에는 여러 가지가 있다.

- [AWS Console](https://aws.amazon.com/console/)
- [AWS CLI](https://aws.amazon.com/cli/)
- [Serverless Framework](https://github.com/serverless/serverless)
- [Terraform](https://github.com/hashicorp/terraform)

여기서는 Serverless Framework를 사용하며, 더 자세한 내용은
[Serverless Framework 설정 가이드](https://www.serverless.com/framework/docs/getting-started)에서
확인할 수 있다.

`serverless.yml` 파일을 생성한다.

```yaml
service: lambda-otel-native
frameworkVersion: '3'
provider:
  name: aws
  runtime: nodejs14.x
  region: '<your-region>'
  environment:
    NODE_OPTIONS: --require lambda-wrapper
functions:
  lambda-otel-test:
    handler: handler.hello
```

오픈텔레메트리가 제대로 동작하려면, `lambda-wrapper.js`가 다른 어떤 파일보다
먼저 포함되어야 한다. `NODE_OPTIONS` 설정이 이를 보장한다.

Serverless Framework를 사용해 Lambda 함수를 배포하지 않는다면, AWS Console UI를
사용해 이 환경 변수를 수동으로 추가해야 한다.

마지막으로, 다음 명령을 실행해 프로젝트를 AWS에 배포한다.

```shell
serverless deploy
```

이제 AWS Console UI를 사용해 새로 배포된 Lambda 함수를 호출할 수 있다. Lambda
함수 호출과 관련된 스팬이 나타나는 것을 확인할 수 있어야 한다.

### 백엔드 확인하기 {#visiting-the-backend}

이제 백엔드에서 Lambda 함수가 생성한 오픈텔레메트리 트레이스를 볼 수 있어야
한다!

## GCP 함수 {#gcp-function}

다음은 Google Cloud Platform(GCP) UI를 사용해
[HTTP로 트리거되는 함수](https://docs.cloud.google.com/run/docs/write-functions)를
계측하는 방법을 보여준다.

### 함수 생성하기 {#creating-function}

GCP에 로그인하고 함수를 배치할 프로젝트를 생성하거나 선택한다. 사이드 메뉴에서
_Serverless_로 이동해 _Cloud Functions_를 선택한다. 다음으로 _Create Function_을
클릭하고, 환경으로
[2세대](https://cloud.google.com/blog/products/serverless/cloud-functions-2nd-generation-now-generally-available)를
선택한 다음, 함수 이름과 리전을 지정한다.

### otelwrapper용 환경 변수 설정하기 {#setup-environment-variable-for-otelwrapper}

닫혀 있다면 _Runtime, build, connections and security settings_ 메뉴를 열고
아래로 스크롤해 다음 값으로 `NODE_OPTIONS` 환경 변수를 추가한다.

```shell
--require ./otelwrapper.js
```

### 런타임 선택하기 {#select-runtime}

다음 화면(_Code_)에서 런타임으로 Node.js 버전 16을 선택한다.

### OTel 래퍼 만들기 {#create-otel-wrapper}

서비스를 계측하는 데 사용할 `otelwrapper.js`라는 새 파일을 만든다.
`SERVICE_NAME`을 제공하고 `<address for your backend>`를 설정했는지 확인한다.

```javascript
/* otelwrapper.js */

const { resourceFromAttributes } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME } = require('@opentelemetry/semantic-conventions');
const api = require('@opentelemetry/api');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const {
  OTLPTraceExporter,
} = require('@opentelemetry/exporter-trace-otlp-http');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const {
  getNodeAutoInstrumentations,
} = require('@opentelemetry/auto-instrumentations-node');

api.diag.setLogger(new api.DiagConsoleLogger(), api.DiagLogLevel.ALL);

const collectorOptions = {
  url: '<address for your backend>',
};

const provider = new NodeTracerProvider({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: '<your function name>',
  }),
  spanProcessors: [
    new BatchSpanProcessor(new OTLPTraceExporter(collectorOptions)),
  ],
});

provider.register();

registerInstrumentations({
  instrumentations: [getNodeAutoInstrumentations()],
});
```

### 패키지 의존성 추가하기 {#add-package-dependencies}

`package.json`에 다음을 추가한다.

```json
{
  "dependencies": {
    "@google-cloud/functions-framework": "^3.0.0",
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/auto-instrumentations-node": "^0.56.1",
    "@opentelemetry/exporter-trace-otlp-http": "^0.200.0",
    "@opentelemetry/instrumentation": "^0.200.0",
    "@opentelemetry/sdk-trace-base": "^2.0.0",
    "@opentelemetry/sdk-trace-node": "^2.0.0",
    "@opentelemetry/resources": "^2.0.0",
    "@opentelemetry/semantic-conventions": "^2.0.0"
  }
}
```

### 함수에 HTTP 호출 추가하기 {#add-http-call-to-function}

다음 코드는 아웃바운드 호출을 보여주기 위해 오픈텔레메트리 웹사이트로 호출을
보낸다.

```javascript
/* index.js */
const functions = require('@google-cloud/functions-framework');
const https = require('https');

functions.http('helloHttp', (req, res) => {
  let url = 'https://opentelemetry.io/';
  https
    .get(url, (response) => {
      res.send(`Response ${response.body}!`);
    })
    .on('error', (e) => {
      res.send(`Error ${e}!`);
    });
});
```

### 백엔드 {#backend}

GCP VM에서 OTel 컬렉터를 실행한다면, 트레이스를 보낼 수 있도록
[VPC 액세스 커넥터를 생성](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access)해야
할 가능성이 높다.

### 배포하기 {#deploy}

UI에서 Deploy를 선택하고 배포가 준비될 때까지 기다린다.

### 테스트하기 {#testing}

test 탭에서 cloud shell을 사용해 함수를 테스트할 수 있다.
