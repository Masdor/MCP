<p align="center">
  <img src="docs/assets/mcp-logo.png" alt="MCP Logo" width="200" />
</p>

<h1 align="center">MCP — Managed Control Platform</h1>

<p align="center">
  <strong>Eine vollständige, KI-gestützte IT-Operations-Plattform für Managed Service Provider</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Version-v7-blue?style=flat-square" alt="Version" />
  <img src="https://img.shields.io/badge/Containers-35-green?style=flat-square" alt="Containers" />
  <img src="https://img.shields.io/badge/Docker_Compose-v2-blue?style=flat-square" alt="Docker Compose" />
  <img src="https://img.shields.io/badge/License-Proprietary-red?style=flat-square" alt="License" />
  <img src="https://img.shields.io/badge/Platform-Linux_x86__64-lightgrey?style=flat-square" alt="Platform" />
</p>

---

## Überblick

**MCP** (Managed Control Platform) bündelt 35 Docker-Container in fünf isolierte Stacks und liefert damit eine schlüsselfertige Operations-Zentrale für MSPs: Ticketing, Monitoring, Secrets Management, Remote-Zugriff, Workflow-Automatisierung und eine lokale KI-Pipeline — alles hinter einem einzigen Nginx-Reverse-Proxy, geschützt durch Keycloak SSO und CrowdSec IDS.

```
┌─────────────────────────────────────────────────────────────────┐
│                        mcp-edge-net                             │
│  ┌──────────┐                                                   │
│  │  Nginx   │──── / ─────────────────── Dashboard               │
│  │ Reverse  │──── /auth ────────────── Keycloak (SSO)           │
│  │  Proxy   │──── /auto ────────────── n8n (Automation)         │
│  │  :80     │──── /tickets ─────────── Zammad (Helpdesk)        │
│  │          │──── /monitor ─────────── Zabbix (Monitoring)      │
│  │          │──── /dash ────────────── Grafana (Dashboards)     │
│  │          │──── /wiki ────────────── BookStack (Knowledge)    │
│  │          │──── /vault ───────────── Vaultwarden (Passwords)  │
│  │          │──── /manage ──────────── Portainer (Docker UI)    │
│  │          │──── /status ──────────── Uptime Kuma (Uptime)     │
│  │          │──── /mesh ────────────── MeshCentral (Remote)     │
│  │          │──── /remote ──────────── Guacamole (RDP/SSH)      │
│  │          │──── /notify ──────────── ntfy (Push)              │
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architektur

### Stack-Übersicht

| Stack | Container | Funktion |
|-------|-----------|----------|
| **Core** | 8 | PostgreSQL, Redis, pgvector, OpenBao, Nginx, Keycloak, n8n, ntfy |
| **Ops** | 8 + 1 Init | Zammad (Rails, WebSocket, Scheduler), Elasticsearch, Memcached, BookStack + MariaDB, Vaultwarden, Portainer, DIUN |
| **Telemetry** | 8 | Zabbix Server + Web, Grafana + Renderer, Loki, Alloy, Uptime Kuma, CrowdSec |
| **Remote** | 3 | MeshCentral, Guacamole, guacd |
| **AI** | 5 | Ollama, LiteLLM, LangChain Worker, AI Gateway, Redis Queue |
| | **35 total** | |

### Netzwerk-Isolation

```
mcp-data-net     PostgreSQL, Redis, pgvector ← nur Backend-Zugriff
mcp-app-net      Dienst-zu-Dienst-Kommunikation
mcp-edge-net     Nginx ↔ Außenwelt
mcp-ai-net       Ollama, LiteLLM, LangChain — vollständig isoliert
mcp-sec-net      CrowdSec ↔ Nginx Bouncer
```

Kein Container hat direkten Port-Zugang nach außen — sämtlicher Traffic läuft durch den Nginx-Reverse-Proxy.

---

## Features

### IT Operations
- **Ticketing & Helpdesk** — Zammad mit Elasticsearch-Volltextsuche
- **Monitoring** — Zabbix Server mit Grafana-Dashboards und Loki-Log-Aggregation
- **Uptime-Überwachung** — Uptime Kuma für HTTP/TCP/Ping-Checks
- **Secrets Management** — OpenBao (HashiCorp-Vault-Fork) für zentrales Secret Handling
- **Passwort-Manager** — Vaultwarden (Bitwarden-kompatibel)
- **Workflow-Automatisierung** — n8n für No-Code-Workflows und Webhooks
- **Push-Benachrichtigungen** — ntfy für Echtzeit-Alerts
- **Wissensdatenbank** — BookStack als internes Wiki

### Remote-Zugriff
- **MeshCentral** — Agentenbasiertes Remote Desktop Management
- **Apache Guacamole** — Browserbasierter RDP/SSH/VNC-Zugriff ohne Client

### KI-Pipeline (lokal & privat)
- **Ollama** — Lokale LLM-Inferenz (Mistral 7B, Llama 3 8B), optional mit GPU
- **LiteLLM** — Model-Router mit einheitlicher OpenAI-kompatibler API
- **LangChain Worker** — RAG-Pipeline mit pgvector-Embedding-Suche
- **AI Gateway** — Interner API-Endpunkt für Ticket-Analyse, Zusammenfassungen und Klassifikation

### Sicherheit
- **Keycloak** — SSO mit MFA für alle Dienste
- **CrowdSec** — Collaborative Intrusion Detection mit Nginx-Bouncer
- **Netzwerk-Segmentierung** — 5 isolierte Docker-Netzwerke
- **no-new-privileges** — Auf allen Containern aktiviert
- **Keine öffentlichen Ports** — Alles hinter Nginx

### Betrieb
- **Portainer** — Web-UI für Docker-Container-Management
- **DIUN** — Docker Image Update Notifications
- **Grafana Alloy** — Log-Shipping an Loki
- **Automatische Backups** — Skriptgesteuerte Sicherung aller Volumes und Datenbanken

---

## Voraussetzungen

| Ressource | Minimum | Empfohlen |
|-----------|---------|-----------|
| **OS** | Debian 12 / Ubuntu 24.04 | Debian 12 |
| **CPU** | 4 Kerne | 8+ Kerne |
| **RAM** | 16 GB | 32 GB (mit KI-Stack) |
| **Disk** | 80 GB SSD | 200 GB+ NVMe |
| **GPU** | — | NVIDIA (für Ollama) |
| **Software** | Docker Engine 24+, Docker Compose v2 | |

---

## Schnellstart

### 1. Repository klonen

```bash
git clone https://github.com/<org>/MCP.git
cd MCP
```

### 2. Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
chmod 600 .env
```

