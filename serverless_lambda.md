# Serverless Computing and AWS Lambda — Full Theory and Deployment Guide
### Reference Project: Employee Management API (FastAPI)

---

## Part 1: The Theory of Serverless Computing

### 1.1 What Serverless Actually Means

Serverless computing is a cloud execution paradigm where the unit of deployment is a function
or a small application, not a server. The term is deliberately provocative — servers still
exist and always will, but the developer no longer thinks about them, provisions them, patches
them, or pays for them when idle.

The serverless model shifts three traditional responsibilities away from the developer:

- **Infrastructure provisioning** — you do not choose instance types, AMIs, or regions at the
  machine level
- **Runtime management** — the cloud provider installs the OS, the language runtime, and all
  system-level dependencies
- **Capacity planning** — you do not decide how many instances to run; the platform scales
  automatically from zero to millions of requests

What you retain is the application logic. You write code. The cloud runs it.

---

### 1.2 The History That Led to Serverless

Understanding why serverless exists requires understanding what came before it.

**Physical servers (pre-2006):** A company that needed to run a web application bought or leased
physical servers, installed an operating system, configured a web server, deployed the
application, and maintained everything. Capacity planning meant buying hardware months before
it was needed, based on projected traffic. If the projection was wrong, either users suffered
or expensive hardware sat idle.

**Virtual machines / IaaS (2006 onward):** AWS launched EC2, which virtualised physical
servers into on-demand instances. This eliminated hardware procurement but kept everything else.
You still chose the instance type, installed the OS, configured the network, managed scaling
policies, and paid for every hour the instance ran — even at 3am with zero traffic.

**Platform as a Service (2009 onward):** Heroku, then later Elastic Beanstalk, abstracted the
instance management. You provided the code and a `Procfile`; the platform handled the OS,
the web server, and auto-scaling. The instance still ran continuously, but you thought about
it less.

**Containers (2013 onward):** Docker standardised application packaging. Kubernetes managed
container orchestration at scale. Containers were lighter than VMs and started faster, but the
infrastructure layer was still present and still required management.

**Serverless / FaaS (2014 onward):** AWS Lambda launched in November 2014. For the first time,
a developer could deploy a function — not an application, not a server, not a container — and
have it execute on demand without managing any infrastructure. The billing model changed from
per-hour to per-invocation-per-millisecond. An application with zero traffic cost nothing at all.

---

### 1.3 The Four Properties That Define Serverless

A platform is genuinely serverless when it exhibits all four of these properties:

**1. No server management**
The developer writes code. The provider handles everything below the application layer: physical
servers, virtualisation, operating system, runtime installation, security patching, network
configuration. None of these are visible to the developer.

**2. Event-driven execution**
Functions do not run continuously. They run in response to an event — an HTTP request, a file
upload, a database change, a message in a queue, a scheduled timer. When there is no event,
there is no execution and no cost.

**3. Automatic scaling**
The platform scales the number of running function instances in direct proportion to the number
of incoming events. Zero events means zero instances. One million simultaneous events means
one million simultaneous instances. No scaling policy, no threshold configuration, no manual
intervention.

**4. Pay-per-use pricing**
Billing is based on actual consumption: the number of times the function was invoked and the
duration of each invocation in milliseconds. A function that runs once for 100ms and is then
idle for a month costs exactly the same as one that never ran.

---

### 1.4 The Serverless Execution Model in Detail

When an event triggers a Lambda function, the following sequence occurs internally:

**Stage 1 — Event routing**
The event source (API Gateway, S3, SQS, etc.) emits an event. Lambda's control plane receives
it and determines which function and which version should handle it.

**Stage 2 — Container acquisition**
Lambda checks its internal pool of warm containers for this function. A warm container is one
that handled a previous invocation and has been kept frozen in memory. If one is available, it
is thawed and assigned. If not, a new container must be created.

