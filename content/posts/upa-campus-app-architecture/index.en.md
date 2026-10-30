---
title: "UPA Campus App: Architecture and System Engineering (Series)"
date: 2026-10-30T17:00:00Z
description: "Complete 6-part architecture series of the UPA Campus App: Zero-Trust Ingress with NetBird, Nginx reverse proxy, PWA, FastAPI backend, and PostgreSQL persistence."
summary: "The 6-part architecture series of the UPA Campus App: From encrypted ingress tunnels and container orchestration to PWA, FastAPI backend, and relational persistence."
tags: ["Software Architecture", "Docker", "FastAPI", "PostgreSQL", "NetBird", "Enterprise Systems"]
categories: ["Architecture", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_title: "UPA Campus App Architecture"
series_hub: true
aliases:
  - "/en/posts/upa-campus-app-systemhandbuch/"
---

The UPA Campus App centralizes academic announcements, schedules, and event registrations within a modular three-container infrastructure. This six-part technical series documents the end-to-end system architecture: from network security and container orchestration to asynchronous API processing and relational data persistence.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.11-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## System Overview and Architectural Flow

The end-to-end data flow connects the client through a private overlay network into an isolated container cluster:

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

## Architectural Core Principles

1. **Zero-Trust Network Integrity:** The production server exposes zero public router ports. All ingress traffic traverses a private WireGuard mesh via NetBird.
2. **Strict Container Isolation:** Frontend, API layer, and database run in separate containers within a dedicated Docker bridge network.
3. **Stateless Backend with Asynchronous I/O:** FastAPI enforces strict type validation via Pydantic and stateless request handling.
4. **Deterministic Data Persistence:** Clear segregation between structured relational records in PostgreSQL and unstructured media files in dedicated host volumes.

---