Die `.env`-Datei enthält alle Passwörter, Datenbank-Credentials und Image-Tags. Jeder `CHANGE_ME_`-Wert muss durch ein sicheres Passwort ersetzt werden.

### 3. Installation starten

```bash
sudo bash scripts/mcp-install.sh
```

Das Installationsskript durchläuft **6 Phasen** mit automatischen Gate-Checks:

| Phase | Aktion |
|-------|--------|
| 1 | System-Prüfung (Docker, Ressourcen, Netzwerk) |
| 2 | Core-Stack hochfahren (PostgreSQL, Redis, Nginx, Keycloak) |
| 3 | Ops-Stack hochfahren (Zammad, BookStack, Portainer) |
| 4 | Telemetry-Stack hochfahren (Zabbix, Grafana, Loki) |
| 5 | Remote-Stack hochfahren (MeshCentral, Guacamole) |
| 6 | AI-Stack hochfahren (Ollama, LiteLLM, LangChain) |

Bei einem Fehler stoppt die Installation mit einer detaillierten Fehlermeldung und kann anschließend nahtlos fortgesetzt werden:

```bash
sudo bash scripts/mcp-install.sh --resume-from phase3
```

### 4. Zugriff

Nach der Installation ist das Dashboard unter `http://<SERVER-IP>/` erreichbar. Alle Dienste sind über ihre jeweiligen Subpaths verfügbar (siehe Architektur-Diagramm oben).

---

## Betrieb

### Makefile-Kommandos

```bash
make up              # Alle Stacks starten (Core → Ops → Telemetry → Remote → AI)
make down            # Alle Stacks stoppen (umgekehrte Reihenfolge)
make restart         # Neustart aller Stacks
make status          # Status aller 35 Container anzeigen
make logs            # Logs aller Container verfolgen

make up-core         # Einzelnen Stack starten
make down-ai         # Einzelnen Stack stoppen
make logs-telemetry  # Logs eines Stacks verfolgen

make test            # Alle Tests ausführen (Smoke + AI + Security)
make test-smoke      # Smoke-Tests
make test-ai         # AI-Pipeline-Tests
make test-security   # Security-Tests

make backup          # Vollständiges Backup erstellen
make restore         # Backup wiederherstellen (interaktiv)
make pull-images     # Alle Docker-Images aktualisieren

make clean           # DESTRUKTIV — Alle Container + Volumes entfernen
```

### Backup & Restore

```bash
bash scripts/mcp-backup.sh       # Erstellt zeitgestempeltes Backup
bash scripts/mcp-restore.sh      # Interaktive Wiederherstellung
```

### Status-Prüfung

```bash
bash scripts/mcp-status.sh       # Detaillierter Health-Check aller Container
```

---

## Projektstruktur

