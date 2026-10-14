# Blog Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the three Japanese posts from kotakondo.com (WordPress) into this al-folio site as a new **blog** tab.
Each post gets a faithful English translation, and the site keeps no dependency on kotakondo.com.

**Architecture:** A throwaway Python script, kept in the session scratchpad, pulls the posts from the WordPress REST API
and converts them to Markdown. Hand cleanup and a character-level text check prove the Japanese survived intact.
English twins are checked for structural parity with the Japanese. Small config and template changes turn on al-folio's
built-in blog. Since this machine has no Ruby or Docker, the built site is verified from a GitHub Actions PR
artifact: a deliberately "red" first build, then a "green" one after the site changes. A final visual pass runs in
Chrome.

**Tech Stack:** Jekyll / al-folio (Liquid, kramdown GFM), GitHub Actions, Python run through `uv` (beautifulsoup4,
markdownify, markdown-it-py, pyyaml), `gh` CLI, Claude in Chrome.

**Spec:** `docs/superpowers/specs/2026-10-13-blog-migration-design.md`. Read it before starting.

## Global Constraints

- Work only on branch `blog-migration`. Never commit to or push `master`. Merging (which deploys) is the author's call.
- Commit as `kotakondo <kkondo@mit.edu>`, passed per command with `-c` (no git config exists on this machine). Every
  commit message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. PR bodies end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- Nothing in `_posts/`, `_config.yml`, `_includes/`, `_layouts/`, `_pages/` or `blog/` may contain `kotakondo.com`.
- The Japanese post bodies are the WordPress text verbatim, and `text_check.py` must print `OK` for all three. English
  follows the translation rules in Task 3, and `parity_check.py` must print `OK` for all three.
- Markdown: one paragraph per line, an explicit `<br>` at the end of a line for in-paragraph breaks (kramdown
  `hard_wrap` is off), no trailing whitespace.
- Scripts, drafts and downloads live in the scratchpad and are never committed. Downloaded WordPress data goes in
  `$SRC` only. Run every Python script as `$PY script.py …` (it uses `python -I`).
- Pinned tool versions: beautifulsoup4 4.12.3, markdownify 0.13.1, markdown-it-py 3.0.0, pyyaml 6.0.2,
  `actions/upload-artifact@v4`.
- Outward-facing actions (`git push`, `gh pr create`, `gh pr edit`, `gh pr ready`, `gh pr merge`) happen only after the
  author says yes in the conversation. The author runs `gh auth login` themselves.

## Review Focus

1. **Non-ASCII archive URLs.** The `日本語` filter link and tag links such as `/blog/tag/推薦状/` must open their
   archive pages, not 404. Pinned by `check_site.py`'s "local links resolve" check on every post page plus the
   category-page checks (Tasks 4–5).
2. **Displayed dates.** Each post must show its original day (May 7, Sep 7 and Nov 17, 2021) even though CI builds in
   UTC. Pinned by `check_site.py`'s date check (Tasks 4–5).
3. **Phone width (390 px).** The long post with the 3-column results table must not scroll sideways. Pinned by the
   `scrollWidth` check in Task 6, which has a prepared fix.
4. **Homepage unchanged.** No posts list appears, and the nav only gains `blog` at the end. Pinned by `check_site.py`'s
   homepage checks (Tasks 4–5).
5. **Dark mode.** Post text, the tables and the blockquote stay readable. Pinned by the dark-mode screenshot step in
   Task 6.

## Shared environment

Shell state does not persist between Bash calls, so **every command block below starts with**
`source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh`
(written in Task 1, Step 1). If you execute this plan in a different Claude Code session, replace `SCRATCH` (here and
in `env.sh`) with that session's scratchpad directory.

---

### Task 1: Copy the cover images and PDFs into the repo

**Files:**
- Create: `assets/img/blog/grad-school-application.jpeg`, `assets/img/blog/funai-scholarship-application.png`,
  `assets/img/blog/mit-sop.jpeg`
- Create: `assets/pdf/blog/funai-scholarship-application-kota-kondo.pdf`, `assets/pdf/blog/mit-sop-kota-kondo.pdf`
- Scratch: `$WORK/env.sh`, `$WORK/check_assets.py`, `$SRC/posts_api.json`

**Interfaces:**
- Produces: the five asset paths above, which later tasks reference verbatim. `$SRC/posts_api.json` is the WordPress
  REST response (a list of 3 post objects with ids 105, 304, 329). `env.sh` defines `REPO SCRATCH SRC WORK SITE PY
  GH_REPO` and a `commit "<subject>"` shell function.

- [ ] **Step 1: Write the shared environment file**

```bash
mkdir -p /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration
cat > /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh <<'EOF'
REPO=/home/kkondo/papers/kotakondo.github.io
SCRATCH=/tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad
SRC=$SCRATCH/kotakondo_com        # downloaded WordPress data (untrusted input, never executed)
WORK=$SCRATCH/blog_migration      # throwaway scripts, drafts, reports (never committed)
SITE=$SCRATCH/site_build          # CI-built _site, downloaded from the PR artifact
PY="uv run --no-project --quiet --with beautifulsoup4==4.12.3 --with markdownify==0.13.1 --with markdown-it-py==3.0.0 --with pyyaml==6.0.2 python -I"
GH_REPO=kotakondo/kotakondo.github.io
commit() {
  git -C "$REPO" -c user.name=kotakondo -c user.email=kkondo@mit.edu commit -q -m "$1" \
    -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
}
cd "$WORK"
EOF
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh && git -C $REPO status -sb | head -1
```

Expected: `## blog-migration` (no changes listed).

- [ ] **Step 2: Write the failing asset check** at `$WORK/check_assets.py`

```python
"""Fail unless the five migrated assets are in the repo with the expected size and file type.

Usage: check_assets.py <repo>
"""
import sys
from pathlib import Path

ASSETS = {  # path: (bytes as served by kotakondo.com, leading magic bytes)
    "assets/img/blog/grad-school-application.jpeg": (584523, b"\xff\xd8\xff"),
    "assets/img/blog/funai-scholarship-application.png": (612945, b"\x89PNG"),
    "assets/img/blog/mit-sop.jpeg": (290857, b"\xff\xd8\xff"),
    "assets/pdf/blog/funai-scholarship-application-kota-kondo.pdf": (253982, b"%PDF"),
    "assets/pdf/blog/mit-sop-kota-kondo.pdf": (76991, b"%PDF"),
}


def main(repo: str) -> int:
    bad = 0
    for rel, (size, magic) in ASSETS.items():
        path = Path(repo) / rel
        if not path.exists():
            print(f"FAIL {rel}: missing"); bad += 1; continue
        data = path.read_bytes()
        if len(data) != size or not data.startswith(magic):
            print(f"FAIL {rel}: {len(data)} bytes, starts {data[:4]!r}"); bad += 1
        else:
            print(f"OK   {rel}")
    return 1 if bad else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

- [ ] **Step 3: Run it to confirm it fails**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh && $PY $WORK/check_assets.py $REPO; echo "exit=$?"
```

Expected: five `FAIL …: missing` lines, `exit=1`.

