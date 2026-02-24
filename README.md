---

# IIS Deployment Guide

## .NET Framework 4.8 → GitLab CI/CD → Azure VM (Mature Landing Zone)

---

# 1. Overview

This document describes the fully automated deployment of a **.NET Framework 4.8** application to **IIS hosted on an Azure Virtual Machine**, using **GitLab CI/CD**.

The design follows a **mature Azure landing zone architecture**, aligned with enterprise standards for:

* Network segmentation
* Secure remote execution
* Separation of duties
* Repeatable infrastructure
* Idempotent deployment
* CI/CD driven provisioning

---

# 2. High-Level Architecture

```
┌──────────────────────────────┐
│          Developer           │
└──────────────┬───────────────┘
               │ git push
               ▼
┌──────────────────────────────┐
│         GitLab (SaaS)       │
│   - Source Repository        │
│   - CI/CD Pipeline           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   GitLab Windows Runner      │
│   - MSBuild                  │
│   - Publish Profile          │
│   - PowerShell Remoting      │
└──────────────┬───────────────┘
               │ WinRM (Secure)
               ▼
┌──────────────────────────────────────────┐
│        Azure Landing Zone (Prod)        │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Virtual Network (Spoke)           │  │
│  │                                    │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ Azure VM (Windows Server)    │  │  │
│  │  │ - IIS                        │  │  │
│  │  │ - .NET Framework 4.8         │  │  │
│  │  │ - Application Pool            │  │  │
│  │  │ - Website                     │  │  │
│  │  └──────────────────────────────┘  │  │
│  │                                    │  │
│  └────────────────────────────────────┘  │
│                                          │
└──────────────────────────────────────────┘
               │
               ▼
        Application Live
```

---

# 3. Mature Landing Zone Alignment

This deployment assumes the following enterprise components exist:

## 3.1 Network Architecture

* Hub-and-Spoke topology
* Azure Firewall or NVA in Hub
* Private VM subnet
* Network Security Groups (NSG)
* Bastion (optional)
* No public RDP exposure (recommended)

## 3.2 Identity & Access

* Dedicated deployment service account
* Least privilege access to:

  * IIS site
  * Application folder
  * WebAdministration module
* GitLab secrets stored as protected variables

## 3.3 Environment Separation

Recommended structure:

```
/subscriptions
    /dev
    /test
    /prod
```

Each environment has:

* Separate VM
* Separate resource group
* Separate GitLab environment

---

# 4. Deployment Flow

```
git push main
    ↓
Pipeline Triggered
    ↓
Infra Validation (IIS + AppPool + Site)
    ↓
Build Solution (MSBuild)
    ↓
Publish Artifacts
    ↓
Remote PowerShell Session to Azure VM
    ↓
Stop IIS Website
    ↓
Copy Published Files
    ↓
Start IIS Website
    ↓
Application Live
```

---

# 5. GitLab CI/CD Pipeline

## 5.1 Required GitLab Variables

| Variable     | Description             |
| ------------ | ----------------------- |
| IIS_SERVER   | Azure VM hostname or IP |
| IIS_USERNAME | Deployment account      |
| IIS_PASSWORD | Deployment password     |

Mark sensitive variables as:

* Masked
* Protected

---

## 5.2 Publish Profile

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

## 5.3 Full `.gitlab-ci.yml`