**Stage 3 — Cold start initialisation (only when no warm container exists)**
Lambda allocates compute capacity, downloads the deployment package (your zip or container image),
extracts it, starts the language runtime, and executes all global-scope code in your module —
imports, variable declarations, database engine creation, anything outside the handler function.

**Stage 4 — Handler execution**
The handler function is called with the event object and the context object. Your application
logic runs. For the employee API, this is where FastAPI's router would match the URL path and
call the appropriate route function — `register_user`, `login_user`, `get_employs`, etc.

**Stage 5 — Response and freeze**
The handler returns. Lambda captures the return value and sends it back to the caller. The
container is not immediately destroyed — it is frozen in place for a period (typically 5-15
minutes) in case another invocation arrives. If one does, the container is thawed and stage 4
repeats without the overhead of stages 2 and 3. If none arrives within the idle window, Lambda
discards the container.

---

### 1.5 Cold Starts: The Core Trade-Off of Serverless

A cold start is the latency penalty paid when no warm container is available. It has three
components:

**Infrastructure initialisation:** AWS allocating the underlying compute resource. This is
entirely outside developer control and typically takes 50-200ms.

**Runtime initialisation:** Starting the language runtime. Python starts faster than Java or
.NET; Node.js starts faster than Python.

**Module initialisation:** Executing the global scope of your code — all the `import`
statements, all class definitions, all setup code that runs at module load time.

For the Employee Management API, the module initialisation phase includes:

```python
from fastapi import FastAPI, HTTPException, Depends          # FastAPI framework
from sqlalchemy import create_engine, Column, Integer, ...  # SQLAlchemy ORM
from jose import jwt, JWTError                              # JWT library
from passlib.context import CryptContext                    # Password hashing

engine = create_engine(DATABASE_URL, ...)                   # Opens DB connection
Base.metadata.create_all(bind=engine)                       # Checks/creates tables
```

Every one of these lines runs during a cold start and contributes to the latency a user sees
on the first request after the function has been idle.

**Warm starts** skip all of this. The container already has the Python runtime loaded, FastAPI
already instantiated, the SQLAlchemy engine already connected. The handler receives the event
and proceeds immediately to business logic.

**Reducing cold start impact:**
- Place all heavy initialisation at the global scope (not inside the handler), so it runs once
  per container lifetime rather than once per invocation
- Use provisioned concurrency for routes where cold start latency is unacceptable
- Choose a lightweight runtime — Python is a good choice for this project

---

### 1.6 Concurrency in Lambda

Lambda handles concurrency by running multiple container instances in parallel, not by running
multiple threads within a single container (though a single Lambda instance is single-threaded
by default in Python).

```
Request 1 ──→ Container A (warm) ──→ response
Request 2 ──→ Container B (warm) ──→ response
Request 3 ──→ Container C (cold) ──→ response
...
```

**Account-level concurrency limit:** By default, all Lambda functions in an AWS account share
a pool of 1,000 concurrent executions per region. If 1,000 invocations are running simultaneously
and a 1,001st arrives, it is throttled (rejected with a 429 error or retried depending on the
invocation type). This limit can be raised by contacting AWS support.

**Reserved concurrency:** A specific number of concurrent executions set aside exclusively for
one function. If the employee API's `POST /api/v1/login` is given a reserved concurrency of 100,
it always has 100 slots available regardless of what other functions are doing — and it cannot
use more than 100 regardless of demand.

**Provisioned concurrency:** Pre-initialised containers that are always warm and ready. Unlike
reserved concurrency (which is just a capacity reservation), provisioned concurrency actually
runs the initialisation phase ahead of time. A function with provisioned concurrency of 10
has 10 containers already loaded with the Python runtime, FastAPI initialised, and the database
engine connected — waiting for requests to arrive. Cold starts are eliminated for those 10 slots.
The trade-off is cost: provisioned concurrency is billed even when idle.

---

### 1.7 Lambda Execution Environment Internals

Each Lambda execution environment is an isolated microVM (AWS uses Firecracker, an open-source
virtualisation technology they developed). It has:

