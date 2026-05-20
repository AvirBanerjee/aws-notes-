# Google Cloud Deployment — Full Notes
### Cloud Run + App Engine for Python Applications

---

## 1. Google Cloud Overview

Google Cloud Platform (GCP) offers multiple ways to deploy Python applications. The two most common serverless/managed options are:

| | Cloud Run | App Engine |
|---|---|---|
| Type | Container-based | Source/config-based |
| Scaling | 0 → N (scale to zero) | 0 → N (scale to zero) |
| Control | Full (Docker) | Limited (managed runtime) |
| Config file | None (CLI flags) | `app.yaml` |
| Pricing | Per request + CPU/memory | Per instance hour |
| Best for | Microservices, APIs, custom runtimes | Web apps with standard runtimes |

---

## 2. Prerequisites & Setup

### Install Google Cloud CLI
```bash
# macOS (Homebrew)
brew install --cask google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Windows — download installer from:
# https://cloud.google.com/sdk/docs/install
```

### Authenticate and Configure
```bash
# Login to Google account
gcloud auth login

# Set your project
gcloud config set project YOUR_PROJECT_ID

# Set default region
gcloud config set run/region us-central1

# View current config
gcloud config list

# Verify active account
gcloud auth list
```

### Enable Required APIs
```bash
# Enable Cloud Run
gcloud services enable run.googleapis.com

# Enable App Engine
gcloud services enable appengine.googleapis.com

# Enable Container Registry / Artifact Registry
gcloud services enable containerregistry.googleapis.com
gcloud services enable artifactregistry.googleapis.com

# Enable Cloud Build (for auto builds)
gcloud services enable cloudbuild.googleapis.com
```

---

## 3. Python App Structure (Common)

Whether deploying to Cloud Run or App Engine, your project should look like:

```
myapp/
├── main.py               # Entry point (Flask / FastAPI / Django)
├── requirements.txt      # All Python dependencies
├── Dockerfile            # Required for Cloud Run
├── app.yaml              # Required for App Engine
├── .dockerignore         # Exclude files from Docker build
├── .gcloudignore         # Exclude files from gcloud deploy
└── .env                  # Local only — never deploy this
```

### `.dockerignore`
```
.env
__pycache__/
*.pyc
*.pyo
.git/
.gitignore
*.md
```

### `.gcloudignore`
```
.git/
.env
__pycache__/
*.pyc
node_modules/
```

---

## 4. Cloud Run — Overview

Cloud Run is a **fully managed serverless platform** that runs stateless containers. You provide a Docker image and Google handles the rest — scaling, load balancing, HTTPS, and infrastructure.

### How Cloud Run Works
```
You write code
    ↓
Build Docker image
    ↓
Push to Artifact Registry / GCR
    ↓
Deploy to Cloud Run (gcloud run deploy)
    ↓
Cloud Run assigns HTTPS URL automatically
    ↓
Requests come in → containers spin up → scale to zero when idle
```

### Key Concepts

| Concept | Description |
|---|---|
| **Service** | A deployed app with a unique URL |
| **Revision** | A new version of the service (created on each deploy) |
| **Container Instance** | A running copy of your container |
| **Concurrency** | How many requests one container handles at once (default: 80) |
| **Min/Max Instances** | Control cold starts and cost |
| **Region** | Where your service runs (pick closest to users) |

---

## 5. Deploying Python App to Cloud Run

### Step 1 — Write the App (`main.py`)

```python
# Example: FastAPI app
from fastapi import FastAPI
import os

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello from Cloud Run!"}

@app.get("/health")
def health():
    return {"status": "ok"}
```

### Step 2 — Write `requirements.txt`
```txt
fastapi
uvicorn[standard]
gunicorn
```

### Step 3 — Write the `Dockerfile`
```dockerfile
# Use official Python slim image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Cloud Run injects PORT env var — must use it
ENV PORT=8080
EXPOSE 8080

# Start command
CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", \
     "main:app", "--bind", "0.0.0.0:8080"]
```

> ⚠️ **Important**: Cloud Run injects a `PORT` environment variable (default `8080`). Your app **must** listen on this port. Never hardcode a different port.

For Flask:
```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "main:app"]
```

### Step 4 — Build and Test Locally
```bash
# Build
docker build -t my-python-app .

# Test locally
docker run -p 8080:8080 -e PORT=8080 my-python-app

# Visit http://localhost:8080
```

### Step 5 — Deploy Directly to Cloud Run (One Command)
```bash
gcloud run deploy my-python-app \
  --source . \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated
```

> `--source .` tells Cloud Run to **build the Docker image automatically** using Cloud Build — no manual push needed.

