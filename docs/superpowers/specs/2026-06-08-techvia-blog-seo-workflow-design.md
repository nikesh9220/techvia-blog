# Design: SEO Blog Workflow with Postgres Tracking + R2 Images (`techvia-blog-seo`)

**Date:** 2026-06-08
**Status:** Approved design → pending implementation plan
**Author:** Nikesh Pandya

> **Secrets policy:** No credentials (Postgres password, R2 keys, Cloudflare token) are stored in
> this document or anywhere in git. They live **only** in n8n credentials, configured at build time.

## 1. Purpose

The existing `n8n/workflows/techvia-blog-auto.json` publishes one trend-chasing post per day with no
memory, risking topic repeats and producing recognizably "AI-written" copy. This new, **parallel**
workflow (`techvia-blog-seo`) instead:

- Sources **SEO / buyer-intent topics** designed to bring prospective customers to Techvia.
- Tracks every published post in **local Postgres** and **deduplicates** topics over a 3-month window.
- Writes **human-sounding** content (anti-AI-tell prompt discipline).
- Generates **one cover image per article** via OpenRouter, hosts it on **Cloudflare R2**.

The existing `techvia-blog-auto.json` is **left untouched** as a fallback.

## 2. High-level flow

```
Daily Schedule (8AM UTC)
  → Build SEO Context (code: Techvia pillars + buyer-intent framing)
  → LLM #1  Generate ~5 candidate buyer-intent topics  (OpenRouter text model → JSON array)
  → Parse Candidates (code)
  → Postgres  Check Recent Slugs  (SELECT slug WHERE published_at > now() - interval '3 months')
  → Filter Fresh (code: drop already-published, pick top survivor)
       ├─ none fresh → LLM #1b round-2 (exclude round-1 titles) → re-filter
       └─ fresh found ↓
  → LLM #2  Write Article  (human voice; returns JSON: body + slug + seo_description + tags + category)
  → LLM #3  Generate cover image  (OpenRouter image model → base64 data URI)
  → Upload to R2  (HTTP S3 PUT → object key blog/{date}-{slug}.png)
  → Assemble Markdown (code: Hugo frontmatter incl. cover.image = R2 public URL)
  → Base64 Encode → Commit to GitHub (content/posts/{date}-{slug}.md)
  → Postgres  Insert published_topics  (ON CONFLICT (slug) DO NOTHING)
  → Telegram Notify
```

