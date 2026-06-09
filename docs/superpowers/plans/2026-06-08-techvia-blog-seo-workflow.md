# Techvia SEO Blog Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a new n8n workflow `techvia-blog-seo` that publishes one human-sounding, SEO/buyer-intent blog post per day, with Postgres-backed topic dedup + full tracking and a single AI-generated cover image hosted on Cloudflare R2.

**Architecture:** Daily schedule → LLM proposes 5 buyer-intent topics → Postgres dedup over a 3-month window (with one round-2 fallback) → single reused OpenRouter text model writes the article → OpenRouter image model generates a cover → image uploaded to R2 (S3 API) and served from the bucket's public `r2.dev` URL → Hugo markdown committed to GitHub → row inserted into `published_topics` → Telegram notification. The existing `techvia-blog-auto.json` is left untouched as a fallback.

**Tech Stack:** n8n (localhost:5678), PostgreSQL (local, db `techvia`), Cloudflare R2 (S3-compatible), OpenRouter (text + image), GitHub REST API, Telegram Bot, Hugo/PaperMod.

**Reference design:** `docs/superpowers/specs/2026-06-08-techvia-blog-seo-workflow-design.md`

**Secrets — never commit these. Enter only into n8n credentials:**
- Postgres: host `localhost`, port `5432`, db `techvia`, user `techvia`, password `devpassword123`
- R2 S3: endpoint `https://212f5f7833067ea335ce10098a9f6a32.r2.cloudflarestorage.com`, bucket `techvia-blog-images`, access key `447d5b94715f7809ebf0e0901b5a47c9`, secret `468cb9222477ace491f7502bed2f5a6ae517886cb6aa1f191c530663f5d9fe15`
- R2 public URL (safe, already public): `https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev`
- OpenRouter API key: reuse existing n8n credential `DS7MeEgya97Jx9db`; for the HTTP image node you'll also create an httpHeaderAuth credential holding `Authorization: Bearer <openrouter-key>`.

---

## File Structure

- **Create** `db/published_topics.sql` — idempotent schema (table + index). Committed.
- **Create** `n8n/workflows/techvia-blog-seo.json` — the new workflow. Committed.
- **Create** `scripts/test-r2-upload.mjs` — throwaway R2 round-trip check (Node). Committed (harmless).
- **Create** `scripts/test-openrouter-image.mjs` — throwaway OpenRouter image response-shape probe. Committed.
- **Modify** none of the existing workflows.

---

## Phase 0 — De-risk the two unknowns (R2 upload, image response shape)

### Task 0a: Verify R2 upload + public read round-trip

**Files:**
- Create: `scripts/test-r2-upload.mjs`

- [ ] **Step 1: Write the probe script**

```js
// scripts/test-r2-upload.mjs
// Round-trips a tiny PNG to R2 and reads it back via the public URL.
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const ENDPOINT = "https://212f5f7833067ea335ce10098a9f6a32.r2.cloudflarestorage.com";
const BUCKET = "techvia-blog-images";
const PUBLIC = "https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev";
const KEY = "blog/__healthcheck.png";

const s3 = new S3Client({
  region: "auto",
  endpoint: ENDPOINT,
  forcePathStyle: true,
  credentials: {
    accessKeyId: process.env.R2_KEY,
    secretAccessKey: process.env.R2_SECRET,
  },
});

// 1x1 transparent PNG
const png = Buffer.from(
  "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==",
  "base64"
);

await s3.send(new PutObjectCommand({ Bucket: BUCKET, Key: KEY, Body: png, ContentType: "image/png" }));
console.log("uploaded ok");

const res = await fetch(`${PUBLIC}/${KEY}`);
console.log("public GET status:", res.status, "content-type:", res.headers.get("content-type"));
if (res.status !== 200) { console.error("PUBLIC READ FAILED"); process.exit(1); }
console.log("R2 round-trip OK");
```

- [ ] **Step 2: Install the one dependency (temporary) and run**

Run (PowerShell):
```powershell
npm i --no-save @aws-sdk/client-s3
$env:R2_KEY="447d5b94715f7809ebf0e0901b5a47c9"; $env:R2_SECRET="468cb9222477ace491f7502bed2f5a6ae517886cb6aa1f191c530663f5d9fe15"; node scripts/test-r2-upload.mjs
```
Expected output:
```
uploaded ok
public GET status: 200 content-type: image/png
R2 round-trip OK
```

If `public GET status` is **403/401**, the bucket's Public Development URL is not enabled — fix in Cloudflare → R2 → `techvia-blog-images` → Settings → Public Access before continuing. The S3 node config (endpoint/keys) is independently confirmed by the `uploaded ok` line.

- [ ] **Step 3: Commit**

