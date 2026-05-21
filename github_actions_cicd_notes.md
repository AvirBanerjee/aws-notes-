# GitHub Actions CI/CD — Full Notes
### Docker Build → Push to Registry → Auto-Deploy → Secrets & Env Vars

---

## 1. GitHub Actions Overview

GitHub Actions is a CI/CD platform built into GitHub. It lets you automate workflows — build, test, and deploy code — triggered by events like `git push`, pull requests, or schedules.

### Core Concepts

| Term | Meaning |
|---|---|
| **Workflow** | An automated process defined in a `.yml` file |
| **Event** | What triggers the workflow (`push`, `pull_request`, `schedule`) |
| **Job** | A set of steps that run on the same runner |
| **Step** | A single task inside a job (run command or use an action) |
| **Action** | A reusable unit of code (from Marketplace or custom) |
| **Runner** | The VM that executes the job (`ubuntu-latest`, `windows-latest`) |
| **Secret** | Encrypted value stored in GitHub, injected at runtime |
| **Environment** | A deployment target (staging, production) with its own secrets/rules |
| **Artifact** | Files saved from a workflow (logs, build outputs) |
| **Cache** | Saved dependencies to speed up future runs |

### Workflow File Location
```
your-repo/
└── .github/
    └── workflows/
        ├── ci.yml          # Build & test
        ├── deploy.yml      # Deploy to cloud
        └── release.yml     # Release automation
```

---

## 2. Workflow File Structure

```yaml
name: CI/CD Pipeline            # Display name in GitHub UI

on:                             # TRIGGER — what starts this workflow
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:                            # GLOBAL env vars available to all jobs
  IMAGE_NAME: my-python-app
  REGISTRY: ghcr.io

jobs:                           # One or more jobs
  build:                        # Job ID (you name this)
    name: Build and Test        # Display name
    runs-on: ubuntu-latest      # Runner OS

    steps:                      # Sequential steps inside this job
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest
```

---

## 3. Triggers (`on:`)

### Common Trigger Patterns

```yaml
on:
  # Push to specific branches
  push:
    branches:
      - main
      - develop
      - "release/*"       # wildcard — matches release/v1, release/v2, etc.
    paths:
      - "src/**"          # only trigger if files in src/ changed
      - "Dockerfile"

  # Pull requests targeting main
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  # Manual trigger from GitHub UI
  workflow_dispatch:
    inputs:
      environment:
        description: "Deploy to which environment?"
        required: true
        default: "staging"
        type: choice
        options: [staging, production]

  # Scheduled (cron syntax)
  schedule:
    - cron: "0 2 * * 1"   # Every Monday at 2am UTC

  # Trigger from another workflow
  workflow_call:

  # On tag push (for releases)
  push:
    tags:
      - "v*.*.*"           # matches v1.0.0, v2.3.1, etc.
```

---

## 4. GitHub Secrets

Secrets are **encrypted values** stored in GitHub that get injected into workflows as environment variables at runtime. They are never shown in logs.

### Where to Store Secrets

| Location | Scope | Use Case |
|---|---|---|
| **Repository secrets** | Single repo | Most common — per-repo credentials |
| **Environment secrets** | Specific environment (staging/prod) | Different creds per deployment target |
| **Organization secrets** | All repos in org | Shared credentials across projects |

### Adding Secrets

**Via GitHub UI:**
```
Repository → Settings → Secrets and variables → Actions → New repository secret
```

**Via GitHub CLI:**
```bash
# Install GitHub CLI first: brew install gh

# Add a secret to a repo
gh secret set DOCKER_PASSWORD --body "your-password-here"

# Add from a file
gh secret set KUBECONFIG < ~/.kube/config

# List secrets (names only, values are hidden)
gh secret list

# Delete a secret
gh secret delete DOCKER_PASSWORD
```

### Referencing Secrets in Workflows
```yaml
steps:
  - name: Login to Docker Hub
    run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

  - name: Use secret as env var
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      SECRET_KEY: ${{ secrets.SECRET_KEY }}
    run: python deploy.py
```

>  Secrets are masked in logs — if you `echo ${{ secrets.MY_SECRET }}`, GitHub replaces it with `***`.

### Built-in Secrets (Automatic)
```yaml
# GITHUB_TOKEN — auto-generated for each workflow run
# Use for: pushing to repo, creating releases, commenting on PRs

- name: Push to GitHub Packages
  run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
```

---

## 5. Environment Variables (`env:`)

### Three Levels of Env Vars

