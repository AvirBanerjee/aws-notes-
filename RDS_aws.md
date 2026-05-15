# Amazon RDS — Full Study & Revision Notes

---

## 1. What is Amazon RDS?

**Amazon Relational Database Service (RDS)** is a **managed database service** that makes it easy to set up, operate, and scale relational databases in the AWS Cloud. AWS handles routine tasks like hardware provisioning, database setup, patching, and backups.

### Key Benefits
- No need to manage underlying OS or DB engine patches
- Automated backups, snapshots, and Multi-AZ failover
- Easy vertical and horizontal scaling
- Integrated with IAM, VPC, CloudWatch, KMS

---

## 2. Supported Database Engines

| Engine | Notes |
|---|---|
| **MySQL** | Most widely used open-source RDBMS |
| **PostgreSQL** | Advanced open-source, strong compliance |
| **MariaDB** | MySQL fork, community-driven |
| **Oracle** | Enterprise-grade, license required |
| **Microsoft SQL Server** | Windows-native, licensing via AWS or BYOL |
| **Amazon Aurora** | AWS-proprietary, MySQL/PostgreSQL compatible |

> **Exam Tip:** Aurora is part of RDS but is often treated separately due to its unique architecture.

---

## 3. RDS Instance Types

### DB Instance Classes

| Class Family | Use Case |
|---|---|
| **Standard (db.m)** | General purpose — balanced compute, memory, network |
| **Memory Optimized (db.r / db.x)** | High memory workloads (e.g., large in-memory datasets) |
| **Burstable (db.t)** | Dev/test, low-traffic — CPU credits model |

### Storage Types

| Type | Use Case | Throughput |
|---|---|---|
| **gp2 (General Purpose SSD)** | Default, most workloads | Baseline 3 IOPS/GB, burstable |
| **gp3 (General Purpose SSD)** | Newer, independent IOPS/throughput config | Up to 16,000 IOPS |
| **io1 / io2 (Provisioned IOPS SSD)** | I/O-intensive workloads | Up to 256,000 IOPS (io2 Block Express) |
| **Magnetic (Standard)** | Legacy, not recommended | Low |

> Storage **auto-scaling** can be enabled — RDS automatically increases storage when free space is running low (set a maximum threshold).

---

## 4. Multi-AZ Deployments

### How It Works
- RDS provisions a **synchronous standby replica** in a different Availability Zone
- Data is **synchronously replicated** to the standby
- **Automatic failover** occurs if the primary fails (typically 1–2 minutes)
- The standby is **not readable** — it exists only for failover

### Failover Triggers
- Primary instance failure
- AZ outage
- DB instance OS patching
- DB instance type change

### Key Points
- Endpoint DNS automatically resolves to the new primary after failover
- Multi-AZ is for **high availability (HA)**, NOT for performance scaling
- Supported for all engines (Aurora uses a different HA model)

---

## 5. Read Replicas

### Purpose
- Offload **read traffic** from the primary instance
- Used for **scaling read performance**, NOT for HA failover

### How It Works
- Uses **asynchronous replication** (slight lag possible)
- Can be in the **same AZ, different AZ, or different Region**
- Can be **promoted** to standalone DB (breaks replication)

### Limits
- Up to **5 Read Replicas** per RDS instance (MySQL, MariaDB, PostgreSQL)
- Aurora supports up to **15 Read Replicas**

### Cross-Region Read Replicas
- Available for MySQL, MariaDB, PostgreSQL, Oracle, SQL Server
- Useful for **disaster recovery** and **global read scaling**
- Data transfer charges apply across regions

### Read Replica vs Multi-AZ

| Feature | Read Replica | Multi-AZ |
|---|---|---|
| Replication | Asynchronous | Synchronous |
| Purpose | Read scaling | High availability |
| Readable | Yes | No (standby) |
| Failover | Manual (promotion) | Automatic |
| Cross-region | Yes | No (same region, different AZ) |

---

## 6. Amazon Aurora

### What is Aurora?
Aurora is AWS's **cloud-native relational database engine**, compatible with **MySQL** and **PostgreSQL**. It's part of RDS but designed from scratch for the cloud.

### Aurora Architecture
- Storage is **automatically distributed** across 3 AZs, with 6 copies of data (2 per AZ)
- Storage auto-scales from **10 GB up to 128 TB**
- Separates **compute** (DB instances) from **storage** (shared cluster volume)

### Aurora vs Standard RDS

