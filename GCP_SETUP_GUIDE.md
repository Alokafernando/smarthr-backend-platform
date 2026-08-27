# SmartHR Cloud Platform — Complete GCP Setup & Deployment Guide

> [!IMPORTANT]
> This is a **complete step-by-step guide** to deploy the SmartHR HR Management Platform on Google Cloud Platform.
> All commands, configurations, and instructions are tailored to **your specific project**.

---

## 📋 Project Information

| Field | Value |
|---|---|
| **GCP Project ID** | `project-10a84e2a-6485-49ee-825` |
| **GCP Account** | `buddhikaaloka2005@gmail.com` |
| **Organization** | `buddhikaaloka2005-org` |
| **Region / Zone** | `asia-southeast1` / `asia-southeast1-b` |
| **Local Workspace** | `c:\ijse\eca\HR-Management\smarthr-backend-platform` |

---

## 📋 Current GCP Progress (What You Already Have)

| Resource | Name | Type | Status | Details |
|---|---|---|---|---|
| ✅ Cloud SQL (PostgreSQL 18) | `postgress-vm` | `db-custom-2-8192` | **RUNNABLE** | Private IP: `10.55.192.2` |
| ⏹️ Compute Engine VM | `vm-platform-seed` | `e2-medium` | **TERMINATED** | Internal IP: `10.148.0.2` |
| ❌ Cloud Storage Bucket | — | — | Not created | Needed for document-service |
| ❌ Cloud Run | — | — | API not enabled | Needed for frontend |
| ✅ VPC Network | `default` | AUTO | Active | With default firewall rules |

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INTERNET / BROWSER                          │
└──────────────────┬──────────────────────┬───────────────────────────┘
                   │                      │
        ┌──────────▼──────────┐  ┌────────▼────────────┐
        │   Cloud Run         │  │  External IP         │
        │   smarthr-frontend  │  │  (vm-platform-seed)  │
        │   React/Vite :80    │  │  Port 8080           │
        └─────────────────────┘  └────────┬─────────────┘
                                          │
        ┌─────────────────────────────────▼─────────────────────────┐
        │            vm-platform-seed (Compute Engine)              │
        │            e2-medium | 10.148.0.2                         │
        │                                                           │
        │  ┌──────────────┐  ┌──────────────┐                      │
        │  │ Config Server│  │ Eureka Server│                      │
        │  │ :8888        │  │ :8761        │                      │
        │  └──────┬───────┘  └──────┬───────┘                      │
        │         │    Configuration & Discovery                    │
        │  ┌──────▼───────────────────▼──────────────────────────┐  │
        │  │                                                     │  │
        │  │  ┌──────────────┐  ┌─────────────────┐             │  │
        │  │  │ API Gateway  │  │                  │             │  │
        │  │  │ :8080        │──┤  Routing Layer   │             │  │
        │  │  └──────┬───────┘  └─────────────────┘             │  │
        │  │         │                                           │  │
        │  │  ┌──────▼──────┐ ┌──────────────┐ ┌─────────────┐  │  │
        │  │  │User Service │ │Employee Svc  │ │Document Svc │  │  │
        │  │  │:8081        │ │:8082         │ │:8083        │  │  │
        │  │  └──────┬──────┘ └──────┬───────┘ └──┬───────┬──┘  │  │
        │  │         │               │            │       │      │  │
        │  └─────────┼───────────────┼────────────┼───────┼──────┘  │
        └────────────┼───────────────┼────────────┼───────┼─────────┘
                     │               │            │       │
              ┌──────▼───────────────▼──────┐     │       │
              │  Cloud SQL PostgreSQL       │     │       │
              │  postgress-vm               │     │       │
              │  IP: 10.55.192.2            │     │       │
              │  ┌──────────────────────┐   │     │       │
              │  │ smarthr_user_db      │   │     │       │
              │  │ smarthr_employee_db  │   │     │       │
              │  └──────────────────────┘   │     │       │
              └─────────────────────────────┘     │       │
                                                  │       │
              ┌───────────────────────────────────▼──┐    │
              │  MongoDB Atlas                       │    │
              │  smarthr_document_db                  │    │
              │  hr-management.r3kcaqr.mongodb.net    │    │
              └──────────────────────────────────────┘    │
                                                          │
              ┌───────────────────────────────────────────▼──┐
              │  Google Cloud Storage                        │
              │  smarthr-documents-project-10a84e2a           │
              │  Employee files, PDFs, contracts              │
              └──────────────────────────────────────────────┘
