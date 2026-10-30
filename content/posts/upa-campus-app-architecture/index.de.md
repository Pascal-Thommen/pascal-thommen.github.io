---
title: "UPA Campus App: Architektur und System-Engineering (Serie)"
date: 2026-10-30T17:00:00Z
description: "Vollständige 6-teilige Architekturserie der UPA Campus App: Zero-Trust Ingress mit NetBird, Nginx Reverse Proxy, PWA, FastAPI Backend und PostgreSQL Persistenz."
summary: "Die 6-teilige Architekturserie der UPA Campus App: Vom verschlüsselten Ingress-Tunnel über Container-Orchestrierung und PWA bis zum FastAPI-Backend und relationaler Persistenz."
tags: ["Softwarearchitektur", "Docker", "FastAPI", "PostgreSQL", "NetBird", "Enterprise Systems"]
categories: ["Architektur", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_title: "UPA Campus App Architektur"
series_hub: true
aliases:
  - "/posts/upa-campus-app-systemhandbuch/"
---

Die UPA Campus App bündelt akademische Ankündigungen, Vorlesungstermine und Event-Registrierungen in einer modularen Drei-Container-Infrastruktur. Diese sechsteilige Fachserie dokumentiert die vollständige Systemarchitektur: von der Netzwerksicherheit über die Container-Orchestrierung bis hin zur API-Verarbeitung und relationalen Datenpersistenz.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.11-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## Gesamtsystem und Architekturfluss

Der Datenfluss verläuft geradlinig vom Client über ein privates Overlay-Netzwerk bis in die isolierte Container-Landschaft:

```mermaid
flowchart LR
    Client["Client (PWA / Browser)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|Port 8085| Nginx["Frontend Container (Nginx / Web-Admin)"]
    Nginx -->|Statische Assets| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Persistente Tabellen| VolDB[("Volume: postgres_data")]
    Backend -->|Uploads / Medien| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

---

## Architektonische Leitprinzipien

1. **Zero-Trust Netzwerkintegrität:** Der Produktionsserver öffnet keine öffentlichen Ports. Der gesamte Ingress-Verkehr läuft über ein privates WireGuard-Mesh via NetBird.
2. **Strikte Container-Isolation:** Frontend, API-Schicht und Datenbank laufen in separaten Containern innerhalb eines dedizierten Docker-Bridge-Netzwerks.
3. **Stateless Backend mit asynchroner I/O:** FastAPI gewährleistet typsichere Validierung über Pydantic und zustandslose Request-Verarbeitung.
4. **Deterministische Datenpersistenz:** Trennung von strukturierten relationalen Daten in PostgreSQL und unstrukturierten Mediendateien in dedizierten Host-Volumes.

---