```yaml
env:                            # 1. WORKFLOW level — available to ALL jobs
  APP_ENV: production
  IMAGE_NAME: my-app

jobs:
  deploy:
    env:                        # 2. JOB level — available to all steps in this job
      DEPLOY_REGION: us-east-1

    steps:
      - name: Run script
        env:                    # 3. STEP level — available only to this step
          SECRET_KEY: ${{ secrets.SECRET_KEY }}
        run: python script.py
```

### Expressions and Context Variables

```yaml
steps:
  - name: Print context info
    run: |
      echo "Repo:       ${{ github.repository }}"
      echo "Branch:     ${{ github.ref_name }}"
      echo "Commit SHA: ${{ github.sha }}"
      echo "Actor:      ${{ github.actor }}"
      echo "Event:      ${{ github.event_name }}"
      echo "Run ID:     ${{ github.run_id }}"
      echo "Run number: ${{ github.run_number }}"
```

### Setting Dynamic Env Vars Between Steps

```yaml
steps:
  - name: Set IMAGE_TAG from git SHA
    run: echo "IMAGE_TAG=${GITHUB_SHA::8}" >> $GITHUB_ENV

  - name: Use IMAGE_TAG in next step
    run: echo "Building image with tag $IMAGE_TAG"

  - name: Set multiple vars
    run: |
      echo "BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_ENV
      echo "VERSION=$(cat VERSION)" >> $GITHUB_ENV
```

### Output Variables (Pass Between Jobs)

```yaml
jobs:
  build:
    outputs:
      image_tag: ${{ steps.set_tag.outputs.tag }}   # expose output

    steps:
      - name: Set tag
        id: set_tag
        run: echo "tag=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT

  deploy:
    needs: build                                     # run after build job
    steps:
      - name: Use tag from build job
        run: echo "Deploying image tag ${{ needs.build.outputs.image_tag }}"
```

---

## 6. Docker Build in GitHub Actions

### Basic Docker Build Step

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3

  - name: Build Docker image
    run: docker build -t my-app:latest .
```

### Build with Tags and Labels

```yaml
- name: Extract metadata for Docker
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: |
      ghcr.io/${{ github.repository }}
    tags: |
      type=sha,prefix=sha-,format=short        # sha-a1b2c3d
      type=ref,event=branch                    # branch name
      type=semver,pattern={{version}}          # v1.2.3 (on tag push)
      type=semver,pattern={{major}}.{{minor}}  # v1.2
      type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

### Build Cache (Speed Up Builds)

```yaml
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha                # use GitHub Actions cache
    cache-to: type=gha,mode=max
```

### Multi-Platform Build (AMD64 + ARM64)

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build multi-platform image
  uses: docker/build-push-action@v5
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
```

---

## 7. Push to Container Registries

### 7.1 — GitHub Container Registry (GHCR)

GHCR is GitHub's own container registry at `ghcr.io`. Free for public repos, included in GitHub plans for private.

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Log in to GHCR
    uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}    # auto-provided, no setup needed

  - name: Build and push to GHCR
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: ghcr.io/${{ github.repository_owner }}/my-app:latest
```

### 7.2 — Docker Hub

**Secrets to add:**
- `DOCKERHUB_USERNAME` — your Docker Hub username
- `DOCKERHUB_TOKEN` — Docker Hub access token (not your password)
  > Create at: Docker Hub → Account Settings → Security → New Access Token

```yaml
steps:
  - name: Log in to Docker Hub
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKERHUB_USERNAME }}
      password: ${{ secrets.DOCKERHUB_TOKEN }}

  - name: Build and push to Docker Hub
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: |
        ${{ secrets.DOCKERHUB_USERNAME }}/my-app:latest
        ${{ secrets.DOCKERHUB_USERNAME }}/my-app:${{ github.sha }}
```

### 7.3 — AWS Elastic Container Registry (ECR)

**Secrets to add:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`

```yaml
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: ${{ secrets.AWS_REGION }}

  - name: Log in to Amazon ECR
    id: login-ecr
    uses: aws-actions/amazon-ecr-login@v2

  - name: Build and push to ECR
    env:
      ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      IMAGE_TAG: ${{ github.sha }}
    run: |
      docker build -t $ECR_REGISTRY/my-app:$IMAGE_TAG .
      docker push $ECR_REGISTRY/my-app:$IMAGE_TAG
      echo "IMAGE=$ECR_REGISTRY/my-app:$IMAGE_TAG" >> $GITHUB_ENV
