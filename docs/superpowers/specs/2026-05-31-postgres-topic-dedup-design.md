# Design: Postgres Topic Dedup for Blog Automation

**Date:** 2026-05-31
**Status:** Approved (pending spec review)
**Scope:** New n8n workflow `techvia-blog-advance` that prevents republishing topics covered within the last 3 months, backed by a separate PostgreSQL database.

## Problem

The current daily automation (`techvia-blog-auto`) picks one trending topic per day via an LLM and publishes it with no memory of what was already covered. Topics can repeat. We want the pipeline to remember published topics and avoid re-covering anything published in the last 3 months.

## Decisions

| Question | Decision |
|----------|----------|
| Database | Separate Postgres instance already exists; credential supplied in n8n by the user. |
| Dedup match key | **Normalized slug** (lowercase, strip punctuation, drop stopwords, hyphenate). Deterministic, no extra LLM/embedding cost. |
| "Pick another topic" | **Multi-candidate + filter**: LLM returns 5 ranked candidates; one Postgres query removes recently-published slugs; take the highest-ranked survivor. No loops. |
| All candidates recent | **Request more candidates once** (round 2, excluding round-1 titles); if still all covered, skip the day and notify via Telegram. |
| Insert timing | **After a successful GitHub commit** (publish), so failed runs never poison the table. |
| Idempotency | `INSERT ... ON CONFLICT (slug) DO NOTHING`. |
| Delivery | New file `n8n/workflows/techvia-blog-advance.json` (name `techvia-blog-advance`). Existing `techvia-blog-auto.json` left untouched as fallback. |

## Database Schema

Created once in the existing Postgres instance:

```sql
CREATE TABLE IF NOT EXISTS published_topics (
  id           BIGSERIAL PRIMARY KEY,
  slug         TEXT UNIQUE NOT NULL,        -- normalized dedup key
  title        TEXT NOT NULL,
  pillar       TEXT,
  angle        TEXT,
  post_path    TEXT,                        -- content/posts/YYYY-MM-DD-slug.md
  published_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_published_topics_pub_at ON published_topics (published_at);
```

The 3-month window is enforced in SQL: `published_at > now() - interval '3 months'`.

## Slug Normalization

A single shared JS helper, used in both candidate parsing and (implicitly, via the chosen candidate) the insert, so the stored slug always equals the dedup key:

```js
function normalizeSlug(title) {
  const stop = new Set(['the','a','an','and','or','of','to','for','in','on','with','how','why','what','is','are','your','you']);
  return title
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')      // strip punctuation
    .split(/\s+/)
    .filter(w => w && !stop.has(w))    // drop stopwords
    .join('-')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '');
}
```

## Workflow Flow

```
Daily 8AM → HN/TechCrunch/DevTo Feeds → Merge RSS → Flatten News
  → Pick Topics (LLM, 5 ranked candidates as JSON array)
  → Parse Candidates (code: parse array, compute slug for each, attach rank)
  → Check Recent Slugs (Postgres: which candidate slugs are recent?)
  → Filter Fresh (code: drop recent, keep top-ranked survivor or empty)
  → Any Fresh? (IF)
      ├─ yes → Write Article → Extract Metadata → Assemble Markdown → Base64 Encode
      │         → Commit to GitHub
      │         → Insert Topic (Postgres INSERT, ON CONFLICT DO NOTHING)
      │         → Telegram Notify (success)
      └─ no  → Pick Topics R2 (LLM, exclude round-1 titles)
                → Parse Candidates R2 → Check Recent Slugs R2 → Filter Fresh R2
                → Any Fresh R2? (IF)
                    ├─ yes → (rejoins Write Article path)
                    └─ no  → Notify Skip (Telegram: all candidates covered)
```

### Node-by-node changes vs. `techvia-blog-auto`

| Node | Change |
|------|--------|
| **Pick Topics** | Prompt changed: return a JSON **array of 5** ranked candidates (`[{title, angle, pillar}, ...]`) instead of one object. |
| **Parse Candidates** *(new)* | Parse the array, compute `slug` per candidate via `normalizeSlug`, attach `rank` (array order). Output one item per candidate. |
| **Check Recent Slugs** *(new)* | Postgres *Execute Query*: `SELECT slug FROM published_topics WHERE slug = ANY($1) AND published_at > now() - interval '3 months'`. Parameter `$1` = array of candidate slugs. Returns colliding slugs only. |
| **Filter Fresh** *(new)* | Code: remove candidates whose slug is in the collision set; emit the single highest-ranked survivor, or an empty/flagged item if none. |
| **Any Fresh?** *(new IF)* | Branch on whether a survivor exists. |
| **Pick Topics R2 / Parse R2 / Check R2 / Filter R2 / Any Fresh R2?** *(new)* | Second round: same chain, prompt excludes round-1 candidate titles. |
| **Insert Topic** *(new)* | Postgres *Insert* after `Commit to GitHub`: `slug, title, pillar, angle, post_path`. `ON CONFLICT (slug) DO NOTHING`. `published_at` defaults to `now()`. |
| **Notify Skip** *(new)* | Telegram message when round 2 is also fully covered. |
| Write Article → Telegram Notify | Unchanged downstream of topic selection, except the new Insert Topic node inserted between Commit and Notify. |

## Error Handling

- **Postgres unreachable on Check**: query fails → run errors out and Telegram approval/handler surfaces nothing new; treated as a normal n8n run failure (visible in n8n executions). No silent publish without a dedup check.
- **Postgres unreachable on Insert**: post is already committed to GitHub; insert failure is logged by n8n. Because the next run re-queries by slug, a missed insert means the topic *could* be re-picked — acceptable risk, surfaced in n8n execution log. (Future hardening: retry on the Insert node.)
- **Malformed LLM output**: Parse Candidates wraps `JSON.parse` defensively; on parse failure the run errors rather than publishing garbage.
- **`ON CONFLICT DO NOTHING`** makes re-runs and the rare double-publish safe at the DB layer.

## Out of Scope

- Backfilling `published_topics` with the existing 7 posts (optional one-time SQL; can be added later).
- Semantic/embedding-based dedup (explicitly deferred; slug-based chosen).
- Changes to `techvia-blog-auto.json` (left as fallback).
- Cover-image and description-length fixes from the earlier SEO review (separate task).

## Testing / Verification

- Insert a known slug with `published_at = now()`; confirm a run with that topic in the candidate set skips it and picks the next survivor.
- Insert a slug with `published_at = now() - interval '4 months'`; confirm it is NOT skipped (outside window).
- Seed all 5 round-1 candidate slugs as recent; confirm round 2 fires.
- Confirm a successful publish writes exactly one new row with the correct normalized slug and `post_path`.
