---
title: 애플리케이션 서버 구성
linkTitle: 앱 서버 구성
description: Java 애플리케이션 서버에 대한 에이전트 경로 정의 방법 알아보기
weight: 215
cSpell:ignore: asadmin Glassfish Payara setenv wildfly
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

Java 에이전트로 Java 애플리케이션 서버에서 실행되는 앱을 계측할 때는, JVM 인자에
`javaagent` 경로를 추가해야 한다. 이 방법은 서버마다 다르다.

## JBoss EAP / WildFly {#jboss-eap--wildfly}

standalone 구성 파일 끝에 `javaagent` 인자를 추가할 수 있다.

{{< tabpane text=true persist=lang >}}

{{% tab header="Linux" lang=Linux %}}

```sh
# standalone.conf에 추가
JAVA_OPTS="$JAVA_OPTS -javaagent:/path/to/opentelemetry-javaagent.jar"
```

{{% /tab %}} {{% tab header="Windows" lang=Windows %}}

```bat
rem standalone.conf.bat에 추가
set "JAVA_OPTS=%JAVA_OPTS% -javaagent:<Drive>:\path\to\opentelemetry-javaagent.jar"
```

{{% /tab %}} {{< /tabpane >}}

## Jetty {#jetty}

Java 에이전트의 경로를 정의하려면 `-javaagent` 인자를 사용한다.

```shell
java -javaagent:/path/to/opentelemetry-javaagent.jar -jar start.jar
```

`jetty.sh` 파일로 Jetty를 시작한다면, `\<jetty_home\>/bin/jetty.sh` 파일에 다음
줄을 추가한다.

```shell
JAVA_OPTIONS="${JAVA_OPTIONS} -javaagent:/path/to/opentelemetry-javaagent.jar"
```

start.ini 파일로 JVM 인자를 정의한다면, `--exec` 옵션 뒤에 `javaagent` 인자를
추가한다.

```ini
#===========================================================
# Sample Jetty start.ini file
#-----------------------------------------------------------
--exec
-javaagent:/path/to/opentelemetry-javaagent.jar
```

## Glassfish / Payara {#glassfish--payara}

`asadmin` 도구를 사용해 Java 에이전트의 경로를 추가한다.

{{< tabpane text=true >}} {{% tab Linux %}}

```sh
<server_install_dir>/bin/asadmin create-jvm-options "-javaagent\:/path/to/opentelemetry-javaagent.jar"
```

{{% /tab %}} {{% tab Windows %}}

```powershell
<server_install_dir>\bin\asadmin.bat create-jvm-options '-javaagent\:<Drive>\:\\path\\to\\opentelemetry-javaagent.jar'
```

{{% /tab %}} {{< /tabpane >}}

Admin Console에서 `-javaagent` 인자를 추가할 수도 있다. 예를 들어:

1.  <http://localhost:4848>에서 GlassFish Admin Console을 연다.
2.  **Configurations > server-config > JVM Settings**로 이동한다.
3.  **JVM Options > Add JVM Option**을 선택한다.
4.  에이전트의 경로를 입력한다.
    `-javaagent:/path/to/opentelemetry-javaagent.jar`
5.  **Save**하고 서버를 재시작한다.

도메인 디렉터리의 domain.xml 파일에 에이전트에 대한 `<jmv-options>` 항목이
있는지 확인한다.

## Tomcat / TomEE {#tomcat--tomee}

시작 스크립트에 Java 에이전트의 경로를 추가한다. 구성 방법은 설치 방식에 따라
다르다.

**패키지 관리 방식 설치**(apt-get/yum)의 경우, `/etc/tomcat*/tomcat*.conf`에
추가한다.

```sh
JAVA_OPTS="$JAVA_OPTS -javaagent:/path/to/opentelemetry-javaagent.jar"
```

**다운로드 방식 설치**의 경우, `<tomcat>/bin/setenv.sh`(Linux) 또는
`<tomcat>/bin/setenv.bat`(Windows)을 생성하거나 수정한다.

{{< tabpane text=true persist=lang >}}

