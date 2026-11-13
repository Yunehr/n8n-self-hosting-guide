# Complete Step-by-Step Guide to Self-Hosting n8n Locally with Local AI Agents

Building a completely self-hosted, zero-subscription automation ecosystem allows you to run advanced workflows and artificial intelligence models on your own hardware without ongoing SaaS fees, API rate limits, or data privacy risks. By combining n8n for workflow orchestration, Ollama for local Large Language Model (LLM) inference, PostgreSQL for execution state persistence, and Cloudflare Tunnel for secure external webhooks, you can establish an enterprise-grade platform running locally on standard consumer hardware or dedicated home servers.

This guide provides a comprehensive, production-tested walkthrough to take any fresh system from scratch to a fully functional self-hosted n8n environment equipped with a local AI email inbox analysis agent.

---

## 1. System Architecture & Component Overview

The local automation stack relies on containerized microservices managed via Docker Compose. Every service runs within an isolated Docker network, exposing only the necessary interface ports to the host machine.

### Stack Architecture

* **n8n (Workflow Orchestrator)**: Connects triggers, data transformations, API endpoints, and AI nodes into visual automation pipelines.
* **Ollama (Local AI Engine)**: Hosts open-source LLM models (such as `llama3.2` or `qwen2.5`) locally on CPU or GPU hardware, serving an OpenAI-compatible REST API on port `11434`.
* **PostgreSQL (Database Backend)**: Stores workflow definitions, credential keys, execution history, and user settings reliably, replacing the default lightweight SQLite engine.
* **Cloudflare Tunnel (`cloudflared`)**: Creates an encrypted, outbound-only QUIC tunnel from your container stack to Cloudflare's edge network, exposing n8n webhooks over HTTPS without requiring port forwarding, static public IP addresses, or router modifications.

| Service Component | Container Name | Internal Port | External Exposure | Primary Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **n8n** | `n8n` | `5678` | `localhost:5678` | Visual workflow engine & user interface |
| **Ollama** | `ollama` | `11434` | `localhost:11434` | Local LLM inference server |
| **PostgreSQL** | `postgres` | `5432` | Internal network only | Production database for n8n application state |
| **Cloudflare Tunnel** | `cloudflared` | N/A | Outbound QUIC | Ingress routing for external webhooks |

---

## 2. Phase 1: Proof of Concept — Core Infrastructure Deployment

The initial proof-of-concept phase sets up the base container stack on your primary machine (Windows via WSL2 or native Linux).

### System Prerequisites
* **Windows**: Install Docker Desktop with the WSL2 backend enabled, running Ubuntu 22.04 or 24.04 LTS.
* **Linux**: Install Docker Engine (v27+) and the Docker Compose plugin (`docker-compose-plugin`).
* **Hardware Allocation**: Minimum 4-core CPU, 8GB RAM (16GB recommended for LLMs), and 20GB free disk space.

> **Critical File Path Rule for Windows WSL2 Users**: Always create your project directory inside the native Linux filesystem (e.g., `~/n8n-stack`) rather than mounted Windows drives (`/mnt/c/Users/...`). Binding mounts across the Windows/Linux host boundary causes severe disk I/O slowdowns and volume permission conflicts.

### Step 1: Prepare the Project Directory and Environment Configuration

Open your Linux terminal (or WSL2 shell) and execute the following commands to set up the project folder:

```bash
mkdir -p ~/n8n-stack && cd ~/n8n-stack
```

### Step 1: Prepare the Project Directory and Environment Configuration

Open your Linux terminal (or WSL2 shell) and execute the following commands to set up the project folder:

```
mkdir -p ~/n8n-stack &amp;&amp; cd ~/n8n-stack

```

#### How to Create and Open the `.env` File

You can create and open the `.env` file using terminal commands or Graphical User Interface (GUI):

