# Cloud Databases — Full Notes
### AWS RDS · Azure Database for PostgreSQL · GCP Cloud SQL

---

## 1. Overview & Comparison

All three are fully managed relational database services — you don't manage the OS, patching, backups, or replication. You just connect and use.

| Feature | AWS RDS | Azure Database for PostgreSQL | GCP Cloud SQL |
|---|---|---|---|
| Engine support | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server | PostgreSQL only (+ Flexible Server) | PostgreSQL, MySQL, SQL Server |
| Deployment types | Single-AZ, Multi-AZ, Aurora | Single Server, Flexible Server | Standalone, HA (Regional) |
| High Availability | Multi-AZ standby | Zone-redundant HA | Regional persistent disk |
| Read replicas | Yes | Yes | Yes |
| Serverless option | Aurora Serverless v2 | No | No |
| Storage auto-grow | Yes | Yes | Yes |
| Max storage | 64 TB (Aurora), 16 TB (RDS) | 16 TB | 64 TB |
| Encryption at rest | Yes (KMS) | Yes (service-managed / CMK) | Yes (Google-managed / CMEK) |
| VPC/Private access | Yes (VPC) | Yes (VNet) | Yes (Private IP / VPC) |
| CLI tool | `aws rds` | `az postgres` | `gcloud sql` |

---

## 2. AWS RDS for PostgreSQL

### 2.1 — What is RDS?

Amazon Relational Database Service (RDS) is a managed database service. RDS for PostgreSQL gives you a fully managed PostgreSQL instance running on EC2 under the hood, but without any EC2 access.

### Key Concepts

| Term | Meaning |
|---|---|
| **DB Instance** | The actual database server |
| **DB Instance Class** | Size of the underlying VM (e.g. `db.t3.micro`) |
| **Multi-AZ** | Standby replica in another Availability Zone for failover |
| **Read Replica** | Read-only copy for scaling reads |
| **Parameter Group** | PostgreSQL configuration settings |
| **Subnet Group** | Which VPC subnets RDS can use |
| **Security Group** | Firewall rules controlling access |
| **Snapshot** | Point-in-time backup |

---

### 2.2 — Create RDS PostgreSQL via AWS CLI

#### Step 1 — Create a DB Subnet Group
```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "My RDS Subnet Group" \
  --subnet-ids subnet-xxxxxx subnet-yyyyyy
```

#### Step 2 — Create a Security Group (Allow PostgreSQL port)
```bash
# Create security group
aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS PostgreSQL access" \
  --vpc-id vpc-xxxxxxxx

# Allow inbound on port 5432 from your IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxx \
  --protocol tcp \
  --port 5432 \
  --cidr YOUR_IP/32
```

#### Step 3 — Create the RDS Instance
```bash
aws rds create-db-instance \
  --db-instance-identifier my-postgres-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username adminuser \
  --master-user-password YourStrongPassword123! \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-name myappdb \
  --vpc-security-group-ids sg-xxxxxxxxxx \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --storage-encrypted \
  --multi-az
```

#### Step 4 — Wait for Instance to be Available
```bash
aws rds wait db-instance-available \
  --db-instance-identifier my-postgres-db
```

#### Step 5 — Get the Endpoint
```bash
aws rds describe-db-instances \
  --db-instance-identifier my-postgres-db \
  --query "DBInstances[0].Endpoint.Address" \
  --output text
```

---

### 2.3 — RDS Connection Strings

#### Standard PostgreSQL Connection String
```
postgresql://adminuser:YourStrongPassword123!@my-postgres-db.xxxxxxxxx.us-east-1.rds.amazonaws.com:5432/myappdb
```

#### Format Breakdown
```
postgresql://USERNAME:PASSWORD@ENDPOINT:PORT/DATABASE
```

| Part | Example |
|---|---|
| Username | `adminuser` |
| Password | `YourStrongPassword123!` |
| Host (Endpoint) | `my-postgres-db.xxxxxxxxx.us-east-1.rds.amazonaws.com` |
| Port | `5432` |
| Database | `myappdb` |