**"One LLM node for text":** a single `lmChatOpenRouter` text-model node is reused by both the
candidate chain (LLM #1) and the writer chain (LLM #2). The two *steps* exist only so dedup runs on a
cheap candidate list **before** spending a full article generation. Plus one separate image node.

## 3. Topic engine (SEO / buyer-intent)

LLM #1 generates candidate topics from Techvia's service pillars — Web Development, Cloud Solutions,
Mobile Development, IT Consulting, SaaS Development, AI Development — framed for **buyer intent**
(problems prospects search for, "how to / cost of / best X for Y / X vs Y / migration / checklist"
long-tail angles that pull commercial-intent traffic toward Techvia services).

- Output: JSON array of 5 ranked candidates, each `{ title, angle, pillar, slug }`.
- `slug` is produced deterministically (see §6) so the dedup key is stable across runs.

## 4. Human-sounding content (LLM #2)

Writer prompt enforces a human voice:
- Vary sentence length; allow short punchy sentences and occasional fragments.
- Team/first-person voice ("we've seen…", "in our projects…"); concrete examples; mild opinion.
- **Ban AI-tell phrases:** "in today's fast-paced world", "delve", "moreover", "furthermore",
  "in conclusion", "unlock the power", "ever-evolving", "landscape", "navigating the".
- Prefer prose; at most one bulleted list; code blocks only where the pillar warrants.
- 900–1200 words, `##`/`###` headings, natural closing CTA to https://techvia.software.
- Returns a single JSON object: `{ body_markdown, slug, seo_description (<150 chars), tags[], category }`.

## 5. Image generation + R2 hosting

- **Generate:** OpenRouter image model, default `google/gemini-2.5-flash-image-preview`, overridable via
  `$vars.IMAGE_MODEL`. Prompt derived from article title + angle. Returns a base64 data URI.
  *(Build-time: verify exact response path — expected `choices[0].message.images[0].image_url.url`.)*
- **Upload:** S3-compatible HTTP PUT to R2.
  - Endpoint: `https://<account>.r2.cloudflarestorage.com` (account id stored in n8n credential).
  - Bucket: `techvia-blog-images`
  - Object key: `blog/{YYYY-MM-DD}-{slug}.png`, `Content-Type: image/png`.
  - Auth: AWS SigV4 with R2 Access Key / Secret (n8n S3/credential or signed HTTP node).
- **Reference in frontmatter:** `cover.image = https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev/blog/{date}-{slug}.png`,
  `cover.relative: false`.
- **Public URL (confirmed):** `https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev` (R2 bucket Public
  Development URL). The `…r2.cloudflarestorage.com` S3 endpoint is auth-only and used for **upload**;
  the `pub-….r2.dev` URL is used for **serving** the image in `cover.image`.

## 6. Postgres (local) — dedup + full tracking

Connection: host `localhost:5432`, database `techvia`, user `techvia` (password in n8n credential only).

```sql
CREATE TABLE IF NOT EXISTS published_topics (
  id              SERIAL PRIMARY KEY,
  slug            TEXT UNIQUE NOT NULL,   -- deterministic normalized dedup key
  title           TEXT NOT NULL,
  pillar          TEXT,
  angle           TEXT,
  seo_description TEXT,
  tags            TEXT[],
  post_path       TEXT,                   -- content/posts/{date}-{slug}.md
  image_url       TEXT,                   -- R2 public URL
  published_at    TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_published_topics_published_at
  ON published_topics (published_at);
```

- **Slug normalization (deterministic, no embeddings):** lowercase → strip punctuation → remove
  stopwords → collapse whitespace → hyphenate. Same function used for candidate slugs and the dedup key.
- **Dedup window:** 3 months. `Check Recent Slugs` selects slugs with `published_at > now() - interval
  '3 months'`; `Filter Fresh` removes those from the candidate set and keeps the top-ranked survivor.
- **Two-round fallback:** if round 1 yields no survivor, LLM #1b regenerates candidates excluding the
  round-1 titles; re-filter. If round 2 also empty → Telegram "skipped, all candidates recent", no commit.
- **Insert timing:** AFTER successful GitHub commit, `ON CONFLICT (slug) DO NOTHING` (idempotent — a
  retried run never double-inserts and only successful publications poison the history).

## 7. Error handling

- **Postgres SELECT failure → block publish.** Never silently skip dedup (would risk a duplicate post).
- **Postgres INSERT failure → log + continue.** The commit already happened; tracking is best-effort.
- **Image generation / R2 upload failure → ** fail the run with a Telegram error notice rather than
  committing a post with a broken `cover.image`. (Build-time decision: optionally allow publish without
  cover — default is to fail, keeping posts consistent.)
- **Malformed LLM JSON → ** defensive `JSON.parse` in code nodes (strip ```` ```json ```` fences) with a
  clear error that surfaces to the execution log + Telegram.

## 8. Credentials required in n8n

| Credential | Status | Use |
|---|---|---|
| OpenRouter (`DS7MeEgya97Jx9db`) | exists | text (LLM #1/#2) + image (LLM #3) |
| GitHub PAT (`WpSYxNEHvb4nvnGh`) | exists | commit post to repo |
| Telegram Bot (`QDbQhiQDyGyIbqse`) | exists | notifications |
| Postgres (local) | **new** | dedup + tracking |
| R2 S3 (access key / secret / endpoint) | **new** | image upload |

## 9. Deliverables

1. SQL setup script (create `published_topics` + index) — run once against local Postgres.
2. `n8n/workflows/techvia-blog-seo.json` — the new workflow.
3. n8n credentials created/wired (Postgres, R2) — manual, secrets never committed.
4. R2 public access enabled + public URL captured.
5. One end-to-end test run verified (post committed, image visible, row inserted, no duplicate on re-run).

## 10. Out of scope / YAGNI

- No semantic/embedding dedup (deterministic slug only).
- No editing of the existing `techvia-blog-auto.json`.
- No live keyword-API integration (LLM-generated buyer-intent topics only).
- No multi-image / in-body images (one cover image per article).
