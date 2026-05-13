---
name: ghost-blog-writer
description: "Research, draft, scrub, AI-SEO-audit, and publish a blog post to any Ghost CMS site via the Admin API. Takes a topic (typically a long-tail query like 'how to fix X', 'how to connect X to Y', 'X vs Y', 'what is X') and produces a complete post: structured for SEO and AI-citation (H2 headings matching query phrasing, TL;DR block, FAQ section, JSON-LD schema), scrubbed of LLM-tell typography, and posted via the Ghost Admin API. Trigger when the user says: 'write a blog post on X', 'publish a Ghost article on X', 'draft a blog entry about X', or any request to create editorial content for a Ghost blog."
version: 1.0.0
emoji: "✍️"
homepage: https://github.com/ratamaha-git/publishing-skills
metadata:
  openclaw:
    requires:
      bins:
        - python3
---

# ghost-blog-writer

End-to-end pipeline for shipping a single long-tail blog post to a Ghost CMS site: **topic -> research -> draft -> scrub -> AI-SEO audit -> publish**. Designed for SEO and AI-citation extractability (FAQ blocks, BreadcrumbList + FAQPage + HowTo schema, query-phrased headings).

The skill takes **one required argument**: the topic. Optional flags control publish state.

```
/ghost-blog-writer <topic>
/ghost-blog-writer <topic> --publish                    # publish live (default: save as draft)
/ghost-blog-writer <topic> --publish-at <ISO-UTC>       # schedule for future publish
/ghost-blog-writer <topic> --angle "<angle>"            # narrow the angle
```

Default state is **draft** — the post lands in Ghost admin for human review before going live, unless `--publish` or `--publish-at` is passed. `--publish-at` accepts an ISO 8601 UTC timestamp (e.g. `2026-05-10T07:42:00Z`) and is mutually exclusive with `--publish`.

---

## Before you start — preflight

This skill targets any Ghost CMS instance (self-hosted or Ghost(Pro)). It uses the Admin API over HTTPS. No Docker, no SSH — just authenticated POST to `/ghost/api/admin/posts/`.

Run these three checks before Step 0 and stop if any fails:

```bash
# 1. Target URL is up — replace <ghost-url> with your site
curl -sS https://<your-ghost-site>/ghost/api/admin/site/ | head -c 80
# Expected: {"site":{"title":"...", ...

# 2. The two required env vars are set (see "Credentials" below)
[ -n "$GHOST_URL" ] && [ -n "$GHOST_ADMIN_KEY" ] && echo "keys present" || echo "MISSING"
```

### Credentials

Two values are required. Get them from **Ghost admin -> Settings -> Integrations -> (your integration)**:

| Env var | Source | Shape |
|---|---|---|
| `GHOST_URL` | Your Ghost site URL | `https://blog.example.com` (no trailing slash) |
| `GHOST_ADMIN_KEY` | Integration -> **Admin API Key** | `<24-hex>:<64-hex>` combined |

The Admin Key is one string of the form `<id>:<secret>`, separated by a colon. Step 8 splits it to build the JWT.

Set them in your shell before invoking the skill:

```bash
export GHOST_URL="https://blog.example.com"
export GHOST_ADMIN_KEY="64a..."   # full <id>:<secret> from Ghost admin
```

**Never commit these values.** Keep them in a `.env` file (gitignored) or your shell profile.

---

## Step 0 — Parse and classify the topic

The topic is the one thing the skill cannot invent. It must arrive as an argument.

| Shape | Example | Treatment |
|---|---|---|
| **Long-tail how-to** | `"how to fix n8n HTTP Request 401 error"` | Ideal. Format = troubleshooting (template 1). |
| **Integration walk-through** | `"how to connect Airtable to Slack with Zapier"` | Format = integration (template 2). |
| **Workflow tutorial** | `"automate invoice processing with Make"` | Format = workflow tutorial (template 3). |
| **Comparison** | `"Zapier vs Make vs n8n"` | Format = comparison (template 4). |
| **Definition / explainer** | `"what is an AI agent"` | Format = explainer (template 5). |
| **Use case / outcome** | `"build a daily Slack digest from RSS with n8n"` | Format = use-case (template 6). |
| **Listicle / roundup** | `"12 best n8n templates for marketing teams"` | Format = listicle (template 7). |
| **Migration guide** | `"migrate from Zapier to n8n"` | Format = migration (template 8). |
| **Release recap** | `"what's new in n8n 1.80"` | Format = release-recap (template 9). |
| **Too vague** | `"AI"`, `"automation"` | **Stop.** Ask the user to narrow it. Suggest 2-3 candidate long-tail variants. |

If `--angle` was passed, append it to the topic. The classification picks the structural template used in Step 3.

---

## Step 1 — Research

The piece must be specific. Real version numbers, real error messages, real screenshots — not generic "best practices."

