# 🚀 Kestra Installation Guide

<div align="center">
  <img src="https://kestra.io/images/kestra-logo.svg" alt="Kestra Logo" width="220" />
</div>

Welcome to the **Kestra Installation Guide**. This guide is designed to help event participants install and run Kestra quickly on **Windows**, **macOS**, and **Linux** using Docker.

Kestra is an open-source orchestration platform for data pipelines, workflows, and automation.

---

## 📌 Why Kestra?

Kestra helps teams build, schedule, observe, and manage data workflows with a strong developer experience and a clean UI.

Popular use cases include:
- Data pipelines
- ETL/ELT orchestration
- Workflow automation
- Scheduled jobs
- Infrastructure automation

---

## 📚 Table of Contents

1. [System Requirements](#-system-requirements)
2. [Install Docker](#-install-docker)
3. [Install Kestra with Docker](#-install-kestra-with-docker)
4. [Windows Setup](#-windows-setup)
5. [macOS Setup](#-macos-setup)
6. [Linux Setup](#-linux-setup)
7. [Troubleshooting](#-troubleshooting)
8. [Next Steps](#-next-steps)

---

## 🖥️ System Requirements

### Minimum Recommended
- CPU: 2+ cores
- RAM: 4 GB minimum, 8 GB recommended
- Storage: 10 GB free space
- Docker: 20.x or newer
- Docker Compose: 1.29+ recommended

### OS Support
- Windows 10/11 (64-bit)
- macOS 11+ (Big Sur or newer)
- Ubuntu 20.04+, Debian 10+, CentOS 7+, or similar

> For best results on Windows, use Docker Desktop with WSL 2 enabled.

---

## 🐳 Install Docker

### Windows

1. Download Docker Desktop:
   - https://www.docker.com/products/docker-desktop/

2. Install Docker Desktop and restart your machine.

3. Enable WSL 2 if prompted:
   ```powershell
   wsl --install
   ```

4. Verify installation:
   ```powershell
   docker --version
   docker run hello-world
   ```

### macOS

1. Download Docker Desktop for Mac:
   - https://www.docker.com/products/docker-desktop/

2. Drag Docker to Applications and launch it.

3. Wait for Docker to finish starting.

4. Verify installation:
   ```bash
   docker --version
   docker run hello-world
   ```

### Linux

#### Ubuntu / Debian
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker
```

#### CentOS / RHEL
```bash
sudo yum install -y docker docker-compose
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker
```

Verify:
```bash
docker --version
docker run hello-world
```

---

## 🚀 Install Kestra with Docker

There are two common ways to run Kestra locally:

- Quick single-container setup for testing
- Docker Compose setup for a more realistic local environment

### Method 1: Quick Start (Simple)

Run the following command:

```bash
docker run --pull=always --rm -it -p 8080:8080 \
  --name kestra \
  kestralabs/kestra:latest-full server local
```

Then open:
- http://localhost:8080

This works on Windows, macOS, and Linux.

### Method 2: Docker Compose (Recommended)

This gives you a more stable setup with PostgreSQL and persistent storage.

#### Step 1: Create a project folder

```bash
mkdir kestra-project
cd kestra-project
```

#### Step 2: Create a file named `docker-compose.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: kestra
      POSTGRES_PASSWORD: kestra
      POSTGRES_DB: kestra
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kestra"]
      interval: 10s
      timeout: 5s
      retries: 5

  kestra:
    image: kestralabs/kestra:latest-full
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8080:8080"
    environment:
      KESTRA_DATABASE_TYPE: postgres
      KESTRA_DATABASE_URL: jdbc:postgresql://postgres:5432/kestra
      KESTRA_DATABASE_USERNAME: kestra
      KESTRA_DATABASE_PASSWORD: kestra
      KESTRA_QUEUES_TYPE: postgres
    volumes:
      - kestra-data:/app/storage
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp:/tmp
    command: server

volumes:
  postgres-data:
  kestra-data:
```

#### Step 3: Start the stack

```bash
docker compose up -d
```

#### Step 4: Check the service

```bash
docker compose ps
```

#### Step 5: Open Kestra UI

Visit:
- http://localhost:8080

#### Step 6: Stop the stack

```bash
docker compose down
```

---

## 🪟 Windows Setup

### Recommended Setup
- Install Docker Desktop
- Enable WSL 2
- Run Kestra from a folder inside your WSL home or local Windows directory

### Common Windows Commands

PowerShell example:
```powershell
mkdir kestra-project
cd kestra-project

# create docker-compose.yml here

docker compose up -d
```

### Windows Troubleshooting

#### Docker Desktop not starting
- Check virtualization is enabled in BIOS
- Reinstall Docker Desktop
- Ensure WSL 2 is installed:
  ```powershell
  wsl --status
  ```

#### Port 8080 already in use
```powershell
netstat -ano | findstr :8080
```
Then kill the process:
```powershell
taskkill /PID <PID> /F
```

#### WSL 2 issues
```powershell
wsl --update
wsl --set-default-version 2
```

> On Windows, Docker Desktop works best when WSL 2 is enabled and your project folder is under the Linux filesystem if possible.

---

## 🍏 macOS Setup

### Recommended Setup
- Install Docker Desktop for Mac
- Use Apple Silicon (M1/M2) image if needed

### Example Commands

```bash
mkdir kestra-project
cd kestra-project
```

Then create your `docker-compose.yml` and run:

```bash
docker compose up -d
```

### macOS Troubleshooting

#### Docker says it is not running
- Open Docker Desktop and wait for the engine to start
- Restart Docker from the menu bar

#### Permission errors
```bash
sudo chown -R $(whoami) ~/.docker
```

#### Apple Silicon issues
If you use M1/M2, make sure the Docker image supports ARM64 or use the correct Docker Desktop setup.

---

## 🐧 Linux Setup

### Ubuntu / Debian Example

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker
```

Then run:

```bash
mkdir kestra-project
cd kestra-project
```

Create `docker-compose.yml` and then:

```bash
docker compose up -d
```

### Linux Troubleshooting

#### Permission denied
```bash
sudo usermod -aG docker $USER
newgrp docker
```

#### Docker service not starting
```bash
sudo systemctl status docker
sudo systemctl restart docker
```

#### Port access problems
Check firewall rules and ensure port 8080 is available.

---

## 🛠️ Troubleshooting

### 1. Kestra does not open at http://localhost:8080

Check whether the container is running:
```bash
docker ps
```

If not running, inspect logs:
```bash
docker logs <container_name>
```

Typical names:
- `kestra`
- `kestra-server`

### 2. Port 8080 already in use

On macOS/Linux:
```bash
lsof -i :8080
```

On Windows:
```powershell
netstat -ano | findstr :8080
```

Then stop the process or use a different port.

### 3. Docker out of memory

Increase memory allocation in Docker Desktop:
- Windows/macOS: Docker Desktop → Settings → Resources → Memory
- Set to at least 4 GB, ideally 6-8 GB

### 4. PostgreSQL connection errors

Check the database service health:
```bash
docker compose ps
```

View logs:
```bash
docker compose logs postgres
```

### 5. Docker command not found

Linux fix:
```bash
sudo apt install -y docker.io docker-compose
```

Windows/macOS fix:
- Reinstall Docker Desktop and restart the system.

### 6. Permission denied during volume creation

On Linux, check ownership:
```bash
sudo chown -R $USER:$USER .
```

---

## 🔍 Quick Reference

### Start Kestra
```bash
docker compose up -d
```

### View logs
```bash
docker compose logs -f
```

### Stop Kestra
```bash
docker compose down
```

### Open UI
- http://localhost:8080

---

## 🎯 Next Steps

Once activated, you can:
- Open the Kestra UI
- Create your first workflow
- Connect a data source or trigger
- Explore the official documentation at https://kestra.io/docs

Official website:
- https://kestra.io/

GitHub:
- https://github.com/kestra-io/kestra

---

## ✅ Final Notes

This setup is perfect for workshops, demos, and hands-on labs. It gives participants a quick and reliable way to get started with Kestra without requiring heavy production infrastructure.

If you want a version tailored specifically for a workshop handout or event booklet, we can also prepare:
- a shorter one-page version
- a slide-friendly version
- a version with branding and event-specific headings

---

<div align="center">
  <img src="https://kestra.io/images/kestra-logo.svg" alt="Kestra Logo" width="180" />
</div>

Made for the Kestra event workshop.
