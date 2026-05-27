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
