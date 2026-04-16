# HNG DevOps Internship — Stage 1: Build & Deploy a Personal API

![Stage](https://img.shields.io/badge/Stage-1-blue) ![Track](https://img.shields.io/badge/Track-DevOps-orange) ![Status](https://img.shields.io/badge/Status-Live-brightgreen)

## Overview

A lightweight personal API built with Python and Flask, deployed on an AWS EC2 instance behind an Nginx reverse proxy with SSL. The API exposes three endpoints providing basic status and personal information.

**Live URL:** `https://mygoal.chickenkiller.com`

---

## Tech Stack

| Component      | Technology                  |
|----------------|-----------------------------|
| Language       | Python 3                    |
| Framework      | Flask                       |
| Web Server     | Nginx (Reverse Proxy)       |
| SSL            | Let's Encrypt (Certbot)     |
| Process Manager| systemd                     |
| Cloud          | AWS EC2 (Ubuntu 22.04 LTS)  |

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

All endpoints return:
- `Content-Type: application/json`
- HTTP status code `200`
- Response time under `500ms`

---

## How to Run Locally

### Prerequisites
- Python 3 installed
- pip installed

### Steps

**1. Clone the repository**
```bash
git clone https://github.com/AlakeDaniel/HNG14-DevOps-Intenship.git
cd HNG14-DevOps-Intenship/stage-1
```

**2. Create and activate a virtual environment**
```bash
python3 -m venv venv
source venv/bin/activate        # On Linux/Mac
venv\Scripts\activate           # On Windows
```

**3. Install dependencies**
```bash
pip install flask
```

**4. Run the application**
```bash
python app.py
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
Internet → Nginx (port 443/80) → Flask App (port 5000)
```

### Nginx Reverse Proxy
Nginx listens on ports 80 and 443, forwarding all requests to the Flask app running locally on port 5000. HTTP requests are automatically redirected to HTTPS with a 301 redirect.

### Persistent Service
The Flask app is managed by **systemd**, ensuring it starts automatically on boot and restarts if it ever crashes.

```bash
# Check service status
sudo systemctl status api

# Restart service
sudo systemctl restart api
```

### SSL
SSL certificate is issued by **Let's Encrypt** via Certbot and auto-renews every 90 days.

---

## Live Deployment

| Endpoint | URL |
|----------|-----|
| Root | https://mygoal.chickenkiller.com/ |
| Health | https://mygoal.chickenkiller.com/health |
| Me | https://mygoal.chickenkiller.com/me |

---

*Part of the HNG Internship DevOps Track — Stage 1*
