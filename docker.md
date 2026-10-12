# 🐳 Docker Master Class: From Zero to Production Hero
## For MERN Stack, Node.js, and Python AI Integration

A complete, hands-on learning path. Work through it top-to-bottom. Each section has **concepts → commands → real project → challenges**.

---

## 📚 Table of Contents

1. [Prerequisites & Setup](#1-prerequisites--setup)
2. [Docker Fundamentals](#2-docker-fundamentals)
3. [Dockerfile Mastery (Node.js First)](#3-dockerfile-mastery-nodejs-first)
4. [Docker Compose (MERN Stack)](#4-docker-compose-mern-stack)
5. [Data Persistence & Volumes](#5-data-persistence--volumes)
6. [Networking Deep Dive](#6-networking-deep-dive)
7. [Environment Variables & Secrets](#7-environment-variables--secrets)
8. [Multi-Stage Builds & Optimization](#8-multi-stage-builds--optimization)
9. [Python + AI Integration](#9-python--ai-integration)
10. [Full MERN + AI Project](#10-full-mern--ai-project)
11. [Production Best Practices](#11-production-best-practices)
12. [Debugging & Troubleshooting](#12-debugging--troubleshooting)
13. [What's Next (Beyond Docker)](#13-whats-next-beyond-docker)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. Prerequisites & Setup

### What You Need Installed

| Tool | Why |
|------|-----|
| **Docker Desktop** (Win/Mac) or **Docker Engine** (Linux) | The runtime |
| **Git** | Version control |
| **Node.js 20+** | For local testing (optional) |
| **VS Code** | Editor with Docker extension |
| **Postman/Thunder Client** | API testing |
| **Terminal** (WSL2 on Windows recommended) | CLI work |

### Verify Installation

```bash
docker --version          # Docker version 24.x+
docker compose version    # v2.x
docker run hello-world    # Should print "Hello from Docker!"
```

### VS Code Extensions to Install

- **Docker** (Microsoft)
- **Dev Containers** (Microsoft)
- **Remote - WSL** (Windows users)
- **MongoDB for VS Code**

### 🧠 Mental Model

```
┌─────────────────────────────────────┐
│  Docker Host (your machine)         │
│  ┌──────────┐  ┌──────────┐         │
│  │Container │  │Container │         │
│  │ (Node)   │  │ (Mongo)  │         │
│  └──────────┘  └──────────┘         │
│  ┌─────────────────────────┐        │
│  │   Docker Engine         │        │
│  └─────────────────────────┘        │
└─────────────────────────────────────┘
```

**Image** = blueprint (read-only) → **Container** = running instance of image.

---

## 2. Docker Fundamentals

### 2.1 Core Commands You'll Use Daily

```bash
# Images
docker images                          # List images
docker pull node:20-alpine             # Download image
docker rmi <image_id>                  # Remove image
docker image prune -a                  # Remove unused images

# Containers
docker ps                              # Running containers
docker ps -a                           # All containers (incl. stopped)
docker run -d -p 3000:3000 --name myapp node:20
docker start/stop/restart <name>
docker rm <name>                       # Remove container
docker logs -f <name>                  # Follow logs
docker exec -it <name> sh              # Shell into container

# System
docker system df                       # Disk usage
docker system prune -a --volumes       # Nuke everything unused
```

### 2.2 Your First Container

```bash
# Run nginx and visit http://localhost:8080
docker run -d -p 8080:80 --name web nginx

# Interactive Ubuntu
docker run -it ubuntu:22.04 bash
# Inside: apt update && apt install curl -y
# Exit: exit
```

### 2.3 Key Flags Explained

| Flag | Meaning |
|------|---------|
| `-d` | Detached (run in background) |
| `-p 8080:80` | Map host port 8080 → container port 80 |
| `-v /host:/container` | Mount volume |
| `-e KEY=value` | Environment variable |
| `--name` | Give container a name |
| `-it` | Interactive + TTY |
| `--rm` | Auto-remove when stopped |
| `--network` | Attach to a network |

### 🎯 Exercise 1
1. Run `redis:alpine` on port 6379 in detached mode.
2. Exec into it and run `redis-cli ping` → should get `PONG`.
3. Check logs, then stop and remove it.

---

## 3. Dockerfile Mastery (Node.js First)

### 3.1 What is a Dockerfile?
A text file with instructions to build a Docker image.

### 3.2 Core Instructions

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host → image |
| `RUN` | Execute command during build |
| `CMD` | Default command at container start |
| `ENTRYPOINT` | Like CMD but harder to override |
| `ENV` | Environment variable |
| `EXPOSE` | Document a port (doesn't publish it) |
| `ARG` | Build-time variable |
| `USER` | Switch to non-root user |

### 3.3 ❌ Bad Dockerfile (Node.js)

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "index.js"]
```

**Problems:** huge image, no layer caching, runs as root, copies `node_modules`.

### 3.4 ✅ Good Dockerfile (Production-ready)

```dockerfile
# ---------- Stage 1: Dependencies ----------
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# ---------- Stage 2: Builder ----------
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build        # if you have TS/Next.js etc.

# ---------- Stage 3: Runtime ----------
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Non-root user
RUN addgroup -S app && adduser -S app -G app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package*.json ./

USER app
EXPOSE 5000
CMD ["node", "dist/index.js"]
```

### 3.5 `.dockerignore` (CRITICAL)

```
node_modules
npm-debug.log
.git
.env
.env.*
dist
build
coverage
.vscode
*.md
Dockerfile
docker-compose*.yml
```

### 3.6 Build & Run

```bash
docker build -t my-api:1.0 .
docker run -d -p 5000:5000 --name api my-api:1.0
docker logs -f api
```

### 🎯 Exercise 2
Create a minimal Express server with `/health` returning `{status:'ok'}`. Write a Dockerfile using the **good** pattern above. Build and test with `curl localhost:5000/health`.

---

## 4. Docker Compose (MERN Stack)

Compose lets you run multi-container apps with one file.

### 4.1 Full MERN Stack `docker-compose.yml`

```yaml
version: "3.9"

services:
  # ---------- MongoDB ----------
  mongo:
    image: mongo:7
    container_name: mern-mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret
    volumes:
      - mongo_data:/data/db
    networks:
      - mern-net
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ---------- Backend (Node/Express) ----------
  backend:
    build:
      context: ./backend
      target: runner
    container_name: mern-backend
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      NODE_ENV: production
      PORT: 5000
      MONGO_URI: mongodb://admin:secret@mongo:27017/mern?authSource=admin
    depends_on:
      mongo:
        condition: service_healthy
    networks:
      - mern-net

  # ---------- Frontend (React + Nginx) ----------
  frontend:
    build:
      context: ./frontend
    container_name: mern-frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - mern-net

volumes:
  mongo_data:

networks:
  mern-net:
    driver: bridge
```

### 4.2 Commands

```bash
docker compose up -d --build     # Build & start
docker compose ps                # Status
docker compose logs -f backend   # Follow logs
docker compose exec backend sh   # Shell in
docker compose down              # Stop & remove
docker compose down -v           # Also wipe volumes
```

### 4.3 Frontend Dockerfile (React + Nginx)

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Serve stage
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 4.4 `nginx.conf` (SPA + API proxy)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://backend:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 🎯 Exercise 3
Build a Todo app: React frontend + Express backend + MongoDB. All three must run with `docker compose up`. Verify by adding and listing todos.

---

## 5. Data Persistence & Volumes

### 5.1 Types of Mounts

| Type | Syntax | Use Case |
|------|--------|----------|
| **Named volume** | `mydata:/data` | Databases, persistent data |
| **Bind mount** | `./src:/app/src` | Dev hot-reload |
| **tmpfs** | `tmpfs:/tmp` | In-memory, secrets |

### 5.2 Named Volume (Mongo example)

```yaml
volumes:
  - mongo_data:/data/db     # Data survives `docker compose down`
```

### 5.3 Bind Mount for Development

`docker-compose.dev.yml`:
```yaml
services:
  backend:
    build: ./backend
    command: npm run dev
    volumes:
      - ./backend:/app        # Live code sync
      - /app/node_modules     # Don't overwrite container's node_modules
    environment:
      NODE_ENV: development
```

Run with: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up`

### 5.4 Inspect & Backup

```bash
docker volume ls
docker volume inspect mern_mongo_data

# Backup a volume
docker run --rm -v mern_mongo_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/mongo-backup.tar.gz -C /data .
```

### 🎯 Exercise 4
1. Start Mongo, add a document.
2. Run `docker compose down` then `up` — verify data persists.
3. Now run `down -v` — data should be gone.

---

## 6. Networking Deep Dive

### 6.1 Default Networks

```bash
docker network ls
# bridge (default), host, none
```

### 6.2 Custom Bridge Network (Best Practice)

```bash
docker network create mynet
docker run -d --name db --network mynet mongo
docker run -d --name api --network mynet -e MONGO_URI=mongodb://db:27017 myapi
```

**Key insight:** Containers on the same custom network can reach each other by **container name** (DNS).

### 6.3 How the MERN stack talks

```
Browser → localhost:3000 → [frontend container] → http://backend:5000 → [backend] → mongodb://mongo:27017
```

### 6.4 Ports vs. Expose

- `EXPOSE 5000` = documentation only
- `-p 5000:5000` = actually publish to host
- In Compose, containers talk on internal ports **without** publishing

### 🎯 Exercise 5
Run two containers on a custom network. From container A, `ping` container B by name. Prove DNS works.

---

## 7. Environment Variables & Secrets

### 7.1 `.env` file

```env
PORT=5000
MONGO_URI=mongodb://admin:secret@mongo:27017/mern?authSource=admin
JWT_SECRET=super-secret-change-me
OPENAI_API_KEY=sk-xxxx
```

### 7.2 Reference in Compose

```yaml
services:
  backend:
    env_file:
      - .env
    environment:
      NODE_ENV: production
```

### 7.3 Secrets (Swarm/Compose v2)

```yaml
secrets:
  jwt_secret:
    file: ./secrets/jwt.txt

services:
  backend:
    secrets:
      - jwt_secret
```

Inside container: `/run/secrets/jwt_secret`

### ⚠️ Never Commit Secrets

- Add `.env` to `.gitignore`
- Use `docker secret` or a vault (AWS Secrets Manager, Doppler, Infisical) in production

### 🎯 Exercise 6
Move `MONGO_URI` and `PORT` out of `docker-compose.yml` into `.env`. Confirm the app still connects.

---

## 8. Multi-Stage Builds & Optimization

### 8.1 Why Multi-Stage?
Smaller images = faster deploys, less attack surface.

### 8.2 Size Comparison (Node.js API)

| Approach | Size |
|----------|------|
| `node:20` + copy everything | ~1.1 GB |
| `node:20-alpine` + good Dockerfile | ~180 MB |
| Multi-stage + `npm ci --omit=dev` | ~90 MB |

### 8.3 Layer Caching Rules

**Docker caches each instruction. Order matters:**

```dockerfile
# ✅ Good: package.json first
COPY package*.json ./
RUN npm ci
COPY . .              # Code changes don't bust npm cache

# ❌ Bad: copies everything, busts cache on every code change
COPY . .
RUN npm ci
```

### 8.4 Image Scanning

```bash
docker scout quickview my-api:1.0
docker scout cves my-api:1.0
```

### 8.5 Tools to Try
- **dive** — visualize image layers: `dive my-api:1.0`
- **hadolint** — Dockerfile linter
- **trivy** — vulnerability scanner

### 🎯 Exercise 7
Take your Exercise 2 image. Run `docker images` to see size. Refactor to multi-stage. Get it under 100 MB.

---

## 9. Python + AI Integration

Now the fun part — adding AI microservices.

### 9.1 Python FastAPI Dockerfile

```dockerfile
FROM python:3.11-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# System deps (for numpy, pillow, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential curl && rm -rf /var/lib/apt/lists/*

# Python deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# App code
COPY . .

# Non-root
RUN useradd -m appuser && chown -R appuser /app
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 9.2 `requirements.txt`

```txt
fastapi==0.115.0
uvicorn[standard]==0.32.0
pydantic==2.9.0
openai==1.50.0
python-dotenv==1.0.1
transformers==4.45.0
torch==2.4.0
sentence-transformers==3.1.0
pymongo==4.9.0
```

### 9.3 `main.py` — AI Microservice

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from openai import OpenAI
import os

app = FastAPI(title="AI Service")
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

class Prompt(BaseModel):
    text: str

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/chat")
def chat(p: Prompt):
    try:
        r = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": p.text}]
        )
        return {"reply": r.choices[0].message.content}
    except Exception as e:
        raise HTTPException(500, str(e))
```

### 9.4 Adding AI Service to Compose

```yaml
services:
  ai-service:
    build: ./ai-service
    container_name: ai-service
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    networks:
      - mern-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      retries: 3
```

### 9.5 Call It From Express

```javascript
// backend/routes/ai.js
import express from "express";
const router = express.Router();

router.post("/ask", async (req, res) => {
  const r = await fetch("http://ai-service:8000/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: req.body.text }),
  });
  res.json(await r.json());
});

export default router;
```

**Note:** `http://ai-service:8000` works because they share the `mern-net` network. 🎉

### 9.6 GPU Support (Optional)

```yaml
ai-service:
  build: ./ai-service
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

Requires **NVIDIA Container Toolkit** on the host.

### 9.7 Model Caching (Don't re-download on every build)

```yaml
services:
  ai-service:
    volumes:
      - hf_cache:/root/.cache/huggingface

volumes:
  hf_cache:
```

### 🎯 Exercise 8
Add a `/sentiment` endpoint using `transformers` pipeline. Wire it into the MERN backend and display results in the React UI.

---

## 10. Full MERN + AI Project

### 10.1 Project Structure

```
mern-ai-app/
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── .gitignore
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   │   ├── index.js
│   │   ├── routes/
│   │   └── models/
│   └── .dockerignore
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── src/
└── ai-service/
    ├── Dockerfile
    ├── requirements.txt
    └── main.py
```

### 10.2 Complete Compose

```yaml
version: "3.9"

services:
  mongo:
    image: mongo:7
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASS}
    networks: [app-net]
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      retries: 5

  ai-service:
    build: ./ai-service
    restart: unless-stopped
    environment:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    volumes:
      - hf_cache:/root/.cache/huggingface
    networks: [app-net]

  backend:
    build:
      context: ./backend
      target: runner
    restart: unless-stopped
    environment:
      MONGO_URI: mongodb://${MONGO_USER}:${MONGO_PASS}@mongo:27017/app?authSource=admin
      AI_URL: http://ai-service:8000
      PORT: 5000
    depends_on:
      mongo: { condition: service_healthy }
      ai-service: { condition: service_started }
    networks: [app-net]

  frontend:
    build: ./frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on: [backend]
    networks: [app-net]

volumes:
  mongo_data:
  hf_cache:

networks:
  app-net:
    driver: bridge
```

### 10.3 Boot Everything

```bash
cp .env.example .env       # fill in secrets
docker compose up -d --build
docker compose ps
open http://localhost:3000
```

### 🎯 Final Project Milestone
Ship a working app where:
- ✅ User submits a prompt in React
- ✅ Backend stores the prompt in MongoDB
- ✅ Backend calls the Python AI service
- ✅ AI responds and it's saved + displayed
- ✅ All services restart cleanly with `docker compose down && up`

---

## 11. Production Best Practices

### 11.1 Dockerfile Checklist
- [ ] Pin base image versions (`node:20.11-alpine`, not `node:latest`)
- [ ] Multi-stage builds
- [ ] Run as non-root `USER`
- [ ] `.dockerignore` present
- [ ] `HEALTHCHECK` defined
- [ ] No secrets baked in
- [ ] Small base image (alpine/slim/distroless)

### 11.2 Compose Checklist
- [ ] `restart: unless-stopped`
- [ ] `healthcheck` on stateful services
- [ ] `depends_on: condition: service_healthy`
- [ ] Named volumes for data
- [ ] Custom network
- [ ] Resource limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

### 11.3 Logging

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

### 11.4 Security

```bash
# Read-only root filesystem
docker run --read-only --tmpfs /tmp myapp

# Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Scan for vulnerabilities
trivy image my-api:1.0
```

### 11.5 Image Tagging Strategy
```bash
docker build -t myapi:1.0.0 -t myapi:latest -t myapi:$(git rev-parse --short HEAD) .
```

---

## 12. Debugging & Troubleshooting

### 12.1 Common Errors

| Error | Fix |
|-------|-----|
| `port is already allocated` | `docker ps` → kill process or change host port |
| `Cannot connect to MongoDB` | Use container name, not `localhost` |
| `ENOENT: package.json` | Check `WORKDIR` + `COPY` paths |
| `permission denied` on volume | `chown` in Dockerfile or `user:` in Compose |
| `exec format error` | Wrong platform — `--platform linux/amd64` |

### 12.2 Debug Commands

```bash
docker logs -f --tail 100 backend
docker inspect backend                 # Full JSON config
docker exec -it backend sh
docker exec -it backend env            # List env vars
docker exec -it backend cat /etc/hosts # DNS
docker stats                           # Live CPU/mem
docker events                          # Real-time events
```

### 12.3 Inside Container Testing

```bash
docker exec -it backend sh
# Now inside:
ping mongo
curl http://ai-service:8000/health
nslookup ai-service
```

### 12.4 Override ENTRYPOINT for Debugging

```bash
docker run -it --entrypoint sh my-api:1.0
```

### 12.5 VS Code Dev Containers
Use `.devcontainer/devcontainer.json` to develop **inside** the container. No more "works on my machine."

---

## 13. What's Next (Beyond Docker)

Once comfortable with this file, level up:

### Orchestration
- **Kubernetes** (minikube/k3d/kind locally)
- **Docker Swarm** (simpler, good stepping stone)

### CI/CD
- **GitHub Actions** — build & push images
- **GitLab CI**, **Jenkins**

### Registries
- **Docker Hub**, **GitHub Container Registry (ghcr.io)**, **AWS ECR**

### IaC
- **Terraform** — provision cloud infra
- **Ansible** — configure servers

### Monitoring
- **Prometheus + Grafana**
- **Loki** for logs

### Sample GitHub Action (build & push)

```yaml
name: Build & Push
on: { push: { branches: [main] } }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: ghcr.io/${{ github.repository }}/backend:latest
```

---

## 14. Cheat Sheet

### Images
```bash
docker build -t name:tag .
docker images
docker rmi name:tag
docker image prune -a
docker history name:tag
```

### Containers
```bash
docker run -d -p 80:80 --name web nginx
docker ps -a
docker start/stop/restart web
docker logs -f web
docker exec -it web sh
docker rm -f web
```

### Compose
```bash
docker compose up -d --build
docker compose ps
docker compose logs -f [service]
docker compose exec [service] sh
docker compose down
docker compose down -v
docker compose restart [service]
```

### Volumes & Networks
```bash
docker volume ls / create / inspect / rm
docker network ls / create / inspect / rm
```

### Cleanup
```bash
docker system prune -a --volumes   # ⚠️ Removes EVERYTHING unused
```

---

## 🎓 Final Words

You now have the complete path from **zero → running MERN + AI in Docker**.

**Your learning loop:**
1. Build it manually with `docker run`
2. Automate with `docker-compose.yml`
3. Optimize the Dockerfile
4. Add a new service (Redis, AI, queue)
5. Ship it to production

**The Docker mindset:**
> "Everything is a container. Everything is code. If it isn't reproducible, it isn't done."

**Ship it. Break it. Rebuild it. Repeat.** 🚀

---

*Last updated: 2024 · Version 1.0 · Made for MERN + AI developers*