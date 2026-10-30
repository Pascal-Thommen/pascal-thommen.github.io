---
title: "UPA Campus App (Kapitel 1): Big Picture und Systemübersicht"
date: 2026-10-30T16:00:00Z
description: "Wie funktioniert eine moderne Web-Applikation von Grund auf? Kapitel 1 erklärt die Drei-Schichten-Architektur der UPA Campus App und zeigt, wie das Pflichtenheft direkt aus der Systemstruktur entsteht."
summary: "Wie baut man eine produktive Campus-Plattform auf schlanker Infrastruktur? Kapitel 1 verbindet die universelle Drei-Schichten-Architektur mit den konkreten Modulen des Pflichtenhefts."
tags: ["Softwarearchitektur", "Docker", "FastAPI", "PostgreSQL", "Enterprise Systems"]
categories: ["Architektur", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 1
series_part_title: "Big Picture und Systemübersicht"
hiddenInList: true
---

Jede belastbare Web-Applikation trennt Darstellung, Geschäftslogik und Datenhaltung in unabhängige Schichten, weil ein einzelner monolithischer Prozess bei steigender Komplexität weder sicher abgesichert noch sauber gewartet werden kann. Die UPA Campus App setzt dieses Prinzip über drei isolierte Docker-Container hinter einem verschlüsselten NetBird-Ingress-Tunnel um. Dieses erste Kapitel vermittelt das Grundverständnis moderner Web-Architekturen und zeigt auf, wie jedes Funktionsmodul des Pflichtenhefts auf dieser technischen Arbeitsteilung aufbaut.

---

## Die Anatomie einer modernen Web-Applikation

Wenn ein Studierender morgens auf dem Smartphone prüft, ob wegen eines Unwetters Vorlesungen ausfallen oder wann die nächste Hausarbeit in Google Classroom fällig ist, wirkt die Anwendung wie ein einziges Programm. Technisch betrachtet findet jedoch eine präzise Staffelübergabe zwischen vier spezialisierten Komponenten statt:

```mermaid
flowchart LR
    Client["Client (Mobil-App / Web-Admin)"] -->|HTTPS 443| NetBird["NetBird Ingress Tunnel"]
    NetBird -->|127.0.0.1:8085| Nginx["Frontend Container (Nginx)"]
    Nginx -->|Statische Assets & App Shell| Client
    Nginx -->|Reverse Proxy /api/ Port 8000| Backend["Backend Container (FastAPI)"]
    Backend -->|SQL Port 5432| DB[("PostgreSQL Container")]
    DB -->|Relationale Tabellen| VolDB[("Volume: postgres_data")]
    Backend -->|Mediendateien| VolMedia[("Volume: uploads_data")]
    Nginx -->|Reverse Proxy /uploads/| VolMedia
```

Um das Zusammenspiel greifbar zu machen, hilft die Analogie eines gut organisierten Universitäts-restaurants:
* **Frontend (Nginx & Browser / App):** Der Gastraum und die Speisekarte. Hier sieht der Gast das Angebot und gibt seine Bestellung auf. Im Gastraum wird jedoch weder gekocht noch abgerechnet.
* **Reverse Proxy (Nginx Routing):** Der Empfangschef an der Tür. Er prüft, wohin ein Anliegen gehört, liefert einfache Broschüren (statische Dateien und Bilder) sofort selbst aus und leitet Bestellungen gezielt an die Küche weiter.
* **Backend (FastAPI):** Die Küche. Sie prüft, ob der Gast überhaupt bestellen darf (Authentifizierung und Rollenprüfung), kontrolliert die Bestellung auf Vollständigkeit (Schema-Validierung) und bereitet das Gericht nach festen Rezepten (Geschäftslogik) zu.
* **Datenbank (PostgreSQL & Volumes):** Das Kühlhaus und das Kassenbuch. Hier lagern alle Zutaten und Buchungen dauerhaft und vor fremdem Zugriff geschützt. Nur die Küche hat einen Schlüssel zum Kühlhaus.

---

## Warum jede Schicht eine exklusive Aufgabe erfüllt

1. **NetBird Ingress Tunnel:** Bildet ein verschlüsseltes WireGuard-Mesh-Netzwerk. Der Server öffnet keine ungeschützten Ports ins öffentliche Internet, weil der Datenverkehr ausschließlich über autorisierte Ende-zu-Ende-Tunnel an die lokale Loopback-Schnittstelle (`127.0.0.1:8085`) gelangt.
2. **Frontend-Container (Nginx):** Liefert sowohl die minimalistische Desktop-Landingpage (mit QR-Code zur Mobil-Installation und App-Vorschau) als auch das Web-Admin-Portal für das Marketing-Team und die mobile PWA aus. Gleichzeitig leitet Nginx als Reverse Proxy Anfragen mit dem Präfix `/api/` intern an das Backend und Anfragen an `/uploads/` an das Medien-Volume weiter.
3. **Backend-Container (FastAPI):** Verarbeitet REST-Anfragen asynchron, validiert jede Eingabe über Pydantic und setzt die fachlichen Regeln des Pflichtenhefts durch. Dazu gehören der exklusive Google OAuth Login für Universitäts-Accounts (`@example-campus.edu`), die Synchronisation mit der Google Classroom API und dem Google Workspace Directory sowie der Abgleich mit dem offiziellen Instagram-Feed.
4. **Datenbank-Container (PostgreSQL):** Speichert strukturierte Datensätze wie Benutzerprofile, Zielgruppen, vereinheitlichte Publikationen, 1-Klick-Anmeldungen (`EventRegistration`), eindeutige Lesebestätigungen (`PostView`) und das unveränderliche Änderungsprotokoll (`AdminAuditLog`) mit voller Transaktionssicherheit (ACID).

---

## Vom Architekturdiagramm zum Pflichtenheft: Das universelle Denkmodell

Wer heute als Wirtschaftsinformatiker oder mit KI-gestützten Entwicklungswerkzeugen (Vibe-Coding) Software entwirft, muss keine syntaktischen Grundlagen wie Schleifen oder CSS-Abstände auswendig lernen. Entscheidend ist die Fähigkeit, jede fachliche Anforderung aus einem Pflichtenheft systematisch in vier architektonische Bausteine zu übersetzen:
1. **Systemgrenze und Ingress:** Wer darf die Anwendung über welchen Netzwerkpfad überhaupt erreichen?
2. **Client-Zustand und Darstellung:** Welche Daten hält das Endgerät lokal für Offline-Geschwindigkeit vor, und welche Aktionen erfordern eine Rückfrage beim Server?
3. **Schnittstellenvertrag und Geschäftslogik:** Welche Regeln prüft die API zwingend, bevor Daten verändert oder Drittsysteme (wie Google Classroom oder Instagram) abgefragt werden?
4. **Relationale Persistenz:** In welchen Tabellen und mit welchen Datenbank-Constraints (wie Eindeutigkeitsregeln gegen Doppelbuchungen) wird der Zustand dauerhaft gesichert?

Wendet man dieses Denkmodell auf die UPA Campus App an, wird sofort sichtbar, warum die vier Hauptmodule des Pflichtenhefts genau so spezifiziert wurden:

```mermaid
flowchart TD
    subgraph Spec["Module des Pflichtenhefts"]
        M1["Modul Inicio: Notfall-Banner, News, 1-Klick-RSVP & Instagram"]
        M2["Modul Agenda: Chronologische Liste, Monatskalender & Classroom"]
        M3["Modul Perfil: Universitäre Identität (@example-campus.edu) & Offline-Speicher"]
        M4["Modul Web-Admin: Einheitlicher Editor, Alphabetische Gruppen & Rollen-Matrix"]
    end
    subgraph Arch["Technische Umsetzung in den Schichten"]
        A1["Vereintes Post-Modell in PostgreSQL + Read-Only Schutz für Instagram Sync"]
        A2["Dual-Quelle: Interne Events aus PostgreSQL + Live-Deadlines via Google Classroom OAuth"]
        A3["Google OAuth 2.0 Domain-Check im Backend + Cache Storage / Room im Client"]
        A4["Google Directory API Cache im Backend + Least-Privilege Rollen ohne Vorauswahl"]
    end
    M1 --> A1
    M2 --> A2
    M3 --> A3
    M4 --> A4
```

* **Warum gibt es ein einziges Publikationsmodell statt getrennter Tabellen für News und Events?** Im Pflichtenheft kann ein Beitrag gleichzeitig eine Nachricht im Feed und ein Termin im Kalender sein (z.B. ein Gastvortrag mit Ankündigungstext, Plakat, Ort, Uhrzeit und 1-Klick-Anmeldung). Weil das Backend beide Ansichten aus derselben Datenbanktabelle (`posts` über `/api/publications`) speist, entsteht eine echte *Single Source of Truth*: Ändert das Marketing-Team den Hörsaal einer Veranstaltung, aktualisieren sich Feed-Karte, Vollbild-Modal und Kalender-Eintrag im selben Augenblick.
* **Warum unterscheidet der Feed zwischen öffentlichem Modus und eingeloggt?** Vor dem Login zeigt der Endpunkt `/api/v1/feed` ausschließlich öffentliche Instagram-Beiträge der Universität. Erst mit einem gültigen JWT-Token nach dem Google-Login liefert das Backend interne Campus-Nachrichten, Zielgruppen-Events und zeitkritische Notfall-Banner (`EMERGENCY_BANNER`) aus. So bleibt die App auf dem Startbildschirm einladend, ohne interne Hochschuldaten preiszugeben.
* **Warum integriert das Agenda-Modul Google Classroom direkt über OAuth-Tokens?** Prüfungs- und Abgabefristen werden von Dozenten bereits in Google Classroom gepflegt. Würde man Dozenten zwingen, Fristen ein zweites Mal im Campus-Portal einzutippen, wären die Daten innerhalb weniger Wochen veraltet. Das Backend speichert daher beim Login ein autorisiertes Classroom-Token und kombiniert die offiziellen Abgabefristen live mit den internen Campus-Veranstaltungen.

---

## Glossar der Basiskonzepte

<details open>
<summary>Zentrale Begriffe der Infrastruktur</summary>

* **Container:** Eine isolierte Laufzeiteinheit, die Anwendungscode und sämtliche Systemabhängigkeiten bündelt. Da Container den Linux-Kernel des Host-Servers gemeinsam nutzen, starten sie in Sekundenbruchteilen und verbrauchen deutlich weniger Arbeitsspeicher als virtuelle Maschinen.
* **Port:** Ein numerischer Netzwerk-Endpunkt zur Adressierung eines Dienstes (z.B. Port 5432 für PostgreSQL oder Port 8000 für FastAPI). In einer sicheren Architektur sind interne Dienst-Ports nur innerhalb des privaten Container-Netzwerks erreichbar.
* **Reverse Proxy:** Ein vorgeschalteter Webserver (Nginx), der alle eingehenden Client-Anfragen am Eingangstor entgegennimmt und anhand des URL-Pfads (`/api/`, `/uploads/`, `/dashboard/`) an den zuständigen internen Dienst verteilt.
* **Volume:** Ein beständiger Speicherbereich auf der Festplatte des Host-Servers, der in den Container eingebunden wird, damit Datenbanktabellen und hochgeladene Bilder bei einem Container-Neustart oder Software-Update erhalten bleiben.

</details>

---

## Container-Orchestrierung und Speicherstrategie

An einer Universität entwickeln Studierende und Administratoren auf unterschiedlichen Betriebssystemen (Windows, macOS, Linux). Ohne Containerisierung führen abweichende Python-Versionen oder fehlende Systembibliotheken regelmäßig zu Fehlern beim Deployment. Docker Compose verbindet die drei Dienste deklarativ in einem abgeschirmten Bridge-Netzwerk:

```yaml
# docker-compose.yml (Auszug mit anonymisierten Platzhaltern)
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

Weil Container selbst zustandslos (stateless) sind und bei jedem Update neu erzeugt werden, trennt die Architektur den persistenten Zustand strikt in zwei benannte Host-Volumes:
1. **Relationale Daten (`postgres_data`):** Sichert Tabellen, Indizes und Transaktionsprotokolle unter `/var/lib/postgresql/data`.
2. **Mediendateien (`uploads_data`):** Speichert hochgeladene Veranstaltungsplakate, Videos (`MP4`, `WEBM`) und PDF-Dokumente unter `/app/uploads`. Große Binärdateien werden bewusst nicht als BLOB in der Datenbank gespeichert, weil sie sonst den Arbeitsspeicher-Cache von PostgreSQL verdrängen und Backups verlangsamen würden.

---

## Fazit

1. **Saubere Aufgabentrennung statt Monolith:** Die Aufteilung in Nginx-Proxy, FastAPI-Backend und PostgreSQL-Datenbank sorgt dafür, dass jede Schicht unabhängig aktualisiert, getestet und abgesichert werden kann.
2. **Pflichtenheft als logische Folge der Architektur:** Anforderungen wie das kombinierte Nachrichten- und Event-Modell (`Single Source of Truth`), die Google-Classroom-Synchronisation oder der geschützte interne Campus-Feed ergeben sich direkt aus dem Vermeiden von Datenredundanz und Sicherheitslücken.
3. **Zustandslose Container, dauerhafte Volumes:** Während die Anwendungs-Container jederzeit gefahrlos neu gebaut werden können, bleiben Datenbanktabellen und Mediendateien in getrennten Host-Volumes dauerhaft geschützt.