### 1a. Identify the search intent

What does someone typing this query want? One sentence — the implicit desire behind the words.

- `"how to fix n8n HTTP 401"` -> wants the exact change to make in the UI to stop the error
- `"Zapier vs Make"` -> wants a quick decision, then a longer breakdown
- `"what is an AI agent"` -> wants a one-paragraph explanation, then how it differs from a workflow

If you can't write one sentence describing the intent, the topic is too vague — go back to Step 0.

### 1b. Seed search and SERP teardown

```
WebSearch("<topic>")
WebSearch("<topic> <current-year>")  # force a fresh lens
```

Extract three structured signals from the page-1 results:

1. **Word count distribution** — eyeball the top 5 results' length. Target 1.1–1.3x the median, not the longest. If the median is 600 words, don't write 1500 — that's padding.
2. **People Also Ask boxes** — Google surfaces 4-8 PAA questions for most queries. These are free FAQ content. Capture verbatim into the FAQ-variant list.
3. **Currently-winning featured snippet** — if there is one, note its format (paragraph, list, table). Write the lead paragraph in that exact shape; that's how you challenge for the snippet.

Goal: write something **more specific or more current** than the existing top results, not a paraphrase.

### 1c. Deep fetch

Pick **2-4 URLs** from the SERP. Prioritize:

- **Vendor docs** — primary sources for the tool being discussed.
- **GitHub issues / changelogs** — for "fix X error" topics, the actual issue thread is gold.
- **Reddit / community forums** — for confirming a workaround actually works in the wild.
- **Existing top-ranked posts** — to see the bar you're clearing.

```
WebFetch(url, "Return the full article body as clean prose. Include code snippets,
error messages, and screenshot references verbatim. Do NOT summarize.")
```

Skip SEO-farm rewrites and listicles with no specifics.

### 1d. Five-question gate before drafting

Before writing, you must be able to answer all five.

1. **What is the exact query intent?** (one sentence from 1a)
2. **What is the direct answer?** (one to two sentences — the lead paragraph in compressed form)
3. **What's the canonical primary source?** (vendor doc, GitHub issue, official changelog — at least one URL)
4. **What's the gotcha most existing posts miss?** (the specific detail that makes this post worth writing). **Hard rule:** if the honest answer is "nothing, I'm summarizing the docs," **abort and tell the user**. A doc paraphrase will rank below the actual docs.
5. **What 3-6 follow-on questions belong in the FAQ?** (long-tail variations of the main query, ideally lifted from the PAA boxes captured in 1b)

If any answer is `?`, keep researching or ask the user for a specific source.

### 1e. Save research artifacts

```bash
mkdir -p tmp/blog-drafts
# <slug> = kebab-case of the topic, e.g. n8n-http-401-fix
```

Files (gitignored):
- `tmp/blog-drafts/<slug>.research.md` — 5-question answers, source list, key quotes
- `tmp/blog-drafts/<slug>.draft.html` — written in Step 3
- `tmp/blog-drafts/<slug>.codeinjection-head.html` — written in Step 7b
- `tmp/blog-drafts/<slug>.payload.json` — written in Step 7f

---

## Step 2 — Pick the format and length band

Each query type maps to a structural template:

| Format | Length band |
|---|---|
| `how-to-fix` (troubleshooting) | 600-1200 |
| `how-to-connect` (integration) | 1000-1500 |
| `how-to-automate` (workflow) | 1000-1500 |
| `x-vs-y` (comparison) | 1200-1500 |
| `what-is` (explainer) | 600-1200 |
| `use-case` (outcome) | 1000-1500 |
| `listicle` (roundup) | 1500-2500 |
| `migration` | 1200-1800 |
| `release-recap` | 800-1400 |

**Hard length range: 600-1500 words for most formats.** Word count = prose inside `<p>` tags + heading text. Excludes code blocks, table cells, figcaptions.

Use the SERP word-count signal from Step 1b to pick a target inside the band (1.1–1.3x the SERP median). Under the floor means the answer is genuinely too thin — add an FAQ expansion, a "common errors" section, or a "how to verify" section. Over the ceiling means the post is sprawling — cut the weakest section. **Never pad to hit a floor.** Google rewards directness; AI Overviews preferentially extract from concise answers.

---

## Step 3 — Draft the post

Write directly in HTML (Ghost accepts HTML via `?source=html`). Allowed tags:

`<p>`, `<h2>`, `<h3>`, `<a>`, `<strong>`, `<em>`, `<code>`, `<pre>`, `<blockquote>`, `<ul>`, `<ol>`, `<li>`, `<table>`, `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>`, `<figure>`, `<figcaption>`, `<img>`.

No inline styles. No `<div>`, no `<span>`, no `<br>`. No H1 (Ghost emits the post title as H1).

### Link policy — internal vs. outbound, follow vs. nofollow

