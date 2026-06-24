# MAESTRO

Personal infrastructure for my IdeaPad P500 server. One old laptop, running Debian 13, hosting my AI workspace, development tools, and monitoring stack. Accessible from anywhere via Tailscale.

## Hardware

| Component | Current | Planned |
|-----------|---------|---------|
| Machine | Lenovo IdeaPad P500 Touch (Model 20253) | - |
| CPU | Intel (integrated graphics only) | - |
| RAM | 12GB DDR3 (8GB + 4GB) | 16GB (2 × 8GB) |
| Storage | 1TB Seagate HDD (5400 RPM) | 500GB SATA SSD (when prices normalize) |
| OS | Debian 13 "Trixie" (headless) | - |
| Network | WiFi + Tailscale | Ethernet preferred long-term |

## Services

| Service | Purpose | Port | Access |
|---------|---------|------|--------|
| **Odysseus** | AI workspace — chat, agents, documents, notes, email, calendar | 7000 | `http://ideapad` |
| **Grafana** | Dashboards for system health | 3001 | `http://ideapad/grafana` |
| **Prometheus** | Metrics collection and storage | 9090 | internal only |
| **Node Exporter** | System metrics (CPU, RAM, disk, temps) | 9100 | internal only |
| **NGINX** | Reverse proxy for all services | 80 | entry point |
| **Tailscale** | Secure remote access from anywhere | - | mesh VPN |

## Architecture

```
Internet → Tailscale → ideapad (Debian 13)
                          ├── NGINX (port 80)
                          │     ├── / → Odysseus (port 7000)
                          │     └── /grafana → Grafana (port 3001)
                          ├── Docker Engine
                          │     ├── odysseus
                          │     ├── grafana + prometheus + node-exporter
                          │     └── (future services)
                          └── Tailscale (SSH + mesh networking)
```

## Getting Started

### Prerequisites

- Debian 13 installed and SSH accessible
- Tailscale account
- This repo cloned to `~/homelab`

### Bootstrap

From a fresh Debian install, run:

```bash
bash bootstrap.sh
```

This installs all packages, configures Docker, sets up Tailscale, deploys NGINX, and starts every service.

### Manual steps after bootstrap

1. **Tailscale auth**: `sudo tailscale up --ssh` and follow the URL
2. **Grafana**: Login at `http://ideapad:3001` (admin/admin), add Prometheus datasource at `http://prometheus:9090`, import dashboard 1860
3. **Odysseus**: Check `docker compose logs odysseus` for the initial admin password
4. **Change all default passwords**

## Directory Structure

```
homelab/
├── README.md                   # This file
├── bootstrap.sh                # One-command server rebuild
├── docker/
│   ├── docker-compose.yml      # All services
│   ├── odysseus/
│   │   ├── .env.example
│   │   └── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── grafana/
│   │   └── dashboards/
│   └── nginx/
│       ├── nginx.conf
│       └── sites/
│           ├── odysseus.conf
│           ├── forgejo.conf
│           └── grafana.conf
├── scripts/
│   ├── backup.sh               # Backup volumes and configs
│   ├── health-check.sh         # Quick service status
│   └── tailscale-setup.sh
└── docs/
    ├── hardware.md             # Specs, upgrades, maintenance log
    ├── services.md             # Service details and config notes
    └── recovery.md             # Disaster recovery from scratch
```

## Design Principles

- **One machine, many services.** Docker keeps them isolated.
- **Headless and remote-first.** SSH and Tailscale. The lid stays closed.
- **Infrastructure as code.** Every config lives in Git. The server is rebuildable.
- **Incremental.** Services get added as needed, not all at once.
- **Local-first, cloud-optional.** Everything runs here. No external dependencies except Tailscale for access.
- **AI-capable, not AI-dependent.** Odysseus gives me an AI workspace. Small local models run here. API models when I need more power. But the system works without AI too.

## Current Status

- [x] Debian 13 installed (headless)
- [x] SSH access (local + Tailscale)
- [x] Docker engine
- [x] NGINX reverse proxy
- [ ] Odysseus deployed
- [ ] Grafana + Prometheus deployed
- [ ] RAM upgraded to 16GB
- [ ] SSD upgrade
- [ ] Automated backups

## Recovery

If the server dies:

1. Reinstall Debian 13
2. `ssh coffee6ean@<ip>`
3. `git clone https://github.com/coffee6ean/homelab.git`
4. `cd homelab && bash bootstrap.sh`
5. Restore backups to Docker volumes

See `docs/recovery.md` for detailed steps.

---

Built for a 2013 Lenovo IdeaPad. Maintained with disgusting amounts of caffeine.
