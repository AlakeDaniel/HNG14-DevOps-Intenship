# HNG DevOps Internship — Stage 1: Build & Deploy a Personal API

![Stage](https://img.shields.io/badge/Stage-1-blue) ![Track](https://img.shields.io/badge/Track-DevOps-orange) ![Status](https://img.shields.io/badge/Status-Live-brightgreen)

## Overview

A lightweight personal API built with Python and Flask, deployed on an AWS EC2 instance behind an Nginx reverse proxy with SSL. The API exposes three endpoints providing basic status and personal information. The app is served via Gunicorn for fast, production-grade performance and managed by systemd for persistent uptime.

**Live URL:** `https://mygoal.chickenkiller.com`

---

## Tech Stack

| Component       | Technology                 |
|-----------------|----------------------------|
| Language        | Python 3                   |
| Framework       | Flask                      |
| WSGI Server     | Gunicorn                   |
| Web Server      | Nginx (Reverse Proxy)      |
| SSL             | Let's Encrypt (Certbot)    |
| Process Manager | systemd                    |
| Cloud           | AWS EC2 (Ubuntu 22.04 LTS) |

---

## Project Structure

```
stage-1/
├── app.py          # Main Flask application
├── .gitignore      # Git ignore rules
└── README.md       # Project documentation
```

---

## API Endpoints

All endpoints return:
- `Content-Type: application/json`
- HTTP status code `200`
- Response time under `500ms`

---

### 1. `GET /`
Returns a simple status message confirming the API is running.

**Request:**
```bash
curl https://mygoal.chickenkiller.com/
```

**Response:**
```json
{
  "message": "API is running"
}
```

---

### 2. `GET /health`
Returns the health status of the API.

**Request:**
```bash
curl https://mygoal.chickenkiller.com/health
```

**Response:**
```json
{
  "message": "healthy"
}
```

---

### 3. `GET /me`
Returns personal information about the developer.

**Request:**
```bash
curl https://mygoal.chickenkiller.com/me
```

**Response:**
```json
{
  "name": "Alake Daniel Adebayo",
  "email": "danieladebayo78ng@gmail.com",
  "github": "https://github.com/AlakeDaniel"
}
```

---

## How to Run Locally

### Prerequisites
- Python 3 installed
- pip installed
- Git installed

### Steps

**1. Clone the repository**
```bash
git clone https://github.com/AlakeDaniel/HNG14-DevOps-Intenship.git
cd HNG14-DevOps-Intenship/stage-1
```

**2. Create and activate a virtual environment**
```bash
python3 -m venv venv

# On Linux/Mac
source venv/bin/activate

# On Windows
venv\Scripts\activate
```

**3. Install dependencies**
```bash
pip install flask gunicorn
```

**4. Run with Flask (development)**
```bash
python app.py
```

**Or run with Gunicorn (production)**
```bash
gunicorn --workers 4 --bind 127.0.0.1:5000 app:app
```

**5. Test the endpoints**
```bash
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/me
```

The API will be available at `http://localhost:5000`

---

## Deployment

The API is deployed on an AWS EC2 instance (Ubuntu 22.04 LTS) with the following setup:

### Architecture
```
Internet → Nginx (port 443/80) → Gunicorn (port 5000) → Flask App
```

### Nginx Reverse Proxy
Nginx listens on ports 80 and 443, forwarding all requests to Gunicorn running locally on port 5000. HTTP requests are automatically redirected to HTTPS with a 301 redirect.

```nginx
server {
    listen 80;
    server_name mygoal.chickenkiller.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name mygoal.chickenkiller.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Persistent Service with systemd
The app is managed by **systemd**, ensuring it starts automatically on boot and restarts if it ever crashes.

```ini
[Unit]
Description=HNG Stage 1 API
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/api
ExecStart=/home/ubuntu/api/venv/bin/gunicorn --workers 4 --bind 127.0.0.1:5000 app:app
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Useful service commands:
```bash
# Check service status
sudo systemctl status api

# Restart service
sudo systemctl restart api

# Stop service
sudo systemctl stop api
```

### SSL
SSL certificate is issued by **Let's Encrypt** via Certbot and auto-renews every 90 days.

```bash
# Test auto-renewal
sudo certbot renew --dry-run
```

---

## Live Endpoints

| Endpoint | URL |
|----------|-----|
| Root     | https://mygoal.chickenkiller.com/ |
| Health   | https://mygoal.chickenkiller.com/health |
| Me       | https://mygoal.chickenkiller.com/me |

---

## Author

**Alake Daniel Adebayo**
- Email: danieladebayo78ng@gmail.com
- GitHub: [AlakeDaniel](https://github.com/AlakeDaniel)

---

*Part of the HNG Internship DevOps Track — Stage 1*
