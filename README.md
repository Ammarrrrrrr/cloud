# 🎯 Essential Commands + GUI Guide with Decision Rationales

## Complete Deployment: CLI Commands, GUI Alternatives, and Why We Chose Each

---

## 📋 Table of Contents

1. [Project Setup](#1-project-setup)
2. [Network Configuration](#2-network-configuration)
3. [IAM & Service Accounts](#3-iam--service-accounts)
4. [Cloud SQL Database](#4-cloud-sql-database)
5. [NFS File Server](#5-nfs-file-server)
6. [Moodle Application](#6-moodle-application)
7. [Load Balancer](#7-load-balancer)

---

## 1. Project Setup

### Essential Commands

```bash
# Create project
gcloud projects create "moodle-eu-demo-12345" --name="Moodle EU Demo"
```

**Command Breakdown:**
- `gcloud` - Google Cloud command-line tool
- `projects create` - Creates a new GCP project
- `"moodle-eu-demo-12345"` - **Project ID** (must be globally unique)
- `--name="Moodle EU Demo"` - **Display name** (human-friendly, can have spaces)
- **What it does:** Creates container for all GCP resources
- **Why needed:** Everything in GCP exists inside a project

```bash
# Set as active project
gcloud config set project "moodle-eu-demo-12345"
```

**Command Breakdown:**
- `gcloud config set` - Modifies gcloud configuration
- `project` - Configuration property to change
- `"moodle-eu-demo-12345"` - Value to set
- **What it does:** Makes this project the default for all future commands
- **Why needed:** So you don't need `--project=...` on every command

```bash
# Link billing account
gcloud beta billing projects link "moodle-eu-demo-12345" --billing-account="XXXXXX-YYYYYY-ZZZZZZ"
```

**Command Breakdown:**
- `gcloud beta billing` - Billing API (beta version)
- `projects link` - Links billing to project
- `--billing-account="XXXXXX-YYYYYY-ZZZZZZ"` - Your billing account ID
- **What it does:** Enables paid resources (VMs, databases, etc.)
- **Why needed:** Can't create any resources without billing

**How to get billing account ID:**
```bash
gcloud beta billing accounts list
```

```bash
# Enable required APIs
gcloud services enable \
  compute.googleapis.com \
  sqladmin.googleapis.com \
  servicenetworking.googleapis.com \
  secretmanager.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com
```

**Command Breakdown:**
- `gcloud services enable` - Enables GCP APIs
- `compute.googleapis.com` - **Compute Engine** (VMs, load balancers)
- `sqladmin.googleapis.com` - **Cloud SQL** (managed databases)
- `servicenetworking.googleapis.com` - **Private service access** (for Cloud SQL private IP)
- `secretmanager.googleapis.com` - **Secret Manager** (password storage)
- `logging.googleapis.com` - **Cloud Logging** (log collection)
- `monitoring.googleapis.com` - **Cloud Monitoring** (metrics)
- **What it does:** Activates APIs so you can use those services
- **Why needed:** APIs are disabled by default (security + cost control)
- **Time:** Takes 30-60 seconds to propagate

```bash
# Set default region and zone
gcloud config set compute/region "europe-west1"
gcloud config set compute/zone "europe-west1-b"
```

**Command Breakdown:**
- `compute/region` - Default region for Compute Engine
- `compute/zone` - Default zone for Compute Engine
- `europe-west1` - Belgium data center
- `europe-west1-b` - Zone B within that region
- **What it does:** Sets defaults so you don't need `--region` and `--zone` flags
- **Why these locations:** EU data residency for GDPR compliance

### GUI Alternative

**Step 1: Create Project**
1. Go to: https://console.cloud.google.com
2. Click dropdown at top → **"New Project"**
3. Enter name: `Moodle EU Demo`
4. Click **"Create"**

**Step 2: Link Billing**
1. Navigation Menu (☰) → **"Billing"**
2. Click **"Link a billing account"**
3. Select your billing account → **"Set Account"**

**Step 3: Enable APIs**
1. Navigation Menu → **"APIs & Services"** → **"Library"**
2. Search and enable each:
   - Compute Engine API
   - Cloud SQL Admin API
   - Service Networking API
   - Secret Manager API
   - Cloud Logging API
   - Cloud Monitoring API

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Europe Region** | GDPR compliance, data residency in EU | US regions (cheaper but compliance issues) |
| **Multiple APIs** | Each service requires explicit enablement | Could use gcloud meta-service (less control) |
| **Project per deployment** | Isolation, clear billing, easy cleanup | Shared project (cheaper but messy) |

---

## 2. Network Configuration

### Essential Commands

```bash
# Create VPC
gcloud compute networks create "moodle-vpc" --subnet-mode=custom
```

**Command Breakdown:**
- `gcloud compute networks create` - Creates a Virtual Private Cloud
- `"moodle-vpc"` - Network name
- `--subnet-mode=custom` - Manual subnet creation (not automatic)
- **What it does:** Creates isolated private network for your resources
- **Result:** Virtual network that spans all GCP regions
- **Why custom mode:** Full control over IP address ranges

```bash
# Create Subnet
gcloud compute networks subnets create "moodle-subnet" \
  --network="moodle-vpc" \
  --region="europe-west1" \
  --range="10.10.0.0/20"
```

**Command Breakdown:**
- `gcloud compute networks subnets create` - Creates subnet within VPC
- `"moodle-subnet"` - Subnet name
- `--network="moodle-vpc"` - Parent VPC network
- `--region="europe-west1"` - Geographic region
- `--range="10.10.0.0/20"` - CIDR block for IP addresses
  - `10.10.0.0` = Starting IP
  - `/20` = Subnet mask providing **4,096 IP addresses**
  - Range: 10.10.0.0 to 10.10.15.255
- **What it does:** Creates IP address range in specific region
- **Why this range:** Private, non-conflicting, sufficient capacity

```bash
# Firewall: Allow SSH
gcloud compute firewall-rules create allow-ssh-admin \
  --network="moodle-vpc" \
  --allow=tcp:22 \
  --source-ranges="0.0.0.0/0"
```

**Command Breakdown:**
- `gcloud compute firewall-rules create` - Creates firewall rule
- `allow-ssh-admin` - Rule name
- `--network="moodle-vpc"` - Which VPC this applies to
- `--allow=tcp:22` - Allow TCP traffic on port 22 (SSH)
- `--source-ranges="0.0.0.0/0"` - From any IP address (entire internet)
- **What it does:** Allows you to SSH into VMs for debugging
- **Security note:** Production should restrict to specific IPs

```bash
# Firewall: Internal communication
gcloud compute firewall-rules create allow-internal \
  --network="moodle-vpc" \
  --allow=tcp,udp,icmp \
  --source-ranges="10.10.0.0/20"
```

**Command Breakdown:**
- `--allow=tcp,udp,icmp` - Allow all protocol types
  - `tcp` - Transmission Control Protocol
  - `udp` - User Datagram Protocol
  - `icmp` - Internet Control Message Protocol (ping)
- `--source-ranges="10.10.0.0/20"` - Only from our subnet
- **What it does:** VMs in subnet can talk to each other freely
- **Why needed:** App servers need to reach database, NFS, etc.

```bash
# Firewall: NFS traffic
gcloud compute firewall-rules create allow-nfs-internal \
  --network="moodle-vpc" \
  --allow=tcp:2049 \
  --source-ranges="10.10.0.0/20"
```

**Command Breakdown:**
- `--allow=tcp:2049` - Port 2049 is NFS protocol
- **What it does:** Allows app servers to mount NFS shared storage
- **Why needed:** Moodle stores uploaded files on NFS

```bash
# Firewall: Health checks (CRITICAL)
gcloud compute firewall-rules create allow-health-checks \
  --network="moodle-vpc" \
  --allow=tcp:80 \
  --source-ranges="130.211.0.0/22,35.191.0.0/16"
```

**Command Breakdown:**
- `--source-ranges="130.211.0.0/22,35.191.0.0/16"` - **Google's health check IP ranges**
  - These are the IPs Google uses to check if your VMs are healthy
- `--allow=tcp:80` - HTTP traffic for health checks
- **What it does:** Allows load balancer to ping your VMs
- **⚠️ CRITICAL:** Without this, load balancer shows "UNHEALTHY" forever
- **Why these IPs:** Google's documented health check source ranges

```bash
# Reserve IP range for private services
gcloud compute addresses create google-managed-services-moodle-vpc \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network="moodle-vpc"
```

**Command Breakdown:**
- `gcloud compute addresses create` - Reserves IP address range
- `--global` - Available globally (not regional)
- `--purpose=VPC_PEERING` - For connecting VPC to Google services
- `--prefix-length=16` - /16 subnet = 65,536 IPs
- **What it does:** Reserves IP block for Cloud SQL and other managed services
- **Why needed:** Cloud SQL needs IPs from your VPC for private access

```bash
# Create private connection to Google services
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --network="moodle-vpc" \
  --ranges=google-managed-services-moodle-vpc
```

**Command Breakdown:**
- `gcloud services vpc-peerings connect` - Creates VPC peering
- `--service=servicenetworking.googleapis.com` - Google service network
- `--network="moodle-vpc"` - Your VPC
- `--ranges=...` - Use the reserved IP range
- **What it does:** Creates private network connection to Google services
- **Result:** Cloud SQL gets private IP in your VPC (e.g., 10.128.0.3)
- **Benefit:** Faster, more secure than public internet connection

### GUI Alternative

**Step 1: Create VPC**
1. Navigation Menu → **"VPC network"** → **"VPC networks"**
2. Click **"Create VPC Network"**
3. Name: `moodle-vpc`
4. Subnet creation mode: **"Custom"**
5. Add subnet:
   - Name: `moodle-subnet`
   - Region: `europe-west1`
   - IP range: `10.10.0.0/20`
6. Click **"Create"**

**Step 2: Firewall Rules**
1. VPC network → **"Firewall"**
2. Click **"Create Firewall Rule"** (repeat for each):

   **Rule 1: SSH Access**
   - Name: `allow-ssh-admin`
   - Network: `moodle-vpc`
   - Direction: Ingress
   - Action: Allow
   - Targets: All instances
   - Source IP ranges: `0.0.0.0/0`
   - Protocols/ports: TCP `22`

   **Rule 2: Internal Traffic**
   - Name: `allow-internal`
   - Source IP ranges: `10.10.0.0/20`
   - Protocols/ports: Allow all

   **Rule 3: NFS**
   - Name: `allow-nfs-internal`
   - Source IP ranges: `10.10.0.0/20`
   - Protocols/ports: TCP `2049`

   **Rule 4: Health Checks** ⚠️ **CRITICAL**
   - Name: `allow-health-checks`
   - Source IP ranges: `130.211.0.0/22, 35.191.0.0/16`
   - Protocols/ports: TCP `80`

**Step 3: Private Service Connection**
1. VPC network → **"VPC network peering"**
2. Click **"Allocate IP range"**
3. Name: `google-managed-services-moodle-vpc`
4. IP range: Automatic (prefix /16)
5. Click **"Allocate"**
6. Click **"Create connection"**
7. Select allocated range → **"Continue"**

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Custom VPC** | Full control over IP ranges, security | Auto-mode VPC (easier but less control) |
| **10.10.0.0/20** | 4,096 IPs sufficient, doesn't conflict with common ranges | 10.0.0.0/8 (too large, wasteful) |
| **Private Service Access** | Cloud SQL gets private IP (secure, fast) | Public IP with authorized networks (less secure) |
| **Health Check IPs** | Required for load balancer to work | Cloud NAT (doesn't solve health checks) |

**⚠️ Common Mistake:** Forgetting health check firewall = Load balancer shows UNHEALTHY forever

---

## 3. IAM & Service Accounts

### Essential Commands

```bash
# Create service account
gcloud iam service-accounts create "moodle-vm-sa" --display-name="Moodle VM SA"
```

**Command Breakdown:**
- `gcloud iam service-accounts create` - Creates a service account
- `"moodle-vm-sa"` - Service account name (ID)
- `--display-name="Moodle VM SA"` - Human-friendly name
- **What it does:** Creates a "robot account" for VMs to use
- **Result:** Email like `moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com`
- **Why needed:** VMs need identity to access GCP services

```bash
# Wait for propagation (IMPORTANT)
sleep 30
```

**Command Breakdown:**
- `sleep 30` - Pause for 30 seconds
- **Why absolutely necessary:** Service account needs time to propagate across GCP infrastructure
- **Without this:** Next commands fail with "service account not found"
- **How long:** 15-60 seconds typically

```bash
# Grant permission: Access secrets
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" \
  --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**Command Breakdown:**
- `gcloud projects add-iam-policy-binding` - Grants IAM role
- `--member="serviceAccount:..."` - Who gets the permission
- `--role="roles/secretmanager.secretAccessor"` - What permission they get
- **What this role allows:** Read secrets from Secret Manager
- **Why needed:** VMs need to retrieve database password
- **Permissions granted:** Read-only access to secrets

```bash
# Grant permission: Connect to Cloud SQL
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" \
  --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

**Command Breakdown:**
- `--role="roles/cloudsql.client"` - Cloud SQL client role
- **What this role allows:** Connect to Cloud SQL via proxy
- **Why needed:** VMs run Cloud SQL Proxy to connect to database
- **Permissions granted:** Connect to any Cloud SQL instance in project

```bash
# Grant permission: Write logs
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" \
  --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

**Command Breakdown:**
- `--role="roles/logging.logWriter"` - Logging writer role
- **What this role allows:** Send logs to Cloud Logging
- **Why needed:** Application and system logs go to central location
- **Permissions granted:** Write logs, not read/delete

```bash
# Grant permission: Write metrics
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" \
  --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" \
  --role="roles/monitoring.metricWriter"
```

**Command Breakdown:**
- `--role="roles/monitoring.metricWriter"` - Monitoring writer role
- **What this role allows:** Send metrics to Cloud Monitoring
- **Why needed:** CPU, memory, custom metrics for dashboards and alerts
- **Permissions granted:** Write metrics only

**🔐 Security Note:** These four roles follow **principle of least privilege**:
- Only grant what's absolutely needed
- Read-only where possible
- No admin or editor roles

### GUI Alternative

**Step 1: Create Service Account**
1. Navigation Menu → **"IAM & Admin"** → **"Service Accounts"**
2. Click **"Create Service Account"**
3. Name: `moodle-vm-sa`
4. Description: `Moodle VM Service Account`
5. Click **"Create and Continue"**

**Step 2: Grant Roles**
6. Add these roles:
   - Secret Manager Secret Accessor
   - Cloud SQL Client
   - Logging Log Writer
   - Monitoring Metric Writer
7. Click **"Continue"** → **"Done"**

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Service Account** | VMs can authenticate to GCP services | Long-lived keys (security risk) |
| **Least Privilege** | Only grants needed permissions | Editor role (too broad, insecure) |
| **Workload Identity** | No key management required | JSON key files (can leak) |
| **Four specific roles** | Granular control, security best practice | Single powerful role (violates least privilege) |

**🔑 Key Point:** Never download service account keys. Attach service account to VM instead.

---

## 4. Cloud SQL Database

### Essential Commands

```bash
# Generate password
SQL_PASSWORD="$(openssl rand -base64 18)"
```

**Command Breakdown:**
- `openssl rand` - Generate random bytes using OpenSSL
- `-base64 18` - 18 bytes encoded as base64 (results in ~24 characters)
- `$(...)` - Command substitution (captures output)
- **What it does:** Creates cryptographically secure random password
- **Example output:** `kJ9mN2pL4qR8sT6vX3yZ1wB2`
- **Why this length:** 18 bytes = ~128 bits entropy (very secure)

```bash
# Create Cloud SQL instance
gcloud sql instances create "moodle-sql-12345" \
  --database-version=MYSQL_8_0 \
  --tier=db-g1-small \
  --region="europe-west1" \
  --network="projects/moodle-eu-demo-12345/global/networks/moodle-vpc" \
  --no-assign-ip \
  --database-flags=max_allowed_packet=1073741824,wait_timeout=10000 \
  --availability-type=zonal \
  --backup-start-time=03:00
```

**Command Breakdown:**

**Basic Configuration:**
- `gcloud sql instances create` - Creates Cloud SQL database instance
- `"moodle-sql-12345"` - Instance name (must be unique in project)
- `--database-version=MYSQL_8_0` - MySQL version 8.0
  - **Why MySQL:** Moodle's primary supported database
  - **Why 8.0:** Latest stable version
- `--region="europe-west1"` - Belgium data center
  - **Why:** Same region as VMs = lower latency

**Machine Type:**
- `--tier=db-g1-small` - Machine type
  - **Specs:** 1.7 GB RAM, 1 shared vCPU
  - **Cost:** ~$30/month
  - **⚠️ Important:** db-f1-micro causes Out Of Memory during Moodle install

**Networking:**
- `--network="projects/moodle-eu-demo-12345/global/networks/moodle-vpc"` - Connect to our VPC
  - Full resource path format required
- `--no-assign-ip` - **Do NOT create public IP**
  - **Security:** Database not accessible from internet
  - **Access method:** Only via private IP from VPC

**Database Flags (CRITICAL):**
- `--database-flags=max_allowed_packet=1073741824,wait_timeout=10000`
  - `max_allowed_packet=1073741824` - **1 GB maximum packet size**
    - **Default:** 4 MB
    - **Why increase:** Moodle sends huge queries during install/restore
    - **Without this:** "MySQL server has gone away" error
  - `wait_timeout=10000` - 10,000 seconds connection timeout
    - **Default:** 28,800 seconds
    - **Why set:** Explicit control, prevents premature disconnects

**Availability:**
- `--availability-type=zonal` - Single zone deployment
  - **Alternative:** `regional` (high availability, 2x cost)
  - **For dev/test:** Zonal sufficient
  - **For production:** Use regional

**Backups:**
- `--backup-start-time=03:00` - Daily backups at 3:00 AM
  - **Why 3 AM:** Low usage time
  - **Retention:** 7 days by default
  - **Point-in-time recovery:** Enabled automatically

**⏰ Time:** This command takes 10-15 minutes to complete

```bash
# Create database
gcloud sql databases create "moodle" --instance="moodle-sql-12345"
```

**Command Breakdown:**
- `gcloud sql databases create` - Creates database schema
- `"moodle"` - Database name
- `--instance="moodle-sql-12345"` - Which SQL instance
- **What it does:** Creates empty database ready for tables
- **Analogy:** Instance = MySQL server, Database = schema

```bash
# Create user
gcloud sql users create "moodleuser" \
  --instance="moodle-sql-12345" \
  --password="$SQL_PASSWORD"
```

**Command Breakdown:**
- `gcloud sql users create` - Creates database user
- `"moodleuser"` - Username
- `--password="$SQL_PASSWORD"` - Uses generated password
- **What it does:** Creates MySQL user account
- **Permissions:** Full access to all databases (default)
- **Will be used by:** Moodle application to connect

```bash
# Store password securely
printf "%s" "$SQL_PASSWORD" | gcloud secrets create moodle-db-password --data-file=-
```

**Command Breakdown:**
- `printf "%s" "$SQL_PASSWORD"` - Print password without newline
  - `%s` - Format as string
- `| gcloud secrets create` - Pipe to Secret Manager
- `moodle-db-password` - Secret name
- `--data-file=-` - Read from stdin (the pipe)
  - `-` means "read from standard input"
- **What it does:** Stores password in encrypted Secret Manager
- **Benefits:** 
  - Encrypted at rest
  - Audit trail (who accessed when)
  - Version controlled
  - Can rotate easily
- **Retrieve with:** `gcloud secrets versions access latest --secret=moodle-db-password`

### GUI Alternative

**Step 1: Create SQL Instance**
1. Navigation Menu → **"SQL"**
2. Click **"Create Instance"** → **"Choose MySQL"**
3. Instance ID: `moodle-sql-12345`
4. Password: Generate strong password (save it!)
5. Database version: **MySQL 8.0**
6. Choose region: **europe-west1**
7. **Configuration options:**

   **Machine type:**
   - Preset: Custom
   - Cores: 1 vCPU
   - Memory: 1.7 GB
   - (This is db-g1-small)

   **Storage:**
   - Type: SSD
   - Size: 10 GB
   - Enable automatic storage increases

   **Connections:**
   - ⚠️ **CRITICAL:** Uncheck "Public IP"
   - Check "Private IP"
   - Network: `moodle-vpc`

   **Data Protection:**
   - Automate backups: **Enabled**
   - Backup time: **03:00**
   - Availability: **Single zone** (zonal)

   **Flags:** (Click "Add flag")
   - Add: `max_allowed_packet` = `1073741824`
   - Add: `wait_timeout` = `10000`

8. Click **"Create Instance"** (takes 10-15 minutes)

**Step 2: Create Database**
1. Click on your instance → **"Databases"** tab
2. Click **"Create database"**
3. Name: `moodle`
4. Click **"Create"**

**Step 3: Create User**
1. **"Users"** tab → **"Add User Account"**
2. Username: `moodleuser`
3. Password: (use strong password)
4. Host name: Leave as `%` (any host)
5. Click **"Add"**

**Step 4: Store Password in Secret Manager**
1. Navigation Menu → **"Secret Manager"**
2. Click **"Create Secret"**
3. Name: `moodle-db-password`
4. Secret value: (paste database password)
5. Click **"Create Secret"**

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Cloud SQL** | Managed, automatic backups, HA option | Self-hosted MySQL (more work, less reliable) |
| **MySQL 8.0** | Latest stable, Moodle compatible | PostgreSQL (not Moodle's primary DB) |
| **db-g1-small** | Sufficient for dev/test, prevents OOM | db-f1-micro (crashes during install) |
| **Private IP only** | Secure, no internet exposure | Public IP (security risk) |
| **max_allowed_packet=1GB** | Prevents "gone away" errors on large queries | Default 4MB (Moodle restore fails) |
| **Secret Manager** | Secure password storage, audit trail | Hardcoded passwords (insecure) |
| **Zonal** | Cost-effective for dev/test | Regional HA (2x cost, overkill for dev) |

**⚠️ Critical Setting:** `max_allowed_packet=1073741824` - Without this, Moodle installation fails!

---

## 5. NFS File Server

### Essential Commands

```bash
# Create NFS startup script
cat > nfs-startup.sh <<'EOF'
#!/bin/bash
set -e
apt-get update
apt-get install -y nfs-kernel-server
mkdir -p /srv/moodledata
mkfs.ext4 -F /dev/disk/by-id/google-moodle-nfs-data
echo "/dev/disk/by-id/google-moodle-nfs-data /srv/moodledata ext4 defaults,nofail 0 2" >> /etc/fstab
mount -a
chown -R www-data:www-data /srv/moodledata
echo "/srv/moodledata 10.10.0.0/20(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
exportfs -ra
systemctl restart nfs-kernel-server
EOF
```

**Startup Script Breakdown (Line by Line):**

1. `#!/bin/bash` - Script runs in Bash shell
2. `set -e` - Exit immediately if any command fails
3. `apt-get update` - Update package lists
4. `apt-get install -y nfs-kernel-server` - Install NFS server
   - `-y` - Auto-confirm (don't ask)
5. `mkdir -p /srv/moodledata` - Create directory for Moodle files
   - `-p` - Create parent directories if needed
6. `mkfs.ext4 -F /dev/disk/by-id/google-moodle-nfs-data` - Format disk
   - `mkfs.ext4` - Make ext4 filesystem
   - `-F` - Force (don't ask for confirmation)
   - `/dev/disk/by-id/google-moodle-nfs-data` - Attached disk
7. `echo "..." >> /etc/fstab` - Add to auto-mount configuration
   - `/etc/fstab` - File that defines what to mount at boot
   - `defaults,nofail` - Standard options + don't fail boot if disk missing
   - `0 2` - Don't dump, check filesystem on boot
8. `mount -a` - Mount all filesystems from fstab
9. `chown -R www-data:www-data /srv/moodledata` - Set ownership
   - `www-data` - Apache/web server user
   - `-R` - Recursive (all files and subdirectories)
10. `echo "/srv/moodledata 10.10.0.0/20(...)" > /etc/exports` - Configure NFS export
    - `/srv/moodledata` - Directory to share
    - `10.10.0.0/20` - Who can access (our subnet only)
    - `rw` - Read-write access
    - `sync` - Write to disk immediately (safer, slower)
    - `no_subtree_check` - Performance optimization
    - `no_root_squash` - Don't map root to anonymous user
11. `exportfs -ra` - Reload NFS exports
    - `-r` - Re-export all
    - `-a` - All directories
12. `systemctl restart nfs-kernel-server` - Restart NFS service

```bash
# Create NFS VM
gcloud compute instances create "moodle-nfs" \
  --zone="europe-west1-b" \
  --machine-type=e2-micro \
  --network="moodle-vpc" \
  --subnet="moodle-subnet" \
  --boot-disk-size=20GB \
  --create-disk=name=moodle-nfs-data,size=50GB,type=pd-standard \
  --metadata-from-file=startup-script=nfs-startup.sh
```

**Command Breakdown:**
- `gcloud compute instances create` - Creates virtual machine
- `"moodle-nfs"` - VM name
- `--zone="europe-west1-b"` - Specific zone in region
- `--machine-type=e2-micro` - Smallest VM type
  - **Specs:** 0.25-2 vCPU (shared), 1 GB RAM
  - **Cost:** ~$7/month
  - **Why sufficient:** File serving is not CPU-intensive
- `--network="moodle-vpc"` - Connect to our VPC
- `--subnet="moodle-subnet"` - Use our subnet
- `--boot-disk-size=20GB` - OS disk size
  - **Type:** Standard persistent disk (default)
  - **Usage:** Operating system only
- `--create-disk=name=moodle-nfs-data,size=50GB,type=pd-standard` - Attach data disk
  - `name=moodle-nfs-data` - Disk name (referenced in startup script)
  - `size=50GB` - Data disk size
  - `type=pd-standard` - Standard persistent disk (not SSD)
    - **Why standard:** Cheaper, sufficient performance for file storage
    - **Cost:** ~$8/month vs SSD $25/month
- `--metadata-from-file=startup-script=nfs-startup.sh` - Attach startup script
  - Script runs automatically when VM boots
  - **Why this method:** Avoids shell escaping issues

**What this creates:**
- 1 VM running NFS server
- 2 disks: 20GB boot + 50GB data
- NFS share accessible at `<VM_IP>:/srv/moodledata`

```bash
# Get NFS IP (wait 30 seconds first)
sleep 30
NFS_IP=$(gcloud compute instances describe "moodle-nfs" \
  --zone="europe-west1-b" \
  --format="value(networkInterfaces[0].networkIP)")
echo "NFS IP: $NFS_IP"
```

**Command Breakdown:**
- `sleep 30` - Wait for VM to start
- `gcloud compute instances describe` - Get VM details
- `--format="value(networkInterfaces[0].networkIP)"` - Extract private IP only
  - `networkInterfaces[0]` - First network interface
  - `networkIP` - Internal IP address
- **Result:** IP like `10.10.0.5`
- **Why needed:** App servers need this IP to mount NFS share

### GUI Alternative

**Step 1: Create VM**
1. Navigation Menu → **"Compute Engine"** → **"VM instances"**
2. Click **"Create Instance"**
3. Name: `moodle-nfs`
4. Region: `europe-west1`
5. Zone: `europe-west1-b`
6. Machine type: **e2-micro** (2 vCPU, 1 GB)
7. Boot disk: Debian 12, 20 GB
8. **Additional disks:**
   - Click "Add new disk"
   - Name: `moodle-nfs-data`
   - Type: Standard persistent disk
   - Size: 50 GB
   - Click "Save"
9. **Networking:**
   - Network: `moodle-vpc`
   - Subnet: `moodle-subnet`
   - External IP: None
10. **Management → Automation:**
    - Paste startup script (from above)
11. Click **"Create"**

**Step 2: Get NFS IP**
1. Click on the instance name
2. Note the **Internal IP** (e.g., `10.10.0.5`)
3. **Save this IP** - you'll need it for Moodle VMs

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Custom NFS VM** | Low cost, simple setup | Filestore (expensive: $200+/month) |
| **e2-micro** | Sufficient for file serving, very cheap | Larger instance (wasteful) |
| **50GB Standard Disk** | Good balance of cost/performance | SSD (3x cost for marginal benefit) |
| **NFS Protocol** | Native Linux support, simple | GlusterFS (complex, overkill) |
| **Single NFS server** | Simplicity for dev/test | Multiple NFS servers (HA but complex) |
| **10.10.0.0/20 export** | Only our subnet can access | 0.0.0.0/0 (security risk) |

**💰 Cost Comparison:**
- Custom NFS VM: ~$15/month
- Filestore: ~$200/month
- **Savings: $185/month** (92% cheaper)

**📝 Production Note:** For production, migrate to Filestore for HA and better performance.

---

## 6. Moodle Application

### Essential Commands

```bash
# Create Moodle startup script
cat > moodle-startup.sh <<'EOF'
#!/bin/bash
set -e
apt-get update
apt-get install -y apache2 php php-mysql php-xml php-curl php-zip php-gd php-intl php-mbstring php-soap nfs-common unzip curl
```

**Startup Script - Package Installation:**
- `apt-get update` - Refresh package lists
- `apt-get install -y` - Install packages (-y = auto-confirm)

**Packages Explained:**
- `apache2` - **Web server** (serves HTTP requests)
- `php` - **PHP language** (Moodle is written in PHP)
- `php-mysql` - **MySQL connector** (connects PHP to database)
- `php-xml` - **XML processing** (required by Moodle)
- `php-curl` - **HTTP client** (for external requests)
- `php-zip` - **ZIP handling** (backup/restore functionality)
- `php-gd` - **Image manipulation** (resize user photos, thumbnails)
- `php-intl` - **Internationalization** (multi-language support)
- `php-mbstring` - **Multibyte strings** (Unicode, non-English characters)
- `php-soap` - **SOAP web services** (some Moodle plugins need this)
- `nfs-common` - **NFS client** (mount network shares)
- `unzip` - **Extract archives** (for Moodle download)
- `curl` - **Download files** (get Moodle tarball)

```bash
# Fix: Increase max_input_vars
sed -i 's/;max_input_vars = 1000/max_input_vars = 5000/' /etc/php/*/apache2/php.ini
sed -i 's/;max_input_vars = 1000/max_input_vars = 5000/' /etc/php/*/cli/php.ini
```

**Command Breakdown:**
- `sed -i` - Edit file in-place (modify actual file)
- `'s/old/new/'` - Substitute old text with new
- `;max_input_vars = 1000` - **Old:** Commented out (disabled)
- `max_input_vars = 5000` - **New:** Enabled with value 5000
- `/etc/php/*/apache2/php.ini` - PHP config for Apache
- `/etc/php/*/cli/php.ini` - PHP config for command-line
- `*` - Wildcard (matches any PHP version directory)
- **Why needed:** Moodle installation form has >4,000 input fields
- **Without this:** Installation fails with "max_input_vars exceeded"
- **Default:** 1000 (too low for Moodle)

```bash
# Mount NFS
mkdir -p /var/moodledata
echo "10.10.0.5:/srv/moodledata /var/moodledata nfs defaults,_netdev 0 0" >> /etc/fstab
mount -a
chown -R www-data:www-data /var/moodledata
```

**Command Breakdown:**
- `mkdir -p /var/moodledata` - Create mount point
- `echo "..." >> /etc/fstab` - Add to auto-mount file
  - `10.10.0.5:/srv/moodledata` - **Remote:** NFS server and share
  - `/var/moodledata` - **Local:** Where to mount
  - `nfs` - Filesystem type
  - `defaults,_netdev` - Standard options + wait for network
  - `0 0` - Don't dump, don't fsck
- `mount -a` - Mount everything in fstab
- `chown -R www-data:www-data` - Make web server the owner
- **Result:** All Moodle VMs share same files

```bash
# Cloud SQL Proxy
curl -o /usr/local/bin/cloud-sql-proxy -L https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.4/cloud-sql-proxy.linux.amd64
chmod +x /usr/local/bin/cloud-sql-proxy
```

**Command Breakdown:**
- `curl -o /usr/local/bin/cloud-sql-proxy` - Download to specific location
  - `-o` - Output filename
  - `-L` - Follow redirects
  - `/usr/local/bin/` - Standard location for executables
- `chmod +x` - Make file executable
  - `+x` - Add execute permission
- **What is Cloud SQL Proxy:** 
  - Secure tunnel to Cloud SQL
  - Listens on localhost:3306
  - Encrypts all traffic
  - Handles authentication automatically

```bash
cat > /etc/systemd/system/cloud-sql-proxy.service <<SERVICE
[Unit]
Description=Cloud SQL Auth Proxy
After=network-online.target
[Service]
ExecStart=/usr/local/bin/cloud-sql-proxy moodle-eu-demo-12345:europe-west1:moodle-sql-12345 --port 3306 --private-ip
Restart=always
User=root
[Install]
WantedBy=multi-user.target
SERVICE
```

**Systemd Service File Breakdown:**
- `[Unit]` section:
  - `Description` - What this service does
  - `After=network-online.target` - Start after network is ready
- `[Service]` section:
  - `ExecStart` - Command to run
    - `moodle-eu-demo-12345:europe-west1:moodle-sql-12345` - **Connection string**
      - Format: `PROJECT:REGION:INSTANCE`
    - `--port 3306` - Listen on MySQL default port
    - `--private-ip` - **Use private IP** (not public)
  - `Restart=always` - Auto-restart if crashes
  - `User=root` - Run as root user
- `[Install]` section:
  - `WantedBy=multi-user.target` - Start automatically at boot
- **What this does:** Creates a background service that maintains DB connection

```bash
systemctl daemon-reload
systemctl enable --now cloud-sql-proxy
```

**Command Breakdown:**
- `systemctl daemon-reload` - Tell systemd about new service file
- `systemctl enable --now` - Enable at boot AND start immediately
  - `enable` - Start on boot
  - `--now` - Start right now
  - `cloud-sql-proxy` - Service name
- **Result:** Proxy running, MySQL available on localhost:3306

```bash
# Download Moodle
cd /var/www/html
if [ ! -f config.php ]; then
  rm -f index.html
  curl -L -o moodle.tgz https://download.moodle.org/download.php/direct/stable404/moodle-latest-404.tgz
  tar -xzf moodle.tgz --strip-components=1
  chown -R www-data:www-data /var/www/html
fi
```

**Command Breakdown:**
- `cd /var/www/html` - Go to web root directory
- `if [ ! -f config.php ]` - If Moodle not already installed
  - `! -f` - File does not exist
- `rm -f index.html` - Remove default Apache page
- `curl -L -o moodle.tgz ...` - Download Moodle
  - `-L` - Follow redirects
  - `-o moodle.tgz` - Save as moodle.tgz
  - URL downloads Moodle 4.4 (latest stable)
- `tar -xzf moodle.tgz --strip-components=1` - Extract archive
  - `-x` - Extract
  - `-z` - Decompress (gzip)
  - `-f` - From file
  - `--strip-components=1` - Remove top directory (moodle/)
    - **Without this:** Files in /var/www/html/moodle/
    - **With this:** Files directly in /var/www/html/
- `chown -R www-data:www-data` - Set web server as owner

```bash
a2enmod rewrite
systemctl restart apache2
EOF
```

**Command Breakdown:**
- `a2enmod rewrite` - Enable Apache mod_rewrite module
  - **What it does:** Allows URL rewriting
  - **Why needed:** Moodle uses pretty URLs (e.g., /course/view.php?id=1 → /course/1)
- `systemctl restart apache2` - Restart web server
  - **Why:** Apply configuration changes

---

```bash
# Create instance template
gcloud compute instance-templates create "moodle-template" \
  --machine-type=e2-small \
  --network="moodle-vpc" \
  --subnet="moodle-subnet" \
  --service-account="moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --metadata-from-file=startup-script=moodle-startup.sh \
  --tags=moodle-app \
  --boot-disk-size=20GB
```

**Command Breakdown:**
- `gcloud compute instance-templates create` - Creates VM blueprint
- `"moodle-template"` - Template name
- `--machine-type=e2-small` - VM size
  - **Specs:** 2 vCPU, 2 GB RAM
  - **Cost:** ~$15/month per VM
  - **Why e2-small:** Good balance for web serving
  - **Why not smaller:** e2-micro too slow
  - **Why not larger:** Wasteful for Moodle
- `--network="moodle-vpc"` - Which VPC
- `--subnet="moodle-subnet"` - Which subnet
- `--service-account="..."` - Attach service account
  - **Full email format required**
- `--scopes=https://www.googleapis.com/auth/cloud-platform` - API access
  - **What it grants:** Full access to all Cloud APIs
  - **Alternative:** Individual scopes (more complex)
- `--metadata-from-file=startup-script=moodle-startup.sh` - Attach startup script
  - Script runs every time VM starts
- `--tags=moodle-app` - Network tag
  - **Used by:** Firewall rules to identify VMs
  - **Used by:** Load balancer to find backends
- `--boot-disk-size=20GB` - OS disk size
- **What this creates:** Reusable VM configuration (not an actual VM yet)

```bash
# Create managed instance group
gcloud compute instance-groups managed create "moodle-mig" \
  --base-instance-name=moodle \
  --template="moodle-template" \
  --size=1 \
  --zone="europe-west1-b"
```

**Command Breakdown:**
- `gcloud compute instance-groups managed create` - Creates auto-scaling group
- `"moodle-mig"` - Group name
- `--base-instance-name=moodle` - VM name prefix
  - **Result:** VMs named `moodle-abcd`, `moodle-efgh`, etc.
- `--template="moodle-template"` - Which template to use
- `--size=1` - Start with 1 VM
- `--zone="europe-west1-b"` - Zone for VMs
- **What this creates:** Group of identical VMs based on template
- **Features:** Auto-scaling, self-healing, rolling updates

```bash
# Configure auto-scaling
gcloud compute instance-groups managed set-autoscaling "moodle-mig" \
  --zone="europe-west1-b" \
  --min-num-replicas=1 \
  --max-num-replicas=3 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=60
```

**Command Breakdown:**
- `set-autoscaling` - Configure auto-scaling policy
- `--min-num-replicas=1` - **Never** go below 1 VM
  - **Why:** Always need at least one instance running
- `--max-num-replicas=3` - **Never** exceed 3 VMs
  - **Why:** Cost control, prevent runaway scaling
- `--target-cpu-utilization=0.6` - Scale when CPU > 60%
  - **0.6** = 60%
  - **Why 60%:** Good balance (responsive but not wasteful)
  - **What happens:** 
    - CPU > 60% for ~60 seconds → Add VM
    - CPU < 60% for ~10 minutes → Remove VM
- `--cool-down-period=60` - Wait 60 seconds between scaling actions
  - **Why:** Prevents rapid scaling (thrashing)
  - **Example:** Add VM, wait 60s, check again

**Auto-Scaling Examples:**
- Normal load (30% CPU): 1 VM running
- Exam starts (70% CPU): Adds VM #2, then VM #3 if needed
- Exam ends (40% CPU): Slowly removes extra VMs back to 1

**💰 Cost Impact:**
- Normal: 1 VM = $15/month
- Peak: 3 VMs for 2 days = $15 + $2 extra
- Without auto-scaling (always 3): $45/month
- **Savings:** $30/month (67%)

### GUI Alternative

**Step 1: Create Instance Template**
1. Navigation Menu → **"Compute Engine"** → **"Instance templates"**
2. Click **"Create Instance Template"**
3. Name: `moodle-template`
4. Machine type: **e2-small** (2 vCPU, 2 GB)
5. Boot disk: Debian 12, 20 GB
6. **Identity and API access:**
   - Service account: `moodle-vm-sa`
   - Access scopes: **Allow full access to all Cloud APIs**
7. **Networking:**
   - Network: `moodle-vpc`
   - Subnet: `moodle-subnet`
   - External IP: None
   - Network tags: `moodle-app`
8. **Management → Automation:**
   - Paste startup script (update NFS IP: `10.10.0.5`)
9. Click **"Create"**

**Step 2: Create Managed Instance Group**
1. **"Instance groups"** → **"Create Instance Group"**
2. Name: `moodle-mig`
3. Location: **Single zone**
4. Zone: `europe-west1-b`
5. Instance template: `moodle-template`
6. **Autoscaling:**
   - Mode: **Autoscale**
   - Minimum: `1`
   - Maximum: `3`
   - Autoscaling metrics:
     - Metric type: **CPU utilization**
     - Target: `60%`
   - Cool-down period: `60` seconds
7. Click **"Create"**

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Apache + PHP** | Moodle's recommended stack | Nginx + PHP-FPM (more complex) |
| **PHP 8.2** | Latest stable, best performance | PHP 7.4 (end of life) |
| **e2-small** | Good balance for web serving | e2-micro (too small, slow) |
| **Managed Instance Group** | Auto-scaling, self-healing | Single VM (no HA, manual scaling) |
| **Auto-scale 1-3** | Handles traffic spikes, controls cost | Always 3 VMs (wastes money) |
| **60% CPU target** | Responsive scaling without over-provisioning | 80% (too aggressive, poor UX) |
| **Cloud SQL Proxy** | Secure, automatic auth, encrypted | Direct connection (requires IP whitelisting) |
| **NFS mount** | Shared storage across all VMs | Local storage (files not shared) |
| **max_input_vars=5000** | Moodle install requires this | Default 1000 (installation fails) |

**🔄 Auto-Scaling Behavior:**
- 1 VM normally (low cost)
- CPU > 60% for 60s → Add VM
- CPU < 60% for 10 min → Remove VM
- Exam time traffic spike → Auto-scales to 3 VMs
- After exam → Scales back to 1 VM

---

## 7. Load Balancer

### Essential Commands

```bash
# Create health check
gcloud compute health-checks create http moodle-hc \
  --port=80 \
  --request-path=/login/index.php \
  --global
```

**Command Breakdown:**
- `gcloud compute health-checks create http` - Creates HTTP health check
- `moodle-hc` - Health check name
- `--port=80` - Check port 80 (HTTP)
- `--request-path=/login/index.php` - ⚠️ **CRITICAL PATH**
  - **What Google does:** Makes HTTP GET request to this URL every few seconds
  - **Expected response:** 200 OK
  - **If 200 OK:** VM is HEALTHY → receives traffic
  - **If not 200:** VM is UNHEALTHY → no traffic
- `--global` - Available globally (not regional)

**⚠️ Why /login/index.php and not / ?**
- `/` → Returns 303 redirect → Health check interprets as FAILURE
- `/login/index.php` → Returns 200 OK → Health check PASSES
- **Critical decision:** This single path choice makes or breaks deployment

**What health checks do:**
- Check every VM every 5-10 seconds
- If VM fails 2-3 checks → Mark UNHEALTHY
- If UNHEALTHY → Load balancer stops sending traffic
- If VM recovers → Mark HEALTHY again

```bash
# Create backend service
gcloud compute backend-services create moodle-backend \
  --protocol=HTTP \
  --health-checks=moodle-hc \
  --global
```

**Command Breakdown:**
- `gcloud compute backend-services create` - Creates backend pool
- `moodle-backend` - Backend service name
- `--protocol=HTTP` - Use HTTP protocol
  - **Alternative:** HTTPS (requires SSL certificate)
- `--health-checks=moodle-hc` - Link to health check
- `--global` - Global load balancer
- **What it does:** Creates container for backend VMs
- **Think of it as:** Group of servers that can handle requests

```bash
# Add instance group to backend
gcloud compute backend-services add-backend moodle-backend \
  --instance-group="moodle-mig" \
  --instance-group-zone="europe-west1-b" \
  --global
```

**Command Breakdown:**
- `add-backend` - Add VMs to backend service
- `--instance-group="moodle-mig"` - Our managed instance group
- `--instance-group-zone` - Where the group is located
- **What it does:** Tells load balancer which VMs can serve traffic
- **Result:** Load balancer now knows about our auto-scaling group

```bash
# Create URL map
gcloud compute url-maps create moodle-map \
  --default-service=moodle-backend
```

**Command Breakdown:**
- `gcloud compute url-maps create` - Creates routing rules
- `moodle-map` - URL map name
- `--default-service=moodle-backend` - Default target for all traffic
- **What it does:** Routes all requests to moodle-backend
- **Advanced use:** Can route /api/* to one backend, /static/* to another

```bash
# Reserve external IP
gcloud compute addresses create moodle-lb-ip --global
```

**Command Breakdown:**
- `gcloud compute addresses create` - Reserves IP address
- `moodle-lb-ip` - IP reservation name
- `--global` - Global anycast IP
  - **What anycast means:** Same IP works worldwide
  - **Routing:** Traffic goes to nearest Google point-of-presence
- **What it does:** Reserves static external IP
- **Why reserve:** IP doesn't change if you recreate load balancer

```bash
# Get the IP
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')
echo "Load Balancer IP: $LB_IP"
```

**Command Breakdown:**
- `describe` - Get details about resource
- `--format='value(address)'` - Extract just the IP address
  - **Without format:** Shows JSON with all details
  - **With format:** Shows just `34.107.234.56`
- `$(...)` - Capture output into variable
- **Result:** `LB_IP="34.107.234.56"`

```bash
# Create HTTP proxy
gcloud compute target-http-proxies create moodle-http-proxy \
  --url-map=moodle-map
```

**Command Breakdown:**
- `gcloud compute target-http-proxies create` - Creates HTTP proxy
- `moodle-http-proxy` - Proxy name
- `--url-map=moodle-map` - Links to routing rules
- **What it does:** Terminates HTTP connections, routes based on URL map
- **In the chain:** Internet → Proxy → URL Map → Backend → VMs

```bash
# Create forwarding rule
gcloud compute forwarding-rules create moodle-http-fr \
  --global \
  --address=moodle-lb-ip \
  --target-http-proxy=moodle-http-proxy \
  --ports=80
```

**Command Breakdown:**
- `gcloud compute forwarding-rules create` - Creates forwarding rule (final piece)
- `moodle-http-fr` - Forwarding rule name
- `--address=moodle-lb-ip` - Use our reserved IP
- `--target-http-proxy=moodle-http-proxy` - Link to HTTP proxy
- `--ports=80` - Listen on port 80 (HTTP)
- **What it does:** Binds everything together
  - IP address → Forwarding rule → HTTP proxy → URL map → Backend → VMs
- **This is the final connection:** Makes load balancer actually work

**Load Balancer Components Chain:**
```
User Request (http://34.107.234.56)
    ↓
Forwarding Rule (port 80)
    ↓
HTTP Proxy (handles HTTP protocol)
    ↓
URL Map (routing rules)
    ↓
Backend Service (pool of VMs)
    ↓
Health Check (verify VMs are working)
    ↓
Managed Instance Group (actual VMs)
    ↓
Response back to user
```

**⏰ Timing:**
- Creation: 5 minutes
- Health checks stabilize: 2-5 minutes
- Total until traffic flows: 7-10 minutes

### GUI Alternative

**Step 1: Create Load Balancer**
1. Navigation Menu → **"Network Services"** → **"Load balancing"**
2. Click **"Create Load Balancer"**
3. **Type:** HTTP(S) Load Balancing
4. Click **"Start configuration"**
5. Internet facing → **"From Internet to my VMs"**
6. Click **"Continue"**
7. Name: `moodle-lb`

**Step 2: Backend Configuration**
8. Click **"Backend configuration"**
9. Backend services → **"Create a backend service"**
   - Name: `moodle-backend`
   - Protocol: HTTP
   - Named port: http (80)
   - **Backends:**
     - Click "Add backend"
     - Instance group: `moodle-mig`
     - Port: 80
   - **Health check:**
     - Create new health check
     - Name: `moodle-hc`
     - Protocol: HTTP
     - Port: 80
     - Request path: `/login/index.php` ⚠️ **CRITICAL**
     - Click "Save"
   - Click "Create"

**Step 3: Frontend Configuration**
10. Click **"Frontend configuration"**
11. Click **"Add frontend IP and port"**
    - Name: `moodle-http-frontend`
    - Protocol: HTTP
    - IP version: IPv4
    - IP address: **Create IP address**
      - Name: `moodle-lb-ip`
      - Click "Reserve"
    - Port: 80
    - Click "Done"

**Step 4: Review and Create**
12. Review settings
13. Click **"Create"**
14. Wait 5-10 minutes for load balancer to be ready
15. **Copy the IP address** shown

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **Global Load Balancer** | Anycast IP, fast worldwide | Regional LB (limited to one region) |
| **HTTP (not HTTPS)** | Simpler for dev/test | HTTPS (requires SSL cert management) |
| **Health check: /login/index.php** | Always returns 200, even if not configured | / (returns 303 redirect, fails health check) |
| **Single backend** | Sufficient for single MIG | Multiple backends (complex, not needed) |
| **Reserved IP** | Static IP for DNS | Ephemeral IP (changes on recreate) |
| **Port 80** | Standard HTTP | Custom port (requires user to specify) |

**⚠️ Critical Detail:** Health check path MUST be `/login/index.php` not `/`
- `/` → 303 redirect → Health check FAILS → Load balancer shows UNHEALTHY
- `/login/index.php` → 200 OK → Health check PASSES → Load balancer HEALTHY

**🕐 Timeline:**
- Load balancer creation: 5 minutes
- Health check stabilization: 2-5 minutes
- Total time until traffic flows: 7-10 minutes

---

## 8. Complete Moodle Installation

### Essential Commands

```bash
# Get database password
DB_PASS=$(gcloud secrets versions access latest --secret=moodle-db-password)
```

**Command Breakdown:**
- `gcloud secrets versions access` - Retrieves secret value
- `latest` - Most recent version
- `--secret=moodle-db-password` - Secret name
- `$(...)` - Captures output into variable
- **What it does:** Gets database password from Secret Manager
- **Why this way:** Secure, audited access to credentials
- **Alternative:** Hardcoded password (insecure, bad practice)

```bash
# Get VM name (first one from MIG)
VM_NAME=$(gcloud compute instances list \
  --filter="tags.items=moodle-app" \
  --format="value(name)" \
  --limit=1)
```

**Command Breakdown:**
- `gcloud compute instances list` - Lists all VMs
- `--filter="tags.items=moodle-app"` - Only VMs with tag "moodle-app"
  - **Result:** Finds VMs from our managed instance group
- `--format="value(name)"` - Extract just the name
- `--limit=1` - Only get first one
- `$(...)` - Store name in variable
- **Result:** `VM_NAME="moodle-abcd"` (random suffix)
- **Why needed:** We don't know exact VM name (auto-generated)

```bash
# SSH and run Moodle CLI installer
gcloud compute ssh $VM_NAME \
  --zone="europe-west1-b" \
  --command="sudo -u www-data php /var/www/html/admin/cli/install_database.php --lang=en --adminuser=admin --adminpass='Moodle!2025' --adminemail=admin@example.com --agree-license"
```

**Command Breakdown:**
- `gcloud compute ssh` - SSH into VM
- `$VM_NAME` - The VM we found above
- `--zone="europe-west1-b"` - VM location
- `--command="..."` - Run this command remotely (don't open interactive shell)

**Remote Command Breakdown:**
- `sudo -u www-data` - Run as web server user
  - **Why:** Files need correct ownership
- `php /var/www/html/admin/cli/install_database.php` - Moodle's CLI installer
  - **Alternative:** Web-based installer (times out, needs manual clicking)
- **Installer Arguments:**
  - `--lang=en` - English language
  - `--adminuser=admin` - Admin username
  - `--adminpass='Moodle!2025'` - Admin password (change after first login!)
  - `--adminemail=admin@example.com` - Admin email
  - `--agree-license` - Accept Moodle GPL license

**What the installer does:**
1. Creates config.php with database settings
2. Creates all Moodle database tables (~400 tables)
3. Inserts default data
4. Creates admin user
5. Sets up site configuration

**⏰ Time:** 1-2 minutes

```bash
# Get load balancer IP
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')

echo "✅ Moodle is ready!"
echo "URL: http://$LB_IP"
echo "User: admin"
echo "Pass: Moodle!2025"
```

**Command Breakdown:**
- Gets load balancer IP (same as before)
- Displays access information
- **URL format:** `http://34.107.234.56`
- **Login:** admin / Moodle!2025

**🎉 At this point, Moodle is fully operational!**

### GUI Alternative

**Step 1: Get Database Password**
1. Navigation Menu → **"Secret Manager"**
2. Click on `moodle-db-password`
3. Click on version **"1"**
4. Click **"View secret value"**
5. **Copy the password**

**Step 2: SSH to VM**
1. Navigation Menu → **"Compute Engine"** → **"VM instances"**
2. Find VM starting with `moodle-` (e.g., `moodle-abcd`)
3. Click **"SSH"** button
4. Wait for terminal to open

**Step 3: Run Moodle Installer**
In the SSH terminal, run:
```bash
sudo -u www-data php /var/www/html/admin/cli/install_database.php \
  --lang=en \
  --adminuser=admin \
  --adminpass='Moodle!2025' \
  --adminemail=admin@example.com \
  --agree-license
```

**Step 4: Get Load Balancer IP**
1. Navigation Menu → **"Network Services"** → **"Load balancing"**
2. Click on `moodle-lb`
3. **Copy the IP address** shown at top
4. Open browser: `http://YOUR_IP_ADDRESS`

### 🤔 Decision Rationales

| Decision | Why This Choice | Alternative Considered |
|----------|----------------|----------------------|
| **CLI installer** | Fast, no timeout, scriptable | Web installer (60s timeout, manual) |
| **Pre-set admin password** | Known credentials, easy to change | Random password (need to reset) |
| **sudo -u www-data** | Runs as web server user (correct permissions) | Running as root (permission issues) |
| **--agree-license** | Non-interactive installation | Manual click (requires human) |

---

## 📊 Technology Choice Summary

### Overall Architecture Decisions

| Component | Choice | Why | Alternative | Why Not Alternative |
|-----------|--------|-----|-------------|-------------------|
| **Cloud Platform** | Google Cloud | Better for Kubernetes future, good pricing | AWS | More expensive for compute |
| **Region** | europe-west1 | GDPR compliance, EU data residency | us-central1 | Compliance issues |
| **Compute** | Managed Instance Group | Auto-scaling, self-healing | Single VM | No HA, manual scaling |
| **Database** | Cloud SQL | Managed, automatic backups | Self-hosted | More work, less reliable |
| **Storage** | NFS on VM | Cheap ($15/month) | Filestore | Expensive ($200/month) |
| **Load Balancer** | Global HTTP(S) LB | Anycast, global reach | Network LB | Regional only |
| **Networking** | Private VPC | Secure, isolated | Default VPC | Shared, less secure |
| **Authentication** | Service Accounts | No key management | JSON keys | Security risk |
| **Secrets** | Secret Manager | Audit trail, versioning | Environment variables | Hard to rotate |

### Cost Optimization Decisions

| Decision | Monthly Cost | Why Chosen | Expensive Alternative | Cost Difference |
|----------|--------------|------------|---------------------|----------------|
| **e2-small (not e2-medium)** | $15 | Sufficient for Moodle | e2-medium: $30 | Save $15/month |
| **Custom NFS (not Filestore)** | $15 | Good enough for dev/test | Filestore: $200+ | Save $185/month |
| **Zonal DB (not regional)** | $30 | Acceptable for dev/test | Regional: $60 | Save $30/month |
| **Standard disk (not SSD)** | $8 | NFS sufficient with standard | SSD: $25 | Save $17/month |
| **Auto-scale 1-3 (not fixed 3)** | $15 avg | Scales with demand | Always 3: $45 | Save $30/month |
| **HTTP (not HTTPS)** | $0 | Development setup | HTTPS + cert: $5+ | Save $5/month |

**Total Monthly Cost:** ~$95/month
**If using expensive alternatives:** ~$370/month
**Savings:** $275/month (74% cheaper)

### Performance Decisions

| Decision | Impact | Why Chosen |
|----------|--------|------------|
| **max_allowed_packet=1GB** | Prevents DB errors | Moodle sends large queries |
| **max_input_vars=5000** | Installation succeeds | Install form has 4000+ fields |
| **Private IP for SQL** | Faster queries | No internet hop |
| **NFS in same zone** | Low latency | Same datacenter as VMs |
| **60% CPU target** | Responsive scaling | Adds capacity before slowdown |
| **Health check /login/index.php** | Always healthy | / returns 303 redirect |

### Security Decisions

| Decision | Security Benefit | Why Chosen |
|----------|-----------------|------------|
| **No public IPs on VMs** | Can't be attacked directly | Internet access via NAT only |
| **Private Cloud SQL** | No DB internet exposure | Only VPC can connect |
| **Service Accounts** | No leaked keys | Workload identity |
| **Secret Manager** | Encrypted password storage | Better than env vars |
| **Firewall rules** | Explicit allow only | Default deny all |
| **Health check IP whitelist** | Only Google can health check | Reduces attack surface |

---

## 🎯 Command Execution Order

**Complete deployment in order:**

```bash
# 1. Project (5 minutes)
gcloud projects create + enable APIs

# 2. Network (3 minutes)
VPC + Subnet + Firewall rules + Private access

# 3. IAM (2 minutes)
Service account + Roles

# 4. Database (15 minutes) ⏰
Cloud SQL instance + Database + User + Secret

# 5. NFS (5 minutes)
Create VM + Format disk + Export share

# 6. Application (10 minutes)
Instance template + MIG + Auto-scaling

# 7. Load Balancer (10 minutes)
Health check + Backend + Frontend + IP

# 8. Installation (2 minutes)
CLI installer + Admin account

Total: ~52 minutes
```

---

## ✅ Verification Commands

```bash
# Check VM is running
gcloud compute instances list

# Check database is ready
gcloud sql instances list

# Check load balancer health
gcloud compute backend-services get-health moodle-backend --global

# Check if Moodle responds
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')
curl -I http://$LB_IP/login/index.php
```

**Expected:**
- VMs: RUNNING
- Database: RUNNABLE
- Health: HEALTHY
- HTTP: 200 OK

---

## 🚨 Common Issues and Fixes

| Issue | Symptom | Fix Command | GUI Fix |
|-------|---------|-------------|---------|
| **Unhealthy backend** | Load balancer doesn't work | Add health check firewall | VPC → Firewall → Allow 130.211.0.0/22 |
| **DB Gone Away error** | Moodle install fails | Set max_allowed_packet flag | SQL → Edit → Flags → max_allowed_packet=1073741824 |
| **Install form fails** | "max_input_vars exceeded" | Increase PHP setting | Already in startup script |
| **Can't connect to SQL** | Cloud SQL Proxy fails | Check private IP flag | SQL → Connections → Private IP only |
| **NFS mount fails** | VMs can't access files | Check NFS firewall rule | VPC → Firewall → Allow 2049 from subnet |

---

## 📝 Final Checklist

Before considering deployment complete:

- [ ] Load balancer shows **HEALTHY**
- [ ] Can access Moodle via HTTP
- [ ] Can login with admin/Moodle!2025
- [ ] Database connection works
- [ ] Files persist across VMs
- [ ] Auto-scaling is configured
- [ ] Health checks are passing
- [ ] All firewall rules exist
- [ ] Secret Manager has password
- [ ] Backups are scheduled

---

---

## 📚 Command Reference Summary

### Complete Command List (Copy-Paste Ready)

```bash
# ========== 1. PROJECT SETUP ==========
gcloud projects create "moodle-eu-demo-12345" --name="Moodle EU Demo"
gcloud config set project "moodle-eu-demo-12345"
gcloud beta billing projects link "moodle-eu-demo-12345" --billing-account="XXXXXX-YYYYYY-ZZZZZZ"
gcloud services enable compute.googleapis.com sqladmin.googleapis.com servicenetworking.googleapis.com secretmanager.googleapis.com logging.googleapis.com monitoring.googleapis.com
gcloud config set compute/region "europe-west1"
gcloud config set compute/zone "europe-west1-b"

# ========== 2. NETWORK ==========
gcloud compute networks create "moodle-vpc" --subnet-mode=custom
gcloud compute networks subnets create "moodle-subnet" --network="moodle-vpc" --region="europe-west1" --range="10.10.0.0/20"
gcloud compute firewall-rules create allow-ssh-admin --network="moodle-vpc" --allow=tcp:22 --source-ranges="0.0.0.0/0"
gcloud compute firewall-rules create allow-internal --network="moodle-vpc" --allow=tcp,udp,icmp --source-ranges="10.10.0.0/20"
gcloud compute firewall-rules create allow-nfs-internal --network="moodle-vpc" --allow=tcp:2049 --source-ranges="10.10.0.0/20"
gcloud compute firewall-rules create allow-health-checks --network="moodle-vpc" --allow=tcp:80 --source-ranges="130.211.0.0/22,35.191.0.0/16"
gcloud compute addresses create google-managed-services-moodle-vpc --global --purpose=VPC_PEERING --prefix-length=16 --network="moodle-vpc"
gcloud services vpc-peerings connect --service=servicenetworking.googleapis.com --network="moodle-vpc" --ranges=google-managed-services-moodle-vpc

# ========== 3. IAM ==========
gcloud iam service-accounts create "moodle-vm-sa" --display-name="Moodle VM SA"
sleep 30
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/secretmanager.secretAccessor"
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/cloudsql.client"
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/logging.logWriter"
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"

# ========== 4. DATABASE ==========
SQL_PASSWORD="$(openssl rand -base64 18)"
gcloud sql instances create "moodle-sql-12345" --database-version=MYSQL_8_0 --tier=db-g1-small --region="europe-west1" --network="projects/moodle-eu-demo-12345/global/networks/moodle-vpc" --no-assign-ip --database-flags=max_allowed_packet=1073741824,wait_timeout=10000 --availability-type=zonal --backup-start-time=03:00
gcloud sql databases create "moodle" --instance="moodle-sql-12345"
gcloud sql users create "moodleuser" --instance="moodle-sql-12345" --password="$SQL_PASSWORD"
printf "%s" "$SQL_PASSWORD" | gcloud secrets create moodle-db-password --data-file=-

# ========== 5. NFS (see startup script above) ==========
gcloud compute instances create "moodle-nfs" --zone="europe-west1-b" --machine-type=e2-micro --network="moodle-vpc" --subnet="moodle-subnet" --boot-disk-size=20GB --create-disk=name=moodle-nfs-data,size=50GB,type=pd-standard --metadata-from-file=startup-script=nfs-startup.sh
sleep 30
NFS_IP=$(gcloud compute instances describe "moodle-nfs" --zone="europe-west1-b" --format="value(networkInterfaces[0].networkIP)")

# ========== 6. APPLICATION (see startup script above) ==========
gcloud compute instance-templates create "moodle-template" --machine-type=e2-small --network="moodle-vpc" --subnet="moodle-subnet" --service-account="moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --scopes=https://www.googleapis.com/auth/cloud-platform --metadata-from-file=startup-script=moodle-startup.sh --tags=moodle-app --boot-disk-size=20GB
gcloud compute instance-groups managed create "moodle-mig" --base-instance-name=moodle --template="moodle-template" --size=1 --zone="europe-west1-b"
gcloud compute instance-groups managed set-autoscaling "moodle-mig" --zone="europe-west1-b" --min-num-replicas=1 --max-num-replicas=3 --target-cpu-utilization=0.6 --cool-down-period=60

# ========== 7. LOAD BALANCER ==========
gcloud compute health-checks create http moodle-hc --port=80 --request-path=/login/index.php --global
gcloud compute backend-services create moodle-backend --protocol=HTTP --health-checks=moodle-hc --global
gcloud compute backend-services add-backend moodle-backend --instance-group="moodle-mig" --instance-group-zone="europe-west1-b" --global
gcloud compute url-maps create moodle-map --default-service=moodle-backend
gcloud compute addresses create moodle-lb-ip --global
gcloud compute target-http-proxies create moodle-http-proxy --url-map=moodle-map
gcloud compute forwarding-rules create moodle-http-fr --global --address=moodle-lb-ip --target-http-proxy=moodle-http-proxy --ports=80

# ========== 8. INSTALLATION ==========
DB_PASS=$(gcloud secrets versions access latest --secret=moodle-db-password)
VM_NAME=$(gcloud compute instances list --filter="tags.items=moodle-app" --format="value(name)" --limit=1)
gcloud compute ssh $VM_NAME --zone="europe-west1-b" --command="sudo -u www-data php /var/www/html/admin/cli/install_database.php --lang=en --adminuser=admin --adminpass='Moodle!2025' --adminemail=admin@example.com --agree-license"
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')
echo "URL: http://$LB_IP | User: admin | Pass: Moodle!2025"
```

---

## 🎯 Key Concepts Explained

### What is gcloud?
- **Google Cloud SDK** command-line tool
- Controls all GCP resources
- Alternative to web console
- Faster for repetitive tasks
- Scriptable and automatable

### What is a Service Account?
- Robot account (non-human identity)
- VMs use it to access GCP services
- No password, uses cryptographic keys
- Can be granted IAM roles
- **Email format:** `name@project.iam.gserviceaccount.com`

### What is Cloud SQL Proxy?
- **Secure tunnel** to Cloud SQL
- Runs on VM, listens on localhost:3306
- Handles authentication via service account
- Encrypts all traffic
- **Benefit:** App connects to localhost (simple), proxy handles security

### What is a Managed Instance Group?
- Collection of identical VMs
- Created from instance template
- **Auto-scaling:** Adds/removes VMs based on metrics
- **Self-healing:** Replaces failed VMs automatically
- **Rolling updates:** Updates VMs with zero downtime

### What is a Load Balancer?
- Distributes traffic across multiple VMs
- **Health checks:** Only sends to healthy VMs
- **Global:** Serves users from nearest location
- **Auto-scaling aware:** Works with MIG automatically

### What is max_allowed_packet?
- MySQL setting for maximum query size
- **Default:** 4 MB
- **Our setting:** 1 GB (1073741824 bytes)
- **Why:** Moodle sends huge queries when restoring courses
- **Without it:** "MySQL server has gone away" error

### What is max_input_vars?
- PHP setting for maximum form fields
- **Default:** 1000 fields
- **Our setting:** 5000 fields
- **Why:** Moodle installation form has >4,000 checkboxes
- **Without it:** "max_input_vars exceeded" error

---

## 📊 Command Categories

### Resource Creation Commands
```bash
gcloud compute networks create         # Create VPC
gcloud compute instances create        # Create VM
gcloud sql instances create            # Create database
gcloud compute instance-templates create  # Create template
gcloud compute instance-groups managed create  # Create MIG
```

### Configuration Commands
```bash
gcloud config set                      # Set defaults
gcloud compute firewall-rules create  # Configure firewall
gcloud services enable                 # Enable APIs
gcloud projects add-iam-policy-binding # Grant permissions
```

### Query Commands
```bash
gcloud compute instances list          # List VMs
gcloud compute instances describe      # Get VM details
gcloud sql instances list              # List databases
gcloud secrets versions access         # Get secret value
```

### Connection Commands
```bash
gcloud compute ssh                     # SSH into VM
gcloud sql connect                     # Connect to database
```

---

## 🔑 Critical Parameters Explained

### Networking
| Parameter | Value | Why |
|-----------|-------|-----|
| `--subnet-mode` | custom | Full control over IP ranges |
| `--range` | 10.10.0.0/20 | 4,096 IPs, non-conflicting |
| `--no-assign-ip` | (Cloud SQL) | Security: no public internet access |
| `--source-ranges` | 130.211.0.0/22,35.191.0.0/16 | Google's health check IPs |

### Database
| Parameter | Value | Why |
|-----------|-------|-----|
| `--tier` | db-g1-small | Balance cost/performance, prevents OOM |
| `--database-version` | MYSQL_8_0 | Latest stable, Moodle compatible |
| `max_allowed_packet` | 1073741824 (1GB) | Prevents "gone away" errors |
| `--backup-start-time` | 03:00 | Low usage time |

### Compute
| Parameter | Value | Why |
|-----------|-------|-----|
| `--machine-type` | e2-small | 2vCPU/2GB sufficient for web serving |
| `--min-num-replicas` | 1 | Always have one instance |
| `--max-num-replicas` | 3 | Cost control |
| `--target-cpu-utilization` | 0.6 (60%) | Responsive scaling |

### Load Balancer
| Parameter | Value | Why |
|-----------|-------|-----|
| `--request-path` | /login/index.php | Returns 200 OK (not 303 redirect) |
| `--port` | 80 | Standard HTTP port |
| `--global` | (flag) | Anycast IP, worldwide access |

---

## 🛠️ Troubleshooting Commands

### Check Status
```bash
# List all VMs
gcloud compute instances list

# Check VM details
gcloud compute instances describe VM_NAME --zone=europe-west1-b

# Check load balancer health
gcloud compute backend-services get-health moodle-backend --global

# Check database status
gcloud sql instances list
gcloud sql instances describe moodle-sql-12345

# Check firewall rules
gcloud compute firewall-rules list --filter="network:moodle-vpc"
```

### SSH and Debug
```bash
# SSH into VM
gcloud compute ssh VM_NAME --zone=europe-west1-b

# Run command on VM
gcloud compute ssh VM_NAME --zone=europe-west1-b --command="systemctl status apache2"

# View logs on VM
gcloud compute ssh VM_NAME --zone=europe-west1-b --command="sudo tail /var/log/apache2/error.log"
```

### Get Information
```bash
# Get NFS IP
gcloud compute instances describe moodle-nfs --zone=europe-west1-b --format="value(networkInterfaces[0].networkIP)"

# Get load balancer IP
gcloud compute addresses describe moodle-lb-ip --global --format='value(address)'

# Get database password
gcloud secrets versions access latest --secret=moodle-db-password

# List all service accounts
gcloud iam service-accounts list
```

---

## 🎓 Understanding the Architecture

### Data Flow: User Request to Response

```
1. User types: http://34.107.234.56
   ↓
2. DNS resolves to: Load Balancer IP
   ↓
3. Forwarding Rule (port 80) receives request
   ↓
4. HTTP Proxy processes request
   ↓
5. URL Map routes to: moodle-backend
   ↓
6. Backend Service checks: Which VMs are HEALTHY?
   ↓
7. Health Check says: VM moodle-abcd is HEALTHY
   ↓
8. Load Balancer sends request to: moodle-abcd
   ↓
9. Apache on VM receives HTTP request
   ↓
10. PHP processes Moodle code
    ↓
11. Cloud SQL Proxy connects to: Database (localhost:3306)
    ↓
12. Cloud SQL returns data
    ↓
13. PHP reads files from: NFS (/var/moodledata)
    ↓
14. PHP generates HTML response
    ↓
15. Apache sends response back through load balancer
    ↓
16. User sees: Moodle page
```

### Component Relationships

```
┌─────────────────────────────────────────┐
│          Internet (Users)               │
└────────────────┬────────────────────────┘
                 ↓
┌────────────────────────────────────────┐
│  Load Balancer (34.107.234.56:80)     │ ← Firewall allows health checks
│  - Forwarding Rule                     │
│  - HTTP Proxy                          │
│  - URL Map                             │
│  - Backend Service                     │
│  - Health Check (/login/index.php)    │
└────────────────┬────────────────────────┘
                 ↓
┌────────────────────────────────────────┐
│  Managed Instance Group (1-3 VMs)     │ ← Auto-scales based on CPU
│  - moodle-abcd (10.10.0.3)            │
│  - moodle-efgh (10.10.0.4)            │ ← Added when busy
│  - moodle-ijkl (10.10.0.5)            │ ← Added when very busy
└─────────┬──────────────────────────────┘
          │
          ├─────────────────┬────────────────┐
          ↓                 ↓                ↓
┌───────────────┐  ┌─────────────────┐  ┌──────────────┐
│ Cloud SQL     │  │  NFS Server     │  │ Secret Mgr   │
│ (Private IP)  │  │  (10.10.0.5)    │  │ (Passwords)  │
│ MySQL 8.0     │  │  50GB storage   │  │              │
└───────────────┘  └─────────────────┘  └──────────────┘
```

---

## 💡 Command Patterns

### Creating Resources
**Pattern:** `gcloud <service> <resource-type> create <name> [options]`

**Examples:**
```bash
gcloud compute networks create "name" --options
gcloud sql instances create "name" --options
gcloud iam service-accounts create "name" --options
```

### Modifying Resources
**Pattern:** `gcloud <service> <resource-type> <action> <name> [options]`

**Examples:**
```bash
gcloud compute firewall-rules update "name" --options
gcloud sql instances patch "name" --options
gcloud compute instance-groups managed set-autoscaling "name" --options
```

### Querying Resources
**Pattern:** `gcloud <service> <resource-type> list|describe [filters]`

**Examples:**
```bash
gcloud compute instances list
gcloud sql instances describe "name"
gcloud compute instances list --filter="tags.items=moodle-app"
```

---

## 📋 Flags Reference

### Common Flags

| Flag | Purpose | Example Value |
|------|---------|--------------|
| `--project` | Which GCP project | moodle-eu-demo-12345 |
| `--region` | Geographic region | europe-west1 |
| `--zone` | Specific zone | europe-west1-b |
| `--network` | Which VPC | moodle-vpc |
| `--subnet` | Which subnet | moodle-subnet |
| `--global` | Global resource | (no value) |
| `--quiet` | No confirmation prompts | (no value) |
| `--format` | Output format | value(name), json, yaml |

### Resource-Specific Flags

**Compute Engine:**
- `--machine-type` - VM size (e2-micro, e2-small, etc.)
- `--boot-disk-size` - OS disk size in GB
- `--tags` - Network tags for firewall rules
- `--metadata-from-file` - Attach startup script

**Cloud SQL:**
- `--database-version` - MySQL version
- `--tier` - Machine type for database
- `--no-assign-ip` - Don't create public IP
- `--database-flags` - MySQL configuration
- `--backup-start-time` - When to run backups

**Networking:**
- `--allow` - What traffic to allow
- `--source-ranges` - Where traffic comes from
- `--subnet-mode` - Auto or custom subnets
- `--range` - IP address range (CIDR)

---

## 🚨 Critical Settings (Don't Skip!)

| Setting | Where | Value | Consequence if Wrong |
|---------|-------|-------|---------------------|
| **Health check path** | Load balancer | /login/index.php | UNHEALTHY status forever |
| **Health check firewall** | Firewall rules | 130.211.0.0/22, 35.191.0.0/16 | UNHEALTHY status forever |
| **max_allowed_packet** | Cloud SQL flags | 1073741824 (1GB) | "MySQL server has gone away" |
| **max_input_vars** | PHP config | 5000 | Installation fails |
| **--private-ip flag** | Cloud SQL Proxy | Required | Connection fails |
| **30s wait** | After service account | Must wait | Permissions not found |
| **Private IP only** | Cloud SQL | --no-assign-ip | Security risk if public |

---

## 📖 Glossary of Command Terms

| Term | Meaning | Example |
|------|---------|---------|
| **Instance** | Virtual Machine | Compute instance, SQL instance |
| **Template** | VM blueprint | Instance template |
| **MIG** | Managed Instance Group | Auto-scaling group |
| **Backend** | Pool of VMs | Backend service |
| **Frontend** | External entry point | Forwarding rule |
| **Health Check** | Automated test | HTTP check on port 80 |
| **Firewall Rule** | Network filter | Allow/deny traffic |
| **Service Account** | Robot user | Identity for VMs |
| **IAM Role** | Permission set | cloudsql.client |
| **VPC** | Virtual network | Isolated private network |
| **Subnet** | IP range | 10.10.0.0/20 |
| **Zone** | Data center | europe-west1-b |
| **Region** | Geographic area | europe-west1 (Belgium) |

---

**Document:** COMMANDS_AND_GUI_GUIDE.md (Enhanced with explanations)
**Purpose:** Essential commands with detailed explanations, GUI alternatives, and decision rationales
**Level:** Practical implementation guide with learning
**Date:** December 19, 2025

**You now understand every command, what it does, why it's needed, and how to do it via GUI! 🎯📚**

