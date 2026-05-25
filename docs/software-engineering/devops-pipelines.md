
# GitHub Actions Pipeline

<br />

Proposed 3 GitHub Action workflows to support continuous delivery with our [branching strategy](https://github.com/microsoft-sgp/csusg-dev-crew/wiki/Branching).


| Azure / GitHub Environment | Git Branch | Pipeline/Workflow Name | Approval Required | Description |
|---|---|---|---|---|
| SIT | `main` | MBS - Main Branch to SIT Environment | No | Continuous integration environment where feature branches are merged and validated through automated build, unit testing, and integration testing before release preparation. |
| UAT | `release/1.4` | RBUPP - Release Branch - UAT, Pre-production, Production | Yes | User Acceptance Testing environment used for business validation and sign-off of the scoped release version before promotion to higher environments. |
| Pre-Production | `release/1.4` | 1. RBUPP - Release Branch - UAT, Pre-production, Production <br /><br /> 2. Hotfix Branch - Pre-production, Production | Yes | Production-like staging environment used for final regression testing, deployment verification, and operational readiness checks before go-live. |
| Production | `release/1.4` | 1. RBUPP - Release Branch - UAT, Pre-production, Production <br /><br /> 2. Hotfix Branch - Pre-production, Production | Yes | Production environment |



<br />

## Main Branch to SIT Environment

```yaml
name: Main branch to SIT

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write   # required for Azure OIDC login

env:
  APP_NAME: my-app
  ACR_NAME: myacr                          # e.g. myacr (without .azurecr.io)
  ACR_LOGIN_SERVER: myacr.azurecr.io
  RESOURCE_GROUP: rg-sit
  CONTAINER_APP_NAME: my-app-sit
  CONTAINER_APP_ENV: cae-sit               # Container Apps environment name

concurrency:
  group: main-sit-deploy
  cancel-in-progress: true

jobs:
  # -------------------------
  # 1. BUILD & PUSH IMAGE
  # -------------------------
  build:
    name: Build, Test & Push Image
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.vars.outputs.image_tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: echo "install dependencies"

      - name: Run unit tests
        run: echo "run tests"

      - name: Set image tag
        id: vars
        run: echo "image_tag=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log in to ACR
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.ACR_LOGIN_SERVER }}/${{ env.APP_NAME }}:${{ steps.vars.outputs.image_tag }} .
          docker push ${{ env.ACR_LOGIN_SERVER }}/${{ env.APP_NAME }}:${{ steps.vars.outputs.image_tag }}

  # -------------------------
  # 2. DEPLOY TO SIT (AUTO)
  # -------------------------
  deploy-sit:
    name: Deploy to SIT
    needs: build
    runs-on: ubuntu-latest
    environment: sit   # no approval
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP_NAME }}
          containerAppEnvironment: ${{ env.CONTAINER_APP_ENV }}
          imageToDeploy: ${{ env.ACR_LOGIN_SERVER }}/${{ env.APP_NAME }}:${{ needs.build.outputs.image_tag }}
          targetPort: 8080
          ingress: external

      - name: Smoke Test
        run: |
          echo "Running smoke tests on SIT..."
          FQDN=$(az containerapp show -n ${{ env.CONTAINER_APP_NAME }} -g ${{ env.RESOURCE_GROUP }} --query properties.configuration.ingress.fqdn -o tsv)
          curl -fsSL "https://${FQDN}/health" || exit 1
```

<br />

## RBUPP - Release Branch - UAT, Pre-Production, Production 

```yaml
name: Release branch → UAT → pre-prod → production
on:
  push:
    branches:
      - 'release/*'

permissions:
  contents: write   # needed for tagging
  id-token: write   # needed for Azure OIDC login

env:
  APP_NAME: my-app
  ACR_NAME: myacr                 # without .azurecr.io
  ACR_LOGIN_SERVER: myacr.azurecr.io
  TARGET_PORT: 8080

jobs:
  # -------------------------
  # 1. BUILD ONCE & PUSH IMAGE
  # -------------------------
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    # Skip if the commit is a merge of a hotfix (avoid re-deploying hotfix merges to UAT)
    if: ${{ !(contains(toLowerCase(github.event.head_commit.message), 'merge') && contains(toLowerCase(github.event.head_commit.message), 'hotfix')) }}
    outputs:
      version: ${{ steps.extract_version.outputs.version }}
      image: ${{ steps.image.outputs.image }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Extract version from branch
        id: extract_version
        run: |
          BRANCH_NAME=${GITHUB_REF#refs/heads/}
          VERSION=${BRANCH_NAME#release/}
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      - name: Compute image reference
        id: image
        run: |
          SHORT_SHA=${GITHUB_SHA::7}
          IMAGE="${{ env.ACR_LOGIN_SERVER }}/${{ env.APP_NAME }}:${{ steps.extract_version.outputs.version }}-${SHORT_SHA}"
          echo "image=$IMAGE" >> $GITHUB_OUTPUT
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Log in to ACR
        run: az acr login --name ${{ env.ACR_NAME }}
      - name: Build and push Docker image
        run: |
          docker build -t ${{ steps.image.outputs.image }} .
          docker push ${{ steps.image.outputs.image }}

  # -------------------------
  # 2. DEPLOY TO UAT (Immediate / Auto-cancel)
  # -------------------------
  deploy-uat:
    name: Deploy to UAT
    needs: build
    runs-on: ubuntu-latest
    environment: uat
    concurrency:
      group: deploy-uat
      cancel-in-progress: true # Newer feature branches will auto-cancel older runs here
    env:
      RESOURCE_GROUP: rg-uat
      CONTAINER_APP_NAME: my-app-uat
      CONTAINER_APP_ENV: cae-uat
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy to Azure Container Apps (UAT)
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP_NAME }}
          containerAppEnvironment: ${{ env.CONTAINER_APP_ENV }}
          imageToDeploy: ${{ needs.build.outputs.image }}
          targetPort: ${{ env.TARGET_PORT }}
          ingress: external
      - name: Smoke Test (UAT)
        run: echo "Running basic verification tests on UAT..."

  # -------------------------
  # 3. DEPLOY TO PRE-PRODUCTION (Manual Approval / Queue Lock)
  # -------------------------
  deploy-preprod:
    name: Deploy to Pre-Prod
    needs: [build, deploy-uat]
    runs-on: ubuntu-latest
    environment: pre-production
    concurrency:
      group: deploy-pre-prod   # Shared lock namespace with HBPP hotfix workflow
      cancel-in-progress: false # Safe queuing; hotfixes won't kill pending releases
    env:
      RESOURCE_GROUP: rg-preprod
      CONTAINER_APP_NAME: my-app-preprod
      CONTAINER_APP_ENV: cae-preprod
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy to Azure Container Apps (Pre-Prod)
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP_NAME }}
          containerAppEnvironment: ${{ env.CONTAINER_APP_ENV }}
          imageToDeploy: ${{ needs.build.outputs.image }}
          targetPort: ${{ env.TARGET_PORT }}
          ingress: external
      - name: Smoke Test (Pre-Prod)
        run: echo "Running staging verification tests..."

  # -------------------------
  # 4. DEPLOY TO PRODUCTION (Manual Approval / Queue Lock)
  # -------------------------
  deploy-production:
    name: Deploy to Production
    needs: [build, deploy-preprod]
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: deploy-production # Shared lock namespace with HBPP hotfix workflow
      cancel-in-progress: false # Safe queuing; protects running or approved jobs from destruction
    env:
      RESOURCE_GROUP: rg-prod
      CONTAINER_APP_NAME: my-app-prod
      CONTAINER_APP_ENV: cae-prod
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy to Azure Container Apps (Production)
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP_NAME }}
          containerAppEnvironment: ${{ env.CONTAINER_APP_ENV }}
          imageToDeploy: ${{ needs.build.outputs.image }}
          targetPort: ${{ env.TARGET_PORT }}
          ingress: external
      - name: Post-Deployment Tagging
        run: echo "Production deployment successful."
```

<br />

## HBPP - Hotfix branch - Pre-Production, Production

```yaml
name: Hotfix branch → Pre-production → Prod

on:
  push:
    branches:
      - 'hotfix/*'   # ONLY triggers on hotfix branches

permissions:
  contents: write   # needed for tagging
  id-token: write   # needed for Azure OIDC login

env:
  APP_NAME: my-app
  ACR_NAME: myacr                       # without .azurecr.io
  ACR_LOGIN_SERVER: myacr.azurecr.io
  TARGET_PORT: 8080

jobs:
  # -------------------------
  # 1. BUILD ONCE & PUSH IMAGE
  # -------------------------
  build-and-test-hotfix:
    name: Build & Verify Hotfix
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.extract_version.outputs.version }}
      image: ${{ steps.image.outputs.image }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract hotfix version from branch
        id: extract_version
        run: |
          BRANCH_NAME=${GITHUB_REF#refs/heads/}
          VERSION=${BRANCH_NAME#hotfix/}
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Run hotfix tests
        run: echo "Running unit tests for hotfix..."

      - name: Compute image reference
        id: image
        run: |
          SHORT_SHA=${GITHUB_SHA::7}
          IMAGE="${{ env.ACR_LOGIN_SERVER }}/${{ env.APP_NAME }}:hotfix-${{ steps.extract_version.outputs.version }}-${SHORT_SHA}"
          echo "image=$IMAGE" >> $GITHUB_OUTPUT

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log in to ACR
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ steps.image.outputs.image }} .
          docker push ${{ steps.image.outputs.image }}

  # -------------------------
  # 2. DEPLOY TO PRE-PROD (MANUAL APPROVAL)
  # -------------------------
  deploy-preprod:
    name: Deploy to Pre-Prod
    needs: build-and-test-hotfix
    runs-on: ubuntu-latest
    environment: pre-production # Matched to target the exact environment used in the release pipeline
    concurrency:
      group: deploy-pre-prod    # Shares namespace with main pipeline
      cancel-in-progress: false # Safe queuing; prevents wiping out concurrent UPP pipeline states
    env:
      RESOURCE_GROUP: rg-preprod
      CONTAINER_APP_NAME: my-app-preprod
      CONTAINER_APP_ENV: cae-preprod
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Container Apps (Pre-Prod Hotfix)
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP_NAME }}
          containerAppEnvironment: ${{ env.CONTAINER_APP_ENV }}
          imageToDeploy: ${{ needs.build-and-test-hotfix.outputs.image }}
          targetPort: ${{ env.TARGET_PORT }}
          ingress: external

      - name: Smoke Test (Pre-Prod Hotfix)
        run: echo "Running staging verification tests for hotfix..."

  # -------------------------
  # 3. DEPLOY TO PRODUCTION (MANUAL APPROVAL)
  # -------------------------
  deploy-production:
    name: Deploy to Production
    needs: [build-and-test-hotfix, deploy-preprod]
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: deploy-production  # Shares namespace with main pipeline
      cancel-in-progress: false # Safe queuing; prevents dropping live or approved manual validation queues midway
    env:
      RESOURCE_GROUP: rg-prod
      CONTAINER_APP_NAME: my-app-prod
      CONTAINER_APP_ENV: cae-prod
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Container Apps (Production Hotfix)
        uses: azure/container-apps-deploy

```