| Feature | Aurora | Standard RDS |
|---|---|---|
| Replication | 6 copies, 3 AZs | 2 copies, 2 AZs |
| Read Replicas | Up to 15 | Up to 5 |
| Failover | ~30 seconds | 1–2 minutes |
| Storage | Shared, auto-scaling | Attached to instance |
| Performance | 5x MySQL, 3x PostgreSQL | Standard |

### Aurora Variants

| Variant | Description |
|---|---|
| **Aurora Provisioned** | Standard clusters with fixed instances |
| **Aurora Serverless v1/v2** | Auto-scales compute up/down; pay per use |
| **Aurora Global Database** | Low-latency global reads; cross-region DR |
| **Aurora Multi-Master** | Multiple write nodes (MySQL only, being deprecated) |
| **Babelfish for Aurora PostgreSQL** | Runs SQL Server T-SQL queries on PostgreSQL |

### Aurora Serverless v2
- Scales in fine-grained increments (ACUs — Aurora Capacity Units)
- Minimum 0.5 ACU, maximum 128 ACU
- Great for variable/unpredictable workloads

---

## 7. Backups & Snapshots

### Automated Backups
- Enabled by default
- Backed up to **S3** (managed by AWS, not visible in your S3 console)
- **Retention period**: 1–35 days (default: 7 days)
- Point-in-time recovery (PITR) within the retention window
- Backups taken during the **backup window** (maintenance window)
- First backup is full; subsequent are incremental

### Manual Snapshots
- User-initiated
- **Retained indefinitely** (no expiry) unless deleted
- Can be copied across regions and accounts
- Restoring creates a **new DB instance**

### Restore Process
- You cannot restore IN PLACE — a new DB instance is always created
- RDS uses the latest automated backup and applies transaction logs for PITR

### Automated Backup vs Manual Snapshot

| Feature | Automated Backup | Manual Snapshot |
|---|---|---|
| Retention | 1–35 days | Indefinite |
| Trigger | Automatic (scheduled) | Manual |
| Point-in-time restore | Yes | No (only to snapshot time) |
| Deleted with DB? | Yes (can retain) | No |

---

## 8. Security

### Network Security
- RDS instances are deployed inside a **VPC**
- Security Groups control inbound/outbound traffic
- Can use **DB Subnet Groups** to place RDS in private subnets (best practice)
- **No public accessibility** by default (toggle available)

### Encryption
- **At rest**: AES-256, using **AWS KMS** keys
- Encryption must be enabled at creation time (cannot enable later directly)
- Encrypted snapshots can be copied across regions
- To encrypt an unencrypted instance: snapshot → copy snapshot with encryption → restore

### In-Transit Encryption
- SSL/TLS connections supported for all engines
- Can **force SSL** using parameter groups

### IAM Authentication
- Supported for **MySQL** and **PostgreSQL**
- Uses an **IAM authentication token** (valid 15 minutes) instead of password
- Works with IAM roles — good for applications on EC2/Lambda

### Secrets Manager Integration
- RDS credentials can be stored and rotated automatically in **AWS Secrets Manager**

### Audit Logging
- Enable enhanced monitoring and **CloudWatch Logs**
- MySQL: enable general log, slow query log, error log via parameter groups
- PostgreSQL: use `log_connections`, `log_statement` parameters

---

## 9. RDS Proxy

### What Is It?
A **fully managed database proxy** that sits between your application and RDS.

### Why Use It?
- **Connection pooling** — reduces the number of open connections to RDS
- **Improved availability** — handles failover gracefully; reduces failover time by up to 66%
- **IAM authentication** enforcement
- **Secrets Manager integration** for credentials

### Use Cases
- Lambda functions (which can open thousands of short-lived connections)
- Microservices with many instances hitting the same DB
- High connection churn applications

### Supported Engines
- MySQL, PostgreSQL, MariaDB, SQL Server, Aurora MySQL, Aurora PostgreSQL

---

## 10. Parameter Groups & Option Groups

### Parameter Groups
- **DB Parameter Groups**: Control DB engine configuration (e.g., `max_connections`, `character_set`)
- **Cluster Parameter Groups**: For Aurora clusters
- Changes to **static parameters** require a reboot
- Changes to **dynamic parameters** apply immediately

### Option Groups
- Used to enable **additional features** for engines that support them
- Examples: Oracle TDE, SQL Server Transparent Data Encryption, MySQL Memcached

---

## 11. Monitoring & Performance

### Amazon CloudWatch Metrics (Key Ones)
| Metric | Description |
|---|---|
| `CPUUtilization` | % CPU used |
| `DatabaseConnections` | Current active connections |
| `FreeStorageSpace` | Available storage |
| `ReadIOPS / WriteIOPS` | I/O operations per second |
| `ReadLatency / WriteLatency` | Average I/O latency |
| `FreeableMemory` | Available RAM |
| `ReplicaLag` | Lag on Read Replica |