```powershell
git add scripts/test-r2-upload.mjs; git commit -m "test: R2 upload + public read round-trip probe"
```

### Task 0b: Probe the OpenRouter image-generation response shape

**Files:**
- Create: `scripts/test-openrouter-image.mjs`

- [ ] **Step 1: Write the probe**

```js
// scripts/test-openrouter-image.mjs
// Confirms the model id works and prints WHERE the base64 image lives in the response.
const KEY = process.env.OPENROUTER_KEY;
const MODEL = "google/gemini-2.5-flash-image-preview";

const res = await fetch("https://openrouter.ai/api/v1/chat/completions", {
  method: "POST",
  headers: { "Authorization": `Bearer ${KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    model: MODEL,
    modalities: ["image", "text"],
    messages: [{ role: "user", content: "A clean, modern flat-illustration hero image about cloud migration for a software agency blog. No text in the image." }],
  }),
});
const data = await res.json();
console.log("http", res.status);
const msg = data?.choices?.[0]?.message;
console.log("has message.images:", Array.isArray(msg?.images), "count:", msg?.images?.length);
const url = msg?.images?.[0]?.image_url?.url;
console.log("first image url starts with:", url ? url.slice(0, 40) : "(none)");
if (!url) { console.error("NO IMAGE RETURNED — inspect full payload below"); console.error(JSON.stringify(data).slice(0, 1500)); process.exit(1); }
console.log("IMAGE OK — path is choices[0].message.images[0].image_url.url");
```

- [ ] **Step 2: Run**

Run (PowerShell):
```powershell
$env:OPENROUTER_KEY="<your-openrouter-key>"; node scripts/test-openrouter-image.mjs
```
Expected (the key line to confirm):
```
IMAGE OK — path is choices[0].message.images[0].image_url.url
```
If the path differs, **record the actual JSON path** — it is used verbatim in Task 5's "Extract Image" node. If the model id is rejected, try `google/gemini-2.0-flash-exp` or check OpenRouter's image-capable model list and update `$vars.IMAGE_MODEL` accordingly.

- [ ] **Step 3: Commit**

```powershell
git add scripts/test-openrouter-image.mjs; git commit -m "test: OpenRouter image response-shape probe"
```

---

## Phase 1 — Postgres schema

### Task 1: Create the `published_topics` table

**Files:**
- Create: `db/published_topics.sql`

- [ ] **Step 1: Write the schema file**

```sql
-- db/published_topics.sql
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

- [ ] **Step 2: Locate psql (not on PATH) and apply the schema**

Run (PowerShell) — finds the bundled psql and runs the file:
```powershell
$psql = (Get-ChildItem "C:\Program Files\PostgreSQL\*\bin\psql.exe" -ErrorAction SilentlyContinue | Select-Object -First 1).FullName
if (-not $psql) { Write-Error "psql not found under C:\Program Files\PostgreSQL — see fallback in Step 2b" } else {
  $env:PGPASSWORD="devpassword123"
  & $psql -h localhost -U techvia -d techvia -f db/published_topics.sql
}
```
Expected output:
```
CREATE TABLE
CREATE INDEX
```

- [ ] **Step 2b (fallback if psql not found): apply schema via n8n**

If Step 2 could not locate psql: complete Task 2 first (Postgres credential), then in the n8n UI add a temporary **Postgres** node, set operation **Execute Query**, paste the full SQL from Step 1, and click **Execute step**. Delete the temporary node afterward.

- [ ] **Step 3: Verify the table exists**

Run (PowerShell, psql path from Step 2):
```powershell
$env:PGPASSWORD="devpassword123"; & $psql -h localhost -U techvia -d techvia -c "\d published_topics"
```
Expected: a table description listing columns `id, slug, title, pillar, angle, seo_description, tags, post_path, image_url, published_at`.

- [ ] **Step 4: Commit**

```powershell
git add db/published_topics.sql; git commit -m "feat: published_topics schema for blog dedup + tracking"
```

---

## Phase 2 — n8n credentials (manual, secrets never committed)

### Task 2: Create the Postgres, R2, and OpenRouter-HTTP credentials in n8n

**Files:** none (n8n UI work).

- [ ] **Step 1: Postgres credential**

n8n → Credentials → New → **Postgres**. Name it `Techvia Postgres`.
- Host `localhost`, Database `techvia`, User `techvia`, Password `devpassword123`, Port `5432`, SSL `disable`.
Click **Test connection** → expect "Connection successful". Note the saved credential id.

- [ ] **Step 2: R2 (S3) credential**