#### With SSL (Required for Production)
```
postgresql://adminuser:password@endpoint:5432/myappdb?sslmode=require
```

#### Python (SQLAlchemy)
```python
import os
from sqlalchemy import create_engine

DATABASE_URL = os.environ.get("DATABASE_URL")
# Example value:
# postgresql://adminuser:pass@endpoint.rds.amazonaws.com:5432/myappdb?sslmode=require

engine = create_engine(DATABASE_URL)
```

#### Python (psycopg2)
```python
import psycopg2
import os

conn = psycopg2.connect(
    host="my-postgres-db.xxxxxxxxx.us-east-1.rds.amazonaws.com",
    port=5432,
    database="myappdb",
    user="adminuser",
    password=os.environ.get("DB_PASSWORD"),
    sslmode="require"
)
```

#### Django `settings.py`
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "myappdb",
        "USER": "adminuser",
        "PASSWORD": os.environ.get("DB_PASSWORD"),
        "HOST": "my-postgres-db.xxxxxxxxx.us-east-1.rds.amazonaws.com",
        "PORT": "5432",
        "OPTIONS": {
            "sslmode": "require",
        },
    }
}
```

---

### 2.4 — RDS Security

#### Network Security
```bash
# Keep RDS in private subnet (--no-publicly-accessible)
# Only allow access from specific EC2 instances or Lambda functions
# Add app's security group as the source (not a raw IP)

aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-id \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app-server-id    # allow from app server SG only
```

#### Enable Encryption at Rest
```bash
# Add to create command:
--storage-encrypted \
--kms-key-id arn:aws:kms:us-east-1:123456789:key/your-key-id
```
> ⚠️ Encryption must be set at creation time — you cannot enable it on an existing instance. You must take a snapshot and restore into an encrypted instance.

#### Enable SSL/TLS in Transit
```bash
# Force SSL for all connections via Parameter Group
aws rds create-db-parameter-group \
  --db-parameter-group-name force-ssl \
  --db-parameter-group-family postgres15 \
  --description "Force SSL"

aws rds modify-db-parameter-group \
  --db-parameter-group-name force-ssl \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"

aws rds modify-db-instance \
  --db-instance-identifier my-postgres-db \
  --db-parameter-group-name force-ssl \
  --apply-immediately
```

#### IAM Database Authentication (No Password)
```bash
# Enable IAM auth on the instance
aws rds modify-db-instance \
  --db-instance-identifier my-postgres-db \
  --enable-iam-database-authentication \
  --apply-immediately

# Generate auth token in Python (expires in 15 min)
import boto3

client = boto3.client("rds", region_name="us-east-1")
token = client.generate_db_auth_token(
    DBHostname="endpoint.rds.amazonaws.com",
    Port=5432,
    DBUsername="iam_user",
    Region="us-east-1"
)
# Use token as password in connection string
```

#### Use AWS Secrets Manager for Credentials
```bash
# Store credentials
aws secretsmanager create-secret \
  --name prod/myapp/postgres \
  --secret-string '{"username":"adminuser","password":"YourPass","host":"endpoint","dbname":"myappdb"}'

# Retrieve in Python
import boto3, json

