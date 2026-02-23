# 📋 CRUD Application — MEAN Stack with Docker & CI/CD

A full-stack **CRUD (Create, Read, Update, Delete)** application built with the **MEAN stack** (MongoDB, Express.js, Angular, Node.js), containerized with **Docker**, and automated with **GitHub Actions CI/CD pipeline**.

> **Live URL:** `http://3.8.3.228/`

---

## 📑 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Docker Configuration](#docker-configuration)
  - [Backend Dockerfile](#backend-dockerfile)
  - [Frontend Dockerfile](#frontend-dockerfile)
  - [Docker Compose](#docker-compose)
- [Nginx Configuration](#nginx-configuration)
  - [Reverse Proxy (Main)](#reverse-proxy-main)
  - [Frontend Nginx](#frontend-nginx)
- [CI/CD Pipeline](#cicd-pipeline)
  - [Pipeline Overview](#pipeline-overview)
  - [GitHub Secrets Required](#github-secrets-required)
  - [Pipeline Stages](#pipeline-stages)
- [Setup & Deployment](#setup--deployment)
  - [Prerequisites](#prerequisites)
  - [Local Development](#local-development)
  - [Production Deployment](#production-deployment)
- [API Endpoints](#api-endpoints)
- [Screenshots](#screenshots)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                      AWS EC2 VM                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Nginx Reverse Proxy (:80)          │    │
│  │                                                 │    │
│  │   /api/*  ──►  Backend Container (:8080)        │    │
│  │   /*      ──►  Frontend Container (:80)         │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Frontend   │  │   Backend    │  │   MongoDB    │  │
│  │   (Angular)  │  │  (Node.js)   │  │   (Mongo 6)  │  │
│  │   Nginx:80   │  │  Express:8080│  │    :27017    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│         All connected via Docker Network                │
└─────────────────────────────────────────────────────────┘
```

**Flow:**
1. User opens `http://3.8.3.228/` in browser
2. Nginx reverse proxy receives the request on port **80**
3. Routes `/api/*` requests → **Backend** (Node.js + Express)
4. Routes all other requests → **Frontend** (Angular + Nginx)
5. Backend communicates with **MongoDB** for data operations

---

## Tech Stack

| Layer          | Technology              | Purpose                    |
| -------------- | ----------------------- | -------------------------- |
| **Frontend**   | Angular 15              | User Interface (SPA)       |
| **Backend**    | Node.js + Express.js    | REST API Server            |
| **Database**   | MongoDB 6.0             | NoSQL Data Storage         |
| **Web Server** | Nginx 1.25              | Reverse Proxy & Static Files |
| **Container**  | Docker + Docker Compose | Containerization           |
| **CI/CD**      | GitHub Actions          | Automated Build & Deploy   |
| **Registry**   | Docker Hub              | Image Storage              |
| **Cloud**      | AWS EC2 (Ubuntu)        | Production Server          |

---

## Project Structure

```
crud-dd-task-mean-app/
├── .github/
│   └── workflows/
│       └── ci-cd.yml              # GitHub Actions CI/CD pipeline
├── backend/
│   ├── Dockerfile                 # Backend Docker image config
│   ├── package.json               # Node.js dependencies
│   ├── server.js                  # Express server entry point
│   └── app/
│       ├── config/
│       │   └── db.config.js       # MongoDB connection config
│       ├── controllers/
│       │   └── tutorial.controller.js  # CRUD logic
│       ├── models/
│       │   ├── index.js           # Mongoose setup
│       │   └── tutorial.model.js  # Tutorial schema
│       └── routes/
│           └── turorial.routes.js # API route definitions
├── frontend/
│   ├── Dockerfile                 # Frontend Docker image (multi-stage)
│   ├── nginx.conf                 # Nginx config for Angular routing
│   ├── package.json               # Angular dependencies
│   └── src/
│       └── app/
│           ├── components/        # Angular components
│           │   ├── add-tutorial/
│           │   ├── tutorial-details/
│           │   └── tutorials-list/
│           ├── models/            # TypeScript interfaces
│           └── services/          # HTTP service layer
├── nginx/
│   └── nginx.conf                 # Reverse proxy configuration
├── docker-compose.yml             # Multi-container orchestration
└── README.md                      # This file
```

---

## Docker Configuration

### Backend Dockerfile

**File:** `backend/Dockerfile`

```dockerfile
# Production image using lightweight Alpine
FROM node:18-alpine

WORKDIR /app

# Copy dependency files first (Docker layer caching)
COPY package*.json ./

# Install only production dependencies
RUN npm install --production

# Copy application source code
COPY . .

EXPOSE 8080

CMD ["node", "server.js"]
```

**Key Points:**
- Uses `node:18-alpine` — lightweight (~50MB vs ~350MB for full image)
- Copies `package*.json` first for **Docker layer caching** — dependencies are cached unless `package.json` changes
- `--production` flag skips devDependencies, reducing image size

---

### Frontend Dockerfile

**File:** `frontend/Dockerfile`

```dockerfile
# Stage 1: Build Angular app with Node.js
FROM node:18-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build -- --configuration production

# Stage 2: Serve with Nginx (no Node.js needed!)
FROM nginx:1.25-alpine

RUN rm -rf /usr/share/nginx/html/*
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Key Points:**
- **Multi-stage build** — Stage 1 compiles Angular, Stage 2 serves static files
- Final image contains only Nginx + compiled HTML/JS/CSS (~25MB)
- Node.js is NOT included in the production image

---

### Docker Compose

**File:** `docker-compose.yml`

```yaml
version: '3.8'

services:
  # Database
  mongodb:
    image: mongo:6.0
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    networks:
      - app-network

  # Backend API
  backend:
    image: yaswanth3333/crud-backend:latest
    container_name: backend
    restart: always
    ports:
      - "8080:8080"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/dd_db
      - CORS_ORIGIN=*
      - PORT=8080
    depends_on:
      - mongodb
    networks:
      - app-network

  # Frontend (Angular)
  frontend:
    image: yaswanth3333/crud-frontend:latest
    container_name: frontend
    restart: always
    ports:
      - "4200:80"
    depends_on:
      - backend
    networks:
      - app-network

  # Nginx Reverse Proxy
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

volumes:
  mongodb_data:

networks:
  app-network:
    driver: bridge
```

**Key Points:**
- **4 services**: MongoDB, Backend, Frontend, Nginx
- `mongodb_data` volume persists database data across container restarts
- `app-network` allows containers to communicate using service names as hostnames
- `depends_on` ensures correct startup order
- `restart: always` auto-restarts crashed containers

---

## Nginx Configuration

### Reverse Proxy (Main)

**File:** `nginx/nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    # API requests → Backend container
    location /api/ {
        proxy_pass http://backend:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # All other requests → Frontend container
    location / {
        proxy_pass http://frontend:80/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**How Routing Works:**
| Request URL | Nginx Routes To | Container |
|---|---|---|
| `http://3.8.3.228/` | `frontend:80` | Angular App |
| `http://3.8.3.228/tutorials` | `frontend:80` | Angular App |
| `http://3.8.3.228/api/tutorials` | `backend:8080` | Node.js API |
| `http://3.8.3.228/api/tutorials/123` | `backend:8080` | Node.js API |

---

### Frontend Nginx

**File:** `frontend/nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;   # Angular SPA routing support
    }
}
```

**Why `try_files`?**
- Angular is a Single Page Application (SPA)
- When user navigates to `/tutorials`, the browser requests that path from Nginx
- Without `try_files`, Nginx returns 404 because there's no `/tutorials` file
- `try_files` falls back to `index.html`, letting Angular's router handle the URL

---

## CI/CD Pipeline

### Pipeline Overview

```
┌──────────┐     ┌────────────────┐     ┌──────────────┐     ┌──────────┐
│  Developer│────►│  GitHub Repo   │────►│GitHub Actions │────►│  AWS VM  │
│  git push │     │  (main branch) │     │  CI/CD        │     │  Deploy  │
└──────────┘     └────────────────┘     └──────────────┘     └──────────┘
                                              │
                                              ▼
                                        ┌──────────────┐
                                        │  Docker Hub   │
                                        │  (Images)     │
                                        └──────────────┘
```

**Trigger:** Every `git push` to `main` branch (or manual trigger from GitHub UI)

---

### GitHub Secrets Required

Go to: **GitHub Repo → Settings → Secrets and variables → Actions**

| Secret Name | Description | Example |
|---|---|---|
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token | `dckr_pat_xxxx...` |
| `VM_HOST` | EC2 instance public IP | `3.8.3.228` |
| `VM_USERNAME` | SSH username for the VM | `ubuntu` |
| `VM_SSH_KEY` | Private SSH key (`.pem` file content) | `-----BEGIN RSA PRIVATE KEY-----...` |

**How to create Docker Hub Token:**
1. Go to [hub.docker.com](https://hub.docker.com) → Login
2. Account Settings → Security → Personal Access Tokens
3. Click **Generate New Token** → Name: `github-cicd` → Access: **Read & Write**
4. Copy the token

---

### Pipeline Stages

**File:** `.github/workflows/ci-cd.yml`

#### Stage 1: Build & Push Docker Images

```yaml
build-and-push:
  runs-on: ubuntu-latest
  steps:
    - Checkout code from GitHub
    - Set up Docker Buildx
    - Login to Docker Hub
    - Build & Push Backend Image (tagged: latest + commit SHA)
    - Build & Push Frontend Image (tagged: latest + commit SHA)
```

#### Stage 2: Deploy to VM

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: build-and-push        # Waits for Stage 1 to succeed
  steps:
    - Copy docker-compose.yml & nginx config to VM via SCP
    - SSH into VM and run:
        - docker compose pull   # Pull latest images
        - docker compose down   # Stop old containers
        - docker compose up -d  # Start new containers
        - docker image prune    # Clean up old images
```

**Result:** Every `git push` → Images built → Pushed to Docker Hub → Deployed on VM → App is live! 🚀

---

## Setup & Deployment

### Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Docker | 20+ | Container runtime |
| Docker Compose | v2+ | Multi-container orchestration |
| Node.js | 18+ | Local development (optional) |
| Git | 2+ | Version control |
| AWS Account | — | EC2 instance for hosting |

---

### Local Development

```bash
# 1. Clone the repository
git clone https://github.com/yaswanth756/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app

# 2. Start all services
docker compose up -d

# 3. Open in browser
# App:     http://localhost
# API:     http://localhost:8080/api/tutorials
# MongoDB: localhost:27017
```

**Stop the app:**
```bash
docker compose down
```

**Stop and remove all data:**
```bash
docker compose down -v    # -v removes MongoDB volume too
```

---

### Production Deployment

#### Step 1: Launch AWS EC2 Instance

1. Go to AWS Console → EC2 → **Launch Instance**
2. **Name:** `crud-mean-app-server`
3. **AMI:** Ubuntu 22.04 LTS
4. **Instance type:** `t2.micro` (free tier) or `t2.small`
5. **Key pair:** Create or select existing `.pem` key
6. **Security Group** — Open these ports:

| Port | Type | Source | Purpose |
|------|------|--------|---------|
| 22 | SSH | Your IP | SSH access |
| 80 | HTTP | 0.0.0.0/0 | Web app (Nginx) |
| 8080 | Custom TCP | 0.0.0.0/0 | Backend API |

#### Step 2: Install Docker on EC2

```bash
# SSH into your VM
ssh -i your-key.pem ubuntu@3.8.3.228

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin

# Add user to docker group (no sudo needed for docker commands)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

#### Step 3: Configure GitHub Secrets

Add the 4 secrets mentioned in [GitHub Secrets Required](#github-secrets-required).

#### Step 4: Push Code & Auto-Deploy

```bash
# Make a change, commit, and push
git add .
git commit -m "deploy: update application"
git push origin main

# GitHub Actions will automatically:
# 1. Build Docker images
# 2. Push to Docker Hub
# 3. SSH into VM and deploy
```

#### Step 5: Verify Deployment

```bash
# Check running containers on VM
ssh -i your-key.pem ubuntu@3.8.3.228
docker compose ps

# Or test from your browser:
# Frontend:  http://3.8.3.228/
# API:       http://3.8.3.228/api/tutorials
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/tutorials` | Get all tutorials |
| `GET` | `/api/tutorials/:id` | Get tutorial by ID |
| `POST` | `/api/tutorials` | Create new tutorial |
| `PUT` | `/api/tutorials/:id` | Update tutorial by ID |
| `DELETE` | `/api/tutorials/:id` | Delete tutorial by ID |
| `DELETE` | `/api/tutorials` | Delete all tutorials |
| `GET` | `/api/tutorials?title=keyword` | Search tutorials by title |

**Example — Create a tutorial:**
```bash
curl -X POST http://3.8.3.228/api/tutorials \
  -H "Content-Type: application/json" \
  -d '{"title": "Docker Tutorial", "description": "Learn Docker basics"}'
```



## Docker Hub Images

| Image | URL |
|-------|-----|
| Backend | [yaswanth3333/crud-backend](https://hub.docker.com/r/yaswanth3333/crud-backend) |
| Frontend | [yaswanth3333/crud-frontend](https://hub.docker.com/r/yaswanth3333/crud-frontend) |

**Pull manually:**
```bash
docker pull yaswanth3333/crud-backend:latest
docker pull yaswanth3333/crud-frontend:latest
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Site not loading | Check EC2 Security Group — ports 80, 8080 must be open |
| Container crashing | Run `docker compose logs <service-name>` to see errors |
| MongoDB connection failed | Ensure MongoDB container is running: `docker compose ps` |
| CI/CD pipeline failed | Check GitHub Actions logs — verify all 4 secrets are set |
| Images not pulling | Verify Docker Hub token has Read & Write permissions |
| Port already in use | Run `docker compose down` first, then `docker compose up -d` |

---

## Author

**Yaswanth** — [GitHub](https://github.com/yaswanth756)

---