{{% tab header="Linux" lang=Linux %}}

```sh
# <tomcat_home>/bin/setenv.sh에 추가
CATALINA_OPTS="$CATALINA_OPTS -javaagent:/path/to/opentelemetry-javaagent.jar"
```

{{% /tab %}} {{% tab header="Windows" lang=Windows %}}

```bat
rem <tomcat_home>\bin\setenv.bat에 추가
set CATALINA_OPTS=%CATALINA_OPTS% -javaagent:"<Drive>:\path\to\opentelemetry-javaagent.jar"
```

{{% /tab %}} {{< /tabpane >}}

**Windows 서비스 설치**의 경우, `<tomcat>/bin/tomcat*w.exe`를 사용해 Java 탭의
Java Options에 `-javaagent:<Drive>:\path\to\opentelemetry-javaagent.jar`를
추가한다.

## WebLogic {#weblogic}

도메인 시작 스크립트에 Java 에이전트의 경로를 추가한다.

{{< tabpane text=true persist=lang >}}

{{% tab header="Linux" lang=Linux %}}

```sh
# <domain_home>/bin/startWebLogic.sh에 추가
export JAVA_OPTIONS="$JAVA_OPTIONS -javaagent:/path/to/opentelemetry-javaagent.jar"
```

{{% /tab %}} {{% tab header="Windows" lang=Windows %}}

```bat
rem <domain_home>\bin\startWebLogic.cmd에 추가
set JAVA_OPTIONS=%JAVA_OPTIONS% -javaagent:"<Drive>:\path\to\opentelemetry-javaagent.jar"
```

{{% /tab %}} {{< /tabpane >}}

관리 서버 인스턴스의 경우, admin console을 사용해 `-javaagent` 인자를 추가한다.

## WebSphere Liberty Profile {#websphere-liberty-profile}

`jvm.options` 파일에 Java 에이전트의 경로를 추가한다. 단일 서버의 경우
`${server.config.dir}/jvm.options`를 편집하고, 모든 서버의 경우
`${wlp.install.dir}/etc/jvm.options`를 편집한다.

```ini
-javaagent:/path/to/opentelemetry-javaagent.jar
```

파일을 저장한 후 서버를 재시작한다.

## WebSphere Traditional {#websphere-traditional}

WebSphere Admin Console을 열고 다음 단계를 따른다.

<!-- markdownlint-disable blanks-around-fences -->

1.  **Servers > Server type > WebSphere application servers**로 이동한다.
2.  서버를 선택한다.
3.  **Java and Process Management > Process Definition**으로 이동한다.
4.  **Java Virtual Machine**을 선택한다.
5.  **Generic JVM arguments**에 에이전트의 경로를 입력한다.
    `-javaagent:/path/to/opentelemetry-javaagent.jar`.
6.  구성을 저장하고 서버를 재시작한다.

## 미리 정의된 JMX 메트릭 활성화하기 {#enable-predefined-jmx-metrics}

Java 에이전트에는 여러 인기 애플리케이션 서버를 위한 미리 정의된 JMX 메트릭
구성이 포함되어 있지만, 기본적으로 활성화되어 있지 않다. 미리 정의된 메트릭
수집을 활성화하려면, `otel.jmx.target.system` 시스템 속성의 값으로 대상 목록을
지정한다. 예를 들어:

```bash
$ java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.jmx.target.system=jetty,tomcat \
     ... \
     -jar myapp.jar
```

`otel.jmx.target.system`에 대해 알려진 애플리케이션 서버 값은 다음과 같다.

- [`jetty`](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/library/jetty.md)
- [`tomcat`](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/library/tomcat.md)
- [`wildfly`](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/library/wildfly.md)

> [!NOTE]
>
> 이 목록은 전부를 망라한 것이 아니며, 다른 JMX 대상 시스템도 지원된다.

각 애플리케이션 서버에서 추출되는 메트릭 목록은 앞의 이름을 선택하거나,
[추가 세부 정보 및 커스터마이즈 기능](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/jmx-metrics#predefined-metrics)을
참고한다.
