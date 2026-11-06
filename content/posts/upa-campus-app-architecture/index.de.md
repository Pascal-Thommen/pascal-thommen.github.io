---
title: "UPA Campus Progressive Web App: Systemarchitektur und Engineering (Serie)"
date: 2026-11-06T18:00:00Z
description: "Projektseite und sechsteilige Architekturserie zur UPA Campus Progressive Web App: FastAPI, PostgreSQL 16, Service Worker PWA, Google OAuth 2.0, NetBird Zero Trust Tunnel und multimodale KI."
summary: "Zentrale Campus-Plattform der Universidad Paraguayo Alemana (UPA). Systemarchitektur, technischer Stack (FastAPI, PostgreSQL 16, Service Worker PWA, NetBird) und sechsteiliger Architektur-Leitfaden."
tags: ["FastAPI", "PostgreSQL 16", "Service Worker PWA", "Google OAuth 2.0", "NetBird", "Enterprise Systems"]
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
  - "/posts/upa-campus-pwa/"
cover:
  image: "cover.png"
  alt: "Systemarchitektur der UPA Campus Progressive Web App: Service Worker PWA, FastAPI, PostgreSQL 16, NetBird und Gemini Flash"
  caption: "UPA Campus Progressive Web App: Single Source of Truth mit FastAPI und PostgreSQL 16"
  relative: true
  hiddenInSingle: true
---

