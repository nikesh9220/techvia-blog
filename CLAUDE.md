# Techvia Blog

## Project
Hugo static blog at `blogs.techvia.software` — deployed via Cloudflare Pages from `github.com/nikesh9220/techvia-blog`.

- **Stack:** Hugo v0.147+ Extended + PaperMod theme + Cloudflare Pages + Cloudinary (images)
- **Main site:** https://www.techvia.software/
- **GitHub:** git@github.com:nikesh9220/techvia-blog.git
- **Branch:** master

## Local Dev
```bash
hugo server --bind 0.0.0.0   # http://localhost:1313
hugo --minify                 # production build → public/
```

## Adding New Posts
Create `content/posts/YYYY-MM-DD-slug.md` using this frontmatter:

```yaml
---
title: ""
date: YYYY-MM-DD
draft: false
tags: ["tag1", "tag2"]
categories: ["Category"]
description: ""   # 150 chars max, used for SEO meta
cover:
  image: ""       # Cloudinary URL
  alt: ""
  relative: false
showToc: true
---
```

## n8n Automation
n8n commits articles directly to `content/posts/` on the master branch. Cloudflare auto-deploys on every push. Use the frontmatter above exactly.

## Branding
- **Font:** Inter (Google Fonts)
- **Brand Blue:** `#2563EB` — links, logo, nav, tags
- **Deep Blue:** `#1E40AF` — article headings
- **Dark Slate:** `#1E293B` — body text
- **Light Gray:** `#F8FAFC` — page background
- Custom CSS: `assets/css/extended/custom.css`

## Cloudflare Pages Config
- Build command: `hugo --minify`
- Output dir: `public`
- `wrangler.jsonc` is committed — do not delete it

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Theme missing | `git submodule update --init --recursive` |
| Post not showing | Check `draft: false` in frontmatter |
| Build fails on Cloudflare | Ensure `HUGO_VERSION = 0.147.0` env var is set |
| Search broken | Ensure `outputs` includes `JSON` in hugo.toml |