- A fixed amount of memory (128 MB to 10,240 MB, configured by you)
- CPU allocated proportionally to memory (you cannot set CPU independently)
- An ephemeral `/tmp` filesystem with 512 MB to 10 GB of writable space
- A read-only filesystem containing your deployment package
- Environment variables set in the function configuration
- A network interface (optionally connected to a VPC)

The isolation is complete. Two simultaneous invocations cannot share memory, communicate
through the filesystem, or affect each other in any way. Each is a separate microVM.

**What persists within a container across warm invocations:**
- Global variables
- Objects instantiated at module scope (the SQLAlchemy `engine` object, for example)
- Files written to `/tmp` (though you should not depend on this being present)
- Any in-memory cache you build

**What does not persist:**
- Anything written to `/tmp` once the container is discarded
- Any state in global variables once the container is discarded
- Database connections if the underlying DB server closes them due to idle timeout

---

### 1.8 The Lambda Programming Model

Lambda imposes a specific contract on your code:

```python
def handler(event: dict, context: LambdaContext) -> dict:
    # event: the input data from the trigger
    # context: runtime metadata
    # return: the response (format depends on trigger type)
    pass
```

**The event object** is a plain Python dictionary whose structure is entirely determined by what
triggered the function. For API Gateway HTTP API, it looks like:

```python
{
    "version": "2.0",
    "routeKey": "POST /api/v1/register",
    "rawPath": "/api/v1/register",
    "rawQueryString": "",
    "headers": {
        "content-type": "application/json",
        "authorization": "Bearer eyJ..."
    },
    "body": '{"fullname": "Ali Hassan", "email": "ali@example.com", "password": "secret"}',
    "isBase64Encoded": False,
    "requestContext": {
        "http": {
            "method": "POST",
            "path": "/api/v1/register"
        }
    }
}
```

**The context object** provides runtime information:

```python
context.function_name          # "employee-api"
context.function_version       # "$LATEST" or a version number
context.aws_request_id         # unique ID for this invocation
context.memory_limit_in_mb     # "512"
context.get_remaining_time_in_millis()  # ms until timeout
```

**The return value** for API Gateway must be:

```python
{
    "statusCode": 200,
    "headers": {"Content-Type": "application/json"},
    "body": '{"id": 1, "fullname": "Ali Hassan", "email": "ali@example.com"}'
}
```

This event-in, response-out contract is why a bridge library (Mangum) is needed to run FastAPI
on Lambda. FastAPI expects ASGI, not raw Lambda events. Mangum translates between the two.

---

### 1.9 Trigger Types and Invocation Models

Lambda functions are invoked in one of three models:

**Synchronous invocation**
The caller sends an event and blocks waiting for the response. API Gateway always uses this
model. For the employee API, every HTTP request from a client is a synchronous invocation.
If Lambda throttles or errors, API Gateway immediately returns a 429 or 502 to the client.

**Asynchronous invocation**
The caller sends an event and immediately receives an acknowledgment (202 Accepted). Lambda
processes the event in the background. S3, SNS, and EventBridge use this model. If the function
fails, Lambda can retry automatically up to 2 times with configurable delay. A dead-letter queue
can capture events that exhaust all retries.

**Stream/polling invocation**
Lambda polls a stream or queue on your behalf. SQS, DynamoDB Streams, and Kinesis use this
model. Lambda reads records in batches, passes the batch to the handler, and only advances
the stream cursor after a successful response. Failed batches are retried.

For the employee API, only synchronous invocation via API Gateway is relevant. But understanding
the other models matters when extending the API — for example, sending a welcome email after
registration would be better handled asynchronously via SQS rather than blocking the register
response.

---

### 1.10 Layers and Dependency Management

A Lambda Layer is a zip archive of libraries, runtimes, or other dependencies that can be
attached to multiple Lambda functions. Instead of bundling all of `requirements.txt` inside
every deployment package, shared dependencies can live in a layer.

