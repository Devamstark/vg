# SmartShop — Deployment Work Summary

> ⚠️ This document contains **no passwords, API keys, or internal ports**. Refer to your private `.env` file and Dokploy Dashboard for credentials.

---

## 🌐 Live Services & URLs

| Service | URL | Purpose |
|:---|:---|:---|
| 🛍️ **SmartShop Store** | [https://smartshop1.us](https://smartshop1.us) | Main customer-facing e-commerce website |
| 🌐 **WWW Redirect** | [https://www.smartshop1.us](https://www.smartshop1.us) | Redirects to main store |
| ⚙️ **Django Admin** | [https://api.smartshop1.us/admin/](https://api.smartshop1.us/admin/) | Admin panel to manage products, orders, users, categories |
| 🔌 **REST API** | [https://api.smartshop1.us/api/](https://api.smartshop1.us/api/) | Backend API consumed by the frontend |
| 🗄️ **Adminer** | [https://db.smartshop1.us](https://db.smartshop1.us) | Web UI to browse and query the PostgreSQL database |
| 📁 **FileBrowser** | *(accessible via VPS IP — internal only)* | Browse and manage uploaded product media files | 157.90.149.223:8888
| 🪣 **MinIO Console** | [https://minio.smartshop1.us](https://minio.smartshop1.us) | Self-hosted S3-compatible object storage web UI |
| 🔗 **MinIO S3 API** | [https://s3.smartshop1.us](https://s3.smartshop1.us) | S3-compatible API endpoint used for Dokploy backups |

---

## 🖥️ Infrastructure

| Item | Value |
|:---|:---|
| **VPS Provider** | HostAsia |
| **Plan** | Budget VPS 1 (2 Core, 4GB RAM, 20GB NVMe) |
| **OS** | Ubuntu Linux |
| **Deployment Platform** | [Dokploy](https://dokploy.com) (self-hosted, open-source) |
| **Reverse Proxy** | Traefik (managed by Dokploy) |
| **SSL Certificates** | Let's Encrypt (auto-renewed by Traefik) |
| **Container Runtime** | Docker + Docker Compose |

---

## ✅ What Was Built & Deployed

### 1. 🐳 Docker Setup
- **`Dockerfile.backend`** — builds the Django API server and runs it with Gunicorn
- **`Dockerfile.frontend`** — builds the React app with Vite, then serves it with Nginx
- **`docker-compose.yml`** — defines all services: database, backend, frontend, adminer, minio

### 2. 🌐 Reverse Proxy & SSL (Traefik via Dokploy)
- Dokploy installs and manages **Traefik** as the reverse proxy
- All subdomains (`api.`, `db.`, `minio.`, `s3.`, `www.`) are routed automatically via Docker labels
- **SSL certificates** are issued and renewed automatically via Let's Encrypt — no manual work needed

### 3. ⚛️ Frontend (React + Vite → Nginx)
- React app is built at Docker image build-time with the API URL baked in
- Production build served using **Nginx** (lightweight, fast web server)
- **SPA routing** configured — all 404s fall back to `index.html` for React Router to handle
- Traefik routes `smartshop1.us` and `www.smartshop1.us` to the frontend container

### 4. 🐍 Backend (Django + Gunicorn)
- Django REST Framework API with JWT authentication
- **Gunicorn** used as the production WSGI server
- **WhiteNoise** middleware added to serve Django admin static files (CSS/JS)
- **Media files** (product images) served via Django's `serve` view — works in both dev and production
- On container start: automatically runs `migrate` and `collectstatic`
- Traefik routes `api.smartshop1.us` to the backend container

### 5. 🗃️ Database (PostgreSQL)
- **PostgreSQL 15** running in Docker with a persistent named volume
- Healthcheck ensures the backend waits for the database to be ready before starting
- **Adminer** deployed for easy database browsing at `https://db.smartshop1.us`

### 6. 🔒 Security & CORS Configuration
- `CORS_ALLOWED_ORIGINS` configured to only allow requests from `https://smartshop1.us`
- `CSRF_TRUSTED_ORIGINS` set for secure Django admin form submissions
- `SECURE_PROXY_SSL_HEADER` configured so Django correctly detects HTTPS behind Traefik
- `ALLOWED_HOSTS` restricted to only the app's domains
- All secrets stored as **environment variables** — never hardcoded

### 7. 📁 FileBrowser (Media File Manager)
- **FileBrowser** runs directly on the VPS, mounted to the backend media volume
- Provides a visual file manager (like a mini S3) to browse and manage uploaded product images
- Accessible from a browser via VPS IP (internal use only — not on a public domain)

### 8. 🪣 MinIO (Self-Hosted S3 for Backups)
- **MinIO** added as a Docker service — provides S3-compatible object storage
- Stores backup files created by Dokploy's automatic database backup feature
- Web console at `https://minio.smartshop1.us` for managing buckets and files
- S3 API endpoint at `https://s3.smartshop1.us` — used by Dokploy to push backups

---

## 🔄 Deployment Workflow

```
Developer pushes code to GitHub (main branch)
        │
        ▼
Dokploy detects change → pulls latest code
        │
        ▼
Docker builds new images (backend + frontend)
        │
        ▼
Containers restarted with updated images
        │
        ▼
Site is live at https://smartshop1.us
```

---

## �️ Architecture Overview

```
Internet (HTTPS)
      │
      ▼
┌─────────────────────────────────────────┐
│           Traefik (Reverse Proxy)        │
│           SSL via Let's Encrypt          │
└─────────────────────────────────────────┘
      │
      ├── smartshop1.us ──────────► Frontend Container (Nginx)
      │                                    │
      │                              React SPA served as
      │                              static HTML/CSS/JS
      │
      ├── api.smartshop1.us ──────► Backend Container (Gunicorn)
      │                                    │
      │                              Django REST API
      │                              Django Admin Panel
      │                              Media file serving
      │                                    │
      │                             ┌──────▼──────┐
      │                             │  PostgreSQL  │
      │                             │  (Database)  │
      │                             └─────────────┘
      │
      ├── db.smartshop1.us ───────► Adminer (DB Browser UI)
      │
      ├── minio.smartshop1.us ────► MinIO Console (S3 UI)
      │
      └── s3.smartshop1.us ───────► MinIO S3 API (Backup Storage)

FileBrowser (internal) ──────────► backend_media Docker Volume
```

---

## 💾 Manual Backup Commands

```bash
# Backup PostgreSQL database to a SQL file
docker exec <db-container-name> \
  pg_dump -U <db_user> <db_name> > backup_$(date +%F).sql

# Backup product images (media files)
tar -czf media_backup_$(date +%F).tar.gz \
  /var/lib/docker/volumes/<project>_backend_media/_data/
```

> Replace `<db-container-name>`, `<db_user>`, `<db_name>`, and `<project>` with your actual values from Dokploy.

---

## 🔑 Where Are Credentials Stored?

| Item | Location |
|:---|:---|
| Django secret key, DB password | Dokploy Dashboard → Environment Variables |
| Django superuser | Created via `createsuperuser` command |
| FileBrowser login | Managed via FileBrowser CLI on VPS |
| MinIO credentials | Dokploy Dashboard → Environment Variables |
| Database connection string | `DATABASE_URL` environment variable |