### Step 6 — Deploy via Artifact Registry (Manual Build)

```bash
# Configure Docker to use gcloud credentials
gcloud auth configure-docker us-central1-docker.pkg.dev

# Create Artifact Registry repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1

# Build image
docker build -t us-central1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-python-app:latest .

# Push image
docker push us-central1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-python-app:latest

# Deploy from Artifact Registry
gcloud run deploy my-python-app \
  --image us-central1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-python-app:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated
```

---

## 6. Cloud Run — Configuration Flags

### Common `gcloud run deploy` Flags

```bash
gcloud run deploy SERVICE_NAME \
  --image IMAGE_URL \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \       # Public access
  --port 8080 \                   # Container port
  --memory 512Mi \                # RAM (256Mi, 512Mi, 1Gi, 2Gi, 4Gi)
  --cpu 1 \                       # vCPUs (1, 2, 4, 8)
  --concurrency 80 \              # Requests per container instance
  --min-instances 0 \             # Scale to zero (cost saving)
  --max-instances 10 \            # Max containers
  --timeout 300 \                 # Request timeout in seconds
  --set-env-vars KEY=VALUE,KEY2=VALUE2   # Environment variables
```

### Set Environment Variables
```bash
gcloud run services update my-python-app \
  --region us-central1 \
  --set-env-vars \
    SECRET_KEY="your-secret",\
    DATABASE_URL="postgresql://user:pass@host/db",\
    DEBUG="false"
```

### Use Secret Manager (Production Recommended)
```bash
# Create a secret
echo -n "my-secret-value" | gcloud secrets create SECRET_KEY --data-file=-

# Grant Cloud Run access to the secret
gcloud secrets add-iam-policy-binding SECRET_KEY \
  --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Reference secret in Cloud Run
gcloud run deploy my-python-app \
  --set-secrets=SECRET_KEY=SECRET_KEY:latest
```

---

## 7. Cloud Run — Useful Commands

```bash
# List all services
gcloud run services list --region us-central1

# Describe a service (see URL, revisions, config)
gcloud run services describe my-python-app --region us-central1

# View logs
gcloud logs read --project YOUR_PROJECT_ID \
  --filter="resource.type=cloud_run_revision" \
  --limit 50

# Stream live logs
gcloud beta run services logs tail my-python-app --region us-central1

# Delete service
gcloud run services delete my-python-app --region us-central1

# Get service URL
gcloud run services describe my-python-app \
  --region us-central1 \
  --format "value(status.url)"

# List revisions
gcloud run revisions list --service my-python-app --region us-central1

# Rollback to previous revision
gcloud run services update-traffic my-python-app \
  --region us-central1 \
  --to-revisions REVISION_NAME=100
```

---

## 8. App Engine — Overview

App Engine is GCP's original PaaS offering. You define your app's configuration in `app.yaml` and deploy source code directly — Google manages the runtime, scaling, and infrastructure.

### Two Environments

| | Standard Environment | Flexible Environment |
|---|---|---|
| Language support | Python 3.8–3.12 (built-in) | Any (via Docker) |
| Scaling | Very fast (seconds) | Slower (minutes) |
| Scale to zero | Yes | No (min 1 instance) |
| Custom runtimes | No | Yes |
| Pricing | Per request | Per instance hour |
| Use case | Web apps, APIs | Custom runtimes, long requests |

---

## 9. `app.yaml` — Full Configuration Reference

`app.yaml` is the heart of App Engine deployment. It lives in your project root.

### Minimal `app.yaml` (Standard Environment)
```yaml
runtime: python311
entrypoint: gunicorn -b :$PORT main:app
```

### Full `app.yaml` (Standard Environment)
```yaml
# ---- REQUIRED ----
runtime: python311          # python38, python39, python310, python311, python312

# ---- ENTRYPOINT ----
# Command to start your app. $PORT is injected by App Engine.
entrypoint: gunicorn -b :$PORT -w 4 -k uvicorn.workers.UvicornWorker main:app

# ---- SERVICE ----
# 'default' is the main service. You can have multiple services.
service: default

# ---- ENVIRONMENT ----
env: standard               # standard or flex

# ---- INSTANCE CLASS ----
# Controls CPU and memory per instance
instance_class: F2          # F1(default), F2, F4, F4_1G

# ---- SCALING ----
automatic_scaling:
  min_idle_instances: automatic
  max_idle_instances: automatic
  min_pending_latency: automatic
  max_pending_latency: automatic
  max_instances: 10
  min_instances: 0

# ---- ENVIRONMENT VARIABLES ----
env_variables:
  FLASK_ENV: "production"
  DEBUG: "false"
  SECRET_KEY: "your-secret-key"       # use Secret Manager for real secrets

# ---- STATIC FILES (optional) ----
handlers:
  - url: /static
    static_dir: static/
  - url: /.*
    script: auto

# ---- INCLUDED/EXCLUDED FILES ----
# Files to skip during deployment
skip_files:
  - ^(.*/)?\.git/.*$
  - ^(.*/)?\.env$
  - ^(.*/)?(#.*#|.*\.pyc)$

# ---- NETWORK (optional) ----
network:
  session_affinity: true    # Route same user to same instance
```

