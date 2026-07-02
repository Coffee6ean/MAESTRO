# NGINX + Tailscale Serve + Odysseus

How the IdeaPad serves Odysseus and other apps through an NGINX reverse proxy, accessible from anywhere on the Tailscale network via HTTPS.

## Architecture

```txt
Browser (remote, via Tailscale)
       │
       ▼
   Tailscale Serve (HTTPS → localhost:80)
       │
       ▼
   NGINX (Docker container, port 80)
       │
       ▼
   Odysseus (7000)  │  Grafana (3000)  │  future apps...
```

- **Tailscale Serve** provides HTTPS and remote access. No open ports on the router.
- **NGINX** is the reverse proxy. One entry point on port 80. Routes by hostname or path.
- **Apps** run isolated on the internal Docker network. They never face the internet directly.

---

## Current URLs

| URL | App |
|-----|-----|
| `https://ideapad.velociraptor-tint.ts.net/` | MAESTRO landing page |
| `https://ideapad.velociraptor-tint.ts.net:8443/` | Odysseus |
| `https://ideapad.velociraptor-tint.ts.net:9443/` | Grafana (soon) |

---

## Layer 1: Tailscale Serve

```bash
# Main entry point — all HTTPS traffic to NGINX on port 80
sudo tailscale serve --bg 80

# Per-app routes (bypass NGINX for direct access)
sudo tailscale serve --bg --https 8443 http://localhost:7000   # Odysseus
sudo tailscale serve --bg --https 9443 http://localhost:3000   # Grafana
```

**What this does:**

- `tailscale serve` listens on your Tailscale IP and forwards HTTPS traffic to a local port
- `--bg` runs it in the background
- `--https 8443` exposes the app on a specific port
- No certificates to manage — Tailscale handles TLS automatically

---

## Layer 2: NGINX

### Config

`~/docker/nginx/default.conf`:

```txt
# Default — landing page
server {
    listen 80;
    server_name localhost;

    location / {
        return 200 "MAESTRO — <a href='https://ideapad.velociraptor-tint.ts.net:8443/'>Odysseus</a> | <a href='https://ideapad.velociraptor-tint.ts.net:9443/'>Grafana</a>";
    }
}

# Odysseus (via hostname routing — future use)
server {
    listen 80;
    server_name odysseus.velociraptor-tint.ts.net;

    location / {
        proxy_pass http://odysseus-odysseus-1:7000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
    }
}
```

**Key directives:**

| Directive | Purpose |
|-----------|---------|
| `proxy_pass http://odysseus-odysseus-1:7000` | Forward to the Odysseus container on the internal Docker network |
| `proxy_http_version 1.1` | Required for WebSocket support |
| `Upgrade` / `Connection` headers | Enable WebSockets — needed for streaming AI responses |
| `proxy_read_timeout 86400` | 24-hour timeout — prevents long AI responses from being cut off |
| `server_name` blocks | Route different hostnames to different apps (ready for subdomain support) |

### Launch

```bash
docker run -d \
  --name nginx \
  --network internal \
  -p 80:80 \
  -v ~/docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:latest
```

**Flags:**

| Flag | Purpose |
|------|---------|
| `--network internal` | Custom Docker network shared with all apps |
| `-p 80:80` | Host port 80 → container port 80 |
| `-v ...:ro` | Mount config as read-only |
| `nginx:latest` | Official NGINX image |

---

## The `internal` Docker Network

All containers that need to talk to each other share this network:

```bash
# Create once
docker network create internal

# Odysseus containers joined to it
docker network connect internal odysseus-odysseus-1
docker network connect internal odysseus-chromadb-1
docker network connect internal odysseus-ntfy-1
docker network connect internal odysseus-searxng-1
```

This allows NGINX to reach `odysseus-odysseus-1:7000` by container name — Docker resolves it to the internal IP automatically.

---

## Troubleshooting

**NGINX container exits immediately:**

```bash
docker logs nginx
```

Usually a syntax error in `default.conf` or an upstream container name that doesn't exist (like Grafana before it's deployed).

**Port 80 already in use:**

```bash
sudo lsof -i :80
```

If host-level NGINX is running, stop it: `sudo systemctl stop nginx && sudo systemctl disable nginx`

**Tailscale Serve not responding:**

```bash
tailscale serve status
```

Should show `proxy http://127.0.0.1:80`. If not, restart: `sudo tailscale serve --https=443 off && sudo tailscale serve --bg 80`

**502 Bad Gateway:**
NGINX can't reach the upstream. Check both containers are on the same network:

```bash
docker network inspect internal | grep -E "nginx|odysseus"
```

**Odysseus CSS not loading under a subdirectory:**
Odysseus uses absolute paths (`/static/style.css`) and does not currently support a `ROOT_PATH` config. For now, access it directly via its dedicated Tailscale Serve port (8443). A feature request has been opened to add subdirectory support.

---

## Adding a New App

1. Start the app on the `internal` network:

```bash
docker run -d --name newapp --network internal -p 5000:5000 newapp:latest
```

2. Add a Tailscale Serve route:

```bash
sudo tailscale serve --bg --https 5443 http://localhost:5000
```

3. Add a link to the landing page in `~/docker/nginx/default.conf`

4. Restart NGINX:

```bash
docker restart nginx
```