n8n → Credentials → New → **S3** (the generic/MinIO-style S3, NOT "AWS S3"). Name it `Techvia R2`.
- Endpoint `https://212f5f7833067ea335ce10098a9f6a32.r2.cloudflarestorage.com`
- Region `auto`
- Access Key ID `447d5b94715f7809ebf0e0901b5a47c9`
- Secret Access Key `468cb9222477ace491f7502bed2f5a6ae517886cb6aa1f191c530663f5d9fe15`
- Force path style: **ON**
Note the saved credential id.

- [ ] **Step 3: OpenRouter HTTP header credential (for the image node)**

n8n → Credentials → New → **Header Auth**. Name it `OpenRouter HTTP`.
- Name: `Authorization`
- Value: `Bearer <your-openrouter-key>`
Note the saved credential id.

- [ ] **Step 4: Record the three new credential ids**

Write them into a scratch note (not committed). You will paste them into the workflow JSON in Task 3. The existing reusable ids are: OpenRouter `DS7MeEgya97Jx9db`, GitHub PAT `WpSYxNEHvb4nvnGh`, Telegram `QDbQhiQDyGyIbqse`.

---

## Phase 3 — Build the workflow JSON

### Task 3: Author `techvia-blog-seo.json`

**Files:**
- Create: `n8n/workflows/techvia-blog-seo.json`

- [ ] **Step 1: Write the complete workflow file**

Paste the JSON below. Replace the three `REPLACE_*` credential ids with the ids from Task 2. Leave the existing OpenRouter/GitHub/Telegram ids as-is.

