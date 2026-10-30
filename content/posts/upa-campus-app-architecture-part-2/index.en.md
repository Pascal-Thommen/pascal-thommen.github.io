---
title: "UPA Campus App (Chapter 2): Server and Docker Infrastructure"
date: 2026-10-30T16:15:00Z
description: "Containerization, port isolation, and persistence: How the UPA Campus App secures PostgreSQL, FastAPI, and Nginx with Docker Compose."
summary: "How do containerization, port isolation, and data persistence work in production? Chapter 2 dives into Dockerfiles, Compose orchestration, and storage strategies."
tags: ["Docker", "Software Architecture", "PostgreSQL", "FastAPI", "Enterprise Systems"]
categories: ["Architecture", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 2
series_part_title: "Server and Docker Infrastructure"
hiddenInList: true
---

Modern web applications demand reproducible runtimes, isolated failure domains, and reliable data persistence. The UPA Campus App addresses these requirements through three modular Docker containers for database, REST API, and web frontend. This second part examines Dockerfile design, declarative orchestration via Docker Compose, port isolation mechanisms, and persistent data storage.

---

## Containerization Compared to Virtual Machines

Why does the UPA Campus App rely on Docker instead of traditional virtual machines? The key distinction lies in the virtualization layer and resource overhead:

```mermaid
flowchart TD
    subgraph VM_Arch["Virtual Machines (Hardware Virtualization)"]
        direction TB
        AppVM["Application & Libraries"]
        GuestOS["Full Guest Operating System"]
        Hypervisor["Hypervisor"]
        HostHW1["Host System & Hardware"]
        AppVM --> GuestOS --> Hypervisor --> HostHW1
    end
    subgraph Docker_Arch["Containers (OS Virtualization)"]
        direction TB
        AppDocker["Application & Libraries"]
        Engine["Docker Engine"]
        SharedKernel["Shared Linux Kernel"]
        HostHW2["Host System & Hardware"]
        AppDocker --> Engine --> SharedKernel --> HostHW2
    end
```

Traditional virtual machines (VMs) virtualize physical hardware. Every VM requires a full guest operating system with a dedicated kernel, memory management, and background system services. This binds multiple gigabytes of memory per instance and incurs boot times measured in minutes.

Docker containers, in contrast, share the Linux kernel of the host system. Docker relies on two native Linux kernel features:
* **Namespaces:** Isolate process trees (PID), network interfaces (NET), user IDs (USER), and filesystem mounts (MNT). A container sees only its own processes and cannot inspect other containers.
* **Control Groups (cgroups):** Restrict and monitor hardware resources such as memory, CPU time, and disk I/O. A memory leak or runaway job in one container cannot exhaust the host server.

Containers start in fractions of a second and consume only the memory and CPU cycles that the actual application process needs.

Furthermore, Docker eliminates environment drift across development workstations. Across macOS, Linux, or Windows with WSL2, the container encapsulates the Python runtime, system packages, Nginx configurations, and database schemas with bit-for-bit parity. The local development environment matches the production server exactly.

---

## The Three Containers of the UPA Campus App

The campus infrastructure is divided into three specialized containers communicating across a shared Docker network.

```mermaid
flowchart LR
    Browser["Client Browser"] -->|Port 8085| Nginx["Frontend Container: Nginx / PWA"]
    Nginx -->|Reverse Proxy /api/ Port 8000| FastAPI["Backend Container: FastAPI"]
    FastAPI -->|SQL Port 5432| Postgres[("Database Container: PostgreSQL")]
    Postgres -->|ACID Persistence| VolPG[("Host Volume: postgres_data")]
    FastAPI -->|Media & Documents| VolUpload[("Host Volume: uploads_data")]
```

### 1. Database: Official PostgreSQL Image

The system utilizes the official `postgres:16-alpine` image for relational storage. Choosing an official image provides critical advantages:
* No overhead for maintaining database compilation scripts or security patches.
* Alpine Linux keeps the base image footprint under 100 MB.
* Configuration is entirely declarative through environment variables (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`).

Writing custom Dockerfiles for standard relational databases is generally an anti-pattern that increases maintenance burden without architectural benefit.

### 2. Backend: Custom Dockerfile with Python and FastAPI

The backend handles business logic and serves the REST API. The build is based on `python:3.12-slim`:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app \
    UPLOAD_DIR=/app/uploads

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN addgroup --system appgroup && adduser --system --group appuser

COPY src/ ./src
COPY alembic/ ./alembic
COPY alembic.ini .

RUN mkdir -p /app/uploads && chown -R appuser:appgroup /app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=15s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key architectural decisions in the backend Dockerfile:
* **Optimized Layer Caching:** The copy and installation of `requirements.txt` occurs before the source code copy. When updating application code, Docker leverages the cached dependency layer, cutting build times significantly.
* **Principle of Least Privilege:** The application runs under an unprivileged user (`appuser`) instead of `root`. If a third-party dependency is compromised, an attacker lacks root privileges inside the container.
* **Built-in Healthcheck:** Docker checks the `/health` endpoint every 15 seconds. If the check fails three consecutive times, the container is marked as `unhealthy`.

### 3. Frontend: Custom Dockerfile with Nginx

The frontend delivers the administrative Progressive Web App (PWA) and coordinates traffic routing:

```dockerfile
FROM nginx:alpine

RUN rm -rf /usr/share/nginx/html/*

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=15s --timeout=5s --start-period=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1:80/frontend-health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

Nginx fulfills two responsibilities:
* **Static Asset Delivery:** Compresses HTML, CSS, JavaScript, and PWA assets using Gzip for minimal latency.
* **Reverse Proxy:** Transparently proxies incoming requests targeting `/api/` to the FastAPI backend. Clients interact solely with the frontend port, eliminating Cross-Origin Resource Sharing (CORS) complications in production.

### 4. Orchestration with Docker Compose

Docker Compose coordinates the multicontainer topology declaratively:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: campus-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: db_user
      POSTGRES_PASSWORD: secret_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U db_user -d app_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - campus-network

  backend:
    build: ./backend
    container_name: campus-backend
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      APP_ENV: production
      DATABASE_URL: postgresql://db_user:secret_password@postgres:5432/app_db
    volumes:
      - uploads_data:/app/uploads
    networks:
      - campus-network

  frontend:
    build: ./web-admin
    container_name: campus-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "127.0.0.1:8085:80"
    networks:
      - campus-network

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

A crucial operational safeguard is service synchronization:
`depends_on` configured with `condition: service_healthy` ensures that the backend container starts only after PostgreSQL is verified ready via `pg_isready`. Without this check, the API container risks boot loops during startup database migrations.

---

## Security and Ports: Strict Network Isolation

A frequent mistake in container deployments is exposing internal service ports to the host interface.

```mermaid
flowchart LR
    subgraph PublicInternet["Public Internet"]
        Attacker["Automated Scanners"]
    end
    subgraph HostServer["Host Server"]
        NetBird["NetBird Mesh Tunnel"]
        subgraph DockerBridge["Isolated Docker Bridge Network"]
            Frontend["Frontend (Port 80)"]
            Backend["Backend (Port 8000)"]
            DB[("PostgreSQL (Port 5432)")]
        end
    end
    Attacker -.->|Blocked: No Public Ports| HostServer
    NetBird -->|Tunnel Only Access| Frontend
    Frontend -->|Internal Proxy| Backend
    Backend -->|Internal SQL Query| DB
```

The UPA Campus App enforces two core network rules:

### 1. Internal Ports Remain Confined to the Bridge Network

Neither PostgreSQL port 5432 nor FastAPI port 8000 is mapped in the Compose `ports` section. They exist solely within the private `campus-network` bridge. Inter-service traffic routes over container DNS names (`postgres:5432`, `backend:8000`). Neither the host operating system nor external networks can address these ports directly.

### 2. Strict Loopback Interface Binding

The frontend binds port 80 using `127.0.0.1:8085:80` rather than the default `8085:80`.

By default, Docker publishes ports to `0.0.0.0`. On Linux, the Docker daemon modifies host routing tables (`iptables`), often bypassing host firewall rules like UFW. Binding explicitly to `127.0.0.1` ensures that the port is accessible only through localhost. External ingress is routed strictly through an encrypted NetBird WireGuard mesh tunnel, leaving zero public ingress ports open to the internet.

---

## Persistence and Volumes: Ephemeral Containers vs. Durable Storage

Containers adhere to the twelve-factor methodology: they are stateless and disposable.

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Container Layer (Ephemeral)"]
        AppCode["Application Code & Runtime"]
        TempFS["Writable Container Layer (UnionFS)"]
        KillAction["Container Rebuild / Redeploy"]
        TempFS -.->|Destroyed on recreation| KillAction
    end
    subgraph HostStorage["Host System (Durable)"]
        PGVolume[("Docker Volume: postgres_data")]
        UploadVolume[("Docker Volume: uploads_data")]
    end
    ContainerLifecycle -->|Mounted at /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Mounted at /app/uploads| UploadVolume
```

### The Ephemeral Container Filesystem

Every running container has an isolated, writable storage layer (OverlayFS / UnionFS). Any files written into the container filesystem during execution reside strictly within this thin layer.

When a container is deleted or rebuilt via `docker compose down` and `docker compose up -d`, this writable layer is permanently discarded. If PostgreSQL wrote database files to standard container paths, any application update would cause total data loss.

### Named Docker Volumes for Persistence

Docker volumes decouple data storage from container lifecycles:
* **Separation of Concerns:** The container holds immutable application binaries, while stateful data resides safely in the host storage pool (`/var/lib/docker/volumes/`).
* **Lifecycle Independence:** The named volume `postgres_data` persists across container recreation. Rebuilding an image with `docker compose up -d --build` replaces the container without affecting the volume mounted at `/var/lib/postgresql/data`.
* **High Performance:** Volumes bypass the copy-on-write overhead of the container filesystem, delivering bare-metal disk throughput.

### Volume Segregation: Relational Records and Uploads

The architecture isolates persistence across two designated volumes:
1. `postgres_data`: Houses database table files, transaction write-ahead logs, and indexes.
2. `uploads_data`: Stores user-submitted files such as documents and images uploaded via the backend.

Segregating media files from database storage avoids bloated database backups and preserves optimal cache efficiency in PostgreSQL.

---

## Key Takeaways

1. **Guaranteed Parity:** Lean base images, unprivileged users, and layered Dockerfiles eliminate environment inconsistencies between local development and production.
2. **Defensive Network Topology:** Internal ports remain isolated inside the bridge network, while localhost binding combined with mesh tunneling blocks direct internet attacks.
3. **Stateless Compute with Durable Storage:** Stateless containers permit safe zero-downtime updates, while named host volumes reliably safeguard relational data and binary assets.
