---

# App Service Deployment Guide

## .NET Framework 4.8 → GitLab CI/CD → Azure App Service

### Secure Enterprise Implementation (Mature Landing Zone)

---

# 1. Overview

This document describes a secure, enterprise-grade CI/CD implementation for deploying a **.NET Framework 4.8** application to **Azure App Service (Windows)** using **GitLab CI/CD**.

The solution aligns with a **Mature Azure Landing Zone** including:

* Identity isolation
* Least privilege access
* Network segmentation
* Private access patterns
* Deployment slots
* Centralized monitoring
* Zero-downtime deployments

---

# 2. Target Architecture (Mature Landing Zone)

```
┌──────────────────────────────┐
│          Developer           │
└──────────────┬───────────────┘
               │ git push
               ▼
┌──────────────────────────────┐
│         GitLab CI/CD         │
│  - Build (.NET 4.8)          │
│  - Publish                   │
│  - Package (ZIP)             │
│  - Azure CLI Deployment      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│           Azure Landing Zone (Spoke)        │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ Azure App Service (Windows PaaS)     │  │
│  │ - Runtime: .NET 4.8                  │  │
│  │ - Private Endpoint (optional)        │  │
│  │ - VNet Integration                   │  │
│  │ - Managed Identity                   │  │
│  │ - Deployment Slots                   │  │
│  │ - Application Insights               │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ Log Analytics Workspace               │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

# 3. Security Architecture

## 3.1 Identity Model

Recommended (Enterprise):

* Azure AD Service Principal OR Workload Identity
* RBAC scoped to Resource Group
* No subscription-wide Contributor role
* Secrets stored only in GitLab protected variables

Best Practice:

* Use Managed Identity for runtime access to Azure resources
* Use Service Principal only for deployment

---

## 3.2 Network Model (Recommended)

* App Service integrated with VNet
* Private Endpoint enabled
* Public access restricted
* Fronted by:

  * Azure Application Gateway OR
  * Azure Front Door (Enterprise pattern)

---

# 4. Azure Resource Provisioning (CLI)

Assume resource group already exists.

---

## 4.1 Create App Service Plan (Production Tier)

```bash
az appservice plan create \
  --name myapp-plan-prod \
  --resource-group my-rg-prod \
  --sku P1v3 \
  --is-linux false
```

---

## 4.2 Create Web App (Windows / .NET 4.8)

```bash
az webapp create \
  --name myapp-prod \
  --resource-group my-rg-prod \
  --plan myapp-plan-prod \
  --runtime "DOTNET|4.8"
```

---

## 4.3 Enforce HTTPS Only

```bash
az webapp update \
  --name myapp-prod \
  --resource-group my-rg-prod \
  --https-only true
```

---

## 4.4 Enable System Managed Identity

```bash
az webapp identity assign \
  --name myapp-prod \
  --resource-group my-rg-prod
```

---

## 4.5 Enable Application Insights

```bash
az monitor app-insights component create \
  --app myapp-prod-ai \
  --location westeurope \
  --resource-group my-rg-prod \
  --application-type web
```

---

# 5. Deployment Strategy

Enterprise deployment uses:

* ZIP Deploy
* Deployment Slots
* Slot Swap
* No direct production deployment
* Versioned artifacts

---

# 6. Required GitLab CI/CD Variables

Add under:

GitLab → Settings → CI/CD → Variables

| Variable              | Description          |
| --------------------- | -------------------- |
| AZURE_CLIENT_ID       | Service Principal ID |
| AZURE_CLIENT_SECRET   | Secret               |
| AZURE_TENANT_ID       | Tenant ID            |
| AZURE_SUBSCRIPTION_ID | Subscription ID      |
| RESOURCE_GROUP        | Resource Group       |
| APP_NAME              | App Service Name     |

Mark secrets:

* Protected
* Masked

---

# 7. Publish Profile (Required in Project)

`Properties/PublishProfiles/FolderProfile.pubxml`

```xml
<Project>
  <PropertyGroup>
    <WebPublishMethod>FileSystem</WebPublishMethod>
    <PublishUrl>publish\</PublishUrl>
    <DeleteExistingFiles>true</DeleteExistingFiles>
  </PropertyGroup>
</Project>
```

---

# 8. Full Secure Enterprise GitLab Pipeline

```yaml
stages:
  - build
  - deploy_staging
  - swap

variables:
  SLOT_NAME: "staging"

build:
  stage: build
  tags:
    - windows
  script:
    - msbuild YourApp.sln /p:Configuration=Release /p:DeployOnBuild=true /p:PublishProfile=FolderProfile
    - powershell Compress-Archive publish\* app.zip
  artifacts:
    paths:
      - app.zip

deploy_staging:
  stage: deploy_staging
  image: mcr.microsoft.com/azure-cli
  script:
    - az login --service-principal \
        -u $AZURE_CLIENT_ID \
        -p $AZURE_CLIENT_SECRET \
        --tenant $AZURE_TENANT_ID

    - az account set --subscription $AZURE_SUBSCRIPTION_ID

    - az webapp deployment slot create \
        --name $APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --slot $SLOT_NAME || echo "Slot exists"

    - az webapp deployment source config-zip \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --slot $SLOT_NAME \
        --src app.zip

  only:
    - main

swap:
  stage: swap
  image: mcr.microsoft.com/azure-cli
  script:
    - az login --service-principal \
        -u $AZURE_CLIENT_ID \
        -p $AZURE_CLIENT_SECRET \
        --tenant $AZURE_TENANT_ID

    - az account set --subscription $AZURE_SUBSCRIPTION_ID

    - az webapp deployment slot swap \
        --name $APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --slot $SLOT_NAME \
        --target-slot production

  when: manual
  only:
    - main
```

---

# 9. Enterprise Deployment Flow

```
git push main
      ↓
Build
      ↓
Publish
      ↓
ZIP Package
      ↓
Deploy to Staging Slot
      ↓
Validate
      ↓
Manual Approval
      ↓
Slot Swap (Zero Downtime)
      ↓
Production Live
```

---

# 10. Observability & Monitoring

Enterprise monitoring includes:

* Application Insights enabled
* Log Analytics workspace
* Diagnostic logs
* Availability tests
* Alert rules (CPU, memory, 5xx errors)
* Smart Detection enabled

---

# 11. Enterprise Hardening Checklist

* HTTPS Only enabled
* TLS 1.2 minimum
* Managed Identity enabled
* Secrets in Key Vault
* No public SCM access (if private endpoint used)
* RBAC scoped to RG only
* Deployment via slot only
* Production slot never directly deployed

---

# 12. Zero Downtime Strategy

Deployment slot approach ensures:

* No traffic interruption
* Warm-up before swap
* Rollback via reverse swap
* Controlled release approvals

---

# 13. Future Evolution Path

* Infrastructure as Code (Terraform/Bicep)
* Blue/Green multi-region
* Azure Front Door global routing
* Web Application Firewall
* Dev/Test/Prod environment separation
* Canary deployment strategy

---

# 14. Summary

This solution provides:

* Fully automated CI/CD
* Secure enterprise authentication
* Zero-downtime deployment
* Landing zone compliance
* PaaS-based architecture
* Operational efficiency
* Reduced infrastructure management overhead

---

End of Document.