For the employee API, the `requirements.txt` contains several large libraries (SQLAlchemy,
pydantic, passlib, FastAPI). These could be separated into a layer:

```
Layer zip:
  python/
    lib/
      python3.11/
        site-packages/
          fastapi/
          sqlalchemy/
          passlib/
          pydantic/
          jose/
          mangum/
          ...
```

The function zip then contains only `main.py` and small project-specific files. This separation
makes deployments faster (uploading 5 KB instead of 50 MB for a code change) and allows the
same dependency layer to be reused across multiple functions.

Maximum: 5 layers per function, total unzipped size of all layers plus the function package
must not exceed 250 MB.

---

### 1.11 Lambda Destinations and Error Handling

When a Lambda function fails on an asynchronous invocation, the event can be routed to a
destination for inspection and reprocessing. Lambda supports two kinds of destinations:

- **On success:** Route the result to another Lambda function, an SQS queue, an SNS topic,
  or EventBridge
- **On failure:** Same routing options, used for dead-letter handling

For the employee API, if it were extended to process background jobs (generating reports,
sending emails, syncing data), destinations would allow failed jobs to be captured and
retried or logged without losing the original payload.

For synchronous HTTP invocations (API Gateway), errors are returned directly to the client
as HTTP responses. FastAPI's `HTTPException` translates cleanly into Lambda's statusCode response.

---

### 1.12 VPC Configuration

By default, Lambda runs in AWS-managed infrastructure outside any customer VPC. It has internet
access but cannot reach resources inside a private VPC — such as an RDS database that is not
publicly accessible.

To give Lambda access to a VPC, you configure:
- The VPC ID
- One or more subnet IDs (private subnets recommended)
- A security group that allows outbound traffic to the database port

Lambda then attaches an Elastic Network Interface (ENI) to the VPC on your behalf. The function
can now reach RDS, ElastiCache, and other VPC-private resources as if it were an EC2 instance
inside the same network.

**Trade-off:** VPC attachment historically increased cold start time significantly (up to 10
seconds) because Lambda had to provision a new ENI for each cold start. AWS resolved this in
2020 by pre-provisioning ENIs. Cold starts in VPC-attached functions are now comparable to
non-VPC functions, but there is still a small additional latency.

For the employee API, VPC configuration is required if RDS is deployed in a private subnet
(which is best practice).

---

### 1.13 IAM and the Lambda Execution Role

Every Lambda function is assigned an **execution role** — an IAM role that defines what AWS
services the function is permitted to call. The function's code runs with the permissions of
this role.

For the employee API on Lambda, the execution role would need:
- `AWSLambdaBasicExecutionRole` — write logs to CloudWatch (always needed)
- `AWSLambdaVPCAccessExecutionRole` — manage ENIs to connect to VPC (if using VPC)
- RDS-specific permissions if using IAM authentication for the database

If the function tried to call an AWS service (S3, SES, DynamoDB) without the appropriate IAM
permission on its execution role, the SDK would throw an `AccessDeniedException` at runtime.
The principle of least privilege applies: grant only the permissions the function actually needs.

---

### 1.14 Observability: Logs and Monitoring

Every `print()` statement and every log message from Python's `logging` module in the handler
is automatically captured by Lambda and written to **AWS CloudWatch Logs**. Each function has
its own log group (`/aws/lambda/function-name`) and each container instance creates its own
log stream.

A single invocation's log output looks like:

```
START RequestId: abc-123 Version: $LATEST
INFO:     127.0.0.1:443 - "POST /api/v1/login HTTP/1.1" 200 OK
END RequestId: abc-123
REPORT RequestId: abc-123  Duration: 234.56 ms  Billed Duration: 235 ms
        Memory Size: 512 MB  Max Memory Used: 89 MB  Init Duration: 1243.11 ms
```

The `Init Duration` field appears only on cold starts. It shows the time spent in stage 3
(module initialisation). This is the number to watch when optimising cold start performance.

