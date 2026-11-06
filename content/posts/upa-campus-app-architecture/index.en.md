---
title: "UPA Campus Progressive Web App: System Architecture and Engineering (Series)"
date: 2026-11-06T18:00:00Z
description: "Project showcase and six-part architecture series on the UPA Campus Progressive Web App: FastAPI, PostgreSQL 16, Service Worker PWA, Google OAuth 2.0, NetBird Zero Trust Tunnel, and multimodal AI."
summary: "Central campus platform for Universidad Paraguayo Alemana (UPA). System architecture, tech stack (FastAPI, PostgreSQL 16, Service Worker PWA, NetBird), and six-part engineering deep dive."
tags: ["FastAPI", "PostgreSQL 16", "Service Worker PWA", "Google OAuth 2.0", "NetBird", "Enterprise Systems"]
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
  - "/posts/upa-campus-app-systemhandbuch/"
  - "/posts/upa-campus-pwa/"
cover:
  image: "cover.png"
  alt: "System Architecture of the UPA Campus Progressive Web App: Service Worker PWA, FastAPI, PostgreSQL 16, NetBird, and Gemini Flash"
  caption: "UPA Campus Progressive Web App: Single Source of Truth with FastAPI and PostgreSQL 16"
  relative: true
  hiddenInSingle: true
---

The UPA Campus Progressive Web App consolidates official announcements, academic event calendars, course deadlines, and emergency alerts for Universidad Paraguayo Alemana (UPA) into a single platform. Instead of scattered communication across WhatsApp groups, emails, and social media, the system establishes a relational Single Source of Truth for students, faculty, and administration.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![PostgreSQL 16](https://img.shields.io/badge/PostgreSQL%2016-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![PWA](https://img.shields.io/badge/Service%20Worker-PWA-5A0FC8?style=flat-square&logo=pwa&logoColor=white)
![NetBird](https://img.shields.io/badge/NetBird-Zero%20Trust%20Tunnel-1A202C?style=flat-square&logo=wireguard&logoColor=white)
![Google OAuth 2.0](https://img.shields.io/badge/Google%20OAuth%202.0-4285F4?style=flat-square&logo=google&logoColor=white)

---

## Project Profile and Responsibilities

* **Project:** UPA Campus Progressive Web App (`v1.0.0`)
* **Role:** Project Lead, System Architecture, and Backend Implementation
* **Tech Stack:** FastAPI (Python 3.12), PostgreSQL 16, Service Worker PWA (`sw.js`), Google OAuth 2.0 (`@upa.edu.py`), NetBird Zero Trust Tunnel (WireGuard), Nginx Single-Origin Reverse Proxy
* **Core Capabilities:** Mobile First Campus PWA, automated event extraction via multimodal AI (Google Gemini Flash), Google Classroom synchronization, Web Push notifications (Firebase Cloud Messaging HTTP v1)
* **Source Code and Documentation:** [GitHub Repository (Pascal-Thommen/upa-app)](https://github.com/Pascal-Thommen/upa-app)

![System Architecture of the UPA Campus Progressive Web App](architecture-diagram.svg)

---

## Core Engineering Capabilities

### 1. Mobile First Campus PWA with Service Worker
The student and faculty client is built as a cross-platform Progressive Web App (`web-app/`) because a single web codebase eliminates commercial app store licensing overhead and delivers instant updates across Android and iOS via Service Worker (`/sw.js`). Local App Shell caching keeps the feed and academic calendar accessible even in lecture halls with unstable mobile reception. On desktop screens, an automatic viewport switch presents a visual device preview with a dynamic QR code for immediate mobile installation.

### 2. Automated Event Extraction via Multimodal AI
University events are frequently published as visual posters on Instagram (`@upalemana`). To integrate these events into the structured campus calendar without manual re-entry, the backend (`backend/src/services/ai_extractor.py`) processes synchronized posters and captions through Google Gemini Flash, extracting event titles, dates, times, and locations into validated JSON fields. Inside the editorial cockpit (`/dashboard/publicaciones`), the same pipeline powers 1-click form autocompletion (`Autocompletar con IA`), paired with client-side WebP image compression (1080px max edge) and a three-state review badge (`No revisado`, `IA`, reviewer first name).

### 3. Google OAuth 2.0 and Google Classroom Synchronization
Access to internal campus resources is cryptographically restricted to verified `@upa.edu.py` accounts via Google OAuth 2.0 and signed JWT session tokens (`HS256`). Within the *Agenda* module, the backend merges institutional events with personal assignment deadlines fetched from the Google Classroom API in the `America/Asuncion` timezone. Automatic OAuth token refresh and a persistent PostgreSQL fallback cache (`classroom_deadlines_cache`) ensure high availability even during temporary upstream API outages.

### 4. Push Notifications and NetBird Zero Trust Tunnel
Time-critical alerts (`EMERGENCY_BANNER`, such as severe weather closures) and event reminders are dispatched as native Web Push notifications via Firebase Cloud Messaging (FCM HTTP v1), including automated cleanup of unregistered device tokens. At the network boundary, the campus server exposes zero inbound firewall ports: an outbound NetBird WireGuard tunnel securely connects the loopback-bound Nginx reverse proxy (`127.0.0.1:8085`) to the public ingress endpoint.

---

## The Learning Path Across Six Architectural Layers

Using the UPA Campus Progressive Web App as a production reference system, this series dissects a modern university platform systematically from the outside in. Each chapter focuses on one dedicated architectural layer and explains which concrete operational challenges it solves:

```mermaid
flowchart TD
    K1["Chapter 1: Mental Model & Domain\nFrom Functional Specification to Layered Architecture"] --> K2["Chapter 2: Server & Containers\nProcess Isolation, Dockerfiles, Healthchecks & Volumes"]
    K2 --> K3["Chapter 3: Network Boundary & Ingress\nZero-Port WireGuard Tunnels & Single-Origin Routing"]
    K3 --> K4["Chapter 4: Client Engineering\nService Worker PWA, Offline Resilience & State-Driven UI"]
    K4 --> K5["Chapter 5: Backend & API Contracts\nPydantic Validation, Multimodal AI & Least Privilege"]
    K5 --> K6["Chapter 6: Relational Persistence\nPostgreSQL Schema, Database Constraints, Media Streaming & Alembic"]
```

---
