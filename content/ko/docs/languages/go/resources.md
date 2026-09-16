---
title: 리소스
weight: 70
cSpell:ignore: sdktrace thirdparty
default_lang_commit: 12f31f62fcc466532513f6ebccb060c9ea5b9fe4
---

{{% docs/languages/resources-intro %}}

리소스는 트레이서, 미터, 로거 프로바이더의 초기화 시점에 할당해야 하며, 속성과
거의 비슷한 방식으로 생성된다.

```go
res := resource.NewWithAttributes(
    semconv.SchemaURL,
    semconv.ServiceNameKey.String("myService"),
    semconv.ServiceVersionKey.String("1.0.0"),
    semconv.ServiceInstanceIDKey.String("abcdef12345"),
)

provider := sdktrace.NewTracerProvider(
    ...
    sdktrace.WithResource(res),
)
```

리소스 속성에 대한 [관례적인 이름](/docs/concepts/semantic-conventions/)을
제공하기 위해 `semconv` 패키지를 사용한다는 점에 유의한다. 이는 이러한 시맨틱
컨벤션으로 생성된 텔레메트리의 소비자가 관련 속성을 쉽게 찾아내고 그 의미를
이해할 수 있도록 돕는다.

리소스는 `resource.Detector` 구현체를 통해 자동으로 감지될 수도 있다. 이러한
`Detector`는 현재 실행 중인 프로세스, 그것이 실행되는 운영체제, 해당 운영체제
인스턴스를 호스팅하는 클라우드 프로바이더, 그 밖의 다양한 리소스 속성에 대한
정보를 발견할 수 있다.

```go
res, err := resource.New(
	context.Background(),
	resource.WithFromEnv(),      // Discover and provide attributes from OTEL_RESOURCE_ATTRIBUTES and OTEL_SERVICE_NAME environment variables.
	resource.WithTelemetrySDK(), // Discover and provide information about the OpenTelemetry SDK used.
	resource.WithProcess(),      // Discover and provide process information.
	resource.WithOS(),           // Discover and provide OS information.
	resource.WithContainer(),    // Discover and provide container information.
	resource.WithHost(),         // Discover and provide host information.
	resource.WithAttributes(attribute.String("foo", "bar")), // Add custom resource attributes.
	// resource.WithDetectors(thirdparty.Detector{}), // Bring your own external Detector implementation.
)
if errors.Is(err, resource.ErrPartialResource) || errors.Is(err, resource.ErrSchemaURLConflict) {
	log.Println(err) // Log non-fatal issues.
} else if err != nil {
	log.Fatalln(err) // The error may be fatal.
}
```