**AWS X-Ray** provides distributed tracing — it instruments each invocation and traces the
request across Lambda, API Gateway, RDS, and any other AWS service involved. For the employee
API, X-Ray would show the exact time spent in each route handler and how much of that time
was database queries.

---

## Part 2: Deploying the Employee API to AWS Lambda

### 2.1 What Changes in the Code

Before deploying, three changes are needed in `main.py`. No route, model, schema, or
authentication logic needs to be touched.

**Change 1 — Add Mangum and replace the ASGI app alias**

`mangum` is the ASGI-to-Lambda bridge. It receives the raw API Gateway event, wraps it in an
ASGI-compatible scope, passes it through FastAPI's full middleware and routing stack, and returns
the response in the format Lambda and API Gateway expect.

```python
# Add to imports
from mangum import Mangum

# At the very bottom of main.py, replace:
application = app

# With:
handler = Mangum(app)
```

Everything between those two lines — every route, middleware, model, schema, and dependency —
remains unchanged. Mangum acts as a transparent adapter.

**Change 2 — Remove the SQLite default from DATABASE_URL**

```python
# Before
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:////tmp/app.db")

# After
DATABASE_URL = os.environ.get("DATABASE_URL")
if not DATABASE_URL:
    raise RuntimeError("DATABASE_URL environment variable is not set")
```

Failing loudly on startup is better than silently creating a SQLite file that will lose all
data when the container is discarded.

Also update the `create_engine` call to remove the SQLite-specific argument:

```python
# Before
engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False}  # SQLite-only argument
)

# After
engine = create_engine(DATABASE_URL)
```

**Change 3 — Externalise the secret key**

```python
# Before
SECRET_KEY = "SUPERSECRETSHHHHHHHHH"

# After
SECRET_KEY = os.environ.get("JWT_SECRET_KEY")
if not SECRET_KEY:
    raise RuntimeError("JWT_SECRET_KEY environment variable is not set")
```

The full updated top section of `main.py` after all three changes:

```python
import os
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from fastapi import FastAPI
from mangum import Mangum

from pydantic import BaseModel, ConfigDict
from typing import List

from sqlalchemy import create_engine, Column, Integer, String, Boolean, ForeignKey
from sqlalchemy.orm import declarative_base, sessionmaker, Session, relationship

from jose import jwt, JWTError
from passlib.context import CryptContext
from datetime import datetime, timedelta

# -------------------- APP --------------------
app = FastAPI(title="Employee Management API", version="2.0")

# -------------------- CORS --------------------
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)

# -------------------- DATABASE --------------------
DATABASE_URL = os.environ.get("DATABASE_URL")
if not DATABASE_URL:
    raise RuntimeError("DATABASE_URL environment variable is not set")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# -------------------- SECURITY --------------------
SECRET_KEY = os.environ.get("JWT_SECRET_KEY")
if not SECRET_KEY:
    raise RuntimeError("JWT_SECRET_KEY environment variable is not set")

ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

# ... rest of main.py unchanged ...

# -------------------- LAMBDA HANDLER --------------------
handler = Mangum(app)
```

---

### 2.2 Updating requirements.txt

Add `mangum` to `requirements.txt`:

```
mangum==0.17.0
```

`gunicorn` can be removed — it is not used by Lambda. Keeping it does not break anything but
adds unnecessary size to the deployment package.

---

### 2.3 Setting Up the Database (RDS PostgreSQL)

Lambda cannot use the SQLite file at `/tmp/app.db`. A proper relational database accessible
over the network is required.

**Step 1: Create an RDS PostgreSQL instance**

In the AWS Console → RDS → Create database:
- Engine: PostgreSQL
- Template: Free Tier (for learning) or Production
- DB instance identifier: `employee-api-db`
- Master username: `dbadmin`
- Master password: choose and store securely
- Instance type: `db.t3.micro` for learning
- Storage: 20 GB gp2
- VPC: place in the same VPC where Lambda will run
- Public access: No (Lambda will access it through the VPC)
- Initial database name: `employee_api`

