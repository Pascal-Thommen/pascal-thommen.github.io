---
title: "UPA Campus App: Architektur und System-Engineering (Serie)"
date: 2026-10-30T17:00:00Z
description: "Sechsteilige Architekturserie zur UPA Campus App: Das mentale Modell moderner Web-Applikationen für Wirtschaftsinformatiker und KI-gestütztes Software-Engineering."
summary: "Wie wird aus einem fachlichen Pflichtenheft eine produktive Web-Applikation? Der sechsteilige Leitfaden durch Infrastruktur, Netzwerkgrenzen, Clients, APIs und Datenmodelle."
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

Warum scheitern viele Softwareprojekte im Studium oder beim KI-gestützten Programmieren (Vibe-Coding), sobald die erste Benutzeroberfläche steht? Weil Sprachmodelle zwar sekundenschnell funktionierenden Programmcode liefern, ohne architektonische Leitplanken jedoch unweigerlich widersprüchliche Datenstrukturen, offene Sicherheitslücken und instabile Schnittstellen erzeugen. Wer als Wirtschaftsinformatiker oder Entwickler eine belastbare Plattform baut, lernt kein Pflichtenheft auswendig, sondern durchdringt die technische Ursache hinter jeder einzelnen Anforderung.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![Python](https://img.shields.io/badge/Python%203.12-3776AB?style=flat-square&logo=python&logoColor=white)

---

## Der Lernpfad durch die sechs Architekturebenen

Anhand der UPA Campus App zerlegt diese Serie eine produktive Hochschulplattform systematisch von außen nach innen. Jedes Kapitel beleuchtet genau eine Systemebene und zeigt, welche konkreten Probleme des Hochschulalltags dort gelöst werden:

```mermaid
flowchart TD
    K1["Kapitel 1: Denkmodell & Domäne\nVom Pflichtenheft zur vierstufigen Schichtenarchitektur"] --> K2["Kapitel 2: Server & Container\nProzess-Isolation, Dockerfiles, Healthchecks & Volumes"]
    K2 --> K3["Kapitel 3: Netzwerkgrenze & Ingress\nWireGuard-Tunnel ohne Portfreigabe & Single-Origin-Routing"]
    K3 --> K4["Kapitel 4: Client-Engineering\nApp Shell, Offline-Resilienz & zustandsgesteuerte UI"]
    K4 --> K5["Kapitel 5: Backend & API-Verträge\nPydantic-Validierung, Service-Schicht & Least Privilege"]
    K5 --> K6["Kapitel 6: Relationale Persistenz\nTabellendesign, Datenbank-Constraints, Medien-Streaming & Alembic"]
```

---

## Was diese Serie vermittelt

* **Architekturverständnis statt Syntax-Memorierung:** Programmierbefehle lassen sich jederzeit nachschlagen oder generieren. Systemgrenzen, Zustandsflüsse und Transaktionssicherheit müssen dagegen vom Menschen präzise vorgegeben werden.
* **Pflichtenheft als technischer Bauplan:** Jede Produktregel der UPA Campus App hat einen handfesten architektonischen Grund. Wer diesen Grund kennt, trifft bei Erweiterungen und Edge-Cases eigenständig die richtigen Designentscheidungen.
* **Vom großen Überblick bis zur Datenbankzeile:** Die Serie startet in Kapitel 1 mit dem Zusammenspiel der Gesamtschichten und führt Schritt für Schritt bis zur physischen Speicherung auf dem Server in Kapitel 6.

---
