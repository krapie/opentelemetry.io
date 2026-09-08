---
title: OBI 설정하기
linkTitle: 설정
description: OBI를 설정하고 실행하는 방법을 알아본다.
weight: 10
default_lang_commit: d9f9ecf7f33ab10ccb86aff881a64af2e866883f
---

OBI를 설정하고 실행하는 방법에는 여러 가지 옵션이 있다.

- [Helm으로 쿠버네티스에 OBI 설정하기](kubernetes-helm/)
- [쿠버네티스에 OBI 설정하기](kubernetes/)
- [Docker에 OBI 설정하기](docker/)
- [독립형(standalone) 프로세스로 OBI 설정하기](standalone/)

구성 옵션 및 데이터 내보내기(export) 모드에 대한 정보는
[OBI 구성](../configure/) 문서를 참고한다.

> [!NOTE]
>
> OBI를 사용하여 트레이스를 생성(generate)할 계획이라면,
> [Routes Decorator](../configure/routes-decorator/) 구성에 대한 문서 섹션을
> 반드시 읽어보길 바란다. OBI는 코드를 전혀 수정하지 않고 애플리케이션을 자동
> 계측하므로, 자동으로 할당되는 서비스 이름과 URL이 예상과 다를 수 있다.
