---
title: 개발 환경 설정과 빌드, 서비스 실행 등을 위한 명령어
linkTitle: 개발 환경 설정 등
description: >-
  클라우드 IDE 및 로컬 환경 설정과 사이트의 빌드, 서비스 실행, 체크 명령어
what-next: >
  이제 [빌드](#build)하고 [서비스를 실행](#serve)하며 웹사이트 파일을 업데이트할
  준비가 되었다. 변경 사항을 제출하는 방법에 대한 세부 내용은 [콘텐츠
  제출](../pull-requests)을 참고한다.
weight: 60
cSpell:ignore: TOCSS
default_lang_commit: 55cf22ecef4f7db0860404d4766bf9cc130ae82b
---

> [!WARNING] 지원되는 빌드 환경
>
> 빌드는 Linux 기반 환경과 macOS에서 공식적으로 지원된다.
> [DevContainer](#devcontainers)와 같은 다른 환경은 최선을 다해 지원한다.
> Windows에서 빌드하려면 Windows Subsystem for Linux 명령줄 [WSL][]을 사용해
> Linux와 비슷한 절차를 따를 수 있다.

## 클라우드 IDE 설정 {#cloud-ide-setup}

### Gitpod {#gitpod}

[Gitpod.io][gitpod.io]를 통해 작업하려면 다음과 같이 한다.

1.  이 저장소를 포크한다. 도움이 필요하면 [저장소 포크하기][fork]를 참고한다.
2.  [gitpod.io/workspaces][]에서 새 워크스페이스를 만들거나(이 작업은 한 번만
    한다) 자신의 포크 위에서 기존 워크스페이스를 연다. 다음과 같은 형식의 링크로
    방문할 수도 있다.
    `https://gitpod.io#https://github.com/YOUR_GITHUB_ID/opentelemetry.io`.

    > **참고**: 이 저장소에서 작업할 수 있는 권한이 있거나 그냥 둘러보고 싶다면
    > <https://gitpod.io/#https://github.com/open-telemetry/opentelemetry.io>를
    > 연다.

Gitpod은 저장소별 패키지를 자동으로 설치해준다. {{% param what-next %}}

### Codespaces {#codespaces}

GitHub [Codespaces][]를 통해 작업하려면 다음과 같이 한다.

1. 웹사이트 저장소를 [포크한다][Fork].
2. 자신의 포크에서 Codespace를 연다.

개발 환경은 [DevContainer](#devcontainers) 설정을 통해 초기화된다.
{{% param what-next %}}

## 로컬 설정 {#local-setup}

1.  웹사이트 저장소를 <{{% param github_repo %}}>에서 [포크][Fork]한 다음
    [클론][clone]한다.
2.  저장소 디렉터리로 이동한다.

    ```sh
    cd opentelemetry.io
    ```

3.  `.nvmrc` 파일에 고정된 Node.js 릴리스([활성 LTS][nodejs-rel] 버전)를
    설치한다. Node 설치를 관리하려면 [nvm][]을 권장한다. Linux에서는 다음을
    실행한다.

    ```sh
    nvm install
    ```

    [Windows에서 설치하려면][nodejs-win] `.nvmrc` 자체를 읽지 않는
    [nvm-windows][]를 사용한다. 다음 명령어는 고정된 버전을 전달한다. Windows
    PowerShell이 아니라 `cmd`를 사용하는 것을 권장한다.

    ```cmd
    for /f %v in (.nvmrc) do nvm install %v && nvm use %v
    ```

4.  devcontainer가 사용하는 [lock-exact, script-suppressing 설정][ci-install]을
    이용해 npm 패키지와 그 밖의 사전 준비물을 가져온다.

    ```sh
    npm run install:safe
    ```

    또는 표준 설치를 사용한다.

    ```sh
    npm install
    ```

    두 설치 방법 모두 커밋된 `package-lock.json`에 고정된 의존성 버전을
    사용하며, 실행되는 모든 의존성 라이프사이클 스크립트는 검토된 허용 목록의
    적용을 받는다. 관련 내용: [의존성 업데이트하기][dep-updates].

원하는 IDE를 실행한다. {{% param what-next %}}

### 빌드 {#build}

사이트를 빌드하려면 다음을 실행한다.

```sh
npm run build
```

생성된 사이트 파일은 `public` 아래에 있다.

> [!IMPORTANT]
>
> 다음과 비슷한 빌드 또는 서비스 실행 명령어 **오류**가 나타난다면 다음과 같이
> 한다.
>
> ```log
> ERROR error building site: ...[long message]... TOCSS: failed to transform "/scss/main.scss" (text/x-scss)
> ```
>
> 또는:
>
> ```log
> ERROR failed to load modules: module "github.com/FortAwesome/Font-Awesome" not found
> ```
>
> 이는 대개 [로컬 설정](#local-setup)의 모든 단계를 완료하지 않았기 때문에
> 발생한다. 특히 이 명령어를 (다시) 실행한다.
>
> ```sh
> npm install
> ```

### 서비스 실행 {#serve}

사이트를 서비스로 실행하려면 다음을 실행한다.

```sh
npm run serve
```

사이트는 [localhost:1313][]에서 서비스된다.

serve 명령어는 디스크가 아니라 메모리에서 파일을 서비스한다.

Netlify 리디렉션을 테스트하려면 자신의 PR에 대한 [배포
미리보기][deploy preview]를 사용한다.

macOS에서 `too many open files`나 `pipe failed`와 같은 오류가 나타난다면 파일
디스크립터 한도를 늘려야 할 수도 있다.
[Hugo 이슈 #6109](https://github.com/gohugoio/hugo/issues/6109)를 참고한다.

### 콘텐츠와 서브모듈 {#content-and-submodules}

웹사이트는 다음 콘텐츠로 빌드된다.

- [Hugo][] 기본 설정에 따른 `content/`, `static/` 등 아래의 파일들.
- `config/_default/module-template.yaml`의 Hugo [설정][config]으로 정의된 마운트
  포인트. 마운트는 [content-modules][] 아래의 git 서브모듈에서 직접 가져오거나,
  `content-modules`에서 전처리된 콘텐츠(`tmp/` 아래에 배치됨)를 가져오며, 그
  밖의 다른 곳에서는 가져오지 않는다.

[config]: https://github.com/open-telemetry/opentelemetry.io/tree/main/config
[content-modules]:
  https://github.com/open-telemetry/opentelemetry.io/tree/main/content-modules

### 서브모듈 변경 사항 {#submodule-changes}

[content-modules][] 서브모듈 내부의 콘텐츠를 변경하는 경우, 먼저 해당 서브모듈의
저장소에 (서브모듈 변경 사항을 포함하는) PR을 제출해야 한다. 서브모듈 PR이
승인된 후에만 서브모듈을 업데이트해 이 웹사이트에 변경 사항이 나타나게 할 수
있다.

`content-modules` 변경 사항은 서브모듈 자체보다는 해당 서브모듈이 연결된
저장소에서 작업하는 것이 관리하기 가장 쉽다.

숙련된 기여자는 서브모듈 안에서 직접 작업할 수 있다. 그러면 (서브모듈) 변경
사항을 직접 빌드하고 서비스로 실행할 수 있다. 기본적으로 CI 스크립트는 실행할
때마다 서브모듈을 가져온다. 서브모듈 안에서 작업하는 동안 이 동작을 막으려면
환경 변수 `GET=no`를 설정한다. 또한 PR을 제출하기 전에 서브모듈에서
`git fetch --unshallow`를 실행해야 한다. 또는 `DEPTH=100`을 설정하고 서브모듈을
다시 가져온다.

## DevContainer 지원 {#devcontainers}

이 저장소는 다양한 클라우드 및 로컬 IDE에서 지원하는 [Development
Containers][devcontainers]에서 사용하도록 설정되어 있다. 지원되는 IDE는 (알파벳
순으로) 다음과 같다.

- [Codespaces][cs-devc]
- [DevPod](https://devpod.sh/docs/developing-in-workspaces/devcontainer-json)
- [Gitpod](https://ona.com/docs/ona/configuration/devcontainer/overview)
- [VSCode](https://code.visualstudio.com/docs/devcontainers/containers#_installation)

## 도구 {#tools}

### Code-excerpter {#code-excerpter}

이 저장소의 소스 파일과 동기화된 상태를 유지해야 하는 코드 스니펫에는
[code-excerpter][]를 사용한다. 모든 로케일의 사이트 페이지에 코드 발췌를 포함할
수 있지만, 코드 발췌를 사용하는 원본 콘텐츠는 영어로 `content/en` 아래에 작성한
다음 로컬라이제이션 팀이 각자의 로케일로 업데이트한다.

영어 원본 페이지에서는 업데이트할 펜스 코드 블록(fenced code block) 바로 앞에
파일 발췌 지시어를 배치한다.

````md
<?code-excerpt path-base="examples/java/getting-started"?>

<?code-excerpt "src/main/java/otel/DiceApplication.java" from="@SpringBootApplication"?>

```java
@SpringBootApplication
public class DiceApplication {
  public static void main(String[] args) {
    SpringApplication app = new SpringApplication(DiceApplication.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
  }
}
```
````

여러 발췌가 같은 디렉터리에서 나오는 경우, 페이지 상단 근처에서 선택적으로
`path-base` 지시어를 한 번 사용한다. `code-excerpt` 지시어 구문에 대한 세부
사항은 [code-excerpter][] readme를 참고한다.

**펜스 코드가 아니라** 소스 파일이나 지시어를 편집한다. 그런 다음 다음
[npm 스크립트](/site/build/npm-scripts/)를 실행한다.

```sh
npm run fix:code-excerpts
```

code-excerpt가 최신 상태인지 확인하려면 다음을 실행한다.

```sh
npm run check:code-excerpts
```

[code-excerpter]: https://github.com/chalin/code-excerpter

<!-- prettier-ignore-start -->
[ci-install]: /site/build/dependencies/#install-contracts
[dep-updates]: /site/build/dependencies/#updating
[clone]: https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository
[codespaces]: https://docs.github.com/en/codespaces
[cs-devc]: https://docs.github.com/en/codespaces/setting-up-your-project-for-codespaces/adding-a-dev-container-configuration/introduction-to-dev-containers#about-dev-containers
[deploy preview]: ../pull-requests/#site-deploys-and-pr-previews
[devcontainers]: https://containers.dev/
[fork]: https://docs.github.com/en/get-started/quickstart/fork-a-repo
[gitpod.io]: https://gitpod.io
[gitpod.io/workspaces]: https://gitpod.io/workspaces
[hugo]: https://gohugo.io
[localhost:1313]: http://localhost:1313
[nodejs-rel]: https://nodejs.org/en/about/previous-releases
[nodejs-win]: https://docs.microsoft.com/en-us/windows/dev-environment/javascript/nodejs-on-windows
[nvm-windows]: https://github.com/coreybutler/nvm-windows
[nvm]: https://github.com/nvm-sh/nvm/blob/master/README.md#installing-and-updating
[WSL]: https://learn.microsoft.com/en-us/windows/wsl/install
<!-- prettier-ignore-end -->