client = boto3.client("secretsmanager", region_name="us-east-1")
secret = client.get_secret_value(SecretId="prod/myapp/postgres")
creds = json.loads(secret["SecretString"])
DATABASE_URL = f"postgresql://{creds['username']}:{creds['password']}@{creds['host']}/{ creds['dbname']}"
```

#### Key Security Rules — RDS
- Never set `--publicly-accessible` in production.
- Always enable `--storage-encrypted`.
- Always use `sslmode=require` in connection strings.
- Use Secrets Manager or Parameter Store — never hardcode credentials.
- Use IAM auth for applications running on AWS (EC2, Lambda, ECS).
- Restrict security group to allow only specific application security groups.
- Enable automated backups (`--backup-retention-period 7`).
- Enable deletion protection: `--deletion-protection`.

---

## 3. Azure Database for PostgreSQL

### 3.1 — Deployment Types

| Type | Description |
|---|---|
| **Single Server** | Legacy, being retired. Don't use for new projects. |
| **Flexible Server** | Current standard. Recommended for all new deployments. |

This guide covers **Flexible Server** only.

### Key Concepts

| Term | Meaning |
|---|---|
| **Flexible Server** | Current Azure PostgreSQL offering |
| **Compute Tier** | Burstable / General Purpose / Memory Optimized |
| **SKU** | VM size (e.g. `Standard_B1ms`) |
| **VNet Integration** | Private access via Azure Virtual Network |
| **Private DNS Zone** | Resolves server hostname within a VNet |
| **Firewall Rules** | IP-based allow rules for public access |
| **Server Parameters** | PostgreSQL configuration (like `max_connections`) |

---

### 3.2 — Create Azure PostgreSQL Flexible Server via CLI

#### Step 1 — Create Resource Group
```bash
az group create \
  --name myDatabaseRG \
  --location eastus
```

#### Step 2 — Create VNet and Subnet (for private access)
```bash
# Create VNet
az network vnet create \
  --resource-group myDatabaseRG \
  --name myVNet \
  --address-prefix 10.0.0.0/16

# Create subnet for the database
az network vnet subnet create \
  --resource-group myDatabaseRG \
  --vnet-name myVNet \
  --name dbSubnet \
  --address-prefixes 10.0.1.0/24 \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers
```

#### Step 3 — Create Private DNS Zone
```bash
az network private-dns zone create \
  --resource-group myDatabaseRG \
  --name mypostgres.private.postgres.database.azure.com

az network private-dns link vnet create \
  --resource-group myDatabaseRG \
  --zone-name mypostgres.private.postgres.database.azure.com \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false
```

#### Step 4 — Create the Flexible Server
```bash
az postgres flexible-server create \
  --resource-group myDatabaseRG \
  --name my-postgres-server \
  --location eastus \
  --admin-user adminuser \
  --admin-password YourStrongPassword123! \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --version 15 \
  --storage-size 32 \
  --database-name myappdb \
  --vnet myVNet \
  --subnet dbSubnet \
  --private-dns-zone mypostgres.private.postgres.database.azure.com \
  --high-availability Disabled \
  --backup-retention 7 \
  --geo-redundant-backup Disabled
```

#### Step 5 — Get Server FQDN
```bash
az postgres flexible-server show \
  --resource-group myDatabaseRG \
  --name my-postgres-server \
  --query "fullyQualifiedDomainName" \
  --output tsv
# Output: my-postgres-server.postgres.database.azure.com
```

#### Step 6 — Add Firewall Rule (if public access needed)
```bash
az postgres flexible-server firewall-rule create \
  --resource-group myDatabaseRG \
  --name my-postgres-server \
  --rule-name AllowMyIP \
  --start-ip-address YOUR_IP \
  --end-ip-address YOUR_IP
```

---

### 3.3 — Azure PostgreSQL Connection Strings

#### Standard Format
```
postgresql://adminuser:YourPassword@my-postgres-server.postgres.database.azure.com:5432/myappdb?sslmode=require
```

#### Format Breakdown
```
postgresql://USERNAME:PASSWORD@SERVER_FQDN:PORT/DATABASE?sslmode=require
```

| Part | Example |
|---|---|
| Username | `adminuser` |
| Password | `YourStrongPassword123!` |
| Host (FQDN) | `my-postgres-server.postgres.database.azure.com` |
| Port | `5432` |
| Database | `myappdb` |
| SSL | `sslmode=require` |

#### Azure Portal — Get Connection String
Azure Portal → Your server → **Connection strings** tab → Copy the pre-filled string.

#### Python (SQLAlchemy)
```python
import os
from sqlalchemy import create_engine

DATABASE_URL = os.environ.get("DATABASE_URL")
# postgresql://adminuser:pass@my-postgres-server.postgres.database.azure.com:5432/myappdb?sslmode=require

engine = create_engine(DATABASE_URL)
```

#### Python (psycopg2)
```python
import psycopg2
import os

