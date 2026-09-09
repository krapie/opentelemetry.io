---
title: 이미지 제공 서비스
linkTitle: 이미지 제공
aliases: [imageprovider] # cSpell:disable-line
default_lang_commit: 66bbef71cecd6be3d6f8a461f6e54a7368a79139
---

이 서비스는 프론트엔드에서 사용되는 이미지를 제공한다. 이미지는 NGINX
인스턴스에서 정적으로 호스팅된다. NGINX 서버는
[nginx-otel 모듈](https://github.com/nginxinc/nginx-otel/tree/main)로 계측되어
있다.

자세한 내용은
[이미지 제공 서비스 소스 코드](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/image-provider/)를
참고한다.