Die UPA Campus Progressive Web App bündelt Mitteilungen, Veranstaltungskalender, akademische Abgabefristen und Eilmeldungen der Universidad Paraguayo Alemana (UPA) in einer zentralen Plattform. Statt verteilter Kommunikationskanäle wie WhatsApp, E-Mail und Instagram entsteht eine einheitliche Quelle der Wahrheit (Single Source of Truth) für Studierende, Dozierende und Verwaltung.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![PostgreSQL 16](https://img.shields.io/badge/PostgreSQL%2016-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![PWA](https://img.shields.io/badge/Service%20Worker-PWA-5A0FC8?style=flat-square&logo=pwa&logoColor=white)
![NetBird](https://img.shields.io/badge/NetBird-Zero%20Trust%20Tunnel-1A202C?style=flat-square&logo=wireguard&logoColor=white)
![Google OAuth 2.0](https://img.shields.io/badge/Google%20OAuth%202.0-4285F4?style=flat-square&logo=google&logoColor=white)

---

## Projektprofil und Verantwortung

* **Projekt:** UPA Campus Progressive Web App (`v1.0.0`)
* **Rolle:** Projektleitung, Systemarchitektur und Backend Implementierung
* **Tech Stack:** FastAPI (Python 3.12), PostgreSQL 16, Service Worker PWA (`sw.js`), Google OAuth 2.0 (`@upa.edu.py`), NetBird Zero Trust Tunnel (WireGuard), Nginx Single-Origin Reverse Proxy
* **Kernfeatures:** Mobile First Campus PWA, automatische Event Extraktion mit multimodaler KI (Google Gemini Flash), Google Classroom Synchronisation, Web Push Benachrichtigungen (Firebase Cloud Messaging HTTP v1)
* **Quellcode und Dokumentation:** [GitHub Repository (Pascal-Thommen/upa-app)](https://github.com/Pascal-Thommen/upa-app)

![Systemarchitektur der UPA Campus Progressive Web App](architecture-diagram.svg)

---

## Technische Kernfunktionen im Überblick

### 1. Mobile First Campus PWA mit Service Worker
Die Anwendung für Studierende und Dozierende ist als plattformunabhängige Progressive Web App (`web-app/`) konzipiert, weil eine einheitliche Web-Codebasis den Wartungs- und Lizenzaufwand proprietärer App-Stores vermeidet und Updates über den Service Worker (`/sw.js`) ohne Verzögerung auf Android und iOS ausrollt. Das optische Grundgerüst (App Shell) wird lokal auf dem Endgerät zwischengespeichert, sodass Feed und Kalender auch in Hörsälen mit schwachem Mobilfunkempfang abrufbar bleiben. Auf Desktop-Bildschirmen leitet eine automatische Geräteweiche auf eine Vorschau mit dynamischem QR-Code zur direkten Smartphone-Installation weiter.

### 2. Automatische Event Extraktion mit multimodaler KI
Hochschulveranstaltungen werden vom Marketing-Team primär als grafische Plakate auf Instagram (`@upalemana`) veröffentlicht. Damit diese Termine ohne doppelte Texteingabe im akademischen Kalender erscheinen, analysiert das Backend (`backend/src/services/ai_extractor.py`) synchronisierte Plakate und Bildunterschriften über das multimodale Modell Google Gemini Flash und überführt Titel, Datum, Uhrzeit und Ort in strukturierte JSON-Felder. Im Redaktionsportal (`/dashboard/publicaciones`) steht diese Extraktion zusätzlich als 1-Klick-Funktion (`Autocompletar con IA`) bereit, ergänzt durch clientseitige WebP-Bildkomprimierung auf maximal 1080 Pixel Kantenlänge und ein dreistufiges Prüf-Badge (`No revisado`, `IA`, Vorname des Prüfers).

### 3. Google OAuth 2.0 und Google Classroom Synchronisation
Der Zugang zu internen Campus-Inhalten ist über Google OAuth 2.0 und signierte JWT-Sitzungstokens (`HS256`) kryptografisch auf verifizierte Konten der Hochschul-Domain `@upa.edu.py` beschränkt. Im Modul *Agenda* verknüpft das Backend die zentralen Hochschulveranstaltungen direkt mit den persönlichen Abgabefristen aus der Google Classroom API in der Zeitzone `America/Asuncion`. Ein automatischer OAuth-Token-Refresh und ein persistenter Datenbank-Fallback (`classroom_deadlines_cache`) stellen sicher, dass Fristen auch bei kurzzeitigen Ausfällen externer Google-Schnittstellen verfügbar bleiben.

### 4. Push Benachrichtigungen und NetBird Zero Trust Tunnel
Zeitkritische Hinweise (`EMERGENCY_BANNER`, etwa bei kurzfristigem Vorlesungsausfall) und Veranstaltungsupdates werden über Firebase Cloud Messaging (FCM HTTP v1) als native Web-Push-Benachrichtigungen zugestellt, wobei das Backend abgemeldete Gerätetokens automatisch bereinigt. Auf Netzwerkschicht verzichtet der Campus-Server auf eingehende Firewall-Portfreigaben: Ein ausgehender NetBird-Tunnel auf Basis von WireGuard verbindet den lokal an `127.0.0.1:8085` gebundenen Nginx-Reverse-Proxy verschlüsselt mit dem öffentlichen Ingress-Endpunkt.

---

## Der Lernpfad durch die sechs Architekturebenen

Anhand der UPA Campus Progressive Web App zerlegt diese Serie eine produktive Hochschulplattform systematisch von außen nach innen. Jedes Kapitel beleuchtet genau eine Systemebene und zeigt, welche konkreten Probleme des Hochschulalltags dort gelöst werden:

```mermaid
flowchart TD
    K1["Kapitel 1: Denkmodell & Domäne\nVom Pflichtenheft zur vierstufigen Schichtenarchitektur"] --> K2["Kapitel 2: Server & Container\nProzess-Isolation, Dockerfiles, Healthchecks & Volumes"]
    K2 --> K3["Kapitel 3: Netzwerkgrenze & Ingress\nWireGuard-Tunnel ohne Portfreigabe & Single-Origin-Routing"]
    K3 --> K4["Kapitel 4: Client-Engineering\nService Worker PWA, Offline-Resilienz & zustandsgesteuerte UI"]
    K4 --> K5["Kapitel 5: Backend & API-Verträge\nPydantic-Validierung, Multimodale KI & Least Privilege"]
    K5 --> K6["Kapitel 6: Relationale Persistenz\nPostgreSQL-Schema, Unique-Constraints, Medien-Streaming & Alembic"]
```

---
