# publishing-skills

Three composable Claude skills for shipping a long-tail SEO blog post end-to-end:

| Skill | What it does | Clawhub |
|---|---|---|
| **`blog-topic-research`** | Validates a topic has real, verifiable demand (PAA, Reddit threads, GitHub issues) before drafting. Outputs a research-proof scaffold the writer skill can consume. | [clawhub.ai/skills/blog-topic-research](https://clawhub.ai/skills/blog-topic-research) |
| **`ghost-blog-writer`** | End-to-end pipeline for a single Ghost CMS post: research -> draft -> scrub LLM tells -> AI-SEO audit -> publish via Admin API. Adds FAQPage + BreadcrumbList + HowTo JSON-LD for AI-citation extractability. | [clawhub.ai/skills/ghost-blog-writer](https://clawhub.ai/skills/ghost-blog-writer) |
| **`blog-figure-svg`** | Generates accessible SVG figures (flow, comparison bars, taxonomy, terminal mocks, OG feature cards) with system-font palette, rasterizes to compressed PNG. | [clawhub.ai/skills/blog-figure-svg](https://clawhub.ai/skills/blog-figure-svg) |

Together, they form a complete pipeline: **research the topic -> write the post -> illustrate it -> publish to Ghost**.

## Installing

### Via skills.sh (any agent runtime)

Installs all 3 skills into Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 50+ other runtimes at once:

```bash
npx skills add ratamaha-git/publishing-skills
```

### Via clawhub CLI (Claude / OpenClaw runtimes)

```bash
npm i -g clawhub
clawhub login
clawhub skill install blog-topic-research
clawhub skill install ghost-blog-writer
clawhub skill install blog-figure-svg
```

### Manually

Each skill is a single `SKILL.md` file with YAML frontmatter. Drop it into your project's `.claude/skills/<name>/` directory (or your agent runtime's equivalent skill folder) and reload.

```bash
git clone https://github.com/ratamaha-git/publishing-skills.git
cp -r publishing-skills/skills/* .claude/skills/
```

## How they compose

```
+----------------------+      +------------------+      +-------------------+      +-------------------+
| blog-topic-research  | ---> | (operator review)| ---> | ghost-blog-writer | <--- |  blog-figure-svg  |
| 'is the topic worth  |      |  'pick what to   |      | 'write + publish' |      | 'illustrate the   |
|  writing about?'     |      |   write next'    |      |                   |      |   post'           |
+----------------------+      +------------------+      +-------------------+      +-------------------+
                                                                |
                                                                v
                                                       Ghost CMS (your blog)
```

- **`blog-topic-research`** runs first — it answers "does anyone actually search this?" before you spend tokens drafting.
- **`ghost-blog-writer`** consumes the research-proof scaffold (or runs from scratch), drafts the HTML, validates the payload, and posts to Ghost.
- **`blog-figure-svg`** is invoked from within the writer's illustration step (or standalone) when a post needs charts, diagrams, or a feature card.

## Requirements

- **`blog-topic-research`** — no system dependencies beyond `WebSearch` / `WebFetch` in the agent runtime.
- **`ghost-blog-writer`** — Python 3 (`requests`, `pyjwt`), `GHOST_URL` and `GHOST_ADMIN_KEY` env vars pointing at your Ghost instance.
- **`blog-figure-svg`** — Python 3, plus one SVG rasterizer (ImageMagick / `rsvg-convert` / Inkscape / `cairosvg`) and optionally `pngquant` for compression.

## Provenance

Extracted and generalized from the editorial pipeline running [automatelab.tech](https://automatelab.tech). The internal versions (`al-topic-research`, `al-write-blog-post`) are coupled to that site's tag taxonomy, cluster palettes, and backlog state; this repo strips those internals and parameterizes everything site-specific (hostname, author slug, tags, palettes) so the skills work against any Ghost blog.

## License

[MIT-0](LICENSE) — public domain equivalent. Use, modify, redistribute without attribution.
