---
title: "UPA Campus App: Architecture and Container Infrastructure"
date: 2026-10-30T16:00:00Z
description: "Production system architecture of the UPA Campus App: From NetBird Ingress Tunnel through Nginx and FastAPI to PostgreSQL persistence."
summary: "How is a modern campus platform structured? Part 1 analyzes the interplay between PWA, Nginx reverse proxy, FastAPI backend, and PostgreSQL volumes."
tags: ["Software Architecture", "Docker", "FastAPI", "PostgreSQL", "Enterprise Systems"]
categories: ["Architecture", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
draft: false
---

How do you design a reliable campus platform for students and faculty using minimal infrastructure? The UPA Campus App isolates frontend delivery, API logic, and data persistence across three lightweight Docker containers, secured behind an encrypted NetBird ingress tunnel. This guide breaks down the core architecture, the container topology, and the persistence model for relational records and media files.

---

## Architecture Overview

Traffic flows from the client through a private overlay network into an isolated container cluster:

```mermaid
flowchart LR
    Client["Client (PWA / Browser)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|Port 8085| Nginx["Frontend Container (Nginx / Web-Admin)"]
    Nginx -->|Static Assets| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Persistent Tables| VolDB[("Volume: postgres_data")]
    Backend -->|Uploads / Media| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

---

## Component Roles and Responsibilities

Each layer in the stack handles a single responsibility:

1. **NetBird Ingress Tunnel:** Creates an encrypted WireGuard mesh network. The host exposes no public ports to the open internet because traffic enters exclusively through authenticated peer-to-peer tunnels.
2. **Frontend Container (Nginx):** Serves the administrative PWA as precompiled HTML, CSS, and JavaScript. Nginx also serves as the reverse proxy: requests targeting `/api/` route internally to the backend, while requests to `/uploads/` stream persisted media files. The client interacts with a single unified domain.
3. **Backend Container (FastAPI):** Exposes the REST API, enforces schema validation via Pydantic, and executes domain business logic. This includes Google OAuth verification for institutional accounts (`@example-campus.edu`), role-based authorization, and academic event processing.
4. **Database Container (PostgreSQL):** Stores relational records including user profiles, permission matrices, schedules, and announcements with full ACID guarantees on an Alpine Linux base image.

---

## Core Infrastructure Concepts

<details open>
<summary>Glossary of Infrastructure Primitives</summary>

* **Container:** An isolated execution unit packaging application code and dependencies. Containers share the host kernel, enabling sub-second startups and minimal memory overhead compared to virtual machines.
* **Port:** A logical network construct used to direct traffic. Port forwarding maps an external host port (such as 8085) to an internal container port (such as 80).
* **Reverse Proxy:** An intermediate server that receives external traffic and routes it to backend services. It abstracts internal topologies, manages SSL termination, and consolidates endpoints under a single origin.
* **Volume:** A dedicated host-managed filesystem mount independent of container lifecycles, ensuring state persists across container rebuilds and updates.

</details>

---

## Container Infrastructure

Why run the UPA Campus App in Docker? Engineering teams across universities work on diverse operating systems (Linux, macOS, Windows). Uncontainerized setups lead to package drift, mismatched Python interpreters, and divergent database configurations. Docker enforces runtime parity between development machines and production nodes.

All three services run in an isolated bridge network:

```yaml
# docker-compose.yml (simplified layout with neutral placeholders)
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
    networks:
      - campus-network

  backend:
    build: ./backend
    container_name: campus-backend
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DATABASE_URL: postgresql://db_user:secret_password@postgres:5432/app_db
      APP_ENV: production
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
      - "8085:80"
    networks:
      - campus-network

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

Services discover one another via internal Docker DNS (`postgres:5432`, `backend:8000`). Neither PostgreSQL nor FastAPI binds ports to the host interface directly, keeping the network perimeter closed.

---

## Data Persistence Strategy

Containers are stateless. Rebuilding an image purges its internal filesystem. Reliable production workloads separate ephemeral compute from persistent state:

1. **Structured Data via Database Volumes (`postgres_data`):** PostgreSQL persists tables, indexes, and write-ahead logs to `/var/lib/postgresql/data`. Mounting this path to a named Docker volume preserves database state across upgrades.
2. **Unstructured Media via Asset Volumes (`uploads_data`):** Images, event banners, and documents live under `/app/uploads`. Avoiding binary blobs inside PostgreSQL prevents database bloat and cache invalidation bottlenecks. PostgreSQL stores relative file paths, while Nginx handles file delivery with cache headers.

---

## Key Takeaways

1. **Clear Layer Decoupling:** Isolating reverse proxy, API, and database layers prevents monolithic entanglements and enables independent updates.
2. **Stateless Compute, Stateful Volumes:** Containers remain expendable, while relational state and uploads stay protected in dedicated host volumes.
3. **Attack Surface Minimization:** Internal services communicate across a private Docker bridge, with NetBird eliminating the need for open public ports.
