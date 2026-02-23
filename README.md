# MEAN Stack CRUD Application — DevOps Deployment

A full-stack CRUD application built with the **MEAN stack** (MongoDB, Express, Angular 15, Node.js), containerized with Docker, deployed on an Ubuntu VM with Nginx reverse proxy, and automated with a CI/CD pipeline using GitHub Actions.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Step 1: Local Development Setup](#step-1-local-development-setup)
- [Step 2: Docker Hub Setup](#step-2-docker-hub-setup)
- [Step 3: Build and Push Docker Images](#step-3-build-and-push-docker-images)
- [Step 4: Ubuntu VM Setup (Cloud)](#step-4-ubuntu-vm-setup-cloud)
- [Step 5: Deploy on VM with Docker Compose](#step-5-deploy-on-vm-with-docker-compose)
- [Step 6: GitHub Repository Setup](#step-6-github-repository-setup)
- [Step 7: CI/CD Pipeline Configuration](#step-7-cicd-pipeline-configuration)
- [Step 8: Testing the CI/CD Pipeline](#step-8-testing-the-cicd-pipeline)
- [Nginx Reverse Proxy Explained](#nginx-reverse-proxy-explained)
- [Troubleshooting](#troubleshooting)
- [Screenshots](#screenshots)

---

## Architecture Overview

```
                         ┌─────────────────────────────────────────────┐
                         │              Ubuntu VM (Cloud)              │
                         │                                             │
  User Browser ──────►   │  ┌──────────────────────────────────────┐   │
  (Port 80)              │  │        Nginx Reverse Proxy           │   │
                         │  │           (Port 80)                  │   │
                         │  │                                      │   │
                         │  │   /api/*  ──►  Backend (Port 8080)   │   │
                         │  │   /*      ──►  Frontend (Port 4200)  │   │
                         │  └──────────────────────────────────────┘   │
                         │                                             │
                         │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
                         │  │ Frontend │  │ Backend  │  │ MongoDB  │  │
                         │  │ (Angular)│  │ (Node.js)│  │ (DB)     │  │
                         │  │ Nginx    │  │ Express  │  │ Port     │  │
                         │  │ :4200    │  │ :8080    │  │ :27017   │  │
                         │  └──────────┘  └──────────┘  └──────────┘  │
                         │         All connected via Docker Network    │
                         └─────────────────────────────────────────────┘
```

**How it works:**
1. User opens `http://<VM_IP>` in their browser (port 80)
2. Nginx receives the request
3. If the URL starts with `/api/`, Nginx forwards it to the **Backend** container
4. If the URL is anything else (`/`, `/tutorials`, etc.), Nginx forwards it to the **Frontend** container
5. The Backend talks to **MongoDB** using Docker's internal DNS (`mongodb://mongodb:27017/dd_db`)

---

## Project Structure

```
crud-dd-task-mean-app/
├── .github/
│   └── workflows/
│       └── deploy.yml              # CI/CD pipeline (GitHub Actions)
├── backend/
│   ├── Dockerfile                  # Docker image for backend
│   ├── .dockerignore               # Files to exclude from Docker build
│   ├── package.json
│   ├── server.js                   # Express server entry point
│   └── app/
│       ├── config/db.config.js     # MongoDB connection (uses env variable)
│       ├── controllers/            # API logic
│       ├── models/                 # Mongoose schemas
│       └── routes/                 # API routes
├── frontend/
│   ├── Dockerfile                  # Multi-stage Docker image (build + serve)
│   ├── .dockerignore
│   ├── nginx.conf                  # Nginx config inside frontend container
│   ├── package.json
│   └── src/                        # Angular source code
├── nginx/
│   └── nginx.conf                  # Nginx reverse proxy config (main entry)
├── docker-compose.yml              # Defines all 4 services
├── .gitignore
└── README.md
```

---

## Prerequisites

- **Docker** and **Docker Compose** installed
- **Docker Hub** account ([hub.docker.com](https://hub.docker.com))
- **GitHub** account
- **Cloud VM** (AWS EC2 / Azure / GCP / DigitalOcean) running Ubuntu 22.04+
- **SSH key pair** for VM access
- **Node.js 18+** and **Angular CLI** (for local development only)

---

## Step 1: Local Development Setup

```bash
# Clone the repository
git clone https://github.com/<your-username>/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app

# Backend
cd backend
npm install
node server.js
# Backend runs on http://localhost:8080

# Frontend (in a new terminal)
cd frontend
npm install
ng serve --port 8081
# Frontend runs on http://localhost:8081
```

> **Note:** For local development, you need MongoDB running locally on port 27017.

---

## Step 2: Docker Hub Setup

1. Go to [hub.docker.com](https://hub.docker.com) and create an account (if you don't have one)
2. Create an **Access Token**:
   - Go to **Account Settings** → **Security** → **New Access Token**
   - Name it `github-actions`
   - Copy and save the token (you won't see it again!)
3. Create two repositories on Docker Hub:
   - `<your-username>/crud-backend`
   - `<your-username>/crud-frontend`

---

## Step 3: Build and Push Docker Images

```bash
# From the project root directory
cd crud-dd-task-mean-app

# Login to Docker Hub
docker login -u <your-dockerhub-username>

# Build and push Backend image
docker build -t <your-dockerhub-username>/crud-backend:latest ./backend
docker push <your-dockerhub-username>/crud-backend:latest

# Build and push Frontend image
docker build -t <your-dockerhub-username>/crud-frontend:latest ./frontend
docker push <your-dockerhub-username>/crud-frontend:latest
```

**What's happening?**
- `docker build` reads the `Dockerfile` and creates an image
- `-t` tags the image with your Docker Hub username and repository name
- `docker push` uploads the image to Docker Hub

---

## Step 4: Ubuntu VM Setup (Cloud)

### 4.1 Create the VM

**AWS EC2 example:**
1. Launch an EC2 instance:
   - **AMI:** Ubuntu 22.04 LTS
   - **Instance type:** t2.medium (recommended) or t2.micro (free tier)
   - **Security Group:** Allow inbound ports: **22** (SSH), **80** (HTTP), **8080** (Backend)
   - **Key pair:** Create or use existing `.pem` key
2. Note down the **Public IP address**

### 4.2 SSH into the VM

```bash
# Connect to your VM
ssh -i <your-key.pem> ubuntu@<VM_PUBLIC_IP>
```

### 4.3 Install Docker and Docker Compose on the VM

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io

# Start Docker and enable it on boot
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to the docker group (so you don't need 'sudo' every time)
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt install -y docker-compose-plugin

# Log out and log back in for group changes to take effect
exit
# SSH back in
ssh -i <your-key.pem> ubuntu@<VM_PUBLIC_IP>

# Verify installation
docker --version
docker compose version
```

### 4.4 Install Git and Clone the Repository

```bash
# Install Git
sudo apt install -y git

# Clone your repository
cd ~
git clone https://github.com/<your-username>/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app
```

---

## Step 5: Deploy on VM with Docker Compose

```bash
# Navigate to the project directory
cd ~/crud-dd-task-mean-app

# Set your Docker Hub username as an environment variable
export DOCKERHUB_USERNAME=<your-dockerhub-username>

# Pull the Docker images from Docker Hub
docker compose pull

# Start all services in the background
docker compose up -d

# Check that all containers are running
docker compose ps

# Check logs (optional)
docker compose logs -f
```

**Expected output of `docker compose ps`:**
```
NAME            IMAGE                                    STATUS          PORTS
mongodb         mongo:6.0                                Up              0.0.0.0:27017->27017/tcp
backend         <username>/crud-backend:latest           Up              0.0.0.0:8080->8080/tcp
frontend        <username>/crud-frontend:latest          Up              0.0.0.0:4200->80/tcp
nginx-proxy     nginx:1.25-alpine                        Up              0.0.0.0:80->80/tcp
```

**Access the application:** Open `http://<VM_PUBLIC_IP>` in your browser!

---

## Step 6: GitHub Repository Setup

### 6.1 Create a GitHub Repository

1. Go to [github.com](https://github.com) → **New Repository**
2. Name: `crud-dd-task-mean-app`
3. Visibility: Public
4. Don't initialize with README (we already have one)

### 6.2 Push Code to GitHub

```bash
cd crud-dd-task-mean-app

# Initialize git (if not already)
git init
git add .
git commit -m "Initial commit: MEAN stack CRUD app with Docker and CI/CD"

# Add remote and push
git remote add origin https://github.com/<your-username>/crud-dd-task-mean-app.git
git branch -M main
git push -u origin main
```

---

## Step 7: CI/CD Pipeline Configuration

### 7.1 Add GitHub Secrets

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these 4 secrets:

| Secret Name | Value | Description |
|---|---|---|
| `DOCKERHUB_USERNAME` | `your-dockerhub-username` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | `dckr_pat_xxxxx` | Docker Hub access token (from Step 2) |
| `VM_HOST` | `1.2.3.4` | Your VM's public IP address |
| `VM_USERNAME` | `ubuntu` | SSH username for the VM |
| `VM_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----...` | Contents of your private SSH key file |

### 7.2 How to Get the SSH Private Key

```bash
# On your local machine, display the private key content
cat ~/.ssh/your-key.pem
# OR
cat ~/.ssh/id_rsa

# Copy the ENTIRE output (including BEGIN and END lines) and paste as VM_SSH_KEY secret
```

### 7.3 Understanding the Pipeline

The CI/CD pipeline (`.github/workflows/deploy.yml`) does the following:

```
Push to 'main' branch
        │
        ▼
┌─────────────────────────┐
│  Job 1: Build & Push    │
│                         │
│  1. Checkout code       │
│  2. Login to Docker Hub │
│  3. Build backend image │
│  4. Push backend image  │
│  5. Build frontend image│
│  6. Push frontend image │
└────────┬────────────────┘
         │ (only if Job 1 succeeds)
         ▼
┌─────────────────────────┐
│  Job 2: Deploy          │
│                         │
│  1. SSH into VM         │
│  2. git pull latest code│
│  3. docker compose pull │
│  4. docker compose up   │
│  5. Cleanup old images  │
└─────────────────────────┘
```

---

## Step 8: Testing the CI/CD Pipeline

1. Make a small change (e.g., edit this README)
2. Commit and push:

```bash
git add .
git commit -m "Test CI/CD pipeline"
git push origin main
```

3. Go to your GitHub repository → **Actions** tab
4. Watch the pipeline run!
5. Once complete, refresh `http://<VM_PUBLIC_IP>` to see the changes

---

## Nginx Reverse Proxy Explained

We use **two Nginx instances** in this setup:

### 1. Nginx Reverse Proxy (`nginx/nginx.conf`)
- Runs as a separate container on **port 80**
- Acts as the **main entry point** for the entire application
- Routes requests:
  - `/api/*` → Backend container (port 8080)
  - `/*` → Frontend container (port 4200)

### 2. Nginx inside Frontend Container (`frontend/nginx.conf`)
- Serves the compiled Angular static files (HTML, CSS, JS)
- Handles Angular client-side routing with `try_files`
- Without this, refreshing on `/tutorials` would give a 404

**Why Nginx as reverse proxy?**
- Single entry point (port 80) — users don't need to know about internal ports
- Can add SSL/HTTPS later without changing the app
- Load balancing capability if you scale up
- Security — internal services aren't directly exposed

---

## Troubleshooting

### Check container status
```bash
docker compose ps
```

### View logs for a specific service
```bash
docker compose logs backend
docker compose logs frontend
docker compose logs mongodb
docker compose logs nginx
```

### Restart a specific service
```bash
docker compose restart backend
```

### Rebuild and restart everything
```bash
docker compose down
docker compose pull
docker compose up -d
```

### Check if MongoDB is accessible
```bash
docker exec -it mongodb mongosh
# Inside mongosh:
show dbs
use dd_db
db.tutorials.find()
```

### Common Issues

| Issue | Solution |
|---|---|
| Port 80 already in use | `sudo systemctl stop apache2` or `sudo systemctl stop nginx` |
| Cannot connect to MongoDB | Check if mongodb container is running: `docker compose ps` |
| Frontend shows blank page | Check frontend logs: `docker compose logs frontend` |
| API calls fail (CORS error) | Check backend CORS_ORIGIN env variable in docker-compose.yml |
| CI/CD SSH fails | Verify VM_SSH_KEY secret has the complete private key |
| Docker permission denied | Run `sudo usermod -aG docker $USER` and re-login |

---

## Screenshots

> **Add your screenshots in this section after deployment**

### 1. CI/CD Configuration and Execution
<!-- ![GitHub Actions Pipeline](screenshots/cicd-pipeline.png) -->
- GitHub Actions workflow running
- Build and push job completing successfully
- Deploy job executing SSH commands

### 2. Docker Image Build and Push
<!-- ![Docker Hub Images](screenshots/dockerhub-images.png) -->
- Docker Hub repositories showing pushed images
- Image tags (latest + commit SHA)

### 3. Application Deployment and Working UI
<!-- ![Working Application](screenshots/app-ui.png) -->
- Tutorial list page
- Add tutorial page
- Edit tutorial page

### 4. Nginx Setup and Infrastructure
<!-- ![Infrastructure](screenshots/infrastructure.png) -->
- `docker compose ps` showing all running containers
- Nginx reverse proxy configuration
- VM/Cloud instance details

---

## Tech Stack

| Component | Technology |
|---|---|
| Frontend | Angular 15 |
| Backend | Node.js + Express.js |
| Database | MongoDB 6.0 |
| Containerization | Docker |
| Orchestration | Docker Compose |
| Reverse Proxy | Nginx |
| CI/CD | GitHub Actions |
| Cloud | Ubuntu VM (AWS/Azure/GCP) |

---

## Quick Reference Commands

```bash
# Start the application
docker compose up -d

# Stop the application (keeps data)
docker compose stop

# Stop and remove containers (keeps data in volumes)
docker compose down

# Stop, remove containers AND delete data
docker compose down -v

# View running containers
docker compose ps

# View all logs
docker compose logs -f

# Restart after pulling new images
docker compose pull && docker compose up -d --force-recreate