```
MCP/
├── compose/                     # Docker Compose pro Stack
│   ├── core/                    #   Core-Stack (8 Container)
│   ├── ops/                     #   Ops-Stack (8+1 Container)
│   ├── telemetry/               #   Telemetry-Stack (8 Container)
│   ├── remote/                  #   Remote-Stack (3 Container)
│   └── ai/                      #   AI-Stack (5 Container)
├── config/                      # Konfigurationsdateien
│   ├── ai/                      #   LLM-Modelle, RAG-Config
│   ├── alloy/                   #   Log-Collector-Config
│   ├── crowdsec/                #   IDS-Regeln
│   ├── grafana/                 #   Datasources, Dashboards
│   ├── keycloak/                #   Realm-Export
│   ├── loki/                    #   Log-Aggregation
│   ├── n8n/                     #   Workflow-Templates
│   ├── nginx/                   #   Reverse-Proxy-Routing
│   ├── ntfy/                    #   Push-Notification-Config
│   └── portainer/               #   Container-Management
├── containers/                  # Custom Container-Builds
│   ├── ai-gateway/              #   FastAPI-basiertes AI Gateway
│   └── langchain-worker/        #   RAG-Pipeline Worker
├── scripts/                     # Operations-Skripte
│   ├── mcp-install.sh           #   6-Phasen-Installation
│   ├── mcp-start.sh             #   Geordnetes Hochfahren
│   ├── mcp-stop.sh              #   Geordnetes Herunterfahren
│   ├── mcp-status.sh            #   Health-Check
│   ├── mcp-backup.sh            #   Backup aller Volumes + DBs
│   ├── mcp-restore.sh           #   Interaktive Wiederherstellung
│   ├── mcp-pull-images.sh       #   Image-Updates
│   ├── init-db.sh               #   PostgreSQL-Initialisierung
│   ├── init-pgvector.sh         #   pgvector-Setup
│   └── gen-test-env.sh          #   Test-Umgebung generieren
├── tests/                       # Automatisierte Tests
│   ├── smoke-test.sh            #   Erreichbarkeit aller Dienste
│   ├── ai-pipeline-test.sh      #   KI-Endpunkt-Validierung
│   └── security-test.sh         #   Sicherheits-Audit
├── docs/                        # Dokumentation
├── images/                      # Container-Build-Artefakte
├── logs/                        # Installations- und Fehler-Logs
├── models/                      # LLM-Modell-Dateien
├── Makefile                     # Alle Betriebskommandos
└── .env.example                 # Umgebungsvariablen-Vorlage
```

---

## Technologie-Stack

| Kategorie | Technologien |
|-----------|-------------|
| **Datenbanken** | PostgreSQL 16, Redis 7, pgvector, MariaDB 11.6, Elasticsearch 8.17, Memcached |
| **Identity** | Keycloak 26 (SSO + MFA) |
| **Monitoring** | Zabbix 7.0, Grafana 11.4, Loki 3.3, Alloy, Uptime Kuma |
| **Helpdesk** | Zammad 6.4 |
| **Automatisierung** | n8n 1.76 |
| **Sicherheit** | CrowdSec, OpenBao 2.1, Vaultwarden 1.32 |
| **Remote** | MeshCentral, Apache Guacamole 1.5 |
| **KI** | Ollama, LiteLLM, LangChain, pgvector (RAG) |
| **Reverse Proxy** | Nginx 1.27 |
| **Container** | Docker Compose v2, Portainer 2.21, DIUN 4.28 |

---

## Sicherheitshinweise

- Alle `CHANGE_ME_`-Werte in `.env` **müssen** vor der Installation durch sichere, einzigartige Passwörter ersetzt werden.
- Die `.env`-Datei enthält sämtliche Secrets und darf **niemals** in ein Git-Repository committed werden.
- Der AI Gateway ist ausschließlich intern erreichbar und wird **nicht** über Nginx exponiert.
- Regelmäßige Backups über `make backup` werden dringend empfohlen.
- CrowdSec-Bouncer-Keys sollten nach der Installation rotiert werden.

---

## Fehlerbehandlung

Falls die Installation fehlschlägt, wird automatisch ein detaillierter Error-Log unter `logs/` erstellt. Die Installation kann anschließend gezielt fortgesetzt werden:

```bash
# Ab einer bestimmten Phase fortsetzen
sudo bash scripts/mcp-install.sh --resume-from phase3

# Nur eine bestimmte Phase ausführen
sudo bash scripts/mcp-install.sh --only phase6

# Komplett neu installieren
sudo bash scripts/mcp-install.sh --clean
```

---

## Lizenz

Proprietär — Alle Rechte vorbehalten.

---

<p align="center">
  <sub>Built with 🐳 Docker &nbsp;·&nbsp; Secured by 🛡️ CrowdSec &nbsp;·&nbsp; Powered by 🤖 Ollama</sub>
</p>
