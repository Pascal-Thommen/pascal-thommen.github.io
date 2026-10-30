---
title: "UPA Campus App: Architecture and System Engineering (Series)"
date: 2026-10-30T17:00:00Z
description: "Complete 6-part architecture series of the UPA Campus App: How modern web applications are engineered from the ground up and why every functional specification rule stems from architectural necessity."
summary: "From encrypted ingress tunnels and container orchestration to PWAs, FastAPI backends, and relational persistence: A practical engineering guide to web architecture and specification design."
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

Modern web applications are not built by merely stitching user interfaces together, but by orchestrating network security, container isolation, stateless business logic, and transactional persistence. In an era where AI assistants generate syntax in seconds, engineering success no longer depends on memorizing loop constructs or CSS properties, but on holistic system thinking: system boundaries, interface contracts, data flows, and relational data models. Using the UPA Campus App as a concrete reference system, this six-part series conveys a universal mental model for modern web applications and demonstrates why every requirement in a functional specification acts as a logical blueprint derived from system architecture.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.12-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## System Overview and Architectural Flow

A single interaction on a smartphone or desktop portal traverses four strictly isolated layers within milliseconds, moving from the encrypted ingress tunnel down to the persistent storage engine:

```mermaid
flowchart LR
    Client["Client (PWA / Mobile App)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|127.0.0.1:8085| Nginx["Frontend Container (Nginx)"]
    Nginx -->|Static Assets & App Shell| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Relational Tables| VolDB[("Volume: postgres_data")]
    Backend -->|Media Files| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

---

## Why Architecture and Functional Specifications Are Inseparable

In software projects, a functional specification is often mistaken for an arbitrary wishlist. In a resilient web architecture, the opposite holds true: every product requirement protects the system against inconsistent state, security vulnerabilities, or resource exhaustion.

1. **Zero-Trust Networking and Identity:** The production host exposes zero public router ports, routing all ingress traffic through an encrypted WireGuard mesh (NetBird) to the local Nginx proxy. At the application layer, this security posture continues through mandatory Google OAuth authentication for institutional accounts and strict least-privilege role management: new accounts initialize with all permission checkboxes unchecked, and development login shortcuts are completely absent in production.
2. **Strict Container and Client Decoupling:** The platform isolates static delivery (Nginx), core API logic (FastAPI), and relational storage (PostgreSQL) in separate containers. Consequently, a single unified backend serves both the desktop editorial cockpit and the mobile student interface featuring local offline storage and Google Classroom deadline synchronization.
3. **Single Source of Truth in the Backend:** Instead of scattering campus news, calendar events, and emergency alerts across separate silos, a unified publication model (`/api/publications`) handles all formats in a single structure. Functional rules follow directly from this model: pinned items (`is_pinned`) require a mandatory expiration timestamp (`expires_at`) so outdated weather warnings never clutter the top of the feed permanently. Posts synchronized from Instagram remain read-only in the editorial portal because periodic background synchronization would otherwise overwrite manual edits on the next cycle.
4. **Deterministic Data Persistence:** Structured metadata, unique read receipts (`PostView`), and one-click event registrations (`EventRegistration`) reside with full ACID guarantees in PostgreSQL, whereas uploaded images, videos, and PDF documents bypass the database buffer pool entirely via a dedicated host volume.

---
