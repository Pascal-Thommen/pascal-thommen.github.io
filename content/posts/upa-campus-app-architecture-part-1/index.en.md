---
title: "UPA Campus App (Chapter 1): Big Picture and System Overview"
date: 2026-10-30T16:00:00Z
description: "How does a modern web application work from the ground up? Chapter 1 explains the three-tier architecture of the UPA Campus App and shows how the functional specification emerges directly from system design."
summary: "How do you build a production campus platform on lean infrastructure? Chapter 1 connects universal three-tier web architecture with the concrete modules of the functional specification."
tags: ["Software Architecture", "Docker", "FastAPI", "PostgreSQL", "Enterprise Systems"]
categories: ["Architecture", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 1
series_part_title: "Big Picture and System Overview"
hiddenInList: true
---

Every resilient web application separates presentation, business logic, and data persistence into independent layers because a single monolithic process cannot be secured or maintained cleanly as complexity grows. The UPA Campus App implements this separation across three isolated Docker containers behind an encrypted NetBird ingress tunnel. This first chapter establishes a foundational understanding of modern web architecture and demonstrates how every functional module in the specification builds directly on this division of labor.

---

## The Anatomy of a Modern Web Application

When a student checks their smartphone in the morning to see whether severe weather has suspended in-person lectures or when the next assignment is due in Google Classroom, the application feels like a single program. Under the hood, however, every interaction triggers a coordinated handoff across four specialized components:

```mermaid
flowchart LR
    Client["Client (Mobile App / Web-Admin)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|127.0.0.1:8085| Nginx["Frontend Container (Nginx)"]
    Nginx -->|Static Assets & App Shell| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Relational Tables| VolDB[("Volume: postgres_data")]
    Backend -->|Media Files| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

A helpful analogy is a well-run university restaurant:
* **Frontend (Nginx & Browser / App):** The dining room and the menu. Guests browse options and place orders here, but no food is cooked and no ledgers are kept at the table.
* **Reverse Proxy (Nginx Routing):** The host at the front desk. The host inspects every incoming request, hands out printed brochures (static files and images) immediately, and routes meal orders to the kitchen.
* **Backend (FastAPI):** The kitchen. It verifies whether the guest is authorized to order (authentication and role checks), validates that the order ticket is complete (schema validation), and prepares the dish according to strict recipes (business logic).
* **Database (PostgreSQL & Volumes):** The walk-in vault and accounting ledger. All ingredients and records are stored here durably and locked away from the public. Only the kitchen holds a key to the vault.

---

## Why Every Layer Serves an Exclusive Responsibility

1. **NetBird Ingress Tunnel:** Establishes an encrypted WireGuard mesh network. The production server opens zero public ports to the internet because traffic enters exclusively through authenticated peer-to-peer tunnels bound to the local loopback interface (`127.0.0.1:8085`).
2. **Frontend Container (Nginx):** Delivers the minimalist desktop landing page (displaying a single QR code for mobile installation alongside store-style app previews), the desktop editorial portal for the marketing team, and the mobile PWA. Simultaneously, Nginx operates as a reverse proxy, forwarding `/api/` traffic internally to the backend and `/uploads/` requests to the media volume.
3. **Backend Container (FastAPI):** Processes REST requests asynchronously, validates every payload via Pydantic, and enforces the business rules of the functional specification. This includes mandatory Google OAuth login restricted to institutional accounts (`@example-campus.edu`), synchronization with the Google Classroom API and Google Workspace Directory, and automated ingestion of the official Instagram feed.
4. **Database Container (PostgreSQL):** Stores structured relational entities such as user profiles, audience groups, unified publications, one-click event registrations (`EventRegistration`), unique read receipts (`PostView`), and the immutable administrative audit trail (`AdminAuditLog`) with full ACID guarantees.

---

## From System Architecture to the Functional Specification: A Universal Mental Model

For business informatics engineers and modern AI-assisted builders (vibe-coders), building production software is not about memorizing loop syntax or CSS spacing. What matters is translating business requirements from a functional specification into four architectural building blocks:
1. **System Boundary and Ingress:** Who is allowed to reach the system over which encrypted network path?
2. **Client State and Presentation:** Which data is cached locally on the device for instant offline responsiveness, and which interactions require server synchronization?
3. **Interface Contract and Domain Logic:** Which invariants must the API enforce before mutating state or querying external systems (such as Google Classroom or Instagram)?
4. **Relational Persistence:** Which database tables and integrity constraints (such as uniqueness rules against duplicate bookings) guarantee durable state?

Applying this mental model to the UPA Campus App reveals immediately why the four core modules of the functional specification are structured the way they are:

```mermaid
flowchart TD
    subgraph Spec["Functional Specification Modules"]
        M1["Home Module (Feed): Emergency Banners, News, 1-Click RSVP & Instagram"]
        M2["Agenda Module: Chronological List, Monthly Calendar & Classroom"]
        M3["Profile Module: Institutional Identity (@example-campus.edu) & Offline Control"]
        M4["Web-Admin Module: Unified Editor, Alphabetical Groups & Role Matrix"]
    end
    subgraph Arch["Architectural Implementation Across Layers"]
        A1["Unified Post Model in PostgreSQL + Read-Only Protection for Instagram Sync"]
        A2["Dual Source: Internal Events from PostgreSQL + Live Deadlines via Google Classroom OAuth"]
        A3["Google OAuth 2.0 Domain Check in Backend + Cache Storage / Room on Client"]
        A4["Google Directory API Cache in Backend + Least-Privilege Roles Unchecked by Default"]
    end
    M1 --> A1
    M2 --> A2
    M3 --> A3
    M4 --> A4
```

* **Why use a single unified publication model instead of separate tables for news and events?** In campus operations, a single item is frequently both a news announcement and a calendar event (for example, a guest lecture with an article body, poster, auditorium location, start time, and 1-click RSVP). Because the backend feeds both the news stream and the calendar from the same database table (`posts` via `/api/publications`), the system maintains a strict *Single Source of Truth*: when the marketing team updates a room number, the feed card, fullscreen modal, and calendar entry update simultaneously.
* **Why does the campus feed distinguish between unauthenticated and authenticated modes?** Prior to login, the `/api/v1/feed` endpoint returns only public Instagram posts from the university account. Only after presenting a valid JWT obtained via Google OAuth does the backend deliver internal announcements, degree-specific events, and urgent emergency banners (`EMERGENCY_BANNER`). This keeps the initial screen welcoming while protecting internal university communications.
* **Why integrate Google Classroom directly via OAuth tokens in the Agenda module?** Professors already maintain assignment deadlines inside Google Classroom. Asking faculty to re-enter deadlines manually into a second portal would guarantee outdated schedules within weeks. Instead, the backend stores an authorized Classroom token upon login and merges live academic deadlines with internal campus events on the fly.

---

## Core Infrastructure Concepts

<details open>
<summary>Glossary of Infrastructure Primitives</summary>

* **Container:** An isolated runtime unit that packages application code together with all system dependencies. Because containers share the host Linux kernel, they start in milliseconds and consume a fraction of the memory required by virtual machines.
* **Port:** A numbered network endpoint used to address a specific service (such as port 5432 for PostgreSQL or port 8000 for FastAPI). In a hardened architecture, internal service ports are reachable only inside the private container network.
* **Reverse Proxy:** A front-facing web server (Nginx) that receives all incoming client requests at the perimeter and routes them by URL path (`/api/`, `/uploads/`, `/dashboard/`) to the appropriate internal service.
* **Volume:** A persistent storage directory on the host disk mounted into a container so that database tables and uploaded media files survive container restarts and image upgrades.

</details>

---

## Container Orchestration and Storage Strategy

Across a university project, developers and administrators work on different operating systems (Windows, macOS, Linux). Without containerization, mismatched Python versions or missing system libraries cause frequent deployment failures. Docker Compose wires the three services declaratively inside an isolated bridge network:

```yaml
# docker-compose.yml (excerpt with anonymized placeholders)
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
      ALLOWED_EMAIL_DOMAIN: example-campus.edu
      APP_ENV: production
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

Because containers themselves are stateless and recreated on every deployment, persistent state is segregated into two named host volumes:
1. **Relational Data (`postgres_data`):** Persists tables, indexes, and write-ahead logs under `/var/lib/postgresql/data`.
2. **Media Assets (`uploads_data`):** Stores uploaded event posters, videos (`MP4`, `WEBM`), and PDF documents under `/app/uploads`. Large binary files are intentionally kept out of PostgreSQL BLOB columns so they do not evict index pages from database RAM or bloat daily database backups.

---

## Key Takeaways

1. **Clean Layer Separation over Monoliths:** Splitting the system into an Nginx proxy, a FastAPI backend, and a PostgreSQL database allows each tier to be updated, tested, and secured independently.
2. **The Specification as a Logical Consequence of Architecture:** Product rules such as the unified publication model (`Single Source of Truth`), live Google Classroom synchronization, and authenticated feed filtering stem directly from avoiding data duplication and security exposure.
3. **Stateless Containers, Durable Volumes:** Application containers remain disposable at any time, while relational records and uploaded media stay safely persisted in dedicated host volumes.
