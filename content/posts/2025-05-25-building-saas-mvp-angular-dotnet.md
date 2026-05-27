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
