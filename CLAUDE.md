# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Multi-site content archive for the **MK Group** brand family. Each subfolder captures one website as local Obsidian Markdown documents with downloaded assets and full-page screenshots.

**Sites archived:**

| Subfolder | Site | Pages | Language |
|-----------|------|-------|----------|
| `mkbayarea.com/` | mkbayarea.com | 133 | zh-CN / en |
| `estatesmk.com/` | estatesmk.com | 37 | zh-CN / en |
| `bayareaschoolguide.com/` | bayareaschoolguide.com | 92 | zh-CN / en |

All three site subfolders live inside `MKgroup/`.

---

## Directory Structure

Each site subfolder follows the same layout:

```
MKgroup/<site-folder>/
├── pages/          # One .md per archived page
├── assets/         # Downloaded images (logos, photos, thumbnails)
├── screenshots/    # Full-page PNG screenshots (1440px wide)
└── sitemap.md      # Progress tracker — all pages with ✅/⬜ status
```

No HTML files are saved — only the processed `.md` output.

---

## Slug Convention

URL paths map to filenames by replacing `/` with `-` and stripping the leading slash:

| URL path | Slug / filename |
|----------|----------------|
| `/` | `index` |
| `/sell/process` | `sell-process` |
| `/districts/palo-alto/gunn-high-school` | `districts-palo-alto-gunn-high-school` |

---

## Markdown Format

Pages use **Obsidian Flavored Markdown** with YAML frontmatter:

```yaml
---
title:
aliases:
  - <short title>
url:              # full source URL
page_path:        # URL path only (e.g. /sell/process)
site:             # domain (e.g. mkbayarea.com)
organization:     # brand name
date_archived:    # YYYY-MM-DD
language:         # list, e.g. [zh-CN, en]
screenshot:       # relative path from project root (e.g. screenshots/sell-process.png)
tags:
  - MKgroup
  - real-estate
  - bay-area
---
```

**Content conventions:**
- Images: `![[assets/filename.png]]` — paths relative to site root, not `pages/`
- Callouts: `> [!quote]`, `> [!tip]`, `> [!success]` for highlighted content
- Tables for stats, comparisons, fee structures
- End every page with `![[screenshots/<slug>.png]]`

---

## Scraping Pipeline

New pages are scraped with a two-script pipeline. Scripts live in the installed skill:

```
~/.claude/skills/scrape-static-website/scripts/
├── scrape-page.js   # Puppeteer renderer → prints content JSON to stdout
└── gen-md.js        # Reads JSON from stdin → writes pages/<slug>.md
```

**Per-page command:**
```bash
node ~/.claude/skills/scrape-static-website/scripts/scrape-page.js \
  "<url>" "<project-root>" 2>/dev/null \
  | node ~/.claude/skills/scrape-static-website/scripts/gen-md.js "<project-root>"
```

**Parallel batch of 5:**
```bash
ROOT="/Users/junyuanlin/Project/MKgroup/mkbayarea.com"
SCRAPE=~/.claude/skills/scrape-static-website/scripts/scrape-page.js
GEN=~/.claude/skills/scrape-static-website/scripts/gen-md.js

for url in URL1 URL2 URL3 URL4 URL5; do
  (node "$SCRAPE" "$url" "$ROOT" 2>/dev/null | node "$GEN" "$ROOT") &
done
wait
```

**Puppeteer** must be installed at `/tmp/pptr-shot/node_modules/puppeteer`. If missing:
```bash
mkdir -p /tmp/pptr-shot && cd /tmp/pptr-shot
echo '{"name":"pptr-shot","version":"1.0.0","type":"commonjs"}' > package.json
npm install puppeteer
```

Site-specific gen-md variants (different org name, tags, footer) are at:
- `/tmp/gen-md-estatesmk.js` — for estatesmk.com
- `/tmp/gen-md-schoolguide.js` — for bayareaschoolguide.com

---

## Discovering Pages

Fetch all URLs from a site's sitemap, then filter exclusions:

```bash
curl -s "https://<domain>/sitemap.xml" \
  | grep -o '<loc>[^<]*</loc>' \
  | sed 's/<loc>//g;s/<\/loc>//g' \
  | sort
```

**Standard exclusions:** `/privacy`, `/privacy/en`, `/terms`, `/terms/en`, `/videos/*`, `/en/*` (English mirrors of Chinese pages).

---

## sitemap.md Format

Each site has a `sitemap.md` in its root tracking all pages:

```markdown
| Slug | URL | MD File | Status |
|------|-----|---------|--------|
| `sell-process` | https://mkbayarea.com/sell/process | [[pages/sell-process]] | ✅ |
```

Update ✅/⬜ after scraping. The status column uses ✅ for archived, ⬜ for pending.

---

## Skill

The `/scrape-static-website` skill (installed at `~/.claude/skills/scrape-static-website/`) automates the full workflow: sitemap discovery → filtering → batched parallel scraping → sitemap.md generation. Invoke it with `/scrape-static-website` or by asking Claude to scrape/archive a website.