```json
{
  "name": "techvia-blog-seo",
  "nodes": [
    {
      "id": "n-sched", "name": "Daily 8AM", "type": "n8n-nodes-base.scheduleTrigger", "typeVersion": 1.2, "position": [240, 400],
      "parameters": { "rule": { "interval": [{ "field": "cronExpression", "expression": "0 8 * * *" }] } }
    },
    {
      "id": "n-ctx", "name": "Build SEO Context", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [440, 400],
      "parameters": { "jsCode": "const pillars = ['Web Development','Cloud Solutions','Mobile Development','IT Consulting','SaaS Development','AI Development'];\nconst today = new Date().toISOString().split('T')[0];\nreturn [{ json: { today, pillars: pillars.join(', ') } }];" }
    },
    {
      "id": "n-cand", "name": "Topic Candidates", "type": "@n8n/n8n-nodes-langchain.chainLlm", "typeVersion": 1.4, "position": [640, 400],
      "parameters": { "promptType": "define", "text": "=You are an SEO content strategist for Techvia (techvia.software), a software agency offering: {{ $json.pillars }}.\n\nPropose 5 blog post topics with strong BUYER INTENT that would attract prospective customers searching Google and pull them toward hiring Techvia. Favor long-tail, commercial-intent angles: how-to, cost-of, best-X-for-Y, X-vs-Y, migration, checklist, mistakes-to-avoid.\n\nReturn ONLY valid JSON, no markdown fences:\n{\"candidates\":[{\"title\":\"...\",\"angle\":\"one-sentence hook\",\"pillar\":\"one of the listed pillars\"}]}\nExactly 5 candidates." }
    },
    {
      "id": "n-llm-text", "name": "LLM Text", "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter", "typeVersion": 1, "position": [640, 600],
      "parameters": { "model": "={{ $vars.WRITING_MODEL || 'anthropic/claude-sonnet-4-5' }}", "options": { "maxTokens": 3500 } },
      "credentials": { "openRouterApi": { "id": "DS7MeEgya97Jx9db", "name": "OpenRouter account" } }
    },
    {
      "id": "n-parse", "name": "Parse Candidates", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [840, 400],
      "parameters": { "jsCode": "const STOP = new Set(['a','an','the','and','or','but','of','to','in','on','for','with','at','by','from','as','is','are','your','you','how','what','why','best','vs','guide']);\nfunction slugify(t){ return t.toLowerCase().replace(/[^a-z0-9\\s-]/g,' ').split(/\\s+/).filter(w=>w && !STOP.has(w)).join('-').replace(/-+/g,'-').replace(/^-|-$/g,'').slice(0,80); }\nconst raw = $json.text;\nconst parsed = JSON.parse(raw.replace(/```json\\n?|```/g,'').trim());\nconst candidates = (parsed.candidates||[]).map((c,i)=>({ title:c.title, angle:c.angle, pillar:c.pillar, slug: slugify(c.title), rank: i }));\nreturn [{ json: { candidates } }];" }
    },
    {
      "id": "n-check", "name": "Check Recent Slugs", "type": "n8n-nodes-base.postgres", "typeVersion": 2.5, "position": [1040, 400],
      "parameters": { "operation": "executeQuery", "query": "SELECT slug FROM published_topics WHERE published_at > now() - interval '3 months';", "options": {} },
      "credentials": { "postgres": { "id": "REPLACE_PG_CRED_ID", "name": "Techvia Postgres" } }
    },
    {
      "id": "n-filter", "name": "Filter Fresh", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [1240, 400],
      "parameters": { "jsCode": "const recent = new Set($('Check Recent Slugs').all().map(i => i.json.slug));\nconst candidates = $('Parse Candidates').first().json.candidates;\nconst fresh = candidates.filter(c => !recent.has(c.slug)).sort((a,b)=>a.rank-b.rank);\nconst chosen = fresh[0] || null;\nreturn [{ json: { chosen, hasFresh: !!chosen, round: 1 } }];" }
    },
    {
      "id": "n-if1", "name": "Fresh Found?", "type": "n8n-nodes-base.if", "typeVersion": 2.2, "position": [1440, 400],
      "parameters": { "conditions": { "options": { "caseSensitive": true, "version": 2 }, "combinator": "and", "conditions": [ { "leftValue": "={{ $json.hasFresh }}", "rightValue": true, "operator": { "type": "boolean", "operation": "true", "singleValue": true } } ] } }
    },
    {
      "id": "n-cand2", "name": "Topic Candidates R2", "type": "@n8n/n8n-nodes-langchain.chainLlm", "typeVersion": 1.4, "position": [1440, 640],
      "parameters": { "promptType": "define", "text": "=You are an SEO content strategist for Techvia (techvia.software), offering: {{ $('Build SEO Context').first().json.pillars }}.\n\nPropose 5 NEW buyer-intent blog topics that are DIFFERENT from these already-considered titles: {{ $('Parse Candidates').first().json.candidates.map(c=>c.title).join('; ') }}.\n\nReturn ONLY valid JSON, no fences:\n{\"candidates\":[{\"title\":\"...\",\"angle\":\"...\",\"pillar\":\"...\"}]}\nExactly 5 candidates." }
    },
    {
      "id": "n-parse2", "name": "Parse Candidates R2", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [1640, 640],
      "parameters": { "jsCode": "const STOP = new Set(['a','an','the','and','or','but','of','to','in','on','for','with','at','by','from','as','is','are','your','you','how','what','why','best','vs','guide']);\nfunction slugify(t){ return t.toLowerCase().replace(/[^a-z0-9\\s-]/g,' ').split(/\\s+/).filter(w=>w && !STOP.has(w)).join('-').replace(/-+/g,'-').replace(/^-|-$/g,'').slice(0,80); }\nconst parsed = JSON.parse($json.text.replace(/```json\\n?|```/g,'').trim());\nconst candidates = (parsed.candidates||[]).map((c,i)=>({ title:c.title, angle:c.angle, pillar:c.pillar, slug: slugify(c.title), rank: i }));\nreturn [{ json: { candidates } }];" }
    },
    {
      "id": "n-filter2", "name": "Filter Fresh R2", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [1840, 640],
      "parameters": { "jsCode": "const recent = new Set($('Check Recent Slugs').all().map(i => i.json.slug));\nconst candidates = $('Parse Candidates R2').first().json.candidates;\nconst fresh = candidates.filter(c => !recent.has(c.slug)).sort((a,b)=>a.rank-b.rank);\nconst chosen = fresh[0] || null;\nreturn [{ json: { chosen, hasFresh: !!chosen, round: 2 } }];" }
    },
    {
      "id": "n-if2", "name": "Fresh R2?", "type": "n8n-nodes-base.if", "typeVersion": 2.2, "position": [2040, 640],
      "parameters": { "conditions": { "options": { "caseSensitive": true, "version": 2 }, "combinator": "and", "conditions": [ { "leftValue": "={{ $json.hasFresh }}", "rightValue": true, "operator": { "type": "boolean", "operation": "true", "singleValue": true } } ] } }
    },
    {
      "id": "n-skip", "name": "Notify Skip", "type": "n8n-nodes-base.telegram", "typeVersion": 1.2, "position": [2240, 760],
      "parameters": { "resource": "message", "operation": "sendMessage", "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}", "text": "=⏭️ <b>Blog skipped</b> — all candidate topics (2 rounds) were published in the last 3 months. No post today.", "additionalFields": { "parse_mode": "HTML" } },
      "credentials": { "telegramApi": { "id": "QDbQhiQDyGyIbqse", "name": "Telegram Bot" } }
    },
    {
      "id": "n-resolve", "name": "Resolve Chosen", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [2040, 360],
      "parameters": { "jsCode": "return [{ json: { chosen: $json.chosen } }];" }
    },
    {
      "id": "n-write", "name": "Write Article", "type": "@n8n/n8n-nodes-langchain.chainLlm", "typeVersion": 1.4, "position": [2240, 360],
      "parameters": { "promptType": "define", "text": "=You are a senior writer at Techvia (techvia.software). Write a blog post that reads like a real human practitioner wrote it — NOT like AI.\n\nTopic: {{ $json.chosen.title }}\nAngle: {{ $json.chosen.angle }}\nPillar: {{ $json.chosen.pillar }}\n\nVoice rules:\n- Vary sentence length; short punchy sentences are good; an occasional fragment is fine.\n- Team/first-person voice (\"we've shipped...\", \"in our projects...\"); concrete examples; mild opinion.\n- BANNED phrases: \"in today's fast-paced world\", \"delve\", \"moreover\", \"furthermore\", \"in conclusion\", \"unlock the power\", \"ever-evolving\", \"landscape\", \"navigating the\".\n- Prefer prose; at most one bulleted list; code blocks only where they genuinely help.\n- 900-1200 words, use ## and ### headings, end with a natural CTA to https://techvia.software.\n\nReturn ONLY valid JSON, no fences:\n{\"body_markdown\":\"the full article markdown body, no frontmatter\",\"seo_description\":\"under 150 chars\",\"tags\":[\"tag1\",\"tag2\",\"tag3\"],\"category\":\"the pillar\"}" }
    },
    {
      "id": "n-parse-art", "name": "Parse Article", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [2440, 360],
      "parameters": { "jsCode": "const art = JSON.parse($json.text.replace(/```json\\n?|```/g,'').trim());\nconst chosen = $('Resolve Chosen').first().json.chosen;\nreturn [{ json: { ...art, title: chosen.title, slug: chosen.slug, pillar: chosen.pillar, angle: chosen.angle } }];" }
    },
    {
      "id": "n-img", "name": "Generate Image", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [2640, 360],
      "parameters": {
        "method": "POST", "url": "https://openrouter.ai/api/v1/chat/completions",
        "authentication": "predefinedCredentialType", "nodeCredentialType": "httpHeaderAuth",
        "sendBody": true, "contentType": "json",
        "body": "={{ JSON.stringify({ model: ($vars.IMAGE_MODEL || 'google/gemini-2.5-flash-image-preview'), modalities: ['image','text'], messages: [{ role: 'user', content: 'A clean modern flat-illustration hero image for a software-agency blog post titled \"' + $json.title + '\". Theme: ' + $json.pillar + '. Professional, minimal, no text in the image.' }] }) }}",
        "options": {}
      },
      "credentials": { "httpHeaderAuth": { "id": "REPLACE_OPENROUTER_HTTP_CRED_ID", "name": "OpenRouter HTTP" } }
    },
    {
      "id": "n-img-extract", "name": "Extract Image", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [2840, 360],
      "parameters": { "jsCode": "const msg = $json.choices && $json.choices[0] && $json.choices[0].message;\nconst url = msg && msg.images && msg.images[0] && msg.images[0].image_url && msg.images[0].image_url.url;\nif (!url) { throw new Error('No image returned from OpenRouter: ' + JSON.stringify($json).slice(0,300)); }\nconst b64 = url.split(',')[1];\nconst art = $('Parse Article').first().json;\nconst today = new Date().toISOString().split('T')[0];\nconst key = `blog/${today}-${art.slug}.png`;\nreturn [{ json: { key, today }, binary: { data: { data: b64, mimeType: 'image/png', fileName: `${art.slug}.png` } } }];" }
    },
    {
      "id": "n-r2", "name": "Upload to R2", "type": "n8n-nodes-base.s3", "typeVersion": 1, "position": [3040, 360],
      "parameters": { "operation": "upload", "bucketName": "techvia-blog-images", "fileName": "={{ $json.key }}", "binaryData": true, "binaryPropertyName": "data", "additionalFields": { "acl": "publicRead" } },
      "credentials": { "s3": { "id": "REPLACE_R2_CRED_ID", "name": "Techvia R2" } }
    },
    {
      "id": "n-assemble", "name": "Assemble Markdown", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [3240, 360],
      "parameters": { "jsCode": "const art = $('Parse Article').first().json;\nconst today = $('Extract Image').first().json.today;\nconst key = $('Extract Image').first().json.key;\nconst imageUrl = `https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev/${key}`;\nconst filename = `${today}-${art.slug}.md`;\nconst esc = s => String(s||'').replace(/\"/g,'\\\\\"');\nconst frontmatter = [\n '---',\n `title: \"${esc(art.title)}\"`,\n `date: ${today}`,\n 'draft: false',\n `tags: ${JSON.stringify(art.tags||[])}`,\n `categories: [\"${esc(art.category||art.pillar)}\"]`,\n `description: \"${esc(art.seo_description)}\"`,\n 'cover:',\n `  image: \"${imageUrl}\"`,\n `  alt: \"${esc(art.title)}\"`,\n '  relative: false',\n 'showToc: true',\n '---',\n ''\n].join('\\n');\nreturn [{ json: { filename, markdown: frontmatter + art.body_markdown, slug: art.slug, title: art.title, pillar: art.pillar, angle: art.angle, seo_description: art.seo_description, tags: art.tags||[], post_path: `content/posts/${filename}`, image_url: imageUrl } }];" }
    },
    {
      "id": "n-b64", "name": "Base64 Encode", "type": "n8n-nodes-base.code", "typeVersion": 2, "position": [3440, 360],
      "parameters": { "jsCode": "return [{ json: { ...$json, content_b64: Buffer.from($json.markdown).toString('base64') } }];" }
    },
    {
      "id": "n-gh", "name": "Commit to GitHub", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [3640, 360],
      "parameters": {
        "method": "PUT", "url": "=https://api.github.com/repos/nikesh9220/techvia-blog/contents/content/posts/{{ $json.filename }}",
        "authentication": "predefinedCredentialType", "nodeCredentialType": "httpHeaderAuth",
        "sendHeaders": true, "headerParameters": { "parameters": [ { "name": "Accept", "value": "application/vnd.github+json" }, { "name": "X-GitHub-Api-Version", "value": "2022-11-28" } ] },
        "sendBody": true, "contentType": "json",
        "body": "={{ JSON.stringify({ message: 'feat: seo post \"' + $json.title + '\" via n8n', content: $json.content_b64, branch: 'master', committer: { name: 'Techvia Bot', email: 'bot@techvia.software' } }) }}",
        "options": {}
      },
      "credentials": { "httpHeaderAuth": { "id": "WpSYxNEHvb4nvnGh", "name": "GitHub PAT" } }
    },
    {
      "id": "n-insert", "name": "Insert Topic", "type": "n8n-nodes-base.postgres", "typeVersion": 2.5, "position": [3840, 360],
      "parameters": {
        "operation": "executeQuery",
        "query": "INSERT INTO published_topics (slug, title, pillar, angle, seo_description, tags, post_path, image_url) VALUES ($1,$2,$3,$4,$5,$6,$7,$8) ON CONFLICT (slug) DO NOTHING;",
        "options": { "queryReplacement": "={{ $('Assemble Markdown').first().json.slug }},={{ $('Assemble Markdown').first().json.title }},={{ $('Assemble Markdown').first().json.pillar }},={{ $('Assemble Markdown').first().json.angle }},={{ $('Assemble Markdown').first().json.seo_description }},={{ '{' + ($('Assemble Markdown').first().json.tags||[]).join(',') + '}' }},={{ $('Assemble Markdown').first().json.post_path }},={{ $('Assemble Markdown').first().json.image_url }}" }
      },
      "credentials": { "postgres": { "id": "REPLACE_PG_CRED_ID", "name": "Techvia Postgres" } }
    },
    {
      "id": "n-notify", "name": "Notify Success", "type": "n8n-nodes-base.telegram", "typeVersion": 1.2, "position": [4040, 360],
      "parameters": { "resource": "message", "operation": "sendMessage", "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}", "text": "=✅ <b>Published!</b>\n\n📝 {{ $('Assemble Markdown').first().json.title }}\n🔗 https://blogs.techvia.software/posts/{{ $('Assemble Markdown').first().json.slug }}/\n🖼️ {{ $('Assemble Markdown').first().json.image_url }}\n\nCloudflare deploying... 🚀", "additionalFields": { "parse_mode": "HTML" } },
      "credentials": { "telegramApi": { "id": "QDbQhiQDyGyIbqse", "name": "Telegram Bot" } }
    }
  ],
  "connections": {
    "Daily 8AM": { "main": [[{ "node": "Build SEO Context", "type": "main", "index": 0 }]] },
    "Build SEO Context": { "main": [[{ "node": "Topic Candidates", "type": "main", "index": 0 }]] },
    "LLM Text": { "ai_languageModel": [[{ "node": "Topic Candidates", "type": "ai_languageModel", "index": 0 }, { "node": "Topic Candidates R2", "type": "ai_languageModel", "index": 0 }, { "node": "Write Article", "type": "ai_languageModel", "index": 0 }]] },
    "Topic Candidates": { "main": [[{ "node": "Parse Candidates", "type": "main", "index": 0 }]] },
    "Parse Candidates": { "main": [[{ "node": "Check Recent Slugs", "type": "main", "index": 0 }]] },
    "Check Recent Slugs": { "main": [[{ "node": "Filter Fresh", "type": "main", "index": 0 }]] },
    "Filter Fresh": { "main": [[{ "node": "Fresh Found?", "type": "main", "index": 0 }]] },
    "Fresh Found?": { "main": [[{ "node": "Resolve Chosen", "type": "main", "index": 0 }], [{ "node": "Topic Candidates R2", "type": "main", "index": 0 }]] },
    "Topic Candidates R2": { "main": [[{ "node": "Parse Candidates R2", "type": "main", "index": 0 }]] },
    "Parse Candidates R2": { "main": [[{ "node": "Filter Fresh R2", "type": "main", "index": 0 }]] },
    "Filter Fresh R2": { "main": [[{ "node": "Fresh R2?", "type": "main", "index": 0 }]] },
    "Fresh R2?": { "main": [[{ "node": "Resolve Chosen", "type": "main", "index": 0 }], [{ "node": "Notify Skip", "type": "main", "index": 0 }]] },
    "Resolve Chosen": { "main": [[{ "node": "Write Article", "type": "main", "index": 0 }]] },
    "Write Article": { "main": [[{ "node": "Parse Article", "type": "main", "index": 0 }]] },
    "Parse Article": { "main": [[{ "node": "Generate Image", "type": "main", "index": 0 }]] },
    "Generate Image": { "main": [[{ "node": "Extract Image", "type": "main", "index": 0 }]] },
    "Extract Image": { "main": [[{ "node": "Upload to R2", "type": "main", "index": 0 }]] },
    "Upload to R2": { "main": [[{ "node": "Assemble Markdown", "type": "main", "index": 0 }]] },
    "Assemble Markdown": { "main": [[{ "node": "Base64 Encode", "type": "main", "index": 0 }]] },
    "Base64 Encode": { "main": [[{ "node": "Commit to GitHub", "type": "main", "index": 0 }]] },
    "Commit to GitHub": { "main": [[{ "node": "Insert Topic", "type": "main", "index": 0 }]] },
    "Insert Topic": { "main": [[{ "node": "Notify Success", "type": "main", "index": 0 }]] }
  },
  "settings": { "executionOrder": "v1", "saveManualExecutions": true, "callerPolicy": "workflowsFromSameOwner", "errorWorkflow": "" },
  "staticData": null
}
```

- [ ] **Step 2: Validate the JSON parses**

Run (PowerShell):
```powershell
node -e "JSON.parse(require('fs').readFileSync('n8n/workflows/techvia-blog-seo.json','utf8')); console.log('valid json')"
```
Expected: `valid json`

- [ ] **Step 3: Commit**

```powershell
git add n8n/workflows/techvia-blog-seo.json; git commit -m "feat: techvia-blog-seo n8n workflow (postgres dedup + R2 image)"
```

---

## Phase 4 — Import & dry-run in n8n

### Task 4: Import the workflow and confirm credentials resolve

**Files:** none (n8n UI).

- [ ] **Step 1: Import**

n8n → Workflows → **Import from File** → select `n8n/workflows/techvia-blog-seo.json`. Open it.

- [ ] **Step 2: Confirm every credential is bound**

Check these nodes show a green/linked credential (re-select from dropdown if any show "select credential"): `LLM Text` (OpenRouter), `Check Recent Slugs` + `Insert Topic` (Techvia Postgres), `Generate Image` (OpenRouter HTTP), `Upload to R2` (Techvia R2), `Commit to GitHub` (GitHub PAT), `Notify Skip` + `Notify Success` (Telegram Bot).

- [ ] **Step 3: Confirm n8n variables exist**

n8n → Variables. Ensure `TELEGRAM_CHAT_ID` is set (used by the old workflow already). Optionally set `WRITING_MODEL` and `IMAGE_MODEL`; otherwise the defaults in the JSON apply.

### Task 5: Section dry-runs (pin failures early)

**Files:** none (n8n UI — use "Execute step"/partial execution).

- [ ] **Step 1: Topic + dedup section**

Pin `Daily 8AM`, then execute up to `Filter Fresh`. Open `Filter Fresh` output: expect `chosen` = an object with `title/angle/pillar/slug` and `hasFresh: true` (table is empty on first run, so round 1 should always find fresh).

- [ ] **Step 2: Writer section**

Execute through `Parse Article`. Expect output keys `body_markdown` (long markdown string, 900-1200 words), `seo_description` (<150 chars), `tags` (array), `title`, `slug`, `pillar`. If `JSON.parse` throws, the writer emitted prose around the JSON — tighten the prompt's "Return ONLY valid JSON" line or lower the model temperature.

- [ ] **Step 3: Image + R2 section**

Execute through `Upload to R2`. Expect `Extract Image` to produce a binary `data` property and a `key` like `blog/2026-06-09-<slug>.png`. Expect `Upload to R2` to succeed. Then open the public URL `https://pub-c642e0958d1943cda1efc53d0e9bb68d.r2.dev/blog/<date>-<slug>.png` in a browser → the image loads. If `Extract Image` throws "No image returned", re-check the JSON path from Task 0b and update the node.

