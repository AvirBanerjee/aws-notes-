# Employee Management API — Full Notes
### FastAPI + SQLAlchemy + JWT + Azure Deployment

---

## 1. Project Overview

This is a **RESTful API** built with FastAPI that manages employees.  
It includes user authentication (JWT), employee CRUD operations, and is backed by SQLite (or any SQL database via SQLAlchemy).

### Tech Stack

| Layer | Technology |
|---|---|
| Framework | FastAPI |
| Database ORM | SQLAlchemy |
| Database | SQLite (dev) / PostgreSQL (prod) |
| Auth | JWT via `python-jose` |
| Password Hashing | `passlib[bcrypt]` |
| Validation | Pydantic v2 |
| Server | Uvicorn / Gunicorn |

---

## 2. Project File Structure

```
employee-api/
├── main.py               # All app code (models, routes, auth)
├── requirements.txt      # Python dependencies
├── Dockerfile            # For Docker/ACR deployment
├── .env                  # Local env vars (never commit this)
└── startup.txt           # Optional: Azure startup command
```

### `requirements.txt`
```txt
fastapi
uvicorn[standard]
sqlalchemy
python-jose[cryptography]
passlib[bcrypt]
python-multipart
pydantic
```
> `python-multipart` is required for OAuth2 form-based login to work.

---

## 3. App Initialization

```python
app = FastAPI(title="Employee Management API", version="2.0")
```

- Creates the FastAPI application instance.
- `title` and `version` appear in the auto-generated `/docs` (Swagger UI).

---

## 4. CORS Middleware

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)
```

- Allows the API to be called from any frontend (browser).
- `allow_origins=["*"]` means all origins are permitted.
- **In production**, replace `["*"]` with your specific frontend URL:
  ```python
  allow_origins=["https://myapp.com"]
  ```

---

## 5. Database Setup

```python
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:////tmp/app.db")

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

- `DATABASE_URL` is read from environment variable — falls back to SQLite at `/tmp/app.db`.
- `check_same_thread=False` is a SQLite-specific fix for multi-threaded access.
- `SessionLocal` is a factory that creates new DB sessions per request.
- `Base` is the parent class all ORM models inherit from.
- `Base.metadata.create_all(bind=engine)` — auto-creates tables on startup if they don't exist.

---

## 6. Database Models (SQLAlchemy ORM)

### UserDB
```python
class UserDB(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    fullname = Column(String)
    email = Column(String, unique=True)
    password = Column(String)          # stores bcrypt hash, NOT plain text
    employs = relationship("EmployDB", back_populates="owner")
```

### EmployDB
```python
class EmployDB(Base):
    __tablename__ = "employs"
    id = Column(Integer, primary_key=True, index=True)
    fullname = Column(String)
    email = Column(String)
    isOnProject = Column(Boolean)
    experience = Column(Integer)
    completed = Column(Integer)
    description = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))   # links employee to user
    owner = relationship("UserDB", back_populates="employs")
```

- `relationship()` sets up a **one-to-many** link: one user → many employees.
- `ForeignKey("users.id")` in `EmployDB` stores which user owns this employee record.

---

## 7. Pydantic Schemas

Pydantic schemas handle **request validation** (input) and **response shaping** (output). They are separate from ORM models.

| Schema | Purpose |
|---|---|
| `UserCreate` | Validate register request body |
| `UserResponse` | Shape the user data returned in responses |
| `EmployCreate` | Validate create/update employee request body |
| `EmployResponse` | Shape employee data in responses (includes `id`) |
| `Token` | Shape the login response (token + type) |

```python
class EmployResponse(EmployCreate):
    id: int
    model_config = ConfigDict(from_attributes=True)
```
- `from_attributes=True` (Pydantic v2) tells Pydantic to read data from ORM object attributes (instead of dict).
- `EmployResponse` extends `EmployCreate` and simply adds the `id` field.

---

## 8. Security & Authentication

### Password Hashing
```python
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain, hashed):
    return pwd_context.verify(plain, hashed)
```
- Passwords are **never stored as plain text** — always hashed with bcrypt.
- `verify_password` compares a plain text input against the stored hash.

### JWT Token Creation
```python
SECRET_KEY = "SUPERSECRETSHHHHHHHHH"   # ⚠️ move to env var in production
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```
- Creates a signed JWT token that expires in 60 minutes.
- The token payload carries the user's `email`.
- **⚠️ Production fix**: move `SECRET_KEY` to an environment variable.