| Destination | `rel` attribute |
|---|---|
| Your own blog (other posts on `$GHOST_URL`) | none — internal, follow |
| Anything else (vendor docs, GitHub, news, social, all third-party) | `rel="nofollow noopener"` |

Do not use `target="_blank"` — most Ghost themes handle outbound link UX themselves.

### Voice checks while drafting

- **Open with a TL;DR block.** First child of the body is `<p><strong>TL;DR:</strong> ...</p>` — a single sentence, 8-40 words, that answers the query directly with specific nouns (tool name, version, error code, command). LLM citation hook. Asserted in Step 7g.
- **Lead paragraph follows the TL;DR** with one or two sentences of context (when this hits, who it bites, why other guides miss the cause). It is not a re-statement of the answer.
- **H2/H3 phrased like queries.** `## How to fix the "ECONNREFUSED" error in n8n` beats `## Fixing the connection error`.
- **Specific over general.** Real version numbers, real error messages, real prices. No "modern", "powerful", "robust", "seamless."
- **Impersonal voice.** "Here's the fix." Not "we found that" and not "I tried this."
- **Forensic linking.** Every external claim links on the noun phrase that names the source. Every external link has `rel="nofollow noopener"`.
- **End with a `<h2>FAQ</h2>` block** — 3-6 H3 questions, each with a 1-3 sentence answer.
- **Self-check:** *Does the TL;DR stand alone as a quotable answer? Does the lead paragraph add context the TL;DR doesn't have? If either fails, rewrite.*

Save to `tmp/blog-drafts/<slug>.draft.html`.

---

## Step 4 — Scrub LLM tells

Run **before** the AI-SEO audit. The audit may add vocabulary the scrub would then need to remove; do the order this way.

### 4a. Character scrub (automatic)

Replace common LLM-tell characters with ASCII equivalents:

```bash
python3 -c "
import sys, pathlib
p = pathlib.Path(sys.argv[1])
t = p.read_text(encoding='utf-8')
# em-dash/en-dash -> hyphen
t = t.replace('—', '-').replace('–', '-')
# smart quotes -> straight quotes
t = t.replace('“', '\"').replace('”', '\"')
t = t.replace('‘', \"'\").replace('’', \"'\")
# ellipsis -> three dots
t = t.replace('…', '...')
# zero-width / non-breaking space -> regular space or empty
t = t.replace('​', '').replace(' ', ' ')
p.write_text(t, encoding='utf-8')
print('scrubbed', sys.argv[1])
" tmp/blog-drafts/<slug>.draft.html
```

### 4b. Prose-level tells (manual)

Search the draft for these banned phrases and rewrite:

- "delve into", "delving"
- "in today's fast-paced world", "in the ever-evolving"
- "robust", "seamless", "powerful", "cutting-edge"
- "harness the power of"
- "it's worth noting that", "it's important to note"
- "navigate the landscape", "navigating the complexities"
- "unlock the potential of", "unleash"
- "game-changer", "revolutionize"
- "leverage" (as a verb)

Rewrite every hit — do not just delete; the surrounding sentence is usually also lazy.

---

## Step 5 — AI-SEO audit

Run the audit against the draft, checking each pass:

1. **Structure pass** — does the lead answer the query in the first paragraph; do H2s match query phrasing; is each section self-contained.
2. **Authority pass** — at least one cited primary source (vendor doc / GitHub issue / changelog) on a relevant noun phrase.
3. **Freshness pass** — current year referenced where it makes sense; version numbers are current.
4. **Schema readiness** — Ghost emits Article + Person + Organization schema automatically. Step 7b adds FAQPage + BreadcrumbList (always) and HowTo (procedural posts only). Confirm the FAQ block has H3 question + paragraph answer pairs the 7b extractor can parse.
5. **Long-tail coverage** — does the FAQ block capture 3-6 long-tail variants of the main query.
6. **Platform-fact pass** — any claim about a specific shell, OS, language runtime, or tool is a verifiable fact, not a vibe. Verify the load-bearing ones against vendor docs before publish.

Apply recommendations **in place** in the draft, then re-run Step 4a (the audit may have re-introduced smart quotes).

### Non-negotiable invariants

- **Body is within the format's length band** (Step 2). Count via the snippet below.
- **TL;DR is the first `<p>` of the body**, opens with `<strong>TL;DR:</strong>`, 8-40 words, single sentence.
- **Lead paragraph (second `<p>`) answers the query** in 1-2 sentences.
- **At least one primary-source link** with `rel="nofollow noopener"`.
- **FAQ block at the end** with 3-6 H3/p pairs.
- **Every external `<a>` carries `rel="nofollow noopener"`.**
- **Zero U+2014, U+201C, U+201D, U+2018, U+2019, U+2026, U+00A0, U+200B.**