```

### 7.4 — Azure Container Registry (ACR)

**Secrets to add:**
- `AZURE_CLIENT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_TENANT_ID`
- `ACR_LOGIN_SERVER` (e.g. `myregistry.azurecr.io`)

```yaml
steps:
  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}
      # AZURE_CREDENTIALS is a JSON object with clientId, clientSecret, tenantId, subscriptionId

  - name: Log in to ACR
    uses: docker/login-action@v3
    with:
      registry: ${{ secrets.ACR_LOGIN_SERVER }}
      username: ${{ secrets.AZURE_CLIENT_ID }}
      password: ${{ secrets.AZURE_CLIENT_SECRET }}

  - name: Build and push to ACR
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}
```

### 7.5 — GCP Artifact Registry

**Secrets to add:**
- `GCP_SA_KEY` — Service Account JSON key (base64 encoded)
- `GCP_PROJECT_ID`

```yaml
steps:
  - name: Authenticate to GCP
    uses: google-github-actions/auth@v2
    with:
      credentials_json: ${{ secrets.GCP_SA_KEY }}

  - name: Set up Cloud SDK
    uses: google-github-actions/setup-gcloud@v2

  - name: Configure Docker for Artifact Registry
    run: gcloud auth configure-docker us-central1-docker.pkg.dev

  - name: Build and push to Artifact Registry
    run: |
      IMAGE=us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/my-app:${{ github.sha }}
      docker build -t $IMAGE .
      docker push $IMAGE
```

---

## 8. Auto-Deploy Workflows

### 8.1 — Deploy to AWS ECS (Elastic Container Service)

**Secrets:** `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/my-app:$IMAGE_TAG .
          docker push $ECR_REGISTRY/my-app:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/my-app:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download ECS task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition my-task \
            --query taskDefinition > task-definition.json

      - name: Update ECS task definition with new image
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: my-container
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy to ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: my-ecs-service
          cluster: my-cluster
          wait-for-service-stability: true
```

### 8.2 — Deploy to Azure App Service

**Secrets:** `AZURE_WEBAPP_PUBLISH_PROFILE` or `AZURE_CREDENTIALS`

```yaml
name: Deploy to Azure App Service

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Log in to ACR
        uses: docker/login-action@v3
        with:
          registry: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_SECRET }}

      - name: Build and push image
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }} .
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: my-python-app
          images: ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}
```

### 8.3 — Deploy to GCP Cloud Run

**Secrets:** `GCP_SA_KEY`, `GCP_PROJECT_ID`

```yaml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]

env:
  SERVICE: my-python-app
  REGION: us-central1

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      - name: Build and push image
        run: |
          IMAGE=${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/${{ env.SERVICE }}:${{ github.sha }}
          docker build -t $IMAGE .
          docker push $IMAGE
          echo "IMAGE=$IMAGE" >> $GITHUB_ENV

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE }} \
            --image $IMAGE \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated \
            --set-env-vars APP_ENV=production \
            --set-secrets SECRET_KEY=SECRET_KEY:latest \
            --memory 512Mi \
            --min-instances 0 \
            --max-instances 10
```

### 8.4 — Deploy via SSH to a VPS/Server

**Secrets:** `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`

```yaml
name: Deploy via SSH

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myapp
            docker pull ghcr.io/${{ github.repository }}:latest
            docker-compose down
            docker-compose up -d
            docker image prune -f
```

---

## 9. Full CI/CD Pipeline — Complete Example

This is a production-ready pipeline that:
1. Runs tests on every push and PR
2. Builds and pushes Docker image on merge to `main`
3. Deploys automatically to staging then production

```yaml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:

  # ── JOB 1: Test ────────────────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    services:
      # Spin up a PostgreSQL container for tests
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"             # cache pip dependencies

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          SECRET_KEY: test-secret-key
        run: |
          pytest --cov=. --cov-report=xml -v

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  # ── JOB 2: Build & Push ────────────────────────────────────
  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: test                    # only runs if test job passes
    if: github.ref == 'refs/heads/main'   # only on main branch

    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,format=short
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── JOB 3: Deploy to Staging ──────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging           # uses staging environment secrets

    steps:
      - name: Deploy to staging
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: my-app-staging
          region: us-central1
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          env_vars: |
            APP_ENV=staging
          secrets: |
            SECRET_KEY=SECRET_KEY:latest
            DATABASE_URL=DB_URL_STAGING:latest

  # ── JOB 4: Deploy to Production ───────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production        # requires manual approval (configure in GitHub)

    steps:
      - name: Deploy to production
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: my-app-production
          region: us-central1
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          env_vars: |
            APP_ENV=production
          secrets: |
            SECRET_KEY=SECRET_KEY:latest
            DATABASE_URL=DB_URL_PROD:latest
```

---

## 10. Environments (Staging vs Production)

GitHub Environments let you define **separate secrets** and **protection rules** per deployment target.

### Setup (GitHub UI)
```
Repository → Settings → Environments → New environment
→ Name: "staging" or "production"
→ Add environment secrets (separate from repo secrets)
→ Add protection rules:
   - Required reviewers (manual approval before deploy)
   - Wait timer (delay before deploy runs)
   - Deployment branches (only deploy from main)
