# 🌐 SmartShop - Network Architecture

> **Status**: ✅ Live in Production  
> **Server**: HostAsia VPS (Ubuntu 22.04 LTS)  
> **Orchestrator**: Dokploy  
> **Last Updated**: February 2026

---

## 1. Overview

SmartShop runs entirely on a single VPS using Docker containers orchestrated by **Dokploy**. All external traffic enters through **Traefik**, which acts as reverse proxy, SSL terminator, and router. Containers communicate privately on a Docker bridge network (`dokploy-network`). No traffic goes directly to any container — everything flows through Traefik.

---

## 2. Full Network Diagram

```mermaid
graph TD
    %% Nodes
    User(("👤 User / Internet"))
    DNS{"🌐 Cloudflare DNS"}
    
    subgraph VPS ["🖥️ HostAsia VPS (Ubuntu)"]
        style VPS fill:#f9f9f9,stroke:#333,stroke-width:2px
        
        Traefik["🚥 Traefik Reverse Proxy<br/>(SSL Termination :443)"]
        style Traefik fill:#ffcc00,stroke:#333
        
        subgraph DockerNet ["🐳 dokploy-network (Internal Bridge)"]
            style DockerNet fill:#e1f5fe,stroke:#0277bd,stroke-dasharray: 5 5
            
            Frontend["⚛️ Frontend<br/>(Nginx :80)"]
            Backend["🐍 Backend<br/>(Gunicorn :8000)"]
            MinIO["🪣 MinIO<br/>(S3 :9000 / UI :9001)"]
            Adminer["🗃️ Adminer<br/>(DB UI :8080)"]
            DB[("🐘 PostgreSQL<br/>(:5432)")]
            style DB fill:#336791,color:white
        end
        
        Vol_Media[("💾 Vol: media")]
        Vol_Static[("💾 Vol: static")]
        Vol_DB[("💾 Vol: pg_data")]
    end

    %% Edge Connections
    User -- "HTTPS (443)" --> Traefik
    DNS -.- "Resolves IP" -> Traefik
    
    %% Routing
    Traefik -- "smartshop1.us" --> Frontend
    Traefik -- "api.smartshop1.us" --> Backend
    Traefik -- "minio.smartshop1.us" --> MinIO
    Traefik -- "db.smartshop1.us" --> Adminer
    
    %% Internal Comms
    Frontend -. "API Calls (Client-side)" .-> Traefik
    Backend <--> DB
    Backend -- "Store Images" --> MinIO
    Adminer --> DB
    
    %% Volumes
    Backend --- Vol_Media
    Backend --- Vol_Static
    DB --- Vol_DB
```

### ASCII Version
```
════════════════════════════════════════════════════════════════
                       INTERNET (HTTPS)
════════════════════════════════════════════════════════════════
                              │
                   DNS Resolution (Cloudflare)
                              │
              ┌───────────────┴──────────────────┐
              │          smartshop1.us            │
              │    api.smartshop1.us              │
              │    www.smartshop1.us              │  → All point to VPS IP
              │    db.smartshop1.us               │
              │    minio.smartshop1.us            │
              │    s3.smartshop1.us               │
              └───────────────┬──────────────────┘
                              │
                              ▼
════════════════════════════════════════════════════════════════
              VPS — HostAsia (Ubuntu 22.04 LTS)
════════════════════════════════════════════════════════════════
                              │
              ┌───────────────▼───────────────────────────────┐
              │           TRAEFIK (Docker Container)           │
              │                                               │
              │  • Listens: port 80 (HTTP) and 443 (HTTPS)   │
              │  • HTTP → HTTPS redirect (automatic)          │
              │  • SSL via Let's Encrypt (auto-renewed)       │
              │  • Routes by hostname (Host header)           │
              │  • Reads Docker labels for routing rules      │
              └──┬───────────┬──────────┬────────┬───────────┘
                 │           │          │        │
        ┌────────▼───┐ ┌─────▼───┐ ┌───▼────┐ ┌─▼──────────┐
        │  Frontend  │ │ Backend │ │Adminer │ │   MinIO    │
        │  (Nginx)   │ │(Gunicorn│ │(DB UI) │ │(S3 Storage)│
        │            │ │)        │ │        │ │            │
        │ :80        │ │ :8000   │ │ :8080  │ │:9000/:9001 │
        │            │ │         │ │        │ │            │
        │smartshop   │ │api.smart│ │db.smart│ │minio.smart │
        │1.us        │ │shop1.us │ │shop1.us│ │shop1.us    │
        └────────────┘ └────┬────┘ └───┬────┘ └────────────┘
                            │          │
                     ┌──────▼──────────▼──┐
                     │    PostgreSQL       │
                     │   (Docker :5432)    │
                     │  Internal only —    │
                     │  no Traefik route   │
                     └────────────────────┘
```

---

## 3. Docker Network

All containers are connected to a single Docker bridge network: **`dokploy-network`**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    dokploy-network (bridge)                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container name   │ Internal hostname │ Port(s) exposed  │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  traefik          │ traefik           │ 80, 443 (public) │   │
│  │  frontend         │ frontend          │ 80 (internal)    │   │
│  │  backend          │ backend           │ 8000 (internal)  │   │
│  │  db               │ db                │ 5432 (internal)  │   │
│  │  adminer          │ adminer           │ 8080 (internal)  │   │
│  │  minio            │ minio             │ 9000, 9001 (int) │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Containers reference each other by service name:               │
│    backend → db:5432  (PostgreSQL connection)                    │
│    adminer → db:5432  (DB browser connection)                    │
│    traefik → backend:8000  (via Docker socket routing)           │
│    traefik → frontend:80   (via Docker socket routing)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Traffic Flow (Step by Step)