---

## Phase 5 — End-to-end + dedup verification

### Task 6: Full run, verify all five outputs

**Files:** none (n8n UI + git/psql verification).

- [ ] **Step 1: Execute the whole workflow once**

Click **Execute Workflow**. Expect it to reach `Notify Success` with no red nodes.

- [ ] **Step 2: Verify the GitHub commit**

Run (PowerShell):
```powershell
git fetch origin master; git -c core.pager=cat log origin/master --oneline -1
```
Expected: a commit `feat: seo post "<title>" via n8n`. Confirm a new file exists under `content/posts/`.

- [ ] **Step 3: Verify the frontmatter + image link**

Pull the new post and confirm the frontmatter has a `cover.image` pointing at `pub-c642...r2.dev` and that the URL returns 200 (open in browser). Confirm the body reads naturally and contains none of the banned AI-tell phrases.

- [ ] **Step 4: Verify the Postgres row**

Run (PowerShell, psql path from Task 1):
```powershell
$env:PGPASSWORD="devpassword123"; & $psql -h localhost -U techvia -d techvia -c "SELECT slug, title, pillar, image_url, published_at FROM published_topics ORDER BY published_at DESC LIMIT 3;"
```
Expected: one row matching the just-published post, with `image_url` set.

- [ ] **Step 5: Verify the Telegram notification**

