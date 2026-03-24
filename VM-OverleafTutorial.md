# Overleaf on TrueNAS - Setup Guide

Complete guide to set up self-hosted Overleaf accessible via `https://overleaf.yourdomain.com`

## Prerequisites

- TrueNAS server
- Ubuntu Server 24.04.3 LTS ISO
- Domain with Cloudflare DNS
- Cloudflare Tunnel app installed on TrueNAS
- **Minimum**: 4 CPU cores, 8GB RAM, 200GB disk

---

# Step 1: Create Ubuntu VM

## 1.1 Basic Configuration

TrueNAS Web UI → **Virtualization** → **Virtual Machines** → **Add**

```
Guest OS: Linux
Name: UbuntuServerLTS24
Boot Method: UEFI
☑ Enable Display
Bind: [TrueNAS-IP]
Password: [set VNC password]
☑ Start on Boot
```

## 1.2 CPU and Memory

```
Virtual CPUs: 4
Cores: 1
Threads: 1
CPU Mode: Host Passthrough  ← IMPORTANT
Memory: 8192 MB
```

**Note**: Must use "Host Passthrough" for MongoDB 8.0 compatibility.

## 1.3 Disk

```
Type: ZVOL
Size: 200 GiB
Mode: VIRTIO
```

## 1.4 Network

```
☑ Attach NIC
NIC Type: VIRTIO
Bridge: br0
```

## 1.5 Installation Media

```
Image: [Your Ubuntu 24.04.3 ISO path]
```

Click **Save**.

---

# Step 2: Install Ubuntu

## 2.1 Access Console

**Virtualization** → **Virtual Machines** → **UbuntuServerLTS24** → **Start** → **Display**

Or visit: `http://[TrueNAS-IP]:[VNC-PORT]/spice_auto.html` (Bind IP: TrueNAS IP)

## 2.2 Ubuntu Installation

Follow installer with these key settings:

```
Network: DHCP (auto)
Storage: Use entire disk
Profile:
  - Server name: overleaf-server
  - Username: yourname (remember this!)
  - Password: [your password]
SSH: ☑ Install OpenSSH server
Snaps: Skip all
```

Reboot when done.

## 2.3 Find VM IP Address

From TrueNAS shell:
```bash
arp -a | grep "00:a0:98"
```

Or check the VM console login screen.

---

# Step 3: Prepare VM

## 3.1 SSH into VM

```bash
ssh yourname@[VM-IP]
```

## 3.2 Verify AVX Support

```bash
grep -o 'avx[^ ]*' /proc/cpuinfo | sort -u
```

Should show: `avx` and `avx2`

## 3.3 Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose git -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

---

# Step 4: Install Overleaf

## 4.1 Clone Repository

```bash
git clone https://github.com/overleaf/toolkit.git overleaf-toolkit
cd overleaf-toolkit
bin/init
```

## 4.2 Configure Network

Edit `config/overleaf.rc`:
```bash
nano config/overleaf.rc
```

Change line 12:
```bash
OVERLEAF_LISTEN_IP=0.0.0.0
```

Save: `Ctrl+X`, `Y`, `Enter`

## 4.3 Configure Domain

Edit `config/variables.env`:
```bash
nano config/variables.env
```

Uncomment and set (lines 18-21):
```bash
OVERLEAF_BEHIND_PROXY=true
OVERLEAF_SECURE_COOKIE=true

OVERLEAF_SITE_URL=https://overleaf.yourdomain.com
```

Save: `Ctrl+X`, `Y`, `Enter`

## 4.4 Start Services

```bash
bin/up -d
```

Wait 30 seconds, then verify:
```bash
docker ps
```

Should show 3 containers running: `sharelatex`, `mongo`, `redis`

---

# Step 5: Configure Cloudflare Tunnel

## 5.1 Add Route

TrueNAS Web UI → **Apps** → **Cloudflare Tunnel** → **Edit**

Add Public Hostname:
```
Subdomain: overleaf
Domain: yourdomain.com
Service Type: HTTP
Service URL: http://[VM-IP]:80
```

Example: `http://[VM-IP]:80`

Click **Save**.

## 5.2 Verify

After 1-2 minutes, visit:
```
https://overleaf.yourdomain.com
```

You should see Overleaf!

---

# Step 6: Create Admin Account

Visit:
```
https://overleaf.yourdomain.com/launchpad
```

Register with your email and password. Done!

---

# Step 7: Install Full TeX Live (Recommended)

