<p align="center">
  <img src="docs/assets/mcp-logo.png" alt="MCP Logo" width="200"/>
</p>

<h1 align="center">MCP — Managed Control Platform</h1>

<p align="center">
  <strong>AI-Powered IT Operations Center for Managed Service Providers</strong><br>
  <em>Local. Intelligent. Automated.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-7.0-blue?style=flat-square" alt="Version"/>
  <img src="https://img.shields.io/badge/status-Beta-orange?style=flat-square" alt="Status"/>
  <img src="https://img.shields.io/badge/containers-35-green?style=flat-square" alt="Containers"/>
  <img src="https://img.shields.io/badge/stacks-5-teal?style=flat-square" alt="Stacks"/>
  <img src="https://img.shields.io/badge/networks-5-purple?style=flat-square" alt="Networks"/>
  <img src="https://img.shields.io/badge/AI-Ollama%20+%20LangChain-blueviolet?style=flat-square" alt="AI"/>
  <img src="https://img.shields.io/badge/os-Debian%2012%20%7C%20Ubuntu%2024.04-red?style=flat-square" alt="OS"/>
  <img src="https://img.shields.io/badge/license-Proprietary-lightgrey?style=flat-square" alt="License"/>
</p>

<p align="center">
  🇬🇧 <strong>English</strong> · 🇩🇪 <a href="README_de.md">Deutsch</a> · 🇸🇦 <a href="README_ar.md">العربية</a>
</p>

