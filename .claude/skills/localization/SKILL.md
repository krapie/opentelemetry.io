---
name: localization
description: >-
  Plan, translate, and validate Korean (ko) localization for opentelemetry.io.
  Use for checking i18n drift, translating English docs to Korean, and
  validating existing Korean translations.
argument-hint: 'plan | translate <en-file-path> | validate [<ko-file-path>]'
allowed-tools: Bash Read Glob Grep WebFetch
model: sonnet
effort: medium
---

# Localization Skill (Korean / ko)

**Primary authority**: The [localization guide][] is the canonical source of
truth for all process rules. When this skill conflicts with it, trust the guide.

> **RECOVERY NOTE (2026-09-09)**: This file is git-excluded (`.git/info/exclude`)
> and lives only on local disk. Partway through the project that built this
> skill, the working environment reset and this file reverted to its very
> first draft — every correction below, all sourced from the actual merged
> PRs (#10431, #10440, #10448) and real mentor review comments, had to be
> reconstructed from conversation history. If you are reading this and
> something here looks wrong or thin, **check the actual merged
> `content/ko/**` files and real PR review threads before trusting this file
> blindly** — shipped state and real reviewer comments outrank this document,
> every time a conflict shows up. Consider un-excluding this file from git so
> a future reset doesn't repeat this.

[localization guide]: ../../../content/en/docs/contributing/localization.md

## Arguments {#arguments}

- `plan` — show i18n drift status and suggest next files to translate.
- `translate <en-file-path>` — produce a Korean translation of the given
  English file.
- `validate [<ko-file-path>]` — check one Korean file (or all of `content/ko/`)
  for drift and style issues.
- If `$ARGUMENTS` is empty, run `plan`.

## Workflow

### plan

1. Run `npm run check:i18n -- content/ko 2>&1 | head -80` to see drift. (The
   script takes a PATH argument, not a `--language` flag — that flag does not
   exist and just prints usage.)
2. List files under `content/ko/` with `find content/ko -name '*.md' | sort`.
3. Report:
   - Files already translated (with their drift status).
   - English files in `content/en/docs/` not yet translated.
   - Suggested priority order (concepts → getting-started → collector →
     instrumentation → specs).

### translate \<en-file-path\>

**Step 1 — Read source and capture SHA**

Read the English file. Capture the current HEAD SHA of that file — it becomes
the `default_lang_commit` value in front matter, which is how
`npm run check:i18n` detects future drift:

```bash
git log -1 --format=%H -- <en-file-path>
```

Also scan for structural cues: heading anchors (`{#foo}`), blockquote notation
rules (`> **Spelling**:`), reference-style link labels (`[label][ref]` and
`[ref]: url`) — these have specific preservation rules (see
[Structural conventions](#structural-conventions)).

**Step 2 — Resolve new terms**

For every term not already in the [Korean term table](#korean-term-table):

1. Fetch CNCF Korean glossary:
   `https://glossary.cncf.io/ko/` — search for the term.
2. Fetch Kubernetes Korean docs:
   `https://kubernetes.io/ko/docs/concepts/` — grep for the term.
3. If still unresolved, check AWS CloudWatch Korean docs or OTel Japanese
   glossary for analogous usage.
4. Fall back to phonetic transliteration only when no standard exists.

Also decide whether the term should be **translated** or **stay English** using
the [Term treatment rule](#term-treatment-rule).

**Step 3 — Translate**

Apply the [Korean style rules](#korean-style-rules) and
[Structural conventions](#structural-conventions) throughout.

Output the complete translated file content.

**Step 4 — Self-check**

Before finishing, walk the [Validation checklist](#validation-checklist)
against the output. Fix any violations before the next step.

**Step 5 — Output commit and PR content**

After the file content, print:

```
---
Commit message (title only — no body, no Co-Authored-By trailer; every
merged `ko` commit in #10431/#10440/#10448 is a single title line):
[i18n][ko] Localize <title> to Korean

PR title:
[i18n][ko] Localize <title> to Korean

PR body:
<!-- MAINTAINER NOTE: each list item should be on a single line. -->

- [x] I have read and followed the [Contributing](https://opentelemetry.io/docs/contributing/) docs, especially the "**First-time contributing?**" section.
- [x] This PR has content that I did not fully write myself.
  - [x] I used AI and I have read and followed the [Generative AI Contribution Policy](https://github.com/open-telemetry/community/blob/main/policies/genai.md).
- [x] I have the experience and knowledge necessary to understand, review, and validate all content in this PR.[^I-know-my-stuff]

[^I-know-my-stuff]:
    Yes, I can answer maintainer questions about the content of this PR, without using AI.

---

- <one-line summary of what this PR adds/changes, in English>
- <optional second bullet with extra context, e.g. "This is part of the
  Korean localization effort, following the same process used for other
  locales.">

Translation references

- [CNCF Cloud Native Glossary (KO)](https://glossary.cncf.io/ko/)
- [Kubernetes Docs (KO)](https://kubernetes.io/ko/docs/reference/glossary/)
- [AWS CloudWatch Docs (KO)](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/)

Closes #<tracking-issue>
```

> **Verified against the actual merged PRs** (#10431, #10448) — do not
> deviate from this shape:
> - The checklist block above is `.github/PULL_REQUEST_TEMPLATE.md` verbatim
>   (GitHub inserts it automatically when opening a PR from this repo). Check
>   all boxes `[x]` — translation work here is AI-assisted, so "I used AI"
>   is honestly true, matching what both merged PRs actually did.
> - The description below the `---` is a **short plain-English bullet list**,
>   not a Korean-language "## 변경 사항 (Changes)" section — that heading
>   style was never actually used in any merged `ko` PR. Keep it to 1–3
>   bullets.
> - "Translation references" (English heading, English bullets) only
>   appears when relevant — omit it for small fixes that aren't a fresh
>   translation.
> - A per-term table only appeared for the fully separate `ko` glossary
>   entries added to the [term table](#korean-term-table); it's not a
>   standard part of the PR body — don't add one unless the PR itself
>   introduces new terminology worth calling out.
>
> Note: commit and push are done by the human contributor, not this skill.
> Do not append "Part of #9577" — that was the one-time `ko` locale
> bootstrap issue (homepage + glossary) and is now closed. Link only the
> per-page/section tracking issue for this translation, if one exists.

### validate \[ko-file-path\]

1. If a path is given, check that file. Otherwise check all `content/ko/**/*.md`.
2. For each file, run:
   ```bash
   npm run check:i18n -- content/ko/<path> 2>&1 | tail -3
   ```
3. Walk the [Validation checklist](#validation-checklist) against the file.
4. Report drift (stale `default_lang_commit`) and any checklist violations,
   grouped by section (Front matter / Style / OpenTelemetry notation /
   Structural / Prose layout / Terminology).

## Korean style rules {#korean-style-rules}

These rules reflect decisions made by the Korean localization team and must be
applied consistently across all `content/ko/` files.

### Writing style (평어체)

Use plain declarative form (~다 endings) throughout technical documentation.
Do **not** use polite form (~습니다/~세요).

> Confirmed team-wide standard: PR #10431 briefly considered polite form as a
> homepage-only exception, but the merged homepage uses plain declarative form
> throughout (see `content/ko/_index.md`), matching the glossary (#10448).
> There is no per-page exception — apply 평어체 everywhere in `content/ko/`.

| Polite (avoid)  | Plain declarative (use) |
|-----------------|-------------------------|
| ~입니다         | ~이다                   |
| ~합니다         | ~한다                   |
| ~됩니다         | ~된다                   |
| ~참고하세요     | ~참고한다               |
| ~있습니다       | ~있다                   |

### OpenTelemetry notation

- **First occurrence** in a document: `오픈텔레메트리(OpenTelemetry)`
- **Subsequent occurrences**: `오픈텔레메트리` (no English in parentheses)
- Never use `오픈텔레메트리` without introducing the English form first.

### Line breaks

Don't hand-wrap at all — write the paragraph as prose (or a rough
one-sentence-per-line draft if that's easier while translating), then run
`npx prettier --write <file>` before committing and let it set the final
line breaks. See the [Prettier / formatting note](#prettier--formatting-note):
`content/ko` is hard-reflowed to 80 columns by CI, mid-sentence breaks
included — manual "one sentence per line" wrapping will not survive that
pass, so don't rely on it.
- Do **not** align line breaks with the English source (moot once prettier
  reflows it, but don't fight it while drafting either).
- Blank line between paragraphs (standard Markdown) — this part prettier
  preserves.

### English terms in parentheses

When first introducing a translated term, append the English original in
parentheses: `옵저버빌리티(observability)`, `속성(Attribute)`.
Subsequent uses in the same section may omit the parenthetical.

### Reference links

Keep reference links (`~참고한다`) in the **same paragraph** as the definition,
not split into a separate paragraph.

### Bold text immediately followed by a Korean particle {#bold-particle-pitfall}

**Pitfall**: `**한글(English)**는`/`을`/`이` (a Korean particle attached
directly after a closing `**` that is itself preceded by punctuation, e.g.
`)`) fails to render as bold. CommonMark's right-flanking-delimiter rule
requires a closing `**` preceded by punctuation to also be *followed* by
whitespace or punctuation — a Korean particle is a letter, not whitespace or
punctuation (Korean attaches particles with no space), so Goldmark leaves the
`**` un-rendered.

This actually shipped to production in `content/ko/_index.md`:
`**오픈텔레메트리(OpenTelemetry)**는` printed literal asterisks on
opentelemetry.io/ko/ (fixed by inserting a space after the closing `**`).

**Fix**: insert a plain space right after the closing `**`, before the
particle: `**한글(English)** 는`. Check for this pattern (parenthetical gloss
+ `**` + particle, no space) whenever you introduce bold text around a term
with an English gloss.

### API type identifiers keep English as the primary form {#api-identifiers}

Names a developer literally types in code — SDK instrument and class names like
`Counter`, `UpDownCounter`, `Gauge`, `Histogram` — are **not** everyday
technical concepts. They stay English, with the Korean phonetic form as the
*gloss*, inverting the usual Korean-primary order:

```markdown
- **Counter(카운터)**: 시간이 지남에 따라 누적되는 값이다.
- **Asynchronous Counter(비동기 카운터)**: **Counter**와 동일하지만 ...
```

Verified against both mature locales, which agree the identifier survives
translation: ja writes `**Counter（カウンター）**`, zh keeps a bare `**Counter**`.
Later prose *about* the thing may use the Korean form (`게이지는 동기적이다`) —
it's the identifier that stays English, not every mention.

Contrast with `배기지(baggage)` or `집계(Aggregation)`: those name concepts, not
API symbols, so Korean leads. The test is whether a reader would type the word
into an editor.

The same rule extends to literal schema/config identifiers a reader would
grep for verbatim — e.g. a Log Data Model's field-name column (`Timestamp`,
`SeverityText`, `Body`) or a CLI env-var/flag name — even when the identical
word also names a general concept elsewhere on the same page (`Resource` the
field vs. 리소스 the concept). When a collision like that is possible within
one document, add a short `> **표기 규칙**:` blockquote noting the field names
are literal identifiers, distinct from the general term.

### Image alt text and title {#image-alt-text}

Translate an image's **alt text and title attribute**; never touch the URL.
Confirmed against `observability-primer.md` in all three mature locales:

```markdown
![Sample Trace](/img/waterfall-trace.svg 'Trace waterfall diagram')   ← en
![トレースの例](/img/waterfall-trace.svg 'トレースのウォーターフォール図')  ← ja
![链路示例](/img/waterfall-trace.svg '链路瀑布图')                        ← zh
![샘플 트레이스](/img/waterfall-trace.svg '트레이스 워터폴 다이어그램')      ← ko
```

This is not a contradiction of "do not copy image assets" — the asset stays
shared and unduplicated; only the surrounding prose is localized.

## What the mentor actually flags {#mentor-review-patterns}

Distilled from every inline review comment on the merged ko PRs (#10431, #10448)
by @seokho-son, the ko localization mentor. Ranked by frequency — a reviewer
should check these first.

### 1. 영문 병기 — add the English gloss (most frequent request by far)

The mentor repeatedly asks for *more* glossing, not less:

> 영문을 병기하면 좋을 것 같습니다. (뉘앙스가 있는 기술적 용어라, 처음 읽는
> 사람들은 무슨 의미인지 파악하기가 어려울 수 있을 것 같아서요.)

Applied to `계측(instrument)`, `트레이스(traces)`, `메트릭(metrics)`,
`배기지(baggage)`, `옵저버빌리티(observability)`. The test is **reader
recognition**: a Korean reader meeting this term for the first time should be
able to map it back to the English they'll see in code and other docs.

**But not for ordinary Korean vocabulary** — the one place he asked to *remove*
a gloss:

> 영문을 굳이 병기하지 않아도, 한글로 의미가 충분히 전달된다고 생각하는데
> 어떠신가요?

...on `철자 및 대소문자 표기` (spelling and capitalization). So: gloss domain
terms, not plain words. `운영체제(operating system)` is a plain word — no gloss.

**The pipeline verbs are an exception — they DO get glossed.** `generate`,
`collect`, `receive`, `process`, `export` describe the OTel data pipeline and
function as domain terms here, not ordinary vocabulary. The merged
`glossary.md` glosses them in two separate entries:

```markdown
텔레메트리 데이터를 수신(receive), 처리(process), 내보내기(export)하기 위한 ...
[텔레메트리 데이터](/docs/concepts/signals/)를 생성(generate),
[수집(collect)](...) 및 [내보내기(export)](...)할 수 있다.
```

Verified 2026-09-03 against shipped `content/ko/docs/concepts/glossary.md`
after a review pass wrongly flagged `수집(collect), 처리(process)` as
over-glossing. **When a gloss decision is contested, grep the merged
translations before changing anything** — shipped state beats both the skill's
prose and a reviewer's reasoning.

### 2. Consistency within a single document

> 일관성 차원에서 정리가 필요해 보입니다.
> 문서 내에서 일관성 유지를 위해 수정해보았습니다.

He normalized `[X 스펙][x]을 참고한다` across every entry in the glossary, and
made 프로세싱/필터링/라우팅 all phonetic because two of the three already were.
If a page renders the same construct two ways, that's a finding — and this
applies across an entire section too, not just one file. A term rendered two
ways by two different translators working the same batch is the single most
common defect this project has actually found in review, over and over.

### 3. 원문 준용 — follow the source's form

> 원문 형태를 유지하자면, "이다."는 필요 없을 것 같습니다.
> 원문과 비슷하게 `,` 를 사용하면 어떨까요?

Don't add sentence-final verbs to what the source left as a noun phrase; don't
substitute `·` for the source's `,`; don't restructure a sentence the source
kept simple. This also means gloss **casing** follows the English source's own
dominant casing in that exact spot — lowercase where the source is lowercase
mid-sentence (`트레이스(trace)`, matching EN's own 18:7 lowercase-to-capital
ratio in that file), capital where the source is itself consistently
capitalized (`배기지(Baggage)`, matching EN's own 18:12 ratio the other way).
Check the actual source ratio per term rather than picking one rule for all
terms.

### 4. Notation rules that only apply to English

> "영문에서"를 추가해서, 한글 문서에서의 규칙은 아님을 알려드리는 것은 어떨까요?

A `> **표기 규칙**:` blockquote about English capitalization must say it governs
**영문에서** — otherwise a Korean reader takes it as a rule for Korean prose.

### 5. Contested and unresolved

- **Heading gloss capitalization.** He proposed lowercase throughout
  (`### 집계(aggregation)`), keeping the capital only where a notation rule
  demands it (`### 컬렉터(Collector)` — "참고: 첫 대문자를 유지"). Those
  suggestions were **not applied**; #10448 merged with Title Case
  (`### 집계(Aggregation)`). Follow the shipped state — Title Case — but know
  this thread is unresolved and may be reopened.
- **Style form.** #10431 debated 평어체 vs 존댓말 for the homepage, with a
  Kubernetes precedent for a homepage exception. The merged homepage has
  **zero** polite-form endings. No exception exists — 평어체 everywhere.
- **Maintainer vs. Administrator/Manager.** Multiple already-merged files
  (`community/_index.md`, `ecosystem/_includes/freeze-notice.md`,
  `docs/concepts/instrumentation/libraries.md`) use `관리자` for the OTel
  governance role "maintainer." A later translator independently coined
  `유지보수자` specifically to avoid `관리자` colliding with an unrelated
  "Slack channel manager" role in the same file. Current default: match
  existing precedent (`관리자`) per the "grep merged translations first"
  rule above — but this is a real, unresolved corpus-wide term question
  worth raising with @seokho-son rather than deciding unilaterally file by
  file.

## Structural conventions {#structural-conventions}

Patterns derived from `content/ko/docs/concepts/glossary.md`. Apply to all
`content/ko/**` files.

### Heading anchors

The English source uses implicit anchors from heading text (e.g., `### API`
implies `#api`). Korean titles break this, so **always attach an explicit
anchor** matching the English source's implicit or explicit anchor:

```markdown
### 집계(Aggregation) {#aggregation}
### 컨텍스트 전파 {#context-propagation}
### API {#api}
```

(The gloss in parentheses is a separate decision — see
[English annotation on first use](#english-annotation-on-first-use) — but the
anchor always attaches after it, unaffected by whether a gloss is present.)

The anchor is what internal links (`[집계](#aggregation)`) resolve to, so it
must match exactly. Never translate the anchor.

Two Goldmark quirks worth knowing rather than guessing at:
- Each space/slash in a heading is converted **separately**, so
  `Foo / Bar` → `foo--bar` (double hyphen), and `Step 1 - Start the X` →
  `step-1---start-the-x` (triple hyphen — space, dash, space each convert).
- A dotted identifier like `receiver.Factory` **drops the dot** rather than
  hyphenating it: `## Implementing the receiver.Factory interface` →
  `{#implementing-the-receiverfactory-interface}`. Verified against `ja`'s
  independently-produced translation of the same heading, which resolved to
  the identical anchor.

When genuinely unsure, build a one-page local Hugo test site with the exact
heading text and check the real output — cheaper than shipping a broken
in-page anchor link.

### Term treatment rule {#term-treatment-rule}

Decide per term whether the section title (and body mentions) translate to
Korean or stay in English:

- **Stay English** — acronyms (`API`, `HTTP`, `JSON`, `SDK`, `REST`, `RPC`,
  `DAG`, `APM`, `OC`, `OT`), proper nouns and brand names
  (`OpenTelemetry`, `OpenTracing`, `OpenCensus`, `OpAMP`, `OTLP`, `OTEP`,
  `OTelCol`, `gRPC`, `Contrib`, `zPages`, `Proto`).
- **Translate to Korean** — everyday technical concepts with an established
  Korean equivalent (`집계`, `애플리케이션`, `속성`, `카디널리티`, `컬렉터`,
  `데이터 소스`, `분산 트레이싱`, `배기지`).

> `Baggage` was reconsidered during #10431/#10448 review and now translates to
> `배기지` (phonetic), not "keep English" — do not treat it as a proper noun.

When in doubt, check the [Korean term table](#korean-term-table).

### English annotation on first use

For a **translated** term, append the English original in parentheses on first
use within a section:

- Full expansion for acronyms:
  `애플리케이션 프로그래밍 인터페이스(Application Programming Interface)`
- Concept phrase: `벤더 중립적인(vendor-agnostic)`, `근본 원인(root cause)`,
  `키-값 쌍`
- Common term: `카디널리티(Cardinality)`, `엔티티(Entity)`,
  `타임스탬프(timestamp)`

Subsequent mentions in the same section may drop the parenthetical.

**Capitalization of the parenthetical gloss**: match the term's *canonical*
casing, not a blanket lowercase rule:

- A term with an explicit notation rule (see any `> **표기 규칙**:`
  blockquote in `content/ko/docs/concepts/glossary.md`) always uses that
  casing: `오픈텔레메트리(OpenTelemetry)`, `컬렉터(Collector)`, `OpAMP`, `OTel`.
- In a **heading**, mirror the English glossary's own Title Case heading:
  `### 집계(Aggregation) {#aggregation}`, `### 카디널리티(Cardinality)
  {#cardinality}`, `### 배기지(Baggage) {#baggage}`.
- In **body prose**, use the source's natural sentence casing — usually
  lowercase for common nouns: `옵저버빌리티(observability)`,
  `벤더 중립적인(vendor-agnostic)`. But check the *specific term's* actual
  ratio in the English source before assuming lowercase — some terms (e.g.
  `Baggage`) are capitalized by the source throughout, not just at sentence
  starts, and should be glossed capitalized to match (`배기지(Baggage)`).
- Whichever you pick, stay consistent for that term across the whole file
  **and across sibling files in the same batch/section** — cross-file drift
  from parallel translators is the most common real defect found in review.

### Notation-rule blockquotes

Translate the label but keep the blockquote structure:

- `> **Spelling**:` → `> **표기 규칙**:` (or `> **표기**:` for shorter form)

Keep quoted English literals (like `` `OTEL` ``, `` `OpAMP` ``) unchanged —
they are the very thing the rule is about.

### Reference-style links

Reference labels are identifiers, not display text. **Keep them exactly as in
the English source**, including case:

```markdown
[분산 트레이싱][distributed tracing]     ← label stays English
[스팬][Span]                              ← label stays as source
[OpenTelemetry Enhancement Proposal][]    ← collapsed form, English kept
```

And keep the definitions at the bottom untouched except for translating
comments/labels only if they carry meaning:

```markdown
[distributed tracing]: ../signals/traces/
[OpenTelemetry Collector]: /docs/collector/
```

When a link's display text is a translated term, wrap the Korean text with the
English label: `[한국어 표시][English label]`.

**Collapsed reference form is a trap when you translate the display text.**
`[versions][]` (collapsed — display text IS the label) cannot simply become
`[버전][]`, because the label `versions` (lowercase, English) no longer
matches a translated display text. If you translate the display text, expand
to the two-part form instead: `[버전][versions]`, keeping the original label.
A file that gets this right once and then reverts to the bare collapsed form
later (leaving literal English inline in an otherwise Korean sentence) is a
real defect that has shipped in this project — check every occurrence of a
repeated warning/callout, not just its first instance.

> **Caution**: a reference label may be translated to Korean (e.g. new,
> single-purpose definitions), but only if you re-check *every* inline usage
> of that label in the file still resolves. `content/ko/docs/concepts/
> glossary.md` currently ships with a real mismatch from this exact mistake:
> the definition was changed to `[오픈텔레메트리 컬렉터]: /docs/collector/`
> but the inline uses still say `[오픈텔레메트리 컬렉터][OpenTelemetry
> Collector]` (English label) — a silently broken reference. Default to
> keeping labels in English exactly as in the source; only translate a label
> when you can verify all its usages in one pass.

### Front matter

- `title` — translate to Korean (e.g., `용어집`).
- `linkTitle` — translate to Korean if present; note the English source may
  deliberately shorten this for the sidebar nav versus a longer `title` — try
  to preserve that same length relationship rather than making both identical.
- `description` — translate to Korean. If `OpenTelemetry` appears in it, use
  the same notation as body prose — `오픈텔레메트리(OpenTelemetry)` — do not
  leave it as bare English. (Confirmed by the merged
  `content/ko/docs/concepts/glossary.md` front matter; this reverses an
  earlier "keep it English-only in description" assumption that never
  actually shipped.)
- `weight` — preserve as-is.
- `default_lang_commit` — set to git SHA of the English source (Step 1).
- Do not add front matter keys not present in the English source — but do
  check the source for an explicit comment like "DO NOT COPY the following
  config to non-en pages" (seen on `blog/_index.md`'s `link_check_exclude_path`)
  or "cascade is omitted because it is for the en locale only" — these mark
  specific keys as EN-only, not the whole front matter block. Keep every
  other key the comment doesn't call out (e.g. `menu:` on that same file is
  kept by every locale, including ko).

## Korean term table {#korean-term-table}

Established translations. Always prefer these over ad-hoc transliteration.

| English                  | Korean              | Source       |
|--------------------------|---------------------|--------------|
| Observability            | 옵저버빌리티          | #10431/#10448 (supersedes CNCF KO 관찰 가능성) |
| Distributed tracing      | 분산 트레이싱        | K8s KO       |
| Attribute                | 속성                | K8s KO       |
| Resource                 | 리소스              | K8s KO       |
| Service                  | 서비스              | K8s KO       |
| Metric / Metrics         | 메트릭              | OTel JP analogy |
| Trace / Traces           | 트레이스            | OTel JP analogy |
| Log / Logs               | 로그                | phonetic     |
| Span                     | 스팬                | phonetic     |
| Signal                   | 시그널              | phonetic     |
| Telemetry                | 텔레메트리          | phonetic; used throughout glossary.md |
| Collector                | 컬렉터              | phonetic     |
| Exporter                 | 익스포터            | phonetic     |
| Receiver                 | 리시버              | phonetic     |
| Processor                | 프로세서            | phonetic, matches 리시버/익스포터 |
| Connector                | 커넥터              | phonetic, same pattern |
| Extension                | 익스텐션            | phonetic, same pattern |
| Pipeline                 | 파이프라인          | phonetic     |
| Gateway (deployment)     | 게이트웨이          | phonetic     |
| Agent (deployment)       | 에이전트            | phonetic     |
| Aggregation              | 집계                | natural KO   |
| Sampling                 | 샘플링              | phonetic     |
| Head / Tail sampling     | 헤드 샘플링 / 테일 샘플링 | phonetic; Collector processor *names* (e.g. `Tail Sampling Processor`) stay English per #api-identifiers, the concept translates |
| Baggage                  | 배기지              | #10431/#10448 (phonetic; reversed from earlier "keep EN") |
| Context propagation      | 컨텍스트 전파        | phonetic+KO  |
| Downstream               | 다운스트림          | standard KO software term; NOT 하류 (reads as "downriver") |
| Ingestion                | 유입                | kept distinct from 수집 (collect) — do not conflate |
| Semantic conventions     | 시맨틱 컨벤션        | phonetic     |
| Cardinality              | 카디널리티          | phonetic     |
| Cardinality limit        | 카디널리티 제한      | #10448 (upstream sync 6524d4801) |
| Metadata                 | 메타데이터          | #10448       |
| Data source              | 데이터 소스         | #10448       |
| Observability backend    | 옵저버빌리티 백엔드   | #10448       |
| Observability frontend   | 옵저버빌리티 프론트엔드| #10448      |
| Instrumented library     | 계측된 라이브러리    | natural KO   |
| Instrumentation library  | 계측 라이브러리      | natural KO   |
| Automatic instrumentation| 자동 계측 (with the space) | natural KO — the space is load-bearing, a review finding fixed a file that dropped it |
| Manual instrumentation   | 수동 계측           | natural KO, mirrors 자동 계측 |
| Zero-code instrumentation| 제로 코드 계측       | natural KO   |
| Specification            | 명세                | natural KO; heading gloss `명세(Specification)` is the majority pattern across signal pages, though the merged glossary's own entry is bare — both are seen, prefer glossed to match the majority |
| Distribution (OTel)      | 배포판              | natural KO   |
| Distribution (Linux)     | 배포판              | same word, disambiguated by context; K8s/Linux-ecosystem usage |
| Extensibility             | 확장성              | natural KO (what-is-opentelemetry) |
| Entity                   | 엔티티              | phonetic     |
| Transaction              | 트랜잭션            | phonetic     |
| Dimension                | 차원                | natural KO   |
| Label                    | 레이블              | phonetic     |
| Field                    | 필드                | phonetic     |
| Tag                      | 태그                | phonetic     |
| Propagators              | 전파자              | natural KO; precedent for the general "-er/-or → 자" agentive pattern |
| Authenticator (Collector extension) | 인증자      | same agentive pattern as 전파자; keep as an agent noun, not an abstract "authentication" — verified the English source uses it as a countable component noun throughout |
| Status                   | 상태                | natural KO   |
| Application              | 애플리케이션        | phonetic     |
| Library                  | 라이브러리          | phonetic     |
| Language                 | 언어                | natural KO   |
| Event                    | 이벤트              | phonetic     |
| Request                  | 요청                | natural KO   |
| Configuration (noun)     | 구성                | NOT 설정 — a real, repeated cross-file defect; 설정 is only correct as the verb "to set" a specific value, never as the noun for "a configuration" or "the config file" |
| Kubernetes               | 쿠버네티스          | phonetic     |
| Operator (K8s)           | 오퍼레이터          | phonetic     |
| FaaS / Function as a Service | 서비스형 함수    | matches the established -aaS pattern (IaaS→서비스형 인프라, PaaS→서비스형 플랫폼, SaaS→서비스형 소프트웨어) |
| DaemonSet                | (keep English)      | K8s resource-kind identifier, no established KO translation found; matches how other literal K8s kind names stay English |
| Load balancing (exporter)| 로드 밸런싱          | phonetic     |
| Resolver                 | 리졸버              | phonetic     |
| Backpressure             | 백프레셔            | phonetic     |
| Batch / Batching         | 배치 / 배칭         | phonetic     |
| Maintainer (governance role) | 관리자          | existing precedent across 3+ merged files; contested — see mentor-review-patterns #5 |

## Validation checklist {#validation-checklist}

Walk this list against every translated file before handing off:

**Front matter**

- [ ] `title` and `description` translated to Korean.
- [ ] If `description` mentions OpenTelemetry, it uses
      `오픈텔레메트리(OpenTelemetry)`, not bare English.
- [ ] `default_lang_commit` set to git SHA of English source.
- [ ] `weight` preserved from source.
- [ ] No extra front matter keys introduced, and no EN-only key (per an
      explicit source comment) accidentally copied.

**Style (평어체)**

- [ ] No `~습니다`, `~입니다`, `~합니다`, `~됩니다`, `~세요` endings anywhere.
- [ ] All verb endings use plain declarative (`~이다` / `~한다` / `~된다` /
      `~참고한다`).

**OpenTelemetry notation**

- [ ] First occurrence uses `오픈텔레메트리(OpenTelemetry)`.
- [ ] Subsequent occurrences use `오픈텔레메트리` only.
- [ ] If `description` mentions OpenTelemetry, it also uses
      `오픈텔레메트리(OpenTelemetry)`.

**Structural**

- [ ] Every Korean heading has an explicit `{#english-anchor}` matching the
      source's implicit or explicit anchor — double-check double/triple-hyphen
      and dotted-identifier cases rather than assuming.
- [ ] Reference link labels (`[...][label]` and `[label]: url`) are unchanged
      from source; no collapsed-form `[translated-text][]` mismatch.
- [ ] Blockquote notation rules translated to `> **표기 규칙**:`; quoted
      English literals inside them preserved.
- [ ] Terms follow the [Term treatment rule](#term-treatment-rule) (acronyms /
      proper nouns stay English; concepts translate to Korean); API/schema
      identifiers follow [#api-identifiers](#api-identifiers) instead.
- [ ] No bold+particle rendering pitfall ([#bold-particle-pitfall](#bold-particle-pitfall)).

**Prose layout**

- [ ] Ran `npx prettier --write <file>` (or `npm run fix:format`) as the
      last step — don't trust hand-wrapped line breaks.
- [ ] Reference sentences kept in the same paragraph as the definition.
- [ ] Blank line between paragraphs.

**Terminology**

- [ ] Every translated term appears in the [term table](#korean-term-table) or
      has a documented source (CNCF KO / K8s KO / etc.).
- [ ] English annotation on first use of each translated term within a section.
- [ ] **Cross-file consistency**: if this file is one of several translated in
      the same batch/section, grep sibling files for shared terms before
      finishing — this is where most real defects actually turn up.

**Fidelity**

- [ ] Read the Korean back against the English sentence-by-sentence for at
      least the introduction and any dense/technical paragraph — check for
      dropped negation, reversed agency (passive→active changing who did
      what), and added or removed claims. A grammatically fluent Korean
      sentence that quietly changes the source's meaning passes every
      mechanical check above and is still wrong.

## Prettier / formatting note

**Correction (verified against real CI failures on #11536/#11537):**
`content/ko` is **not** exempted in `.prettierignore` the way `content/ja`,
`content/uk`, `content/zh` are — those three are listed under the "no prose
wrap" comment block; `ko` is not. That means `content/ko` files are checked
by CI's first format pass using the project-wide default
(`package.json`'s `"prettier"` key: `proseWrap: "always"`, printWidth 80) —
they get **hard-reflowed to 80 columns**, not preserved as manually wrapped.
Do **not** hand-wrap "one sentence per line" and expect it to survive CI —
it won't; prettier will re-wrap mid-sentence to fit the column width.

**Required step**: after writing or editing any `content/ko/**/*.md` file,
always run `npx prettier --write <file>` (or `npm run fix:format`) before
committing, and do not hand-tune line breaks afterward. This is not
optional — two merged fix PRs failed CI's `FILE FORMAT` check purely from
hand-wrapped lines not matching prettier's 80-col reflow, including a
1-line diff (adding a single space) that shifted wrap points for an entire
paragraph.

This project deliberately did **not** pursue getting `content/ko` added to
the prose-wrap-preserve exemption list (the two-line fix PR #7445 used for
`uk` is the exact recipe if that's ever wanted) — matching upstream's own
wrapping keeps diffs against peers' PRs and mentor review comments legible,
rather than fighting the formatter.

## References

- [Localization guide (primary authority)][localization guide]
- [pr-checks.md](../../../content/en/docs/contributing/pr-checks.md)
- [sig-practices.md](../../../content/en/docs/contributing/sig-practices.md)
- Reference PR for Korean homepage (style rules, term decisions):
  https://github.com/open-telemetry/opentelemetry.io/pull/10431
- Reference PR for Korean setup (tooling, `check:i18n`, formatting):
  https://github.com/open-telemetry/opentelemetry.io/pull/10440
- Reference PR for Korean glossary (structural conventions, term table):
  https://github.com/open-telemetry/opentelemetry.io/pull/10448
- Tracking issue (all `ko` locale setup, now closed):
  https://github.com/open-telemetry/opentelemetry.io/issues/9577
- CNCF Korean glossary: https://glossary.cncf.io/ko/
- Kubernetes Korean docs: https://kubernetes.io/ko/docs/
