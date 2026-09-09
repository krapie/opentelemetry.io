---
title: macOS에 컬렉터 설치하기
linkTitle: macOS
weight: 200
default_lang_commit: 9f912d59a165ded5dec82d0e1a94c2aef54e5c57
---

macOS [릴리스][releases]는 Intel 및 ARM 시스템에서 사용할 수 있다. 릴리스는
gzip으로 압축된 tarball(`.tar.gz`) 형태로 제공된다. 압축을 풀려면 다음 명령을
실행한다.

{{< tabpane text=true >}} {{% tab Intel %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_darwin_amd64.tar.gz
tar -xvf otelcol_{{% param vers %}}_darwin_amd64.tar.gz
```

{{% /tab %}} {{% tab ARM %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_darwin_arm64.tar.gz
tar -xvf otelcol_{{% param vers %}}_darwin_arm64.tar.gz
```

{{% /tab %}} {{< /tabpane >}}

컬렉터의 모든 릴리스에는 압축을 풀고 실행할 수 있는 실행 파일이 포함되어 있다.

[releases]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases
