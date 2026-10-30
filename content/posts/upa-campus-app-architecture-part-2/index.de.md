---
title: "UPA Campus App (Kapitel 2): Server und Docker Infrastruktur"
date: 2026-10-30T16:15:00Z
description: "Containerisierung, Port-Isolation und Persistenz: Wie die UPA Campus App PostgreSQL, FastAPI und Nginx mit Docker Compose absichert und warum die Container-Trennung eine Kernentscheidung der Spezifikation ist."
summary: "Wie funktionieren Containerisierung, Healthcheck-Startketten und persistente Volumes in der Praxis? Kapitel 2 vertieft Dockerfiles, Port-Isolation und die architektonische Entkopplung."
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
series_part_title: "Server und Docker Infrastruktur"
hiddenInList: true
---

Moderne Webanwendungen verlangen reproduzierbare Laufzeitumgebungen, klar getrennte Ausfallbereiche und verlässliche Datenpersistenz, weil ein einzelner Softwarefehler im Frontend niemals die Datenbank oder die API-Schnittstelle mitreißen darf. Die UPA Campus App setzt diese Anforderung durch drei modular entkoppelte Docker-Container für Datenbank, REST-API und Web-Frontend um. Dieses zweite Kapitel analysiert das Funktionsprinzip von Linux-Containern, den Aufbau der Dockerfiles, die koordinierte Startkette via Docker Compose sowie die Trennung von flüchtigem Anwendungscode und dauerhaftem Speicher.

---

## Containerisierung gegenüber virtuellen Maschinen

Warum setzt die UPA Campus App auf Docker statt auf klassische virtuelle Maschinen (VMs) oder eine direkte Installation auf dem Server-Betriebssystem? Der entscheidende Unterschied liegt in der Virtualisierungsebene und dem Ressourcenverbrauch:

```mermaid
flowchart TD
    subgraph VM_Arch["Virtuelle Maschinen (Hardware-Virtualisierung)"]
        direction TB
        AppVM["Applikation & Bibliotheken"]
        GuestOS["Vollständiges Gast-Betriebssystem"]
        Hypervisor["Hypervisor"]
        HostHW1["Host-System & Hardware"]
        AppVM --> GuestOS --> Hypervisor --> HostHW1
    end
    subgraph Docker_Arch["Container (Betriebssystem-Virtualisierung)"]
        direction TB
        AppDocker["Applikation & Bibliotheken"]
        Engine["Docker Engine"]
        SharedKernel["Gemeinsamer Linux-Kernel"]
        HostHW2["Host-System & Hardware"]
        AppDocker --> Engine --> SharedKernel --> HostHW2
    end
```

Klassische virtuelle Maschinen emulieren die gesamte Computer-Hardware. Jede VM benötigt ein eigenes Gast-Betriebssystem mit eigenem Kernel und Hintergrunddiensten, was pro Instanz mehrere Gigabyte Arbeitsspeicher bindet und den Start auf Minuten verzögert.

Docker-Container teilen sich dagegen den Linux-Kernel des Host-Servers und nutzen zwei native Kernel-Mechanismen zur Isolation:
* **Namespaces:** Trennen Prozessbäume (`PID`), Netzwerkschnittstellen (`NET`), Benutzerkennungen (`USER`) und Dateisystem-Einhängepunkte (`MNT`). Jeder Container sieht ausschließlich seine eigenen Prozesse.
* **Control Groups (cgroups):** Begrenzen und überwachen den Verbrauch von Arbeitsspeicher, CPU-Zeit und Festplatten-I/O, sodass ein einzelner Dienst niemals den gesamten Host-Server blockieren kann.

Für ein universitäres Projektteam löst Docker zudem das klassische Problem abweichender Entwickler-Rechner ("Works on my machine"): Egal ob ein Teammitglied unter Windows mit WSL2, macOS oder Linux entwickelt, kapselt das Container-Image die exakte Python-Version, die C-Bibliotheken für den Datenbanktreiber und die Nginx-Konfiguration bitgenau ein.

---

## Warum Frontend und Backend in getrennten Containern laufen

In der Architekturspezifikation der UPA Campus App (Entscheidung 3.3: *Desacoplamiento de Contenedores Frontend y Backend*) wurde bewusst festgelegt, das Web-Portal und die FastAPI-Schicht nicht in einen gemeinsamen Container zu packen:

```mermaid
flowchart LR
    Browser["Client Browser / App"] -->|127.0.0.1:8085| Nginx["Frontend Container: Nginx (Statische UI & Proxy)"]
    Nginx -->|Reverse Proxy /api/ Port 8000| FastAPI["Backend Container: FastAPI (Geschäftslogik)"]
    FastAPI -->|SQL Port 5432| Postgres[("Datenbank Container: PostgreSQL")]
    Postgres -->|ACID Tabellen| VolPG[("Host Volume: postgres_data")]
    FastAPI -->|Bilder, Videos & PDFs| VolUpload[("Host Volume: uploads_data")]
```

Diese strikte Trennung bringt im laufenden Campus-Betrieb drei handfeste Vorteile:
1. **Unabhängige Deployments ohne API-Unterbrechung:** Wenn das Marketing-Team eine rein optische Anpassung am Redaktions-Cockpit (`web-admin`) oder an der QR-Landingpage wünscht, wird lediglich der leichtgewichtige Nginx-Container neu gebaut und in einer Sekunde ausgetauscht. Der FastAPI-Container und die aktiven Datenbankverbindungen der Android-App laufen ohne jede Unterbrechung weiter.
2. **Isolierte Angriffs- und Ausfalldomänen:** Der Nginx-Container enthält keinerlei Python-Interpreter, keine Datenbank-Zugangsdaten und keine Google-Service-Account-Schlüssel. Selbst eine theoretische Schwachstelle in der Auslieferung statischer Assets gewährt keinen Zugriff auf den API-Kern.
3. **Zero-Mock-Entwicklung für das Android-Team:** Weil das Backend samt automatischer Datenbank-Migration und repräsentativen Seed-Daten in einem eigenen Container gekapselt ist, können die Mobile-Entwickler mit einem einzigen Befehl (`docker compose up -d`) lokal einen vollwertigen API-Server starten und direkt gegen echte Endpunkte statt gegen künstliche Attrappen programmieren.

---

## Der Aufbau der drei Container im Detail

### 1. Datenbank: Offizielles PostgreSQL Image

