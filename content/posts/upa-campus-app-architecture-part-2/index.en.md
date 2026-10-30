---
title: "UPA Campus App (Chapter 2): Server and Docker Infrastructure"
date: 2026-10-30T16:15:00Z
description: "Containerization, port isolation, and persistence: How the UPA Campus App secures PostgreSQL, FastAPI, and Nginx with Docker Compose and why container decoupling is a core specification decision."
summary: "How do containerization, healthcheck startup chains, and persistent volumes work in practice? Chapter 2 explores Dockerfiles, port isolation, and architectural decoupling."
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

Modern web applications require reproducible runtime environments, isolated failure domains, and reliable data persistence because a fault in the user interface must never bring down the database or the API engine. The UPA Campus App fulfills these requirements through three decoupled Docker containers for the database, REST API, and web frontend. This second chapter examines how Linux containers work under the hood, how the Dockerfiles are structured, how Docker Compose coordinates startup order, and how stateless code is separated from durable storage.

---

## Containerization Compared to Virtual Machines

Why does the UPA Campus App run in Docker rather than inside traditional virtual machines (VMs) or directly on the host operating system? The core difference lies in the virtualization boundary and resource overhead:

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

Traditional virtual machines emulate physical hardware. Each VM boots an entire guest operating system with its own kernel and background daemons, consuming gigabytes of RAM per instance and taking minutes to start.

Docker containers share the host Linux kernel and rely on two native kernel primitives for isolation:
* **Namespaces:** Isolate process trees (`PID`), network interfaces (`NET`), user IDs (`USER`), and filesystem mounts (`MNT`). Each container sees only its own processes.
* **Control Groups (cgroups):** Meter and cap memory, CPU cycles, and disk I/O so that a runaway process in one service cannot starve the rest of the host server.

For a university engineering team, Docker also eliminates the classic "works on my machine" problem: whether a developer works on Windows with WSL2, macOS, or Linux, the container image locks in the exact Python version, C libraries for database drivers, and Nginx routing rules with bit-for-bit parity.

---

## Why Frontend and Backend Run in Separate Containers

In the architectural specification of the UPA Campus App (Decision 3.3: *Frontend and Backend Container Decoupling*), the editorial web portal and the FastAPI service are intentionally separated into distinct containers rather than bundled into a single image:

```mermaid
flowchart LR
    Browser["Client Browser / App"] -->|127.0.0.1:8085| Nginx["Frontend Container: Nginx (Static UI & Proxy)"]
    Nginx -->|Reverse Proxy /api/ Port 8000| FastAPI["Backend Container: FastAPI (Business Logic)"]
    FastAPI -->|SQL Port 5432| Postgres[("Database Container: PostgreSQL")]
    Postgres -->|ACID Tables| VolPG[("Host Volume: postgres_data")]
    FastAPI -->|Images, Videos & PDFs| VolUpload[("Host Volume: uploads_data")]
```

This separation provides three concrete operational advantages:
1. **Zero-Downtime UI Deployments:** When the marketing team requests a visual refinement to the editorial cockpit (`web-admin`) or the QR landing page, only the lightweight Nginx container is rebuilt and swapped out in under a second. The FastAPI container and active database connections serving Android clients continue running uninterrupted.
2. **Isolated Security Boundaries:** The Nginx container contains no Python runtime, no database credentials, and no Google service account keys. Even a hypothetical vulnerability in static asset delivery cannot read backend secrets.
3. **Zero-Mock Environment for Mobile Developers:** Because the backend container packages automatic schema migrations and representative campus seed data, Android developers can spin up the entire real stack locally with a single command (`docker compose up -d`) and build against live endpoints instead of mock stubs.

---

## Inside the Three Containers

### 1. Database: Official PostgreSQL Image