- [ ] **Step 4: Download the post data and the five assets**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
mkdir -p $SRC $REPO/assets/img/blog $REPO/assets/pdf/blog
UA='Mozilla/5.0'; U=http://kotakondo.com/wp-content/uploads
curl -fsS -A "$UA" -o $SRC/posts_api.json 'http://kotakondo.com/wp-json/wp/v2/posts?per_page=100'
curl -fsS -A "$UA" -o $REPO/assets/img/blog/grad-school-application.jpeg       $U/2018/04/MIT.jpeg
curl -fsS -A "$UA" -o $REPO/assets/img/blog/funai-scholarship-application.png  $U/2021/09/Screen-Shot-2021-09-06-at-2.47.23-PM-1.png
curl -fsS -A "$UA" -o $REPO/assets/img/blog/mit-sop.jpeg                       $U/2021/11/6E2E5D17-D2B5-4AD5-8628-0AD406297196_1_105_c-1.jpeg
curl -fsS -A "$UA" -o $REPO/assets/pdf/blog/funai-scholarship-application-kota-kondo.pdf $U/2021/09/kotakondo_application_public.pdf
curl -fsS -A "$UA" -o $REPO/assets/pdf/blog/mit-sop-kota-kondo.pdf             $U/2021/11/MITSoPKotaKondoPublic.pdf
python3 -I -c "import json,sys; print([p['id'] for p in json.load(open(sys.argv[1]))])" $SRC/posts_api.json
```

Expected: `[329, 304, 105]`. If kotakondo.com no longer answers, stop and tell the author: the content can no longer be
fetched.

- [ ] **Step 5: Run the asset check and look at the images**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh && $PY $WORK/check_assets.py $REPO; echo "exit=$?"
```

Expected: five `OK` lines, `exit=0`. Then open the three images with the Read tool and confirm:
- `grad-school-application.jpeg` is MIT's Great Dome at dusk;
- `funai-scholarship-application.png` is the Funai Foundation (FFIT) website banner;
- `mit-sop.jpeg` is MIT's Great Dome in daylight.