* **Option A — Terminal Editor (Nano / VS Code)**:  
```  
# Open with terminal text editor  
nano .env  
# Or open directly in VS Code (if installed in WSL2/Linux)  
code .env  
```
* **Option B — Windows File Explorer (GUI)**: From your WSL2 terminal inside `~/n8n-stack`, open Windows File Explorer directly into the Linux folder by running:  
```  
explorer.exe .  
```  
In File Explorer, right-click to create a new text file named `.env` and edit it with Notepad or your preferred Windows text editor.
* **Option C — One-Line Terminal Command (Automated)**: Paste the following block directly into your terminal to write the file automatically:

```
cat &lt;&lt; 'EOF' &gt; ~/.n8n-stack/.env
# General Settings
GENERIC_TIMEZONE=UTC
TZ=UTC

# Database Configuration
POSTGRES_USER=n8n_db_user
POSTGRES_PASSWORD=your_secure_postgres_password_here
POSTGRES_DB=n8n_database

# n8n Instance Configuration
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true

# Security Encryption Key (Generate a random 32-character hex string)
N8N_ENCRYPTION_KEY=your_random_32_character_encryption_key
EOF

```

If creating the file manually via Option A or B, paste the following configuration into your `.env` file and save:

```
# General Settings
GENERIC_TIMEZONE=UTC
TZ=UTC

# Database Configuration
POSTGRES_USER=n8n_db_user
POSTGRES_PASSWORD=your_secure_postgres_password_here
POSTGRES_DB=n8n_database

# n8n Instance Configuration
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true

# Security Encryption Key (Generate a random 32-character hex string)
N8N_ENCRYPTION_KEY=your_random_32_character_encryption_key

```

### Step 2: Create the `docker-compose.yml` File

Next, create the primary container orchestration configuration file `docker-compose.yml` inside `~/n8n-stack`.

#### How to Create and Open `docker-compose.yml`

* **Terminal Editor**: Run `nano docker-compose.yml` or `code docker-compose.yml` in your terminal.
* **GUI File Explorer**: Run `explorer.exe .` in WSL2, right-click inside the `n8n-stack` folder, select **New Text Document**, rename it to `docker-compose.yml` (ensuring the `.txt` extension is removed), and open it in your editor.

Paste the following container definition into `docker-compose.yml` and save:

```yaml
version: '3.8'

networks:
  n8n-network:
    driver: bridge

volumes:
  n8n_data:
  ollama_data:
  postgres_data:

services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    networks:
      - n8n-network
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - PGDATA=/var/lib/postgresql/data
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    networks:
      - n8n-network
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama

  n8n:
    image: docker.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    networks:
      - n8n-network
    ports:
      - "5678:5678"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=${N8N_PORT}
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=${N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
```

### Step 3: Launch and Verify the Container Stack

Start the microservice stack in detached mode:

```bash
docker compose up -d
```

Verify that all three containers are healthy and running:

```bash
docker compose ps
```

You can verify n8n health by making an HTTP request to its readiness endpoint:

```bash
curl -f http://localhost:5678/healthz/readiness
```

Open `http://localhost:5678` in your browser to complete the initial owner account creation wizard.

---

## 3. Phase 2: Proof of Concept — Local AI Model Engine Integration

With n8n and Ollama running on the same Docker bridge network (`n8n-network`), n8n can communicate directly with Ollama using the internal container hostname `http://ollama:11434`.

### Step 1: Pull Local LLM Models

Execute a command directly inside the running `ollama` container to pull lightweight, high-performance instruction models:

```bash
# Pull Meta's lightweight 3B parameter model (optimized for speed and low RAM)
docker exec -it ollama ollama pull llama3.2

# Alternatively, pull Qwen 2.5 7B for superior structured JSON extraction
docker exec -it ollama ollama pull qwen2.5:7b
```

Verify that the model is loaded into local storage:

```bash
docker exec -it ollama ollama list
```

### Step 2: Configure Ollama Credentials in n8n UI

1. Log into your n8n canvas at `http://localhost:5678`.
2. Navigate to **Credentials** in the left menu and click **Add Credential**.
3. Search for and select **Ollama**.
4. In the **Base URL** field, set:
   ```text
   http://ollama:11434
   ```
   *(Note: Do NOT use `http://localhost:11434` inside the n8n UI credential box, as `localhost` inside a container points to the n8n container itself rather than the Ollama service).*
