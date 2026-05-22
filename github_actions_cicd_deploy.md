# Simplest FastAPI App — Deploy to AWS (GUI Steps)
### Step-by-step using GitHub, Docker Hub, and AWS Console

---

## What You Will Build

A tiny FastAPI app with one route:
```
GET /health → {"status": "ok"}
```
Deployed live on the internet using AWS.

---

## Tools Used (All Free Tier)

| Tool | Purpose |
|---|---|
| GitHub | Store your code |
| Docker Hub | Store your Docker image |
| AWS App Runner | Host your app (simplest AWS option) |
| GitHub Actions | Auto-deploy on every push |

> AWS App Runner is used here instead of ECS — it is **much simpler** for beginners.  
> No clusters, no task definitions, no subnets needed.

---

## Part 1 — Create the App Files

Create a folder on your computer called `fastapi-app`.  
Inside it, create these 3 files:

---

### File 1 — `main.py`
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

---

### File 2 — `requirements.txt`
```
fastapi
uvicorn
```

---

### File 3 — `Dockerfile`
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

Your folder should look like this:
```
fastapi-app/
├── main.py
├── requirements.txt
└── Dockerfile
```

---

## Part 2 — Push Code to GitHub

### Step 1 — Create a GitHub account
Go to **https://github.com** → Sign up (if you don't have one)

### Step 2 — Create a new repository
1. Click the **+** button (top right) → **New repository**
2. Name it: `fastapi-app`
3. Set to **Public**
4. Click **Create repository**

### Step 3 — Upload your files
1. On the new repo page, click **uploading an existing file**
2. Drag and drop all 3 files (`main.py`, `requirements.txt`, `Dockerfile`)
3. Scroll down → Click **Commit changes**

Your code is now on GitHub ✅

---

## Part 3 — Create a Docker Hub Account & Token

### Step 1 — Create Docker Hub account
Go to **https://hub.docker.com** → Sign up

### Step 2 — Create an Access Token (used instead of password)
1. Click your **profile icon** (top right) → **Account Settings**
2. Click **Security** in the left menu
3. Click **New Access Token**
4. Name it: `github-actions`
5. Permission: **Read, Write, Delete**
6. Click **Generate**
7. **Copy the token** — you will not see it again

---

## Part 4 — Add Secrets to GitHub

Your GitHub Actions workflow needs Docker Hub credentials.

### Steps
1. Go to your `fastapi-app` repo on GitHub
2. Click **Settings** (top menu of the repo)
3. In the left sidebar → **Secrets and variables** → **Actions**
4. Click **New repository secret** and add these two:

| Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | your Docker Hub username |
| `DOCKERHUB_TOKEN` | the token you copied in Part 3 |

---

## Part 5 — Add GitHub Actions Workflow

### Step 1 — Create the workflow file on GitHub
1. In your repo, click **Add file** → **Create new file**
2. In the filename box type exactly:
   ```
   .github/workflows/deploy.yml
   ```
   (GitHub will auto-create the folders)

### Step 2 — Paste this content into the file
```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/fastapi-app:latest
```

### Step 3 — Commit the file
Scroll down → Click **Commit new file**

### Step 4 — Watch it run
1. Click the **Actions** tab in your repo
2. You will see a workflow running
3. Click it to watch the steps live
4. Wait for the green ✅ checkmark

When it finishes, your image is on Docker Hub ✅

### Verify on Docker Hub
Go to **https://hub.docker.com** → **Repositories**  
You should see `your-username/fastapi-app`

---

## Part 6 — Deploy on AWS App Runner (GUI)

AWS App Runner pulls your Docker image and hosts it — no server setup needed.

### Step 1 — Create an AWS account
Go to **https://aws.amazon.com** → Create account (requires credit card, but Free Tier is free)

### Step 2 — Open App Runner
1. Log in to **AWS Console** → https://console.aws.amazon.com
2. In the search bar at the top, type **App Runner**
3. Click **AWS App Runner**

### Step 3 — Create a service
1. Click **Create service**

### Step 4 — Configure Source
1. Source: select **Container registry**
2. Provider: select **Docker Hub** (Public)
3. Image URI: type your image name exactly as:
   ```
   your-dockerhub-username/fastapi-app:latest
   ```
4. Deployment trigger: select **Automatic**
   > This means every new image push will auto-redeploy
5. Click **Next**

### Step 5 — Configure Service Settings
1. Service name: `fastapi-app`
2. CPU: `0.25 vCPU`
3. Memory: `0.5 GB`
4. Port: `8000`
5. Leave everything else as default
6. Click **Next**

### Step 6 — Review and Create
1. Review the settings
2. Click **Create and deploy**
3. Wait 2–3 minutes for the status to show **Running**

### Step 7 — Get your live URL
On the service page you will see a URL like:
```
https://xxxxxxxxxx.us-east-1.awsapprunner.com
```

### Step 8 — Test it
Open your browser and go to:
```
https://xxxxxxxxxx.us-east-1.awsapprunner.com/health
```

You should see:
```json
{"status": "ok"}
```

Your app is live on the internet ✅

---

## Part 7 — Auto-Deploy Test

Now every time you push code to GitHub, it will automatically redeploy.

### Test it:
1. Go to your GitHub repo
2. Click on `main.py`
3. Click the **pencil icon** (Edit)
4. Change `"ok"` to `"healthy"`
   ```python
   return {"status": "healthy"}
   ```
5. Click **Commit changes**

### What happens automatically:
```
You commit to main
      ↓
GitHub Actions runs (builds new Docker image)
      ↓
New image pushed to Docker Hub
      ↓
AWS App Runner detects new image
      ↓
App redeployes automatically
      ↓
/health returns {"status": "healthy"}
```

Check the **Actions** tab to watch it build, then visit your URL again.

---

## Summary — What You Did

| Step | What you did |
|---|---|
| Part 1 | Created 3 files: `main.py`, `requirements.txt`, `Dockerfile` |
| Part 2 | Pushed code to GitHub |
| Part 3 | Created Docker Hub account + access token |
| Part 4 | Added 2 secrets to GitHub (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) |
| Part 5 | Added `deploy.yml` — GitHub Actions builds and pushes Docker image on every push |
| Part 6 | Created AWS App Runner service — pulls image and hosts it with a public URL |
| Part 7 | Confirmed auto-deploy works end to end |