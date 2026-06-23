# Design: TMS SEO Blog Workflow (`techvia-blog-tms`)

**Date:** 2026-06-23
**Status:** Approved design → pending implementation plan
**Author:** Nikesh Pandya (with Claude Code)

> **Secrets policy:** No credentials (OpenRouter key, GitHub PAT, Telegram token, Postgres
> password) are stored in this document or anywhere in git. They live **only** in n8n
> credentials, configured at build time.

## 1. Purpose

Add a **new, parallel** daily workflow that publishes **freight/TMS buyer-intent SEO articles**
to the Hugo blog and funnels readers to the Techvia TMS product page
(`https://www.techvia.software/products/tms`). It clones the structure of the existing
`techvia-blog-pipeline.json` ("techvia-daily-blog") — including the Telegram approval gate —
but replaces the trend/RSS topic brain with an LLM freight-topic generator, adds Postgres
dedup, and rewrites the author voice for freight operations.

The existing `techvia-blog-pipeline.json` and `techvia-blog-auto.json` are **left untouched**.

## 2. Decisions (from brainstorming)

| Decision | Choice |
|---|---|
| Relationship to existing workflows | **New parallel workflow**, others untouched |
| Topic engine | **LLM-generated** buyer-intent TMS topics daily (seeded with priority keywords) |
| Dedup | **Postgres** `published_topics`, 3-month window (reuse SEO-design table) |
| Approval | **Keep Telegram approval gate** — operator taps 1 / 2 / 3 / ⏭ Skip |
| Cover images | **None in v1** (text-only); R2 covers deferred to v2 |
| CTA target | `https://www.techvia.software/products/tms` |
| Cadence | Daily, 07:00 UTC (staggered after the 06:00 general publisher) |

## 3. High-level flow

```
Daily 7AM (cron 0 7 * * *)
  → Build TMS Context (code: freight pillars + buyer-intent framing + seed keywords)
  → LLM #1  Generate Candidates  (OpenRouter text model → JSON array of 5)
  → Parse Candidates (code: parse JSON, deterministic slugs, build approval text)
  → Postgres  Check Recent Slugs  (SELECT slug WHERE published_at > now() - interval '3 months')
  → Filter Fresh (code: drop already-published, keep top 3 survivors)
       ├─ none fresh → Telegram "all recent, skipped today" (stop)
       └─ fresh found ↓
  → Send Approval Message (Telegram: top 3 fresh, inline buttons 1/2/3/Skip)
  → Store Resume URL (staticData) → Wait for Approval (webhook resume)
  → Skipped?
       ├─ true  → Notify Skip (Telegram) (stop)
       └─ false ↓ (operator picked a topic; topic_title/topic_angle/topic_slug resolved)
  → LLM #2  Write Article  (freight expert, human voice, CTA → /products/tms)
  → Extract Metadata (LLM → JSON: slug, seo_description, tags, category)
  → Assemble Markdown (code: Hugo frontmatter + body; filename {date}-{slug}.md)
  → Base64 Encode → Commit to GitHub (content/posts/{date}-{slug}.md, branch master)
  → Postgres Insert published_topics (ON CONFLICT (slug) DO NOTHING)
  → Notify Success (Telegram: title + live URL)
```

The approval-callback plumbing (inline button → webhook resume with `isSkip` /
`topic_title` / `topic_angle` / `topic_slug`) is handled by the existing
`techvia-telegram-handler.json`, exactly as for the current pipeline.

## 4. Topic engine (LLM #1) — buyer-intent freight/TMS

System prompt frames the model as a **freight-tech content strategist** for Techvia TMS (a TMS
for small-to-medium US freight brokers and carriers). It generates **commercial buyer-intent**
topics — the questions a broker, dispatcher, or carrier types into Google when they have a
problem a TMS solves — using long-tail angles (best X for Y / X vs Y / cost of / how to /
checklist / "… explained").

Seed/priority keywords pinned in the prompt (anchor the model and guarantee coverage):
- Best TMS for small trucking companies
- Broker dispatch workflow
- EDI 204 explained
- How freight brokers automate dispatch

The model expands around these with siblings such as: *TMS vs spreadsheets for brokers,
carrier onboarding checklist, what is a rate confirmation (RateCon), how to reduce dispatcher
workload, freight invoicing & settlement basics, driver settlement / owner-operator pay,
load lifecycle from tender to cash, choosing a TMS — buyer's checklist.*

- **Output:** JSON array of 5 ranked candidates, each `{ title, angle, slug }`.
- `slug` deterministic (see §6) so the dedup key is stable across runs.
- The 4 seed keywords are eligible candidates themselves on early runs; dedup then rotates
  the workflow onto fresh long-tails over time.

## 5. Writer (LLM #2) — human freight voice + product funnel

Writer prompt enforces a human, freight-operator voice (same anti-AI-tell discipline as the
SEO design):

