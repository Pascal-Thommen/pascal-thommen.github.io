---
title: "UPA Campus App (Kapitel 2): Server und Container-Architektur"
date: 2026-10-30T16:15:00Z
description: "Warum läuft die UPA Campus App in drei getrennten Docker-Containern? Kapitel 2 erklärt Betriebssystem-Virtualisierung, Dockerfiles, Healthcheck-Startketten und persistente Volumes."
summary: "Wie sorgen Docker-Container, Healthchecks und Volumes für stabilen Betrieb ohne Datenverlust? Kapitel 2 zeigt den technischen Aufbau der Server-Infrastruktur."
tags: ["Docker", "Softwarearchitektur", "PostgreSQL", "FastAPI", "Enterprise Systems"]
categories: ["Architektur", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 2
series_part_title: "Server und Container-Architektur"
hiddenInList: true
---

Wie stellt man sicher, dass eine Web-Applikation auf jedem Entwickler-Laptop und auf dem Produktivserver exakt gleich läuft, ohne dass ein Fehler in der Benutzeroberfläche die Datenbank beeinträchtigt? Durch die Kapselung von Webserver, Geschäftslogik und Datenbank in drei isolierte Linux-Container mit klar getrennten Speicherbereichen. Dieses zweite Kapitel beleuchtet die Funktionsweise von Docker, den Aufbau produktionsreifer Dockerfiles, die koordinierte Startkette nach einem Serverneustart und die Trennung zwischen flüchtigem Programmcode und dauerhaften Nutzdaten.

---

## Betriebssystem-Virtualisierung statt schwerfälliger Virtueller Maschinen

In Hochschulprojekten arbeiten Teammitglieder oft auf völlig unterschiedlichen Systemen (Windows, macOS, Linux). Wird Python oder PostgreSQL direkt auf dem jeweiligen Betriebssystem installiert, führen abweichende Paketversionen rasch zu Fehlern, die nur auf einzelnen Rechnern auftreten.

Klassische virtuelle Maschinen (VMs) lösen dieses Problem zwar durch komplette Hardware-Emulation, bringen aber für jeden Dienst ein eigenes Gast-Betriebssystem samt eigenem Kernel mit. Das kostet mehrere Gigabyte Arbeitsspeicher und verlangsamt den Systemstart erheblich. Docker-Container nutzen stattdessen direkt den Linux-Kernel des Host-Servers:

```mermaid
flowchart TD
    subgraph VM_Stack["Klassische Virtuelle Maschine"]
        direction TB
        V1["Anwendung & Bibliotheken"] --> V2["Vollständiges Gast-OS (eigener Kernel)"]
        V2 --> V3["Hypervisor"] --> V4["Server-Hardware"]
    end
    subgraph Container_Stack["Docker-Containerisierung"]
        direction TB
        C1["Anwendung & Bibliotheken"] --> C2["Docker Engine"]
        C2 --> C3["Gemeinsamer Linux-Kernel (Namespaces & cgroups)"] --> C4["Server-Hardware"]
    end
```

Zwei Mechanismen im Linux-Kernel machen diese leichtgewichtige Isolation möglich:
* **Namespaces:** Geben jedem Container einen eigenen Sichtbereich für Prozesse (`PID`), Netzwerkadapter (`NET`) und Verzeichnisbäume (`MNT`). Ein Container sieht fremde Prozesse auf demselben Server nicht.
* **Control Groups (cgroups):** Regeln die Zuteilung von Arbeitsspeicher und Rechenzeit, damit eine rechenintensive Abfrage nicht das gesamte System ausbremst.

---

## Warum das Pflichtenheft drei getrennte Container vorschreibt

In der Architekturspezifikation der UPA Campus App ist festgelegt, Web-Frontend (`campus-frontend`), API-Server (`campus-backend`) und Datenbank (`campus-postgres`) nicht in einen gemeinsamen Container zu packen, sondern als eigenständige Einheiten zu betreiben:

```mermaid
flowchart LR
    UI["Container 1: campus-frontend\nNginx (Web-Admin & PWA)"] -->|Interne API-Aufrufe| API["Container 2: campus-backend\nPython 3.12 & FastAPI"]
    API -->|SQL-Transaktionen| DB[("Container 3: campus-postgres\nPostgreSQL 16 Alpine")]
```

Diese Dreiteilung löst drei praktische Herausforderungen im Projektalltag:
1. **Oberflächen-Updates ohne API-Unterbrechung:** Wenn das Marketing-Team eine Anpassung am Layout des Redaktionsportals wünscht, wird lediglich der schlanke Nginx-Container neu gebaut und in Sekundenbruchteilen ausgetauscht. Das Backend und die laufenden Verbindungen der mobilen App bleiben davon unberührt.
2. **Begrenzung der Angriffsfläche:** Der Frontend-Container besitzt weder einen Python-Interpreter noch Datenbank-Zugangsdaten oder Google-API-Schlüssel.
3. **Echte Testumgebung für die Mobile-Entwicklung:** Weil das Backend samt automatischer Datenbank-Initialisierung als fertiger Container startet, kann das Android-Team mit einem einzigen Befehl (`docker compose up -d`) lokal einen vollwertigen Server hochfahren, statt gegen starre Platzhalterdaten zu entwickeln.

---

## Ein Blick in das Backend-Dockerfile: Geschwindigkeit, Rechte und Selbstheilung

Während für die Datenbank das geprüfte Standard-Image `postgres:16-alpine` genutzt wird, definiert ein eigenes `Dockerfile` die Laufzeitumgebung des FastAPI-Backends:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app \
    UPLOAD_DIR=/app/uploads

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN addgroup --system appgroup && adduser --system --group appuser

COPY src/ ./src
COPY alembic/ ./alembic
COPY alembic.ini .

RUN mkdir -p /app/uploads && chown -R appuser:appgroup /app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=15s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Drei Konstruktionsprinzipien in dieser Datei sind für das Verständnis moderner Container-Builds entscheidend:
* **Gezieltes Schichten-Caching (Layer Caching):** Docker speichert jeden Befehl als wiederverwendbare Schicht. Weil `COPY requirements.txt .` und `pip install` vor dem Kopieren des Quellcodes (`COPY src/ ./src`) stehen, überspringt Docker die Paketinstallation, solange sich nur der Programmcode ändert. Ein Neubau dauert dadurch wenige Sekunden statt mehrerer Minuten.
* **Verzicht auf Root-Rechte (`USER appuser`):** Ohne explizite Vorgabe führt Docker Programme als Systemadministrator (`root`) aus. Der Wechsel auf den eingeschränkten Benutzer `appuser` sorgt dafür, dass der Prozess keine Systemdateien des Containers verändern kann.
* **Laufende Funktionsprüfung (`HEALTHCHECK`):** Ein gestarteter Prozess bedeutet noch nicht, dass die Anwendung auch auf Anfragen reagiert. Über die `HEALTHCHECK`-Regel prüft Docker alle 15 Sekunden den Endpunkt `/health` und erkennt sofort, falls der Serverprozess blockiert ist.

---

## Koordinierte Startkette und dauerhafte Volumes in Docker Compose

Startet ein Server nach einer Wartung neu, fahren alle Container gleichzeitig hoch. Da PostgreSQL beim Start zunächst seine internen Transaktionsprotokolle prüft, während Python fast sofort bereitsteht, würde das Backend ohne Abstimmung ins Leere greifen und mit einem Verbindungsfehler abbrechen.

In der `docker-compose.yml` verhindert eine zustandsabhängige Startbedingung (`condition: service_healthy`) diesen Wettlauf:

```yaml
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U db_user -d app_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - campus-network

  backend:
    build: ./backend
    container_name: campus-backend
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      APP_ENV: production
      DATABASE_URL: postgresql://db_user:secret_password@postgres:5432/app_db
      ALLOWED_EMAIL_DOMAIN: example-campus.edu
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

Erst wenn das Prüfwerkzeug `pg_isready` meldet, dass die Datenbank Verbindungen akzeptiert, gibt Docker den Start des Backends frei.

### Warum Container flüchtig sein müssen und Volumes die Daten retten

Wird ein Software-Update eingespielt (`docker compose up -d --build`), löscht Docker den bisherigen Container vollständig und ersetzt ihn durch eine frische Instanz aus dem aktualisierten Image. Alles, was direkt in das interne Dateisystem (`OverlayFS`) des Containers geschrieben wurde, geht dabei verloren:

```mermaid
flowchart TD
    subgraph Fluechtig["Flüchtige Container-Laufzeit (Wird bei jedem Update ersetzt)"]
        Code["Schreibgeschütztes Image (Python, FastAPI, Nginx)"]
        Overlay["Temporäre Container-Schicht (OverlayFS)"]
    end
    subgraph Dauerhaft["Beständiger Host-Speicher (Überlebt jedes Update)"]
        VolDB[("Volume: postgres_data\nTabellen, Indizes & Transaktions-Logs")]
        VolMedia[("Volume: uploads_data\nPlakate, MP4/WEBM-Videos & PDFs")]
    end
    Fluechtig -->|Eingehängt unter /var/lib/postgresql/data| VolDB
    Fluechtig -->|Eingehängt unter /app/uploads| VolMedia
```

Damit bei einem Update weder Benutzerkonten noch hochgeladene Veranstaltungsmedien verloren gehen, lagert die Architektur alle veränderlichen Daten in zwei benannte **Docker Volumes** auf der Festplatte des Host-Servers aus:
1. **`postgres_data`:** Bewahrt sämtliche Datenbanktabellen und Indizes außerhalb des Containers auf.
2. **`uploads_data`:** Speichert die über das Redaktionsportal hochgeladenen Bilder, Videos und PDF-Dokumente dauerhaft unter `/app/uploads`.

---

## Fazit

1. **Reproduzierbare Umgebungen durch Container-Trennung:** Die Aufteilung in drei spezialisierte Container für Frontend, Backend und Datenbank ermöglicht unterbrechungsfreie Oberflächen-Updates und identische Bedingungen auf jedem Entwicklerrechner.
2. **Kontrollierter Systemstart und sichere Persistenz:** Während Healthchecks (`pg_isready`) Absturzschleifen beim Hochfahren verhindern, stellen die ausgelagerten Volumes (`postgres_data` und `uploads_data`) sicher, dass Programmcode jederzeit aktualisiert werden kann, ohne bestehende Daten zu gefährden.
