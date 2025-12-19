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

# Set as active project
gcloud config set project "moodle-eu-demo-12345"

# Link billing account
gcloud beta billing projects link "moodle-eu-demo-12345" --billing-account="XXXXXX-YYYYYY-ZZZZZZ"

# Enable required APIs
gcloud services enable compute.googleapis.com sqladmin.googleapis.com servicenetworking.googleapis.com secretmanager.googleapis.com logging.googleapis.com monitoring.googleapis.com

# Set default region and zone
gcloud config set compute/region "europe-west1"
gcloud config set compute/zone "europe-west1-b"
```

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

# Create Subnet
gcloud compute networks subnets create "moodle-subnet" --network="moodle-vpc" --region="europe-west1" --range="10.10.0.0/20"

# Firewall: Allow SSH
gcloud compute firewall-rules create allow-ssh-admin --network="moodle-vpc" --allow=tcp:22 --source-ranges="0.0.0.0/0"

# Firewall: Internal communication
gcloud compute firewall-rules create allow-internal --network="moodle-vpc" --allow=tcp,udp,icmp --source-ranges="10.10.0.0/20"

# Firewall: NFS traffic
gcloud compute firewall-rules create allow-nfs-internal --network="moodle-vpc" --allow=tcp:2049 --source-ranges="10.10.0.0/20"

# Firewall: Health checks (CRITICAL)
gcloud compute firewall-rules create allow-health-checks --network="moodle-vpc" --allow=tcp:80 --source-ranges="130.211.0.0/22,35.191.0.0/16"

# Reserve IP range for private services
gcloud compute addresses create google-managed-services-moodle-vpc --global --purpose=VPC_PEERING --prefix-length=16 --network="moodle-vpc"

# Create private connection to Google services
gcloud services vpc-peerings connect --service=servicenetworking.googleapis.com --network="moodle-vpc" --ranges=google-managed-services-moodle-vpc
```

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

# Wait for propagation (IMPORTANT)
sleep 30

# Grant permissions
gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/logging.logWriter"

gcloud projects add-iam-policy-binding "moodle-eu-demo-12345" --member="serviceAccount:moodle-vm-sa@moodle-eu-demo-12345.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"
```

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

# Create database
gcloud sql databases create "moodle" --instance="moodle-sql-12345"

# Create user
gcloud sql users create "moodleuser" --instance="moodle-sql-12345" --password="$SQL_PASSWORD"

# Store password securely
printf "%s" "$SQL_PASSWORD" | gcloud secrets create moodle-db-password --data-file=-
```

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

# Create NFS VM
gcloud compute instances create "moodle-nfs" \
  --zone="europe-west1-b" \
  --machine-type=e2-micro \
  --network="moodle-vpc" \
  --subnet="moodle-subnet" \
  --boot-disk-size=20GB \
  --create-disk=name=moodle-nfs-data,size=50GB,type=pd-standard \
  --metadata-from-file=startup-script=nfs-startup.sh

# Get NFS IP (wait 30 seconds first)
sleep 30
NFS_IP=$(gcloud compute instances describe "moodle-nfs" --zone="europe-west1-b" --format="value(networkInterfaces[0].networkIP)")
echo "NFS IP: $NFS_IP"
```

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

# Fix: Increase max_input_vars
sed -i 's/;max_input_vars = 1000/max_input_vars = 5000/' /etc/php/*/apache2/php.ini
sed -i 's/;max_input_vars = 1000/max_input_vars = 5000/' /etc/php/*/cli/php.ini

# Mount NFS
mkdir -p /var/moodledata
echo "10.10.0.5:/srv/moodledata /var/moodledata nfs defaults,_netdev 0 0" >> /etc/fstab
mount -a
chown -R www-data:www-data /var/moodledata

# Cloud SQL Proxy
curl -o /usr/local/bin/cloud-sql-proxy -L https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.4/cloud-sql-proxy.linux.amd64
chmod +x /usr/local/bin/cloud-sql-proxy

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

systemctl daemon-reload
systemctl enable --now cloud-sql-proxy

# Download Moodle
cd /var/www/html
if [ ! -f config.php ]; then
  rm -f index.html
  curl -L -o moodle.tgz https://download.moodle.org/download.php/direct/stable404/moodle-latest-404.tgz
  tar -xzf moodle.tgz --strip-components=1
  chown -R www-data:www-data /var/www/html
fi

a2enmod rewrite
systemctl restart apache2
EOF

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

# Create managed instance group
gcloud compute instance-groups managed create "moodle-mig" \
  --base-instance-name=moodle \
  --template="moodle-template" \
  --size=1 \
  --zone="europe-west1-b"

# Configure auto-scaling
gcloud compute instance-groups managed set-autoscaling "moodle-mig" \
  --zone="europe-west1-b" \
  --min-num-replicas=1 \
  --max-num-replicas=3 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=60
```

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

# Create backend service
gcloud compute backend-services create moodle-backend \
  --protocol=HTTP \
  --health-checks=moodle-hc \
  --global

# Add instance group to backend
gcloud compute backend-services add-backend moodle-backend \
  --instance-group="moodle-mig" \
  --instance-group-zone="europe-west1-b" \
  --global

# Create URL map
gcloud compute url-maps create moodle-map \
  --default-service=moodle-backend

# Reserve external IP
gcloud compute addresses create moodle-lb-ip --global

# Get the IP
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')
echo "Load Balancer IP: $LB_IP"

# Create HTTP proxy
gcloud compute target-http-proxies create moodle-http-proxy \
  --url-map=moodle-map

# Create forwarding rule
gcloud compute forwarding-rules create moodle-http-fr \
  --global \
  --address=moodle-lb-ip \
  --target-http-proxy=moodle-http-proxy \
  --ports=80
```

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

# Get VM name (first one from MIG)
VM_NAME=$(gcloud compute instances list --filter="tags.items=moodle-app" --format="value(name)" --limit=1)

# SSH and run Moodle CLI installer
gcloud compute ssh $VM_NAME --zone="europe-west1-b" --command="sudo -u www-data php /var/www/html/admin/cli/install_database.php --lang=en --adminuser=admin --adminpass='Moodle!2025' --adminemail=admin@example.com --agree-license"

# Get load balancer IP
LB_IP=$(gcloud compute addresses describe moodle-lb-ip --global --format='value(address)')

echo "✅ Moodle is ready!"
echo "URL: http://$LB_IP"
echo "User: admin"
echo "Pass: Moodle!2025"
```

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

**Document:** COMMANDS_AND_GUI_GUIDE.md
**Purpose:** Essential commands, GUI alternatives, decision rationales
**Level:** Practical implementation guide
**Date:** December 19, 2025

**You now have both CLI and GUI paths with full understanding of WHY each choice was made! 🎯**

