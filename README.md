
---

# 💙💚 Blue–Green Deployment Project – Summary

## 1. **Project Goals**

* Run **two identical Node.js app instances** (`Blue` & `Green`) behind an **Nginx reverse proxy**.
* Normal traffic → `Blue`.
* Failover: If `Blue` fails, **Nginx automatically retries** and serves from `Green` **within the same client request**.
* Entire setup is **Docker Compose-based** and **parameterized with `.env`**.
* Includes **chaos endpoints** (`/chaos/start`, `/chaos/stop`) to simulate failures.

---

## 2. **High-level Architecture**

```text
Client → Nginx (public endpoint) → upstreams: Blue (primary) + Green (backup)
                      │
             Blue (localhost:8081)  Green (localhost:8082)
```

* Nginx uses `ACTIVE_POOL` to determine the primary app (default: `Blue`).
* Docker images pre-built: `yimikaade/wonderful:devops-stage-two`.
* Healthchecks ensure only healthy apps receive traffic.

---

## 3. **Project Files Overview**

```text
blue-green-project/
├─ docker-compose.yml       # 3 services: app_blue, app_green, nginx
├─ .env.example             # Template for environment variables
├─ .env                     # Local configuration
├─ nginx/
│  ├─ default.conf.tmpl     # Nginx template
│  ├─ start.sh              # Render template & start nginx
│  └─ reload.sh             # Render template & reload nginx
├─ README.md                # Usage instructions
```

**Key Highlights:**

* **docker-compose.yml** – parameterized using `${BLUE_IMAGE}`, `${GREEN_IMAGE}`, `${ACTIVE_POOL}`.
* **.env.example** – defines `BLUE_IMAGE`, `GREEN_IMAGE`, `ACTIVE_POOL`, `RELEASE_ID_BLUE/GREEN`, `PORT`, `NGINX_PUBLIC_PORT`.
* **nginx/default.conf.tmpl** – placeholders: `PRIMARY_HOST`, `BACKUP_HOST`, `PRIMARY_PORT`, `BACKUP_PORT`.
* **start.sh / reload.sh** – template rendering + start/reload Nginx.

---

## 4. **Step-by-Step Setup**

### A. **Install Prerequisites**

```bash
sudo apt update
sudo apt install -y docker.io docker-compose curl jq
sudo usermod -aG docker $USER
# Log out/in or restart shell
```

* Ensure **Docker Desktop WSL integration** is ON.

---

### B. **Environment File (`.env.example`)**

```text
BLUE_IMAGE=yimikaade/wonderful:devops-stage-two
GREEN_IMAGE=yimikaade/wonderful:devops-stage-two
ACTIVE_POOL=blue
RELEASE_ID_BLUE=blue-v1
RELEASE_ID_GREEN=green-v1
PORT=3000
NGINX_PUBLIC_PORT=8080
```

* Copy & edit `.env` locally:

```bash
cp .env.example .env
```

---

### C. **docker-compose.yml**

* Services: `app_blue`, `app_green`, `nginx`.
* Ports: `8081` (blue), `8082` (green), `8080` (nginx).
* Healthchecks ensure only healthy app receives traffic.
* Example (shortened):

```yaml
services:
  app_blue:
    image: ${BLUE_IMAGE}
    container_name: app_blue
    environment:
      - RELEASE_ID=${RELEASE_ID_BLUE}
      - APP_POOL=blue
      - PORT=${PORT}
    ports:
      - "8081:${PORT}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${PORT}/healthz"]
      interval: 5s
      retries: 3

  nginx:
    image: nginx:stable
    container_name: bg_nginx
    depends_on:
      - app_blue
      - app_green
    ports:
      - "${NGINX_PUBLIC_PORT}:80"
    volumes:
      - ./nginx/default.conf.tmpl:/etc/nginx/templates/default.conf.tmpl:ro
      - ./nginx/start.sh:/etc/nginx/start.sh:ro
      - ./nginx/reload.sh:/etc/nginx/reload.sh:ro
    environment:
      - ACTIVE_POOL=${ACTIVE_POOL}
      - APP_PORT=${PORT}
    command: ["/bin/sh", "-c", "/etc/nginx/start.sh"]
```

---

### D. **Nginx Template (`default.conf.tmpl`)**

```nginx
upstream app_upstream {
    server PRIMARY_HOST:PRIMARY_PORT max_fails=1 fail_timeout=3s;
    server BACKUP_HOST:BACKUP_PORT backup;
}

server {
    listen 80;

    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    proxy_next_upstream_tries 2;

    location / {
        proxy_pass http://app_upstream;
        proxy_set_header X-App-Pool $upstream_http_x_app_pool;
        proxy_set_header X-Release-Id $upstream_http_x_release_id;
    }

    location /healthz {
        proxy_pass http://app_upstream/healthz;
    }
}
```

---

### E. **Nginx Start & Reload Scripts**

**start.sh**:

```bash
#!/bin/sh
set -eu
# Determine primary/backup based on ACTIVE_POOL
# Render template
nginx -t && nginx -g 'daemon off;'
```

**reload.sh**:

```bash
#!/bin/sh
set -eu
# Determine primary/backup based on ACTIVE_POOL
# Render template
nginx -t && nginx -s reload
```

* Make executable:

```bash
chmod +x nginx/start.sh nginx/reload.sh
```

---

### F. **Run & Verify Locally**

```bash
docker-compose up -d
docker-compose ps
```

* **Verify Blue active**:

```bash
curl -i http://localhost:8080/version
# Expect X-App-Pool: blue
# Expect X-Release-Id: blue-v1
```

* **Chaos Testing**:

```bash
curl -X POST http://localhost:8081/chaos/start?mode=error
curl -i http://localhost:8080/version
# Nginx should failover → X-App-Pool: green
curl -X POST http://localhost:8081/chaos/stop
```

* Confirm recovery of Blue.

* Optional reload inside container:

```bash
docker exec bg_nginx /etc/nginx/reload.sh
```

---

### G. **EC2 Deployment Steps**

1. Spin up EC2 with ports `22, 8080, 8081, 8082`.
2. SSH & install Docker + Compose:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

3. Clone repo & start stack:

```bash
git clone <repo_url>
cd blue-green-project
sudo docker compose up -d
docker ps
```

4. Test baseline → induce chaos → verify failover → stop chaos → confirm recovery.

---

## 5. **Troubleshooting Notes**

| Issue                    | Solution                                                           |
| ------------------------ | ------------------------------------------------------------------ |
| `docker-compose` missing | Use `docker compose` (V2) or install plugin                        |
| Port 8080 in use         | Identify with `ss/netstat`; change mapping or stop conflicting app |
| Vim permission error     | `sudo chown $USER:$USER <file>`                                    |
| Containers `unhealthy`   | Check healthcheck endpoint & correct env vars                      |
| SSH key error            | `chmod 400 project-key.pem`                                        |

---

## ✅ **Key Takeaways**

* Fully automated **Blue–Green deployment** with Docker Compose.
* Zero-downtime failover via **Nginx upstream `backup` + `proxy_next_upstream`**.
* Chaos testing to verify resilience.
* Environment-driven configuration for **flexibility and reproducibility**.

---

## 🏁 **Conclusion**

This Blue/Green deployment ensures robust failover, zero downtime, and header integrity under real conditions. The automated verification and CI/CD integration make it ideal for modern DevOps pipelines with rapid rollback and reliability guarantees.

---
Author: Anthony Usoro 
Slack Username: @anthonyusoro 
Project: Blue/Green Node.js Deployment with Nginx Failover Documentation 
Date: November 3, 2025

---
