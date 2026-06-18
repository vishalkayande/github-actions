# 🚀 Automated CI/CD Pipeline — GitHub Actions + Docker + Kubernetes on AWS EC2

![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS%20EC2-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

> Every `git push` to `main` automatically builds, containerizes, and deploys the application to a Kubernetes cluster — zero manual steps.

---

## 📌 Project Overview

This project implements a fully automated CI/CD pipeline for a Node.js application. The pipeline uses **GitHub Actions** to build a Docker image, push it to **DockerHub**, and deploy it to a **Kubernetes cluster** running on **AWS EC2** via a self-hosted runner.

### 🔄 Pipeline Flow

```
Developer pushes code
        │
        ▼
  GitHub Actions (Triggered)
        │
        ├─── Job 1: build-and-push (GitHub-hosted runner)
        │         ├── Checkout code
        │         ├── Login to DockerHub
        │         └── Build & Push image (tagged with commit SHA)
        │
        └─── Job 2: deploy (Self-hosted runner on EC2)
                  ├── Checkout code
                  ├── Replace DOCKER_IMAGE placeholder in deployment.yaml
                  └── kubectl apply → Kubernetes rolling update
```

---

## 🛠️ Tech Stack

|          Tool          |                     Purpose                            |
|------------------------|--------------------------------------------------------|
| **GitHub Actions**     | CI/CD automation — triggers on every push to `main`    |
| **Docker**             | Containerizes the Node.js application                  |
| **DockerHub**          | Stores Docker images tagged with commit SHA            |
| **AWS EC2**            | Hosts the Kubernetes cluster and self-hosted runner    |
| **Kubernetes (k3s)**   | Orchestrates container deployment and service exposure |
| **Self-Hosted Runner** | Executes deployment steps directly on EC2              |

---

## 📁 Project Structure

```
github-actions/
├── .github/
│   └── workflows/
│       └── cicd.yml          # GitHub Actions workflow
├── k8s/
│   ├── deployment.yaml       # Kubernetes Deployment manifest
│   └── service.yaml          # Kubernetes Service manifest
├── Dockerfile                # Docker image definition
├── package.json
├── index.html / app files
└── README.md
```

---

## ⚙️ GitHub Actions Workflow

```yaml
name: CI/CD Pipeline — GitHub Actions + Kubernetes

on:
  push:
    branches: [ main ]

jobs:

  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/app:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Replace image in deployment.yaml
        run: |
          sed -i 's|DOCKER_IMAGE|${{ secrets.DOCKER_USERNAME }}/app:${{ github.sha }}|g' \
            k8s/deployment.yaml

      - name: Apply Kubernetes Manifests
        run: kubectl apply -f k8s/
```

---

## 🔐 GitHub Secrets Required

Go to your repo → **Settings → Secrets and variables → Actions** and add:

| Secret Name       |               Description                                    |
|-------------------|--------------------------------------------------------------|
| `DOCKER_USERNAME` | Your DockerHub username                                      |
| `DOCKER_PASSWORD` | DockerHub access token (Settings → Security → Access Tokens) |

---

## 🖥️ EC2 & Kubernetes Setup

### 1. Create EC2 Instance
- **AMI:** Ubuntu Server 22.04 LTS
- **Instance Type:** t2.medium (minimum for Kubernetes)
- **Storage:** 20 GB
- **Security Group:** Allow ports 22, 80, 443, 30000–32767

### 2. Install k3s (Lightweight Kubernetes)
```bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

### 3. Install Docker
```bash
sudo apt update && sudo apt install docker.io -y
sudo systemctl start docker && sudo systemctl enable docker
```

### 4. Configure ubuntu User
```bash
# Copy kubeconfig to ubuntu user
sudo mkdir -p /home/ubuntu/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/ubuntu/.kube/config
sudo chown ubuntu:ubuntu /home/ubuntu/.kube/config

# Add ubuntu to docker group
sudo usermod -aG docker ubuntu
newgrp docker
```

### 5. Register Self-Hosted Runner
```bash
# As ubuntu user — paste commands from:
# GitHub → Repo → Settings → Actions → Runners → New self-hosted runner

mkdir actions-runner && cd actions-runner
# (paste download + config commands from GitHub)
./run.sh
```

---

## 🚀 How to Use

1. **Clone this repository**
   ```bash
   git clone https://github.com/vishalkayande/github-actions.git
   cd github-actions
   ```

2. **Add GitHub Secrets** (`DOCKER_USERNAME`, `DOCKER_PASSWORD`)

3. **Set up EC2 + Kubernetes + Self-Hosted Runner** (see above)

4. **Push any change to trigger the pipeline**
   ```bash
   git add .
   git commit -m "your message"
   git push origin main
   ```

5. **Watch it deploy automatically** under the **Actions** tab on GitHub

---

## ✅ Verify Deployment

```bash
# On EC2 as ubuntu user:
kubectl get pods
kubectl get svc

# Access the app in browser:
http://<EC2-PUBLIC-IP>:30080
```

---

## 🐛 Troubleshooting

|           Error               |                  Fix                                   |
|-------------------------------|--------------------------------------------------------|
| `kubectl: permission denied`  | `sudo chown ubuntu:ubuntu ~/.kube/config`              |
| `docker: permission denied`   | `sudo usermod -aG docker ubuntu && newgrp docker`      |
| Runner shows offline          | SSH to EC2 → `cd actions-runner && ./run.sh`           |
| DockerHub push failed         | Verify `DOCKER_USERNAME` / `DOCKER_PASSWORD` secrets   |
| Pod in `CrashLoopBackOff`     | `kubectl logs <pod-name>` to check app errors          |
| App not accessible in browser | Add inbound rule for port 30080 in EC2 Security Group  |

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).
