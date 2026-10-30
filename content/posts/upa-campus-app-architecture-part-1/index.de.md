---
title: "UPA Campus App: Architektur und Container-Infrastruktur"
date: 2026-10-30T16:00:00Z
description: "Systemarchitektur der UPA Campus App: Vom NetBird Ingress Tunnel über Nginx und FastAPI bis zur PostgreSQL-Datenpersistenz."
summary: "Wie ist eine produktive Campus-Plattform strukturiert? Teil 1 analysiert das Zusammenspiel von PWA, Nginx-Reverse-Proxy, FastAPI-Backend und PostgreSQL-Volumes."
tags: ["Softwarearchitektur", "Docker", "FastAPI", "PostgreSQL", "Enterprise Systems"]
categories: ["Architektur", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
draft: false
---

Wie baut man eine zuverlässige Campus-Plattform für Studierende und Dozenten mit schlanker Infrastruktur? Die UPA Campus App trennt Benutzeroberfläche, API-Logik und Datenhaltung strikt in drei leichtgewichtige Docker-Container, abgesichert über einen verschlüsselten NetBird-Ingress-Tunnel. Dieser Leitfaden erklärt die Systemarchitektur, das Zusammenspiel der Container und die Speicherstrategie für relationale Daten und Mediendateien.

---

## Systemübersicht

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

## Rollenverteilung der Kernkomponenten

Jede Schicht im System erfüllt eine klar abgegrenzte Aufgabe:

1. **NetBird Ingress Tunnel:** Bildet ein verschlüsseltes WireGuard-Mesh-Netzwerk. Der Produktionsserver öffnet keine ungeschützten Ports ins öffentliche Internet, weil der Zugriff ausschließlich über autorisierte Peer-to-Peer-Tunnel erfolgt.
2. **Frontend-Container (Nginx):** Liefert die Web-Admin-PWA als vorkompilierte HTML-, CSS- und JavaScript-Dateien aus. Gleichzeitig fungiert Nginx als Reverse Proxy: Anfragen an `/api/` werden intern an das Backend weitergereicht, Anfragen an `/uploads/` bedienen gespeicherte Bilddateien. Der Browser kommuniziert für alle Aufgaben mit einer einzigen Domain.
3. **Backend-Container (FastAPI):** Verarbeitet REST-Anfragen, validiert Eingabedaten mit Pydantic und setzt Geschäftsregeln durch. Dazu zählen die Google-OAuth-Authentifizierung für Universitäts-Accounts (`@example-campus.edu`), die Berechtigungsprüfung sowie die Aggregation von Veranstaltungsdaten.
4. **Datenbank-Container (PostgreSQL):** Verwaltet relationale Datensätze wie Profile, Rollen, Vorlesungstermine und Ankündigungen mit vollständiger ACID-Garantie auf Basis eines schlanken Alpine-Linux-Images.

---

## Glossar der Basiskonzepte

<details open>
<summary>Zentrale Begriffe der Infrastruktur</summary>

* **Container:** Eine isolierte Software-Einheit, die Anwendungs-Code und Abhängigkeiten bündelt. Container nutzen den Kernel des Host-Systems gemeinsam, weshalb sie ressourcensparend arbeiten und in Sekundenbruchteilen starten.
* **Port:** Ein logischer Netzwerk-Kanal zur Adressierung von Diensten. Beim Port-Mapping wird ein externer Host-Port (etwa 8085) auf einen internen Container-Port (etwa 80) weitergeleitet.
* **Reverse Proxy:** Ein vorgeschalteter Server, der Anfragen entgegennimmt und an interne Microservices oder Container verteilt. Er schützt interne Netzwerke, vereinfacht SSL-Zertifikate und bündelt Pfade unter einer gemeinsamen Adresse.
* **Volume:** Ein dedizierter Speicherbereich auf dem Host-Dateisystem, der außerhalb des flüchtigen Container-Dateisystems liegt und Daten über Lebenszyklen von Containern hinweg sichert.

</details>

---

## Container-Infrastruktur im Detail

Warum Docker für die UPA Campus App? An Universitäten arbeiten Entwicklungsteams auf unterschiedlichen Betriebssystemen (Linux, macOS, Windows). Ohne Containerisierung führen abweichende Python-Versionen, Node-Abhängigkeiten oder lokale Datenbank-Konfigurationen regelmäßig zu Fehlern. Docker garantiert identische Laufzeitumgebungen von der lokalen Entwicklung bis zum Serverbetrieb.

Die drei Container laufen gemeinsam in einem isolierten Docker-Bridge-Netzwerk:

```yaml
# docker-compose.yml (vereinfachter Auszug mit neutralen Platzhaltern)
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

Innerhalb des Netzwerks erreichen sich die Dienste über ihre Servicenamen (`postgres:5432`, `backend:8000`). Weder die Datenbank noch die FastAPI-Instanz müssen ihre Ports direkt an den Host binden, weil der Nginx-Frontend-Container den gesamten Datenverkehr regelt.

---

## Datenpersistenz und Speicherstrategie

Container sind zustandslos (stateless). Wird ein Container gestoppt oder durch ein Update neu erzeugt, verfällt sein internes Dateisystem. Produktionsdaten erfordern daher eine zweigeteilte Speicherstrategie:

1. **Strukturierte Daten im Host-Volume (`postgres_data`):** PostgreSQL schreibt Tabellen, Indizes und Transaktions-Logs in `/var/lib/postgresql/data`. Durch die Bindung an ein benanntes Docker-Volume bleiben alle Daten beim Austausch des PostgreSQL-Images erhalten.
2. **Ungestrukturierte Mediendaten im Upload-Volume (`uploads_data`):** Wenn Administratoren Veranstaltungs-Flyer oder Bilder hochladen, landen diese Dateien im Volume `/app/uploads`. Sie werden bewusst nicht als BLOBs in der PostgreSQL-Datenbank abgelegt, weil große Binärdateien Datenbank-Backups aufblähen und Datenbank-Caches belasten. Die Datenbank speichert lediglich den relativen Dateipfad, während Nginx die statische Datei mit optimierten Caching-Headern direkt an den Client ausliefert.

---

## Fazit

1. **Klare Schichtentrennung:** Die Aufteilung in Frontend-Proxy, Backend-API und Datenbank verhindert monolithische Abhängigkeiten und vereinfacht gezielte Updates.
2. **Zustandsloser Code, persistente Volumes:** Container bleiben jederzeit ersetzbar, während relationale Daten und hochgeladene Medien in getrennten Host-Volumes gesichert sind.
3. **Minimale Angriffsfläche:** Interne Container kommunizieren über das Docker-Bridge-Netzwerk, während NetBird den Zugang ohne offene öffentliche Ports absichert.
