---
title: "UPA Campus App (Chapter 2): Server and Container Architecture"
date: 2026-10-30T16:15:00Z
description: "Why does the UPA Campus App run across three separate Docker containers? Chapter 2 explains OS-level virtualization, Dockerfiles, healthcheck boot chains, and persistent volumes."
summary: "How do Docker containers, healthchecks, and volumes ensure reliable operation without data loss? Chapter 2 examines the technical design of the server infrastructure."
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
series_part_title: "Server and Container Architecture"
hiddenInList: true
---

How do you ensure that a web application runs identically on every student developer's laptop and on the production server, while guaranteeing that a frontend issue cannot bring down the database? By packaging the web server, business logic, and database engine into three isolated Linux containers backed by dedicated storage volumes. This second chapter explains OS-level containerization, the structure of production-ready Dockerfiles, coordinated startup sequences after a server reboot, and the separation between disposable application containers and durable data volumes.

---

## OS-Level Virtualization Versus Heavyweight Virtual Machines

In university software teams, developers routinely work across different operating systems (Windows, macOS, Linux). Installing Python or PostgreSQL directly on the host machine quickly leads to mismatched library versions and deployment bugs that only appear on specific computers.

Traditional virtual machines (VMs) solve this by emulating entire hardware stacks, but each VM requires its own full guest operating system and kernel. That overhead consumes gigabytes of RAM and slows boot times to minutes. Docker containers share the host server's Linux kernel instead:

```mermaid
flowchart TD
    subgraph VM_Stack["Traditional Virtual Machine"]
        direction TB
        V1["Application & Libraries"] --> V2["Full Guest OS (Dedicated Kernel)"]
        V2 --> V3["Hypervisor"] --> V4["Server Hardware"]
    end
    subgraph Container_Stack["Docker Containerization"]
        direction TB
        C1["Application & Libraries"] --> C2["Docker Engine"]
        C2 --> C3["Shared Linux Kernel (Namespaces & cgroups)"] --> C4["Server Hardware"]
    end
```

Two native Linux kernel features make this lightweight isolation possible:
* **Namespaces:** Give each container an isolated view of process trees (`PID`), network interfaces (`NET`), and filesystem mounts (`MNT`). A container cannot inspect processes belonging to sibling containers.
* **Control Groups (cgroups):** Meter and cap memory and CPU usage so a resource-heavy query cannot starve the rest of the host.

---

## Why the Specification Mandates Three Separate Containers

The architectural specification of the UPA Campus App deliberately splits the web portal (`campus-frontend`), the API server (`campus-backend`), and the relational database (`campus-postgres`) into three distinct containers rather than bundling them into one image:

```mermaid
flowchart LR
    UI["Container 1: campus-frontend\nNginx (Web-Admin & PWA)"] -->|Internal API Calls| API["Container 2: campus-backend\nPython 3.12 & FastAPI"]
    API -->|SQL Transactions| DB[("Container 3: campus-postgres\nPostgreSQL 16 Alpine")]
```

This three-part split addresses three practical engineering requirements:
1. **Zero-Downtime UI Deployments:** When the marketing team requests a visual adjustment to the editorial portal, only the lightweight Nginx container is rebuilt and swapped out in a fraction of a second. The FastAPI backend and active mobile app connections continue running uninterrupted.
2. **Contained Attack Surface:** The frontend container holds no Python runtime, no database credentials, and no Google API keys.
3. **Zero-Mock Local Environment for Mobile Developers:** Because the backend container bundles automatic schema migrations and seed data, Android developers can spin up a complete API server locally with a single command (`docker compose up -d`) and build against real endpoints instead of static mocks.

---

## Inside the Backend Dockerfile: Build Speed, Privileges, and Self-Monitoring

While the database uses the official `postgres:16-alpine` image, a custom `Dockerfile` defines the runtime environment for the FastAPI backend:

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

Three design choices inside this file matter for production reliability:
* **Deliberate Layer Caching:** Docker caches each instruction as an immutable layer. Placing `COPY requirements.txt .` and `pip install` before `COPY src/ ./src` ensures that Docker reuses the installed Python packages whenever only application code changes, cutting build times from minutes to seconds.
* **Dropping Root Privileges (`USER appuser`):** By default, container processes run as `root`. Switching to the unprivileged system user `appuser` ensures the Python process cannot modify system binaries inside the container.
* **Active Liveness Probes (`HEALTHCHECK`):** A running process ID does not guarantee that the web server is responsive. Every 15 seconds, Docker invokes `/health` inside the container and flags the instance as `unhealthy` if FastAPI fails to respond.

---

## Coordinated Startup Chains and Persistent Volumes in Docker Compose

When a server reboots after maintenance, all containers launch at the same time. Because PostgreSQL needs several seconds to verify its transaction logs while Python starts almost instantaneously, an uncoordinated backend would immediately fail to connect to the database and enter a crash loop.

In `docker-compose.yml`, a health-based dependency condition (`condition: service_healthy`) prevents this race condition:

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
      ALLOWED_EMAIL_DOMAIN: example-campus.edu
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

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

Docker waits until the `pg_isready` utility confirms that PostgreSQL is accepting SQL connections before starting the backend container.

### Disposable Containers Versus Durable Host Volumes

Deploying a software update (`docker compose up -d --build`) destroys the running container and replaces it with a fresh instance created from the new image. Any file written directly to the container's internal writable layer (`OverlayFS`) is wiped out during this replacement:

```mermaid
flowchart TD
    subgraph Ephemeral["Disposable Container Runtime (Replaced on Every Update)"]
        Code["Read-Only Image (Python, FastAPI, Nginx)"]
        Overlay["Temporary Container Layer (OverlayFS)"]
    end
    subgraph Persistent["Durable Host Storage (Survives Every Update)"]
        VolDB[("Volume: postgres_data\nTables, Indexes & Transaction Logs")]
        VolMedia[("Volume: uploads_data\nPosters, MP4/WEBM Videos & PDFs")]
    end
    Ephemeral -->|Mounted at /var/lib/postgresql/data| VolDB
    Ephemeral -->|Mounted at /app/uploads| VolMedia
```

To ensure neither student accounts nor uploaded event videos disappear during an update, the architecture mounts two named **Docker Volumes** from the host filesystem into the containers:
1. **`postgres_data`:** Stores all relational tables, indexes, and write-ahead logs outside the container lifecycle.
2. **`uploads_data`:** Persists all images, videos, and PDF documents uploaded through the editorial portal under `/app/uploads`.

---

## Key Takeaways

1. **Reproducible Environments Through Container Decoupling:** Splitting the stack into three specialized containers for frontend, backend, and database enables zero-downtime UI updates and identical runtime behavior across all developer machines.
2. **Deterministic Boot Order and Safe Persistence:** Healthchecks (`pg_isready`) prevent startup race conditions, while dedicated host volumes (`postgres_data` and `uploads_data`) allow application containers to be rebuilt at any time without losing data.