```bash
# Word count (excludes code blocks, table cells, figcaptions)
python3 -c "
import sys, re, pathlib
html = pathlib.Path(sys.argv[1]).read_text(encoding='utf-8')
no_code = re.sub(r'<pre\b[^>]*>.*?</pre>', ' ', html, flags=re.S|re.I)
no_table = re.sub(r'<table\b[^>]*>.*?</table>', ' ', no_code, flags=re.S|re.I)
no_fig = re.sub(r'<figure\b[^>]*>.*?</figure>', ' ', no_table, flags=re.S|re.I)
text = re.sub(r'<[^>]+>', ' ', no_fig)
words = re.findall(r\"[A-Za-z0-9][A-Za-z0-9'-]*\", text)
print(f'{len(words)} words')
" tmp/blog-drafts/<slug>.draft.html
```

```bash
# nofollow coverage on external links — expected: 0 violations
python3 -c "
import re, sys, pathlib
from urllib.parse import urlparse
html = pathlib.Path(sys.argv[1]).read_text(encoding='utf-8')
import os
ghost_host = urlparse(os.environ.get('GHOST_URL', '')).hostname or ''
internal = {ghost_host, f'www.{ghost_host}' if ghost_host else ''}
internal = {h for h in internal if h}
violations = []
for m in re.finditer(r'<a\b([^>]*)>', html, flags=re.I):
    attrs = m.group(1)
    href = re.search(r'href=\"([^\"]+)\"', attrs, flags=re.I)
    if not href: continue
    host = urlparse(href.group(1)).hostname or ''
    if host and host not in internal:
        rel = re.search(r'rel=\"([^\"]+)\"', attrs, flags=re.I)
        rel_val = (rel.group(1) if rel else '').lower()
        if 'nofollow' not in rel_val:
            violations.append(href.group(1))
for v in violations: print('MISSING nofollow:', v)
print(f'{len(violations)} violation(s)')
" tmp/blog-drafts/<slug>.draft.html
```

---

## Step 6 — Illustrate the post (optional)

Figures are not required for every post, but recommended for posts over 800 words. **Rule of thumb:** 1 figure per ~500 body words.

For figure generation (SVG flow diagrams, comparison charts, taxonomy diagrams) see the companion `blog-figure-svg` skill — it generates accessible SVG figures with consistent styling and ships them through Ghost's image upload endpoint.

For screenshots, capture from the live tool (Playwright, real session, etc.), crop to the relevant region, redact tokens or personal data. Save as `tmp/blog-drafts/<slug>-<N>-<short-name>.png`.

### Upload images to Ghost CDN

```bash
python3 - <<'PY'
import os, sys, pathlib, datetime, requests, jwt

GHOST_URL = os.environ['GHOST_URL'].rstrip('/')
key = os.environ['GHOST_ADMIN_KEY']
kid, secret = key.split(':', 1)

iat = int(datetime.datetime.now(datetime.timezone.utc).timestamp())
token = jwt.encode(
    {'iat': iat, 'exp': iat + 5 * 60, 'aud': '/admin/'},
    bytes.fromhex(secret),
    algorithm='HS256',
    headers={'kid': kid, 'alg': 'HS256', 'typ': 'JWT'},
)

img_path = pathlib.Path(sys.argv[1])
with img_path.open('rb') as f:
    r = requests.post(
        f"{GHOST_URL}/ghost/api/admin/images/upload/",
        headers={'Authorization': f'Ghost {token}'},
        files={'file': (img_path.name, f, 'image/png')},
        data={'purpose': 'image'},
    )
r.raise_for_status()
print(r.json()['images'][0]['url'])
PY
# Pass the image path as the only argument
```

### Splice figure tags into the draft

```html
<figure>
  <img src="<uploaded-png-url>" alt="<full description with all numbers and labels>" loading="lazy">
  <figcaption>One sentence restating the takeaway in plain English (15-30 words).</figcaption>
</figure>
```

**Caption rules:**
- Required on every figure. Step 7g asserts this.
- 15-30 words, restating the takeaway (not "Figure showing X" — say what the reader should conclude).
- Allowed tags inside `<figcaption>`: `<a>` (with `rel="nofollow noopener"` for external), `<em>`.

---

## Step 7 — Build the Ghost post payload

### 7a. Headline and slug rules

**Headline** (becomes the SEO title unless `meta_title` overrides):

- Under **70 chars**.
- Match the search query closely.
- Lead with the verb / noun the searcher typed.

**Slug** (URL fragment):

- **<=60 chars.**
- **Strip stop words** — drop `the`, `a`, `an`, `for`, `with`, `in`, `to`, `of`, `on`, `and`, `or`, `is`, `are`.
- **No version numbers** — `n8n-1-45-2-fix` goes stale; `n8n-http-401-fix` does not.
- **Match the primary keyword**, not the full headline.

