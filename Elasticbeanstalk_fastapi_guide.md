# AWS Elastic Beanstalk — Theory & Deployment Guide
### FastAPI Employee Management API — Step-by-Step

> **Course:** Cloud Computing | **Stack:** FastAPI · SQLAlchemy · JWT · AWS

---

## Table of Contents

1. [Introduction to Elastic Beanstalk](#1-introduction-to-elastic-beanstalk)
2. [IaaS → PaaS → Serverless Evolution](#2-iaas--paas--serverless-evolution)
3. [Elastic Beanstalk Architecture & Components](#3-elastic-beanstalk-architecture--components)
4. [Supported Platforms & Pricing](#4-supported-platforms--pricing)
5. [Your FastAPI Application Overview](#5-your-fastapi-application-overview)
6. [Pre-Deployment Fixes (Critical!)](#6-pre-deployment-fixes-critical)
7. [Final Project Structure](#7-final-project-structure)
8. [Deployment — AWS Console](#8-deployment--aws-console-step-by-step)
9. [Deployment — EB CLI](#9-deployment--eb-cli-command-line)
10. [Environment Variables & Configuration](#10-environment-variables--configuration)
11. [Testing the Deployed API](#11-testing-the-deployed-api)
12. [Debugging & Common Errors](#12-debugging--common-errors)
13. [Updating & Redeploying](#13-updating--redeploying)
14. [Elastic Beanstalk vs AWS Lambda](#14-elastic-beanstalk-vs-aws-lambda)
15. [Summary & Next Steps](#15-summary--next-steps)

---

## 1. Introduction to Elastic Beanstalk

### What is AWS Elastic Beanstalk?

AWS Elastic Beanstalk is a fully managed **Platform-as-a-Service (PaaS)** offering from Amazon Web Services. It allows developers to deploy and scale web applications and services **without managing the underlying infrastructure**.

You simply upload your application code, and Beanstalk automatically handles:
- Deployment & provisioning
- Load balancing
- Auto-scaling
- Application health monitoring
- OS patching & updates

### Key Benefits

| Benefit | Description |
|---|---|
| **Fast Deployment** | Deploy in minutes — no server setup required |
| **No Infrastructure Management** | AWS manages OS patching and server updates |
| **Auto Scaling** | Automatically scales up/down based on traffic |
| **Full Control** | You retain full access to all underlying AWS resources |
| **Free to Use** | No extra charge for Beanstalk itself — pay only for EC2, RDS used |
| **Multi-Platform** | Supports Python, Node.js, Java, .NET, PHP, Ruby, Go, Docker |

>  **Elastic Beanstalk is ideal for developers who want to focus on writing code, not managing servers.**

---

## 2. IaaS → PaaS → Serverless Evolution

Understanding where Elastic Beanstalk fits in the cloud service model is essential before deploying any application.

```
EC2  (Students already know this!)
 └── "You manage everything: OS, runtime, scaling, servers"

Elastic Beanstalk
 └── "You just upload code → AWS manages the rest (PaaS)"

AWS Lambda
 └── "No servers at all → You only write functions (Serverless)"
```

### Comparison Table

| Aspect | IaaS (EC2) | PaaS (Beanstalk) | Serverless (Lambda) |
|---|---|---|---|
| **You manage** | OS, runtime, scaling, app | Only app code | Only function logic |
| **AWS manages** | Hardware only | OS, scaling, LB, runtime | Everything |
| **Server concept** | Always-on VM | EC2 under the hood | No servers at all |
| **Scaling** | Manual / configure | Automatic | Instant automatic |
| **Pricing** | Per hour (running) | Per hour (EC2 cost) | Per request |
| **Idle cost** | Yes | Yes | No |
| **Deploy method** | SSH + manual | ZIP upload / EB CLI | ZIP / inline code |
| **Best for** | Full control, legacy apps | Web APIs, backends | Event-driven, microservices |


---

## 3. Elastic Beanstalk Architecture & Components

### Core Components

| Component | Description |
|---|---|
| **Application** | A logical container — the top-level grouping. One application holds multiple environments. |
| **Environment** | A running version of your app. You can have `production`, `staging`, `dev` environments. |
| **Application Version** | A specific ZIP file you uploaded. Environments point to one version at a time. |
| **Environment Tier** | Web Server (handles HTTP) or Worker (processes background tasks from SQS). |
| **Platform** | The language runtime — e.g. Python 3.11 on Amazon Linux 2023. |
| **EC2 Instance(s)** | The actual virtual machine(s) running your app. Auto-managed by Beanstalk. |
| **Load Balancer** | Distributes HTTP traffic across multiple EC2 instances. |
| **Auto Scaling Group** | Automatically adds/removes EC2 instances based on CPU/traffic thresholds. |
| **Security Group** | Firewall rules auto-created by Beanstalk (same concept students know from EC2!). |
| **S3 Bucket** | Beanstalk stores your uploaded application ZIP files here automatically. |
| **CloudWatch** | Metrics and logs — CPU usage, request counts, error rates. |

### Request Flow

```
USER / Browser
      │
      ▼
Elastic Load Balancer  (port 80 / 443)
      │
      ▼
EC2 Instance(s)  ──  Auto Scaling Group
  └── Gunicorn + Uvicorn
        └── FastAPI App (your code)
              │
              ▼
        SQLite / RDS Database
```



---

## 4. Supported Platforms & Pricing

### Supported Platforms

| Platform | Versions | Common Use |
|---|---|---|
| **Python** | 3.8, 3.9, 3.11, 3.12 | FastAPI, Django, Flask |
| **Node.js** | 16, 18, 20 | Express, NestJS, Next.js |
| **Java** | 8, 11, 17, 21 | Spring Boot |
| **.NET** | 6, 7, 8 | ASP.NET Core |
| **PHP** | 8.1, 8.2, 8.3 | Laravel, WordPress |
| **Ruby** | 3.0, 3.1, 3.2 | Ruby on Rails |
| **Go** | 1.x | Go web servers |
| **Docker** | Single / Multi-container | Any containerized app |

### Pricing Model

> ✅ **Elastic Beanstalk itself is completely FREE.**

You only pay for the underlying AWS resources it provisions:

| Resource | Cost |
|---|---|
| **EC2 t3.micro** | Free tier — 750 hrs/month |
| **Load Balancer** | ~$18/month if enabled |
| **RDS Database** | Varies by instance type |
| **Data Transfer** | First 1 GB/month free |

> 💡 **For learning/demos:** Use `t3.micro` with a single instance (no load balancer) to stay within AWS Free Tier.

---

## 5. Your FastAPI Application Overview

The application being deployed is a full-featured **Employee Management REST API** built with FastAPI. It includes JWT authentication, SQLite via SQLAlchemy, and full CRUD operations.

### Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Web Framework** | FastAPI | REST API, Swagger UI auto-generation |
| **ORM** | SQLAlchemy | Database models and queries |
| **Database** | SQLite (app.db) | Lightweight embedded database |
| **Auth** | JWT (python-jose) | Stateless token-based authentication |
| **Password Hash** | passlib + bcrypt | Secure password storage |
| **Schema** | Pydantic v2 | Request/Response validation |
| **CORS** | FastAPI Middleware | Allow frontend cross-origin requests |
| **Server** | Gunicorn + Uvicorn | ASGI production server |

### API Endpoints

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| `POST` | `/api/v1/register` | No | Register new user |
| `POST` | `/api/v1/login` | No | Login, returns JWT token |
| `GET` | `/api/v1/dashboard` | ✅ Yes | Get user stats |
| `POST` | `/api/v1/employ` | ✅ Yes | Create new employee |
| `GET` | `/api/v1/employs` | ✅ Yes | List all employees |
| `GET` | `/api/v1/employ/{id}` | ✅ Yes | Get single employee |
| `PUT` | `/api/v1/employ/{id}` | ✅ Yes | Update employee |
| `DELETE` | `/api/v1/employ/{id}` | ✅ Yes | Delete employee |

---

## 6. Pre-Deployment Fixes (Critical!)

> ⚠️ **4 problems found in the project that WILL cause deployment failure if not fixed.**

### Problem Summary

| # | Problem | Impact | Fix |
|---|---|---|---|
| 1 | File named `main.py` | Beanstalk can't find app entry point | Add `Procfile` |
| 2 | `requiremnts.txt` (typo) | Dependencies not installed | Rename to `requirements.txt` |
| 3 | `sqlite:///./app.db` path | Permission error on Beanstalk | Change to `sqlite:////tmp/app.db` |
| 4 | `venv/` included in ZIP | Bloated ZIP, dependency conflicts | Exclude from ZIP |

---

### Fix 1 — Add a `Procfile`

Create a new file named `Procfile` (no extension) in your project root:

```
web: gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:application --bind 0.0.0.0:8000
```

Also add this line at the **very bottom** of `main.py`:

```python
# Add at the very bottom of main.py
application = app   # Beanstalk and Gunicorn look for this variable name
```

---

### Fix 2 — Rename requirements file

Rename `requiremnts.txt` → `requirements.txt`

Make sure it contains all required packages:

```txt
fastapi
uvicorn
gunicorn
sqlalchemy
python-jose[cryptography]
passlib[bcrypt]
python-multipart
```

> ⚠️ `python-multipart` is **required** for `OAuth2PasswordRequestForm` (your `/login` endpoint). Without it you'll get a `422 Unprocessable Entity` error.

---

### Fix 3 — Fix the SQLite Path

On Beanstalk, the working directory can have permission restrictions. Use `/tmp` which is always writable:

```python
import os

# ❌ Original (can fail on Beanstalk):
# DATABASE_URL = "sqlite:///./app.db"

# ✅ Fixed — uses absolute /tmp path:
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:////tmp/app.db")
# Note: 4 slashes = sqlite:// + /tmp/app.db (absolute path)
```

> 💡 This also allows you to override with an RDS URL via environment variable later.

---

### Fix 4 — Exclude venv from ZIP

Never include these folders/files in your deployment ZIP:

```
❌ venv/           → Your local virtual environment
❌ __pycache__/    → Compiled Python bytecode
❌ .git/           → Git repository data
❌ app.db          → Local SQLite database
❌ Dockerfile      → Not needed for Python platform
```

---

### Add Health Check Endpoints

Beanstalk pings your app's root endpoint to verify it's running. Add these to `main.py`:

```python
@app.get("/")
def root():
    return {"status": "healthy", "service": "Employee Management API"}

@app.get("/health")
def health_check():
    return {"status": "ok"}
```

---

## 7. Final Project Structure

After applying all fixes, your project should look like this:

```
BACKEND/
│
├── main.py              ✅  INCLUDE  (application = app added at bottom)
├── Procfile             ✅  INCLUDE  (NEW FILE — tells Beanstalk how to start)
├── requirements.txt     ✅  INCLUDE  (RENAMED — was 'requiremnts.txt')
├── .gitignore           ✅  INCLUDE  (optional but good practice)
│
├── venv/                ❌  EXCLUDE
├── __pycache__/         ❌  EXCLUDE
├── .git/                ❌  EXCLUDE
├── app.db               ❌  EXCLUDE
└── Dockerfile           ❌  EXCLUDE
```

### Creating the ZIP on Windows

```
1. Open BACKEND folder in File Explorer
2. Hold Ctrl and click to select:
   - main.py
   - Procfile
   - requirements.txt
3. Right-click → Send to → Compressed (zipped) folder
4. Name it: employee-api.zip
```

Or using Git Bash / terminal:

```bash
zip employee-api.zip main.py Procfile requirements.txt
```

> ⚠️ **Common mistake:** Students zip the entire BACKEND folder, which includes `venv/` inside. Always ZIP the files directly, not the containing folder.

---

## 8. Deployment — AWS Console (Step-by-Step)

> Best for the first deployment demo — students can see every setting visually.

### STEP 1 — Navigate to Elastic Beanstalk

```
1. Go to https://console.aws.amazon.com
2. Search for "Elastic Beanstalk" in the search bar
3. Click on Elastic Beanstalk in the results
4. Click the orange "Create Application" button
```

### STEP 2 — Configure Application Name

```
Application name:   employee-management-api
Environment name:   employee-management-api-env  (auto-filled)
Domain:             leave as auto-generated
```

### STEP 3 — Choose Platform

```
Platform type:      Managed platform
Platform:           Python
Platform branch:    Python 3.11 running on 64bit Amazon Linux 2023
Platform version:   (Recommended)
```

### STEP 4 — Upload Application Code

```
Application code:   ✅ Upload your code
Version label:      v1.0
Click "Choose file" → Select employee-api.zip
Wait for green checkmark ✅
```

### STEP 5 — Configure Service Access

```
Service role:           Create and use new service role
                        Name: aws-elasticbeanstalk-service-role
EC2 key pair:           ✅ Select your existing key pair
                        (same .pem file from your EC2 lessons!)
EC2 instance profile:   Create new → aws-elasticbeanstalk-ec2-role
```

### STEP 6 — Configure Instance

```
Instance type:    t3.micro  (free tier eligible)
Root volume:      8 GB (default)
```

### STEP 7 — Review and Submit

```
1. Click "Skip to review"
2. Review all settings
3. Click the orange "Submit" button
4. Watch the Events tab — deployment takes 3-5 minutes ⏳
```

**Events students will see:**
```
✅ Created security group
✅ Created Auto Scaling group
✅ Launched EC2 instance    ← "See! Same EC2 from our EC2 class!"
✅ Environment health: Ok
```

### STEP 8 — Verify Deployment

```
1. Wait for environment health: Ok (green)
2. Click the generated URL
3. Add /docs → See Swagger UI ✅
```

---

## 9. Deployment — EB CLI (Command Line)


### Installation

```bash
# Install EB CLI
pip install awsebcli

# Verify
eb --version
# EB CLI 3.x.x (Python 3.x.x)

# Configure AWS credentials (if not done)
aws configure
# AWS Access Key ID: ****
# AWS Secret Access Key: ****
# Default region: us-east-1
# Default output format: json
```

### Initialize and First Deploy

```bash
# Navigate to your project
cd BACKEND

# Step 1: Initialize (run once per project)
eb init
# Prompts:
#   Select region       → your preferred region
#   Application name    → employee-management-api
#   Platform            → Python
#   Python version      → Python 3.11
#   CodeCommit          → No
#   SSH keypair         → Yes → select your key

# Step 2: Create environment + deploy (first time)
eb create employee-management-env
# Takes 3-5 minutes — provisions EC2, SG, ASG, etc.

# Step 3: Open app in browser
eb open
```

### Common EB CLI Commands

```bash
# Deploy code changes
eb deploy

# Deploy with version label
eb deploy --label "v1.1-fix-health-endpoint"

# Check environment status
eb status

# View logs
eb logs

# SSH into the EC2 instance
eb ssh

# View environment variables
eb printenv

# Set environment variables
eb setenv KEY=value KEY2=value2

# View deployment history
eb appversion

# Roll back to previous version
eb deploy --version v1.0

# Terminate environment (stops billing)
eb terminate employee-management-env
```

---

## 10. Environment Variables & Configuration

> ⚠️ Never hardcode secrets like `SECRET_KEY` in your code. Use environment variables.

### Setting Variables — Console

```
Elastic Beanstalk Console
  → Your Environment
  → Configuration
  → Software  (click Edit)
  → Environment properties section
  → Add:
       SECRET_KEY     = your-super-secret-key-here
       DATABASE_URL   = sqlite:////tmp/app.db
       ENVIRONMENT    = production
  → Click Apply
```

### Setting Variables — EB CLI

```bash
eb setenv SECRET_KEY=your-super-secret-key DATABASE_URL=sqlite:////tmp/app.db

# View all current environment variables
eb printenv
```

### Update main.py to Use Environment Variables

```python
import os

# Replace hardcoded values:
SECRET_KEY = os.environ.get("SECRET_KEY", "fallback-dev-key-change-in-prod")
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:////tmp/app.db")

# The rest of your code stays the same
```

---

## 11. Testing the Deployed API

Once deployed, your URL will look like:
```
http://employee-management-api.us-east-1.elasticbeanstalk.com
```

### Access Swagger UI

```
http://your-url.elasticbeanstalk.com/docs    → Swagger UI
http://your-url.elasticbeanstalk.com/redoc   → ReDoc UI
```

### Test Flow (in order)

| Step | Action | Endpoint | Notes |
|---|---|---|---|
| 1 | Register a user | `POST /api/v1/register` | `{"fullname":"John","email":"j@test.com","password":"pass123"}` |
| 2 | Login | `POST /api/v1/login` | Form: username + password → returns JWT token |
| 3 | Authorize | Swagger 🔒 button | Paste the `access_token` value |
| 4 | Dashboard | `GET /api/v1/dashboard` | Returns fullname, email, employee count |
| 5 | Create employee | `POST /api/v1/employ` | Send employee JSON body |
| 6 | List employees | `GET /api/v1/employs` | Returns array of all employees |
| 7 | Get employee | `GET /api/v1/employ/1` | Returns single employee by ID |
| 8 | Update employee | `PUT /api/v1/employ/1` | Send updated JSON body |
| 9 | Delete employee | `DELETE /api/v1/employ/1` | Returns success message |

---

## 12. Debugging & Common Errors

### How to Access Logs

```bash
# Method 1: AWS Console
Elastic Beanstalk → Environment → Logs → Request Last 100 Lines

# Method 2: EB CLI
eb logs

# Method 3: SSH into instance
eb ssh
sudo cat /var/log/web.stdout.log
sudo cat /var/log/eb-engine.log
```

### Common Errors & Fixes

| Error / Symptom | Likely Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Wrong Procfile or app crashed | Check `eb logs`, verify Procfile syntax |
| `Health: Severe` (red) | App failed to start | `eb logs` → look for Python traceback |
| `ModuleNotFoundError` | Package missing | Add to `requirements.txt`, redeploy |
| `No module 'multipart'` | Missing dependency | Add `python-multipart` to requirements |
| `application not callable` | Wrong variable name | Add `application = app` in `main.py` |
| `OperationalError SQLite` | Wrong DB path/permissions | Change to `sqlite:////tmp/app.db` |
| `401 Unauthorized` | Token expired or missing | Login again, copy fresh JWT |
| `422 Unprocessable Entity` | Wrong request body format | Check Pydantic schema in Swagger |
| ZIP upload fails | `venv/` included, file too large | Re-ZIP source files only |

### Health Check

Beanstalk checks your app by calling `GET /`. Make sure it returns HTTP 200:

```python
@app.get("/")
def root():
    return {"status": "healthy", "service": "Employee Management API"}
```

---

## 13. Updating & Redeploying

### Console Method

```
1. Make your code changes in VS Code
2. Create a new ZIP file
3. Elastic Beanstalk → Environment → Upload and Deploy
4. Choose file → select new ZIP
5. Enter new version label (e.g. v1.1)
6. Click Deploy → wait for health: Ok ✅
```

### EB CLI Method (faster)

```bash
# Deploy changes
eb deploy

# With version label
eb deploy --label "v1.1-add-health-endpoint"

# View all versions
eb appversion

# Roll back
eb deploy --version v1.0
```

> 💡 **Deployment policies:** Beanstalk supports `All at once` (default, fastest), `Rolling`, and `Immutable`. For class demos, use `All at once`.

---

## 14. Elastic Beanstalk vs AWS Lambda

| Factor | Elastic Beanstalk | AWS Lambda |
|---|---|---|
| **Server model** | EC2-based (always running) | No servers (runs on demand) |
| **Max execution** | Unlimited | 15 minutes |
| **Cold start** | None | 100ms–3s on first request |
| **Idle billing** | Yes — EC2 runs 24/7 | No — only billed per request |
| **Free tier** | 750 hrs EC2 t3.micro/month | 1 million requests/month FREE |
| **WebSocket** | Full support | Limited |
| **File system** | EBS persistent storage | `/tmp` only (512 MB, temporary) |
| **Database** | SQLite or RDS | RDS, DynamoDB (no SQLite) |
| **Scaling speed** | 1–3 minutes to add instance | Milliseconds |
| **Max memory** | EC2 RAM (1 GB–384 GB) | 10 GB maximum |
| **Background tasks** | Yes | No (function must return fast) |
| **Complexity** | Medium | Low |

### Use Beanstalk when...
- ✅ Your app runs long-running tasks (file processing, ML inference)
- ✅ You need WebSockets or persistent connections
- ✅ Migrating an existing server-based app to AWS
- ✅ Students are coming from EC2 experience
- ✅ Steady, predictable traffic

### Use Lambda when...
- ✅ Sporadic or unpredictable traffic patterns
- ✅ Each request is short and independent (< 15 minutes)
- ✅ You want truly zero idle cost
- ✅ Building microservices or event-driven architecture
- ✅ Working with S3 events, DynamoDB streams, API Gateway triggers

---

## 15. Summary & Next Steps

### What You've Covered

- ✅ What Elastic Beanstalk is and how it fits the IaaS → PaaS → Serverless story
- ✅ Internal architecture: EC2, Load Balancer, Auto Scaling Group, Security Groups
- ✅ Pre-deployment fixes: Procfile, requirements.txt typo, SQLite path, venv exclusion
- ✅ Step-by-step deployment via AWS Console and EB CLI
- ✅ Securing secrets with environment variables
- ✅ Testing the deployed FastAPI app with Swagger UI
- ✅ Debugging deployment failures using logs
- ✅ When to choose Beanstalk vs Lambda

---

### Suggested Teaching Schedule

| Session | Topic | Activity |
|---|---|---|
| **Session 1** | Recap EC2 + Intro Beanstalk | Whiteboard: EC2 vs Beanstalk comparison |
| **Session 2** | Pre-deployment fixes + ZIP | Students fix `main.py` and create ZIP |
| **Session 3** | Console deployment (live demo) | Teacher deploys — students watch Events tab |
| **Session 4** | EB CLI setup + deploy | Students deploy from their own terminals |
| **Session 5** | Env vars + testing + logs | Students add env vars, test all endpoints |
| **Session 6** | Beanstalk vs Lambda intro | Discussion: when would you use Lambda? |

---

### Next Topics to Cover

| Topic | Description |
|---|---|
| **AWS Lambda + API Gateway** | Deploy the same FastAPI app as a serverless function using `Mangum` |
| **AWS RDS** | Replace SQLite with PostgreSQL on RDS for production-ready storage |
| **AWS S3** | Store uploaded files (employee photos, documents) in S3 |
| **AWS CloudFront** | Add a CDN in front of your API for global performance |
| **CI/CD with GitHub Actions** | Auto-deploy to Beanstalk when you push code to GitHub |
| **Docker on Beanstalk** | Deploy using the `Dockerfile` already in the student project |

---

