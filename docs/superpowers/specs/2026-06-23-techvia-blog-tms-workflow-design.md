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
`techvia-blog-pipeline.json` ("techvia-daily-blog") but replaces the trend/RSS topic brain with
an LLM freight-topic generator that ranks and auto-selects by SEO, and rewrites the author voice
for freight operations. It runs **fully autonomously** (no human approval) — see §2.

The existing `techvia-blog-pipeline.json` and `techvia-blog-auto.json` are **left untouched**.

## 2. Decisions (from brainstorming)

| Decision | Choice |
|---|---|
| Relationship to existing workflows | **New parallel workflow**, others untouched |
| Topic engine | **LLM-generated** buyer-intent TMS topics daily (seeded with priority keywords) |
| Dedup | **Deferred to v2** — no database in v1 (Postgres connectivity pending); topics are LLM-generated fresh each run |
| Approval | **Autonomous** — AI ranks candidates by SEO and auto-picks the best. Telegram approval removed: the inbound callback/webhook resume can't work while n8n runs in **local Docker**. Re-add the gate once n8n is on a server. |
| Writer model | `$vars.WRITING_MODEL` default **`anthropic/claude-sonnet-4`** (via OpenRouter) |
| Cover images | **None in v1** (text-only); R2 covers deferred to v2 |
| CTA target | `https://www.techvia.software/products/tms` |
| Cadence | Daily, 07:00 UTC (staggered after the 06:00 general publisher) |

## 3. High-level flow

```
Daily 7AM (cron 0 7 * * *)
  → Build TMS Context (code: freight framing + seed keywords)
  → LLM #1  Generate Candidates  (OpenRouter → JSON array of 5, RANKED best-first by SEO)
  → Parse Candidates (code: parse JSON, deterministic slugs, preserve rank order)
  → Select Best Topic (code: auto-pick candidate[0]; build writer prompt; throw if none)
  → LLM #2  Write Article  (freight expert, human voice, CTA → /products/tms)
  → Extract Metadata (LLM → JSON: seo_description, tags, category)
  → Assemble Markdown (code: Hugo frontmatter + body; filename {date}-{slug}.md)
  → Base64 Encode → Commit to GitHub (content/posts/{date}-{slug}.md, branch master)
  → Notify Success (Telegram: title + live URL)
```

**Autonomous, no human gate.** The AI ranks its 5 candidates best-first on SEO opportunity and
the workflow auto-selects #1 — there is no Telegram approval step, because the inbound callback
resume cannot reach n8n while it runs in local Docker. `Notify Success` is a plain outbound
`sendMessage` (works fine from Docker). When n8n moves to a server, the
`Send Approval → Wait → Skipped?` gate (and the shared `techvia-telegram-handler.json`) can be
reinserted before `Write Article`.

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
- The 4 seed keywords are eligible candidates themselves on early runs; the prompt pushes the
  model toward fresh long-tail siblings on later runs. (v2 dedup, §6, will enforce this.)

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

## 6. Deduplication — deferred to v2

**v1 ships without a database.** Topic novelty relies on the LLM generating fresh buyer-intent
candidates each run, ranked best-first on SEO; with no human gate, novelty rests on that
ranking plus the breadth of the freight keyword space. `Parse Candidates` still produces a deterministic `topic_slug` (lowercase → strip
punctuation → drop stopwords → hyphenate) which is used for both the filename and the chosen
topic, so when Postgres dedup is added in v2 the key is already stable.

**v2 (when the Postgres credential connects):** reintroduce a `Check Recent Slugs` SELECT before
approval and an `Insert Published Topic` after the GitHub commit, against the `published_topics`
table from `2026-06-08-techvia-blog-seo-workflow-design.md` (3-month window, `ON CONFLICT (slug)
DO NOTHING`). For Dockerised n8n the credential host is `host.docker.internal`, not `localhost`.

## 7. Metadata + frontmatter

`Extract Metadata` (LLM) returns `{ seo_description (<150 chars), tags[], category }`
where `category` ∈ `Freight & Logistics | Dispatch | TMS | Trucking | Freight Tech`. The slug
comes from `Resolve Topic` (deterministic), not from the metadata LLM.
`Assemble Markdown` emits the exact Hugo frontmatter from the blog CLAUDE.md (title, date,
draft:false, tags, categories, description, showToc:true) — **no `cover.image` in v1**.
Filename: `content/posts/{YYYY-MM-DD}-{slug}.md`.

## 8. Error handling

- **Malformed LLM JSON →** defensive `JSON.parse` in code nodes (strip ```` ```json ```` fences).
- **No candidates →** `Has Fresh?` false branch → Telegram notice, clean stop, no commit.
- **GitHub commit failure →** surfaces in execution log + Telegram error; no success notice.

## 9. Credentials required in n8n (all already exist)

| Credential | n8n id (existing) | Use |
|---|---|---|
| OpenRouter | `DS7MeEgya97Jx9db` | candidates / writer / metadata |
| GitHub PAT | `WpSYxNEHvb4nvnGh` | commit post (header auth) |
| Telegram Bot | `QDbQhiQDyGyIbqse` | success notification (outbound only) |

Workflow variables reused: `$vars.TELEGRAM_CHAT_ID`, `$vars.WRITING_MODEL`
(default `anthropic/claude-sonnet-4`). No database credential in v1.

## 10. Deliverables

1. `n8n/workflows/techvia-blog-tms.json` — the new workflow (import into n8n).
2. One end-to-end test: trigger manually → AI picks top SEO topic → post commits → appears on
   blog with a working `/products/tms` CTA → Telegram success notice arrives.

## 11. Out of scope / YAGNI (v1)

- No database / dedup (deferred to v2; see §6).
- No cover images / R2 (deferred to v2; SEO design already specs the mechanism).
- No edits to `techvia-blog-pipeline.json` or `techvia-blog-auto.json`.
- No live keyword-API integration (LLM-generated buyer-intent topics only).