```python
import re
STOP = {'the','a','an','for','with','in','to','of','on','and','or','is','are'}
slug = "-".join(t for t in re.findall(r'[a-z0-9]+', topic.lower()) if t not in STOP)
slug = slug[:60].rstrip('-')
```

### 7b. Build JSON-LD schema for `codeinjection_head` (FAQPage + BreadcrumbList + HowTo)

Ghost emits Article/BlogPosting/Person/Organization schema by default. This skill **adds three more** for AI-citation extractability:

- **FAQPage** — mandatory. Every post has a FAQ block (Step 3 rule).
- **BreadcrumbList** — mandatory. `Home > <Primary Tag> > <Post Title>`.
- **HowTo** — only for procedural formats with >=3 step-shaped H2s.

**Critical Ghost gotcha:** Ghost converts the source HTML to its Lexical rich-text format on save, and the deserialiser silently drops `<script>` nodes — so JSON-LD inlined in the draft body **disappears in the live page** even though it was present in the POST payload. The blocks must go in the post's `codeinjection_head` field (stored verbatim and rendered into `<head>` via `{{ghost_head}}`).

**Never append `<script type="application/ld+json">` to the body HTML.** Build it once via this step into `<slug>.codeinjection-head.html`; Step 7f wires it into the payload's `codeinjection_head` field.

If the live page is missing schema after publish, the recovery is a follow-up PUT to `/posts/<id>/?source=html` setting `codeinjection_foot` (or `codeinjection_head`) to the contents of `<slug>.codeinjection-head.html`. Echo the post's current `updated_at` to avoid 409.

```bash
# Args: slug, headline, format, primary-tag-name, ghost-url
python3 - "<slug>" "<headline>" "<format>" "<primary-tag>" "$GHOST_URL" <<'PY'
import json, re, pathlib, sys
slug, headline, fmt, primary_tag, ghost_url = sys.argv[1:6]
ghost_url = ghost_url.rstrip('/')
draft = pathlib.Path(f"tmp/blog-drafts/{slug}.draft.html")
html = draft.read_text(encoding='utf-8')

def slugify(s):
    return re.sub(r'[^a-z0-9]+', '-', s.lower()).strip('-')

blocks = []

# 1. BreadcrumbList — always
blocks.append(("BreadcrumbList", {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": [
        {"@type":"ListItem","position":1,"name":"Home","item":f"{ghost_url}/"},
        {"@type":"ListItem","position":2,"name":primary_tag,
         "item":f"{ghost_url}/tag/{slugify(primary_tag)}/"},
        {"@type":"ListItem","position":3,"name":headline,
         "item":f"{ghost_url}/{slug}/"},
    ],
}))

# 2. FAQPage — extracted from the FAQ block
m = re.search(r'<h2[^>]*>\s*FAQ\s*</h2>(.*)$', html, flags=re.S|re.I)
qa = []
if m:
    pairs = re.findall(r'<h3[^>]*>(.*?)</h3>\s*<p[^>]*>(.*?)</p>', m.group(1), flags=re.S|re.I)
    qa = [{"@type":"Question",
           "name": re.sub(r'<[^>]+>','',q).strip(),
           "acceptedAnswer":{"@type":"Answer","text": re.sub(r'<[^>]+>','',a).strip()}}
          for q, a in pairs]
if qa:
    blocks.append(("FAQPage", {"@context":"https://schema.org","@type":"FAQPage","mainEntity":qa}))
else:
    print("WARN: no FAQ Q/A pairs found — Step 3 requires an FAQ block", file=sys.stderr)

# 3. HowTo — procedural formats with >=3 step-shaped H2s
if fmt in {"how-to-fix", "how-to-connect", "how-to-automate", "use-case", "migration"}:
    h2s = re.findall(r'<h2[^>]*>(.*?)</h2>', html)
    proc = [re.sub(r'<[^>]+>','',h).strip() for h in h2s
            if re.match(r'^\s*(Step|How to|Fix|Configure|Set up|Install|Create|Add|Enable)',
                        re.sub(r'<[^>]+>','',h).strip(), flags=re.I)]
    if len(proc) >= 3:
        blocks.append(("HowTo", {"@context":"https://schema.org","@type":"HowTo",
                                 "name": headline,
                                 "step":[{"@type":"HowToStep","name":s,"position":i+1}
                                         for i,s in enumerate(proc)]}))

ci = "\n".join(f'<script type="application/ld+json">{json.dumps(b, ensure_ascii=False)}</script>'
               for _, b in blocks)
pathlib.Path(f"tmp/blog-drafts/{slug}.codeinjection-head.html").write_text(ci, encoding='utf-8')
print(f"wrote {len(blocks)} JSON-LD block(s): {[t for t,_ in blocks]}")
PY
```

### 7c. Feature image (recommended)

