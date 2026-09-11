---
title: 교육
menu: { main: { weight: 45 } }
description: 오픈텔레메트리(OpenTelemetry) 자격증과 강의
type: docs
body_class: ot-training
hide_feedback: true
# LF course image from:
# https://training.linuxfoundation.org/wp-content/uploads/2024/10/LFS148-Course-Badge-300x300.png
params:
  LFS148: https://training.linuxfoundation.org/training/getting-started-with-opentelemetry-lfs148/
default_lang_commit: 662edd797da7c0d65ffb50537187c736a081ba2a
cSpell:ignore: otca
---

이 페이지는 점점 늘어나는 오픈텔레메트리 교육 자료를 소개한다. 업데이트를
확인하려면 자주 방문한다!

## 자격증 {#certifications}

[Cloud Native Certifications][]에서 제공하는 오픈텔레메트리 공인
어소시에이트(OpenTelemetry Certified Associate, OTCA) 자격증을 취득해 자신의
오픈텔레메트리 전문성을 증명한다.

<!-- prettier-ignore -->
[![OTCA badge]][OTCA certification]
{.badge--otca .card-and-img-position .hk-no-external-icon}

[Cloud Native Certifications]: https://www.cncf.io/training/certification/
[OTCA badge]: lft-badge-opentelemetry-associate2.svg
[OTCA certification]: https://www.cncf.io/training/certification/otca/

## 강의 {#courses}

[Cloud Native Training Courses for OpenTelemetry][CNTCOT]에서 제공하고 Linux
Foundation이 운영하는 **무료** 강의다.

<div class="card--course-wrapper">
<div class="card card--course" style="width: 20rem">

<!-- prettier-ignore -->
![LFS148 course badge][]
{.border-0 .pt-3 .w-75 .m-auto}

<div class="card-body ps-4 pe-4 bg-light-subtle">
  <div class="h4 card-title pt-2 pb-2">
    <span class="badge text-bg-secondary float-end">FREE</span>
    Getting Started with OpenTelemetry
  </div>
  <p class="card-text">
    소프트웨어 개발자, DevOps 엔지니어, 사이트 신뢰성 엔지니어(SRE), 그리고
    애플리케이션과 환경 전반에 텔레메트리 솔루션을 구현하고자 하는 모든 사람을
    위해 설계된 강의다.
  </p>
  <p class="card-text text-body-secondary small">
    온라인, 자기 주도 학습, 8-10시간,
    <a href="{{% param LFS148 %}}">자세히 알아보기</a>.
  </p>
  <p class="text-center m-0 pt-1 pb-2">
    <a href="{{% param LFS148 %}}" target="_blank" rel="noopener" class="btn btn-primary">
      등록
    </a>
  </p>
</div>

</div>
</div>

[CNTCOT]: https://www.cncf.io/training/courses/?_sft_lf-project=opentelemetry
[LFS148 course badge]: LFS148-Course-Badge-300x300.avif

{{% comment %}}

<!-- Alternative design. Keeping for possible use later -->

<div class="card mb-3" style="max-width: 540px; margin: auto">
  <div class="row p-2">
    <div class="col-md-5 d-flex align-items-center">
      <img src="LFS148-Course-Badge-300x300.avif"
        class="img-initial m-auto"
        alt="LFS148 course badge">
    </div>
    <div class="col-md-7">
      <div class="card-body p-3">
        <h5 class="card-title">Getting Started with OpenTelemetry</h5>
        <p class="card-text">
          A course designed for software developers, DevOps engineers, site reliability engineers (SREs), and more looking to implement telemetry solutions across apps and environments.
        </p>
        <p class="card-text text-body-secondary small">
          Online, self-paced, 8-10 hrs,
          <a href="{{% param LFS148 %}}">learn more</a>.
        </p>
        <p class="text-center w-100">
          <a href="{{% param LFS148 %}}" target="_blank" rel="noopener" class="btn btn-primary ">
            Register
          </a>
        </p>
      </div>
    </div>
  </div>
</div>

{{% /comment %}}