```

---

## 🔌 Port & Service Mapping

| Service | Port | Database | API Routes |
|---|---|---|---|
| **Config Server** | `8888` | — | Internal config distribution |
| **Eureka Server** | `8761` | — | Service discovery dashboard |
| **API Gateway** | `8080` | — | All `/api/v1/**` routes |
| **User Service** | `8081` | PostgreSQL `smarthr_user_db` | `/api/v1/auth/**`, `/api/v1/users/**` |
| **Employee Service** | `8082` | PostgreSQL `smarthr_employee_db` | `/api/v1/employees/**`, `/api/v1/departments/**`, `/api/v1/positions/**`, `/api/v1/attendance/**`, `/api/v1/leaves/**`, `/api/v1/salaries/**`, `/api/v1/dashboard/**` |
| **Document Service** | `8083` | MongoDB Atlas + GCS Bucket | `/api/v1/documents/**` |

---

## Phase 1: GCP Project & CLI Setup

### Step 1.1 — Verify gcloud CLI Installation

Open a **new PowerShell window** (important — the terminal must be restarted after installing gcloud SDK) and run:

```powershell
# Test if gcloud is available in PATH
gcloud --version
```

If `gcloud` is **not recognized**, use the full path for all commands:
```powershell
$gcloud = "C:\Users\TUF_Gaming\AppData\Local\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd"
& $gcloud --version
```

> [!TIP]
> To permanently fix the PATH issue, open a **new** PowerShell window after installation, or restart your computer. The gcloud installer should have already added itself to your PATH.

### Step 1.2 — Set Active Project

```powershell
gcloud config set project project-10a84e2a-6485-49ee-825
gcloud config list
```

### Step 1.3 — Verify Authentication

```powershell
gcloud auth list
# Should show: * buddhikaaloka2005@gmail.com
```

---

## Phase 2: Enable Required GCP APIs

### Step 2.1 — Enable All Required APIs

**Option A — Via GCP Console (GUI):**

1. Go to [APIs & Services > Library](https://console.cloud.google.com/apis/library?project=project-10a84e2a-6485-49ee-825)
2. Search for and **Enable** each of these:

| # | API Name | Search Term | Purpose |
|---|---|---|---|
| 1 | Compute Engine API | `compute` | ✅ Already enabled (you created VMs) |
| 2 | Cloud SQL Admin API | `cloud sql admin` | ✅ Already enabled (you created `postgress-vm`) |
| 3 | Service Networking API | `service networking` | ✅ Already enabled (Cloud SQL private IP) |
| 4 | Cloud Storage JSON API | `cloud storage` | Document file uploads to GCS bucket |
| 5 | Cloud Run Admin API | `cloud run` | Frontend deployment (React/Vite) |
| 6 | Cloud Build API | `cloud build` | Build frontend container images |
| 7 | IAM API | `iam` | Service accounts and permissions |
| 8 | Artifact Registry API | `artifact registry` | Store Docker container images |

**Option B — Via CLI (faster):**

```powershell
gcloud services enable `
    compute.googleapis.com `
    sqladmin.googleapis.com `
    storage.googleapis.com `
    servicenetworking.googleapis.com `
    iam.googleapis.com `
    run.googleapis.com `
    cloudbuild.googleapis.com `
    artifactregistry.googleapis.com
```

> [!NOTE]
> This takes 1-2 minutes. You only need to do this once per project.

---

## Phase 3: Database Setup (Cloud SQL PostgreSQL)

### Step 3.1 — Create Application Databases

Your Cloud SQL instance `postgress-vm` is already running (PostgreSQL 18, Private IP `10.55.192.2`). We need to create the databases and a dedicated application user.

**Option A — Via GCP Console (GUI):**

1. Go to [SQL](https://console.cloud.google.com/sql?project=project-10a84e2a-6485-49ee-825)
2. Click on **`postgress-vm`**
3. Go to the **Databases** tab
4. Click **"Create Database"**
   - Database name: `smarthr_user_db`
   - Character set: `UTF8`
   - Click **Create**
5. Click **"Create Database"** again
   - Database name: `smarthr_employee_db`
   - Character set: `UTF8`
   - Click **Create**
6. Go to the **Users** tab
7. Click **"Add User Account"**
   - Username: `smarthr_app`
   - Password: `SmartHR@2026!Secure` (or your own strong password)
   - Click **Add**

**Option B — Via CLI:**

```powershell
# Create databases
gcloud sql databases create smarthr_user_db --instance=postgress-vm --charset=UTF8
gcloud sql databases create smarthr_employee_db --instance=postgress-vm --charset=UTF8

# Create application user
gcloud sql users create smarthr_app --instance=postgress-vm --password="SmartHR@2026!Secure"
```

### Step 3.2 — Verify Databases Created

```powershell
gcloud sql databases list --instance=postgress-vm
```

Expected output:
```
NAME                 CHARSET  COLLATION
postgres             UTF8     en_US.UTF8
smarthr_user_db      UTF8     en_US.UTF8
smarthr_employee_db  UTF8     en_US.UTF8
```

### Step 3.3 — Grant Database Permissions

Connect to the Cloud SQL instance via Cloud Shell or proxy:

```powershell
# Option 1: Connect via Cloud SQL Proxy (from local machine)
gcloud sql connect postgress-vm --user=postgres

# Once connected to psql, run:
```

```sql
-- Grant privileges to smarthr_app user
GRANT ALL PRIVILEGES ON DATABASE smarthr_user_db TO smarthr_app;
GRANT ALL PRIVILEGES ON DATABASE smarthr_employee_db TO smarthr_app;

-- Connect to each database and grant schema privileges
\c smarthr_user_db
GRANT ALL ON SCHEMA public TO smarthr_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO smarthr_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO smarthr_app;

\c smarthr_employee_db
GRANT ALL ON SCHEMA public TO smarthr_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO smarthr_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO smarthr_app;

\q
```

> [!IMPORTANT]
> **Cloud SQL Private IP:** `10.55.192.2`
> Your microservices running on `vm-platform-seed` (same VPC `default`) can connect to this IP directly on port `5432`.

---

## Phase 4: MongoDB Atlas Setup (document-service)

### Step 4.1 — Verify MongoDB Atlas Cluster

Your `document-service` connects to MongoDB Atlas. Verify:

1. Go to [MongoDB Atlas Dashboard](https://cloud.mongodb.com)
2. Log in with your account
3. Select your cluster: **`hr-management`**
4. Verify the database `smarthr_document_db` exists (or it will be auto-created by Spring Data MongoDB)

### Step 4.2 — Allow GCP VM Access

1. In MongoDB Atlas, go to **Network Access** (left sidebar)
2. Click **"Add IP Address"**
3. Click **"Allow Access from Anywhere"** (`0.0.0.0/0`)
   - Or add only your VM's external IP for better security
4. Click **Confirm**

> [!NOTE]
> **MongoDB Connection URI (already configured in your config-repo):**
> ```
> mongodb+srv://buddhikaaloka2005_db_user:pEvgEDUlw0Qbrh99@hr-management.r3kcaqr.mongodb.net/smarthr_document_db
> ```

---

## Phase 5: Create Google Cloud Storage Bucket

### Step 5.1 — Create the Documents Bucket

**Option A — Via Console:**

1. Go to [Cloud Storage > Buckets](https://console.cloud.google.com/storage/browser?project=project-10a84e2a-6485-49ee-825)
2. Click **"Create"**
3. Settings:
   - **Bucket name:** `smarthr-documents-project-10a84e2a` *(must be globally unique)*
   - **Location type:** Region → `asia-southeast1 (Singapore)`
   - **Storage class:** Standard
   - **Access control:** Uniform (recommended)
   - **Protection:** None (default)
4. Click **"Create"**
5. If prompted about public access prevention, click **"Confirm"**

**Option B — Via CLI:**

```powershell
gcloud storage buckets create gs://smarthr-documents-project-10a84e2a `
    --location=asia-southeast1 `
    --default-storage-class=STANDARD `
    --uniform-bucket-level-access
```

### Step 5.2 — Set Bucket Permissions

Grant the Compute Engine service account access to the bucket:

```powershell
# Get the project number
gcloud projects describe project-10a84e2a-6485-49ee-825 --format="value(projectNumber)"

# Grant storage access to Compute Engine default service account
gcloud storage buckets add-iam-policy-binding gs://smarthr-documents-project-10a84e2a `
    --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" `
    --role="roles/storage.objectAdmin"
```

> [!TIP]
> Replace `<PROJECT_NUMBER>` with the number returned from the first command (e.g., `967905708839`).

---

## Phase 6: VPC Firewall Rules

### Step 6.1 — Create Microservice Communication Rules

You currently have default firewall rules + `temp-rule1`. Let's add specific rules for SmartHR.

**Option A — Via Console:**

Go to [VPC Network > Firewall](https://console.cloud.google.com/networking/firewalls/list?project=project-10a84e2a-6485-49ee-825)

**Rule 1: Allow API Gateway Public Access (HTTP from Internet)**

1. Click **"Create Firewall Rule"**
2. Settings:
   - **Name:** `allow-smarthr-gateway-http`
   - **Network:** `default`
   - **Priority:** `1000`
   - **Direction of traffic:** Ingress
   - **Action on match:** Allow
   - **Targets:** All instances in the network *(or use tag `smarthr-backend`)*
   - **Source filter:** IPv4 ranges
   - **Source IPv4 ranges:** `0.0.0.0/0`
   - **Protocols and ports:** Specified → **TCP:** `8080, 8761`
3. Click **"Create"**

**Rule 2: Allow Microservice Internal Ports**

1. Click **"Create Firewall Rule"**
2. Settings:
   - **Name:** `allow-smarthr-internal-services`
   - **Network:** `default`
   - **Priority:** `1000`
   - **Direction:** Ingress
   - **Action:** Allow
   - **Targets:** All instances in the network
   - **Source IPv4 ranges:** `10.128.0.0/9`
   - **Protocols and ports:** TCP: `8080, 8081, 8082, 8083, 8761, 8888, 5432`
3. Click **"Create"**

**Option B — Via CLI:**

```powershell
# Rule 1: Allow public access to API Gateway and Eureka Dashboard
gcloud compute firewall-rules create allow-smarthr-gateway-http `
    --network=default `
    --direction=INGRESS `
    --priority=1000 `
    --action=ALLOW `
    --rules=tcp:8080,tcp:8761 `
    --source-ranges=0.0.0.0/0 `
    --description="Allow public HTTP access to SmartHR API Gateway and Eureka"

# Rule 2: Allow internal microservice communication
gcloud compute firewall-rules create allow-smarthr-internal-services `
    --network=default `
    --direction=INGRESS `
    --priority=1000 `
    --action=ALLOW `
    --rules=tcp:8080,tcp:8081,tcp:8082,tcp:8083,tcp:8761,tcp:8888,tcp:5432 `
    --source-ranges=10.128.0.0/9 `
    --description="Allow internal communication between SmartHR microservices"
```

> [!NOTE]
> Your existing `default-allow-internal` rule already allows all TCP traffic within the VPC (`10.128.0.0/9`), so Rule 2 is mostly for documentation clarity. Rule 1 is essential for external access.

---

## Phase 7: Prepare Backend VM (`vm-platform-seed`)

### Step 7.1 — Start the VM

**Via Console:**
1. Go to [Compute Engine > VM Instances](https://console.cloud.google.com/compute/instances?project=project-10a84e2a-6485-49ee-825)
2. Select `vm-platform-seed`
3. Click **"▶ Start/Resume"** at the top
4. Wait until status shows **Running** (green checkmark)

**Via CLI:**
```powershell
gcloud compute instances start vm-platform-seed --zone=asia-southeast1-b
```

### Step 7.2 — Assign an External IP (if not already assigned)

Check if the VM has an external IP:
```powershell
gcloud compute instances describe vm-platform-seed --zone=asia-southeast1-b --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

If empty, add one:

1. Go to **VPC Network > IP Addresses**
2. Click **"Reserve External Static Address"**
   - Name: `smarthr-backend-ip`
   - Region: `asia-southeast1`
   - Attached to: `vm-platform-seed`
3. Click **Reserve**

Or via CLI:
```powershell
# Reserve a static IP
gcloud compute addresses create smarthr-backend-ip --region=asia-southeast1

# Attach it to the VM
gcloud compute instances add-access-config vm-platform-seed `
    --zone=asia-southeast1-b `
    --address=$(gcloud compute addresses describe smarthr-backend-ip --region=asia-southeast1 --format="get(address)")
```

### Step 7.3 — SSH into `vm-platform-seed`

**Via Console:** Click the **SSH** button next to the VM in Compute Engine dashboard.

**Via CLI:**
```powershell
gcloud compute ssh vm-platform-seed --zone=asia-southeast1-b
```

### Step 7.4 — Install Java 25, Node.js, and PM2

Run these commands **inside the VM** (after SSH):

```bash
# ============================================
# System Update
# ============================================
sudo apt update && sudo apt upgrade -y

# ============================================
# Install Java 25 (JDK)
# ============================================
# Java 25 is not in default apt repos — install via SDKMAN or manual download
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 25-open
java -version
# Expected: openjdk version "25.x.x"

# ============================================
# Install Node.js 20 LTS
# ============================================
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v

# ============================================
# Install PM2 (Process Manager for JARs)
# ============================================
sudo npm install -g pm2
pm2 -v

# ============================================
# Install additional tools
# ============================================
sudo apt install -y git curl unzip wget

# ============================================
# Create SmartHR application directories
# ============================================
sudo mkdir -p /opt/smarthr/apps
sudo mkdir -p /opt/smarthr/logs
sudo chown -R $USER:$USER /opt/smarthr

echo "✅ VM environment setup complete!"
```

### Step 7.5 — Set VM Access Scopes for GCS

Ensure the VM can access Cloud Storage (for document-service):

1. Go to **Compute Engine > VM Instances**
2. **Stop** the VM first (required to change scopes)
3. Click the VM name → **Edit**
4. Scroll to **Identity and API access**
5. Change **Access scopes** to: **"Allow full access to all Cloud APIs"**
6. Click **Save**
7. **Start** the VM again

---

## Phase 8: Build Microservices Locally

### Step 8.1 — Build All JARs

Run this in PowerShell **on your local Windows machine**:

```powershell
$baseDir = "c:\ijse\eca\HR-Management"

# ============================================
# Build Backend Platform (parent POM + submodules)
# Config Server, Eureka Server, API Gateway
# ============================================
Write-Host "========================================" -ForegroundColor Green
Write-Host " Building smarthr-backend-platform..." -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Green
Push-Location "$baseDir\smarthr-backend-platform"
mvn clean package -DskipTests
Pop-Location

# ============================================
# Build Business Microservices
# ============================================
$services = @("smarthr-user-service", "smarthr-employee-service", "smarthr-document-service")

foreach ($svc in $services) {
    $svcPath = "$baseDir\$svc"
    if (Test-Path $svcPath) {
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host " Building $svc..." -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Push-Location $svcPath
        mvn clean package -DskipTests
        Pop-Location
    } else {
        Write-Host "⚠️  $svc not found at $svcPath" -ForegroundColor Yellow
    }
}

Write-Host "`n✅ All builds complete!" -ForegroundColor Green
```

### Step 8.2 — Verify JAR Files

```powershell
# List all generated JARs
Get-ChildItem -Path "c:\ijse\eca\HR-Management\*\target\*.jar" -Exclude "*-sources.jar","*-javadoc.jar","*-original.jar" | 
    ForEach-Object { Write-Host "$($_.Name) - $([math]::Round($_.Length/1MB, 2)) MB" }
```

Expected output (approximately):
```
config-server-0.0.1-SNAPSHOT.jar - ~40 MB
eureka-server-0.0.1-SNAPSHOT.jar - ~55 MB
api-gateway-0.0.1-SNAPSHOT.jar - ~50 MB
user-service-0.0.1-SNAPSHOT.jar - ~60 MB
employee-service-0.0.1-SNAPSHOT.jar - ~60 MB
document-service-0.0.1-SNAPSHOT.jar - ~55 MB
```

---

## Phase 9: Upload & Deploy JARs to VM

### Step 9.1 — Create Staging Bucket for Deployment

```powershell
gcloud storage buckets create gs://smarthr-deploy-staging --location=asia-southeast1
```

### Step 9.2 — Upload All JARs to Staging Bucket

```powershell
# Upload all generated JARs
$jarPaths = @(
    "c:\ijse\eca\HR-Management\smarthr-backend-platform\config-server\target",
    "c:\ijse\eca\HR-Management\smarthr-backend-platform\eureka-server\target",
    "c:\ijse\eca\HR-Management\smarthr-backend-platform\api-gateway\target"
)

# Add business services if they exist
@("smarthr-user-service", "smarthr-employee-service", "smarthr-document-service") | ForEach-Object {
    $p = "c:\ijse\eca\HR-Management\$_\target"
    if (Test-Path $p) { $jarPaths += $p }
}

foreach ($dir in $jarPaths) {
    $jar = Get-ChildItem "$dir\*.jar" -Exclude "*-sources.jar","*-javadoc.jar","*-original.jar" | Select-Object -First 1
    if ($jar) {
        Write-Host "Uploading $($jar.Name)..." -ForegroundColor Cyan
        gcloud storage cp $jar.FullName "gs://smarthr-deploy-staging/$($jar.Name)"
    }
}

Write-Host "`n✅ All JARs uploaded!" -ForegroundColor Green

# Verify uploads
gcloud storage ls gs://smarthr-deploy-staging/
```

### Step 9.3 — Download JARs on VM

SSH into `vm-platform-seed` and download:

```bash
cd /opt/smarthr/apps

# Download all JARs from staging bucket
gsutil cp gs://smarthr-deploy-staging/*.jar .

# Rename JARs for cleaner names (optional but recommended)
# Adjust filenames to match your actual JAR names
for jar in *.jar; do
    echo "Downloaded: $jar ($(du -h $jar | cut -f1))"
done

ls -la /opt/smarthr/apps/
```

---

## Phase 10: Create PM2 Ecosystem Configuration

### Step 10.1 — Create `ecosystem.config.js`

Run this **inside `vm-platform-seed`** via SSH:

```bash
cat << 'ECOSYSTEM' > /opt/smarthr/ecosystem.config.js
module.exports = {
  apps: [
    // ======================================
    // 1. CONFIG SERVER (Start First)
    // Provides centralized configuration
    // ======================================
    {
      name: 'config-server',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/config-server-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8888',
        SPRING_PROFILES_ACTIVE: 'native'
      },
      out_file: '/opt/smarthr/logs/config-server-out.log',
      error_file: '/opt/smarthr/logs/config-server-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    },

    // ======================================
    // 2. EUREKA SERVER (Start Second)
    // Service discovery registry
    // ======================================
    {
      name: 'eureka-server',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/eureka-server-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8761',
        EUREKA_HOSTNAME: 'localhost',
        CONFIG_SERVER_URL: 'http://localhost:8888'
      },
      out_file: '/opt/smarthr/logs/eureka-server-out.log',
      error_file: '/opt/smarthr/logs/eureka-server-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    },

    // ======================================
    // 3. USER SERVICE
    // Handles authentication, user management
    // Connects to Cloud SQL PostgreSQL
    // ======================================
    {
      name: 'user-service',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/user-service-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8081',
        CONFIG_SERVER_URL: 'http://localhost:8888',
        EUREKA_SERVER_URL: 'http://localhost:8761/eureka/',
        SPRING_DATASOURCE_URL: 'jdbc:postgresql://10.55.192.2:5432/smarthr_user_db',
        DATABASE_USERNAME: 'smarthr_app',
        DATABASE_PASSWORD: 'SmartHR@2026!Secure',
        JWT_SECRET: '8d414e13c9cb1ccc8b88c38d4f3ec22f8318473a980d9134bda2d8d783d73bc7'
      },
      out_file: '/opt/smarthr/logs/user-service-out.log',
      error_file: '/opt/smarthr/logs/user-service-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    },

    // ======================================
    // 4. EMPLOYEE SERVICE
    // Handles employees, departments, attendance, leaves, salaries
    // Connects to Cloud SQL PostgreSQL
    // ======================================
    {
      name: 'employee-service',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/employee-service-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8082',
        CONFIG_SERVER_URL: 'http://localhost:8888',
        EUREKA_SERVER_URL: 'http://localhost:8761/eureka/',
        SPRING_DATASOURCE_URL: 'jdbc:postgresql://10.55.192.2:5432/smarthr_employee_db',
        SPRING_DATASOURCE_USERNAME: 'smarthr_app',
        SPRING_DATASOURCE_PASSWORD: 'SmartHR@2026!Secure'
      },
      out_file: '/opt/smarthr/logs/employee-service-out.log',
      error_file: '/opt/smarthr/logs/employee-service-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    },

    // ======================================
    // 5. DOCUMENT SERVICE
    // Handles file uploads and document management
    // Connects to MongoDB Atlas + GCS Bucket
    // ======================================
    {
      name: 'document-service',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/document-service-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8083',
        CONFIG_SERVER_URL: 'http://localhost:8888',
        EUREKA_SERVER_URL: 'http://localhost:8761/eureka/',
        MONGO_URI: 'mongodb+srv://buddhikaaloka2005_db_user:pEvgEDUlw0Qbrh99@hr-management.r3kcaqr.mongodb.net/smarthr_document_db?appName=hr-management',
        GCP_PROJECT_ID: 'project-10a84e2a-6485-49ee-825',
        GCS_BUCKET_NAME: 'smarthr-documents-project-10a84e2a'
      },
      out_file: '/opt/smarthr/logs/document-service-out.log',
      error_file: '/opt/smarthr/logs/document-service-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    },

    // ======================================
    // 6. API GATEWAY (Start Last)
    // Routes all external requests to microservices
    // ======================================
    {
      name: 'api-gateway',
      script: 'java',
      args: [
        '-Xms256m', '-Xmx512m',
        '-jar', '/opt/smarthr/apps/api-gateway-0.0.1-SNAPSHOT.jar'
      ],
      cwd: '/opt/smarthr',
      env: {
        SERVER_PORT: '8080',
        CONFIG_SERVER_URL: 'http://localhost:8888',
        EUREKA_SERVER_URL: 'http://localhost:8761/eureka/',
        JWT_SECRET: '8d414e13c9cb1ccc8b88c38d4f3ec22f8318473a980d9134bda2d8d783d73bc7'
      },
      out_file: '/opt/smarthr/logs/api-gateway-out.log',
      error_file: '/opt/smarthr/logs/api-gateway-error.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
    }
  ]
};
ECOSYSTEM

echo "✅ ecosystem.config.js created at /opt/smarthr/ecosystem.config.js"
```

> [!WARNING]
> **Update JAR filenames** if your actual JAR names differ from the defaults (e.g., `config-server-0.0.1-SNAPSHOT.jar`). Check with `ls /opt/smarthr/apps/`.
>
> **Update `DATABASE_PASSWORD`** if you chose a different password in Phase 3.

---

## Phase 11: Start All Microservices

### Step 11.1 — Sequential Startup Script

Run this **inside `vm-platform-seed`** via SSH:

```bash
cd /opt/smarthr

echo "============================================"
echo " Starting SmartHR Microservices Platform"
echo "============================================"

# 1. Config Server (must start first — all services depend on it)
echo "▶ Starting Config Server..."
pm2 start ecosystem.config.js --only config-server
echo "⏳ Waiting 15 seconds for Config Server to initialize..."
sleep 15

# 2. Eureka Server (service discovery — services register here)
echo "▶ Starting Eureka Server..."
pm2 start ecosystem.config.js --only eureka-server
echo "⏳ Waiting 15 seconds for Eureka Server to initialize..."
sleep 15

# 3. Business Services (can start in parallel)
echo "▶ Starting User Service..."
pm2 start ecosystem.config.js --only user-service
echo "▶ Starting Employee Service..."
pm2 start ecosystem.config.js --only employee-service
echo "▶ Starting Document Service..."
pm2 start ecosystem.config.js --only document-service
echo "⏳ Waiting 20 seconds for all services to register with Eureka..."
sleep 20

# 4. API Gateway (start last — needs all services registered)
echo "▶ Starting API Gateway..."
pm2 start ecosystem.config.js --only api-gateway

echo ""
echo "============================================"
echo " ✅ All services started!"
echo "============================================"

# Show status
pm2 status
```

### Step 11.2 — Enable Auto-Restart on VM Reboot

```bash
# Save current PM2 process list
pm2 save

# Generate startup script for systemd
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u $USER --hp /home/$USER

# Verify saved
pm2 save
```

### Step 11.3 — Useful PM2 Commands

```bash
# View all services status
pm2 status

# View real-time logs (all services)
pm2 logs

# View logs for a specific service
pm2 logs user-service
pm2 logs config-server

# Restart a specific service
pm2 restart user-service

# Restart all services
pm2 restart all

# Stop all services
pm2 stop all

# Delete all processes and start fresh
pm2 delete all
```

---

## Phase 12: Deploy Frontend to Cloud Run

### Step 12.1 — Create Frontend `Dockerfile`

In your frontend repository root (e.g., `c:\ijse\eca\HR-Management\smarthr-frontend\`), create `Dockerfile`:

```dockerfile
# ============================================
# Stage 1: Build the React/Vite application
# ============================================
FROM node:20-alpine AS build
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --production=false

# Copy source and build
COPY . .
ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL
RUN npm run build

# ============================================
# Stage 2: Serve with Nginx
# ============================================
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Step 12.2 — Create Frontend `nginx.conf`

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Handle React Router (SPA routing)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Health check endpoint
    location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

### Step 12.3 — Deploy to Cloud Run

First enable the APIs (if not done yet):
```powershell
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
```

Then deploy:
```powershell
# Replace <VM-EXTERNAL-IP> with the external IP of vm-platform-seed
Push-Location "c:\ijse\eca\HR-Management\smarthr-frontend"

gcloud run deploy smarthr-frontend `
    --source . `
    --region asia-southeast1 `
    --allow-unauthenticated `
    --port 80 `
    --set-env-vars="VITE_API_BASE_URL=http://<VM-EXTERNAL-IP>:8080" `
    --build-arg="VITE_API_BASE_URL=http://<VM-EXTERNAL-IP>:8080"

Pop-Location
```

> [!IMPORTANT]
> Replace `<VM-EXTERNAL-IP>` with the actual external IP of `vm-platform-seed`. Get it with:
> ```powershell
> gcloud compute instances describe vm-platform-seed --zone=asia-southeast1-b --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
> ```

After deployment, Cloud Run will give you a URL like:
```
https://smarthr-frontend-xxxxx-as.a.run.app
```

### Step 12.4 — Update API Gateway CORS for Cloud Run URL

After getting the Cloud Run URL, update [`api-gateway.yml`](file:///c:/ijse/eca/HR-Management/smarthr-backend-platform/config-server/src/main/resources/config-repo/api-gateway.yml) to add the Cloud Run URL to allowed origins:

```yaml
# Add to allowedOrigins list:
- "https://smarthr-frontend-xxxxx-as.a.run.app"
```

Then rebuild and redeploy the API Gateway JAR.

---

## Phase 13: Verification & Testing

### Step 13.1 — Get VM External IP

```powershell
$VM_IP = gcloud compute instances describe vm-platform-seed --zone=asia-southeast1-b --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
Write-Host "VM External IP: $VM_IP"
```

### Step 13.2 — Health Check All Services

```powershell
$VM_IP = "<VM-EXTERNAL-IP>"  # Replace with actual IP

# 1. API Gateway Health
Write-Host "Testing API Gateway..." -ForegroundColor Cyan
curl "http://${VM_IP}:8080/actuator/health"

# 2. Eureka Dashboard (open in browser)
Write-Host "`nEureka Dashboard:" -ForegroundColor Cyan
Write-Host "http://${VM_IP}:8761/"
Start-Process "http://${VM_IP}:8761/"
```

### Step 13.3 — Test API Endpoints

```powershell
$VM_IP = "<VM-EXTERNAL-IP>"

# Test Auth / Login
curl -X POST "http://${VM_IP}:8080/api/v1/auth/register" `
    -H "Content-Type: application/json" `
    -d '{"email":"admin@smarthr.com","password":"admin123","firstName":"Admin","lastName":"User","role":"ADMIN"}'

# Test Employee endpoint (requires JWT token)
curl "http://${VM_IP}:8080/api/v1/employees" `
    -H "Authorization: Bearer <JWT_TOKEN>"
```

### Step 13.4 — Verify on GCP Console

| Check | Where to Look | Expected |
|---|---|---|
| ✅ Cloud SQL | [SQL Console](https://console.cloud.google.com/sql?project=project-10a84e2a-6485-49ee-825) | `postgress-vm` running, 3 databases |
| ✅ VM Instance | [Compute Engine](https://console.cloud.google.com/compute/instances?project=project-10a84e2a-6485-49ee-825) | `vm-platform-seed` running with external IP |
| ✅ GCS Bucket | [Cloud Storage](https://console.cloud.google.com/storage?project=project-10a84e2a-6485-49ee-825) | `smarthr-documents-project-10a84e2a` bucket |
| ✅ Frontend | [Cloud Run](https://console.cloud.google.com/run?project=project-10a84e2a-6485-49ee-825) | `smarthr-frontend` serving HTTPS URL |
| ✅ Eureka | Browser: `http://<VM-IP>:8761` | All 4 services registered |

---

## 📊 Final Resource Summary

| GCP Resource | Instance Name | Purpose | Cost Tier |
|---|---|---|---|
| **Cloud SQL PostgreSQL** | `postgress-vm` | `smarthr_user_db` + `smarthr_employee_db` | `db-custom-2-8192` (~$0.20/hr) |
| **Compute Engine VM** | `vm-platform-seed` | All 6 microservices via PM2 | `e2-medium` (~$0.05/hr) |
| **Cloud Storage** | `smarthr-documents-*` | Employee document uploads | Standard (pennies/GB) |
| **MongoDB Atlas** | `hr-management` cluster | `smarthr_document_db` | M0 Free / Dedicated |
| **Cloud Run** | `smarthr-frontend` | React/Vite portal | Free tier for low traffic |

**Estimated daily cost: ~$6-8/day** (well within GCP free trial credits)

> [!CAUTION]
> **Remember to stop resources when not in use** to save credits:
> ```powershell
> # Stop the backend VM
> gcloud compute instances stop vm-platform-seed --zone=asia-southeast1-b
>
> # Note: Cloud SQL continues to charge even when idle.
> # To fully stop, go to SQL Console and click "Stop" on the instance.
> ```

---

## 🔧 Troubleshooting

### Service won't start / CrashLoopBackOff
```bash
# Check PM2 logs for the failing service
pm2 logs <service-name> --lines 100

# Common issues:
# - Wrong JAR filename in ecosystem.config.js
# - Database connection refused (check Cloud SQL IP)
# - Port already in use (pm2 delete all, then restart)
```

### Can't connect to Cloud SQL from VM
```bash
# Test PostgreSQL connectivity from VM
sudo apt install -y postgresql-client
psql -h 10.55.192.2 -U smarthr_app -d smarthr_user_db

# If connection refused, verify:
# 1. Both VMs are in the same VPC (default)
# 2. Cloud SQL has private IP enabled
# 3. Firewall allows TCP 5432 internally
```

### Eureka shows no registered services
```bash
# Check if Config Server is healthy first
curl http://localhost:8888/actuator/health

# Then check Eureka
curl http://localhost:8761/actuator/health

# Services take 30-60 seconds to register after startup
```

### Frontend can't reach API Gateway (CORS errors)
- Ensure the Cloud Run URL is added to the CORS `allowedOrigins` in `api-gateway.yml`
- Ensure `allowCredentials: false` and `allowedHeaders: "*"` are set
- Rebuild and redeploy the API Gateway JAR

---

> [!TIP]
> **Quick Reference — Useful PM2 Commands on VM:**
> ```bash
> pm2 status                    # View all services
> pm2 logs                      # Real-time combined logs
> pm2 logs user-service         # Specific service logs
> pm2 restart all               # Restart everything
> pm2 restart api-gateway       # Restart specific service
> pm2 monit                     # Interactive monitoring dashboard
> ```
