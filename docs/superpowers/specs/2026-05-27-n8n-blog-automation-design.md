# n8n Blog Automation Workflow — Design Spec

**Date:** 2026-05-27  
**Project:** Techvia Blog (`blogs.techvia.software`)  
**Status:** Approved

---

## Overview

A daily automated n8n workflow that researches trending tech topics, gets Telegram approval, writes a full Hugo blog post using OpenRouter, generates a cover image via Gemini Flash + Cloudinary, assembles the markdown file, and commits it directly to GitHub — triggering an automatic Cloudflare Pages deployment.

---

## Credentials Required

| Credential | n8n Type | Status |
|-----------|---------|--------|
| OpenRouter API key | HTTP Header Auth | Ready |
| Google Gemini Flash API key | HTTP Header Auth / Google AI | Ready |
| Cloudinary (cloud name, API key, secret) | Cloudinary node | Ready |
| Telegram Bot token + Chat ID | Telegram node | Ready |
| GitHub Personal Access Token | GitHub node | **Pending — user to add** |

---

## Workflow: `techvia-daily-blog`

### Node 1 — Schedule Trigger
- **Type:** Cron
- **Schedule:** Daily at 06:00 AM (configurable)
- **Output:** Passes timestamp to next node

---

### Node 2 — Trend Research Agent
- **Type:** HTTP Request → OpenRouter Chat Completion
- **Model:** Configurable (default: `google/gemini-flash-1.5` or `anthropic/claude-3.5-sonnet`)
- **Tools used:** Serper.dev API for real-time web search results
- **System prompt:**
  > You are a tech content strategist for Techvia — a company offering Web Development, Cloud Solutions, Mobile Development, and IT Consulting. Search for the 3 most relevant trending tech topics from the last 48 hours that align with these pillars. For each topic return: title (catchy blog headline), angle (1-sentence unique hook), pillar (which service it relates to). Return JSON array only.
- **Output:** JSON array of 3 topic objects `[{ title, angle, pillar }]`

---

### Node 3 — Telegram Approval
- **Type:** Telegram → Send Message + Wait for Response
- **Message format:**
  ```
  📰 Daily Blog Topics — pick one:

  1️⃣ <title 1>
     └ <angle 1> [<pillar 1>]

  2️⃣ <title 2>
     └ <angle 2> [<pillar 2>]

  3️⃣ <title 3>
     └ <angle 3> [<pillar 3>]

  Reply with 1, 2, or 3 to proceed. Reply SKIP to abort.
  ```
- **Timeout:** 30 minutes — if no reply, sends "⏭ No response. Skipping today." and stops workflow
- **Output:** Selected topic object

---

### Node 4 — Content Writer Agent
- **Type:** HTTP Request → OpenRouter Chat Completion
- **Model:** Configurable via n8n variable (e.g. `anthropic/claude-sonnet-4-5`, `openai/gpt-4o`)
- **System prompt:**
  > You are a senior tech writer for Techvia. Write a complete, SEO-optimised blog post for the Hugo static site generator. Use markdown with proper H2/H3 headings. Include: intro (hook + what reader will learn), 3-5 detailed sections with practical examples or code snippets where relevant, and a conclusion with a CTA back to techvia.software. Tone: professional but approachable. Target length: 800-1200 words.
- **Input:** Selected topic title + angle
- **Output:** Raw markdown content string

---

### Node 5A — Metadata Agent *(parallel)*
- **Type:** HTTP Request → OpenRouter Chat Completion
- **Model:** Fast/cheap model (e.g. `google/gemini-flash-1.5`)
- **Task:** Given the article content, return JSON:
  ```json
  {
    "slug": "kebab-case-slug-no-date",
    "seo_description": "max 150 chars",
    "tags": ["tag1", "tag2", "tag3"],
    "category": "one of: Web Development | Cloud Solutions | Mobile Development | IT Consulting | SaaS Development | AI Development"
  }
  ```
- **Output:** Metadata JSON

---

### Node 5B — Image Agent *(parallel)*
- **Type:** Multi-step sub-flow
  1. **Gemini Flash** — given article title + content summary, generate an image search query and alt text
  2. **Unsplash API** (`/search/photos?query=...&per_page=1`) — fetch best matching photo URL
  3. **Cloudinary Upload** — upload image URL to Cloudinary folder `blog/`
  4. **Output:** `{ cloudinary_url, alt_text }`
- **Fallback:** If Unsplash returns no results, use a topic-relevant default from static list per pillar

---

### Node 6 — Assembler
- **Type:** Code node (JavaScript)
- **Input:** content (Node 4) + metadata (Node 5A) + image (Node 5B) + selected topic
- **Logic:**
  ```js
  const today = new Date().toISOString().split('T')[0]; // YYYY-MM-DD
  const filename = `${today}-${metadata.slug}.md`;
  const frontmatter = `---
title: "${topic.title}"
date: ${today}
draft: false
tags: ${JSON.stringify(metadata.tags)}
categories: ["${metadata.category}"]
description: "${metadata.seo_description}"
cover:
  image: "${image.cloudinary_url}"
  alt: "${image.alt_text}"
  relative: false
showToc: true
---

`;
  return { filename, content: frontmatter + articleContent };
  ```
- **Output:** `{ filename, content, path: "content/posts/" }`

---

### Node 7 — GitHub Commit
- **Type:** GitHub node (or HTTP Request to GitHub Contents API)
- **Endpoint:** `PUT /repos/nikesh9220/techvia-blog/contents/content/posts/{filename}`
- **Branch:** `master`
- **Commit message:** `feat: auto-publish "{title}" via n8n`
- **Auth:** GitHub Personal Access Token
- **Output:** Commit SHA + HTML URL of file

---

### Node 8 — Telegram Success Notification
- **Type:** Telegram Send Message
- **Message:**
  ```
  ✅ Published!

  📝 <title>
  🏷 <tags>
  🔗 https://blogs.techvia.software/posts/<slug>/

  Cloudflare is deploying now (~30 sec).
  ```

### Node 8B — Error Handler (on any node failure)
- **Type:** Telegram Send Message (connected to error output of Nodes 4–7)
- **Message:** `❌ Blog automation failed at [node name]: [error message]`

---

## Data Flow Summary

```
Scheduler
  └→ Trend Research (OpenRouter + Search)
       └→ Telegram Approval (wait for 1/2/3)
            └→ Content Writer (OpenRouter)
                 ├→ [parallel] Metadata Agent (OpenRouter fast model)
                 └→ [parallel] Image Agent (Gemini Flash → Unsplash → Cloudinary)
                      └→ Assembler (Code node)
                           └→ GitHub Commit
                                └→ Telegram: ✅ Published
```

---

## Configuration Variables (n8n Workflow Variables)

| Variable | Default | Description |
|---------|---------|-------------|
| `WRITING_MODEL` | `anthropic/claude-sonnet-4-5` | OpenRouter model for content |
| `FAST_MODEL` | `google/gemini-flash-1.5` | OpenRouter model for metadata |
| `CRON_HOUR` | `6` | Hour to run (24h, UTC+5:30) |
| `TELEGRAM_CHAT_ID` | — | Your Telegram chat ID |
| `CLOUDINARY_FOLDER` | `blog` | Cloudinary upload folder |
| `APPROVAL_TIMEOUT_MIN` | `30` | Minutes to wait for Telegram reply |

---

## What's Out of Scope
- Article editing after publish (manual via GitHub)
- Multi-language posts
- Social media sharing (can be added as Node 9 later)
- Duplicate topic detection (future enhancement)