Ghost displays a feature image at the top of every post and in social shares (OG image). Strongly recommended for any post you intend to promote.

You can:
- **Upload a custom image** via the upload endpoint (Step 6 snippet), then reference its URL in `feature_image`.
- **Generate a templated title card** — see the companion `blog-figure-svg` skill for a generator that produces 1600x840 OG cards with a clean headline + brand mark.
- **Skip it** — the post will display without a hero image; OG cards in social previews will fall back to the site default.

Whatever path you pick, set both `feature_image` (URL) and `feature_image_alt` (one-line description, **<=191 chars** — Ghost silently caps at `varchar(191)`).

### 7d. Author byline

Every post needs an author. Use the **slug** of an existing Ghost user (not email, not display name).

```python
"authors": [{"slug": "<author-slug>"}]
```

If the slug doesn't match a real user, Ghost silently substitutes the integration owner. Verify the response's `authors[0].slug` matches what you sent; if not, PUT a correction with the original `updated_at`.

Create authors via **Ghost admin -> Staff** (the API requires a separate auth flow). Capture the slug from the user's profile URL.

### 7e. Tags

Use tag objects with the `name` string (Ghost auto-creates tags that don't exist yet):

```python
"tags": [{"name": "How To"}, {"name": "n8n"}]
```

**Pick 1-3 tags per post.** The first tag is the **primary tag** — it becomes the breadcrumb segment in 7b and is used by most Ghost themes for category labelling.

Maintain a small canonical tag list in your project (don't let the AI invent new tags every post — duplicates dilute SEO). Common patterns: format tags (`How To`, `Tutorial`, `Comparison`, `What Is`) + topic tags (your tool/category names).

### 7f. Build the payload

```bash
python3 - <<'PY'
import json, os, pathlib, sys

# Edit per post:
SLUG = "<slug>"
HEADLINE = "<headline>"
TAGS = [{"name": "How To"}, {"name": "n8n"}]   # first entry is the primary tag passed to 7b
AUTHOR_SLUG = "<author-slug>"
FEATURE_IMAGE = "<https://your-ghost-site/content/images/.../feature.png>"   # or ""
FEATURE_IMAGE_ALT = "<one-line alt text, <=191 chars>"
FEATURE_IMAGE_CAPTION = "<one sentence, 12-25 words, restates the post promise>"
META_TITLE = "<SEO title under 60 chars>"
META_DESCRIPTION = "<SEO description, 140-160 chars>"
CUSTOM_EXCERPT = "<dek shown on index page>"
PUBLISH_FLAG = False              # set by --publish
PUBLISH_AT_ISO = None             # set by --publish-at <iso>

html = pathlib.Path(f"tmp/blog-drafts/{SLUG}.draft.html").read_text(encoding="utf-8")

ci_path = pathlib.Path(f"tmp/blog-drafts/{SLUG}.codeinjection-head.html")
if not ci_path.exists():
    sys.exit("missing codeinjection-head file — run Step 7b before payload build")
codeinjection_head = ci_path.read_text(encoding="utf-8")

# status / published_at:
#   default              -> status="draft",     no published_at
#   --publish            -> status="published", no published_at
#   --publish-at <iso>   -> status="scheduled", published_at="<iso>" (future, UTC)
status, published_at = "draft", None
if PUBLISH_AT_ISO:
    status, published_at = "scheduled", PUBLISH_AT_ISO
elif PUBLISH_FLAG:
    status = "published"

post = {
    "title": HEADLINE,
    "slug": SLUG,
    "html": html,
    "status": status,
    "tags": TAGS,
    "authors": [{"slug": AUTHOR_SLUG}],
    "meta_title": META_TITLE,
    "meta_description": META_DESCRIPTION,
    "custom_excerpt": CUSTOM_EXCERPT,
    "codeinjection_head": codeinjection_head,
}
if FEATURE_IMAGE:
    post["feature_image"] = FEATURE_IMAGE
    post["feature_image_alt"] = FEATURE_IMAGE_ALT
    post["feature_image_caption"] = FEATURE_IMAGE_CAPTION
if published_at:
    post["published_at"] = published_at

payload = {"posts": [post]}
pathlib.Path(f"tmp/blog-drafts/{SLUG}.payload.json").write_text(
    json.dumps(payload, ensure_ascii=False, indent=2), encoding="utf-8")
print("payload written")
PY
```

### 7g. Pre-publish payload validation

Before POSTing, all of these must hold:

```bash
python3 - "<slug>" <<'PY'
import json, pathlib, re, sys
slug = sys.argv[1]
p = json.loads(pathlib.Path(f"tmp/blog-drafts/{slug}.payload.json").read_text())
post = p["posts"][0]
html = post["html"]

assert post.get("authors") and post["authors"][0].get("slug"), "authors[0].slug missing"
assert post.get("tags"), "tags array empty"

# Feature image: if set, alt text is required and capped at 191
if post.get("feature_image"):
    alt = post.get("feature_image_alt", "")
    assert alt.strip(), "feature_image_alt required when feature_image is set"
    assert len(alt) <= 191, \
        f"feature_image_alt is {len(alt)} chars; Ghost silently caps at 191 (varchar(191))"

# JSON-LD in codeinjection_head
ci = post.get("codeinjection_head", "")
assert '"@type": "FAQPage"' in ci or '"@type":"FAQPage"' in ci, \
       "FAQPage JSON-LD missing in codeinjection_head - re-run 7b"
assert '"@type": "BreadcrumbList"' in ci or '"@type":"BreadcrumbList"' in ci, \
       "BreadcrumbList JSON-LD missing in codeinjection_head - re-run 7b"

# TL;DR block check
m_first_p = re.search(r'<p\b[^>]*>(.*?)</p>', html, flags=re.S|re.I)
assert m_first_p, "no <p> in body — TL;DR check cannot run"
first_p_inner = m_first_p.group(1)
assert re.search(r'^\s*<strong>\s*TL;DR\s*:?\s*</strong>', first_p_inner, flags=re.I), \
       "first <p> must open with <strong>TL;DR:</strong>"
_t = re.sub(r'<code\b[^>]*>.*?</code>', '', first_p_inner, flags=re.S|re.I)
_t = re.sub(r'<pre\b[^>]*>.*?</pre>', '', _t, flags=re.S|re.I)
tldr_text = re.sub(r'<[^>]+>', '', _t)
tldr_text = re.sub(r'^\s*TL;DR\s*:?\s*', '', tldr_text, flags=re.I).strip()
tldr_words = len(re.findall(r"[A-Za-z0-9][A-Za-z0-9'\-]*", tldr_text))
assert 8 <= tldr_words <= 40, f"TL;DR must be 8-40 words, got {tldr_words}: {tldr_text!r}"
mid_sentence_ends = len(re.findall(r'(?<!\.)[.!?]\s+[A-Z(]', tldr_text))
assert mid_sentence_ends == 0, \
       f"TL;DR must be a single sentence; got: {tldr_text!r}"

# Scheduled posts need a future timestamp
if post.get("status") == "scheduled":
    import datetime
    pa = post.get("published_at", "")
    assert pa, "scheduled posts require published_at"
    ts = datetime.datetime.fromisoformat(pa.replace("Z","+00:00"))
    assert ts > datetime.datetime.now(datetime.timezone.utc), \
           f"scheduled published_at must be in the future, got {pa}"

# Figure caption gate: every <figure> must contain a non-empty <figcaption>
figures = re.findall(r'<figure\b[^>]*>.*?</figure>', html, flags=re.S|re.I)
uncaptioned = []
for i, fig in enumerate(figures, 1):
    cap = re.search(r'<figcaption\b[^>]*>(.*?)</figcaption>', fig, flags=re.S|re.I)
    if not cap or not re.sub(r'<[^>]+>', '', cap.group(1)).strip():
        src = re.search(r'<img[^>]*src="([^"]+)"', fig)
        uncaptioned.append(f"figure {i} ({src.group(1) if src else 'no src'})")
assert not uncaptioned, \
    "missing/empty <figcaption> on: " + ", ".join(uncaptioned)

print(f"payload OK ({len(figures)} figures, all captioned)")
PY
```

If any assert fires, fix and re-build before Step 8.

---

## Step 8 — Publish via Ghost Admin API

### 8a. Generate a JWT and POST the post

Ghost Admin API uses short-lived JWTs (5 min) keyed by the integration's admin key. The script splits `GHOST_ADMIN_KEY` on the colon, signs a JWT with the secret half, and includes the id half as the `kid` header.

```bash
python3 - "<slug>" <<'PY'
import os, sys, json, pathlib, datetime, requests, jwt

slug = sys.argv[1]
ghost_url = os.environ['GHOST_URL'].rstrip('/')
key = os.environ['GHOST_ADMIN_KEY']
kid, secret = key.split(':', 1)

iat = int(datetime.datetime.now(datetime.timezone.utc).timestamp())
token = jwt.encode(
    {'iat': iat, 'exp': iat + 5 * 60, 'aud': '/admin/'},
    bytes.fromhex(secret),
    algorithm='HS256',
    headers={'kid': kid, 'alg': 'HS256', 'typ': 'JWT'},
)

payload = json.loads(pathlib.Path(f"tmp/blog-drafts/{slug}.payload.json").read_text())
r = requests.post(
    f"{ghost_url}/ghost/api/admin/posts/?source=html",
    headers={
        'Authorization': f'Ghost {token}',
        'Content-Type': 'application/json',
        'Accept-Version': 'v5.0',
    },
    json=payload,
)
if not r.ok:
    print(f"FAILED {r.status_code}: {r.text}", file=sys.stderr)
    sys.exit(1)
resp = r.json()
post = resp['posts'][0]
print(json.dumps({
    'id': post['id'],
    'url': post.get('url'),
    'slug': post.get('slug'),
    'status': post.get('status'),
    'published_at': post.get('published_at'),
    'authors': [a.get('slug') for a in post.get('authors', [])],
}, indent=2))
PY
```

`?source=html` tells Ghost to convert the `html` field into Lexical. Without it, Ghost treats the field as Lexical JSON and the POST fails with a 422.

The response prints the created post's id, url, slug, and status. Capture these for verification.

**Python deps:** `pip install requests pyjwt`. PyJWT 2.x is required (1.x uses a different signature).

### 8b. Report back to the user

- Draft URL or live URL (`<ghost-url>/<slug>/` if published; admin preview URL if draft).
- Ghost admin edit URL (`<ghost-url>/ghost/#/editor/post/<id>`).
- Word count, tag list, author slug.
- Confirmation: scrub passed, AI-SEO audit applied, FAQ block present, JSON-LD injected.
- Figure URLs and captions.

---

## Step 9 — Verify live post (only if `--publish`)

```bash
# Post is reachable
curl -sSI "$GHOST_URL/<slug>/" | head -5

# Post in RSS
curl -sS "$GHOST_URL/rss/" | grep -o "<title>[^<]*</title>" | head -5

# Post in sitemap
curl -sS "$GHOST_URL/sitemap-posts.xml" | grep "<slug>"

# OG + full schema set rendered
curl -sS "$GHOST_URL/<slug>/" | grep -o 'property="og:[^"]*"' | sort -u
curl -sS "$GHOST_URL/<slug>/" | grep -oE '"@type":\s*"[^"]+"' | sort -u
```

**Expected:** `HTTP/2 200`, slug in RSS and sitemap, `og:title`/`og:description` present. The `"@type"` set must include **`Article`** (or `BlogPosting`), **`FAQPage`**, and **`BreadcrumbList`**; procedural how-to posts must also include **`HowTo`**. Missing FAQPage/BreadcrumbList means `codeinjection_head` was dropped — check the Ghost admin Code Injection panel for that post.

---

## What this skill does NOT do

- **Does not commit to git.** Ghost stores posts in its own database. Only file writes happen in `tmp/blog-drafts/` (gitignored).
- **Does not schedule posts by default.** Pass `--publish-at <ISO-UTC>` to schedule one. Without the flag the post lands as draft (default) or live (`--publish`).
- **Does not handle member-only posts, newsletters, or email sends.** Ghost's newsletter flow is manual via admin UI.
- **Does not generate figures.** Use the companion `blog-figure-svg` skill for SVG charts, taxonomies, and flow diagrams.
- **Does not research topics from scratch.** Use the companion `blog-topic-research` skill to validate a topic has real demand signals before drafting.

---

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Key expired / wrong key | Regenerate under Ghost admin -> Integrations; update `GHOST_ADMIN_KEY` |
| `422 Validation failed: Value in [posts.html] cannot be blank` | Missing `?source=html` | Add the query param |
| `422` with `feature_image_alt` in message | Alt text >191 chars (Ghost's silent varchar(191) cap) | Trim to <=191; Step 7g asserts this |
| `404` on slug after publish | Post saved as draft (default) | Drafts only reachable via admin editor URL |
| Body shows as one HTML blob | Ghost fell back to plain-text mode | Re-post with `?source=html` |
| Smart quotes reappear in rendered post | Ghost typographer auto-conversion | Settings -> Publication: turn off "Use typographer's quotes" |
| Wrong slug | Ghost auto-slugged from title | PUT `/posts/<id>/` with corrected slug + current `updated_at` |
| `409 Conflict` on PUT | Stale `updated_at` | Re-GET to refresh, retry |
| Author silently substituted with integration owner | Author slug doesn't exist | Create the user in Ghost admin -> Staff; PUT correction with correct slug |
| Live page missing FAQPage / HowTo `@type` (Step 9) | JSON-LD was inlined in the draft body and stripped by Ghost's Lexical conversion | PUT `/posts/<id>/?source=html` with `codeinjection_head` set to `<slug>.codeinjection-head.html`; echo current `updated_at` to avoid 409 |

---

## Companion skills

- **`blog-topic-research`** — validate a long-tail topic has real demand signals (PAA, Reddit threads, GitHub issues) before drafting. Run this *before* this skill.
- **`blog-figure-svg`** — generate accessible SVG figures (flow diagrams, comparison charts, taxonomy diagrams) with consistent styling. Run this *during Step 6* if the post needs illustrations.

Together, the three form a complete long-tail SEO publishing pipeline: research the topic, write the post, illustrate it, publish to Ghost.