conn = psycopg2.connect(
    host="my-postgres-server.postgres.database.azure.com",
    port=5432,
    database="myappdb",
    user="adminuser",
    password=os.environ.get("DB_PASSWORD"),
    sslmode="require"
)
```

#### Django `settings.py`
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "myappdb",
        "USER": "adminuser",
        "PASSWORD": os.environ.get("DB_PASSWORD"),
        "HOST": "my-postgres-server.postgres.database.azure.com",
        "PORT": "5432",
        "OPTIONS": {
            "sslmode": "require",
        },
    }
}
```

#### Azure App Service / Container Apps — Set as App Setting
```bash
az webapp config appsettings set \
  --name myWebApp \
  --resource-group myRG \
  --settings \
    DATABASE_URL="postgresql://adminuser:pass@my-postgres-server.postgres.database.azure.com:5432/myappdb?sslmode=require"
```

---

### 3.4 — Azure PostgreSQL Security

#### Network Security — Use VNet Integration (Private Access)
- Created above with `--vnet` and `--subnet` flags.
- Server is not accessible from the public internet at all.
- Only resources inside the same VNet (or peered VNet) can connect.

#### SSL/TLS — Enforce Encrypted Connections
```bash
# Check current SSL enforcement
az postgres flexible-server parameter show \
  --resource-group myDatabaseRG \
  --server-name my-postgres-server \
  --name require_secure_transport

# Enforce SSL (on by default for Flexible Server)
az postgres flexible-server parameter set \
  --resource-group myDatabaseRG \
  --server-name my-postgres-server \
  --name require_secure_transport \
  --value on
```

#### Use Azure Key Vault for Credentials
```bash
# Store password in Key Vault
az keyvault create \
  --name myKeyVault \
  --resource-group myDatabaseRG \
  --location eastus

az keyvault secret set \
  --vault-name myKeyVault \
  --name db-password \
  --value "YourStrongPassword123!"

# Retrieve in Python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://myKeyVault.vault.azure.net", credential=credential)
password = client.get_secret("db-password").value
```

#### Use Managed Identity for Passwordless Auth
```bash
# Enable Microsoft Entra (AAD) auth on the server
az postgres flexible-server ad-admin set \
  --resource-group myDatabaseRG \
  --server-name my-postgres-server \
  --display-name "My App Identity" \
  --object-id APP_OBJECT_ID

# In Python — use DefaultAzureCredential to get token
import struct, os
from azure.identity import DefaultAzureCredential
import psycopg2

credential = DefaultAzureCredential()
token = credential.get_token("https://ossrdbms-aad.database.windows.net/.default")

conn = psycopg2.connect(
    host="my-postgres-server.postgres.database.azure.com",
    database="myappdb",
    user="my-app-identity@my-postgres-server",
    password=token.token,
    sslmode="require"
)
```

#### Key Security Rules — Azure PostgreSQL
- Use **VNet Integration** (private access) instead of firewall rules where possible.
- Always set `sslmode=require` in connection strings.
- Use **Azure Key Vault** to store credentials — not App Settings in plain text.
- Prefer **Managed Identity** for app-to-database auth (no passwords at all).
- Enable **Microsoft Defender for PostgreSQL** for threat detection.
- Set a strong admin password — 8+ chars, uppercase, lowercase, numbers, symbols.
- Enable **geo-redundant backups** for disaster recovery in production.
- Never allow `0.0.0.0` to `255.255.255.255` in firewall rules.

---

## 4. GCP Cloud SQL for PostgreSQL

### 4.1 — What is Cloud SQL?

Cloud SQL is Google's fully managed relational database service. For PostgreSQL, it provides automated backups, replication, failover, and patches.

### Key Concepts

| Term | Meaning |
|---|---|
| **Instance** | The Cloud SQL database server |
| **Machine Type** | VM size (e.g. `db-f1-micro`, `db-n1-standard-2`) |
| **HA Configuration** | Regional failover with standby replica |
| **Read Replica** | Read-only copy in same or different region |
| **Cloud SQL Auth Proxy** | Secure connection without opening firewall rules |
| **Private IP** | VPC-connected access (recommended) |
| **Public IP** | Internet-accessible (requires authorized networks) |
| **Authorized Networks** | IP allow-list for public access |