Relational storage runs on the official `postgres:16-alpine` image. Maintaining a custom Dockerfile for a standard database engine adds maintenance burden without architectural benefit. The Alpine variant keeps the image footprint below 100 MB, includes upstream security patches, and configures the database declaratively via environment variables (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`).

### 2. Backend: Custom Dockerfile with Python and FastAPI

The backend executes all REST endpoints and background jobs on top of `python:3.12-slim`:

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

Three design choices in this Dockerfile directly impact production reliability:
* **Layer Caching:** Docker caches each instruction as an immutable layer. Because `COPY requirements.txt .` and `pip install` appear before `COPY src/ ./src`, Docker reuses the installed dependencies whenever only application code changes, reducing build times from minutes to seconds.
* **Least Privilege Execution (`USER appuser`):** By default, container processes run as `root`. Switching to the unprivileged `appuser` ensures that even if a third-party library were exploited, the attacker would have no administrative privileges inside the container.
* **Automated Health Probes (`HEALTHCHECK`):** Every 15 seconds, Docker invokes `/health` inside the container. If FastAPI fails to respond three times in a row, Docker marks the container as `unhealthy`.

### 3. Frontend: Custom Dockerfile with Nginx

The frontend container packages the static portals (Web-Admin cockpit and Campus PWA) together with the Nginx reverse proxy configuration:

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

### 4. Startup Coordination with Docker Compose

What happens when all three containers boot simultaneously after a server reboot? PostgreSQL needs several seconds to verify its write-ahead logs, whereas the Python process starts almost instantaneously. Without startup coordination, FastAPI would attempt to run database migrations before PostgreSQL accepts connections, crashing immediately.

Docker Compose prevents this race condition using `condition: service_healthy`:

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
    build:
      context: .
      dockerfile: web-admin/Dockerfile
    container_name: campus-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "127.0.0.1:8085:80"
    networks:
      - campus-network
```

Docker waits until `pg_isready` confirms that PostgreSQL is actively accepting SQL queries before launching the backend container.

---

## Security Through Port Isolation

A common mistake in container deployments is publishing every internal port to the host machine (for example, mapping `5432:5432` on a production server).

```mermaid
flowchart LR
    subgraph PublicInternet["Public Internet"]
        Attacker["Automated Port Scanners"]
    end
    subgraph HostServer["Host Server"]
        NetBird["NetBird Tunnel Interface"]
        subgraph DockerBridge["Isolated Docker Bridge Network"]
            Frontend["Frontend (Port 80)"]
            Backend["Backend (Port 8000)"]
            DB[("PostgreSQL (Port 5432)")]
        end
    end
    Attacker -.->|Blocked: 0 Open Public Ports| HostServer
    NetBird -->|Access at 127.0.0.1:8085| Frontend
    Frontend -->|Internal DNS: backend:8000| Backend
    Backend -->|Internal DNS: postgres:5432| DB
```

In production, the UPA Campus App enforces two strict networking rules:
1. **No Host Port Bindings for Database and API:** Neither port 5432 nor port 8000 is exposed to the outside world in production. Containers communicate over internal Docker DNS (`postgres:5432` and `backend:8000`). Outside the `campus-network` bridge, those ports do not exist.
2. **Explicit Loopback Binding (`127.0.0.1:8085:80`):** Writing `8085:80` binds the port to all network interfaces (`0.0.0.0`) and causes the Docker daemon on Linux to insert `iptables` rules that bypass host firewalls like UFW. Prefixing the mapping with `127.0.0.1:` restricts Nginx strictly to the local loopback interface, where only the local NetBird daemon can reach it.

---

## Ephemeral Containers vs. Durable Volumes

Why does data inside a standard container vanish whenever you deploy a software update?

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Container Layer (Ephemeral)"]
        AppCode["Immutable Image (Code & Libraries)"]
        TempFS["Writable Container Layer (OverlayFS)"]
        KillAction["Container Update (docker compose up -d --build)"]
        TempFS -.->|Wiped on container replacement| KillAction
    end
    subgraph HostStorage["Host Filesystem (Durable)"]
        PGVolume[("Docker Volume: postgres_data")]
        UploadVolume[("Docker Volume: uploads_data")]
    end
    ContainerLifecycle -->|Mounted at /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Mounted at /app/uploads| UploadVolume
```

Every running container combines read-only image layers with a thin writable top layer (`OverlayFS`). When you deploy a new version via `docker compose up -d --build`, Docker destroys the old container along with its writable layer and boots a fresh container from the updated image.

To ensure that neither student registrations nor uploaded event videos are lost during deployments, Docker Compose mounts two named **Docker Volumes** from the host filesystem (`/var/lib/docker/volumes/`) into the containers:
* **`postgres_data`:** Survives every container replacement and preserves all relational tables, indexes, and transaction logs.
* **`uploads_data`:** Preserves all images (`JPG`, `PNG`, `WEBP`), videos (`MP4`, `WEBM`), and documents (`PDF`) submitted through the unified media upload field.

---

## Key Takeaways

1. **Decoupling Drives Reliability:** Running Nginx, FastAPI, and PostgreSQL in separate containers enables instant UI updates without interrupting backend API connections.
2. **Deterministic Startup Order:** Combining `pg_isready` healthchecks with `condition: service_healthy` ensures the backend never boots against an uninitialized database.
3. **Closed Ports and Durable Volumes:** Internal services communicate exclusively across the private Docker bridge network, while named host volumes guarantee zero data loss across software updates.
