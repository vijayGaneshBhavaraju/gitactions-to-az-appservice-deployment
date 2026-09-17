GitHub Actions → Azure App Service Deployment

This project demonstrates how to use GitHub Actions to create Azure infrastructure and deploy a .NET 8 application to Azure App Service using OIDC authentication.

1. Repository Structure

.github/
└── workflows/
    ├── infra.yml
    └── deploy.yml

infra.yml — creates the Resource Group, App Service Plan and App Service.

deploy.yml — builds and deploys the .NET 8 application.

2. Prerequisites

GitHub repository

Azure subscription

.NET 8 application

Azure App Registration

Federated Credential

Azure RBAC permission

GitHub repository secrets

3. Create Azure App Registration

Azure Portal → Microsoft Entra ID → App registrations → New registration

Example name:

github-actions-azure

Leave Redirect URI empty.

Copy:

Application (client) ID
Directory (tenant) ID

4. Get Azure Subscription ID

Azure Portal → Subscriptions → Your Subscription

Copy the Subscription ID.

5. Create Federated Credential

App Registration → Certificates & secrets → Federated credentials → Add credential

Select GitHub Actions deploying Azure resources.

Configure it for the correct:

GitHub Owner / Organization
GitHub Repository
Repository ID
Branch: main
Audience: api://AzureADTokenExchange

The repository and branch must match the GitHub Actions identity.

6. Give Azure Permission

Azure Subscription → Access control (IAM) → Add role assignment

Use:

Role: Contributor
Assign access to: User, group, or service principal
Member: GitHub Actions App Registration / Service Principal

7. Add GitHub Secrets

GitHub repository → Settings → Secrets and variables → Actions

Create:

AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID

Values:

AZURE_CLIENT_ID       = Application (client) ID
AZURE_TENANT_ID       = Directory (tenant) ID
AZURE_SUBSCRIPTION_ID = Azure Subscription ID

For OIDC authentication, no Azure client secret is required.

8. Infrastructure Workflow

infra.yml creates:

Resource Group
      ↓
App Service Plan
      ↓
App Service

Example structure:

name: Create Azure Infrastructure

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  AZURE_RESOURCE_GROUP: rg-github-actions-demo
  AZURE_LOCATION: southindia
  AZURE_APP_SERVICE_PLAN: asp-github-actions-lab
  AZURE_WEBAPP_NAME: vijay-github-actions-demo

jobs:
  create-infrastructure:
    runs-on: windows-latest

    steps:
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Create Resource Group
        shell: pwsh
        run: |
          az group create `
            --name $env:AZURE_RESOURCE_GROUP `
            --location $env:AZURE_LOCATION

      - name: Create App Service Plan
        shell: pwsh
        run: |
          az appservice plan create `
            --name $env:AZURE_APP_SERVICE_PLAN `
            --resource-group $env:AZURE_RESOURCE_GROUP `
            --location $env:AZURE_LOCATION `
            --sku B1 `
            --is-linux

      - name: Create App Service
        shell: pwsh
        run: |
          az webapp create `
            --name $env:AZURE_WEBAPP_NAME `
            --resource-group $env:AZURE_RESOURCE_GROUP `
            --plan $env:AZURE_APP_SERVICE_PLAN `
            --runtime "DOTNETCORE:8.0"

Run it from:

GitHub → Actions → Create Azure Infrastructure → Run workflow

9. Deployment Workflow

deploy.yml builds and deploys the application.

name: Build and Deploy .NET App

on:
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

env:
  AZURE_RESOURCE_GROUP: rg-github-actions-demo
  AZURE_WEBAPP_NAME: vijay-github-actions-demo

jobs:

  build:
    name: Build Application
    runs-on: windows-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore Dependencies
        shell: pwsh
        run: |
          dotnet restore

      - name: Build and Publish
        shell: pwsh
        run: |
          dotnet publish -c Release -o ./publish

      - name: Upload Application Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app
          path: ./publish

  deploy:
    name: Deploy to Azure App Service
    needs: build
    runs-on: windows-latest

    steps:
      - name: Download Application Artifact
        uses: actions/download-artifact@v4
        with:
          name: app
          path: ./publish

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          package: ./publish

10. Deployment Flow

Push code to main
       ↓
GitHub Actions
       ↓
Build Job
       ↓
Checkout
       ↓
Setup .NET 8
       ↓
dotnet restore
       ↓
dotnet publish
       ↓
Upload Artifact
       ↓
Deploy Job
       ↓
Download Artifact
       ↓
Azure OIDC Login
       ↓
Deploy to App Service

needs: build makes the Deploy job wait for a successful Build job.

11. Manual vs Automatic Trigger

The deployment workflow supports both:

on:
  workflow_dispatch:
  push:
    branches:
      - main

push → deployment starts automatically after a push to main.

workflow_dispatch → deployment can be started manually from the Actions UI.

12. GitHub Actions vs Azure DevOps

Azure DevOps

GitHub Actions

Pipeline

Workflow

Stage

Job / job grouping

Job

Job

Step

Step

Agent

Runner

dependsOn

needs

condition

if

Service Connection

OIDC + Secrets / Environment

Variable

env / Variables

Publish Artifact

upload-artifact

Download Artifact

download-artifact

AzureWebApp@1

azure/webapps-deploy@v3

pool.vmImage

runs-on

13. OIDC Authentication Flow

GitHub Actions
      ↓
OIDC Token
      ↓
Microsoft Entra ID
      ↓
Federated Credential Validation
      ↓
App Registration / Service Principal
      ↓
Azure RBAC
      ↓
Azure Subscription
      ↓
Azure Resources

No long-lived Azure client secret is required.

14. Common Errors

AADSTS70025

No federated identity credential.

Fix: Create/configure the Federated Credential.

AADSTS700213

No matching federated identity record.

Check:

GitHub owner

Organization/Owner ID

Repository

Repository ID

Branch

Audience

Federated Credential configuration

No subscriptions found

OIDC authentication succeeded, but the identity has no Azure RBAC access.

Fix: Assign the required Azure role, such as Contributor, at the required scope.

Additional quota required

Example:

Current Limit (B1 VMs): 0
Amount required: 1

Azure does not have enough quota for the requested App Service Plan/SKU in that region.

Fix: Request quota or use another supported region/SKU.

15. Important Notes

runs-on: windows-latest is the GitHub Actions runner.

It does not determine whether the Azure App Service is Windows or Linux.

--is-linux creates a Linux App Service Plan.

App Service names must be globally unique.

B1 is a paid Azure App Service tier.

Keep credentials in GitHub Secrets; do not hard-code them in YAML.

Infrastructure and application deployment are separated into two workflows.

16. Current Project Flow

                GitHub Repository
                       |
             +---------+---------+
             |                   |
             v                   v
         infra.yml          deploy.yml
             |                   |
             v                   v
      Azure Infrastructure    Build .NET 8
             |                   |
             |              Upload Artifact
             |                   |
             |              Download Artifact
             |                   |
             |               OIDC Login
             |                   |
             +---------> Azure App Service
                              |
                              v
                         Application

17. Next Step

The next enhancement is:

Build
  ↓
Deploy to Staging Slot
  ↓
Test / Validate
  ↓
Approval
  ↓
Production

This will introduce GitHub Actions Environments, deployment protection rules, App Service deployment slots, and production promotion.