---

### 4.2 — Create Cloud SQL PostgreSQL Instance via CLI

#### Step 1 — Enable API
```bash
gcloud services enable sqladmin.googleapis.com
```

#### Step 2 — Create the Instance
```bash
gcloud sql instances create my-postgres-instance \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --root-password=YourStrongPassword123! \
  --storage-type=SSD \
  --storage-size=20GB \
  --storage-auto-increase \
  --backup-start-time=03:00 \
  --retained-backups-count=7 \
  --no-assign-ip \
  --network=projects/YOUR_PROJECT_ID/global/networks/default
```

> `--no-assign-ip` disables public IP — use with `--network` for private IP only access.

#### Step 3 — Create a Database
```bash
gcloud sql databases create myappdb \
  --instance=my-postgres-instance
```

#### Step 4 — Create a User
```bash
gcloud sql users create appuser \
  --instance=my-postgres-instance \
  --password=AppUserPassword123!
```

#### Step 5 — Get Instance Connection Name
```bash
gcloud sql instances describe my-postgres-instance \
  --format="value(connectionName)"
# Output: YOUR_PROJECT_ID:us-central1:my-postgres-instance
```

---

### 4.3 — Cloud SQL Auth Proxy (Recommended Connection Method)

The **Cloud SQL Auth Proxy** is a lightweight process that handles authentication and encryption automatically. You don't need to whitelist IPs or manage SSL certificates manually.

#### Install Cloud SQL Auth Proxy
```bash
# Linux / macOS
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.6.0/cloud-sql-proxy.linux.amd64

chmod +x cloud-sql-proxy

# macOS (Homebrew)
brew install cloud-sql-proxy
```

#### Run the Proxy
```bash
./cloud-sql-proxy YOUR_PROJECT_ID:us-central1:my-postgres-instance \
  --port 5432 &
```

#### Connect via Proxy (localhost)
```bash
psql -h 127.0.0.1 -p 5432 -U appuser -d myappdb
```

#### In Python (via proxy — connect to localhost)
```python
import os
from sqlalchemy import create_engine

# When using Cloud SQL Auth Proxy locally
DATABASE_URL = "postgresql://appuser:AppUserPassword123!@127.0.0.1:5432/myappdb"
engine = create_engine(DATABASE_URL)
```

#### Docker Sidecar Pattern (Cloud Run / GKE)
```yaml
# In Cloud Run — proxy runs as a sidecar
# (add these to your service YAML or use --add-cloudsql-instances flag)
```

```bash
# Deploy Cloud Run service with Cloud SQL
gcloud run deploy my-app \
  --image gcr.io/PROJECT_ID/my-app \
  --add-cloudsql-instances YOUR_PROJECT_ID:us-central1:my-postgres-instance \
  --set-env-vars INSTANCE_CONNECTION_NAME="YOUR_PROJECT_ID:us-central1:my-postgres-instance" \
  --set-env-vars DB_USER="appuser" \
  --set-env-vars DB_PASS="AppUserPassword123!" \
  --set-env-vars DB_NAME="myappdb"
```

#### Python with Cloud Run + Unix Socket (Proxy)
```python
import os
from sqlalchemy import create_engine
from sqlalchemy.engine.url import URL

# Cloud Run uses Unix domain socket via Cloud SQL proxy
db_socket_dir = os.environ.get("DB_SOCKET_DIR", "/cloudsql")
instance_connection_name = os.environ.get("INSTANCE_CONNECTION_NAME")

pool = create_engine(
    URL.create(
        drivername="postgresql+pg8000",
        username=os.environ.get("DB_USER"),
        password=os.environ.get("DB_PASS"),
        database=os.environ.get("DB_NAME"),
        query={
            "unix_sock": f"{db_socket_dir}/{instance_connection_name}/.s.PGSQL.5432"
        }
    )
)
```

---

### 4.4 — Cloud SQL Connection Strings

