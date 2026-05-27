# Implementation Plan: n8n Blog Automation Workflow

**Design spec:** `docs/superpowers/specs/2026-05-27-n8n-blog-automation-design.md`  
**Repo:** `nikesh9220/techvia-blog` (branch: `master`)  
**n8n instance:** `http://localhost:5678`  
**Date:** 2026-05-27

---

## Phase 0 — Documentation Discovery (COMPLETE)

All API signatures confirmed. Key findings:

### Allowed APIs / Confirmed Signatures

| Service | Endpoint | Auth |
|---------|---------|------|
| n8n REST API | `POST http://localhost:5678/api/v1/workflows` | `X-N8N-API-KEY` header |
| OpenRouter | `POST https://openrouter.ai/api/v1/chat/completions` | `Authorization: Bearer KEY` |
| Serper.dev | `POST https://google.serper.dev/news` | `X-API-KEY` header |
| Gemini Flash | `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=KEY` | Query param |
| Unsplash | `GET https://api.unsplash.com/search/photos?query=...&per_page=5` | `Authorization: Client-ID KEY` |
| Cloudinary | `POST https://api.cloudinary.com/v1_1/techvia/image/upload` | Unsigned preset in body |
| GitHub Contents | `PUT https://api.github.com/repos/nikesh9220/techvia-blog/contents/{path}` | `Authorization: Bearer PAT` |
| Telegram sendMessage | `POST https://api.telegram.org/bot{TOKEN}/sendMessage` | Token in URL |
| Telegram answerCallback | `POST https://api.telegram.org/bot{TOKEN}/answerCallbackQuery` | Token in URL |

### Key n8n Node Types
```
n8n-nodes-base.scheduleTrigger   (Schedule Trigger)
n8n-nodes-base.httpRequest       (HTTP Request)
n8n-nodes-base.code              (Code / JavaScript)
n8n-nodes-base.telegram          (Telegram send)
n8n-nodes-base.telegramTrigger   (Telegram receive)
n8n-nodes-base.merge             (Merge parallel branches)
n8n-nodes-base.wait              (Pause execution, resume via webhook)
n8n-nodes-base.set               (Set fields)
n8n-nodes-base.github            (GitHub file operations)
```

### Anti-Patterns to Avoid
- Do NOT use a single workflow to both send a Telegram message and wait for reply — requires the Wait node + a SEPARATE Telegram Trigger workflow
- Do NOT use Gemini for content writing (different request format than OpenRouter) — use OpenRouter for all LLM calls except the image query generation step
- Do NOT try to fit the resume URL into `callback_data` (64-byte limit) — store it in n8n workflow static data keyed by `message_id`
- GitHub Contents API requires base64-encoded content — always use a Code node to encode before the PUT request

---

## Phase 1 — n8n MCP Installation & Credential Setup

**Goal:** Claude Code can interact with n8n programmatically. All credentials exist in n8n.

### Step 1.1 — Install n8n MCP Server

Check Smithery for the n8n MCP package and add to Claude Code settings:

```bash
# Check available n8n MCP packages
# Try in order until one works:
npx -y @illuminaresolutions/n8n-mcp --version
npx -y n8n-mcp-server --version
```

Add to `~/.claude/settings.json` under `mcpServers`:
```json
{
  "mcpServers": {
    "n8n": {
      "command": "npx",
      "args": ["-y", "n8n-mcp-server"],
      "env": {
        "N8N_BASE_URL": "http://localhost:5678",
        "N8N_API_KEY": "<generate from n8n UI>"
      }
    }
  }
}
```

**Fallback if no MCP package works:** Use the `update-config` skill to add an HTTP-based approach, and interact with n8n via direct REST API calls using the Bash/PowerShell tool.

### Step 1.2 — Generate n8n API Key

In n8n UI: **Settings → n8n API → Create new API key**  
Copy the key and add to the MCP env above.

### Step 1.3 — Verify/Create Credentials in n8n UI

Go to **n8n UI → Credentials** and ensure these exist:

| Credential Name | Type | Fields |
|----------------|------|--------|
| `OpenRouter` | HTTP Header Auth | Header: `Authorization`, Value: `Bearer YOUR_KEY` |
| `Serper` | HTTP Header Auth | Header: `X-API-KEY`, Value: `YOUR_SERPER_KEY` |
| `Gemini Flash` | HTTP Header Auth | Header: `x-goog-api-key`, Value: `YOUR_GEMINI_KEY` |
| `Unsplash` | HTTP Header Auth | Header: `Authorization`, Value: `Client-ID YOUR_KEY` |
| `Telegram Bot` | Telegram API | Bot Token |
| `GitHub PAT` | HTTP Header Auth | Header: `Authorization`, Value: `Bearer YOUR_PAT` |

Cloudinary uses unsigned upload — no credential needed in n8n.

### Step 1.4 — Create Cloudinary Unsigned Upload Preset

In Cloudinary Dashboard: **Settings → Upload → Add upload preset**
- Name: `techvia_unsigned`
- Signing mode: `Unsigned`
- Folder: `blog`

### Verification Checklist
- [ ] `GET http://localhost:5678/api/v1/workflows` returns 200 with `X-N8N-API-KEY` header
- [ ] All 6 credentials appear in n8n Credentials list
- [ ] Cloudinary preset `techvia_unsigned` exists and is set to Unsigned

---

## Phase 2 — Workflow A: Main Blog Pipeline

**Goal:** Create the primary workflow JSON and import it into n8n via API.

This workflow handles: Schedule → Trend Research → Telegram Approval (send + wait) → Content Writing → Parallel (Metadata + Image) → Assemble → GitHub Commit → Telegram Success.

### Step 2.1 — Build Workflow JSON

Create file `n8n/workflows/techvia-blog-pipeline.json` in the repo.

**Node: Schedule Trigger**
```json
{
  "id": "node-schedule",
  "name": "Daily 6AM",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [250, 300],
  "parameters": {
    "rule": {
      "interval": [{ "field": "cronExpression", "expression": "0 6 * * *" }]
    }
  }
}
```

**Node: Trend Research (Serper.dev news search)**
```json
{
  "id": "node-serper",
  "name": "Search Trends",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [450, 300],
  "parameters": {
    "method": "POST",
    "url": "https://google.serper.dev/news",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "bodyParameters": {
      "parameters": [
        { "name": "q", "value": "web development cloud solutions technology trends" },
        { "name": "num", "value": "10" },
        { "name": "tbs", "value": "qdr:w" }
      ]
    },
    "options": {}
  },
  "credentials": { "httpHeaderAuth": { "id": "SERPER_CRED_ID", "name": "Serper" } }
}
```

**Node: Topic Selection (OpenRouter)**
```json
{
  "id": "node-topics",
  "name": "Select Topics",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [650, 300],
  "parameters": {
    "method": "POST",
    "url": "https://openrouter.ai/api/v1/chat/completions",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ model: 'google/gemini-flash-1.5', messages: [{ role: 'system', content: 'You are a content strategist for Techvia — a tech company offering Web Development, Cloud Solutions, Mobile Development, and IT Consulting.' }, { role: 'user', content: 'Based on these news items, pick the 3 most relevant trending topics for a Techvia blog post. Return ONLY a JSON array: [{\"title\": \"...\", \"angle\": \"one sentence hook\", \"pillar\": \"service category\"}]. News: ' + JSON.stringify($json.news.slice(0,8).map(n => n.title + ': ' + n.snippet)) }], max_tokens: 500 }) }}",
    "options": {}
  },
  "credentials": { "httpHeaderAuth": { "id": "OPENROUTER_CRED_ID", "name": "OpenRouter" } }
}
```

**Node: Parse Topics (Code)**
```json
{
  "id": "node-parse",
  "name": "Parse Topics",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [850, 300],
  "parameters": {
    "jsCode": "const raw = $json.choices[0].message.content;\nconst topics = JSON.parse(raw.replace(/```json\\n?|```/g, '').trim());\nreturn [{ json: { topics, topicsText: topics.map((t, i) => `${i+1}. <b>${t.title}</b>\\n   ${t.angle} [${t.pillar}]`).join('\\n\\n') } }];"
  }
}
```

**Node: Send Telegram Approval**
```json
{
  "id": "node-tg-send",
  "name": "Send Approval Message",
  "type": "n8n-nodes-base.telegram",
  "typeVersion": 1.2,
  "position": [1050, 300],
  "parameters": {
    "resource": "message",
    "operation": "sendMessage",
    "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}",
    "text": "=📰 <b>Daily Blog Topics — pick one:</b>\\n\\n{{ $json.topicsText }}\\n\\nReply with 1, 2, or 3. Reply SKIP to abort.",
    "additionalFields": {
      "parse_mode": "HTML",
      "replyMarkup": "inlineKeyboard",
      "inlineKeyboard": {
        "rows": [{
          "row": {
            "buttons": [
              { "text": "1️⃣", "callback_data": "topic_1" },
              { "text": "2️⃣", "callback_data": "topic_2" },
              { "text": "3️⃣", "callback_data": "topic_3" },
              { "text": "⏭ SKIP", "callback_data": "topic_skip" }
            ]
          }
        }]
      }
    }
  },
  "credentials": { "telegramApi": { "id": "TELEGRAM_CRED_ID", "name": "Telegram Bot" } }
}
```

**Node: Store Resume URL + Wait**
```json
{
  "id": "node-wait",
  "name": "Wait for Approval",
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "position": [1250, 300],
  "parameters": {
    "resume": "webhook",
    "options": { "webhookSuffix": "/approval" }
  }
}
```

> **Important:** Before the Wait node, add a Code node that stores `$execution.resumeUrl` in workflow static data keyed by the Telegram `message_id`:
```javascript
// Store Resume URL node
const staticData = $getWorkflowStaticData('global');
const messageId = $('Send Approval Message').item.json.result.message_id;
staticData[`resume_${messageId}`] = $execution.resumeUrl;
return $input.all();
```

**Node: Content Writer (OpenRouter)**
```json
{
  "id": "node-writer",
  "name": "Write Article",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1450, 300],
  "parameters": {
    "method": "POST",
    "url": "https://openrouter.ai/api/v1/chat/completions",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ model: $vars.WRITING_MODEL || 'anthropic/claude-sonnet-4-5', messages: [{ role: 'system', content: 'You are a senior tech writer for Techvia (techvia.software — Web Dev, Cloud, Mobile, IT Consulting). Write complete SEO-optimised Hugo blog posts in markdown. Use H2/H3 headings, include code examples where relevant, 800-1200 words, end with CTA to techvia.software.' }, { role: 'user', content: 'Write a full blog post about: ' + $('Wait for Approval').item.json.query.topic_title + '. Angle: ' + $('Wait for Approval').item.json.query.topic_angle }], max_tokens: 2500 }) }}"
  },
  "credentials": { "httpHeaderAuth": { "id": "OPENROUTER_CRED_ID", "name": "OpenRouter" } }
}
```

**Node: Metadata Agent (OpenRouter fast model)**
```json
{
  "id": "node-metadata",
  "name": "Extract Metadata",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1650, 200],
  "parameters": {
    "method": "POST",
    "url": "https://openrouter.ai/api/v1/chat/completions",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ model: 'google/gemini-flash-1.5', messages: [{ role: 'user', content: 'Extract metadata from this blog post. Return ONLY JSON: {\"slug\": \"kebab-case-no-date\", \"seo_description\": \"max 150 chars\", \"tags\": [\"tag1\",\"tag2\",\"tag3\"], \"category\": \"one of: Web Development|Cloud Solutions|Mobile Development|IT Consulting|SaaS Development|AI Development\"}. Post: ' + $('Write Article').item.json.choices[0].message.content.substring(0, 1500) }], max_tokens: 300 }) }}"
  },
  "credentials": { "httpHeaderAuth": { "id": "OPENROUTER_CRED_ID", "name": "OpenRouter" } }
}
```

**Node: Image Query (Gemini Flash)**
```json
{
  "id": "node-image-query",
  "name": "Generate Image Query",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1650, 400],
  "parameters": {
    "method": "POST",
    "url": "=https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key={{ $credentials.geminiFlash.apiKey }}",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ contents: [{ parts: [{ text: 'For a blog post titled \"' + $('Wait for Approval').item.json.query.topic_title + '\", generate a concise Unsplash photo search query (3-5 words, no quotes) and alt text. Return ONLY JSON: {\"search_query\": \"...\", \"alt_text\": \"...\"}' }] }] }) }}"
  }
}
```

**Node: Unsplash Search**
```json
{
  "id": "node-unsplash",
  "name": "Search Unsplash",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1850, 400],
  "parameters": {
    "method": "GET",
    "url": "=https://api.unsplash.com/search/photos?query={{ encodeURIComponent($json.search_query) }}&per_page=5&orientation=landscape",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth"
  },
  "credentials": { "httpHeaderAuth": { "id": "UNSPLASH_CRED_ID", "name": "Unsplash" } }
}
```

**Node: Upload to Cloudinary**
```json
{
  "id": "node-cloudinary",
  "name": "Upload to Cloudinary",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [2050, 400],
  "parameters": {
    "method": "POST",
    "url": "https://api.cloudinary.com/v1_1/techvia/image/upload",
    "sendBody": true,
    "contentType": "form-urlencoded",
    "bodyParameters": {
      "parameters": [
        { "name": "file", "value": "={{ $('Search Unsplash').item.json.results[0].urls.regular }}" },
        { "name": "upload_preset", "value": "techvia_unsigned" },
        { "name": "folder", "value": "blog" }
      ]
    }
  }
}
```

**Node: Merge Parallel Branches**
```json
{
  "id": "node-merge",
  "name": "Merge Results",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3,
  "position": [2250, 300],
  "parameters": { "mode": "combine", "combinationMode": "mergeByPosition" }
}
```

**Node: Assembler (Code)**
```json
{
  "id": "node-assembler",
  "name": "Assemble Markdown",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [2450, 300],
  "parameters": {
    "jsCode": "const today = new Date().toISOString().split('T')[0];\nconst content = $('Write Article').item.json.choices[0].message.content;\nconst metaRaw = $('Extract Metadata').item.json.choices[0].message.content;\nconst meta = JSON.parse(metaRaw.replace(/```json\\n?|```/g, '').trim());\nconst imgUrl = $('Upload to Cloudinary').item.json.secure_url;\nconst imgAlt = $('Generate Image Query').item.json.alt_text || meta.slug;\nconst title = $('Wait for Approval').item.json.query.topic_title;\nconst filename = `${today}-${meta.slug}.md`;\nconst frontmatter = `---\\ntitle: \"${title.replace(/\"/g, '\\\\\"')}\"\\ndate: ${today}\\ndraft: false\\ntags: ${JSON.stringify(meta.tags)}\\ncategories: [\"${meta.category}\"]\\ndescription: \"${meta.seo_description.replace(/\"/g, '\\\\\"')}\"\\ncover:\\n  image: \"${imgUrl}\"\\n  alt: \"${imgAlt}\"\\n  relative: false\\nshowToc: true\\n---\\n\\n`;\nreturn [{ json: { filename, markdown: frontmatter + content, slug: meta.slug, title } }];"
  }
}
```

**Node: GitHub Commit (Code + HTTP Request)**

First a Code node to base64-encode:
```javascript
// Base64 Encode node
const content = $json.markdown;
const encoded = Buffer.from(content).toString('base64');
return [{ json: { ...$json, content_b64: encoded } }];
```

Then HTTP Request:
```json
{
  "id": "node-github",
  "name": "Commit to GitHub",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [2850, 300],
  "parameters": {
    "method": "PUT",
    "url": "=https://api.github.com/repos/nikesh9220/techvia-blog/contents/content/posts/{{ $json.filename }}",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Accept", "value": "application/vnd.github+json" },
        { "name": "X-GitHub-Api-Version", "value": "2022-11-28" }
      ]
    },
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ message: 'feat: auto-publish \"' + $json.title + '\" via n8n', content: $json.content_b64, branch: 'master', committer: { name: 'Techvia Bot', email: 'bot@techvia.software' } }) }}"
  },
  "credentials": { "httpHeaderAuth": { "id": "GITHUB_CRED_ID", "name": "GitHub PAT" } }
}
```

**Node: Telegram Success Notification**
```json
{
  "id": "node-tg-success",
  "name": "Notify Success",
  "type": "n8n-nodes-base.telegram",
  "typeVersion": 1.2,
  "position": [3050, 300],
  "parameters": {
    "resource": "message",
    "operation": "sendMessage",
    "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}",
    "text": "=✅ <b>Published!</b>\\n\\n📝 {{ $('Assemble Markdown').item.json.title }}\\n🔗 https://blogs.techvia.software/posts/{{ $('Assemble Markdown').item.json.slug }}/\\n\\nCloudflare is deploying (~30 sec) 🚀",
    "additionalFields": { "parse_mode": "HTML" }
  },
  "credentials": { "telegramApi": { "id": "TELEGRAM_CRED_ID", "name": "Telegram Bot" } }
}
```

### Step 2.2 — POST Workflow to n8n

```bash
# Via PowerShell / Bash
curl -X POST http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d @n8n/workflows/techvia-blog-pipeline.json
```

### Verification Checklist — Phase 2
- [ ] Workflow appears in n8n UI
- [ ] All nodes visible with correct connections
- [ ] All credential references resolve (no red credential errors)
- [ ] Workflow variables set: `TELEGRAM_CHAT_ID`, `WRITING_MODEL`

---

## Phase 3 — Workflow B: Telegram Approval Handler

**Goal:** A second always-on workflow that receives Telegram button callbacks and resumes the waiting main workflow.

### Step 3.1 — Build Approval Handler JSON

Create `n8n/workflows/techvia-telegram-handler.json`:

**Node: Telegram Trigger**
```json
{
  "id": "node-tg-trigger",
  "name": "Telegram Callback",
  "type": "n8n-nodes-base.telegramTrigger",
  "typeVersion": 1.1,
  "position": [250, 300],
  "parameters": {
    "updates": ["callback_query"]
  },
  "credentials": { "telegramApi": { "id": "TELEGRAM_CRED_ID", "name": "Telegram Bot" } }
}
```

**Node: Extract Choice + Resume URL (Code)**
```javascript
// Extract and Route node
const cb = $json.callback_query;
const choice = cb.data; // "topic_1", "topic_2", "topic_3", "topic_skip"
const messageId = cb.message.message_id;

// Get stored resume URL
const staticData = $getWorkflowStaticData('global');
const resumeUrl = staticData[`resume_${messageId}`];

if (!resumeUrl) {
  return [{ json: { error: 'No waiting workflow found for message ' + messageId } }];
}

// Parse topic number from choice
const topicNum = choice === 'topic_skip' ? null : parseInt(choice.replace('topic_', '')) - 1;

// Clean up static data
delete staticData[`resume_${messageId}`];

return [{ json: { resumeUrl, topicNum, isSkip: choice === 'topic_skip', callbackQueryId: cb.id } }];
```

**Node: Answer Callback (HTTP Request)**
```json
{
  "method": "POST",
  "url": "=https://api.telegram.org/bot{{ $credentials.telegramApi.token }}/answerCallbackQuery",
  "body": "={{ JSON.stringify({ callback_query_id: $json.callbackQueryId, text: $json.isSkip ? '⏭ Skipping today' : '✅ Got it! Writing now...' }) }}"
}
```

**Node: Resume Main Workflow (HTTP Request)**
```json
{
  "method": "POST",
  "url": "={{ $json.resumeUrl }}",
  "sendBody": true,
  "contentType": "json",
  "body": "={{ JSON.stringify({ topicNum: $json.topicNum, isSkip: $json.isSkip }) }}"
}
```

> Note: Only call resume if `!isSkip`. Add an IF node before resume to check `$json.isSkip === false`.

### Step 3.2 — Import and Activate Handler Workflow

```bash
curl -X POST http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d @n8n/workflows/techvia-telegram-handler.json

# Then activate it
curl -X POST http://localhost:5678/api/v1/workflows/{HANDLER_ID}/activate \
  -H "X-N8N-API-KEY: YOUR_KEY"
```

### Verification Checklist — Phase 3
- [ ] Telegram Handler workflow is active (green dot)
- [ ] Sending `/start` to your Telegram bot triggers the handler
- [ ] No errors in n8n execution log for the handler

---

## Phase 4 — Error Handling

**Goal:** Add error notification nodes so failures alert via Telegram instead of silently dying.

### Step 4.1 — Add Error Workflow

Create a minimal error-handler workflow:

```json
{
  "name": "Blog Pipeline Error Handler",
  "nodes": [{
    "name": "Error Trigger",
    "type": "n8n-nodes-base.errorTrigger",
    "parameters": {}
  }, {
    "name": "Telegram Error",
    "type": "n8n-nodes-base.telegram",
    "parameters": {
      "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}",
      "text": "=❌ <b>Blog automation failed</b>\\n\\nNode: {{ $json.execution.lastNodeExecuted }}\\nError: {{ $json.execution.error.message }}\\n\\nCheck: http://localhost:5678",
      "additionalFields": { "parse_mode": "HTML" }
    }
  }]
}
```

### Step 4.2 — Link Error Workflow to Main Pipeline

In n8n UI: Main pipeline → Settings → Error Workflow → select `Blog Pipeline Error Handler`.

---

## Phase 5 — Testing & Activation

### Step 5.1 — Manual Test (dry run)

1. In n8n UI, open the main pipeline
2. Click **Test workflow** (manual trigger)
3. Step through each node, verify outputs match expected shapes:
   - Serper: `$json.news` is an array
   - Topics: parses to 3-item JSON array
   - Telegram: message sent, `result.message_id` present
   - Wait: execution suspends
   - (Manually call resume URL with `{"topicNum": 0, "isSkip": false}`)
   - Content writer: `choices[0].message.content` is markdown
   - Metadata: parses slug, tags, category
   - Unsplash: `results[0].urls.regular` is a valid URL
   - Cloudinary: `secure_url` is `https://res.cloudinary.com/techvia/...`
   - Assembler: `markdown` starts with `---` frontmatter
   - GitHub: returns `201 Created`
   - Telegram: success message received

### Step 5.2 — Activate Main Workflow

```bash
curl -X POST http://localhost:5678/api/v1/workflows/{MAIN_ID}/activate \
  -H "X-N8N-API-KEY: YOUR_KEY"
```

### Final Verification Checklist
- [ ] Workflow runs end-to-end on manual test
- [ ] GitHub commit appears in repo
- [ ] Article visible at `https://blogs.techvia.software/posts/{slug}/` after Cloudflare deploy (~60s)
- [ ] Telegram success notification received
- [ ] Error workflow fires on simulated failure
- [ ] Daily schedule set to desired time

---

## Configuration Reference

| Variable | Where to set | Default |
|---------|-------------|---------|
| `TELEGRAM_CHAT_ID` | n8n workflow variables | — |
| `WRITING_MODEL` | n8n workflow variables | `anthropic/claude-sonnet-4-5` |
| `FAST_MODEL` | n8n workflow variables | `google/gemini-flash-1.5` |
| Cloudinary cloud name | Hardcoded in URLs | `techvia` |
| Upload preset | Cloudinary dashboard | `techvia_unsigned` |
| Cron schedule | Schedule Trigger node | `0 6 * * *` (6am) |

---

## File Structure to Create

```
techvia-blog/
└── n8n/
    └── workflows/
        ├── techvia-blog-pipeline.json       ← Main workflow
        └── techvia-telegram-handler.json    ← Approval handler
```