### `app.yaml` for Flexible Environment
```yaml
runtime: python
env: flex

# Point to your Dockerfile
runtime_config:
  python_version: 3.11

entrypoint: gunicorn -b :$PORT -w 4 -k uvicorn.workers.UvicornWorker main:app

# Flex requires at least 1 instance (no scale to zero)
automatic_scaling:
  min_num_instances: 1
  max_num_instances: 5
  cool_down_period_sec: 180
  cpu_utilization:
    target_utilization: 0.6

resources:
  cpu: 1
  memory_gb: 0.5
  disk_size_gb: 10

env_variables:
  SECRET_KEY: "your-secret"
  DATABASE_URL: "your-db-url"
```

### `app.yaml` Instance Class Guide

| Class | Memory | CPU | Use Case |
|---|---|---|---|
| F1 | 256 MB | 600 MHz | Very small / dev |
| F2 | 512 MB | 1.2 GHz | Default / light apps |
| F4 | 1024 MB | 2.4 GHz | Medium workloads |
| F4_1G | 1024 MB | 2.4 GHz | Memory-heavy apps |

---

## 10. Deploying to App Engine — Step by Step

### Step 1 — Initialize App Engine (First Time Only)
```bash
gcloud app create --project YOUR_PROJECT_ID --region us-central1
```
> You only run this once per project. Region **cannot be changed** after creation.

### Step 2 — Write the App (`main.py`)
```python
from flask import Flask
import os

app = Flask(__name__)

@app.get("/")
def index():
    return {"message": "Hello from App Engine!"}

@app.get("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
```

### Step 3 — Write `requirements.txt`
```txt
flask
gunicorn
```

### Step 4 — Write `app.yaml`
```yaml
runtime: python311
entrypoint: gunicorn -b :$PORT main:app

env_variables:
  FLASK_ENV: "production"
```

### Step 5 — Deploy
```bash
gcloud app deploy
```
> This command reads `app.yaml` from the current directory and deploys.

### Deploy with Specific Version
```bash
gcloud app deploy --version v2 --no-promote
```
> `--no-promote` deploys without sending live traffic — useful for testing.

### Promote a Version to Live
```bash
gcloud app services set-traffic default \
  --splits v2=1 \
  --migrate
```

### Step 6 — Open App in Browser
```bash
gcloud app browse
```

---

## 11. App Engine — Useful Commands

```bash
# Deploy (reads app.yaml automatically)
gcloud app deploy

# Deploy specific file
gcloud app deploy app.yaml

# View running app in browser
gcloud app browse

# List all versions
gcloud app versions list

# Stream logs
gcloud app logs tail -s default

# View logs in Cloud Logging
gcloud logging read "resource.type=gae_app" --limit 50

# Stop a version (save cost)
gcloud app versions stop VERSION_ID

# Delete a version
gcloud app versions delete VERSION_ID

# Describe current service
gcloud app describe

# List services
gcloud app services list

# Traffic splitting (A/B testing)
gcloud app services set-traffic default \
  --splits v1=50,v2=50
```

---

## 12. Cloud Run vs App Engine — When to Use What

| Scenario | Use |
|---|---|
| FastAPI / async Python app | Cloud Run |
| Custom system dependencies | Cloud Run |
| Microservices architecture | Cloud Run |
| Simple Flask web app | App Engine Standard |
| No Docker knowledge needed | App Engine Standard |
| Need WebSockets | Cloud Run (Flex) |
| Simple deployment, less config | App Engine |
| Full runtime control | Cloud Run |
| Long-running background tasks | App Engine Flex or Cloud Run (increase timeout) |

---

## 13. Deploying a FastAPI App — End-to-End Example

This combines everything above into one complete workflow.

### Files Needed
```
fastapi-app/
├── main.py
├── requirements.txt
├── Dockerfile
└── app.yaml          # only if deploying to App Engine
```

### `main.py`
```python
from fastapi import FastAPI
import os

app = FastAPI(title="My Cloud API")

@app.get("/")
def root():
    name = os.environ.get("APP_NAME", "Cloud App")
    return {"message": f"Hello from {name}"}
```