**Step 2: Configure the security group**

The RDS instance has a security group that controls which resources can connect to it on
port 5432. Create a rule:
- Type: PostgreSQL
- Port: 5432
- Source: the security group of your Lambda function (not an IP address — security groups can
  reference other security groups)

**Step 3: (Optional but recommended) Create an RDS Proxy**

RDS Proxy manages a connection pool between Lambda and RDS. Without it, every Lambda container
opens its own connection. With it, all Lambda containers share a managed pool.

In the AWS Console → RDS → Proxies → Create proxy:
- Engine: PostgreSQL
- Target: the RDS instance created above
- IAM authentication: enable
- VPC and subnets: same as the RDS instance

The proxy provides an endpoint (e.g., `employee-api-db.proxy-xxxx.us-east-1.rds.amazonaws.com`)
that you use instead of the direct RDS endpoint in `DATABASE_URL`.

**Step 4: Construct the DATABASE_URL**

```
postgresql://dbadmin:yourpassword@employee-api-db.proxy-xxxx.us-east-1.rds.amazonaws.com:5432/employee_api
```

Store this value in AWS Secrets Manager:

```bash
aws secretsmanager create-secret \
    --name employee-api/database-url \
    --secret-string "postgresql://dbadmin:yourpassword@proxy-endpoint:5432/employee_api"
```

---

### 2.4 Building the Deployment Package

Lambda requires all Python dependencies to be bundled with the function code.
There is no `pip install` step at runtime.

**Create a clean directory and install dependencies:**

```bash
mkdir package
pip install -r requirements.txt --target ./package --platform manylinux2014_x86_64 \
    --implementation cp --python-version 3.11 --only-binary=:all:
```

The `--platform manylinux2014_x86_64` flag is critical. If you are building on macOS or
Windows, native C extensions (like `bcrypt` inside `passlib`, or `pydantic-core`) will be
compiled for your local OS, not for the Amazon Linux 2 environment Lambda uses. This flag
forces pip to download the Linux-compatible binary wheels instead.

**Copy the application code:**

```bash
cp main.py ./package/
```

**Create the zip archive:**

```bash
cd package
zip -r ../employee_api.zip .
cd ..
```

Check the size:

```bash
ls -lh employee_api.zip
```

The zip must be under 50 MB for direct upload. If it is larger (which is common with
SQLAlchemy, pydantic, passlib, etc.), upload it to S3 first:

```bash
aws s3 cp employee_api.zip s3://your-bucket/employee_api.zip
```

---

### 2.5 Creating the Lambda Function

**Step 1: Create an IAM execution role**

In the AWS Console → IAM → Roles → Create role:
- Trusted entity: Lambda
- Attach policies:
  - `AWSLambdaBasicExecutionRole` (CloudWatch logging)
  - `AWSLambdaVPCAccessExecutionRole` (VPC/ENI access for RDS connectivity)
- Role name: `employee-api-lambda-role`

**Step 2: Create the Lambda function**

In the AWS Console → Lambda → Create function:
- Author from scratch
- Function name: `employee-api`
- Runtime: Python 3.11
- Architecture: x86_64
- Execution role: use the role created above

**Step 3: Upload the code**

In the function's Code tab:
- If under 50 MB: upload the zip directly
- If over 50 MB: choose "Upload from S3" and provide the S3 URL

**Step 4: Set the handler**

In the Runtime settings:
- Handler: `main.handler`

This tells Lambda to look for a file named `main.py` and call the object named `handler`
inside it. `handler` is the `Mangum(app)` object, which is callable.

**Step 5: Configure memory and timeout**

In the Configuration tab → General configuration:
- Memory: 512 MB (a reasonable starting point for FastAPI with SQLAlchemy)
- Timeout: 30 seconds (adjust based on your slowest expected query; the default 3 seconds
  is almost certainly too short for database operations)