Für die relationale Datenhaltung kommt das offizielle Image `postgres:16-alpine` zum Einsatz. Ein eigenes Dockerfile für eine Standard-Datenbank wäre ein unnötiges Wartungsrisiko. Das Alpine-Image bleibt unter 100 MB, bringt alle Sicherheits-Patches der PostgreSQL-Community mit und wird vollständig über Umgebungsvariablen (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`) initialisiert.

### 2. Backend: Eigenes Dockerfile mit Python und FastAPI

Das Backend verarbeitet sämtliche REST-Anfragen und Hintergrundaufgaben. Die Basis bildet `python:3.12-slim`:

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

Drei Details in diesem Dockerfile sind für den stabilen Produktionsbetrieb entscheidend:
* **Schichten-Caching (Layer Caching):** Docker speichert jede Anweisung als eigene Schicht. Weil `COPY requirements.txt .` und `pip install` vor `COPY src/ ./src` stehen, muss Docker die Python-Pakete bei normalen Code-Änderungen nicht neu herunterladen. Ein Build dauert dadurch Sekunden statt Minuten.
* **Prinzip minimaler Rechte (`USER appuser`):** Standardmäßig laufen Prozesse in Docker als `root`. Durch den Wechsel auf den Systembenutzer `appuser` besitzt der Python-Prozess selbst im Falle eines Exploits keine Administratorrechte im Container.
* **Aktive Selbstüberwachung (`HEALTHCHECK`):** Alle 15 Sekunden ruft Docker intern den Endpunkt `/health` auf. Antwortet FastAPI dreimal in Folge nicht mit HTTP 200, markiert Docker den Container sofort als `unhealthy`.

### 3. Frontend: Eigenes Dockerfile mit Nginx

Der Frontend-Container bündelt die statischen Oberflächen (Web-Admin-Cockpit und Campus-PWA) sowie die Routing-Regeln:

```dockerfile
FROM nginx:alpine

RUN rm -rf /usr/share/nginx/html/*

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=15s --timeout=5s --start-period=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1:80/frontend-health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

### 4. Startkoordination mit Docker Compose

Was passiert, wenn nach einem Stromausfall alle drei Container gleichzeitig starten? PostgreSQL benötigt einige Sekunden, um seine Transaktionsprotokolle zu prüfen, während Python fast verzögerungsfrei hochfährt. Ohne Koordination würde das Backend sofort versuchen, die Datenbank zu erreichen, mit einem Verbindungsfehler abstürzen und einen Crash-Loop auslösen.

Docker Compose verhindert dieses Wettrennen über eine explizite Gesundheitsprüfung (`condition: service_healthy`):

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
```

Erst wenn das Hilfsprogramm `pg_isready` bestätigt, dass PostgreSQL SQL-Verbindungen annimmt, startet Docker den Backend-Container.

---

## Sicherheit durch Port-Isolation

Ein häufiger Fehler bei Einsteigern ist es, für jeden Container eine `ports`-Freigabe in die Compose-Datei zu schreiben (z.B. `5432:5432` für die Datenbank).

```mermaid
flowchart LR
    subgraph PublicInternet["Öffentliches Internet"]
        Attacker["Automatisierte Port-Scanner"]
    end
    subgraph HostServer["Host Server"]
        NetBird["NetBird Tunnel Interface"]
        subgraph DockerBridge["Isoliertes Docker Bridge Netzwerk"]
            Frontend["Frontend (Port 80)"]
            Backend["Backend (Port 8000)"]
            DB[("PostgreSQL (Port 5432)")]
        end
    end
    Attacker -.->|Blockiert: 0 offene öffentliche Ports| HostServer
    NetBird -->|Zugriff an 127.0.0.1:8085| Frontend
    Frontend -->|Internes DNS: backend:8000| Backend
    Backend -->|Internes DNS: postgres:5432| DB
```

Die UPA Campus App setzt im Produktivbetrieb zwei Schutzmechanismen durch:
1. **Kein Host-Binding für Datenbank und API:** Weder Port 5432 noch Port 8000 werden im Produktivbetrieb nach außen gebunden. Die Container erreichen sich über den internen Docker-DNS-Server anhand ihres Servicenamens (`postgres:5432` und `backend:8000`). Für jedes Gerät außerhalb des Docker-Bridge-Netzwerks sind diese Ports unsichtbar.
2. **Strikte Loopback-Bindung (`127.0.0.1:8085:80`):** Würde man den Frontend-Port als `8085:80` notieren, bände Docker ihn standardmäßig an alle Netzwerkschnittstellen (`0.0.0.0`) und würde dabei auf Linux-Servern automatisch die Regeln der lokalen Firewall (wie UFW) in den `iptables` überschreiben. Durch das vorangestellte `127.0.0.1:` lauscht Nginx ausschließlich auf der lokalen Loopback-Schnittstelle des Servers, wo der lokale NetBird-Tunnel den Verkehr sicher abholt.

---

## Flüchtige Container vs. beständige Volumes

Warum gehen Daten in einem normalen Container verloren, sobald man ein Update einspielt?

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Container-Schicht (Flüchtig)"]
        AppCode["Unveränderbares Image (Code & Bibliotheken)"]
        TempFS["Beschreibbare Container-Schicht (OverlayFS)"]
        KillAction["Container-Update (docker compose up -d --build)"]
        TempFS -.->|Wird beim Update restlos gelöscht| KillAction
    end
    subgraph HostStorage["Host-Dateisystem (Dauerhaft)"]
        PGVolume[("Docker Volume: postgres_data")]
        UploadVolume[("Docker Volume: uploads_data")]
    end
    ContainerLifecycle -->|Eingehängt unter /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Eingehängt unter /app/uploads| UploadVolume
```

Jeder Container besteht aus schreibgeschützten Image-Schichten und einer dünnen, beschreibbaren Ober-Schicht (`OverlayFS`). Spielt man ein Software-Update über `docker compose up -d --build` ein, wirft Docker den alten Container samt seiner beschreibbaren Schicht weg und startet einen frischen Container aus dem neuen Image.

Damit dabei weder die registrierten Studierenden noch die hochgeladenen Veranstaltungs-Videos verschwinden, lagert Docker Compose beide Verzeichnisse in benannte **Docker Volumes** auf dem Host-Dateisystem (`/var/lib/docker/volumes/`) aus:
* **`postgres_data`:** Überlebt jeden Container-Austausch und bewahrt sämtliche Datenbanktabellen, Indizes und Write-Ahead-Logs.
* **`uploads_data`:** Bewahrt alle über den einheitlichen Medien-Upload eingereichten Bilder (`JPG`, `PNG`, `WEBP`), Videos (`MP4`, `WEBM`) und Dokumente (`PDF`).

---

## Fazit

1. **Entkopplung schafft Betriebssicherheit:** Die Trennung von Nginx, FastAPI und PostgreSQL in eigene Container erlaubt unabhängige UI-Updates und schützt den API-Kern vor Fehlern in der Präsentationsschicht.
2. **Deterministische Startkette:** Durch `pg_isready` und Healthchecks wartet das Backend beim Systemstart kontrolliert auf die Datenbank, statt in unkontrollierte Neustart-Schleifen zu geraten.
3. **Geschlossene Ports und langlebige Volumes:** Interne Dienste sprechen nur über das private Docker-Bridge-Netzwerk miteinander, während benannte Volumes dafür sorgen, dass Updates jederzeit ohne Datenverlust möglich sind.
