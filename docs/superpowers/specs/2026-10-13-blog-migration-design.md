# Blog migration from kotakondo.com: design

Date: 2026-10-13. Status: approved in brainstorming, awaiting written-spec review.

## Goal

The WordPress site at kotakondo.com (server and domain) will be terminated entirely. Its Blog tab
(`kotakondo.com/posts/`) holds the only content not already on kotakondo.github.io: three Japanese posts
about applying to graduate school abroad. Move them into this site under a new **blog** tab, add an English
translation of each, and leave no dependency on kotakondo.com. Old kotakondo.com URLs are not preserved
(the domain is going away), and no backup of the rest of the WordPress site is wanted.

## Source content

Fetched from the WordPress 5.7 REST API (`/wp-json/wp/v2/posts`, `X-WP-Total: 3`). There are no comments, no
drafts, and no inline images. Media attached to other post IDs belong to `fw-portfolio` project pages,
which are out of scope.

| WP id | Date | WP slug | Title | Content |
| --- | --- | --- | --- | --- |
| 105 | 2021-05-07 | `application` | 海外大学院受験の振り返り | ~20 KB HTML: 16 headings, a results table, a blockquote, one Unyson `fw-tabs` block with 9 tabs (推薦状, 教授とのコネクション, 研究実績, 奨学金, Statement of Purpose (SoP), GPA, TOEFL, GRE, 面接), stray Amazon `a-size-*` spans |
| 304 | 2021-09-07 | `fos-application` | 船井情報科学振興財団奨学金の申請書を公開します！！ | text, an 応募要項 table, a `wp-block-file` link to `kotakondo_application_public.pdf` |
| 329 | 2021-11-17 | `mit_sop` | 海外大学院留学でMIT航空宇宙に提出したSoPを公開します！！ | text, a `wp-block-file` link to `MITSoPKotaKondoPublic.pdf` |

The three posts total about 9,100 Japanese characters. WordPress category: 大学院受験 and 海外留学 on all three.
WordPress tags (on post 105 only): GRE, SoP, TOEFL, 奨学金, 推薦状, 研究.

## Posts

Six files in a new `_posts/` directory, one Japanese original and one English translation per article, both
dated with the original WordPress publish date:

| Article | Japanese | English |
| --- | --- | --- |
| Retrospective (105) | `2021-05-07-grad-school-application-ja.md` | `2021-05-07-grad-school-application-en.md` |
| Funai application (304) | `2021-09-07-funai-scholarship-application-ja.md` | `2021-09-07-funai-scholarship-application-en.md` |
| MIT SoP (329) | `2021-11-17-mit-sop-ja.md` | `2021-11-17-mit-sop-en.md` |

The existing `permalink: /blog/:year/:title/` gives URLs such as `/blog/2021/mit-sop-ja/`.

Front matter on every post:

- `layout: post`, `title`, `date`, `description` (one sentence, shown in the blog list).
- `categories`: `日本語` or `English`. This is the language, and it drives the filter on the blog index.
- `tags`: the WordPress topics. The Japanese posts use the WordPress names, and the English posts use GRE, SoP,
  TOEFL, scholarships, recommendation letters, research. Retrospective: all six tags (as in WordPress). Funai
  application: 奨学金 / scholarships. MIT SoP: SoP. In WordPress the last two had no tags; these are added so
  every post has a topic.
- `thumbnail`: the article's cover image (shown in the blog list).
- `lang: ja` on Japanese posts only.
- `related_posts: false` (with six posts the box would mostly repeat the other-language twin).

Body, top to bottom:

1. A link to the other language: `[English version](…)` on Japanese posts, `[日本語版](…)` on English posts.
2. English posts only: *Translated from the Japanese original published <Month D, YYYY>.*
3. The cover image.
4. The converted article.

Conversion rules:

- Each of the 9 `fw-tabs` panes becomes a `###` subsection under 出願準備, in the original tab order. Headings
  that were nested inside a pane (Reading, Listening, …, Verbal, Quantitative, Writing) drop one level.
- The `<span>` heading anchors, Amazon `a-size-*`/`a-spacing-*` wrappers and other theme markup are removed.
  Their text is kept.
- Tables become GFM tables, the blockquote becomes a Markdown blockquote, and a `wp-block-file` becomes a
  plain link (`📄 [title (PDF)](/assets/pdf/blog/…)`).
- External links are kept exactly, including dead ones. Dead links are reported, not edited.
- Nothing may link to kotakondo.com.

## Translation