### OAuth2 Scheme
```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/login")
```
- Tells FastAPI that the token is obtained from `/api/v1/login`.
- Automatically adds an **Authorize** button in Swagger UI (`/docs`).

---

## 9. Dependency Injection

### `get_db` — Database Session per Request
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```
- Opens a DB session at the start of each request, closes it after — even if an error occurs.

### `get_current_user` — Protect Routes
```python
def get_current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)):
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    email = payload.get("email")
    user = db.query(UserDB).filter(UserDB.email == email).first()
    return user
```
- Decodes JWT, extracts email, fetches user from DB.
- Any route that includes `Depends(get_current_user)` is **automatically protected**.
- Returns 401 if token is missing, expired, or invalid.

---

## 10. API Routes Reference

### Auth Routes

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| POST | `/api/v1/register` | No | Register new user |
| POST | `/api/v1/login` | No | Login, receive JWT token |

### Dashboard

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| GET | `/api/v1/dashboard` | Yes | Get user info + employee count |

### Employee CRUD

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| POST | `/api/v1/employ` | Yes | Create employee |
| GET | `/api/v1/employs` | Yes | Get all employees (current user's) |
| GET | `/api/v1/employ/{id}` | Yes | Get single employee by ID |
| PUT | `/api/v1/employ/{id}` | Yes | Update employee |
| DELETE | `/api/v1/employ/{id}` | Yes | Delete employee |

### Utility

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| GET | `/` | No | Health check |
| GET | `/health` | No | Health check |

---

## 11. Route Logic Walkthrough

### Register — `POST /api/v1/register`
1. Check if email already exists in DB → 400 if yes.
2. Hash the password.
3. Create `UserDB` object, save to DB.
4. Return user data (without password).

### Login — `POST /api/v1/login`
1. Accepts `username` (email) + `password` as form data (OAuth2 standard).
2. Look up user by email → 400 if not found.
3. Verify password against hash → 400 if wrong.
4. Create and return JWT access token.

### Create Employee — `POST /api/v1/employ`
1. Requires valid JWT (user authenticated).
2. Creates `EmployDB` record linked to `current_user` via `owner=current_user`.
3. Saves to DB.

### Get All Employees — `GET /api/v1/employs`
- Returns `current_user.employs` — only employees belonging to the logged-in user.
- Users **cannot** see each other's employees (data isolation).

### Get / Update / Delete Single Employee
- Queries by `id` AND `user_id == current_user.id`.
- Returns 404 if employee doesn't exist **or** belongs to a different user (security by design).

---

## 12. Deployment — Step by Step

### Option A: Deploy via ZIP to Azure App Service

#### Step 1 — Prepare Files
Make sure your project has:
```
main.py
requirements.txt      # all dependencies listed
```

#### Step 2 — Create Azure Resources
```bash
# Login
az login

# Create resource group
az group create --name employeeApiRG --location eastus

# Create App Service Plan (Linux required for Python)
az appservice plan create \
  --name employeeApiPlan \
  --resource-group employeeApiRG \
  --sku B1 \
  --is-linux

# Create the Web App
az webapp create \
  --resource-group employeeApiRG \
  --plan employeeApiPlan \
  --name employeeManagementAPI \
  --runtime "PYTHON:3.11"
```

#### Step 3 — Set Environment Variables
```bash
az webapp config appsettings set \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --settings \
    SECRET_KEY="your-super-secret-key-here" \
    DATABASE_URL="sqlite:////tmp/app.db"
```

#### Step 4 — Enable Build on Deployment
```bash
az webapp config appsettings set \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --settings SCM_DO_BUILD_DURING_DEPLOYMENT=true
```
> This makes Azure auto-run `pip install -r requirements.txt` on deploy.

#### Step 5 — Set Startup Command
```bash
az webapp config set \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --startup-file "gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000"
```
> Note: The file exports `application = app` at the bottom — you can also use `main:application`.

#### Step 6 — Package and Deploy ZIP
```bash
zip -r employeeapi.zip . -x "*.git*" "__pycache__/*" "*.pyc" ".env"

az webapp deploy \
  --resource-group employeeApiRG \
  --name employeeManagementAPI \
  --src-path employeeapi.zip \
  --type zip
```

#### Step 7 — Verify
```bash
# Get the live URL
az webapp show \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --query "defaultHostName"

