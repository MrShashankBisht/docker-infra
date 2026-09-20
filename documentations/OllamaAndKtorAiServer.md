# System Architecture & CI/CD Deployment Guide

## 1. The CI/CD Pipeline (GitHub Actions to Ubuntu Server)

This pipeline utilizes a push-based SSH deployment strategy. Instead of building the Docker image on GitHub and pushing it to a registry, GitHub securely connects to the production server and triggers a local build.

### Step 1: Server Authentication Setup

The server relies on asymmetric cryptography to authenticate GitHub without passwords. Run these commands on your Ubuntu server to generate and authorize the key:

```bash
# 1. Generate the ED25519 key pair (do not enter a passphrase when prompted)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_key

# 2. Add the public key to your server's authorized guest list
cat ~/.ssh/github_actions_key.pub >> ~/.ssh/authorized_keys

# 3. Print the private key so you can copy it to GitHub Secrets
cat ~/.ssh/github_actions_key
```

Add the output of the last command to your GitHub Repository Secrets as SSH_PRIVATE_KEY, along with SERVER_HOST (your IP) and SERVER_USER (your Ubuntu username).

### Step 2: The GitHub Actions Workflow (.github/workflows/deploy.yml)

This file lives in the Git repository and tells GitHub what to do when code is pushed to the main branch.

```yaml
name: Production Deployment

on:
  push:
    branches:
      - main 

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Execute Remote SSH Commands
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/development/ai/ai_server 
            git pull origin main
            # Trigger the deployment script on the server
            chmod +x ./deploy.sh
            ./deploy.sh

```

### Step 3: The Deployment Script (deploy.sh)

This script executes locally on the Ubuntu server. It leverages Docker Compose to rebuild only the Ktor application, leaving the database and AI models running uninterrupted.

```bash
#!/bin/bash
echo "Initiating deployment sequence..."

# 1. Pull the latest configuration (if any changes to compose file)
docker compose -f ollama-aiserver.yaml pull

# 2. Build the Fat JAR and package it into the Distroless container
# Replace 'ai_server' with your exact compose service name if different
docker compose -f ollama-aiserver.yaml up -d --build ai_server

# 3. Clean up dangling images to prevent disk space exhaustion
docker image prune -f

echo "Deployment complete. Container is now live."
```

## 2. Docker Multi-Stage Build Architecture

The application is containerized using a two-stage process to ensure the final image is secure, lightweight, and free of source code.

### Stage 1: The Builder (Gradle & Shadow Plugin)

Base: gradle:8.8-jdk21-alpine

Process:

Downloads Gradle dependencies (cached).

Compiles Kotlin source code.

The Shadow Plugin (io.github.goooler.shadow) extracts all dependencies (Netty, LangChain4j) and bundles them with the compiled code into a single executable *-all.jar (Fat JAR).

### Stage 2: The Runtime (Google Distroless)

Base: gcr.io/distroless/java21-debian12:nonroot

Process:

Copies only the Fat JAR from Stage 1.

Drops all root privileges (runs as nonroot UID 65532).

Applies enterprise JVM flags: -XX:+ExitOnOutOfMemoryError (fail-fast) and -XX:MaxRAMPercentage=50.0 (dynamic heap scaling).

## 3. AI Architecture & Request Lifecycle

The system is designed to route external HTTP requests through a secure proxy, process business logic in a Kotlin backend, and execute local Large Language Models (LLMs) on dedicated GPU hardware.

| Component | Technology | Network Location | Role |
| :--- | :--- | :--- | :--- |
| **Reverse Proxy** | Nginx | Host Network (Port 80/443) | SSL termination, rate limiting, and routing traffic to the correct container. |
| **Backend API** | Ktor (Kotlin) | `global_network` (Port 8080) | Receives requests, manages conversation history, and orchestrates AI calls via LangChain4j. |
| **AI Engine** | Ollama | `global_network` (Port 11434) | Loads `.GGUF` model weights into VRAM and handles inference/generation. |
| **Hardware Bridge**| NVIDIA Toolkit | Host -> Container | Passes the host's physical GPU into the Ollama container for hardware acceleration. |
| **Database** | PostgreSQL | `global_network` (Port 5432) | Persists user sessions, embeddings, and telemetry data. |

### The LLM Request Flow (Top-to-Bottom)

#### Ingestion (Nginx)

A client sends a POST request to api.yourdomain.com/chat. Nginx intercepts this, handles SSL decryption, and proxies the traffic to the Ktor container.

#### Orchestration (Ktor)

The Ktor server receives the JSON payload. It queries PostgreSQL to retrieve the user's previous conversation history to maintain context.

#### Prompt Construction (LangChain4j)

The backend uses LangChain4j to format the system instructions, the historical context, and the new user query into a single structured prompt.

#### Inference Execution (Ollama + GPU)

Ktor sends the prompt to the Ollama container via internal Docker networking (http://ollama_engine:11434). The NVIDIA Container Toolkit translates this request to the physical GPU. The LLM processes the tokens in VRAM.

#### Streaming Response

Ollama streams the generated tokens back to Ktor. Ktor simultaneously streams these tokens back through Nginx to the client, providing a real-time typing effect.

#### Persistence

Once the generation is complete, Ktor asynchronously saves the final prompt and response pair to the PostgreSQL database for future context.

```
[ External Internet ]
         │
         ▼
[ Nginx (Host: 80/443) ]
         │
         ├─► (Subdirectory /admin/db) ──► [ pgAdmin (Port: 80) ]
         │
         ▼
[ Docker Network: global_network ]
         │
         ├─► [ Ktor Backend (ai_server) ] ───► [ PostgreSQL (postgres-db) ]
         │          │
         │          ▼
         │   [ LangChain4j ]
         │          │
         ▼          ▼
[ Ollama Engine (ollama_engine) ]
         │
         ▼
[ NVIDIA GPU (Hardware via Toolkit) ]
```