```yaml
stages:
  - infra
  - build
  - deploy

variables:
  SITE_NAME: "MyApp"
  APP_POOL: "MyAppPool"
  SITE_PATH: "C:\\inetpub\\wwwroot\\MyApp"

infra:
  stage: infra
  tags:
    - windows
  script:
    - powershell -Command "
        $secpasswd = ConvertTo-SecureString '$env:IIS_PASSWORD' -AsPlainText -Force;
        $cred = New-Object System.Management.Automation.PSCredential ('$env:IIS_USERNAME', $secpasswd);
        $session = New-PSSession -ComputerName $env:IIS_SERVER -Credential $cred;

        Invoke-Command -Session $session -ScriptBlock {
            Import-Module WebAdministration;

            if (-not (Test-Path IIS:\AppPools\$using:APP_POOL)) {
                New-WebAppPool -Name $using:APP_POOL;
                Set-ItemProperty IIS:\AppPools\$using:APP_POOL -Name managedRuntimeVersion -Value 'v4.0';
                Set-ItemProperty IIS:\AppPools\$using:APP_POOL -Name managedPipelineMode -Value 'Integrated';
            }

            if (-not (Test-Path '$using:SITE_PATH')) {
                New-Item -ItemType Directory -Path '$using:SITE_PATH' -Force;
            }

            if (-not (Test-Path IIS:\Sites\$using:SITE_NAME)) {
                New-Website -Name $using:SITE_NAME `
                    -Port 80 `
                    -PhysicalPath '$using:SITE_PATH' `
                    -ApplicationPool $using:APP_POOL;
            }
        };

        Remove-PSSession $session;
      "

build:
  stage: build
  tags:
    - windows
  script:
    - msbuild YourApp.sln /p:Configuration=Release /p:DeployOnBuild=true /p:PublishProfile=FolderProfile
  artifacts:
    paths:
      - publish/

deploy:
  stage: deploy
  tags:
    - windows
  script:
    - powershell -Command "
        $secpasswd = ConvertTo-SecureString '$env:IIS_PASSWORD' -AsPlainText -Force;
        $cred = New-Object System.Management.Automation.PSCredential ('$env:IIS_USERNAME', $secpasswd);
        $session = New-PSSession -ComputerName $env:IIS_SERVER -Credential $cred;

        Invoke-Command -Session $session -ScriptBlock {
            Import-Module WebAdministration;
            Stop-WebSite -Name $using:SITE_NAME;
        };

        Copy-Item -Path publish\* -Destination $env:SITE_PATH -Recurse -Force -ToSession $session;

        Invoke-Command -Session $session -ScriptBlock {
            Start-WebSite -Name $using:SITE_NAME;
        };

        Remove-PSSession $session;
      "
```

---

# 6. IIS Configuration Standards

| Setting           | Value                           |
| ----------------- | ------------------------------- |
| Runtime           | .NET CLR v4.0                   |
| Pipeline Mode     | Integrated                      |
| Idle Timeout      | Disabled (recommended)          |
| App Pool Identity | Custom least-privileged account |
| Logging           | Enabled                         |

---

# 7. Mature Enhancements (Recommended)

## 7.1 Zero-Downtime Deployment

Instead of copying over live files:

```
C:\inetpub\wwwroot\MyApp_v1
C:\inetpub\wwwroot\MyApp_v2
```

Switch IIS physical path after deployment.

---

## 7.2 Versioned Releases

Store builds as:

```
C:\releases\2026.02.24.1
```

Enable rollback capability.

---

## 7.3 Monitoring & Observability

In a mature landing zone:

* Azure Monitor Agent on VM
* Log Analytics Workspace
* IIS logs forwarded
* Windows Event Logs collected
* Health endpoint exposed (/health)

---

## 7.4 Security Controls

* Private subnet VM
* No public RDP
* Just-In-Time VM access
* Dedicated deployment identity
* Secret rotation policy
* Encrypted disks

---

# 8. Production Checklist

* [ ] Dedicated deployment account
* [ ] Application pool isolation
* [ ] HTTPS binding configured
* [ ] SSL certificate installed
* [ ] Backup strategy defined
* [ ] Monitoring integrated
* [ ] Rollback mechanism documented
* [ ] Environment separation enforced

---

# 9. Result

This setup provides:

* Infrastructure-as-Code behavior (via CI)
* Repeatable provisioning
* Automated IIS configuration
* Automated deployment
* Azure landing zone compliance
* Enterprise-ready architecture

---

# 10. Next Evolution (Optional)

* Terraform for VM provisioning
* Blue/Green deployment
* Canary deployment
* Azure DevOps Environment approvals
* Web Farm scaling
* Migration to App Service / AKS (future modernization path)

---