# Stream logs
az webapp log tail \
  --name employeeManagementAPI \
  --resource-group employeeApiRG
```
> Visit `https://<your-app>.azurewebsites.net/docs` for Swagger UI.

---

### Option B: Deploy via Docker + Azure Container Registry (ACR)

#### Step 1 — Write the Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", \
     "main:app", "--bind", "0.0.0.0:8000"]
```

#### Step 2 — Build and Test Locally
```bash
docker build -t employee-api:latest .

# Test locally
docker run -p 8000:8000 \
  -e SECRET_KEY="testsecret" \
  -e DATABASE_URL="sqlite:////tmp/app.db" \
  employee-api:latest

# Visit http://localhost:8000/docs
```

#### Step 3 — Create ACR and Push Image
```bash
# Create registry
az acr create \
  --resource-group employeeApiRG \
  --name employeeACR \
  --sku Basic \
  --admin-enabled true

# Login to ACR
az acr login --name employeeACR

# Tag image
docker tag employee-api:latest employeeACR.azurecr.io/employee-api:latest

# Push image
docker push employeeACR.azurecr.io/employee-api:latest
```

#### Step 4 — Create Web App from ACR Image
```bash
az webapp create \
  --resource-group employeeApiRG \
  --plan employeeApiPlan \
  --name employeeManagementAPI \
  --deployment-container-image-name employeeACR.azurecr.io/employee-api:latest
```

#### Step 5 — Link ACR to App Service (Managed Identity — Recommended)
```bash
# Enable managed identity on the web app
az webapp identity assign \
  --name employeeManagementAPI \
  --resource-group employeeApiRG

# Get principal ID
principalId=$(az webapp identity show \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --query principalId --output tsv)

# Get ACR resource ID
acrId=$(az acr show \
  --name employeeACR \
  --resource-group employeeApiRG \
  --query id --output tsv)

# Assign AcrPull role
az role assignment create \
  --assignee $principalId \
  --role AcrPull \
  --scope $acrId

# Point app to ACR image
az webapp config container set \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --docker-custom-image-name employeeACR.azurecr.io/employee-api:latest \
  --docker-registry-server-url https://employeeACR.azurecr.io
```

#### Step 6 — Set Environment Variables
```bash
az webapp config appsettings set \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --settings \
    SECRET_KEY="your-super-secret-key-here" \
    DATABASE_URL="sqlite:////tmp/app.db" \
    WEBSITES_PORT=8000
```

#### Step 7 — Enable Auto-Deploy on Image Push (Optional)
```bash
az webapp deployment container config \
  --name employeeManagementAPI \
  --resource-group employeeApiRG \
  --enable-cd true
```
> Every `docker push` to ACR will trigger an automatic redeploy.

---

## 13. Production Checklist

- [ ] Move `SECRET_KEY` to Azure App Settings (never hardcode).
- [ ] Replace SQLite with a persistent database (Azure PostgreSQL / MySQL).
- [ ] Set `allow_origins` in CORS to specific frontend URL.
- [ ] Use `WEBSITES_PORT=8000` to match your container's exposed port.
- [ ] Enable Application Insights for monitoring and error tracking.
- [ ] Use deployment slots (staging → production) for zero-downtime deploys.
- [ ] Tag Docker images with version numbers (`v1.0.0`) in addition to `latest`.
- [ ] Enable HTTPS only:
  ```bash
  az webapp update \
    --name employeeManagementAPI \
    --resource-group employeeApiRG \
    --https-only true
  ```

---

## 14. Testing the API (After Deployment)

### Register a user
```bash
curl -X POST https://<your-app>.azurewebsites.net/api/v1/register \
  -H "Content-Type: application/json" \
  -d '{"fullname": "John Doe", "email": "john@example.com", "password": "pass123"}'
```

### Login and get token
```bash
curl -X POST https://<your-app>.azurewebsites.net/api/v1/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=john@example.com&password=pass123"
```

### Create an employee (with token)
```bash
curl -X POST https://<your-app>.azurewebsites.net/api/v1/employ \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "fullname": "Jane Smith",
    "email": "jane@company.com",
    "isOnProject": true,
    "experience": 3,
    "completed": 12,
    "description": "Backend developer"
  }'
```

### Get all employees
```bash
curl https://<your-app>.azurewebsites.net/api/v1/employs \
  -H "Authorization: Bearer <your-token>"
```

> Or use the interactive **Swagger UI** at `https://<your-app>.azurewebsites.net/docs`