5. Click **Save** to verify the connection.

### Step 3: Prompt Engineering & JSON Formatting Guidelines for Local Models

Smaller local LLMs (3B to 8B parameters) can occasionally wrap JSON outputs in conversational text (e.g., *"Here is your summary:"*), which causes downstream n8n nodes to fail when parsing data. 

To ensure 100% deterministic JSON output:
1. Always enable `format: "json"` mode when using raw HTTP Request nodes or Ollama Chat Model configurations.
2. Structure system prompts with explicit schema definitions and negative constraints.

#### Recommended System Prompt Structure:
```text
You are an automated email triage system. Analyze the raw email provided by the user.
Your response MUST be a single, valid JSON object matching this strict schema:

{
  "sender": "string",
  "subject": "string",
  "category": "Billing" | "Technical Support" | "Sales Inquiry" | "General Spam",
  "urgency_score": number (1 to 5),
  "summary": "string (2-3 concise sentences)",
  "action_required": boolean,
  "recommended_next_step": "string"
}

Do NOT include any conversational introduction, Markdown code block fences, or trailing explanations. Return ONLY the raw JSON object. If a field cannot be determined, set its value to null. Never invent facts.
```

---

## 4. Phase 3: Proof of Concept — Webhook Ingress & Email Analysis Agent Workflow

To allow external services or apps to send emails/webhooks to your local n8n instance, we will configure a Cloudflare Tunnel and assemble the complete Email Analysis AI Agent workflow.

### Step 1: Set Up Cloudflare Tunnel (`cloudflared`) for Public Ingress

Cloudflare Zero Trust offers a completely free tunnel service that routes public HTTPS traffic to your local port `5678` without exposing your home IP or opening firewall ports.

1. Create a free account at Cloudflare and add a domain (or use a free Cloudflare sub-domain).
2. Go to **Zero Trust** -> **Networks** -> **Tunnels** -> **Create a Tunnel**.
3. Name your tunnel `n8n-local-tunnel` and select **Docker** as the deployment environment.
4. Copy the provided tunnel token string.
5. Under **Public Hostname Page**, configure the routing:
   * **Subdomain / Domain**: `n8n.yourdomain.com`
   * **Service Service Type**: `HTTP`
   * **URL**: `n8n:5678`
6. Add the `cloudflared` service to your `docker-compose.yml` file:

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    networks:
      - n8n-network
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

Add your token to `.env`:
```env
CLOUDFLARE_TUNNEL_TOKEN=your_cloudflare_tunnel_token_here
```

Update your `.env` file's `WEBHOOK_URL` so n8n generates external URLs matching your domain:
```env
WEBHOOK_URL=https://n8n.yourdomain.com/
```

Restart your Docker Compose stack:
```bash
docker compose up -d
```

---

### Step 2: Building the Email Inbox Analysis AI Agent Workflow

Now build the workflow in n8n. The agent accepts incoming emails (via Email Trigger or Webhook), passes the content to the local Ollama LLM, evaluates urgency and category, and routes actionable items.

#### Workflow Architecture & Routing:
* **Trigger**: Webhook Node (`POST /email-inbox`) or Email Read (IMAP) Node.
* **AI Processing**: AI Agent Node connected to Ollama Chat Model (`llama3.2`).
* **Evaluation & Routing**: IF / Switch Node checking if `urgency_score >= 4` or `action_required == true`.
* **Outputs**: High Urgency Branch (Alert/Notification) vs Standard Branch (Data Table / Audit Log).

#### Detailed Node Configuration Steps:

1. **Trigger Node**:
   * Add a **Webhook Node** set to HTTP Method `POST` and Path `/email-inbox`. (Alternatively, connect an **Email Read (IMAP)** node to poll an inbox directly).
2. **AI Agent Node**:
   * Add an **AI Agent** node to the canvas.
   * Attach a **Chat Model Sub-Node** -> Select **Ollama Chat Model**.
   * In the Ollama sub-node, select your Ollama Credential (`http://ollama:11434`) and set Model Name to `llama3.2`.
