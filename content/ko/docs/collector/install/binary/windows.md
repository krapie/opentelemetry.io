---
title: Windows에 컬렉터 설치하기
linkTitle: Windows
weight: 300
default_lang_commit: 9f912d59a165ded5dec82d0e1a94c2aef54e5c57
---

Windows [릴리스][releases]는 MSI 설치 프로그램과 gzip으로 압축된
tarball(`.tar.gz`) 형태로 제공된다.

## MSI 설치 {#msi-installation}

MSI는 배포판(Distribution)의 이름을 딴 Windows 서비스로 컬렉터를 설치하며, 표시
이름은 "OpenTelemetry Collector"이다. 또한 배포판 이름으로 애플리케이션 이벤트
로그 소스를 등록한다.

```powershell
msiexec /i "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_windows_x64.msi"
```

## 수동 설치 {#manual-installation}

gzip으로 압축된 tarball의 압축을 풀려면 다음 명령을 실행한다.

```powershell
Invoke-WebRequest -Uri "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_windows_amd64.tar.gz" -OutFile "otelcol_{{% param vers %}}_windows_amd64.tar.gz"
tar -xvzf otelcol_{{% param vers %}}_windows_amd64.tar.gz
```

컬렉터의 모든 릴리스에는 설치하고 실행할 수 있는 실행 파일이 포함되어 있다.

[releases]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases
