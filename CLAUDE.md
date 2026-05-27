# Techvia Blog — Claude Code Setup Prompt

## Project Context
You are setting up a Hugo static blog for **Nikesh Pandya / Techvia** at `blogs.techvia.software`.
- Stack: Hugo + PaperMod theme + Cloudflare Pages + Cloudinary (images)
- Content pillars: AI-first development, Angular/.NET tips, Azure/DevOps
- Automation: n8n will push daily articles via GitHub commits

---

## Task: Full Hugo Blog Setup + Test Article

### Step 1 — Check Prerequisites
```bash
hugo version        # Must be v0.110+
git --version       # Must be installed
node --version      # Optional but good to confirm
```

If Hugo not installed:
```bash
# Windows
winget install Hugo.Hugo.Extended

# Mac
brew install hugo
```

---

### Step 2 — Create Hugo Site
```bash
hugo new site techvia-blog
cd techvia-blog
git init
```

---

### Step 3 — Install PaperMod Theme
```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
git submodule update --init --recursive
```

---

### Step 4 — Create hugo.toml
Replace the default `hugo.toml` with this exact config:

```toml
baseURL = "https://blogs.techvia.software/"
languageCode = "en-us"
title = "Techvia Blog"
theme = "PaperMod"
paginate = 10
enableRobotsTXT = true
buildDrafts = false
buildFuture = false

[minify]
  disableXML = true
  minifyOutput = true

[params]
  env = "production"
  title = "Techvia Blog"
  description = "Insights on AI-first development, Angular 18, .NET Core, and Azure DevOps by Nikesh Pandya"
  author = "Nikesh Pandya"
  defaultTheme = "auto"
  ShowReadingTime = true
  ShowShareButtons = true
  ShowPostNavLinks = true
  ShowBreadCrumbs = true
  ShowCodeCopyButtons = true
  ShowRssButtonInSectionTermList = true
  disableThemeToggle = false

  [params.homeInfoParams]
    Title = "Techvia Blog"
    Content = "AI-first development, Angular 18, .NET Core 8, and Azure — by Nikesh Pandya, founder of Techvia."

  [params.socialIcons]
    [[params.socialIcons]]
      name = "linkedin"
      url = "https://linkedin.com/in/nikeshpandya"
    [[params.socialIcons]]
      name = "github"
      url = "https://github.com/nikeshpandya"

[taxonomies]
  category = "categories"
  tag = "tags"
  series = "series"

[markup.goldmark.renderer]
  unsafe = true

[markup.highlight]
  noClasses = false

[[menu.main]]
  identifier = "home"
  name = "Home"
  url = "/"
  weight = 10
[[menu.main]]
  identifier = "tags"
  name = "Tags"
  url = "/tags/"
  weight = 20
[[menu.main]]
  identifier = "categories"
  name = "Categories"
  url = "/categories/"
  weight = 30
[[menu.main]]
  identifier = "search"
  name = "Search"
  url = "/search/"
  weight = 40

[outputs]
  home = ["HTML", "RSS", "JSON"]
```

---

### Step 5 — Create Folder Structure
```bash
mkdir -p content/posts
mkdir -p static/images
mkdir -p archetypes
```

Create `archetypes/posts.md` (this is the template n8n will follow):
```markdown
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: false
tags: []
categories: []
description: ""
cover:
  image: ""
  alt: ""
  caption: ""
  relative: false
showToc: true
tocOpen: false
---

<!-- Article content here -->
```

---

### Step 6 — Add Search Page
```bash
hugo new content/search.md
```

Add this content:
```yaml
---
title: "Search"
layout: "search"
summary: "search"
placeholder: "Search articles..."
---
```

---

### Step 7 — Create 3 Test Articles

**Article 1 — AI Development**

Create `content/posts/2025-05-27-angular-18-signals-claude-api.md`:

```markdown
---
title: "Building AI-First Apps with Angular 18 Signals and Claude API"
date: 2025-05-27
draft: false
tags: ["angular", "ai", "claude-api", "typescript"]
categories: ["AI Development"]
description: "Learn how to integrate Anthropic's Claude API into Angular 18 using the new Signals architecture for reactive AI-powered applications."
cover:
  image: "https://res.cloudinary.com/techvia/image/upload/blog/angular-signals-ai.jpg"
  alt: "Angular 18 Signals with Claude API"
  relative: false
showToc: true
---

## Introduction

Angular 18 introduced a powerful reactivity model with **Signals** — and combined with the Claude API, it unlocks a new class of AI-first web applications. In this post, we'll walk through building a reactive AI assistant component from scratch.

## Why Signals for AI Apps?

Traditional Angular apps used RxJS observables for async state. While powerful, they add cognitive overhead when dealing with streaming AI responses. Signals simplify this:

```typescript
// Traditional RxJS approach
aiResponse$: Observable<string>;

// Signals approach — cleaner, more intuitive
aiResponse = signal<string>('');
isLoading = signal<boolean>(false);
```

## Setting Up Claude API in Angular

Install the HTTP client and set up your environment:

```typescript
// environment.ts
export const environment = {
  claudeApiUrl: 'https://api.anthropic.com/v1/messages',
  claudeModel: 'claude-sonnet-4-20250514'
};
```

## Building the AI Service

```typescript
@Injectable({ providedIn: 'root' })
export class ClaudeService {
  private http = inject(HttpClient);
  
  askClaude(prompt: string): Observable<string> {
    return this.http.post<any>(environment.claudeApiUrl, {
      model: environment.claudeModel,
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }]
    }).pipe(
      map(res => res.content[0].text)
    );
  }
}
```

## The Signal-Based Component

```typescript
@Component({
  selector: 'app-ai-chat',
  template: `
    <div class="ai-container">
      <textarea [(ngModel)]="userInput" placeholder="Ask anything..."></textarea>
      <button (click)="ask()" [disabled]="isLoading()">
        {{ isLoading() ? 'Thinking...' : 'Ask Claude' }}
      </button>
      <div class="response">{{ aiResponse() }}</div>
    </div>
  `
})
export class AiChatComponent {
  userInput = '';
  aiResponse = signal('');
  isLoading = signal(false);
  
  private claude = inject(ClaudeService);
  
  ask() {
    this.isLoading.set(true);
    this.claude.askClaude(this.userInput).subscribe({
      next: (res) => {
        this.aiResponse.set(res);
        this.isLoading.set(false);
      }
    });
  }
}
```

## Conclusion

Combining Angular 18 Signals with Claude API gives you a clean, reactive architecture for AI-powered features. The signal-based approach reduces boilerplate and makes state changes predictable.

**At Techvia**, we use this exact pattern in **FleetIQ** — our AI-first Transport Management System — for route optimization suggestions and document intelligence.
```

---

**Article 2 — Azure DevOps**

Create `content/posts/2025-05-26-dotnet-azure-app-service-ci-cd.md`:

```markdown
---
title: ".NET Core 8 CI/CD to Azure App Service with GitHub Actions"
date: 2025-05-26
draft: false
tags: ["dotnet", "azure", "github-actions", "devops", "ci-cd"]
categories: ["Azure DevOps"]
description: "A complete guide to setting up automated CI/CD pipelines for .NET Core 8 APIs on Azure App Service using GitHub Actions — zero downtime deployments included."
cover:
  image: "https://res.cloudinary.com/techvia/image/upload/blog/dotnet-azure-cicd.jpg"
  alt: ".NET Core 8 Azure CI/CD Pipeline"
  relative: false
showToc: true
---

## Overview

Deploying .NET Core 8 APIs manually is error-prone. Here's how to set up a **production-grade GitHub Actions pipeline** that deploys to Azure App Service with zero downtime on every push to `main`.

## Prerequisites

- Azure App Service (Free or Basic tier)
- GitHub repository with your .NET Core 8 API
- Azure Service Principal or Publish Profile

## The GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Azure App Service

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Test
      run: dotnet test --no-build --verbosity normal
    
    - name: Publish
      run: dotnet publish -c Release -o ./publish
    
    - name: Deploy to Azure
      uses: azure/webapps-deploy@v3
      with:
        app-name: ${{ secrets.AZURE_APP_NAME }}
        publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
        package: ./publish
```

## Setting Up Secrets

In GitHub → Settings → Secrets:
```
AZURE_APP_NAME      = your-app-name
AZURE_PUBLISH_PROFILE = <paste XML from Azure portal>
```

Download publish profile: **Azure Portal → App Service → Get Publish Profile**.

## Zero Downtime with Deployment Slots

```yaml
- name: Deploy to Staging Slot
  uses: azure/webapps-deploy@v3
  with:
    app-name: ${{ secrets.AZURE_APP_NAME }}
    slot-name: staging
    publish-profile: ${{ secrets.AZURE_STAGING_PROFILE }}
    package: ./publish

- name: Swap Staging to Production
  uses: azure/CLI@v2
  with:
    inlineScript: |
      az webapp deployment slot swap \
        --resource-group ${{ secrets.AZURE_RG }} \
        --name ${{ secrets.AZURE_APP_NAME }} \
        --slot staging \
        --target-slot production
```

## Conclusion

With this setup, every push to `main` triggers a full build, test, and zero-downtime deployment. At **Techvia**, this is our standard deployment pattern for all SaaS products including FleetIQ.
```

---

**Article 3 — Techvia/FleetIQ**

Create `content/posts/2025-05-25-building-saas-mvp-angular-dotnet.md`:

```markdown
---
title: "How We Built a SaaS MVP in 30 Days with Angular 18 and .NET Core 8"
date: 2025-05-25
draft: false
tags: ["saas", "angular", "dotnet", "startup", "techvia"]
categories: ["SaaS Development"]
description: "Behind the scenes of building FleetIQ — Techvia's AI-first Transport Management System. Architecture decisions, lessons learned, and what we'd do differently."
cover:
  image: "https://res.cloudinary.com/techvia/image/upload/blog/saas-mvp-build.jpg"
  alt: "Building SaaS MVP Angular .NET"
  relative: false
showToc: true
---

## The Challenge

When we started building **FleetIQ** at Techvia, we had one rule: ship a working MVP in 30 days without compromising on architecture. Here's how we did it.

## Tech Stack Decision

After evaluating several options, we settled on:

| Layer | Technology | Reason |
|-------|-----------|--------|
| Frontend | Angular 18 | Team expertise, enterprise-grade |
| Backend | .NET Core 8 | Performance, Azure native |
| Database | PostgreSQL | Reliability, JSON support |
| AI | Claude API | Best-in-class reasoning |
| Cloud | Azure | Existing infrastructure |
| Auth | Azure AD B2C | Enterprise SSO ready |

## Architecture: Polyrepo Over Monorepo

We chose a **polyrepo** structure early:

```
fleetiq-api/          ← .NET Core 8 Web API
fleetiq-web/          ← Angular 18 SPA
fleetiq-shared/       ← Shared TypeScript types
```

This paid off when we needed to scale the API independently during load testing.

## Day 1–7: Core API

We prioritised the API first:

```csharp
// Clean architecture from day one
FleetIQ.API/
  Controllers/
  FleetIQ.Application/
    Commands/
    Queries/
    Handlers/
  FleetIQ.Domain/
    Entities/
    Interfaces/
  FleetIQ.Infrastructure/
    Repositories/
    Services/
```

## Day 8–20: Angular Frontend

With the API stable, we moved to Angular 18 with standalone components throughout:

```typescript
@Component({
  standalone: true,
  imports: [CommonModule, RouterModule],
  selector: 'app-fleet-dashboard',
  templateUrl: './fleet-dashboard.component.html'
})
export class FleetDashboardComponent {
  vehicles = signal<Vehicle[]>([]);
  // ...
}
```

## Day 21–28: Claude API Integration

The AI layer was our differentiator. We integrated Claude API for:
- **Route optimization suggestions**
- **Document intelligence** (driver docs, permits)
- **Anomaly detection** in fleet data

## Day 29–30: Azure Deployment

CI/CD via GitHub Actions → Azure App Service. Took 4 hours to fully automate.

## Lessons Learned

1. **Standalone components** in Angular 18 saved significant boilerplate
2. **CQRS pattern** in .NET made the codebase scale cleanly
3. **Claude API** integration was faster than expected — 2 days for core features
4. **Polyrepo** > monorepo for our team size and deployment independence

## What's Next for FleetIQ

We're currently onboarding beta customers. If you're managing employee transport or fleet operations, [reach out to us](https://techvia.software).
```

---

### Step 8 — Run Local Dev Server
```bash
hugo server -D --bind 0.0.0.0
```
Visit: `http://localhost:1313`

Verify:
- [ ] Home page loads with 3 articles
- [ ] Each article opens fully
- [ ] Tags and categories work
- [ ] Search page exists
- [ ] Dark/light mode toggle works

---

### Step 9 — Build for Production
```bash
hugo --minify
```

Check `public/` folder is generated with HTML files.

---

### Step 10 — Cloudflare Pages Config

Create `cloudflare-pages.toml` in root (optional but good practice):
```toml
[build]
  command = "hugo --minify"
  publish = "public"

[build.environment]
  HUGO_VERSION = "0.147.0"
```

---

### Step 11 — Push to GitHub
```bash
git add .
git commit -m "feat: initial techvia blog setup with 3 test articles"
git push origin main
```

---

## Expected Output After Setup

```
techvia-blog/
├── hugo.toml                          ← site config
├── content/
│   ├── posts/
│   │   ├── 2025-05-27-angular-18-signals-claude-api.md
│   │   ├── 2025-05-26-dotnet-azure-app-service-ci-cd.md
│   │   └── 2025-05-25-building-saas-mvp-angular-dotnet.md
│   └── search.md
├── static/
│   └── images/
├── archetypes/
│   └── posts.md                       ← n8n article template
├── themes/
│   └── PaperMod/
└── cloudflare-pages.toml
```

---

## n8n Article Frontmatter Reference

When n8n generates articles via Claude API, use this exact frontmatter format:

```yaml
---
title: "{GENERATED_TITLE}"
date: {YYYY-MM-DD}
draft: false
tags: ["{TAG1}", "{TAG2}", "{TAG3}"]
categories: ["{CATEGORY}"]
description: "{SEO_META_DESCRIPTION_150_CHARS}"
cover:
  image: "{CLOUDINARY_OR_UNSPLASH_URL}"
  alt: "{IMAGE_ALT_TEXT}"
  relative: false
showToc: true
---
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Theme not found | Run `git submodule update --init --recursive` |
| Posts not showing | Check `draft: false` in frontmatter |
| Build fails on Cloudflare | Set `HUGO_VERSION = 0.147.0` env var |
| Search not working | Ensure `outputs` includes `JSON` in hugo.toml |
| Cover image not rendering | Check `params.cover.linkFullImages = true` in hugo.toml |