3. **Prompt Configuration**:
   * Connect the incoming Trigger JSON (`$json.body`) to the AI Agent user prompt:
     ```text
     Process the following raw email payload and return the structured JSON analysis according to your system prompt:

     From: {{ $json.body.from }}
     Subject: {{ $json.body.subject }}
     Date: {{ $json.body.date }}
     Body:
     {{ $json.body.text }}
     ```
4. **Data Transformation & Routing**:
   * Add an **Edit Fields (Set)** / **Code** node to parse the model's response string into native n8n JSON fields.
   * Add an **IF Node** checking `{{ $json.urgency_score }} >= 4`.
   * **True Branch**: Send a high-priority alert (via Discord webhook, Telegram, or desktop notification).
   * **False Branch**: Save execution summary to a local n8n Data Table.

---

## 5. Phase 4: Server Migration & Raspberry Pi 5 Optimization

When transitioning this proof of concept from your primary computer to a dedicated personal server or Raspberry Pi 5, specific ARM64 hardware optimizations and kernel configurations must be applied.

### Hardware Best Practices for Raspberry Pi 5
* **Storage Requirements**: Run Docker data directories on a **USB 3.0 SSD or NVMe M.2 HAT**. MicroSD cards degrade rapidly under Docker write operations, leading to container crashes and high I/O latency.
* **RAM Allocation**: An 8GB Raspberry Pi 5 is strongly recommended. 4GB models can host n8n and Postgres, but struggle when running 7B LLMs concurrently.

### Step 1: Apply the Raspberry Pi 5 Kernel Memory Fix (`cgroup`)

By default, Raspberry Pi OS disables the cgroup memory controller in the Linux kernel. This prevents Docker from applying container memory limits and causes startup warnings or crash loops.

1. Open `/boot/firmware/cmdline.txt` in a text editor with root privileges:
   ```bash
   sudo nano /boot/firmware/cmdline.txt
   ```
2. Append the following parameters to the **end of the existing single line** (do NOT create a new line):
   ```text
   cgroup_enable=memory swapaccount=1 cgroup_memory=1
   ```
3. Save the file and reboot the Pi:
   ```bash
   sudo reboot
   ```
4. After rebooting, verify that cgroup memory support is active:
   ```bash
   docker info | grep -i "Memory Limit"
   ```
   *Expected Output*: `Memory Limit: true`

### Step 2: ARM64 Architecture Container Tags

Raspberry Pi 5 runs on a 64-bit ARM architecture (`aarch64`). Standard multi-architecture Docker images (such as `n8nio/n8n:latest`, `ollama/ollama:latest`, and `postgres:16-alpine`) automatically pull the correct `linux/arm64` layers.

If you encounter container crashes returning `exec format error`, it indicates an x86-only image tag was pulled. Always verify image architecture on Docker Hub or explicitly set `platform: linux/arm64` in your `docker-compose.yml`.

### Step 3: Migrating Workflows and Credentials via CLI

When moving your proof-of-concept workflows from Windows WSL2 to your Raspberry Pi server, use the n8n Server CLI commands to export and import items cleanly.

Note: Keep workflow and credential exports in separate target directories to avoid import syntax failures.

#### On Your Source Instance (Windows WSL2):
```bash
# Export all workflows to JSON
docker exec -it n8n n8n export:workflow --backup --output=/home/node/.n8n/backups/workflows/

# Export credentials (using decryption flag or original encryption key)
docker exec -it n8n n8n export:credentials --backup --decrypted --output=/home/node/.n8n/backups/credentials/
```
Copy the JSON files from your source machine's `n8n_data/backups` folder to your Raspberry Pi server.

#### On Your Target Instance (Raspberry Pi Server):
```bash
# Import workflows
docker exec -it n8n n8n import:workflow --separate --input=/home/node/.n8n/backups/workflows/

# Import credentials
docker exec -it n8n n8n import:credentials --separate --input=/home/node/.n8n/backups/credentials/
```

Restart the n8n container on your server to activate all imported workflows and credentials:
```bash
docker compose restart n8n
```

Your zero-cost, privacy-first local automation engine is now completely operational!