By default, Overleaf includes a minimal TeX installation. For journal papers (especially ACS, IEEE, etc.), install the complete TeX Live distribution.

## 7.1 Enter Container

```bash
docker exec -it sharelatex bash
```

## 7.2 Install Full Scheme

```bash
tlmgr install scheme-full
tlmgr path add
```

This installs ~4000 LaTeX packages. Takes 5-10 minutes.

You'll see progress like:
```
[1/4000, 00:01/05:30] install: package1
[2/4000, 00:02/05:28] install: package2
...
```

**Wait for completion. Do not interrupt.**

## 7.3 Save Changes

After installation completes:
```bash
exit
docker commit sharelatex sharelatex/sharelatex:6.1.1-with-texlive-full
```

This saves the full TeX Live installation permanently.

**Note**: Without this, you may get "package not found" errors when compiling journal papers.

---

# Quick Reference

## Access Information

```bash
SSH: ssh yourname@[VM-IP]
Web: https://overleaf.yourdomain.com
Console: http://[TrueNAS-IP]:[VNC-PORT]/spice_auto.html (Bind IP: TrueNAS IP)
```

## Useful Commands

```bash
# Check containers
cd ~/overleaf-toolkit
docker ps

# View logs
docker logs sharelatex
docker logs mongo

# Restart services
bin/stop
bin/up -d

# Update Overleaf
bin/stop
bin/docker-compose pull
bin/up -d
```

## Container Versions

- MongoDB: 8.0
- Redis: 7.4
- Overleaf: 6.1.1 (with full TeX Live ~4000 packages)

---

# Backup

## Backup Overleaf Data

```bash
cd ~/overleaf-toolkit

# Backup MongoDB
docker exec mongo mongodump --out /dump
docker cp mongo:/dump ./backup-$(date +%F)

# Backup all data
sudo tar -czf overleaf-backup-$(date +%F).tar.gz data/

# Copy to TrueNAS
scp overleaf-backup-*.tar.gz root@[TrueNAS-IP]:/mnt/[Pool]/Backups/
```

---

# Optional Configuration

## Enable Email Notifications

Email notifications enable password resets, project invitations, and collaboration notifications.

### Get Gmail App Password

1. Enable 2-Factor Authentication: https://myaccount.google.com/security
2. Generate App Password: https://myaccount.google.com/apppasswords
   - Select **Mail** → **Other (Custom)** → Name it "Overleaf"
   - Copy the 16-character password

### Configure Email

Edit `config/variables.env`:
```bash
nano config/variables.env
```

Uncomment and set these lines (around line 36-45):
```bash
OVERLEAF_EMAIL_FROM_ADDRESS=overleaf@yourdomain.com
OVERLEAF_EMAIL_SMTP_HOST=smtp.gmail.com
OVERLEAF_EMAIL_SMTP_PORT=587
OVERLEAF_EMAIL_SMTP_SECURE=false
OVERLEAF_EMAIL_SMTP_USER=your-email@gmail.com
OVERLEAF_EMAIL_SMTP_PASS=xxxx xxxx xxxx xxxx
```

Replace with your actual Gmail and App Password.

Save: `Ctrl+X`, `Y`, `Enter`

Restart services:
```bash
bin/stop && bin/up -d
```

### Test Email

Visit `https://overleaf.yourdomain.com`, click "Forgot password?" and test with your email.

Check logs if needed:
```bash
docker exec sharelatex env | grep EMAIL
docker logs sharelatex | grep -i mail
```

## Increase Compile Memory

Edit `config/variables.env`:
```bash
OVERLEAF_COMPILE_MEMORY_LIMIT_MB=4096
```

Restart: `bin/stop && bin/up -d`

---

# Architecture

```
Internet
   ↓ HTTPS
Cloudflare (SSL/DDoS)
   ↓ Encrypted Tunnel
TrueNAS → Cloudflare Tunnel App
   ↓ HTTP
Ubuntu VM ([VM-IP])
   ↓
Docker Containers
   ├─ Overleaf (:80)
   ├─ MongoDB (:27017)
   └─ Redis (:6379)
```

---

# Maintenance

## Update Ubuntu (Monthly)
```bash
ssh yourname@[VM-IP]
sudo apt update && sudo apt upgrade -y
sudo reboot
```

## Update Overleaf (As needed)
```bash
cd ~/overleaf-toolkit
bin/stop
bin/docker-compose pull
bin/up -d
```

---

**Setup Complete!** Visit `https://overleaf.yourdomain.com` and start creating LaTeX documents.

**Created**: February 7, 2026
