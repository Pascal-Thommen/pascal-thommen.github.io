---
title: "UPA Campus App: Architektur und System-Engineering (Serie)"
date: 2026-10-30T17:00:00Z
description: "Vollständige 6-teilige Architekturserie der UPA Campus App: Wie moderne Web-Applikationen von Grund auf konzipiert werden und warum jede Regel im Pflichtenheft auf einer technischen Notwendigkeit beruht."
summary: "Vom verschlüsselten Ingress-Tunnel über Container-Orchestrierung und PWA bis zum FastAPI-Backend und relationaler Persistenz: Der praxisnahe Leitfaden für Web-Architektur und Pflichtenheft-Design."
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

Moderne Web-Applikationen entstehen nicht durch das bloße Aneinanderreihen von Benutzeroberflächen, sondern durch das präzise Zusammenspiel von Netzwerksicherheit, Container-Isolation, zustandsloser Geschäftslogik und transaktionaler Datenhaltung. Gerade in einer Zeit, in der KI-Assistenten Programmcode in Sekunden generieren, entscheidet nicht mehr die Kenntnis einzelner Syntax-Schleifen über den Projekterfolg, sondern das holistische Systemverständnis: Systemgrenzen, Schnittstellenverträge, Datenflüsse und relationale Datenmodelle. Am Beispiel der UPA Campus App vermittelt diese sechsteilige Serie das universelle Denkmodell moderner Web-Architekturen und zeigt, warum jede fachliche Vorgabe im Pflichtenheft als logische Blaupause aus der Systemarchitektur hervorgeht.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.12-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## Gesamtsystem und Architekturfluss

Ein Klick auf dem Smartphone oder im Web-Portal durchläuft in wenigen Millisekunden vier streng getrennte Schichten, vom verschlüsselten Ingress-Tunnel bis in den persistenten Speicherkern:

```mermaid
flowchart LR
    Client["Client (PWA / Mobile App)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|127.0.0.1:8085| Nginx["Frontend Container (Nginx)"]
    Nginx -->|Statische Assets & App Shell| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Relationale Tabellen| VolDB[("Volume: postgres_data")]
    Backend -->|Mediendateien| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

---

## Warum Architektur und Pflichtenheft untrennbar sind

In Softwareprojekten wird ein Pflichtenheft oft als bürokratische Wunschliste missverstanden. In einer belastbaren Web-Architektur ist das Gegenteil der Fall: Jede Produktregel schützt das System vor inkonsistenten Daten, Sicherheitslücken oder unkontrolliertem Ressourcenverbrauch.

1. **Zero-Trust Netzwerk und Identität:** Der Server öffnet keine öffentlichen Router-Ports, sondern leitet den Verkehr durch einen verschlüsselten WireGuard-Tunnel (NetBird) an den lokalen Nginx-Proxy. Auf Anwendungsebene setzt sich dieses Prinzip durch den exklusiven Google OAuth Login für Universitäts-Accounts und das Prinzip minimaler Rechte (Least Privilege) bei der Rollenvergabe fort: Neue Konten starten ohne vorausgewählte Berechtigungen, und Entwickler-Schnellzugänge sind im Produktivbetrieb ausgebaut.
2. **Strikte Container- und Client-Entkopplung:** Die Plattform trennt die statische Auslieferung (Nginx) vom API-Kern (FastAPI) und der Datenbank (PostgreSQL). Dadurch bedient dasselbe Backend sowohl das Desktop-Redaktionsportal als auch die mobile Studierenden-Oberfläche mit Offline-Speicher und Google Classroom Synchronisation.
3. **Single Source of Truth im Backend:** Statt Nachrichten, Kalendertermine und Eilmeldungen in getrennten Silos zu verwalten, bündelt ein vereinheitlichtes Datenmodell (`/api/publications`) alle Formate. Daraus leiten sich die fachlichen Schutzregeln direkt ab: Angeheftete Beiträge (`is_pinned`) erzwingen ein Ablaufdatum (`expires_at`), damit veraltete Sturmwarnungen den Feed nicht dauerhaft blockieren. Aus Instagram synchronisierte Beiträge bleiben im Redaktionsportal schreibgeschützt (`Read-Only`), weil der periodische Hintergrund-Sync manuelle Textänderungen sonst beim nächsten Durchlauf überschreiben würde.
4. **Deterministische Datenpersistenz:** Strukturierte Metadaten, eindeutige Lesebestätigungen (`PostView`) und 1-Klick-Anmeldungen (`EventRegistration`) liegen transaktionssicher in PostgreSQL, während hochgeladene Bilder, Videos und PDF-Dokumente das Datenbanksystem nicht belasten, sondern über ein separates Host-Volume direkt ausgeliefert werden.

---