Faithful, sentence by sentence, in the author's voice, keeping every fact, number and hedge. A Japanese-only
proper noun gets a short English gloss in parentheses on first use (for example 船井情報科学振興財団 → the Funai
Foundation for Information Technology). Links stay as in the original even when they point to
Japanese-language pages. Each English post comes with a short list of judgment calls (glosses added, idioms
translated by meaning) for the author to review before merging.

## Assets

- `assets/img/blog/`: `grad-school-application.jpeg` (WP `2018/04/MIT.jpeg`),
  `funai-scholarship-application.png` (WP `2021/09/Screen-Shot-2021-09-06-at-2.47.23-PM-1.png`), `mit-sop.jpeg` (WP
  `2021/11/6E2E5D17-…_1_105_c-1.jpeg`). Each is 0.3–0.6 MB.
- `assets/pdf/blog/`: `funai-scholarship-application-kota-kondo.pdf` (WP `kotakondo_application_public.pdf`),
  `mit-sop-kota-kondo.pdf` (WP `MITSoPKotaKondoPublic.pdf`).
- Not copied: `publlic2021syogaku.pdf`, a byte-identical duplicate attached to post 304 that the post never
  links.

## Site changes

`_config.yml`:

- `blog_nav_title: blog`. This makes the theme render the tab.
- `blog_description`: a short bilingual line replacing al-folio's demo text (draft: *Notes on studying abroad
  and grad school applications / 海外大学院留学の記録*).
- `display_categories: ['日本語', 'English']` and `display_tags: []`, replacing the al-folio demo values.
- `external_sources: []`. The current value (al-folio's demo Medium feed) injects demo posts into
  `site.posts` on every build, and they would appear in the blog.
- `exclude:` gains `docs`, so this spec is not published as part of the site.
- `latest_posts` stays off on the homepage (`_pages/about.md` already has `latest_posts: false`).

Templates:

- `_includes/header.html`: move the Blog `<li>` from after "about" to after the page loop, so the nav reads
  about · publications · projects · cv · blog.
- `_layouts/default.html`: `<html lang="{{ page.lang | default: site.lang }}">`, so Japanese posts render kanji
  with Japanese glyphs. On a page declared `en`, Linux and Windows browsers may use Chinese glyph forms for
  Han characters.
- `blog/index.html`: read time for `lang: ja` posts is computed from the character count of the stripped
  content at ~500 characters per minute (rounded up, minimum 1), instead of words / 180. This applies to both the
  featured-card and list templates. English posts are unchanged.

`CLAUDE.md` (website):
- add a blog row to the content-model table and note the `lang`/category conventions;
- replace the "blog is effectively disabled" paragraph;
- add `_layouts/default.html`, `_includes/header.html` (blog position) and `blog/index.html` to "Local
  deviations from upstream al-folio";
- correct the tooling note: as of 2026-10-13 neither Ruby nor Docker is installed, and the build is checked
  through the PR artifact.

## Process

1. A throwaway script (session scratchpad, not committed) fetches the three posts from the REST API,
   downloads the five assets into the repo, and converts each post's HTML to a draft Japanese Markdown file.
2. Hand cleanup per the conversion rules.
3. Text check (Japanese posts): strip each WordPress original and each Markdown file to plain text and
   compare character by character. The only allowed differences are listed, deliberate ones (tab chrome
   becoming headings, removed Amazon markup). Anything else is fixed before continuing.
4. English translations.
5. Site changes.
6. Link check: request every external link once and report non-2xx/3xx responses to the author.

## Verification and release

There is no local Ruby, Jekyll or Docker, so the build is checked in CI:

- Work happens on branch `blog-migration`. Pushing it and opening a PR needs the author to run
  `gh auth login` once and to approve the push.
- `.github/workflows/deploy.yml` gains an `actions/upload-artifact` step that runs only on
  `pull_request`, uploads `_site`, and keeps it 3 days. The existing deploy step already skips PRs.
- Download the artifact with `gh run download`, serve it locally (`python3 -m http.server`), and check it in
  Chrome:
  - the nav order;
  - `/blog/` with both language filters, and `/blog/category/日本語/` and `/blog/category/english/`;
  - one Japanese and one English post at desktop and phone width (`lang="ja"` present, the cross-links
    resolve, PDFs and images load, the 9 former tabs read as subsections);
  - the read times;
  - no al-folio demo posts anywhere.
- The author reviews the PR (especially the English text) and merges to `master`, which deploys. A failed
  build never deploys, so the live site cannot be broken by a build error.

## Out of scope

The WordPress Portfolio, Resume, About, Contact and Sample pages; a backup of the WordPress site; redirects from
kotakondo.com; a language toggle within a single post; comments (giscus stays unconfigured).