### Enhanced Monitoring
- Real-time OS-level metrics (not just DB engine metrics)
- Granularity down to **1 second**
- Sends data to **CloudWatch Logs**
- Metrics include: CPU, memory, file system, disk I/O, processes

### Performance Insights
- **Visual query performance dashboard**
- Identifies which SQL queries are causing load
- Retention: 7 days free, up to 2 years paid
- Works with: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora

---

## 12. Maintenance & Updates

### Maintenance Window
- Weekly window for applying patches and updates
- You can configure the preferred window
- Minor version upgrades can be set to auto-apply

### Version Upgrades
- **Minor version**: Can be automatic or manual
- **Major version**: Manual only (e.g., PostgreSQL 13 → 14)
- Always test in dev/staging before upgrading production

---

## 13. Pricing Model

| Cost Component | Details |
|---|---|
| **DB Instance hours** | Per instance-hour (varies by instance class and engine) |
| **Storage** | Per GB/month (gp2, gp3, io1, magnetic) |
| **I/O requests** | Charged for magnetic storage only |
| **Provisioned IOPS** | Charged separately for io1/io2 |
| **Backup storage** | Free up to DB size; charged beyond that |
| **Data transfer** | Inbound free; outbound charges apply |
| **Multi-AZ** | ~2x the single-AZ instance cost |

### Cost Optimisation Tips
- Use **Reserved Instances** (1 or 3 year) for predictable workloads — up to 69% savings
- Use **Aurora Serverless** for variable workloads
- Right-size instances regularly using Performance Insights
- Enable storage autoscaling to avoid over-provisioning
- Delete unneeded manual snapshots

---

## 14. High Availability Patterns

### Pattern 1: Multi-AZ + Read Replicas
- Multi-AZ for automatic failover (HA)
- Read Replicas for read scaling
- These are independent — you can have both

### Pattern 2: Aurora Global Database
- Primary cluster in one region
- Up to 5 secondary read-only clusters in other regions
- Replication lag < 1 second
- Supports **managed failover** for disaster recovery

### Pattern 3: RDS Proxy + Multi-AZ
- Proxy handles connection pooling
- Multi-AZ handles failover
- App reconnects to proxy (endpoint doesn't change) — no code changes

---

## 15. RDS vs DynamoDB vs Aurora — Quick Comparison

| Feature | RDS | Aurora | DynamoDB |
|---|---|---|---|
| Type | Relational (SQL) | Relational (SQL) | NoSQL (Key-Value / Document) |
| Schema | Fixed schema | Fixed schema | Flexible schema |
| Scaling | Vertical + Read Replicas | Auto storage + 15 replicas | Horizontal (auto) |
| Latency | Low ms | Very low ms | Single-digit ms |
| Managed | Yes | Yes | Fully serverless |
| Best for | Traditional apps, OLTP | High performance SQL | Scale-out, flexible data |

---

## 16. Common Exam Scenarios

| Scenario | Answer |
|---|---|
| Need automatic failover | Multi-AZ |
| Need to scale reads | Read Replicas |
| Too many DB connections from Lambda | RDS Proxy |
| Encrypt existing unencrypted RDS | Snapshot → Copy with encryption → Restore |
| Need cross-region disaster recovery | Aurora Global Database or Cross-Region Read Replica |
| Need high-performance MySQL/PostgreSQL | Aurora |
| Dev/test DB that can pause to save cost | Aurora Serverless |
| Need point-in-time restore | Enable automated backups |
| RDS in private subnet | Use DB Subnet Group + Security Group |
| Prevent accidental deletion | Enable Deletion Protection |

---

## 17. Key Facts to Remember

- RDS is a **managed** service — you do NOT have SSH access to the underlying OS
- **Deletion protection** can be enabled to prevent accidental deletion
- **Multi-AZ standby is NOT a read replica** — it's invisible and only used for failover
- Storage autoscaling requires a **maximum storage threshold**
- **Aurora storage** replicates 6 copies across 3 AZs automatically
- IAM DB Auth tokens are **valid for 15 minutes**
- Cross-region Read Replicas can be promoted for **regional DR**
- RDS automated backups are **deleted** when you delete the DB instance (unless you choose to retain them)
- You can share **manual snapshots** with other AWS accounts
- **gp3** is the recommended storage type for new instances (more cost-effective than gp2)

---

*Last updated: May 2026 | Based on AWS documentation and SAA-C03 / DVA-C02 exam scope*