# HNG DevOps Internship — Stage 0: Linux Server Setup & Nginx Configuration


## Overview

This document details the complete process of provisioning a Linux server on AWS, configuring Nginx to serve two endpoints, and securing the server with a valid SSL certificate — all on a bare Linux server with no Docker, no Compose, and no automation tools.

**Live Domain:** `https://mygoal.chickenkiller.com`

---

## Table of Contents

1. [Task Requirements](#task-requirements)
2. [Infrastructure Setup](#1-infrastructure-setup)
3. [Domain & DNS Configuration](#2-domain--dns-configuration)
4. [Server Access & Initial Setup](#3-server-access--initial-setup)
5. [User & Security Configuration](#4-user--security-configuration)
6. [Firewall Configuration (UFW)](#5-firewall-configuration-ufw)
7. [Nginx Installation & Configuration](#6-nginx-installation--configuration)
8. [SSL Certificate with Lets Encrypt](#7-ssl-certificate-with-lets-encrypt)
9. [Verification Checklist](#8-verification-checklist)
10. [Lessons Learned](#lessons-learned)

---

## Task Requirements

- Provision a Linux server on any cloud provider
- Create a non-root user `hngdevops` with sudo privileges
- Configure passwordless sudo for `hngdevops` (restricted to `/usr/sbin/sshd` and `/usr/sbin/ufw`)
- Disable root SSH login
- Disable password-based SSH authentication (key-based only)
- Configure UFW to allow only ports 22, 80, and 443
- Install and configure Nginx to serve:
  - `GET /` — a static HTML page with your HNG username visible as text
  - `GET /api` — a JSON response with specific payload
- Obtain a valid SSL certificate using Let's Encrypt (Certbot)
- HTTP requests must redirect to HTTPS with a `301` redirect

**Required `/api` JSON response:**
```json
{
  "message": "HNGI14 Stage 0",
  "track": "DevOps",
  "username": "your-hng-username"
}
```

---

## 1. Infrastructure Setup

### Cloud Provider: AWS EC2

I used **AWS EC2** on the free tier to provision my Linux server.

**Steps:**

1. Log into the [AWS Console](https://console.aws.amazon.com)
2. Navigate to **EC2 → Launch Instance**
3. Configure the instance:
   - **Name:** `hng-stage0`
   - **AMI:** Ubuntu Server 22.04 LTS (Free Tier eligible)
   - **Instance Type:** `t2.micro` (Free Tier)
   - **Key Pair:** Create a new key pair, download the `.pem` file and store it safely
4. **Network Settings:** Create a new security group with the following inbound rules:

| Type  | Protocol | Port | Source    |
|-------|----------|------|-----------|
| SSH   | TCP      | 22   | 0.0.0.0/0 |
| HTTP  | TCP      | 80   | 0.0.0.0/0 |
| HTTPS | TCP      | 443  | 0.0.0.0/0 |

5. Launch the instance and note the **Public IPv4 address**

### Allocate an Elastic IP (Important)

AWS public IPs change every time an instance is stopped and restarted. To prevent the domain from breaking, an Elastic IP was allocated:

1. Go to **EC2 → Elastic IPs → Allocate Elastic IP Address**
2. Click **Associate Elastic IP Address**
3. Select your running instance and associate it
4. Use this static IP for all DNS records going forward

---

## 2. Domain & DNS Configuration

### Getting a Free Domain

A free subdomain was registered at [FreeDNS (afraid.org)](https://freedns.afraid.org):

1. Create a free account at `freedns.afraid.org`
2. Go to **Subdomains → Add Subdomain**
3. Choose a free shared domain (e.g. `chickenkiller.com`)
4. Set your subdomain prefix (e.g. `mygoal`)
5. Set the **Destination** to your AWS Elastic IP address
6. Save the record

### DNS Record Configuration

| Type | Subdomain | Destination       |
|------|-----------|-------------------|
| A    | mygoal    | 52.71.216.229     |

> **Note:** With a free FreeDNS subdomain, adding a `www` prefix record is not straightforward. The solution is to only use the root subdomain (without `www`) for Certbot and Nginx configuration.

### Verify DNS Propagation

After setting up the DNS record, verify it has propagated:

```bash
nslookup mygoal.chickenkiller.com 8.8.8.8
```

The output should return your server's public IP address. You can also check globally using [dnschecker.org](https://dnschecker.org).

---

## 3. Server Access & Initial Setup

### SSH Into the Server

```bash
# Set correct permissions on the key file
chmod 400 your-key.pem

# SSH into the instance
ssh -i your-key.pem ubuntu@52.71.216.229
```

### Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. User & Security Configuration

### Create the `hngdevops` User

```bash
# Create the user
sudo adduser hngdevops

# Add to sudo group
sudo usermod -aG sudo hngdevops
```

### Set Up SSH Key Authentication for `hngdevops`

The grading bot authenticates using a specific public key provided in the `#track-devops` Slack channel. This key must be added to the `hngdevops` user's authorized keys.

```bash
# Create the .ssh directory
sudo mkdir -p /home/hngdevops/.ssh
sudo chmod 700 /home/hngdevops/.ssh

# Add the grading bot's public key
sudo nano /home/hngdevops/.ssh/authorized_keys
# Paste the public key provided in the #track-devops Slack channel

# Set correct permissions
sudo chmod 600 /home/hngdevops/.ssh/authorized_keys
sudo chown -R hngdevops:hngdevops /home/hngdevops/.ssh
```

### Configure Passwordless Sudo (Restricted)

The task requires `hngdevops` to run only `sshd` and `ufw` without a password:

```bash
sudo bash -c 'echo "hngdevops ALL=(root) NOPASSWD:/usr/sbin/sshd,/usr/sbin/ufw" > /etc/sudoers.d/hngdevops'
sudo chmod 440 /etc/sudoers.d/hngdevops
```

### Harden SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Update the following lines (remove `#` if commented out):

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart the SSH service:

```bash
sudo systemctl restart ssh
```

Verify the settings took effect:

```bash
sudo grep -E "PermitRootLogin|PasswordAuthentication|PubkeyAuthentication" /etc/ssh/sshd_config
```

Expected output:
```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

> **Important:** Before closing your current SSH session after making these changes, open a second terminal and confirm you can still connect. This prevents accidental lockout.

---

## 5. Firewall Configuration (UFW)

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow required ports only
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443

# Enable the firewall
sudo ufw enable

# Verify status
sudo ufw status
```

Expected output:
```
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
80                         ALLOW       Anywhere
443                        ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
80 (v6)                    ALLOW       Anywhere (v6)
443 (v6)                   ALLOW       Anywhere (v6)
```

---

## 6. Nginx Installation & Configuration

### Install Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Configure Nginx

Before running Certbot, the Nginx configuration must be set to **HTTP only** (no SSL references). Certbot will automatically add the SSL configuration after issuing the certificate.

```bash
sudo bash -c 'cat > /etc/nginx/sites-available/default' << 'EOF'
server {
    listen 80;
    server_name mygoal.chickenkiller.com;

    location / {
        root /var/www/html;
        index index.html;
    }

    location /api {
        default_type application/json;
        return 200 '{"message":"HNGI14 Stage 0","track":"DevOps","username":"danielcloud"}';
    }
}
EOF
```

> **Important:** Use `default_type application/json` to ensure the correct Content-Type header is returned for the `/api` endpoint.

Test and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Create the Static HTML Page

```bash
sudo nano /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html>
<head><title>HNG Stage 0</title></head>
<body>
  <h1>Welcome</h1>
  <p>HNG Username: danielcloud</p>
</body>
</html>
```

> The username must be **visible text** on the page. Hidden text or HTML comments will fail the grading bot check.

---

## 7. SSL Certificate with Lets Encrypt

### Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Obtain SSL Certificate

```bash
sudo certbot --nginx -d mygoal.chickenkiller.com
```

Follow the interactive prompts:
- Enter your email address for renewal notices
- Agree to the Terms of Service
- Certbot will automatically update your Nginx configuration to handle HTTPS and add a 301 redirect from HTTP to HTTPS

### Verify Auto-Renewal

```bash
sudo certbot renew --dry-run
```

### Final Nginx Configuration (After Certbot)

After Certbot runs successfully, your Nginx config will look similar to this:

```nginx
server {
    listen 80;
    server_name mygoal.chickenkiller.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name mygoal.chickenkiller.com;

    ssl_certificate /etc/letsencrypt/live/mygoal.chickenkiller.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mygoal.chickenkiller.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        root /var/www/html;
        index index.html;
    }

    location /api {
        default_type application/json;
        return 200 '{"message":"HNGI14 Stage 0","track":"DevOps","username":"danielcloud"}';
    }
}
```

---

## 8. Verification Checklist

Run these commands to verify everything is correctly configured before submitting:

```bash
# 1. Verify Nginx is active
sudo systemctl is-active nginx

# 2. Verify UFW is active with correct ports
sudo ufw status

# 3. Verify SSH hardening
sudo grep -E "PermitRootLogin|PasswordAuthentication|PubkeyAuthentication" /etc/ssh/sshd_config

# 4. Verify hngdevops sudoers
sudo cat /etc/sudoers.d/hngdevops

# 5. Verify bot SSH key is in place
sudo cat /home/hngdevops/.ssh/authorized_keys

# 6. Test HTTP redirects to HTTPS (should return 301)
curl -I http://mygoal.chickenkiller.com

# 7. Test HTTPS root endpoint
curl -s https://mygoal.chickenkiller.com

# 8. Test API endpoint
curl -s https://mygoal.chickenkiller.com/api
```

**Expected `/api` response:**
```json
{"message":"HNGI14 Stage 0","track":"DevOps","username":"danielcloud"}
```

---

## Grading Result

| Test                       | Result   | Points   |
|----------------------------|----------|----------|
| http_redirect              | ✅ Pass  | 1/1      |
| https_root                 | ✅ Pass  | 1.5/1.5  |
| https_api                  | ✅ Pass  | 1.5/1.5  |
| ssl_cert                   | ✅ Pass  | 1/1      |
| ssh_connect                | ✅ Pass  | 1/1      |
| ssh_user_exists            | ✅ Pass  | 0.5/0.5  |
| ssh_user_sudo              | ✅ Pass  | 0.5/0.5  |
| ssh_root_login_disabled    | ✅ Pass  | 1/1      |
| ssh_password_auth_disabled | ✅ Pass  | 1/1      |
| ufw_ports                  | ✅ Pass  | 0.5/0.5  |
| nginx_active               | ✅ Pass  | 0.5/0.5  |
| **Total**                  | **✅ 11/11** | **10/10** |

---

## Lessons Learned

**1. Configure Nginx as HTTP-only before running Certbot.**
If your Nginx config references SSL certificate paths that don't exist yet, Nginx will fail to start and Certbot will be unable to run. Always start with a plain HTTP config and let Certbot handle adding SSL.

**2. AWS Security Groups are a separate firewall from UFW.**
Both must have the correct ports open. UFW controls traffic at the OS level inside the instance; Security Groups control traffic reaching the instance from the internet. Having UFW open but Security Group closed will still block all traffic.

**3. DNS must fully propagate before running Certbot.**
Let's Encrypt verifies domain ownership by making an HTTP request to your server. If DNS hasn't propagated yet, Certbot will fail with a `No such authorization` error. Always verify with `nslookup` before running Certbot.

**4. Elastic IPs are essential on AWS.**
AWS public IPs change on every stop/start cycle. Always allocate and associate an Elastic IP to keep your domain pointing to the right server.

**5. SSH key file permissions matter.**
The `.ssh` directory must be `700` and `authorized_keys` must be `600`. Incorrect permissions cause silent authentication failures with no obvious error message.

**6. Username casing in JSON is strictly checked.**
The grading bot does an exact string match — `DanielCloud` and `danielcloud` are treated as completely different values. Always double-check your registered HNG username spelling and casing.

**7. Use `default_type` not `add_header` for JSON Content-Type in Nginx.**
When returning a JSON response with `return 200`, use `default_type application/json` instead of `add_header Content-Type application/json` for reliable Content-Type header delivery.

---

## Tech Stack

| Component   | Technology                    |
|-------------|-------------------------------|
| Cloud       | AWS EC2 (t2.micro)            |
| OS          | Ubuntu Server 22.04 LTS       |
| Web Server  | Nginx                         |
| SSL         | Let's Encrypt (Certbot)       |
| Firewall    | UFW                           |
| DNS         | FreeDNS (afraid.org)          |

---

*Part of the HNG Internship DevOps Track — Stage 0 | Score: 10/10*