```

### Use in Workflow
```yaml
jobs:
  deploy:
    environment: production       # reference the environment by name
    steps:
      - run: echo ${{ secrets.DB_PASSWORD }}  # uses production-specific secret
```

---

## 11. Reusable Workflow Patterns

### Conditional Steps
```yaml
steps:
  - name: Only on main branch
    if: github.ref == 'refs/heads/main'
    run: echo "This is main"

  - name: Only on pull requests
    if: github.event_name == 'pull_request'
    run: echo "This is a PR"

  - name: Only on tag push
    if: startsWith(github.ref, 'refs/tags/v')
    run: echo "This is a release tag"
```

### Matrix Builds (Test on Multiple Python Versions)
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt && pytest
```

### Job Dependencies
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    ...

  build:
    needs: test               # runs after test
    ...

  deploy-staging:
    needs: build              # runs after build
    ...

  deploy-prod:
    needs: [build, deploy-staging]   # runs after both
    ...
```

### Cache Dependencies
```yaml
- name: Cache pip packages
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

---

## 12. Secrets Security Best Practices

```
 Use repository secrets for repo-specific credentials
 Use environment secrets for staging/production separation
 Use GITHUB_TOKEN where possible (auto-generated, no setup)
 Use OIDC (Workload Identity Federation) instead of long-lived keys for AWS/GCP/Azure
  Rotate secrets regularly
  Never log or echo secrets — GitHub masks them but avoid it anyway
 Use least-privilege IAM roles for cloud service accounts
 Audit secret access in the GitHub Security tab

 Never hardcode secrets in workflow files
 Never store secrets in env vars in plain text in workflow files
 Never commit .env files
 Do not use organization-wide secrets for sensitive prod credentials
```

### OIDC — No Long-Lived Credentials (Best Practice)

Instead of storing AWS/GCP/Azure keys as secrets, use **OIDC** to federate identity directly from GitHub.

#### AWS with OIDC
```yaml
- name: Configure AWS via OIDC (no stored keys)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1

# Required: set up the IAM OIDC identity provider in AWS
# and create an IAM role with a trust policy for GitHub Actions
```

#### GCP with OIDC (Workload Identity Federation)
```yaml
- name: Authenticate to GCP via OIDC
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service_account: github-actions@PROJECT_ID.iam.gserviceaccount.com
```

---

## 13. Debugging Workflows

```yaml
# Enable step debug logging
# Set secret: ACTIONS_STEP_DEBUG = true

# Enable runner debug logging
# Set secret: ACTIONS_RUNNER_DEBUG = true

# Print all env vars (for debugging — remove in production)
- name: Debug env
  run: env | sort

# Print context
- name: Print GitHub context
  run: echo '${{ toJSON(github) }}'

# Manually re-run a failed job
# GitHub UI → Actions → Select workflow run → Re-run failed jobs
```

---

## 14. Quick Reference Cheatsheet

### Common Actions
```yaml
uses: actions/checkout@v4               # Clone repo
uses: actions/setup-python@v5           # Set up Python
uses: docker/setup-buildx-action@v3     # Set up Docker Buildx
uses: docker/login-action@v3            # Docker registry login
uses: docker/build-push-action@v5       # Build + push Docker image
uses: docker/metadata-action@v5         # Generate tags/labels
uses: actions/cache@v4                  # Cache dependencies
uses: actions/upload-artifact@v4        # Save files from workflow
uses: actions/download-artifact@v4      # Load saved files
uses: aws-actions/configure-aws-credentials@v4   # AWS auth
uses: azure/login@v1                    # Azure auth
uses: google-github-actions/auth@v2     # GCP auth
```

### Context Variables
```yaml
${{ github.repository }}      # owner/repo
${{ github.ref_name }}        # branch or tag name
${{ github.sha }}             # full commit SHA
${{ github.actor }}           # user who triggered the workflow
${{ github.event_name }}      # push, pull_request, etc.
${{ github.run_id }}          # unique workflow run ID
${{ github.run_number }}      # incrementing run number
${{ runner.os }}              # Linux, Windows, macOS
${{ secrets.MY_SECRET }}      # access a secret
${{ env.MY_VAR }}             # access a workflow env var
${{ needs.JOB_ID.outputs.KEY }}  # output from another job
```

### Workflow Status Badges (Add to README)
```markdown
![CI](https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg)
![Deploy](https://github.com/OWNER/REPO/actions/workflows/deploy.yml/badge.svg?branch=main)
```
