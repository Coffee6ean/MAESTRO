# NGINX + Odysseus Setup

How the IdeaPad serves Odysseus through an NGINX reverse proxy, accessible from anywhere on the Tailscale network.

## Architecture

```txt
Browser (remote, via Tailscale)
       │
       ▼
   ideapad:8081
       │
       ▼
   NGINX (container, port 80 inside container → 8081 on host)
       │
       ▼
   odysseus-odysseus-1:7000 (on odysseus_default network)
       │
       ├── chromadb (search index)
       ├── searxng (web search engine)
       └── ntfy (notifications)
```

NGINX is the **reverse proxy**. It sits between the outside world and Odysseus. Odysseus itself is never exposed directly — only NGINX talks to it. This means:

- One entry point for all services
- Easy to add more services later (just add `location` blocks)
- Odysseus stays isolated on the Docker network

---

## Step-by-Step Setup

### 1. Create the NGINX config

```bash
mkdir -p ~/docker/nginx

cat > ~/docker/nginx/default.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

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
EOF
```

**What each line does:**

| Directive | Purpose |
|-----------|---------|
| `listen 80` | NGINX listens on port 80 inside its container |
| `server_name localhost` | This block handles all requests (no domain filtering yet) |
| `proxy_pass http://odysseus-odysseus-1:7000` | Forward requests to the Odysseus container on its internal port |
| `proxy_http_version 1.1` | Required for WebSocket support |
| `Upgrade` and `Connection` headers | Enable WebSockets — needed for streaming AI responses |
| `Host $host` | Passes the original hostname to Odysseus |
| `X-Real-IP $remote_addr` | Tells Odysseus the real IP of whoever's connecting |
| `proxy_read_timeout 86400` | 24-hour timeout — prevents long AI responses from being cut off |

### 2. Launch NGINX on the Odysseus network

Odysseus creates its own Docker network called `odysseus_default` (Docker Compose names networks `<project>_default`). NGINX must be on the same network to reach the Odysseus container by name.

```bash
docker run -d \
  --name nginx \
  --network odysseus_default \
  -p 8081:80 \
  -v ~/docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:latest
```

**What each flag does:**

| Flag | Purpose |
|------|---------|
| `-d` | Run in detached mode (background) |
| `--name nginx` | Give the container a recognizable name |
| `--network odysseus_default` | Join Odysseus's network so containers can talk |
| `-p 8081:80` | Map host port 8081 → container port 80 (8080 was taken by searxng) |
| `-v ...default.conf:ro` | Mount our config file into the container (read-only) |
| `nginx:latest` | Use the latest official NGINX image |

### 3. Verify

```bash
docker ps | grep nginx
```

Should show `nginx` with status `Up`.

From any device on the Tailscale network, open:

```txt
http://ideapad:8081
```

---

## Key Learnings

### Container names on Docker Compose networks

Docker Compose names containers as `<project>-<service>-<instance>`. Odysseus's main service is named `odysseus-odysseus-1`. On the same Docker network, other containers can reach it using that full name.

To find a container's exact name:

```bash
docker compose ps
```

### Port conflicts

Odysseus's `searxng` service binds to port 8080 on the host (`127.0.0.1:8080`). That's why NGINX couldn't use 8080. Always check what ports are already in use:

```bash
docker ps
```

### Why not expose Odysseus directly?

- **Security**: Only NGINX faces the network. Odysseus is hidden.
- **Flexibility**: Adding more services later (Grafana, Forgejo, Mealie) just means adding `location /grafana/` blocks. One port serves everything.
- **Single config**: Rate limiting, HTTPS, authentication — all done once in NGINX.

### Tailscale makes this work

`ideapad` resolves to the Tailscale IP from any device on the tailnet. No public IP. No port forwarding. No DNS. Just `http://ideapad:8081`.

---

## Troubleshooting

**NGINX container exits immediately:**

```bash
docker logs nginx
```

Usually a syntax error in `default.conf` or the upstream container name is wrong.

**Connection refused:**
The upstream container might not be running. Check:

```bash
docker ps | grep odysseus
```

**502 Bad Gateway:**
NGINX can't reach the upstream. Verify the network:

```bash
docker network inspect odysseus_default | grep -A 5 "nginx"
```

Both `nginx` and `odysseus-odysseus-1` should appear under `Containers`.

**Port already allocated:**

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

Find which container is using the port and either stop it or choose a different host port.