- Vary sentence length; first-person team voice ("we built Techvia TMS because…", "dispatchers
  we talk to…"); concrete freight detail (stops, RateCons, PODs, deadhead, detention).
- **Ban AI-tell phrases:** "in today's fast-paced world", "delve", "moreover", "furthermore",
  "in conclusion", "unlock the power", "ever-evolving", "landscape", "navigating the".
- 900–1200 words, `##`/`###` headings, at most one bulleted list.
- **Product funnel:** weave **Techvia TMS** in where it genuinely answers the article's problem
  (e.g. a dispatch-automation post references the drag-and-drop dispatch board; an EDI-204 post
  references EDI tenders auto-becoming loads), and **close with a natural CTA** linking to
  `https://www.techvia.software/products/tms`. No hard-sell stuffing — one in-body mention +
  one closing CTA is the target.
- Returns the article body in markdown (frontmatter is added later in Assemble).

## 6. Postgres dedup + tracking

Reuses the `published_topics` table from `2026-06-08-techvia-blog-seo-workflow-design.md`
(host `localhost:5432`, db `techvia`, user `techvia`; password in n8n credential only). No
schema change required — TMS posts share the table; global slug dedup also prevents collisions
with the SEO workflow if/when it ships.

```sql
CREATE TABLE IF NOT EXISTS published_topics (
  id              SERIAL PRIMARY KEY,
  slug            TEXT UNIQUE NOT NULL,
  title           TEXT NOT NULL,
  pillar          TEXT,
  angle           TEXT,
  seo_description TEXT,
  tags            TEXT[],
  post_path       TEXT,
  image_url       TEXT,
  published_at    TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_published_topics_published_at
  ON published_topics (published_at);
```

- For TMS posts, `pillar` is set to `"TMS"` so the vertical is queryable later.
- **Slug normalization (deterministic):** lowercase → strip punctuation → remove stopwords →
  collapse whitespace → hyphenate. Same function for candidate slugs and the dedup key.
- **Dedup window:** 3 months. **Single-round (v1):** if no fresh candidate survives →
  Telegram "skipped, all candidates recent", no commit. (A two-round regenerate-and-retry is
  deferred to v2 — the 5-candidate × 3-month window rarely exhausts.)
- **Insert timing:** AFTER successful GitHub commit, `ON CONFLICT (slug) DO NOTHING`.
- **Committed slug == dedup slug:** the deterministic `topic_slug` chosen at approval is used
  for both the filename and the Postgres insert, so dedup never drifts from what was published.
  The metadata LLM supplies only `seo_description`, `tags`, and `category` — never the slug.

## 7. Metadata + frontmatter

`Extract Metadata` (LLM) returns `{ slug, seo_description (<150 chars), tags[], category }`
where `category` ∈ `Freight & Logistics | Dispatch | TMS | Trucking | Freight Tech`.
`Assemble Markdown` emits the exact Hugo frontmatter from the blog CLAUDE.md (title, date,
draft:false, tags, categories, description, showToc:true) — **no `cover.image` in v1**.
Filename: `content/posts/{YYYY-MM-DD}-{slug}.md`.

## 8. Error handling

- **Postgres SELECT failure → block publish** (never risk a duplicate).
- **Postgres INSERT failure → log + continue** (commit already happened; tracking best-effort).
- **Malformed LLM JSON →** defensive `JSON.parse` in code nodes (strip ```` ```json ```` fences).
- **No candidates after 2 rounds →** Telegram notice, clean stop, no commit.
- **GitHub commit failure →** surfaces in execution log + Telegram error; no Postgres insert.

## 9. Credentials required in n8n (all already exist)

| Credential | n8n id (existing) | Use |
|---|---|---|
| OpenRouter | `DS7MeEgya97Jx9db` | LLM #1 / #1b / #2 / metadata |
| GitHub PAT | `WpSYxNEHvb4nvnGh` | commit post |
| Telegram Bot | `QDbQhiQDyGyIbqse` | approval + notifications |
| Postgres (local) | from SEO workflow | dedup + tracking |

Workflow variables reused: `$vars.TELEGRAM_CHAT_ID`, `$vars.WRITING_MODEL`
(default `anthropic/claude-sonnet-4-5`).

## 10. Deliverables

1. `n8n/workflows/techvia-blog-tms.json` — the new workflow (import into n8n).
2. SQL note — reuse existing `published_topics` (no new table) — included in §6.
3. One end-to-end test: topics arrive in Telegram → approve → post commits → appears on blog
   with a working `/products/tms` CTA → row inserted → re-run does not repeat the topic.

## 11. Out of scope / YAGNI (v1)

- No cover images / R2 (deferred to v2; SEO design already specs the mechanism).
- No edits to `techvia-blog-pipeline.json` or `techvia-blog-auto.json`.
- No live keyword-API integration (LLM-generated buyer-intent topics only).
- No new Postgres columns (reuse the existing table).