Confirm the "✅ Published!" message arrived with title, post URL, and image URL.

### Task 7: Dedup re-run test

- [ ] **Step 1: Force a duplicate**

Temporarily edit the `Topic Candidates` prompt to return a fixed candidate set whose top title matches the slug already in `published_topics` (or simply re-run several times and confirm previously-published slugs are never re-selected). The deterministic check: a slug present in `published_topics` must never appear as `chosen`.

- [ ] **Step 2: Confirm round-2 fallback**

If all round-1 candidates are already published, confirm execution flows `Fresh Found?`(false) → `Topic Candidates R2` → `Filter Fresh R2`, and either publishes a fresh round-2 topic or hits `Notify Skip` (sending the "⏭️ Blog skipped" message) — and that **no** GitHub commit and **no** Postgres insert happen on a skip.

- [ ] **Step 3: Confirm idempotent insert**

Run the same slug insert path twice (re-execute from `Insert Topic` with a pinned slug). Verify the row count for that slug stays 1 (the `ON CONFLICT (slug) DO NOTHING` guard holds):
```powershell
$env:PGPASSWORD="devpassword123"; & $psql -h localhost -U techvia -d techvia -c "SELECT slug, count(*) FROM published_topics GROUP BY slug HAVING count(*) > 1;"
```
Expected: **0 rows** (no duplicates).