### `requirements.txt`
```txt
fastapi
uvicorn[standard]
gunicorn
```

### `Dockerfile` (for Cloud Run)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8080
EXPOSE 8080
CMD ["gunicorn", "-w", "2", "-k", "uvicorn.workers.UvicornWorker", \
     "main:app", "--bind", "0.0.0.0:8080"]
```

### `app.yaml` (for App Engine)
```yaml
runtime: python311
entrypoint: gunicorn -b :$PORT -w 2 -k uvicorn.workers.UvicornWorker main:app

instance_class: F2

automatic_scaling:
  max_instances: 5
  min_instances: 0

env_variables:
  APP_NAME: "FastAPI on App Engine"
```

### Deploy to Cloud Run
```bash
gcloud run deploy fastapi-app \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars APP_NAME="FastAPI on Cloud Run" \
  --memory 512Mi \
  --min-instances 0 \
  --max-instances 5
```

### Deploy to App Engine
```bash
gcloud app deploy
```

---

## 14. Environment Variables Best Practices

### Cloud Run
```bash
# Set on deploy
gcloud run deploy SERVICE \
  --set-env-vars KEY1=VAL1,KEY2=VAL2

# Update existing service
gcloud run services update SERVICE \
  --set-env-vars KEY=VAL

# Remove a variable
gcloud run services update SERVICE \
  --remove-env-vars KEY
```

### App Engine
```yaml
# In app.yaml — fine for non-sensitive config
env_variables:
  APP_ENV: "production"
  LOG_LEVEL: "warning"
```

### Secret Manager (Both Platforms — Recommended for Secrets)
```bash
# Create secret
echo -n "super-secret-value" | \
  gcloud secrets create MY_SECRET --data-file=-

# Access in Python code
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = "projects/YOUR_PROJECT_ID/secrets/MY_SECRET/versions/latest"
secret = client.access_secret_version(request={"name": name})
value = secret.payload.data.decode("UTF-8")
```

---

## 15. Production Checklist

### Cloud Run
- [ ] Use `--min-instances 1` if you need to avoid cold starts.
- [ ] Set `--max-instances` to control cost.
- [ ] Use Secret Manager instead of plain `--set-env-vars` for secrets.
- [ ] Set `--memory` and `--cpu` based on profiling.
- [ ] Enable VPC connector if connecting to Cloud SQL or private resources.
- [ ] Use Artifact Registry instead of Container Registry (GCR is deprecated).
- [ ] Set `--concurrency` to match your app's thread/async capacity.

### App Engine
- [ ] Set `instance_class` based on memory requirements.
- [ ] Never put real secrets in `app.yaml` — use Secret Manager.
- [ ] Use `--no-promote` + manual traffic migration for safe deployments.
- [ ] Set `max_instances` to avoid runaway scaling costs.
- [ ] Add `skip_files` to exclude `.env` and cache files.
- [ ] Monitor with Cloud Monitoring and set up alerts.

### Both
- [ ] Enable Cloud Logging for centralized log management.
- [ ] Set up uptime checks in Cloud Monitoring.
- [ ] Use a custom domain with managed SSL (free with GCP).
- [ ] Set `PORT` from environment variable — never hardcode it.

---

## 16. Quick Reference Cheatsheet

```bash
# ---- SETUP ----
gcloud auth login
gcloud config set project PROJECT_ID
gcloud services enable run.googleapis.com appengine.googleapis.com

# ---- CLOUD RUN ----
gcloud run deploy APP --source . --region us-central1 --allow-unauthenticated
gcloud run services list
gcloud run services describe APP --region us-central1
gcloud run services delete APP --region us-central1
gcloud beta run services logs tail APP --region us-central1

# ---- APP ENGINE ----
gcloud app create --region us-central1
gcloud app deploy
gcloud app browse
gcloud app logs tail -s default
gcloud app versions list
gcloud app versions stop VERSION_ID

# ---- ARTIFACT REGISTRY ----
gcloud artifacts repositories create REPO --repository-format=docker --location=us-central1
gcloud auth configure-docker us-central1-docker.pkg.dev
docker tag IMAGE us-central1-docker.pkg.dev/PROJECT/REPO/IMAGE:latest
docker push us-central1-docker.pkg.dev/PROJECT/REPO/IMAGE:latest

# ---- SECRETS ----
echo -n "value" | gcloud secrets create SECRET_NAME --data-file=-
gcloud secrets versions access latest --secret=SECRET_NAME

# ---- GENERAL ----
gcloud config list
gcloud projects list
gcloud services list --enabled
```