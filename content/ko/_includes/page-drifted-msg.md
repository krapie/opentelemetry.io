---
default_lang_commit: b7641c788cd1e72468cca29b76f390b7eed2ec3b
---

<i class="fa-solid fa-triangle-exclamation" style="margin-left: -1.9rem; padding-right: 0.5rem;"></i>
이 페이지의 내용은 <b>오래되었을 수</b> 있으며 일부 링크가 유효하지 않을 수
있다.

{{ if $show_details }}

이 페이지의 <b>최신 버전</b>은 <a href="{{$default_lang_page_url}}">영어</a>로
볼 수 있다.

<details class="mt-2">
  <summary>자세한 정보 ...</summary>
  <p>
    이 페이지가 마지막으로 업데이트된 이후 영어 페이지에 어떤 변경이 있었는지
    확인하려면
    <a href="{{$compare_url}}" class="external-link" target="_blank" rel="noopener" data-proofer-ignore>
      GitHub compare {{$default_lang_commit_short}}..{{$default_lang_hash_short}}
    </a>
    에서 <code>{{$def_lang_path}}</code>를 검색한다.
  </p>
</details>
{{ end }}

{{ if $no_default_lang_page }}

이 페이지에 대응하는 영어 페이지는 더 이상 존재하지 않는다.

{{ end }}