- [ ] **Step 6: Commit**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO add assets/img/blog assets/pdf/blog && commit "Add blog cover images and PDFs from kotakondo.com" && git -C $REPO log --oneline -1
```

---

### Task 2: Japanese posts (converted, verified verbatim)

**Files:**
- Create: `_posts/2021-05-07-grad-school-application-ja.md`, `_posts/2021-09-07-funai-scholarship-application-ja.md`,
  `_posts/2021-11-17-mit-sop-ja.md`
- Scratch: `$WORK/check_posts.py`, `$WORK/text_check.py`, `$WORK/convert.py`, `$WORK/drafts/*.body.md`

**Interfaces:**
- Consumes: `$SRC/posts_api.json` and the asset paths from Task 1.
- Produces: the three `-ja.md` files. Their front matter is exactly as in Step 5. The body starts with the preamble line
  `[English version]({% post_url <base>-en %})`, then a blank line, then the `{% include figure.html … %}` line. Later
  tasks rely on this: `check_posts.py` (used again in Task 3) checks both language variants, and Task 3 mirrors the
  structure.

- [ ] **Step 1: Write the post checker** at `$WORK/check_posts.py`

```python
"""Check the blog posts' front matter, preamble, local links and whitespace against the spec.

Usage: check_posts.py <repo> ja|en|all
"""
import datetime as dt
import re
import sys
from pathlib import Path

import yaml

ARTICLES = {
    "2021-05-07-grad-school-application": {
        "thumb": "assets/img/blog/grad-school-application.jpeg", "published": "May 7, 2021",
        "tags": {"ja": ["GRE", "SoP", "TOEFL", "奨学金", "推薦状", "研究"],
                 "en": ["GRE", "SoP", "TOEFL", "scholarships", "recommendation letters", "research"]}},
    "2021-09-07-funai-scholarship-application": {
        "thumb": "assets/img/blog/funai-scholarship-application.png", "published": "September 7, 2021",
        "tags": {"ja": ["奨学金"], "en": ["scholarships"]}},
    "2021-11-17-mit-sop": {
        "thumb": "assets/img/blog/mit-sop.jpeg", "published": "November 17, 2021",
        "tags": {"ja": ["SoP"], "en": ["SoP"]}},
}
CATEGORY = {"ja": "日本語", "en": "English"}
TIME = {"ja": dt.time(12, 0), "en": dt.time(12, 1)}
OTHER = {"ja": "en", "en": "ja"}
CROSS = {"ja": "English version", "en": "日本語版"}
LOCAL = re.compile(r"\]\((/[^)\s]*)\)")


def problems(repo: Path, base: str, lang: str) -> list:
    spec = ARTICLES[base]
    path = repo / "_posts" / f"{base}-{lang}.md"
    if not path.exists():
        return ["missing"]
    text = path.read_text(encoding="utf-8")
    _, fm_text, body = text.split("---", 2)
    fm = yaml.safe_load(fm_text)
    out = []
    want = {"layout": "post", "categories": [CATEGORY[lang]], "tags": spec["tags"][lang],
            "thumbnail": spec["thumb"], "related_posts": False}
    for key, value in want.items():
        if fm.get(key) != value:
            out.append(f"{key} is {fm.get(key)!r}, want {value!r}")
    for key in ("title", "description"):
        if not isinstance(fm.get(key), str) or not fm[key].strip():
            out.append(f"{key} missing")
    date = fm.get("date")
    if (not isinstance(date, dt.datetime) or date.date().isoformat() != base[:10]
            or date.time() != TIME[lang] or date.tzinfo is not None):
        out.append(f"date is {date!r}, want {base[:10]} {TIME[lang]} with no UTC offset")
    if lang == "ja" and fm.get("lang") != "ja":
        out.append(f"lang is {fm.get('lang')!r}, want 'ja'")
    if lang == "en" and "lang" in fm:
        out.append("English post must not set lang")
    if not (repo / spec["thumb"]).is_file():
        out.append(f"thumbnail file {spec['thumb']} missing")
    lines = [line for line in body.splitlines() if line.strip()]
    expect = [f"[{CROSS[lang]}]({{% post_url {base}-{OTHER[lang]} %}})"]
    if lang == "en":
        expect.append(f"*Translated from the Japanese original published {spec['published']}.*")
    if len(lines) <= len(expect):
        return out + ["body too short"]
    for i, line in enumerate(expect):
        if lines[i] != line:
            out.append(f"preamble line {i + 1} is {lines[i]!r}, want {line!r}")
    fig = lines[len(expect)]
    if not (fig.startswith(f'{{% include figure.html path="{spec["thumb"]}"') and re.search(r'alt="[^"]+"', fig)):
        out.append(f"cover image line is {fig!r}")
    for target in LOCAL.findall(body):
        if not (repo / target.lstrip("/")).is_file():
            out.append(f"local link {target} does not exist")
    if "kotakondo.com" in text:
        out.append("links to kotakondo.com")
    if re.search(r"[ \t]+$", text, re.M):
        out.append("trailing whitespace")
    return out


def main(repo: str, which: str) -> int:
    langs = ["ja", "en"] if which == "all" else [which]
    failed = False
    for base in ARTICLES:
        for lang in langs:
            ps = problems(Path(repo), base, lang)
            print(("OK   " if not ps else "FAIL ") + f"{base}-{lang}.md" + "".join(f"\n       {p}" for p in ps))
            failed = failed or bool(ps)
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1], sys.argv[2]))
```

- [ ] **Step 2: Write the Japanese text-fidelity check** at `$WORK/text_check.py`

```python
"""Fail unless each Japanese post's text equals its WordPress original (whitespace-insensitive).

Usage: text_check.py <posts_api.json> <repo>/_posts
"""
import difflib
import json
import re
import sys
from pathlib import Path

from bs4 import BeautifulSoup
from markdown_it import MarkdownIt

FILES = {105: "2021-05-07-grad-school-application-ja.md",
         304: "2021-09-07-funai-scholarship-application-ja.md",
         329: "2021-11-17-mit-sop-ja.md"}
# Deliberate, spec-listed differences: (text in WordPress, text in Markdown), after whitespace removal.
ALLOWED = [
    ("kotakondo_application_publicDownload", "📄funai-scholarship-application-kota-kondo.pdf"),
    ("MITSoPKotaKondoPublicDownload", "📄mit-sop-kota-kondo.pdf"),
]
WS = re.compile(r"\s+")  # str-mode \s also matches U+3000 (full-width space) and U+00A0 (&nbsp;)


def wp_text(html: str) -> str:
    soup = BeautifulSoup(html, "html.parser")
    for box in soup.select(".fw-tabs-container"):          # tab titles move in front of their panes
        for a in box.select(".fw-tabs ul li a"):
            box.select_one(a["href"]).insert(0, a.get_text(strip=True) + " ")
        box.select_one(".fw-tabs").decompose()
    return WS.sub("", soup.get_text(" "))


def md_text(path: Path) -> str:
    body = path.read_text(encoding="utf-8").split("---", 2)[2]      # drop front matter
    body = "\n".join(line for line in body.splitlines()
                     if not line.startswith(("[English version]", "{% include figure.html")))
    html = MarkdownIt("commonmark").enable("table").render(body)
    return WS.sub("", BeautifulSoup(html, "html.parser").get_text(" "))


def main(api_json: str, posts_dir: str) -> int:
    posts = {p["id"]: p for p in json.load(open(api_json, encoding="utf-8"))}
    failed = False
    for pid, name in FILES.items():
        want = wp_text(posts[pid]["content"]["rendered"])
        for old, new in ALLOWED:
            want = want.replace(old, new)
        path = Path(posts_dir) / name
        if not path.exists():
            print(f"FAIL {name}: missing"); failed = True; continue
        got = md_text(path)
        if got == want:
            print(f"OK   {name}: {len(got)} chars identical")
            continue
        failed = True
        print(f"FAIL {name}: text differs")
        for op, a0, a1, b0, b1 in difflib.SequenceMatcher(None, want, got, autojunk=False).get_opcodes():
            if op != "equal":
                print(f"  {op}: WP[{want[max(0, a0 - 15):a1 + 15]}]  MD[{got[max(0, b0 - 15):b1 + 15]}]")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1], sys.argv[2]))
```

- [ ] **Step 3: Run both checks to confirm they fail**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/check_posts.py $REPO ja; echo "exit=$?"; $PY $WORK/text_check.py $SRC/posts_api.json $REPO/_posts; echo "exit=$?"
```

Expected: three `FAIL … missing` lines from each script, `exit=1` twice.

- [ ] **Step 4: Write the converter** at `$WORK/convert.py` and generate the drafts

```python
"""Draft-convert the three WordPress posts to Markdown bodies (throwaway; see the blog-migration plan).

Usage: convert.py <posts_api.json> <out_dir>
"""
import json
import re
import sys
from pathlib import Path

from bs4 import BeautifulSoup
from markdownify import markdownify

SLUGS = {105: "grad-school-application", 304: "funai-scholarship-application", 329: "mit-sop"}
PDFS = {
    "kotakondo_application_public.pdf": "funai-scholarship-application-kota-kondo.pdf",
    "MITSoPKotaKondoPublic.pdf": "mit-sop-kota-kondo.pdf",
}


def md(html: str) -> str:
    return markdownify(html, heading_style="ATX", bullets="-", escape_misc=False)


def flatten_tabs(soup: BeautifulSoup) -> None:
    """Replace the fw-tabs widget with one <h3> per tab followed by that pane's content."""
    for box in soup.select(".fw-tabs-container"):
        parts = []
        for a in box.select(".fw-tabs ul li a"):
            pane = box.select_one(a["href"])  # "#fw-tabs-<id>-<n>"
            for h1 in pane.find_all("h1"):
                h1.name = "h4"
            parts.append(f"<h3>{a.get_text(strip=True)}</h3>{pane.decode_contents()}")
        box.replace_with(BeautifulSoup("".join(parts), "html.parser"))


def table_to_paragraphs(soup: BeautifulSoup) -> None:
    """The Funai 応募要項 table is one column of alternating title/body rows: emit bold titles + body lines."""
    for fig in soup.select("figure.wp-block-table"):
        out = []
        for td in fig.select("td"):
            if td.find("strong") and td.get_text(strip=True) == td.strong.get_text(strip=True):
                out.append(f"<p><strong>{td.strong.get_text(strip=True)}</strong></p>")
            else:
                out.append(f"<p>{td.decode_contents()}</p>")
        fig.replace_with(BeautifulSoup("".join(out), "html.parser"))


def file_blocks(soup: BeautifulSoup) -> None:
    for div in soup.select("div.wp-block-file"):
        new = PDFS[div.a["href"].rsplit("/", 1)[-1]]
        div.replace_with(BeautifulSoup(f'<p>📄 <a href="/assets/pdf/blog/{new}">{new}</a></p>', "html.parser"))


def convert(post: dict) -> str:
    soup = BeautifulSoup(post["content"]["rendered"], "html.parser")
    for h3 in soup.select(".block-title h3"):
        h3.name = "h2"
    for h2 in soup.select("h2"):
        for span in h2.find_all("span"):
            span.decompose()
    flatten_tabs(soup)
    table_to_paragraphs(soup)
    file_blocks(soup)
    text = md(str(soup))
    text = text.replace("  \n", "<br>\n")                      # markdownify's <br> -> explicit <br>
    text = re.sub(r"^[ \t ]+$", "", text, flags=re.M)     # blank-looking lines (&nbsp; paragraphs)
    text = re.sub(r"[ \t]+$", "", text, flags=re.M)            # trailing whitespace
    return re.sub(r"\n{3,}", "\n\n", text).strip() + "\n"


def main(api_json: str, out_dir: str) -> None:
    out = Path(out_dir)
    out.mkdir(parents=True, exist_ok=True)
    for post in json.load(open(api_json, encoding="utf-8")):
        path = out / f"{SLUGS[post['id']]}.body.md"
        path.write_text(convert(post), encoding="utf-8")
        print(f"wrote {path} ({path.stat().st_size} bytes)")


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2])
```

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/convert.py $SRC/posts_api.json $WORK/drafts && grep -c '^#' $WORK/drafts/*.body.md
```

Expected:
- three `wrote …` lines;
- heading counts `grad-school-application.body.md:25`, `funai-scholarship-application.body.md:1`, `mit-sop.body.md:0`.

The retrospective's headings are `##` for はじめに, 略歴, 海外大学院を目指したきっかけ, 出願結果, 出願準備, 進学先選び, まとめ and 参考になるリンク集.
The nine tabs are `###` (推薦状 … 面接), and Reading/Listening/Speaking/Writing/受験直前の対策/Verbal/Quantitative/Writing are `####`.

- [ ] **Step 5: Assemble the three Japanese post files**

Each file is its front matter block, then the preamble, then the draft body. Run:

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
mkdir -p $REPO/_posts
cat > $REPO/_posts/2021-05-07-grad-school-application-ja.md <<'EOF'
---
layout: post
title: 海外大学院受験の振り返り
date: 2021-05-07 12:00:00
description: 今回は、自分が海外大学院受験を通して得た経験や反省を、来年以降に海外大学院を受験される方に生かしてもらいたいと思い、受験の振り返り記事を書くことにしました。
categories: [日本語]
tags: [GRE, SoP, TOEFL, 奨学金, 推薦状, 研究]
thumbnail: assets/img/blog/grad-school-application.jpeg
lang: ja
related_posts: false
---
[English version]({% post_url 2021-05-07-grad-school-application-en %})

{% include figure.html path="assets/img/blog/grad-school-application.jpeg" class="img-fluid rounded z-depth-1" alt="MITのグレートドーム" %}

EOF
cat $WORK/drafts/grad-school-application.body.md >> $REPO/_posts/2021-05-07-grad-school-application-ja.md

cat > $REPO/_posts/2021-09-07-funai-scholarship-application-ja.md <<'EOF'
---
layout: post
title: 船井情報科学振興財団奨学金の申請書を公開します！！
date: 2021-09-07 12:00:00
description: 海外大学院留学のために支援をしていただいている船井情報科学振興財団（以下、船井財団）の昨年度の僕の申請書を公開します！！
categories: [日本語]
tags: [奨学金]
thumbnail: assets/img/blog/funai-scholarship-application.png
lang: ja
related_posts: false
---
[English version]({% post_url 2021-09-07-funai-scholarship-application-en %})

{% include figure.html path="assets/img/blog/funai-scholarship-application.png" class="img-fluid rounded z-depth-1" alt="船井情報科学振興財団ウェブサイトのバナー" %}

EOF
cat $WORK/drafts/funai-scholarship-application.body.md >> $REPO/_posts/2021-09-07-funai-scholarship-application-ja.md

cat > $REPO/_posts/2021-11-17-mit-sop-ja.md <<'EOF'
---
layout: post
title: 海外大学院留学でMIT航空宇宙に提出したSoPを公開します！！
date: 2021-11-17 12:00:00
description: 海外大学院に必要となるエッセイ（Statement of Purpose, SoP）を公開します！
categories: [日本語]
tags: [SoP]
thumbnail: assets/img/blog/mit-sop.jpeg
lang: ja
related_posts: false
---
[English version]({% post_url 2021-11-17-mit-sop-en %})

{% include figure.html path="assets/img/blog/mit-sop.jpeg" class="img-fluid rounded z-depth-1" alt="MITのグレートドーム" %}

EOF
cat $WORK/drafts/mit-sop.body.md >> $REPO/_posts/2021-11-17-mit-sop-ja.md
```

The `{% post_url … -en %}` targets do not exist until Task 3. That is fine: nothing builds before Task 4.

- [ ] **Step 6: Hand cleanup: split the bold across the line break** in
  `_posts/2021-09-07-funai-scholarship-application-ja.md` (with the Edit tool)

Old:
```markdown
**２０２２年度 Funai Overseas Scholarship<br>
大学院（博士号）留学　応募要項**
```
New:
```markdown
**２０２２年度 Funai Overseas Scholarship**<br>
**大学院（博士号）留学　応募要項**
```

Then read all three files end to end. Fix only conversion artifacts, never wording. Artifacts include a stray `\`
escape, an empty list item, or a line ending in whitespace. The prototype run on 2026-10-13 produced none besides
the edit above.

- [ ] **Step 7: Run both checks to confirm they pass**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/check_posts.py $REPO ja; echo "exit=$?"; $PY $WORK/text_check.py $SRC/posts_api.json $REPO/_posts; echo "exit=$?"
```

Expected:
```
OK   2021-05-07-grad-school-application-ja.md
OK   2021-09-07-funai-scholarship-application-ja.md
OK   2021-11-17-mit-sop-ja.md
exit=0
OK   2021-05-07-grad-school-application-ja.md: 9231 chars identical
OK   2021-09-07-funai-scholarship-application-ja.md: 1766 chars identical
OK   2021-11-17-mit-sop-ja.md: 271 chars identical
exit=0
```

A `FAIL … text differs` shows the WP vs MD context. Restore the missing or changed text from the draft. Do not add it
to `ALLOWED`: only the two file-block substitutions are spec-sanctioned.

- [ ] **Step 8: Commit**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO add _posts && commit "Add the three Japanese blog posts from kotakondo.com" && git -C $REPO log --oneline -1
```

---

### Task 3: English translations

**Files:**
- Create: `_posts/2021-05-07-grad-school-application-en.md`, `_posts/2021-09-07-funai-scholarship-application-en.md`,
  `_posts/2021-11-17-mit-sop-en.md`
- Scratch: `$WORK/parity_check.py`, `$WORK/judgment_calls.md`

**Interfaces:**
- Consumes: the three `-ja.md` files (Task 2) and `check_posts.py`.
- Produces: three `-en.md` files whose front matter is exactly as in Step 3. The body starts with
  `[日本語版]({% post_url <base>-ja %})`, a blank line, the italic "Translated from…" line, a blank line, the
  `{% include figure.html … %}` line, then the translation. Also produces `$WORK/judgment_calls.md`, used in the PR body in
  Task 4.

- [ ] **Step 1: Write the parity check** at `$WORK/parity_check.py`

```python
"""Fail unless each English post has the same document structure as its Japanese original.

Usage: parity_check.py <repo>/_posts
"""
import re
import sys
from pathlib import Path

from bs4 import BeautifulSoup
from markdown_it import MarkdownIt

BASES = ["2021-05-07-grad-school-application", "2021-09-07-funai-scholarship-application", "2021-11-17-mit-sop"]
PREAMBLE = ("[English version]", "[日本語版]", "*Translated from", "{% include figure.html")


def shape(path: Path) -> dict:
    body = path.read_text(encoding="utf-8").split("---", 2)[2]
    body = "\n".join(line for line in body.splitlines() if not line.startswith(PREAMBLE))
    s = BeautifulSoup(MarkdownIt("commonmark").enable("table").render(body), "html.parser")
    return {
        "headings": [h.name for h in s.find_all(re.compile(r"^h[1-6]$"))],
        "paragraphs": len(s.find_all("p")),
        "line breaks": len(s.find_all("br")),
        "bold spans": len(s.find_all("strong")),
        "links": [a["href"] for a in s.find_all("a")],
        "tables": [(len(t.find_all("tr")), len(t.find("tr").find_all(["th", "td"]))) for t in s.find_all("table")],
        "lists": [len(lst.find_all("li", recursive=False)) for lst in s.find_all(["ul", "ol"])],
        "blockquotes": len(s.find_all("blockquote")),
    }


def main(posts_dir: str) -> int:
    failed = False
    for base in BASES:
        ja, en = Path(posts_dir) / f"{base}-ja.md", Path(posts_dir) / f"{base}-en.md"
        if not en.exists():
            print(f"FAIL {en.name}: missing"); failed = True; continue
        a, b = shape(ja), shape(en)
        diffs = [k for k in a if a[k] != b[k]]
        if not diffs:
            print(f"OK   {en.name}: same structure as {ja.name}")
        for k in diffs:
            failed = True
            print(f"FAIL {en.name}: {k} differ\n       ja: {a[k]}\n       en: {b[k]}")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

- [ ] **Step 2: Run the checks to confirm they fail**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/parity_check.py $REPO/_posts; echo "exit=$?"; $PY $WORK/check_posts.py $REPO en; echo "exit=$?"
```

Expected: three `FAIL … missing` lines from each, `exit=1` twice.

- [ ] **Step 3: Write the three English files** with the Write tool: the exact front matter and preamble below, then
  the translated body.

`_posts/2021-05-07-grad-school-application-en.md`:
```markdown
---
layout: post
title: Looking Back on Applying to Graduate Schools Abroad
date: 2021-05-07 12:01:00
description: I decided to write this retrospective so that people applying to graduate schools abroad in the coming years can make use of the experiences and lessons I gained through my own applications.
categories: [English]
tags: [GRE, SoP, TOEFL, scholarships, recommendation letters, research]
thumbnail: assets/img/blog/grad-school-application.jpeg
related_posts: false
---
[日本語版]({% post_url 2021-05-07-grad-school-application-ja %})

*Translated from the Japanese original published May 7, 2021.*

{% include figure.html path="assets/img/blog/grad-school-application.jpeg" class="img-fluid rounded z-depth-1" alt="MIT's Great Dome" %}

<translated body>
```

`_posts/2021-09-07-funai-scholarship-application-en.md`:
```markdown
---
layout: post
title: Publishing My Funai Foundation Scholarship Application!!
date: 2021-09-07 12:01:00
description: I am publishing my application from last year to the Funai Foundation for Information Technology (hereafter, the Funai Foundation), which supports my graduate study abroad!!
categories: [English]
tags: [scholarships]
thumbnail: assets/img/blog/funai-scholarship-application.png
related_posts: false
---
[日本語版]({% post_url 2021-09-07-funai-scholarship-application-ja %})

*Translated from the Japanese original published September 7, 2021.*

{% include figure.html path="assets/img/blog/funai-scholarship-application.png" class="img-fluid rounded z-depth-1" alt="Banner of the Funai Foundation for Information Technology website" %}

<translated body>
```

`_posts/2021-11-17-mit-sop-en.md`:
```markdown
---
layout: post
title: Publishing the SoP I Submitted to MIT AeroAstro for Graduate Study Abroad!!
date: 2021-11-17 12:01:00
description: I am publishing the essay (Statement of Purpose, SoP) required for applying to graduate schools abroad!
categories: [English]
tags: [SoP]
thumbnail: assets/img/blog/mit-sop.jpeg
related_posts: false
---
[日本語版]({% post_url 2021-11-17-mit-sop-ja %})

*Translated from the Japanese original published November 17, 2021.*

{% include figure.html path="assets/img/blog/mit-sop.jpeg" class="img-fluid rounded z-depth-1" alt="MIT's Great Dome" %}

<translated body>
```

(`<translated body>` marks where the translation goes. It is not literal text.)

**Translation rules** (from the spec; the author chose "faithful + brief notes"):
1. **Structure.** Translate the Japanese body block by block, keeping the same headings at the same levels and the same
   paragraphs. Keep every `<br>` at the same position, the same list items, table rows and cells, and bold spans. Keep
   the same links, in order and with identical URLs, including Japanese-language pages and the `📄` PDF links (whose
   text stays the file name). Write one paragraph per line.
2. **Fidelity.** Sentence by sentence, in the author's first-person voice. Keep every fact, number, date, score and hedge
   (「と思います」 → "I think", 「ざっくり」 → "roughly"). Keep the tone markers: 「！！」 may stay "!!", and 「本当に本当に」 is
   "really, truly". Add nothing and drop nothing.
3. **Glosses.** On a Japanese-only proper noun's first use in a post, add a short gloss in parentheses, and don't repeat
   it in the same post. When you are not sure of an organization's official English name, describe it and keep the
   Japanese name in parentheses rather than guessing.
4. **Glossary** (use consistently in all three posts):

| Japanese | English |
| --- | --- |
| 船井情報科学振興財団 / 船井財団 | the Funai Foundation for Information Technology (first use) / the Funai Foundation |
| Funai Overseas Scholarship / 船井財団奨学金 | Funai Overseas Scholarship / the Funai Foundation scholarship |
| XPLANE | XPLANE (a Japanese nonprofit that supports students applying to graduate school abroad) on first use |
| 米国大学院学生会ニュースレター「かけはし」 | *Kakehashi*, the newsletter of the Japanese graduate student association in the US (米国大学院学生会) |
| 九州大学 / ミシガン大学 | Kyushu University / the University of Michigan |
| 推薦状 / 研究実績 / 面接 / 出願 | letters of recommendation / research record / interviews / application(s) |
| 合格 / 不合格 / 足切り | admitted (admission) / rejected / cutoff |
| 王道 | the standard, tried-and-true approach |
| 大樹さん（XPLANE代表/HondaJet） | Daiju (head of XPLANE / HondaJet) |
| 中村さん / 飯山さん / 斎藤さん | Nakamura / Iiyama / Saito |
| Amazon.co.jp book titles | keep the Japanese title as link text, followed by an English gloss outside the link: `[TOEFLテスト英単語3800](url) (TOEFL Test Vocabulary 3800)` |
| ２０２１年６月１日（火） / ３，０００米ドル／月 / ５０万円 | Tuesday, June 1, 2021 / US$3,000 per month / ¥500,000 |

5. **Heading translations** (retrospective): はじめに → Introduction; 略歴 → Background; 海外大学院を目指したきっかけ → Why I Aimed
   for Graduate School Abroad; 出願結果 → Results; 出願準備 → Preparing the Application; 推薦状 → Letters of Recommendation;
   教授とのコネクション → Connections with Professors; 研究実績 → Research Record; 奨学金 → Scholarships; 受験直前の対策 →
   Right Before the Test; 面接 → Interviews; 進学先選び → Choosing Where to Go; まとめ → Wrapping Up; 参考になるリンク集 → Useful
   Links. English headings (Statement of Purpose (SoP), GPA, TOEFL, GRE, Reading, Listening, Speaking, Writing,
   Verbal, Quantitative) stay as they are. Results table: header `| University | Result | Notes |`, with 合格 →
   admitted, 不合格 → rejected, and `Waitlist -> Master 合格` → `Waitlist -> admitted to Master's`.
6. **Funai 応募要項** → `## Application Guidelines`. Keep the bold numbered titles: 1. Purpose, 2. Eligibility,
   3. Number of Awards, 4. Support Period, 5. Support Provided, 6. Required Documents, 7. Application Period,
   8. Selection Process, 9. Award Decision, 10. Obligations of Scholars, 11. Other, 12. Where to Send the Application,
   13. Inquiries. Translate each body fully, keeping its line breaks. Write the postal address in English order
   (4-11-5 Sotokanda, Chiyoda-ku, Tokyo 101-0021).
7. **Judgment calls.** As you translate, append to `$WORK/judgment_calls.md` one line per judgment call: every gloss
   added, every idiom rendered by meaning rather than literally, and every place where the Japanese was ambiguous. Use
   the format `- <post slug>: 「<Japanese>」 → "<English>" — <why>`.

- [ ] **Step 4: Run the checks and fix until they pass**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/parity_check.py $REPO/_posts; echo "exit=$?"; $PY $WORK/check_posts.py $REPO all; echo "exit=$?"; $PY $WORK/text_check.py $SRC/posts_api.json $REPO/_posts; echo "exit=$?"
```

Expected: three `OK … same structure as …` lines, six `OK` post lines, three `OK … chars identical` lines, and
`exit=0` three times. A parity `FAIL` names the differing feature with both sides' values. Fix the English to match,
never the Japanese.

- [ ] **Step 5: Re-read each English post once against its Japanese twin**, paragraph by paragraph, looking for any
  dropped clause, number or hedge. The parity check cannot see those. Fix them, then re-run Step 4.

- [ ] **Step 6: Commit**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO add _posts && commit "Add English translations of the three blog posts" && git -C $REPO log --oneline -1
```

---

### Task 4: PR preview pipeline and the first ("red") CI build

**Files:**
- Modify: `.github/workflows/deploy.yml` (add one step between "Install and Build 🔧" and "Deploy 🚀")
- Scratch: `$WORK/check_site.py`, `$WORK/pr_body.md`, `$SITE/` (downloaded artifact)

**Interfaces:**
- Consumes: the six posts (Tasks 2–3) and `$WORK/judgment_calls.md`.
- Produces:
  - a PR from `blog-migration` into `master`;
  - a workflow that uploads `_site` as artifact `site` on `pull_request` runs;
  - `check_site.py <site dir>`, which prints `OK`/`FAIL`/`INFO` lines and exits 1 on any FAIL (Task 5 and Task 6 rerun
    it);
  - the shell recipe in Step 9, which fetches the latest build into `$SITE`.

- [ ] **Step 1: Write the built-site checker** at `$WORK/check_site.py`

```python
"""Check the CI-built _site for the blog migration. Prints OK/FAIL/INFO lines; exit 1 on any FAIL.

Usage: check_site.py <directory holding the downloaded _site>
"""
import re
import sys
from pathlib import Path
from urllib.parse import unquote, urlsplit

from bs4 import BeautifulSoup

SLUGS = {"grad-school-application": "May 7, 2021",
         "funai-scholarship-application": "September 7, 2021",
         "mit-sop": "November 17, 2021"}
CROSS = {"ja": ("English version", "en"), "en": ("日本語版", "ja")}
results = []


def check(ok, label):
    results.append((bool(ok), label))


def page(site, rel):
    p = site / rel
    return BeautifulSoup(p.read_text(encoding="utf-8"), "html.parser") if p.is_file() else None


def resolves(site, href):
    """True if a root-relative href points at a file of the built site."""
    target = site / unquote(urlsplit(href).path).lstrip("/")
    return target.is_file() or (target / "index.html").is_file()


def local_links(soup):
    for tag, attr in (("a", "href"), ("img", "src")):
        for el in soup.find_all(tag):
            v = el.get(attr, "")
            if v.startswith("/") and not v.startswith("//"):
                yield v


def main(site_dir):
    site = Path(site_dir)
    home = page(site, "index.html")
    nav = [a.get_text(" ", strip=True).replace("(current)", "").strip() for a in home.select("#navbarNav a.nav-link")]
    check(nav == ["about", "publications", "projects", "cv", "blog"], f"nav order is {nav}")
    check("/blog/2021/" not in str(home), "homepage does not list blog posts")

    idx = page(site, "blog/index.html")
    hrefs = [a["href"] for a in idx.select("ul.post-list a.post-title")]
    want = [f"/blog/2021/{s}-{lang}/" for s in SLUGS for lang in ("en", "ja")]
    check(sorted(hrefs) == sorted(want), f"blog index lists exactly the six posts (got {hrefs})")
    check(not (site / "blog/page/2").exists(), "blog index fits on one page")
    cats = [a.get_text(strip=True) for a in idx.select(".tag-category-list a")]
    check(cats == ["日本語", "English"], f"blog filter links are {cats}")
    check("a simple whitespace theme" not in str(idx), "al-folio demo description is gone")
    retro = idx.find("a", href="/blog/2021/grad-school-application-ja/")
    m = retro and re.search(r"(\d+) min read", retro.find_parent("li").get_text(" "))
    check(m and int(m.group(1)) >= 10, f"Japanese retrospective read time >= 10 min (got {m and m.group(1)})")
    years = sorted(p.name for p in (site / "blog").iterdir() if re.fullmatch(r"\d{4}", p.name))
    check(years == ["2021"], f"only 2021 has posts (got {years})")

    for name in ("日本語", "english"):
        cat = page(site, f"blog/category/{name}/index.html")
        check(cat is not None and len(cat.select("a.post-link")) == 3, f"category page {name} lists 3 posts")
    for a in idx.select(".tag-category-list a"):
        check(resolves(site, a["href"]), f"filter link {a['href']} resolves")

    for slug, date in SLUGS.items():
        for lang, (cross_text, other) in CROSS.items():
            rel = f"blog/2021/{slug}-{lang}/index.html"
            soup = page(site, rel)
            if soup is None:
                check(False, f"{rel} exists"); continue
            check(soup.html.get("lang") == lang, f"{rel} has <html lang=\"{lang}\"> (got {soup.html.get('lang')})")
            meta = soup.select_one(".post-meta")
            check(meta is not None and date in meta.get_text(), f"{rel} shows date {date}")
            a = soup.find("a", string=cross_text)
            check(a is not None and a.get("href") == f"/blog/2021/{slug}-{other}/", f"{rel} cross-links to its twin")
            bad = sorted({h for h in local_links(soup) if not resolves(site, h)})
            check(not bad, f"{rel} local links resolve {bad if bad else ''}")
            check("kotakondo.com" not in str(soup), f"{rel} has no kotakondo.com link")

    check(not (site / "docs").exists(), "docs/ (spec and plan) is not published")
    for f in ("CLAUDE.md", "README.md", "CONTRIBUTING.md"):
        if (site / f).exists():
            print(f"INFO {f} is published at /{f} (pre-existing, outside this change)")
    for ok, label in results:
        print(("OK   " if ok else "FAIL ") + label)
    return 0 if all(ok for ok, _ in results) else 1


if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

- [ ] **Step 2: Add the artifact upload step** to `.github/workflows/deploy.yml` (Edit tool)

Old:
```yaml
        purgecss -c purgecss.config.js
    - name: Deploy 🚀
```
New:
```yaml
        purgecss -c purgecss.config.js
    - name: Upload site for PR preview 📦
      if: github.event_name == 'pull_request'
      uses: actions/upload-artifact@v4
      with:
        name: site
        path: _site
        retention-days: 3
    - name: Deploy 🚀
```

- [ ] **Step 3: Commit**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO add .github/workflows/deploy.yml && commit "Upload the built site as an artifact on pull requests" && git -C $REPO log --oneline -1
```

- [ ] **Step 4: Author action: log in to GitHub CLI.** Ask the author to run `! gh auth login` (GitHub.com, browser
  login). Then verify:

```bash
gh auth status 2>&1 | head -3
```

Expected: `Logged in to github.com account kotakondo`. Stop here until it is.

- [ ] **Step 5: Write the PR body** at `$WORK/pr_body.md`

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
{
  echo "Moves the three Japanese blog posts from kotakondo.com (WordPress, being shut down) into a new **blog** tab, each with a faithful English translation. Spec: \`docs/superpowers/specs/2026-10-13-blog-migration-design.md\`."
  echo
  echo "- \`_posts/\`: 3 Japanese originals (text verified identical to WordPress) + 3 English translations (structure verified identical to the Japanese)"
  echo "- \`assets/img/blog/\`, \`assets/pdf/blog/\`: cover images and the two PDFs"
  echo "- \`.github/workflows/deploy.yml\`: uploads \`_site\` as a 3-day artifact on PRs, for previews"
  echo
  echo "## Translation judgment calls (please review)"
  echo
  cat $WORK/judgment_calls.md
  echo
  echo "🤖 Generated with [Claude Code](https://claude.com/claude-code)"
} > $WORK/pr_body.md && wc -l $WORK/pr_body.md
```

- [ ] **Step 6: Author approval, then push and open a draft PR.** Ask the author: "Push `blog-migration` to GitHub and
  open a draft PR? The repo is public." Only after they say yes:

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO push -u origin blog-migration
gh pr create --repo $GH_REPO --base master --head blog-migration --draft \
  --title "Add a blog with the posts migrated from kotakondo.com (日本語 + English)" --body-file $WORK/pr_body.md
```

Expected: a PR URL.

- [ ] **Step 7: Wait for the build**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
sleep 20; RUN=$(gh run list --repo $GH_REPO --branch blog-migration --event pull_request --limit 1 --json databaseId -q '.[0].databaseId')
echo "run $RUN"; gh run watch $RUN --repo $GH_REPO --exit-status > /dev/null; echo "exit=$?"
```

(Use a 600000 ms Bash timeout.) Expected: `exit=0`. If the build fails, run `gh run view $RUN --repo $GH_REPO
--log-failed | tail -60`. A Liquid or `post_url` error names the post file. Fix it, re-run the Task 2/3 checks,
commit, push, and repeat this step.

- [ ] **Step 8: Download the built site**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
RUN=$(gh run list --repo $GH_REPO --branch blog-migration --event pull_request --limit 1 --json databaseId -q '.[0].databaseId')
rm -rf $SITE && gh run download $RUN --repo $GH_REPO -n site -D $SITE && ls $SITE | head
```

Expected: `404.html`, `assets`, `blog`, `index.html`, … (the contents of `_site`).

- [ ] **Step 9: Run the site check and confirm it is red for exactly the planned reasons**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/check_site.py $SITE; echo "exit=$?"
```

Expected: `exit=1`. Every `FAIL` must be one that Task 5 fixes:
- nav order without `blog`;
- the blog index containing the al-folio demo post, or not fitting on one page;
- the filter links being the demo tags;
- the demo description;
- the read time;
- `lang="en"` on the three `-ja` pages;
- extra years;
- `docs/` being published.

Every post-page `exists`, date, cross-link, local-link and `kotakondo.com` line must already be `OK`. If not, fix the
post in Tasks 2–3 terms before continuing. Save the output to `$WORK/site_check_red.txt`.

---

### Task 5: Turn on the blog (config, templates, CLAUDE.md) and the "green" build

**Files:**
- Modify: `_config.yml` (Blog block ~lines 123–125, `external_sources` ~157–159, `exclude` ~204–208,
  `display_tags`/`display_categories` ~266–267)
- Modify: `_includes/header.html` (move the Blog `<li>` block, lines 56–65, to after the page loop)
- Modify: `_layouts/default.html:2`
- Modify: `blog/index.html` (`per_page` line 8; read-time lines 74 and 101)
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: `check_site.py` and the fetch recipe from Task 4.
- Produces: the finished site configuration. Task 6 only verifies visually.

- [ ] **Step 1: `_config.yml`** (Edit tool, four edits)

Old → new 1:
```yaml
blog_nav_title: # your blog must have a title for it to be displayed in the nav bar
blog_description: a simple whitespace theme for academics
```
```yaml
blog_nav_title: blog # your blog must have a title for it to be displayed in the nav bar
blog_description: Notes on studying abroad and grad school applications / 海外大学院留学の記録
```
Old → new 2:
```yaml
external_sources:
  - name: medium.com
    rss_url: https://medium.com/@al-folio/feed
```
```yaml
external_sources: []
```
Old → new 3:
```yaml
exclude:
  - bin
```
```yaml
exclude:
  - bin
  - docs
```
Old → new 4:
```yaml
display_tags: ['formatting', 'images', 'links', 'math', 'code'] # these tags will be displayed on the front page of your blog
display_categories: ['blockquotes'] # these categories will be displayed on the front page of your blog
```
```yaml
display_tags: [] # these tags will be displayed on the front page of your blog
display_categories: ['日本語', 'English'] # language filter on the front page of your blog
```

- [ ] **Step 2: `_includes/header.html`: move the Blog tab to the end.** Delete this block, which comes right after the
  About `</li>` (keep the blank line and `<!-- Other pages -->` that follow it):

```liquid
              {% if site.blog_nav_title %}
              <!-- Blog -->
              <li class="nav-item {% if page.url contains 'blog' %}active{% endif %}">
                <a class="nav-link" href="{{ '/blog/' | relative_url }}">{{ site.blog_nav_title }}
                  {%- if page.url contains 'blog' -%}
                  <span class="sr-only">(current)</span>
                  {%- endif -%}
                </a>
              </li>
              {%- endif %}
```

Then insert the same block between the end of the page loop and the dark-mode toggle:

Old:
```liquid
              {% endfor -%}
              {%- if site.enable_darkmode %}
```
New:
```liquid
              {% endfor -%}
              {% if site.blog_nav_title %}
              <!-- Blog -->
              <li class="nav-item {% if page.url contains 'blog' %}active{% endif %}">
                <a class="nav-link" href="{{ '/blog/' | relative_url }}">{{ site.blog_nav_title }}
                  {%- if page.url contains 'blog' -%}
                  <span class="sr-only">(current)</span>
                  {%- endif -%}
                </a>
              </li>
              {%- endif %}
              {%- if site.enable_darkmode %}
```

Verify: `grep -c 'site.blog_nav_title %}' $REPO/_includes/header.html` prints `1`, and the `<!-- Blog -->` line number is
greater than that of `{%- assign sorted_pages`.

- [ ] **Step 3: `_layouts/default.html` line 2**

Old: `<html lang="{{ site.lang }}">`
New: `<html lang="{{ page.lang | default: site.lang }}">`

- [ ] **Step 4: `blog/index.html`.** In the front matter, `per_page: 5` → `per_page: 10`. Then replace the two
  word-count read-time lines.

Old (featured card, line 74):
```liquid
                      {% assign read_time = post.content | number_of_words | divided_by: 180 | plus: 1 %}
```
New:
```liquid
                      {% if post.lang == 'ja' %}
                        {% assign read_time = post.content | strip_html | strip_newlines | size | divided_by: 500 | plus: 1 %}
                      {% else %}
                        {% assign read_time = post.content | number_of_words | divided_by: 180 | plus: 1 %}
                      {% endif %}
```
Old (post list, line 101):
```liquid
      {% assign read_time = post.content | number_of_words | divided_by: 180 | plus: 1 %}
```
New:
```liquid
      {% if post.lang == 'ja' %}
        {% assign read_time = post.content | strip_html | strip_newlines | size | divided_by: 500 | plus: 1 %}
      {% else %}
        {% assign read_time = post.content | number_of_words | divided_by: 180 | plus: 1 %}
      {% endif %}
```
(The two `post.feed_content` lines, which handle external posts, stay unchanged.)

- [ ] **Step 5: `CLAUDE.md`** (Edit tool)

1. In "## Commands", replace the sentence `Ruby is present on this machine but \`bundle\`/\`jekyll\` are **not installed**, so Docker is the only local path:` with:

   ```markdown
   As of 2026-10-13 neither Ruby nor Docker is installed on this machine. To check a change without deploying, open a
   PR: `deploy.yml` builds it and uploads `_site` as the `site` artifact (kept 3 days). Fetch it with
   `gh run download <run-id> -n site -D <dir>` and serve it with `python3 -m http.server`. With Docker installed:
   ```
2. In the content-model table, add the row `| Blog posts (日本語 + English) | \`_posts/YYYY-MM-DD-<slug>-ja.md\` + \`-en.md\` |`
   after the News row.
3. In the **Navigation** paragraph, replace `The blog is effectively disabled — \`blog_nav_title\` is blank and there is no \`_posts/\` directory — though \`blog/index.html\` and the archive layouts remain.` with
   `The **blog** tab is not a page: \`blog_nav_title\` in \`_config.yml\` turns it on, and \`_includes/header.html\` renders it after the page loop (last).`
4. Add this paragraph after the **News** paragraph:

   ```markdown
   **Blog** (`_posts/`): each article exists twice. `<slug>-ja.md` is the original (`lang: ja`,
   `categories: [日本語]`) and `<slug>-en.md` the translation (`categories: [English]`, no `lang`). They are
   cross-linked on their first line with `{% post_url %}`, so a wrong slug fails the build. English posts are dated one
   minute after their Japanese twin, with no UTC offset, so they list first and the shown day never shifts. kramdown
   `hard_wrap` is off, so an in-paragraph line break needs an explicit `<br>`. `lang: ja` sets `<html lang="ja">`
   (Japanese glyphs for kanji) and makes the blog index estimate read time from characters (÷500) instead of words.
   Cover images go in `assets/img/blog/` (also set as `thumbnail:`) and PDFs in `assets/pdf/blog/`. The posts came
   from kotakondo.com (WordPress), which is being shut down: never link to it. The design is in
   `docs/superpowers/specs/` (`docs` is in `exclude:`, so it is not published).
   ```
5. In "## Gotchas", replace the `_plugins/external-posts.rb` bullet with:
   `- \`_plugins/external-posts.rb\` fetches every \`external_sources\` RSS feed on each build and injects its entries as posts. It is now \`[]\`; it used to point at al-folio's demo Medium feed, which put a demo post on \`/blog/\`.`
6. In "## Local deviations from upstream al-folio", append these bullets:
   ```markdown
   - `_includes/header.html` — the blog tab is rendered after the page loop (last in the nav), not after About.
   - `_layouts/default.html` — `<html lang>` uses `page.lang` when set (Japanese posts).
   - `blog/index.html` — `per_page: 10`; read time for `lang: ja` posts is characters / 500 + 1.
   - `.github/workflows/deploy.yml` — uploads `_site` as artifact `site` on pull requests.
   ```

- [ ] **Step 6: Commit and push**

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
git -C $REPO add _config.yml _includes/header.html _layouts/default.html blog/index.html CLAUDE.md
commit "Turn on the blog tab: last in the nav, language filter, Japanese lang and read time" && git -C $REPO log --oneline -1
git -C $REPO push
```

(The author already approved pushes to this PR branch in Task 4. If they asked to approve each push, ask again here.)

- [ ] **Step 7: Wait, download and re-run the site check.** Repeat Task 4 Steps 7–8 verbatim, then:

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/check_site.py $SITE | tee $WORK/site_check_green.txt; echo "exit=${PIPESTATUS[0]}"
```

Expected: no `FAIL` lines, `exit=0`. `INFO` lines about `CLAUDE.md`/`README.md`/`CONTRIBUTING.md` are reported to the
author in Task 6, not fixed. If a line fails, fix the file it names, commit, push and repeat this step.

---

### Task 6: Visual check in Chrome, link report, hand-off

**Files:**
- Possibly modify: `_posts/2021-05-07-grad-school-application-ja.md` and `-en.md` (only if the phone-width check
  fails; see Step 4)
- Scratch: `$WORK/link_check.py`, `$WORK/link_report.md`, screenshots

**Interfaces:**
- Consumes: `$SITE` (green build), `check_site.py`, `pr_body.md`, `judgment_calls.md`.
- Produces: a PR whose description includes the link report, ready for the author's review.

- [ ] **Step 1: Serve the built site locally** (Bash with `run_in_background: true`)

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh && python3 -I -m http.server 8765 --bind 127.0.0.1 --directory $SITE
```

- [ ] **Step 2: Load the Chrome tools in one call.** Use ToolSearch with
  `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__resize_window`.
  Then call `tabs_context_mcp` and create a new tab. If the extension is not connected, stop and ask the author: either
  connect it, or review the artifact themselves by running the Step 1 command and opening http://127.0.0.1:8765/.

- [ ] **Step 3: Desktop pass (1280×900).** Screenshot each page and confirm:
  - `http://127.0.0.1:8765/`: the nav reads about · publications · projects · cv · blog, and the homepage is otherwise as
    before.
  - `/blog/`: the bilingual description, the `日本語 · English` filter, six entries with thumbnails, and English above
    Japanese within each pair.
  - `/blog/category/%E6%97%A5%E6%9C%AC%E8%AA%9E/`: the three Japanese posts.
  - `/blog/2021/grad-school-application-ja/` and `-en/`:
    - the cross-link, the cover image and the results table;
    - the nine former tabs as `###` sections;
    - the blockquote;
    - JS `document.documentElement.lang` is `"ja"` / `"en"` respectively.
  - `/blog/2021/funai-scholarship-application-ja/`: the PDF link opens the PDF, and 応募要項 reads as bold numbered
    titles with line-broken bodies.
  - `/blog/2021/mit-sop-en/`: the PDF link opens the PDF.

- [ ] **Step 4: Phone pass (390×844, `resize_window`).** On `/blog/`, `/blog/2021/grad-school-application-ja/` and
  `-en/`, run JS:

```javascript
[document.documentElement.scrollWidth, window.innerWidth, document.documentElement.scrollWidth <= window.innerWidth]
```

Expected: the third value is `true` on all three pages. If it is `false` on a post page, the results table is too wide.
Wrap the table in **both** retrospective files like this (blank lines are required):

```markdown
<div class="table-responsive" markdown="1">

| 大学 | 合否結果 | 備考 |
…(table rows unchanged)…

</div>
```

Then re-run `text_check.py`, `parity_check.py` and `check_posts.py all` (all must stay `OK`), commit
"Let the results table scroll horizontally on phones", push, and repeat Task 4 Steps 7–8 and this step.

- [ ] **Step 5: Dark mode.** On `/blog/2021/grad-school-application-ja/` at desktop width, run JS
  `document.getElementById('light-toggle').click()`. Screenshot the results table and the blockquote, and confirm the
  text is readable. Click again to restore.

- [ ] **Step 6: Link report.** Write `$WORK/link_check.py`:

```python
"""Report (never edit) external links in the posts that do not answer 2xx/3xx.

Usage: link_check.py <repo>/_posts
"""
import re
import sys
import urllib.error
import urllib.request
from pathlib import Path

LINK = re.compile(r"\]\((https?://[^)\s]+)\)")


def status(url: str) -> str:
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
    try:
        with urllib.request.urlopen(req, timeout=20) as r:
            return str(r.status)
    except urllib.error.HTTPError as e:
        return str(e.code)
    except Exception as e:  # DNS failure, timeout, TLS error
        return type(e).__name__


def main(posts_dir: str) -> int:
    urls = sorted({u for p in Path(posts_dir).glob("2021-*-ja.md") for u in LINK.findall(p.read_text(encoding="utf-8"))})
    bad = [(status(u), u) for u in urls]
    bad = [(s, u) for s, u in bad if not s.startswith(("2", "3"))]
    print(f"Checked {len(urls)} unique external links; {len(bad)} did not answer 2xx/3xx (left unchanged in the posts).\n")
    if bad:
        print("| status | url |\n| --- | --- |")
        for s, u in bad:
            print(f"| {s} | {u} |")
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
$PY $WORK/link_check.py $REPO/_posts | tee $WORK/link_report.md
```

Expected: about 40 links checked. Amazon.co.jp usually answers 503 to scripts. Note it as "blocks scripted checks" and
don't count it as dead.

- [ ] **Step 7: Update the PR description** (author approval for `gh pr edit`, as in Task 4)

```bash
source /tmp/claude-1003/-home-kkondo-papers/0d541608-33cc-4dd0-8b23-17934c8fd2ee/scratchpad/blog_migration/env.sh
python3 -I - "$WORK" <<'EOF'
import sys
from pathlib import Path
w = Path(sys.argv[1])
body = (w / "pr_body.md").read_text(encoding="utf-8")
sig = "🤖 Generated with [Claude Code](https://claude.com/claude-code)"
extra = "## External links\n\n" + (w / "link_report.md").read_text(encoding="utf-8") + "\n"
(w / "pr_body.md").write_text(body.replace(sig, extra + sig), encoding="utf-8")
EOF
gh pr edit blog-migration --repo $GH_REPO --body-file $WORK/pr_body.md
```

- [ ] **Step 8: Stop the local server** (kill the background `http.server`), close the Chrome tab, and hand off to the
  author:
  - the PR URL and the commit list (`git -C $REPO log --oneline master..blog-migration`);
  - the green `check_site.py` summary, the screenshots, and the phone-width and dark-mode results;
  - the judgment-call list (ask them to read the English posts);
  - the link report;
  - the `INFO` lines about `CLAUDE.md`/`README.md`/`CONTRIBUTING.md` being published, as a separate optional fix;
  - the next steps, which are theirs: mark the PR ready and merge (in the GitHub UI, or tell Claude to run
    `gh pr ready` + `gh pr merge --merge`). Merging deploys.

- [ ] **Step 9: After the author merges: verify live.**

```bash
sleep 240; curl -sS https://kotakondo.github.io/blog/ | grep -o 'href="/blog/2021/[^"]*"' | sort -u
```

Expected: the six post URLs. Tell the author that kotakondo.com can now be cancelled, and that links to the old
WordPress URLs will stop working when it goes away.
