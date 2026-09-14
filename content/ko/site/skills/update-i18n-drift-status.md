---
title: i18n 드리프트 상태 업데이트
description: >-
  로컬라이즈된 콘텐츠 전반에 걸쳐 drifted_from_default 프론트매터 필드를
  업데이트하고, 선택적으로 그 결과로 PR을 여는 방법이다.
default_lang_commit: dc2fb5771163265cb804a39b1dacc536b95bdb96
---

로케일별로 `npm run fix:i18n:status`를 실행하고, 로케일별로 커밋한 뒤,
선택적으로 PR을 열어서 로컬라이즈된 콘텐츠의 `drifted_from_default` 프론트매터
필드를 업데이트하려면 다음 단계를 따른다.

## 인자 {#arguments}

이 스킬은 다음과 같은 선택적 인자를 받는다:

- **`--locale locale,...`** (선택): 처리할 로케일 ID를 쉼표로 구분한 목록이다.
  예: `--locale pt,es,fr`. 생략하면 영어를 제외한 모든 로케일을 처리한다.
- **`--create-pr`** (선택적 플래그): 처리 후 PR을 자동으로 생성한다. 생략하면
  `AskUserQuestion`으로 사용자에게 PR을 생성할지 묻는다.

## 준비 {#preparation}

이 단계들은 메인 저장소를 가리키도록 `upstream` 원격이 설정된 저장소의 로컬
클론이 있다고 가정한다. 이 단계는 저장소 루트에서 로컬로 실행한다.

1. 작업 트리가 클린한지(커밋되지 않은 변경 사항이 없는지) 확인한다.
2. `main`으로 전환하고 최신 변경 사항을 pull한다:

   ```sh
   git checkout main
   git pull upstream main
   ```

3. 작업 브랜치를 생성한다:

   ```sh
   git checkout -b i18n_update-drift-status
   ```

## 로케일 찾기 {#discover-locales}

`--locale`가 전달되지 않았다면, 콘텐츠 디렉터리에서 영어를 제외한 모든 로케일을
찾아낸다:

```sh
find content -maxdepth 1 -mindepth 1 -type d ! -name 'en' -exec basename {} \;
```

이 명령은 줄마다 로케일 ID 하나씩을 반환한다(예: `bn`, `es`, `fr`, …).

`--locale`가 전달되었다면 대신 그 목록을 사용한다.

> [!NOTE] `en`은 절대 포함하지 않는다
>
> 영어는 기본 콘텐츠이며 드리프트될 수 없으므로, 로케일 목록에 포함되거나 이
> 스킬에서 처리되어서는 절대 안 된다. `--locale` 인자에 `en`이 포함되어 있으면
> 무시하거나 오류를 보고한다.

## 로케일별로 드리프트 상태 업데이트하기 {#update-per-locale}

해석된 로케일 목록의 `{LANG_ID}`마다:

1. 드리프트 상태 업데이트 명령을 실행한다:

   ```sh
   npm run fix:i18n:status -- content/{LANG_ID}
   ```

2. PR 설명 표에 쓸 통계를 수집한다:

   ```sh
   # 드리프트된 파일
   grep -rl "drifted_from_default: true" content/{LANG_ID} | wc -l
   # 번역 대상 파일 전체
   grep -rl "default_lang_commit" content/{LANG_ID} | wc -l
   ```

3. 명령이 변경 사항을 만들어냈다면, 스테이징하고 커밋한다:

   ```sh
   git add content/{LANG_ID}
   git commit -m "chore({LANG_ID}): update drift status"
   ```

   해당 로케일에 변경 사항이 없으면 커밋은 건너뛰되 통계는 그대로 기록한다.

## PR 생성하기 {#create-the-pr}

모든 로케일을 처리한 뒤:

- `--create-pr`가 전달되지 **않았다면**, 진행하기 전에 `AskUserQuestion`으로
  사용자에게 PR을 생성할지 묻는다.
- 사용자가 거절하면(또는 `--create-pr`가 전달되지 않았고 사용자가 아니라고
  답하면), 여기서 멈추고 통계를 보고한다.

PR을 생성하려면 브랜치를 push한다:

```sh
git push -u origin i18n_update-drift-status
```

그런 다음 `gh pr create`를 다음과 함께 실행한다:

- **제목**: `[i18n] Update drift status for localized content`
- **설명**: 위에서 수집한 통계로 아래 표를 채우되, 처리된 로케일만 포함한다.

```md
Updates the drift status for localized content.

Status per locale after this PR:

| Locale | Drifted files | Total files |
| ------ | ------------- | ----------- |
| {ID}   | {drifted}     | {total}     |
```