#### Standard (Public IP with SSL)
```
postgresql://appuser:AppUserPassword123!@PUBLIC_IP:5432/myappdb?sslmode=require
```

#### Private IP (Inside VPC)
```
postgresql://appuser:AppUserPassword123!@PRIVATE_IP:5432/myappdb
```

#### Via Cloud SQL Auth Proxy (Localhost)
```
postgresql://appuser:AppUserPassword123!@127.0.0.1:5432/myappdb
```

#### Python (SQLAlchemy) — Proxy
```python
import os
from sqlalchemy import create_engine

DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql://appuser:pass@127.0.0.1:5432/myappdb"
)
engine = create_engine(DATABASE_URL)
```

#### Django `settings.py`
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "myappdb"),
        "USER": os.environ.get("DB_USER", "appuser"),
        "PASSWORD": os.environ.get("DB_PASS"),
        "HOST": "127.0.0.1",          # when using Cloud SQL Auth Proxy
        "PORT": "5432",
    }
}
```

---

### 4.5 — Cloud SQL Security

#### Use Private IP (No Public Exposure)
```bash
# Already covered in create command with --no-assign-ip + --network
# Only resources inside the VPC can reach the database
```

#### Add Authorized Network (if Public IP is used)
```bash
gcloud sql instances patch my-postgres-instance \
  --authorized-networks=YOUR_IP/32
```
> Only use this for dev/testing. In production, always use **private IP** or **Cloud SQL Auth Proxy**.

#### Enable SSL/TLS
```bash
# Require SSL for all connections
gcloud sql instances patch my-postgres-instance \
  --require-ssl

# Create a client certificate
gcloud sql ssl client-certs create my-cert client-key.pem \
  --instance=my-postgres-instance

# Download server CA cert
gcloud sql ssl server-ca-certs list \
  --instance=my-postgres-instance \
  --format="value(cert)" > server-ca.pem
```

#### Use Secret Manager for Credentials
```bash
# Store DB password
echo -n "AppUserPassword123!" | \
  gcloud secrets create db-password --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Reference in Cloud Run deploy
gcloud run deploy my-app \
  --set-secrets=DB_PASS=db-password:latest
```

#### Use IAM Database Authentication (Passwordless)
```bash
# Enable IAM auth on instance
gcloud sql instances patch my-postgres-instance \
  --database-flags=cloudsql.iam_authentication=on

# Create IAM user (maps to a Google/service account)
gcloud sql users create "my-service-account@PROJECT_ID.iam" \
  --instance=my-postgres-instance \
  --type=cloud_iam_service_account

# Grant role inside PostgreSQL
# GRANT ALL PRIVILEGES ON DATABASE myappdb TO "my-service-account@PROJECT_ID.iam";

# Connect with IAM token in Python
from google.auth import default
from google.auth.transport.requests import Request

creds, project = default()
creds.refresh(Request())

import psycopg2
conn = psycopg2.connect(
    host="127.0.0.1",
    port=5432,
    database="myappdb",
    user="my-service-account@PROJECT_ID.iam",
    password=creds.token,
    sslmode="disable"    # proxy handles encryption
)
```

#### Key Security Rules — Cloud SQL
- Use **Private IP** with VPC access — disable public IP in production.
- Use **Cloud SQL Auth Proxy** for secure connections from apps — no firewall rules needed.
- Always use `--require-ssl` if using public IP.
- Store credentials in **Secret Manager** — never in environment variables as plain text.
- Use **IAM database authentication** for services running on GCP.
- Grant the **Cloud SQL Client** role to service accounts, not project-level Editor/Owner.
- Enable **automated backups** and test restore regularly.
- Enable **deletion protection**:
  ```bash
  gcloud sql instances patch my-postgres-instance --deletion-protection
  ```

---

## 5. Connection String Comparison

| Platform | Format |
|---|---|
| **AWS RDS** | `postgresql://user:pass@endpoint.rds.amazonaws.com:5432/db?sslmode=require` |
| **Azure PostgreSQL** | `postgresql://user:pass@server.postgres.database.azure.com:5432/db?sslmode=require` |
| **GCP Cloud SQL (proxy)** | `postgresql://user:pass@127.0.0.1:5432/db` |
| **GCP Cloud SQL (public IP)** | `postgresql://user:pass@PUBLIC_IP:5432/db?sslmode=require` |
| **GCP Cloud SQL (private IP)** | `postgresql://user:pass@PRIVATE_IP:5432/db` |