**Step 6: Set environment variables**

In the Configuration tab → Environment variables:

| Key | Value |
|---|---|
| `DATABASE_URL` | `postgresql://dbadmin:pass@proxy-endpoint:5432/employee_api` |
| `JWT_SECRET_KEY` | A long random secret string (minimum 32 characters) |

For production, retrieve these values from Secrets Manager rather than storing them as plain
environment variables. Lambda supports Secrets Manager integration via the Parameters and
Secrets extension.

**Step 7: Configure VPC**

In the Configuration tab → VPC:
- VPC: the same VPC as your RDS instance
- Subnets: private subnets in at least two availability zones
- Security groups: a security group that allows outbound traffic on port 5432

---

### 2.6 Setting Up API Gateway

Lambda needs an HTTP trigger to handle incoming web requests from clients.
API Gateway acts as the front door.

**Step 1: Create an HTTP API**

In the AWS Console → API Gateway → Create API → HTTP API:
- Integration: Lambda
- Lambda function: `employee-api`
- API name: `employee-api-gateway`
- Stage: `$default` (auto-deploy enabled)

HTTP API (not REST API) is the correct choice here. It has lower latency, lower cost, and
supports the API Gateway v2 event format that Mangum expects by default.

**Step 2: Configure routes**

HTTP API can be configured with a catch-all route that forwards everything to Lambda:
- Method: ANY
- Route: `/{proxy+}`

This means API Gateway forwards every incoming HTTP request — regardless of method or path —
to the Lambda function. FastAPI's router inside Lambda handles the path matching.

Alternatively, individual routes can be explicitly defined for better access control and
observability, but for the employee API a catch-all is simpler.

**Step 3: Note the invoke URL**

After creation, API Gateway provides an invoke URL:
```
https://abc123xyz.execute-api.us-east-1.amazonaws.com
```

This is the base URL for the API. The full endpoints become:
```
POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/register
POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/login
GET  https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/employs
```

The FastAPI interactive docs are available at:
```
https://abc123xyz.execute-api.us-east-1.amazonaws.com/docs
```

---

### 2.7 Testing the Deployment

**Test 1: Health check**

```bash
curl https://abc123xyz.execute-api.us-east-1.amazonaws.com/
```

Expected response:
```json
{"status": "healthy", "service": "Employee Management API"}
```

If this returns a 502 or 503, check CloudWatch logs for the Lambda function.

**Test 2: Register a user**

```bash
curl -X POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/register \
  -H "Content-Type: application/json" \
  -d '{"fullname": "Ali Hassan", "email": "ali@example.com", "password": "mypassword"}'
```

Expected response:
```json
{"id": 1, "fullname": "Ali Hassan", "email": "ali@example.com"}
```

**Test 3: Login**

```bash
curl -X POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ali@example.com&password=mypassword"
```

Expected response:
```json
{"access_token": "eyJ...", "token_type": "bearer"}
```

**Test 4: Create an employee (authenticated)**

```bash
curl -X POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/v1/employ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{
    "fullname": "Sara Ahmed",
    "email": "sara@company.com",
    "isOnProject": true,
    "experience": 3,
    "completed": 12,
    "description": "Backend developer"
  }'
```

---

### 2.8 Reading CloudWatch Logs

Every invocation is logged automatically. To view logs:

AWS Console → CloudWatch → Log groups → `/aws/lambda/employee-api`

Each log stream corresponds to one Lambda container. Inside, you will see one block per
invocation:

```
START RequestId: abc-123 Version: $LATEST
INFO:     - "POST /api/v1/login HTTP/1.1" 200 OK
END RequestId: abc-123
REPORT RequestId: abc-123
    Duration: 312.45 ms
    Billed Duration: 313 ms
    Memory Size: 512 MB
    Max Memory Used: 103 MB
    Init Duration: 1876.34 ms   ← cold start overhead
```