- [ ] **Step 4: Revert any temporary prompt edits and re-commit the clean workflow if changed.**

---

## Phase 6 — Activate

### Task 8: Activate the schedule

- [ ] **Step 1:** In n8n, toggle the `techvia-blog-seo` workflow **Active**. The `Daily 8AM` (cron `0 8 * * *`, server timezone) trigger now fires daily.
- [ ] **Step 2:** Decide on the old workflow: leave `techvia-blog-auto` **inactive** (fallback) so the two don't both publish the same day. Confirm only `techvia-blog-seo` is active.
- [ ] **Step 3 (optional):** Re-export the final workflow from n8n and overwrite `n8n/workflows/techvia-blog-seo.json` so the committed file matches the live one (captures any in-UI fixes), then commit.

---

## Notes / known risks captured during planning

- **OpenRouter image model:** `google/gemini-2.5-flash-image-preview` and the response path `choices[0].message.images[0].image_url.url` are validated in Task 0b — if either differs, Task 0b tells you the real value to put in `Generate Image` / `Extract Image`.
- **R2 public access** must be enabled (Task 0a). The S3 endpoint is upload-only; serving happens via the `pub-….r2.dev` URL.
- **n8n S3 node + R2:** uses the generic `S3` credential type with a custom endpoint and force-path-style ON. If `acl: publicRead` is rejected by R2, drop the `additionalFields.acl` — R2 ignores ACLs and serves via the bucket's public setting instead.
- **Timezone:** cron runs in the n8n instance timezone. If 8AM must be a specific zone, set the workflow's timezone in Settings.
- **Writer JSON reliability:** if the model occasionally wraps JSON in prose, the `Parse Article`/`Parse Candidates` fence-stripping handles ```` ```json ```` but not free prose — keep the "Return ONLY valid JSON" instruction and consider a low temperature.
```