> **Project Status:** MCP v7 is in **Beta**. Infrastructure, monitoring, and AI pipeline foundations are functional and actively validated in lab environments. Advanced AI features (full RAG pipeline, smart ticketing, predictive ops) are under active development. See [Development Status](#development-status) for details.

---

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Architecture](#architecture)
  - [Stack Overview](#stack-overview)
  - [Network Segmentation](#network-segmentation)
  - [Data Flow](#data-flow)
  - [Unified Dashboard](#unified-dashboard)
- [AI Pipeline](#ai-pipeline)
  - [AI Stack](#ai-stack)
  - [RAG Pipeline](#rag-pipeline)
  - [Smart Ticketing](#smart-ticketing)
  - [AI Workflows (n8n)](#ai-workflows-n8n)
- [Container Infrastructure](#container-infrastructure)
- [Security](#security)
- [Installation](#installation)
  - [Hardware Requirements](#hardware-requirements)
  - [Quick Start](#quick-start)
  - [Installation Phases](#installation-phases)
- [Operations](#operations)
  - [Makefile Reference](#makefile-reference)
  - [Backup & Restore](#backup--restore)
  - [Testing](#testing)
- [Development Status](#development-status)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [Licensing](#licensing)

---

## Overview

**MCP (Managed Control Platform)** is a fully self-hosted, AI-powered IT operations center designed for managed service providers (MSPs) and IT teams. It combines industry-standard monitoring, ticketing, and documentation tools with a local AI stack that analyzes events, generates professional tickets, and learns from resolved incidents — all running **100% on-premise** with zero cloud dependencies.

MCP v7 orchestrates **35 Docker containers** (34 long-running + 1 init) across **5 isolated networks**, organized into 5 functional stacks behind a single Nginx reverse proxy.

```
                       ┌──────────────────────────────────────────┐
                       │           http://<SERVER-IP>              │
                       ├──────────────────────────────────────────┤
  Single .env    ────→ │  Nginx Reverse Proxy (13 Service Paths)  │
  One Install Script   │                                          │
  6-Phase Gate System  │   Core · Ops · Telemetry · Remote · AI   │
                       └──────────────────────────────────────────┘
```

---

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **100% Offline** | No external API calls, no cloud, no telemetry — Ollama replaces OpenAI |
| **AI-First** | Every signal (log, metric, alert) flows through the local AI pipeline |
| **Secure-by-Design** | 5 isolated networks, Keycloak SSO + MFA, CrowdSec IDS, no-new-privileges |
| **Human-in-the-Loop** | AI recommends, humans decide — configurable confidence thresholds |
| **One Command** | `sudo bash scripts/mcp-install.sh` — install, validate, and run |

---

## Architecture

### Stack Overview

| Stack | # | Containers | Purpose |
|-------|---|-----------|---------|
| **Core** | 8 | PostgreSQL, Redis, pgvector, OpenBao, Nginx, Keycloak, n8n, ntfy | Databases, identity, proxy, automation, notifications |
| **Ops** | 8+1 | Zammad (3×), Elasticsearch, Memcached, BookStack + MariaDB, Vaultwarden, Portainer, DIUN | Ticketing, search, wiki, passwords, Docker management |
| **Telemetry** | 8 | Zabbix (2×), Grafana + Renderer, Loki, Alloy, Uptime Kuma, CrowdSec | Monitoring, dashboards, logging, uptime, IDS |
| **Remote** | 3 | MeshCentral, Guacamole, guacd | Agent-based & browser-based remote access |
| **AI** | 5 | Ollama, LiteLLM, LangChain Worker, AI Gateway, Redis Queue | LLM inference, model routing, RAG pipeline, job queue |
| | **35** | *(34 long-running + 1 init)* | |

### Network Segmentation

MCP uses 5 isolated Docker bridge networks for defense-in-depth. No container has direct external port access — all traffic routes through the Nginx reverse proxy.

```
┌─────────────────────────────────────────────────────────────┐
│                    HOST MACHINE (LAN)                         │
│                                                              │
│  ┌─ mcp-edge-net (172.20.0.0/24) ──────────────────────┐   │
│  │  Nginx → Port 80/443 (only external-facing network)   │   │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌─ mcp-app-net (172.20.1.0/24) ──────────────────────┐    │
│  │  All application containers (internal comms)         │    │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌─ mcp-data-net (172.20.2.0/24) ────────────────────┐     │
│  │  PostgreSQL, Redis, pgvector, Elasticsearch         │     │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌─ mcp-sec-net (172.20.3.0/24) ─────────────────────┐     │
│  │  OpenBao (secrets), CrowdSec (IDS)                  │     │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌─ mcp-ai-net (172.20.4.0/24) ─────────────────────┐      │
│  │  Ollama (GPU), LiteLLM, LangChain, AI Gateway      │      │
│  │  ⚠ NEVER expose to public internet                  │      │
│  └────────────────────────────────────────────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                       SIGNAL SOURCES                              │
│                                                                   │
│  Printers ──┐                                                     │
│  Servers ───┤                                                     │
│  DNS/DHCP ──┤     ┌──────────┐     ┌──────────┐                  │
│  Security ──┼────→│  Zabbix  │────→│   n8n    │                  │
│  Logs ──────┤     │ (Monitor)│     │(Workflow)│                  │
│  Alerts ────┘     └──────────┘     └────┬─────┘                  │
│  CrowdSec ──────────────┘               │                        │
│                                         ▼                        │
│                  ┌─────────────┐  ┌───────────┐                  │
│                  │   Grafana   │  │AI Pipeline │                  │
│                  │   + Loki    │  │(LangChain) │                  │
│                  │ (Dashboard) │  └─────┬─────┘                  │
│                  └─────────────┘        │                        │
│                                  ┌─────┴──────┐                 │
│                                  │ redis-queue │                 │
│                                  │ (Job Queue) │                 │
│                                  └─────┬──────┘                 │
│                                        ▼                        │
│                            ┌─────────────────┐                  │
│                            │  Ollama (Local   │                  │
│                            │  LLM / GPU)      │                  │
│                            └────────┬────────┘                  │
│                                     │                           │
│                  ┌─────────────┬────┴────┬────────┐             │
│                  ▼             ▼         ▼        ▼             │
│            ┌──────────┐ ┌────────┐ ┌───────┐ ┌──────┐          │
│            │  Zammad  │ │pgvector│ │ ntfy  │ │ Wiki │          │
│            │(Tickets) │ │ (RAG)  │ │(Push) │ │(Docs)│          │
│            └──────────┘ └────────┘ └───────┘ └──────┘          │
└──────────────────────────────────────────────────────────────────┘
```

### Unified Dashboard

All services are accessible through the Nginx reverse proxy under a single IP:

```
http://<IP>/              Dashboard (landing page)
http://<IP>/auth          Keycloak (Identity & SSO)
http://<IP>/auto          n8n (Workflow Automation)
http://<IP>/notify        ntfy (Push Notifications)
http://<IP>/tickets       Zammad (Helpdesk & Ticketing)
http://<IP>/wiki          BookStack (Knowledge Base)
http://<IP>/vault         Vaultwarden (Password Manager)
http://<IP>/manage        Portainer (Docker Management)
http://<IP>/monitor       Zabbix (Infrastructure Monitoring)
http://<IP>/dash          Grafana (Dashboards & Logs)
http://<IP>/status        Uptime Kuma (Availability Monitoring)
http://<IP>/mesh          MeshCentral (Remote Desktop)
http://<IP>/remote        Guacamole (Browser-based RDP/SSH/VNC)
```

---

## AI Pipeline

### AI Stack

The AI stack runs entirely on-premise. No data leaves the server.

| Component | Role | Technology |
|-----------|------|-----------|
| **Ollama** | LLM inference engine (GPU-accelerated) | Mistral 7B / Llama 3 8B, Q4_K_M quantization |
| **LiteLLM** | Model router with OpenAI-compatible API | Failover, load balancing, unified endpoint |
| **LangChain Worker** | RAG pipeline & analysis orchestration | Document retrieval, context injection, structured output |
| **AI Gateway** | Internal REST API (FastAPI) | Job queue, deduplication, embedding, metrics |
| **Redis Queue** | Dedicated job queue for AI workloads | Isolated from main Redis cache |
| **pgvector** | Vector database for RAG knowledge base | Embedding storage & similarity search |

> **Hardware Note:** The AI stack requires a minimum of 32 GB RAM. GPU (NVIDIA with 12+ GB VRAM) is recommended for production-speed inference (~60 tok/s vs ~15 tok/s CPU-only).

### RAG Pipeline

The RAG (Retrieval Augmented Generation) pipeline enriches every AI analysis with historical context:

```
1. EVENT INTAKE        →  Zabbix trigger, Loki alert, CrowdSec finding, or manual request
2. CONTEXT ENRICHMENT  →  RAG search in pgvector for similar past incidents
3. DATA AGGREGATION    →  Current metrics from Zabbix + Loki + container status
4. LLM ANALYSIS        →  Local model analyzes event + context + metrics
5. TICKET GENERATION   →  Structured ticket with root-cause analysis & recommendations
6. NOTIFICATION        →  Push alert via ntfy to responsible team
7. FEEDBACK LOOP       →  Resolved tickets flow back into the knowledge base
```

### Smart Ticketing

AI-generated tickets follow a structured format optimized for MSP workflows:

| Section | Content | Data Source |
|---------|---------|-------------|
| Error Description | What happened — technical summary | Zabbix triggers + Loki logs |
| Root-Cause Analysis | WHY it happened — causal chain | AI correlation across all sources |
| Impact Analysis | Which services/clients are affected | Service map + Zabbix dependencies |
| Historical Context | Has this happened before? What helped? | pgvector RAG over past tickets |
| Recommended Actions | Immediate fix + long-term prevention | AI + knowledge base + best practices |
| Confidence Score | AI certainty level | Model self-assessment |

**Confidence-based routing:**

| Confidence | AI Action | Human Action |
|------------|-----------|-------------|
| **High** (75–100%) | Create ticket + assign | Gets notified |
| **Medium** (50–74%) | Create draft ticket | Must review & approve |
| **Low** (< 50%) | Alert only, no ticket | Takes over completely |

### AI Workflows (n8n)

All AI workflows run via n8n using the AI Gateway as central interface:

| # | Workflow | Trigger | Output |
|---|----------|---------|--------|
| W1 | Alert → AI Analysis | Zabbix webhook | Zammad ticket with AI analysis |
| W2 | Log Anomaly → Ticket | Loki alert rule | Ticket + Grafana annotation |
| W3 | Security Finding → Ticket | CrowdSec / Trivy | Priority security ticket |
| W4 | Ticket Close → Knowledge | Zammad webhook | BookStack article + pgvector embedding |
| W5 | BookStack → RAG Ingestion | BookStack webhook | Automatic knowledge base update |
| W6 | Predictive Alert | Cron (every 6h) | Warning ticket if risk threshold exceeded |
| W7 | Capacity Planning | Cron (weekly) | Resource trend report |
| W8 | Daily Health Report | Cron (daily 08:00) | System status summary via ntfy |

---

## Container Infrastructure

<details>
<summary><strong>Click to expand full container list (35 containers)</strong></summary>

### Core Stack — 8 Containers (#1–#8)

| # | Container | Image | Purpose |
|---|-----------|-------|---------|
| 1 | mcp-postgres | postgres:16-alpine | Central database |
| 2 | mcp-redis | redis:7-alpine | Cache & queue |
| 3 | mcp-pgvector | pgvector/pgvector:pg16 | Vector DB for RAG |
| 4 | mcp-openbao | openbao:2.1 | Secrets management |
| 5 | mcp-nginx | nginx:1.27-alpine | Reverse proxy + dashboard |
| 6 | mcp-keycloak | keycloak:26.0 | Identity & MFA (SSO) |
| 7 | mcp-n8n | n8nio/n8n:1.76.1 | Workflow automation |
| 8 | mcp-ntfy | binwiederhier/ntfy | Local push notifications |

### Ops Stack — 8+1 Containers (#9–#16 + init)

| # | Container | Image | Purpose |
|---|-----------|-------|---------|
| — | mcp-zammad-init | ghcr.io/zammad/zammad:6.4.1 | One-shot DB migration (exits after) |
| 9 | mcp-zammad-rails | ghcr.io/zammad/zammad:6.4.1 | Ticketing web UI |
| 10 | mcp-zammad-websocket | ghcr.io/zammad/zammad:6.4.1 | Real-time WebSocket |
| 11 | mcp-zammad-scheduler | ghcr.io/zammad/zammad:6.4.1 | Background jobs |
| 12 | mcp-elasticsearch | elasticsearch:8.17.0 | Full-text search for Zammad |
| 13 | mcp-zammad-memcached | memcached:1.6-alpine | Session cache |
| — | mcp-bookstack-db | mariadb:11.6 | BookStack database |
| 14 | mcp-bookstack | linuxserver/bookstack:24.12.1 | Knowledge base & wiki |
| 15 | mcp-vaultwarden | vaultwarden/server:1.32.5 | Password manager (Bitwarden-compatible) |
| 16 | mcp-portainer | portainer/portainer-ce:2.21.5 | Docker management UI |
| — | mcp-diun | crazymax/diun:4.28 | Docker image update notifications |

### Telemetry Stack — 8 Containers (#17–#24)

| # | Container | Image | Purpose |
|---|-----------|-------|---------|
| 17 | mcp-zabbix-server | zabbix-server-pgsql:7.0.0-alpine | Monitoring engine |
| 18 | mcp-zabbix-web | zabbix-web-nginx-pgsql:7.0.0-alpine | Zabbix web UI |
| 19 | mcp-grafana | grafana/grafana:11.4.0 | Dashboards & visualization |
| 20 | mcp-loki | grafana/loki:3.3.2 | Log aggregation |
| 21 | mcp-alloy | grafana/alloy:v1.5.1 | Log & metric collector |
| 22 | mcp-uptime-kuma | louislam/uptime-kuma:1 | Availability monitoring |
| 23 | mcp-crowdsec | crowdsecurity/crowdsec | Intrusion detection (IDS) |
| 24 | mcp-grafana-renderer | grafana/grafana-image-renderer | PDF/image export for reports |

### Remote Stack — 3 Containers (#25–#27)

| # | Container | Image | Purpose |
|---|-----------|-------|---------|
| 25 | mcp-meshcentral | ghcr.io/ylianst/meshcentral | Agent-based remote desktop |
| 26 | mcp-guacamole | guacamole/guacamole:1.5.5 | Browser-based remote access |
| 27 | mcp-guacd | guacamole/guacd:1.5.5 | Guacamole proxy daemon |

### AI Stack — 5 Containers (#28–#32)

| # | Container | Image | Purpose |
|---|-----------|-------|---------|
| 28 | mcp-ollama | ollama/ollama | LLM inference (GPU) |
| 29 | mcp-litellm | ghcr.io/berriai/litellm | AI gateway / model router |
| 30 | mcp-langchain | custom (Python 3.12) | AI pipeline worker |
| 31 | mcp-ai-gateway | custom (FastAPI) | Internal AI REST API |
| 32 | mcp-redis-queue | redis:7-alpine | Dedicated AI job queue |

</details>

---

## Security

MCP is built with secure-by-default principles:

| Layer | Measure |
|-------|---------|
| **Network** | 5 isolated Docker bridge networks; only `mcp-edge-net` is LAN-accessible |
| **Identity** | Keycloak SSO with MFA for all web services |
| **Secrets** | OpenBao (Vault fork) for centralized secret management |
| **Passwords** | Auto-generated, 32+ characters, stored in `.env` (chmod 600) |
| **Containers** | `no-new-privileges` on all containers, non-root users where possible |
| **IDS** | CrowdSec with local-only scenarios (offline mode, no cloud blocklists) |
| **Proxy** | All traffic through Nginx; no container has external port bindings |
| **AI** | AI Gateway is internal-only — **never** exposed to public internet |
| **Logs** | All access logged → Alloy → Loki → queryable & AI-analyzable |

> **Critical Rule:** The AI Gateway must remain internal within platform networks. Never expose it to the public internet.

---

## Installation

### Hardware Requirements

| Profile | CPU | RAM | Disk | GPU | Use Case |
|---------|-----|-----|------|-----|----------|
| **Core only** (no AI) | 4 cores | 16 GB | 60 GB NVMe | — | Monitoring, ticketing, wiki — evaluation |
| **Full stack** (min.) | 8 cores | 32 GB | 100 GB NVMe | — | All 35 containers, AI at ~15 tok/s (CPU) |
| **Recommended** | 16 cores | 64 GB | 200 GB NVMe | NVIDIA T4 (16 GB) | Comfortable headroom, AI at ~60 tok/s |
| **Optimal** | 16+ cores | 128 GB | 500 GB NVMe | NVIDIA A10 (24 GB) | Multi-tenant production workloads |

#### RAM Distribution (Full Stack)

| Area | Components | RAM |
|------|-----------|-----|
| Databases | PostgreSQL, pgvector, Redis (×2), MariaDB, Elasticsearch, Memcached | 5 GB |
| Applications | Keycloak, Zammad (3×), BookStack, n8n, Vaultwarden, ntfy | 5 GB |
| Monitoring | Zabbix (2×), Grafana (2×), Loki, Alloy, Uptime Kuma, CrowdSec | 4 GB |
| Remote | MeshCentral, Guacamole, guacd | 1 GB |
| **AI Stack** | **Ollama, LiteLLM, LangChain, AI Gateway** | **12 GB** |
| System + Overhead | OS, Docker, Portainer, DIUN, buffers | 5 GB |
| **TOTAL** | | **~32 GB** |

### Quick Start

```bash
# 1. Clone repository
git clone https://github.com/<org>/MCP.git
cd MCP

# 2. Configure environment
cp .env.example .env
chmod 600 .env
# → Replace ALL 'CHANGE_ME_' values with secure passwords

# 3. Install (6-phase gate system)
sudo bash scripts/mcp-install.sh
```

### Installation Phases

The installation script uses a **Phases + Gates** approach. Every phase has a gate check — if it fails, the script stops immediately with a detailed error log.

```
Phase 1: Preflight     →  Gate ✅/❌  →  Docker, GPU, RAM, disk, .env validation
Phase 2: Environment   →  Gate ✅/❌  →  Networks, volumes, certificates
Phase 3: Core Stack    →  Gate ✅/❌  →  PostgreSQL, Redis, Nginx, Keycloak healthy
Phase 4: Ops Stack     →  Gate ✅/❌  →  Zammad, BookStack, Vaultwarden, Portainer OK
Phase 5: Telemetry     →  Gate ✅/❌  →  Zabbix, Grafana, Loki, CrowdSec OK
Phase 6: AI Stack      →  Gate ✅/❌  →  Ollama + models, AI Gateway, end-to-end test
```

**On failure**, the script produces a detailed error log and can be resumed:

```bash
# Resume from a specific phase
sudo bash scripts/mcp-install.sh --resume-from phase4

# Run only a specific phase
sudo bash scripts/mcp-install.sh --only phase6

# Clean and reinstall
sudo bash scripts/mcp-install.sh --clean
```

**Example failure output:**

```
╔══════════════════════════════════════════════════════════════╗
║  ❌  MCP INSTALLATION STOPPED                                ║
║                                                              ║
║  Phase:     4 — Ops Stack                                    ║
║  Gate:      Zammad UI not loading                            ║
║  Cause:     Assets 404 — RAILS_SERVE_STATIC_FILES missing    ║
║                                                              ║
║  Error log: logs/mcp-install-error-20260217-143022.log       ║
║                                                              ║
║  → Fix the issue, then resume:                               ║
║    sudo bash scripts/mcp-install.sh --resume-from phase4     ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Operations

### Makefile Reference

```bash
# === Lifecycle ===
make up                  # Start all stacks (Core → Ops → Telemetry → Remote → AI)
make down                # Stop all stacks (reverse order)
make restart             # Full restart

# === Individual Stacks ===
make up-core             # Start single stack
make down-ai             # Stop single stack
make logs-telemetry      # Tail single stack logs

# === Status & Logs ===
make status              # Health check all containers
make ps                  # Running containers overview
make logs                # Tail all container logs

# === Testing ===
make test                # Run all tests (smoke + AI + security)
make test-smoke          # Service reachability
make test-ai             # AI pipeline end-to-end
make test-security       # Network isolation & security audit

# === Backup & Restore ===
make backup              # Create timestamped backup
make restore             # Interactive restore

# === Maintenance ===
make pull-images         # Pull latest Docker images
make install             # Full 6-phase installation
make clean               # ⚠ DESTRUCTIVE — remove all containers + volumes
```

### Backup & Restore

```bash
bash scripts/mcp-backup.sh       # Full backup: volumes + DB dumps + configs
bash scripts/mcp-restore.sh      # Interactive restore from backup archive
```

### Testing

| Suite | Scope | Command |
|-------|-------|---------|
| **Smoke** | All containers reachable, health checks pass | `make test-smoke` |
| **AI Pipeline** | Alert → Queue → Analysis → Response | `make test-ai` |
| **Security** | Network isolation, CrowdSec rules, port exposure | `make test-security` |

---

## Development Status

MCP v7 is in active development. This table reflects the honest implementation status:

### ✅ Fully Operational

| Component | Notes |
|-----------|-------|
| Docker Compose architecture (5 stacks, 5 networks, 35 containers) | Production-ready |
| Nginx reverse proxy with 13 service paths + dashboard | Stable |
| Health checks on all containers | Stable |
| Alloy → Loki log pipeline | Stable |
| Grafana datasource provisioning (Loki, Zabbix) | Stable |
| Security hardening (no-new-privileges, network isolation) | Stable |
| AI Gateway — job queue with Redis deduplication | Functional |
| Ollama analysis (basic alert processing) | Functional |
| 6-phase installation script with gate checks | Functional |
| Smoke / AI / Security test suites | Functional |

### 🔨 In Development

| Component | Current State | Target |
|-----------|--------------|--------|
| pgvector RAG integration | Endpoints stubbed | Full vector search with context injection |
| LangChain Worker — LangChain usage | Uses raw httpx calls | LangChain chains, templates, pgvector VectorStore |
| LiteLLM as model router | Container running, unused | Worker routes through LiteLLM with failover |
| Zammad ticket creation from AI | Described in docs | POST to Zammad API with structured ticket data |
| ntfy push from AI pipeline | Described in docs | Severity → priority mapping, channel routing |
| n8n AI workflows (W1–W8) | Workflow directory empty | Full JSON workflow definitions |
| Keycloak SSO for all services | Keycloak running standalone | SSO connected to Grafana, Zammad, BookStack |
| OpenBao secrets management | Container running standalone | Services fetch secrets from Vault instead of .env |
| BookStack → RAG ingestion | Not connected | Wiki pages auto-embedded for RAG knowledge base |
| AI pipeline Grafana dashboard | Not created | Prometheus metrics dashboard for AI operations |

> Full gap analysis and 6-phase optimization plan: [`MCP_v7_Optimierung_Plan.md`](MCP_v7_Optimierung_Plan.md)

---

## Project Structure

```
MCP/
├── .env.example                     ← Environment template (passwords, image tags)
├── Makefile                         ← Operations shortcuts (make up/down/test/backup)
│
├── compose/                         ← Docker Compose per stack
│   ├── core/docker-compose.yml      ←   #1–#8:   PostgreSQL, Redis, pgvector, OpenBao, Nginx, Keycloak, n8n, ntfy
│   ├── ops/docker-compose.yml       ←   #9–#16:  Zammad (3×), ES, BookStack+MariaDB, Vaultwarden, Portainer, DIUN
│   ├── telemetry/docker-compose.yml ←   #17–#24: Zabbix (2×), Grafana (2×), Loki, Alloy, Uptime Kuma, CrowdSec
│   ├── remote/docker-compose.yml    ←   #25–#27: MeshCentral, Guacamole, guacd
│   └── ai/docker-compose.yml        ←   #28–#32: Ollama, LiteLLM, LangChain, AI Gateway, Redis Queue
│
├── config/                          ← Service configurations
│   ├── ai/                          ←   LLM models (models.yml), RAG config, prompt templates
│   ├── alloy/                       ←   Log collector config
│   ├── crowdsec/                    ←   IDS acquisition rules & local scenarios
│   ├── grafana/                     ←   Datasources, dashboard provisioning
│   ├── keycloak/                    ←   Realm export
│   ├── loki/                        ←   Log aggregation config
│   ├── n8n/                         ←   Workflow definitions (W1–W8)
│   ├── nginx/                       ←   Reverse proxy (13 paths), dashboard HTML
│   ├── ntfy/                        ←   Push notification topics & auth
│   └── portainer/                   ←   Container management persistence
│
├── containers/                      ← Custom container builds (multi-stage)
│   ├── ai-gateway/                  ←   FastAPI REST API + Dockerfile
│   └── langchain-worker/            ←   RAG pipeline worker + Dockerfile
│
├── scripts/                         ← Operations scripts
│   ├── mcp-install.sh               ←   6-phase installation with gate checks
│   ├── mcp-start.sh / mcp-stop.sh   ←   Ordered startup / shutdown
│   ├── mcp-status.sh                ←   Health check all containers
│   ├── mcp-backup.sh                ←   Full backup (volumes + DBs)
│   ├── mcp-restore.sh               ←   Interactive restore
│   ├── mcp-pull-images.sh           ←   Pull all Docker images
│   ├── init-db.sh                   ←   PostgreSQL multi-DB initialization
│   ├── init-pgvector.sh             ←   pgvector extension & table setup
│   └── gen-test-env.sh              ←   Generate test environment
│
├── tests/                           ← Automated test suites
│   ├── smoke-test.sh                ←   Service reachability (all containers)
│   ├── ai-pipeline-test.sh          ←   AI end-to-end validation
│   └── security-test.sh             ←   Network isolation & security audit
│
├── docs/                            ← Documentation & assets
├── images/                          ← Exported Docker images (offline deployment)
├── models/                          ← Exported Ollama models (offline deployment)
└── logs/                            ← Installation & error logs
```

---

## Technology Stack

| Category | Technologies |
|----------|-------------|
| **Databases** | PostgreSQL 16, Redis 7, pgvector, MariaDB 11.6, Elasticsearch 8.17, Memcached |
| **Identity** | Keycloak 26 (SSO + MFA) |
| **Monitoring** | Zabbix 7.0, Grafana 11.4, Loki 3.3, Alloy v1.5, Uptime Kuma |
| **Security** | CrowdSec (IDS), OpenBao 2.1 (secrets), Vaultwarden 1.32 (passwords) |
| **Helpdesk** | Zammad 6.4 |
| **Automation** | n8n 1.76, ntfy (push notifications) |
| **Wiki** | BookStack 24.12 |
| **Remote** | MeshCentral, Apache Guacamole 1.5 |
| **AI** | Ollama (Mistral 7B / Llama 3 8B), LiteLLM, LangChain, pgvector |
| **Infrastructure** | Docker Compose v2, Nginx 1.27, Portainer 2.21, DIUN 4.28 |

---

## Offline Deployment

MCP is designed for air-gapped environments with zero internet dependency at runtime:

```bash
# On a machine WITH internet:
bash scripts/mcp-pull-images.sh                          # Pull all 35 images
docker save $(docker images -q) | gzip > images/mcp-images-v7.tar.gz

# Transfer to target machine, then:
docker load < images/mcp-images-v7.tar.gz                # Import images
sudo bash scripts/mcp-install.sh                          # Install offline
```

---

## Roadmap

| Version | Feature | Timeline |
|---------|---------|----------|
| **v7.0** | Infrastructure + basic AI pipeline | ✅ Current |
| **v7.1** | Full RAG pipeline + smart ticketing + n8n workflows | In Progress |
| **v7.2** | AI auto-remediation (restart, scale, cleanup) | Q2 2026 |
| **v7.3** | Customer portal with AI status updates | Q3 2026 |
| **v8.0** | Multi-node (AI on dedicated GPU server) | Q4 2026 |
| **v8.1** | Fine-tuning LLM on own ticket data | Q1 2027 |

---

## Licensing

**MCP is proprietary software.** Copyright © Masdor. All rights reserved.

| Usage | Terms |
|-------|-------|
| **Evaluation** (Lab/Test) | Permitted with written approval only |
| **Commercial License** | Individual license agreement required |
| **Production** | Expressly prohibited without a valid license |

Third-party components remain subject to their respective open-source licenses.

For licensing inquiries: **contact@masdor.de**
For security disclosures: **security@masdor.de**

---

## Disclaimer

MCP is a technical platform that must be validated in a lab environment before production deployment. Documentation provides operational guidance but does not constitute legal or compliance advice.

---

<p align="center">
  <strong>MCP v7 — AI-Powered IT Operations Center</strong><br>
  <em>Local. Intelligent. Automated.</em><br><br>
  <code>sudo bash scripts/mcp-install.sh</code>
</p>