The `Init Duration` line only appears on cold starts. On warm invocations, it is absent.
Watching this value tells you how long the module initialisation phase (imports, database
engine creation, `create_all`) is taking.

---

### 2.9 Updating the Function After Code Changes

When `main.py` is modified, the deployment package must be rebuilt and uploaded:

```bash
# Rebuild (if only main.py changed, only need to update that file in the zip)
zip -u employee_api.zip main.py

# Upload via AWS CLI
aws lambda update-function-code \
    --function-name employee-api \
    --zip-file fileb://employee_api.zip
```

If dependencies in `requirements.txt` changed, the full package rebuild is required:

```bash
rm -rf package employee_api.zip
mkdir package
pip install -r requirements.txt --target ./package \
    --platform manylinux2014_x86_64 \
    --implementation cp --python-version 3.11 \
    --only-binary=:all:
cp main.py ./package/
cd package && zip -r ../employee_api.zip . && cd ..
aws lambda update-function-code \
    --function-name employee-api \
    --zip-file fileb://employee_api.zip
```

---

### 2.10 Common Errors and What Causes Them

**502 Bad Gateway from API Gateway**
Lambda returned an error or an incorrectly formatted response. Check CloudWatch logs for
the actual Python exception. Common causes:
- `DATABASE_URL` environment variable not set (the `RuntimeError` from the startup check)
- Database not reachable (VPC/security group misconfiguration)
- Import error because a dependency was not included in the zip

**Task timed out after 3.00 seconds**
The default timeout of 3 seconds expired before the handler returned. A cold start alone
on this project can exceed 3 seconds. Increase the timeout to at least 30 seconds.

**could not connect to server: Connection refused**
Lambda cannot reach the RDS instance. Check:
- Lambda and RDS are in the same VPC
- Lambda's security group is listed as an inbound source in RDS's security group on port 5432
- If using RDS Proxy: the proxy's security group also permits inbound from Lambda

**[ERROR] Runtime.ImportModuleError**
A Python module in `import` statements is not present in the deployment package. Rebuild the
package and ensure all entries in `requirements.txt` were installed into `./package/`.

**OSError: [Errno 30] Read-only file system**
Code is attempting to write to the filesystem outside `/tmp`. The Lambda filesystem is
read-only except for the `/tmp` directory. If any library or application code writes to
the current directory or anywhere other than `/tmp`, this error occurs.

---

### 2.11 Complete Deployment Checklist

```
Code changes
[ ] Added: from mangum import Mangum
[ ] Changed: handler = Mangum(app) at the bottom of main.py
[ ] Removed: application = app
[ ] Changed: DATABASE_URL has no SQLite default, raises if unset
[ ] Removed: connect_args={"check_same_thread": False} from create_engine
[ ] Changed: SECRET_KEY reads from os.environ.get("JWT_SECRET_KEY")
[ ] Added: mangum to requirements.txt

Database
[ ] RDS PostgreSQL instance created
[ ] RDS Proxy created (recommended)
[ ] Security group allows inbound from Lambda on port 5432

Deployment package
[ ] pip install with manylinux2014_x86_64 platform flag
[ ] main.py copied into package directory
[ ] zip created from inside package directory

Lambda function
[ ] Function created with Python 3.11 runtime
[ ] Handler set to main.handler
[ ] Memory set to 512 MB minimum
[ ] Timeout set to 30 seconds minimum
[ ] DATABASE_URL environment variable set
[ ] JWT_SECRET_KEY environment variable set
[ ] VPC configured with private subnets
[ ] Execution role has BasicExecution and VPCAccess policies

API Gateway
[ ] HTTP API created with Lambda integration
[ ] Catch-all route configured: ANY /{proxy+}
[ ] Invoke URL noted and tested

Verification
[ ] GET / returns healthy status
[ ] POST /api/v1/register creates a user
[ ] POST /api/v1/login returns a JWT token
[ ] Authenticated routes work with the token
[ ] CloudWatch logs confirm invocations are being recorded
```