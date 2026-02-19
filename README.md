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
  <img src="https://img.shields.io/badge/AI-Ollama%20+%20LangChain%20+%20pgvector-blueviolet?style=flat-square" alt="AI"/>
  <img src="https://img.shields.io/badge/os-Debian%2012%20%7C%20Ubuntu%2024.04-red?style=flat-square" alt="OS"/>
  <img src="https://img.shields.io/badge/license-Proprietary-lightgrey?style=flat-square" alt="License"/>
</p>

<p align="center">
  🇬🇧 <strong>English</strong> · 🇩🇪 <a href="README_de.md">Deutsch</a> · 🇸🇦 <a href="README_ar.md">العربية</a>
</p>

> **Project Status:** MCP v7 is in **Beta**. The platform infrastructure, monitoring stack, and AI analysis pipeline are functional and actively validated in lab environments. Production deployment requires individual assessment and a written license agreement.

---

## Table of Contents

- [Overview](#overview)
- [What's New in v7](#whats-new-in-v7)
- [Design Principles](#design-principles)
- [Architecture](#architecture)
- [AI Pipeline](#ai-pipeline)
- [Smart Ticketing](#smart-ticketing)
- [Monitoring Scope](#monitoring-scope)
- [Container Infrastructure](#container-infrastructure)
- [Security](#security)
- [Installation](#installation)
- [Operations](#operations)
- [AI Gateway API](#ai-gateway-api)
- [Day-2 Operations](#day-2-operations)
- [Go-Live Checklist](#go-live-checklist)
- [Known Issues & Patterns](#known-issues--patterns)
- [Development Status](#development-status)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [Licensing](#licensing)

---

## Overview

**MCP (Managed Control Platform)** is a fully self-hosted, AI-powered IT operations center designed for managed service providers (MSPs) and IT teams. It transforms traditional IT operations into an intelligent, automated workflow — from monitoring and alerting to root-cause analysis and ticket generation — all running **100% on-premise** with zero cloud dependencies.

MCP v7 orchestrates **35 Docker containers** across **5 isolated networks**, organized into functional stacks behind a single Nginx reverse proxy. The local AI stack (Ollama + LangChain + pgvector) analyzes events in real-time, searches a knowledge base of past incidents, and generates professional, actionable tickets automatically.

```
 Zabbix Alert                           ┌──────────────────────┐
 CrowdSec Finding  ─→ n8n Workflow ─→  │  AI Gateway (FastAPI) │
 Loki Anomaly                           └──────────┬───────────┘
                                                    │
                                        ┌───────────▼───────────┐
                                        │  LangChain Worker     │
                                        │  1. RAG search        │
                                        │  2. Prompt + context  │
                                        │  3. LLM analysis      │
                                        │  4. Zammad ticket     │
                                        │  5. ntfy push         │
                                        │  6. Feedback → RAG    │
                                        └───────────────────────┘
```

---

## What's New in v7

MCP v7 introduces a fully local AI system that analyzes monitoring data, security events, and network anomalies in real-time.

| Area | v6 (Before) | v7 (Now) | Business Impact |
|------|-------------|----------|-----------------|
| **AI** | pgvector embeddings only | Local LLM + RAG + full analysis pipeline | Automatic fault diagnosis |
| **Tickets** | Manually created | AI writes structured tickets with root-cause | 80% less manual work |
| **Monitoring** | Separate tools, manual correlation | Unified AI correlation across all sources | Cross-tool root-cause analysis |
| **Security** | Periodic scans | CrowdSec real-time IDS + AI analysis | Instant threat assessment |
| **Knowledge** | Manual wiki entries | AI-powered feedback loop (resolved tickets → RAG) | Context from all past incidents |
| **Notifications** | Email (requires internet) | ntfy local push (offline-capable) | Zero cloud dependency |
| **Containers** | 25 | 35 (+10: AI stack, CrowdSec, Portainer, ntfy, renderer) | Fully on-premise |

---

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **100% Offline** | No external API calls, no cloud, no telemetry — Ollama replaces OpenAI |
| **AI-First** | Every signal (log, metric, alert) flows through the local AI pipeline |
| **Secure-by-Design** | 5 isolated networks, Keycloak SSO + MFA, CrowdSec IDS, `no-new-privileges` |
| **Human-in-the-Loop** | AI recommends, humans decide — configurable confidence thresholds |
| **One Command** | `sudo bash scripts/mcp-install.sh` — install, validate, and run with 6 gated phases |

---

## Architecture

### Stack Overview

| Stack | Containers | Components | Purpose |
|-------|-----------|-----------|---------|
| **Core** | 8 | PostgreSQL, Redis, pgvector, OpenBao, Nginx, Keycloak, n8n, ntfy | Databases, identity, proxy, automation, push notifications |
| **Ops** | 10 + 1 init | Zammad (3×), Elasticsearch, Memcached, BookStack + MariaDB, Vaultwarden, Portainer, DIUN | Ticketing, search, wiki, passwords, Docker management |
| **Telemetry** | 8 | Zabbix (2×), Grafana + Renderer, Loki, Alloy, Uptime Kuma, CrowdSec | Monitoring, dashboards, logging, availability, IDS |
| **Remote** | 3 | MeshCentral, Guacamole, guacd | Agent-based & browser-based remote access |
| **AI** | 5 | Ollama, LiteLLM, LangChain Worker, AI Gateway, Redis Queue | LLM inference, model routing, RAG pipeline, job queue |
| | **35 total** | *(34 long-running + 1 init)* | |

### Network Segmentation

5 isolated Docker bridge networks enforce defense-in-depth. No container has direct external port access — all traffic routes through the Nginx reverse proxy.

```
┌─────────────────────────────────────────────────────────────────┐
│  HOST (LAN)                                                      │
│                                                                   │
│  ┌─ mcp-edge-net ─ 172.20.0.0/24 ─────────────────────────┐    │
│  │  Nginx (only network reachable from LAN, ports 80/443)   │    │
│  └─────────────────────────┬────────────────────────────────┘    │
│                             │                                     │
│  ┌─ mcp-app-net ── 172.20.1.0/24 ─────────────────────────┐    │
│  │  Keycloak, n8n, Zammad, BookStack, Vaultwarden,          │    │
│  │  Portainer, Uptime Kuma, ntfy, MeshCentral, Guacamole    │    │
│  └─────────────────────────┬────────────────────────────────┘    │
│                             │                                     │
│  ┌─ mcp-data-net ─ 172.20.2.0/24 ─────────────────────────┐    │
│  │  PostgreSQL, Redis, pgvector, Elasticsearch, MariaDB      │    │
│  └─────────────────────────┬────────────────────────────────┘    │
│                             │                                     │
│  ┌─ mcp-sec-net ── 172.20.3.0/24 ─────────────────────────┐    │
│  │  OpenBao (Secrets), CrowdSec (IDS ↔ Nginx Bouncer)       │    │
│  └─────────────────────────┬────────────────────────────────┘    │
│                             │                                     │
│  ┌─ mcp-ai-net ── 172.20.4.0/24 ──────────────────────────┐    │
│  │  Ollama (GPU), LiteLLM, LangChain Worker, AI Gateway,    │    │
│  │  Redis Queue — ⚠ NEVER EXPOSE TO PUBLIC INTERNET          │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                       SIGNAL SOURCES                              │
│                                                                   │
│  🖨️ Printers ──┐                                                 │
│  🖥️ Servers ───┤     ┌──────────┐     ┌──────────┐              │
│  🌐 DNS/DHCP ──┼────→│  Zabbix  │────→│   n8n    │              │
│  📊 Logs ──────┤     │ (Monitor)│     │(Workflow)│              │
│  🔔 Alerts ────┘     └──────────┘     └────┬─────┘              │
│  🛡️ CrowdSec ──────────────┘               │                    │
│                                             ▼                    │
│                    ┌─────────────┐   ┌────────────┐              │
│                    │   Grafana   │   │ AI Gateway  │              │
│                    │   + Loki    │   │  (FastAPI)  │              │
│                    │ (Dashboard) │   └──────┬─────┘              │
│                    └─────────────┘          │                    │
│                                      ┌─────┴──────┐             │
│                                      │ Redis Queue │             │
│                                      └─────┬──────┘             │
│                                            ▼                    │
│                                 ┌────────────────────┐          │
│                                 │  LangChain Worker   │          │
│                                 │  ① RAG → pgvector   │          │
│                                 │  ② Prompt Template  │          │
│                                 │  ③ LiteLLM → Ollama │          │
│                                 └─────────┬──────────┘          │
│                                           │                     │
│                    ┌──────────┬───────────┼──────────┐          │
│                    ▼          ▼           ▼          ▼          │
│              ┌──────────┐ ┌────────┐ ┌───────┐ ┌────────┐      │
│              │  Zammad  │ │pgvector│ │ ntfy  │ │BookStack│      │
│              │(Ticket)  │ │(→ RAG) │ │(Push) │ │ (Wiki) │      │
│              └──────────┘ └────────┘ └───────┘ └────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

### Unified Dashboard

All 13 services are accessible through Nginx under a single IP:

| Path | Service | Function |
|------|---------|----------|
| `/` | **Dashboard** | Landing page with service overview |
| `/auth` | **Keycloak** | Identity & SSO with MFA |
| `/auto` | **n8n** | Workflow automation |
| `/notify` | **ntfy** | Push notifications |
| `/tickets` | **Zammad** | Helpdesk & ticketing |
| `/wiki` | **BookStack** | Knowledge base |
| `/vault` | **Vaultwarden** | Password manager |
| `/manage` | **Portainer** | Docker management UI |
| `/monitor` | **Zabbix** | Infrastructure monitoring |
| `/dash` | **Grafana** | Dashboards & log explorer |
| `/status` | **Uptime Kuma** | Availability monitoring |
| `/mesh` | **MeshCentral** | Remote desktop (agent-based) |
| `/remote` | **Guacamole** | Browser-based RDP/SSH/VNC |

---

## AI Pipeline

### Components

| Component | Role | Technology |
|-----------|------|-----------|
| **Ollama** | LLM inference engine (GPU-accelerated) | Mistral 7B Instruct Q4_K_M, Llama 3 8B (fallback) |
| **LiteLLM** | Model router with OpenAI-compatible API | Failover routing, unified endpoint |
| **LangChain Worker** | Full analysis pipeline orchestrator | LangChain 0.3 + pgvector VectorStore |
| **AI Gateway** | Internal REST API (FastAPI) | Job queue, deduplication, embedding, search, metrics |
| **Redis Queue** | Dedicated AI job queue | Isolated from main Redis cache (`mcp-redis-queue`) |
| **pgvector** | Vector database for RAG knowledge base | 768-dim embeddings, similarity search, audit log |
| **nomic-embed-text** | Embedding model | Vectorization for RAG knowledge base |

### Pipeline Flow (per Job)

The LangChain Worker executes a complete 7-step pipeline for each alert:

```
1. JOB INTAKE         Redis BRPOP from mcp:queue:analyze
                       ↓
2. RAG SEARCH         Embed description → pgvector similarity search (top_k=5, threshold=0.7)
                       → Returns similar past incidents from last 90 days
                       ↓
3. PROMPT ASSEMBLY    Load template from config/ai/prompts/alert-analysis.txt
                       → Fill: {alert_type}, {source}, {hostname}, {severity}, {description},
                         {metrics}, {logs}, {crowdsec_alerts}, {rag_results}
                       ↓
4. LLM ANALYSIS       LiteLLM (http://litellm:4000) → OpenAI-compatible API
                       → Fallback: Ollama direct (http://ollama:11434)
                       → Returns structured JSON: root_cause, impact, actions, confidence
                       ↓
5. TICKET CREATION    If severity ≥ high OR confidence = High:
                       → POST http://zammad-rails:3000/api/v1/tickets
                       → Structured ticket with AI analysis, priority, recommended actions
                       ↓
6. NOTIFICATION       ntfy push to mcp-alerts channel
                       → Severity → Priority mapping (Critical=5, High=4, Medium=3, Low=2)
                       ↓
7. FEEDBACK LOOP      Embed analysis summary → store in pgvector
                       → Future RAG searches will find this incident
                       → Audit log entry with confidence score, model used, processing time
```

### AI Configuration

```yaml
# config/ai/models.yml — Model routing via LiteLLM
primary_model:   mistral:7b-instruct-v0.3-q4_K_M     # Analysis, tickets, root-cause
fallback_model:  llama3:8b-instruct-q4_K_M            # Automatic failover
embedding_model: nomic-embed-text                      # 768-dim vectors for RAG

# config/ai/rag-config.yml — RAG pipeline tuning
chunk_size: 512          # Document chunking for ingestion
chunk_overlap: 50        # Overlap between chunks
top_k: 5                 # Similar documents per query
similarity_threshold: 0.7  # Minimum cosine similarity
max_age_days: 90         # RAG search window
max_concurrent_jobs: 3   # Parallel analysis jobs
job_timeout: 120s        # Per-job timeout
```

---

## Smart Ticketing

Every AI-generated ticket follows a structured format optimized for MSP workflows. The ticket content is produced by a German-language prompt template:

| Field | Content | Source |
|-------|---------|--------|
| `ticket_title` | Concise incident title | AI-generated |
| `root_cause` | Why it happened — 1–2 sentences | LLM correlation |
| `impact` | Gering / Mittel / Hoch / Kritisch | LLM assessment |
| `affected_services` | Which services/clients are impacted | Service map analysis |
| `immediate_action` | Concrete steps to fix now | AI + RAG knowledge base |
| `long_term_solution` | Prevention strategy | AI + best practices |
| `confidence` | High / Medium / Low with reasoning | Model self-assessment |
| `ticket_priority` | 1_low / 2_normal / 3_high / 4_urgent | Mapped from impact + confidence |

### Confidence-Based Routing

| Confidence | Severity | Action |
|------------|----------|--------|
| **High** + High/Critical severity | Automatic | Ticket created + assigned + ntfy push |
| **Medium** | Automatic | Ticket created as draft + ntfy push |
| **Low** | Alert only | ntfy notification, no ticket — human takes over |

### Example: Printer Monitoring

**Scenario:** Network printer `192.168.1.50` reports paper jam

1. **Zabbix** detects SNMP trap → status change
2. **n8n** workflow triggered → enriches alert with metrics, toner status, error history
3. **AI Gateway** receives alert → queues in Redis, deduplication check (15 min window)
4. **LangChain Worker** pops job → RAG finds 3 similar printer incidents from past 90 days
5. **Ollama/Mistral** analyzes → "Recurring paper jam in tray 2, feed roller likely worn"
6. **Zammad** ticket created: `[DRUCKER] HP LaserJet 402 — Recurring Paper Jam Tray 2`
7. **ntfy** push sent to responsible technician (priority 3 = medium)
8. **pgvector** stores analysis embedding → future similar incidents find this solution

---

## Monitoring Scope

| Area | What | How | AI Action |
|------|------|-----|-----------|
| **Printers** | Status, toner, paper jams, page count | SNMP via Zabbix | Ticket + predict toner change |
| **Servers** | CPU, RAM, disk, services, Docker health | Zabbix Agent + Alloy | Proactive ticket before failure |
| **Network** | Ping, bandwidth, packet loss | Zabbix + Uptime Kuma | Escalation on outage |
| **DNS/DHCP** | Resolution, latency, lease pool | Zabbix checks | Alert on anomaly |
| **Security** | Failed logins, port scans, cert expiry | CrowdSec + Zabbix + Loki | Security ticket + risk assessment |
| **Containers** | Health, restarts, logs, resource usage | Docker API + Alloy → Loki | Auto-ticket on crash loop |
| **Certificates** | TLS/SSL expiry dates | Zabbix + Uptime Kuma | Ticket at 30/14/7 days before expiry |
| **Intrusion** | Brute-force, scans, anomalies | CrowdSec → Loki → AI | Immediate ticket + IP block |

---

## Container Infrastructure

<details>
<summary><strong>Click to expand full container list (35 containers)</strong></summary>

### Core Stack — 8 Containers (#1–#8)

| # | Container | Image | Purpose | Network(s) |
|---|-----------|-------|---------|------------|
| 1 | mcp-postgres | postgres:16-alpine | Central database | data |
| 2 | mcp-redis | redis:7-alpine | Cache & queue | data |
| 3 | mcp-pgvector | pgvector/pgvector:pg16 | Vector DB for RAG | data, ai |
| 4 | mcp-openbao | openbao:2.1 | Secrets management | sec, app |
| 5 | mcp-nginx | nginx:1.27-alpine | Reverse proxy + dashboard | edge, app |
| 6 | mcp-keycloak | keycloak:26.0 | Identity & MFA (SSO) | app, data |
| 7 | mcp-n8n | n8nio/n8n:1.76.1 | Workflow automation | app, data |
| 8 | mcp-ntfy | binwiederhier/ntfy | Local push notifications | app |

### Ops Stack — 10+1 Containers (#9–#16 + supporting)

| # | Container | Image | Purpose | Network(s) |
|---|-----------|-------|---------|------------|
| — | mcp-zammad-init | zammad:6.4.1 | One-shot DB migration (exits) | app, data |
| 9 | mcp-zammad-rails | zammad:6.4.1 | Ticketing web UI | app, data |
| 10 | mcp-zammad-websocket | zammad:6.4.1 | Real-time WebSocket | app |
| 11 | mcp-zammad-scheduler | zammad:6.4.1 | Background jobs | app, data |
| 12 | mcp-elasticsearch | elasticsearch:8.17.0 | Full-text search | app |
| — | mcp-zammad-memcached | memcached:1.6-alpine | Session cache | app |
| — | mcp-bookstack-db | mariadb:11.6 | BookStack database | data |
| 13 | mcp-bookstack | bookstack:24.12.1 | Knowledge base & wiki | app, data |
| 14 | mcp-vaultwarden | vaultwarden:1.32.5 | Password manager | app |
| 15 | mcp-portainer | portainer-ce:2.21.5 | Docker management UI | app |
| 16 | mcp-diun | diun:4.28 | Image update notifications | app |

### Telemetry Stack — 8 Containers (#17–#24)

| # | Container | Image | Purpose | Network(s) |
|---|-----------|-------|---------|------------|
| 17 | mcp-zabbix-server | zabbix-server-pgsql:7.0.0-alpine | Monitoring engine | app, data |
| 18 | mcp-zabbix-web | zabbix-web-nginx-pgsql:7.0.0-alpine | Zabbix web UI | app, data |
| 19 | mcp-grafana | grafana:11.4.0 | Dashboards & visualization | app, data |
| 20 | mcp-loki | loki:3.3.2 | Log aggregation | app, data |
| 21 | mcp-alloy | alloy:v1.5.1 | Log & metric collector | app |
| 22 | mcp-uptime-kuma | uptime-kuma:1 | Availability monitoring | app |
| 23 | mcp-crowdsec | crowdsec:latest | Intrusion detection (offline) | sec, app |
| 24 | mcp-grafana-renderer | grafana-image-renderer | PDF/image export | app |

### Remote Stack — 3 Containers (#25–#27)

| # | Container | Image | Purpose | Network(s) |
|---|-----------|-------|---------|------------|
| 25 | mcp-meshcentral | meshcentral:latest | Agent-based remote desktop | app |
| 26 | mcp-guacamole | guacamole:1.5.5 | Browser-based RDP/SSH/VNC | app, data |
| 27 | mcp-guacd | guacd:1.5.5 | Guacamole proxy daemon | app |

### AI Stack — 5 Containers (#28–#32)

| # | Container | Image | Purpose | Network(s) |
|---|-----------|-------|---------|------------|
| 28 | mcp-ollama | ollama:latest | LLM inference (GPU) | ai |
| 29 | mcp-litellm | litellm:main-stable | Model router / gateway | ai |
| 30 | mcp-langchain | custom (Python 3.12) | AI analysis pipeline worker | ai, data |
| 31 | mcp-ai-gateway | custom (FastAPI) | Internal AI REST API | ai, app |
| 32 | mcp-redis-queue | redis:7-alpine | Dedicated AI job queue | ai |

</details>

---

## Security

| Layer | Measure |
|-------|---------|
| **Network** | 5 isolated Docker bridge networks; only `mcp-edge-net` reachable from LAN |
| **Identity** | Keycloak SSO with TOTP MFA for all web services |
| **Secrets** | OpenBao (HashiCorp Vault fork) for centralized secret management |
| **Passwords** | Auto-generated, 32+ characters, `.env` with `chmod 600` |
| **Containers** | `no-new-privileges` on all containers, non-root users, read-only FS where possible |
| **IDS** | CrowdSec with `online_client.enabled: false` (local-only scenarios, no cloud) |
| **Proxy** | All traffic through Nginx; zero external port bindings on any container |
| **AI Gateway** | Internal-only on `mcp-ai-net` — **never exposed to public internet** |
| **Logs** | All access → Alloy → Loki → queryable, auditable, AI-analyzable |
| **Two Redis** | `mcp-redis` (cache) and `mcp-redis-queue` (AI jobs) — never mixed |

---

## Installation

### Hardware Requirements

| Profile | CPU | RAM | Disk | GPU | Use Case |
|---------|-----|-----|------|-----|----------|
| **Core only** (no AI) | 4 cores | 16 GB | 60 GB NVMe | — | Monitoring, ticketing, wiki — evaluation |
| **Full stack** (min.) | 8 cores | 32 GB | 100 GB NVMe | — | All 35 containers, AI at ~15 tok/s (CPU) |
| **Recommended** | 16 cores | 64 GB | 200 GB NVMe | NVIDIA T4 (16 GB) | Comfortable headroom, AI at ~60 tok/s |
| **Optimal** | 16+ cores | 128 GB | 500 GB NVMe | NVIDIA RTX 4070+ (12 GB) | Multi-tenant production |

#### RAM Distribution (Full Stack, 32 GB)

| Area | Components | RAM |
|------|-----------|-----|
| Core | PostgreSQL, pgvector, Redis (×2), OpenBao, Nginx, Keycloak, n8n, ntfy | 5 GB |
| Ops | Zammad (3×), Elasticsearch, Memcached, BookStack + MariaDB, Vaultwarden, Portainer | 4 GB |
| Telemetry | Zabbix (2×), Grafana (2×), Loki, Alloy, Uptime Kuma, CrowdSec | 4 GB |
| Remote | MeshCentral, Guacamole (2×) | 1 GB |
| **AI** | **Ollama + LiteLLM + LangChain + AI Gateway** | **12 GB** |
| System | OS, Docker, DIUN, buffers | 4 GB |
| Reserve | — | 2 GB |

### Quick Start

```bash
# 1. Clone repository
git clone https://github.com/<org>/MCP.git && cd MCP

# 2. Configure environment (replace ALL 'CHANGE_ME_' values)
cp .env.example .env && chmod 600 .env

# 3. Install (6-phase gate system)
sudo bash scripts/mcp-install.sh
```

### Installation Phases

The installation uses a **Phases + Gates** approach. Each gate must pass before the next phase starts. On failure, the script stops immediately with a detailed error log.

```
Phase 1: Preflight     →  Docker, GPU, RAM, disk, .env, images present
Phase 2: Environment   →  5 networks, volumes, certificates
Phase 3: Core Stack    →  PostgreSQL, Redis, pgvector, Nginx, Keycloak healthy
Phase 4: Ops Stack     →  Zammad, BookStack, Vaultwarden, Portainer OK
Phase 5: Telemetry     →  Zabbix, Grafana, Loki, CrowdSec OK
Phase 6: AI Stack      →  Ollama + models loaded, AI Gateway healthy, end-to-end test
```

**Resume on failure:**

```bash
sudo bash scripts/mcp-install.sh --resume-from phase4   # Resume from phase
sudo bash scripts/mcp-install.sh --only phase6           # Run single phase
sudo bash scripts/mcp-install.sh --clean                  # Full reinstall
```

**Example failure output:**

```
╔══════════════════════════════════════════════════════════════╗
║  ❌  MCP INSTALLATION STOPPED                                ║
║  Phase:  4 — Ops Stack                                       ║
║  Gate:   Zammad UI not loading                               ║
║  Cause:  Assets 404 — RAILS_SERVE_STATIC_FILES missing       ║
║  Log:    logs/mcp-install-error-20260217-143022.log          ║
║  → sudo bash scripts/mcp-install.sh --resume-from phase4     ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Operations

### Makefile Reference

```bash
make up                  # Start all stacks (Core → Ops → Telemetry → Remote → AI)
make down                # Stop all (reverse order)
make restart             # Full restart
make status              # Health check all 35 containers
make ps                  # Quick container overview
make logs                # Tail all logs

make up-core             # Start single stack
make down-ai             # Stop single stack
make logs-telemetry      # Tail single stack

make test                # All tests (smoke + AI + security)
make test-smoke          # Service reachability
make test-ai             # AI pipeline end-to-end
make test-security       # Network isolation audit

make backup              # Timestamped backup (volumes + DBs)
make restore             # Interactive restore
make pull-images         # Pull all Docker images
make clean               # ⚠ DESTRUCTIVE — remove all + volumes
```

### Backup & Restore

```bash
bash scripts/mcp-backup.sh       # Full: PostgreSQL dumps + Docker volumes + configs
bash scripts/mcp-restore.sh      # Interactive restore from timestamped archive
```

### Testing

| Suite | What it validates | Command |
|-------|-------------------|---------|
| **Smoke** | All 35 containers reachable, health checks pass, 13 proxy paths OK | `make test-smoke` |
| **AI Pipeline** | Alert → Redis queue → LangChain analysis → response stored | `make test-ai` |
| **Security** | Network isolation between all 5 networks, port exposure, CrowdSec rules | `make test-security` |

---

## AI Gateway API

The AI Gateway exposes a FastAPI REST API on `http://ai-gateway:8000` (internal only, bearer token required):

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health status of all dependencies (Redis, Ollama, LiteLLM, pgvector) |
| `GET` | `/metrics` | Prometheus-format metrics (requests, queue length, durations) |
| `POST` | `/api/v1/analyze` | Submit alert for AI analysis → queued in Redis |
| `POST` | `/api/v1/embed` | Generate embedding for text → store in pgvector |
| `GET` | `/api/v1/search` | RAG similarity search (query, top_k, threshold) |
| `POST` | `/api/v1/ingest` | Ingest document for RAG (automatic chunking + embedding) |
| `GET` | `/api/v1/jobs` | List all analysis jobs (with pagination) |
| `GET` | `/api/v1/jobs/{id}` | Get specific job status + results |
| `GET` | `/api/v1/models` | List available LLM models via Ollama |
| `DELETE` | `/api/v1/knowledge/{id}` | Remove RAG entry from pgvector |

### Example: Submit Alert

```bash
curl -X POST http://ai-gateway:8000/api/v1/analyze \
  -H "Authorization: Bearer ${AI_GATEWAY_SECRET}" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "zabbix",
    "severity": "high",
    "host": "printer-01.local",
    "description": "Paper jam detected in tray 2, recurring for 3 days",
    "metrics": {"pages_today": 42, "toner_level": 65, "uptime_hours": 720}
  }'
```

---

## Day-2 Operations

### AI Workflows (n8n)

| # | Workflow | Trigger | Output |
|---|----------|---------|--------|
| W1 | Alert → AI Analysis | Zabbix webhook | Zammad ticket with root-cause analysis |
| W2 | Log Anomaly → Ticket | Loki alert rule | Ticket + Grafana annotation |
| W3 | Security Finding → Ticket | CrowdSec / Trivy | Priority security ticket |
| W4 | Ticket Close → Knowledge | Zammad webhook | BookStack article + pgvector embedding |
| W5 | BookStack → RAG Ingestion | BookStack webhook | Automatic knowledge base update |
| W6 | Predictive Alert | Cron (every 6h) | Warning ticket if risk threshold exceeded |
| W7 | Capacity Planning | Cron (weekly) | Resource trend report |
| W8 | Daily Health Report | Cron (daily 08:00) | System status summary via ntfy |

### Scheduled Jobs

| Job | Schedule | Description |
|-----|----------|-------------|
| Backup | Daily 03:00 | Full volume + DB backup |
| AI Daily Report (W8) | Daily 08:00 | System health summary |
| Embedding Refresh | Daily 02:00 | Refresh RAG index |
| Predictive Scan (W6) | Every 6 hours | Trend analysis + anomaly detection |
| Capacity Report (W7) | Weekly Sun 09:00 | Resource planning report |
| AI Model Health | Hourly | Verify Ollama models responsive |

---

## Go-Live Checklist

| Check | Test | Expected |
|-------|------|----------|
| All containers | `make status` | 35/35 healthy, 0 restarts in 1h |
| Dashboard | `http://<IP>/` | All 13 service tiles green |
| Proxy paths | `make test-smoke` | All 13 paths return HTTP 200/302 |
| Keycloak MFA | Login without TOTP | Must fail |
| Zabbix | Add test host | Monitoring data flows |
| AI Gateway | `curl http://<IP>:8000/health` (internal) | `"status": "healthy"` |
| AI pipeline | Submit test alert | Ticket created in Zammad |
| RAG | Query pgvector | Returns similar past incidents |
| ntfy | Check push channel | Notification received |
| CrowdSec | Simulate brute-force | Alert triggered |
| Backup | `make backup` + `make restore` | Round-trip successful |
| Network isolation | `make test-security` | No cross-network leaks |
| Ollama models | `docker exec mcp-ollama ollama list` | mistral + nomic-embed-text loaded |
| No outbound traffic | `tcpdump -i any dst not 192.168.1.0/24` | Zero packets |

---

## Known Issues & Patterns

> Documented from Phase 2–5 installation testing.

| # | Issue | Cause | Fix |
|---|-------|-------|-----|
| 1 | PostgreSQL: `permission denied to create database` | Role missing CREATEDB | `ALTER ROLE zammad CREATEDB;` in init-db.sh |
| 2 | Zammad: Assets 404, loading screen hangs | `RAILS_SERVE_STATIC_FILES` not set | Add to compose/ops env |
| 3 | Loki: CrashLoop `compactor.delete-request-store` | Retention without compactor config | `retention_enabled: false` in loki-config.yml |
| 4 | Zabbix: `fe_sendauth: no password supplied` | Config bind-mount overrides env vars | Remove bind-mount |
| 5 | n8n: Healthcheck `Connection refused` | `localhost` resolves to IPv6 | Use `127.0.0.1`, `start_period: 120s` |
| 6 | Guacamole: 404 on `/` | Context path is `/guacamole/` | Correct URL in tests/nginx |
| 7 | MeshCentral: Empty reply | Uses HTTPS internally on 443 | Use `https://` in health checks |
| 8 | Orphan container warnings | No unified project name | Set `COMPOSE_PROJECT_NAME=mcp` in .env |

### Design Patterns (Prevent Issues)

1. **No bind-mounts for configs that need env vars** — bind-mounts override everything
2. **Always `127.0.0.1` not `localhost`** in healthchecks (IPv6 issue)
3. **`start_period: 120s`** for services with database migrations
4. **Check context paths** before writing healthchecks (e.g., `/guacamole/` not `/`)
5. **Check protocol** (HTTP vs HTTPS) for each service
6. **`COMPOSE_PROJECT_NAME=mcp`** always in `.env`
7. **Two Redis instances** — `mcp-redis` (cache) and `mcp-redis-queue` (AI jobs), never mix
8. **CrowdSec offline** — `online_client.enabled: false`, local scenarios only

---

## Development Status

### ✅ Operational

| Component | Status |
|-----------|--------|
| Docker Compose architecture (5 stacks, 5 networks, 35 containers) | Stable |
| Nginx reverse proxy with 13 service paths + HTML dashboard | Stable |
| Health checks on all containers | Stable |
| Alloy → Loki log pipeline + Grafana datasources (Loki, Zabbix) | Stable |
| Security hardening (no-new-privileges, network isolation) | Stable |
| 6-phase installer with gate checks + resume capability | Stable |
| AI Gateway — full REST API (10 endpoints, bearer auth, Redis dedup) | Functional |
| AI Gateway — pgvector RAG service (search, store, delete, audit log) | Functional |
| LangChain Worker — 7-step pipeline (RAG → prompt → LLM → ticket → ntfy → feedback) | Functional |
| LangChain Worker — LiteLLM-first with Ollama fallback | Functional |
| LangChain Worker — Zammad ticket creation + ntfy push notifications | Functional |
| LangChain Worker — pgvector embedding storage + audit logging | Functional |
| Prompt template system (German, structured JSON output) | Functional |
| Smoke / AI / Security test suites | Functional |

### 🔨 Needs Integration / Validation

| Component | Current State | Target |
|-----------|--------------|--------|
| n8n AI workflows (W1–W8) | Workflow directory empty | JSON workflow definitions triggering the AI Gateway |
| Keycloak SSO integration | Keycloak runs standalone | SSO connected to Grafana, Zammad, BookStack |
| OpenBao secrets rotation | Container runs standalone | Services fetch secrets from Vault instead of `.env` |
| BookStack → RAG ingestion (W5) | Not connected | Wiki pages auto-embedded for RAG |
| Grafana AI pipeline dashboard | Not created | Prometheus metrics dashboard for AI ops |
| CrowdSec → AI Gateway (W3) | Not connected | IDS findings trigger AI security analysis |
| End-to-end production validation | Lab-tested | Full validation under real workloads |

---

## Offline Deployment

MCP is designed for air-gapped environments with zero internet dependency at runtime:

```bash
# On a machine WITH internet:
bash scripts/mcp-pull-images.sh                          # Pull all images

# Transfer to air-gapped machine, then:
sudo bash scripts/mcp-install.sh                          # Install fully offline
```

All Docker images, Ollama models, and CrowdSec scenarios are pre-loaded. No runtime internet access required — verified via `tcpdump`.

---

## Project Structure

```
MCP/
├── .env.example                     ← All passwords, DB creds, image tags (65+ variables)
├── Makefile                         ← Operations shortcuts (make up/down/test/backup/clean)
│
├── compose/                         ← Docker Compose per stack
│   ├── core/docker-compose.yml      ←   #1–#8:   PG, Redis, pgvector, OpenBao, Nginx, Keycloak, n8n, ntfy
│   ├── ops/docker-compose.yml       ←   #9–#16:  Zammad (3×+init), ES, Memcached, BookStack+MariaDB, Vaultwarden, Portainer, DIUN
│   ├── telemetry/docker-compose.yml ←   #17–#24: Zabbix (2×), Grafana (2×), Loki, Alloy, Uptime Kuma, CrowdSec
│   ├── remote/docker-compose.yml    ←   #25–#27: MeshCentral, Guacamole, guacd
│   └── ai/docker-compose.yml        ←   #28–#32: Ollama, LiteLLM, LangChain, AI Gateway, Redis Queue
│
├── containers/                      ← Custom builds (multi-stage Docker, non-root, Python 3.12)
│   ├── ai-gateway/                  ←   FastAPI REST API (asyncpg, pgvector, prometheus-client)
│   │   ├── app/main.py              ←     10 endpoints: analyze, embed, search, ingest, jobs, models, knowledge
│   │   ├── app/services/            ←     rag_service.py, ollama_client.py
│   │   └── Dockerfile
│   └── langchain-worker/            ←   Full analysis pipeline (LangChain 0.3, pgvector, httpx)
│       ├── app/worker.py            ←     7-step pipeline: RAG → prompt → LLM → ticket → ntfy → feedback
│       ├── app/services/            ←     llm_client.py, pgvector_service.py, zammad_client.py, ntfy_client.py
│       ├── app/prompts.py           ←     Template loader + placeholder injection
│       └── Dockerfile
│
├── config/                          ← Service configurations
│   ├── ai/                          ←   models.yml (LiteLLM routing), rag-config.yml, prompts/alert-analysis.txt
│   ├── alloy/                       ←   Log collector (Docker → Loki)
│   ├── crowdsec/                    ←   acquis.yml (log sources), local scenarios
│   ├── grafana/                     ←   datasources.yml, dashboard-provider.yml
│   ├── keycloak/                    ←   Realm export
│   ├── loki/                        ←   loki-config.yml
│   ├── n8n/                         ←   workflows/ (W1–W8 JSON definitions)
│   ├── nginx/                       ←   nginx.conf, conf.d/default.conf (13 proxy paths), dashboard HTML
│   ├── ntfy/                        ←   server.yml (topics, auth)
│   └── portainer/                   ←   Persistence data
│
├── scripts/                         ← Operations
│   ├── mcp-install.sh               ←   6-phase installer with gate checks + resume
│   ├── mcp-start.sh / mcp-stop.sh   ←   Ordered startup/shutdown (Core→…→AI / AI→…→Core)
│   ├── mcp-status.sh                ←   Tabular health check all containers
│   ├── mcp-backup.sh / mcp-restore.sh  ← Backup + interactive restore
│   ├── mcp-pull-images.sh           ←   Pull all images (online prep)
│   ├── init-db.sh                   ←   PostgreSQL multi-DB init (6 databases + roles)
│   ├── init-pgvector.sh             ←   pgvector extension + mcp_knowledge table + index
│   └── gen-test-env.sh              ←   Generate test .env
│
├── tests/
│   ├── smoke-test.sh                ←   All containers + 13 proxy paths
│   ├── ai-pipeline-test.sh          ←   Alert → queue → analysis → response
│   └── security-test.sh             ←   Network isolation + CrowdSec
│
├── docs/                            ← Documentation & assets
├── images/                          ← Exported Docker images (offline deployment)
├── models/                          ← Exported Ollama models (offline deployment)
└── logs/                            ← Install error logs (timestamped)
```

---

## Technology Stack

| Category | Technologies |
|----------|-------------|
| **Databases** | PostgreSQL 16, pgvector (768-dim), Redis 7 (×2), MariaDB 11.6, Elasticsearch 8.17, Memcached |
| **Identity** | Keycloak 26 (SSO + TOTP MFA) |
| **Monitoring** | Zabbix 7.0, Grafana 11.4, Loki 3.3, Alloy v1.5, Uptime Kuma |
| **Security** | CrowdSec (IDS, offline), OpenBao 2.1 (secrets), Vaultwarden 1.32 (passwords) |
| **Helpdesk** | Zammad 6.4 |
| **Automation** | n8n 1.76, ntfy (push notifications) |
| **Wiki** | BookStack 24.12 |
| **Remote** | MeshCentral, Apache Guacamole 1.5 |
| **AI / ML** | Ollama (Mistral 7B, Llama 3 8B, nomic-embed-text), LiteLLM, LangChain 0.3, pgvector |
| **Custom** | AI Gateway (FastAPI + asyncpg + prometheus), LangChain Worker (Python 3.12) |
| **Infra** | Docker Compose v2, Nginx 1.27, Portainer 2.21, DIUN 4.28 |

---

## Roadmap

| Version | Feature | Timeline |
|---------|---------|----------|
| **v7.0** | Infrastructure + AI analysis pipeline | ✅ Current |
| **v7.1** | n8n workflow integration + Keycloak SSO + Grafana AI dashboard | In Progress |
| **v7.2** | AI auto-remediation (restart, scale, cleanup) | Q2 2026 |
| **v7.3** | Customer portal with AI status updates | Q3 2026 |
| **v8.0** | Multi-node (AI on dedicated GPU server) | Q4 2026 |
| **v8.1** | Fine-tuning LLM on own ticket history | Q1 2027 |

---

## Versioning & Support

| Version | Status |
|---------|--------|
| **7.x** | Actively maintained (Beta) |
| **6.x** | Security patches only |
| **≤ 5.x** | End of life |

---

## Licensing

**MCP is proprietary software.** Copyright © Masdor. All rights reserved.

| Usage | Terms |
|-------|-------|
| **Evaluation** (Lab/Test) | Permitted only with written approval |
| **Commercial License** | Individual agreement required |
| **Production** | Prohibited without a valid license |

Third-party components remain subject to their respective open-source licenses. MCP claims no ownership of these projects.

**Contact:** `contact@masdor.de` · **Security:** `security@masdor.de`

---

## Disclaimer

MCP is a technical platform. It must be validated in a lab environment and deployed with appropriate security measures. Documentation provides operational guidance but does not constitute legal or compliance advice.

---

<p align="center">
  <strong>MCP v7 — AI-Powered IT Operations Center</strong><br>
  <em>Local. Intelligent. Automated.</em><br><br>
  <code>sudo bash scripts/mcp-install.sh</code>
</p>