### 4.1 User visits `https://smartshop1.us` (Frontend)

```
1. Browser → DNS → VPS IP:443
2. Traefik receives HTTPS request
3. Traefik terminates SSL (decrypts)
4. Traefik checks Host header: "smartshop1.us"
5. Traefik matches router rule → frontend container
6. Traefik forward HTTP to frontend:80 (internal)
7. Nginx serves React index.html
8. Browser receives HTML, loads JS bundle
9. React app initializes, fetches data from api.smartshop1.us
```

### 4.2 React app calls `https://api.smartshop1.us/api/products/`

```
1. Browser → DNS → VPS IP:443
2. Traefik receives HTTPS request
3. Host header: "api.smartshop1.us" → matches backend router
4. Traefik forwards to backend:8000 (internal)
5. Gunicorn handles request → Django processes it
6. Django queries PostgreSQL at db:5432 (internal Docker network)
7. Django returns JSON response
8. Traefik returns response to browser over HTTPS
```

### 4.3 HTTP → HTTPS Redirect

```
1. Browser visits http://smartshop1.us (port 80)
2. Traefik receives on HTTP entrypoint
3. Traefik middleware: redirect HTTP → HTTPS (301)
4. Browser follows redirect to https://smartshop1.us
```

---

## 5. DNS Records

| Type | Name | Value | Purpose |
|:---|:---|:---|:---|
| A | `smartshop1.us` | `<VPS IP>` | Root domain → VPS |
| A | `www` | `<VPS IP>` | www subdomain → VPS |
| A | `api` | `<VPS IP>` | Backend API → VPS |
| A | `db` | `<VPS IP>` | Adminer DB browser → VPS |
| A | `minio` | `<VPS IP>` | MinIO console → VPS |
| A | `s3` | `<VPS IP>` | MinIO S3 API → VPS |
| MX | `@` | `mail.old-host.com` | Email remains on cPanel |

> **Split DNS**: Website on VPS, email on original cPanel host. MX records left unchanged so email was not disrupted.

---

## 6. Ports Summary

| Port | Protocol | Who Listens | Publicly Accessible? |
|:---|:---|:---|:---|
| 22 | TCP | SSH (VPS) | ✅ Yes (admin only) |
| 80 | TCP | Traefik | ✅ Yes (redirects to 443) |
| 443 | TCP | Traefik | ✅ Yes (HTTPS) |
| 5432 | TCP | PostgreSQL | ❌ Internal only |
| 8000 | TCP | Gunicorn (Django) | ❌ Internal only |
| 8080 | TCP | Adminer | ❌ Internal only (via Traefik) |
| 9000 | TCP | MinIO S3 API | ❌ Internal only (via Traefik) |
| 9001 | TCP | MinIO Console | ❌ Internal only (via Traefik) |

---

## 7. SSL / TLS Architecture

```
Browser ─────────────── TLS ─────────────► Traefik
                        (HTTPS)
                                            │ (HTTP, internal)
                                            ▼
                                        Container
```

- **Traefik** holds the SSL certificates and terminates TLS
- Containers receive plain HTTP internally (no SSL in containers)
- Django knows it's HTTPS via `X-Forwarded-Proto: https` header
- Certs stored on VPS: `/var/lib/dokploy/traefik/certs/`
- Renewal: Automatic via Let's Encrypt ACME
- Cert resolver: `letsencrypt` (configured in Traefik entrypoints)

---

## 8. Docker Volumes (Persistent Storage)

```
┌──────────────────────────────────────────────────────────────┐
│                   Docker Named Volumes                        │
├─────────────────────┬────────────────────────────────────────┤
│ Volume Name         │ Contents                               │
├─────────────────────┼────────────────────────────────────────┤
│ postgres_data       │ PostgreSQL database files              │
│ backend_static      │ Django collectstatic output (CSS/JS)   │
│ backend_media       │ Uploaded product images                │
│ frontend_build      │ React production build (dist/)         │
│ minio_data          │ MinIO object storage (backups)         │
└─────────────────────┴────────────────────────────────────────┘
```

Volume mounts per service:
- `backend` → `backend_static:/app/staticfiles`, `backend_media:/app/media`
- `db`      → `postgres_data:/var/lib/postgresql/data`
- `minio`   → `minio_data:/data`

---

## 9. Firewall Rules (UFW)

| Rule | Port | Direction | Purpose |
|:---|:---|:---|:---|
| Allow | 22/tcp | Inbound | SSH access |
| Allow | 80/tcp | Inbound | HTTP (Traefik) |
| Allow | 443/tcp | Inbound | HTTPS (Traefik) |
| Deny | All others | Inbound | Block all other ports |

> All services (PostgreSQL, Gunicorn, Adminer, MinIO) are only accessible through Traefik — never directly from the internet.

---

## 10. Dokploy Deployment Flow

```
Developer pushes to GitHub (main branch)
        │
        ▼
Dokploy detects change (webhook or manual trigger)
        │
        ▼
Docker builds new images:
  • Dockerfile.frontend  → nginx:alpine + React build
  • Dockerfile.backend   → python:slim + Django
        │
        ▼
docker-compose up --build -d
(pulls/builds, recreates containers, no downtime for DB/MinIO)
        │
        ▼
Traefik picks up new container labels automatically
        │
        ▼
New version is live at https://smartshop1.us
```

---

**Version**: 1.0.0  
**Last Updated**: February 2026  
**Infrastructure**: HostAsia VPS, Docker, Dokploy, Traefik  