### Universal Python Pattern
```python
import os
from sqlalchemy import create_engine

# Always read from environment — never hardcode
DATABASE_URL = os.environ.get("DATABASE_URL")

if not DATABASE_URL:
    raise RuntimeError("DATABASE_URL environment variable is not set")

engine = create_engine(
    DATABASE_URL,
    pool_size=5,           # number of persistent connections
    max_overflow=10,       # extra connections when pool is full
    pool_timeout=30,       # seconds to wait for a connection
    pool_recycle=1800,     # recycle connections every 30 min
    echo=False             # set True to log all SQL (dev only)
)
```

---

## 6. Security Best Practices — All Platforms

### Credentials
```
❌ Never hardcode credentials in code
❌ Never commit .env files to git
❌ Never print DATABASE_URL in logs
✅ Use Secrets Manager (AWS) / Key Vault (Azure) / Secret Manager (GCP)
✅ Rotate passwords regularly
✅ Use IAM/Managed Identity where available (passwordless)
```

### Network
```
✅ Always use private networking (VPC/VNet)
✅ Disable public IP unless absolutely necessary
✅ Restrict access to only the application's IP or security group
✅ Enable SSL/TLS in transit (sslmode=require)
✅ Enable encryption at rest
```

### Access Control
```
✅ Create a dedicated app database user (not the admin/root user)
✅ Grant only necessary privileges (SELECT, INSERT, UPDATE, DELETE on specific tables)
✅ Never connect from apps using the master/admin user
```

```sql
-- Example: Create least-privilege app user in PostgreSQL
CREATE USER appuser WITH PASSWORD 'SecurePassword123!';
GRANT CONNECT ON DATABASE myappdb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;
```

### Operational
```
✅ Enable automated backups (minimum 7 days retention)
✅ Test restore from backup regularly
✅ Enable deletion protection on production instances
✅ Set up monitoring and alerts for connection failures, CPU, storage
✅ Use read replicas to scale read workloads (offload reports/analytics)
✅ Enable point-in-time recovery (PITR)
```

---

## 7. Quick Reference Cheatsheet

### AWS RDS
```bash
# Create
aws rds create-db-instance --db-instance-identifier NAME --engine postgres \
  --db-instance-class db.t3.micro --master-username admin \
  --master-user-password PASS --allocated-storage 20

# Get endpoint
aws rds describe-db-instances --db-instance-identifier NAME \
  --query "DBInstances[0].Endpoint.Address" --output text

# Delete (with final snapshot)
aws rds delete-db-instance --db-instance-identifier NAME \
  --final-db-snapshot-identifier final-snap

# List instances
aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus]"
```

### Azure PostgreSQL
```bash
# Create
az postgres flexible-server create --name NAME --resource-group RG \
  --admin-user admin --admin-password PASS --sku-name Standard_B1ms

# Get FQDN
az postgres flexible-server show --name NAME --resource-group RG \
  --query "fullyQualifiedDomainName" --output tsv

# List servers
az postgres flexible-server list --resource-group RG --output table

# Delete
az postgres flexible-server delete --name NAME --resource-group RG --yes
```

### GCP Cloud SQL
```bash
# Create
gcloud sql instances create NAME --database-version=POSTGRES_15 \
  --tier=db-f1-micro --region=us-central1

# Get connection name
gcloud sql instances describe NAME --format="value(connectionName)"

# Connect via proxy
./cloud-sql-proxy PROJECT:REGION:INSTANCE --port 5432 &
psql -h 127.0.0.1 -U appuser -d myappdb

# List instances
gcloud sql instances list

# Delete
gcloud sql instances delete NAME
```
