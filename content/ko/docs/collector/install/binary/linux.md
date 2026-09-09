---
title: Linux에 컬렉터 설치하기
linkTitle: Linux
weight: 100
default_lang_commit: 9f912d59a165ded5dec82d0e1a94c2aef54e5c57
---

컬렉터의 모든 릴리스에는 Linux amd64/arm64/i386 시스템을 위한 APK, DEB, RPM
패키징이 포함되어 있다. 설치 후 기본 구성은 `/etc/otelcol/config.yaml`에서
확인할 수 있다.

> 참고: 자동 서비스 구성을 위해서는 `systemd`가 필요하다.

## DEB 설치 {#deb-installation}

Debian 시스템에서 시작하려면 다음 명령을 실행한다.

{{< tabpane text=true >}} {{% tab AMD64 %}}

```sh
sudo apt-get update
sudo apt-get -y install wget
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_amd64.deb
sudo dpkg -i otelcol_{{% param vers %}}_linux_amd64.deb
```

{{% /tab %}} {{% tab ARM64 %}}

```sh
sudo apt-get update
sudo apt-get -y install wget
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_arm64.deb
sudo dpkg -i otelcol_{{% param vers %}}_linux_arm64.deb
```

{{% /tab %}} {{% tab i386 %}}

```sh
sudo apt-get update
sudo apt-get -y install wget
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_386.deb
sudo dpkg -i otelcol_{{% param vers %}}_linux_386.deb
```

{{% /tab %}} {{< /tabpane >}}

## RPM 설치 {#rpm-installation}

Red Hat 시스템에서 시작하려면 다음 명령을 실행한다.

{{< tabpane text=true >}} {{% tab AMD64 %}}

```sh
sudo yum update
sudo yum -y install wget systemctl
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_amd64.rpm
sudo rpm -ivh otelcol_{{% param vers %}}_linux_amd64.rpm
```

{{% /tab %}} {{% tab ARM64 %}}

```sh
sudo yum update
sudo yum -y install wget systemctl
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_arm64.rpm
sudo rpm -ivh otelcol_{{% param vers %}}_linux_arm64.rpm
```

{{% /tab %}} {{% tab i386 %}}

```sh
sudo yum update
sudo yum -y install wget systemctl
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_386.rpm
sudo rpm -ivh otelcol_{{% param vers %}}_linux_386.rpm
```

{{% /tab %}} {{< /tabpane >}}

## 수동 설치 {#manual-installation}

Linux [릴리스][releases]는 다양한 아키텍처에서 사용할 수 있다. 바이너리 파일을
다운로드하여 수동으로 설치할 수 있다.

{{< tabpane text=true >}} {{% tab AMD64 %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_amd64.tar.gz
tar -xvf otelcol_{{% param vers %}}_linux_amd64.tar.gz
```

{{% /tab %}} {{% tab ARM64 %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_arm64.tar.gz
tar -xvf otelcol_{{% param vers %}}_linux_arm64.tar.gz
```

{{% /tab %}} {{% tab i386 %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_386.tar.gz
tar -xvf otelcol_{{% param vers %}}_linux_386.tar.gz
```

{{% /tab %}} {{% tab ppc64le %}}

```sh
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v{{% param vers %}}/otelcol_{{% param vers %}}_linux_ppc64le.tar.gz
tar -xvf otelcol_{{% param vers %}}_linux_ppc64le.tar.gz
```

{{% /tab %}} {{< /tabpane >}}

## 자동 서비스 구성 {#automatic-service-configuration}

오픈텔레메트리(OpenTelemetry) 컬렉터가 `systemd` 서비스로 실행될 때는 기본적으로
`/etc/otelcol/config.yaml` 구성 파일로 시작한다.

이 설정을 변경하려면 `systemd` 환경 파일인 `/etc/otelcol/otelcol.conf`에서
`OTELCOL_OPTIONS` 변수를 편집하면 된다. 같은 파일에서 `otelcol` 서비스를 위한
추가 환경 변수도 정의할 수 있다. 지원되는 전체 옵션 목록을 확인하려면 다음
명령을 실행한다.

```sh
/usr/bin/otelcol --help
```

컬렉터 구성 파일(`config.yaml`) 또는 환경 파일(`otelcol.conf`)을 수정했다면,
변경 사항을 적용하기 위해 서비스를 재시작해야 한다.

```sh
sudo systemctl restart otelcol
```

`otelcol` 서비스의 로그 출력을 확인하려면 다음을 실행한다.

```sh
sudo journalctl -u otelcol
```

[releases]:
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases
