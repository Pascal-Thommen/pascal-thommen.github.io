---
title: "UPA Campus App (Kapitel 2): Server und Docker Infrastruktur"
date: 2026-10-30T16:15:00Z
description: "Containerisierung, Port-Isolation und Persistenz: Wie die UPA Campus App PostgreSQL, FastAPI und Nginx mit Docker Compose absichert."
summary: "Wie funktionieren Containerisierung, Port-Isolation und Datenpersistenz in der Praxis? Kapitel 2 vertieft Dockerfiles, Compose-Orchestrierung und Speicherstrategien."
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

Moderne Webanwendungen verlangen reproduzierbare Laufzeitumgebungen, getrennte Ausfallbereiche und verlässliche Datenpersistenz. Die UPA Campus App realisiert diese Anforderungen über drei modular gekoppelte Docker-Container für Datenbank, REST-API und Web-Frontend. Dieser zweite Teil analysiert den Aufbau der Dockerfiles, die deklarative Orchestrierung via Docker Compose, Mechanismen der Port-Isolation sowie die persistente Datenspeicherung.

---

## Containerisierung gegenüber virtuellen Maschinen

Warum setzt die UPA Campus App auf Docker statt auf klassische virtuelle Maschinen? Der Unterschied liegt in der Virtualisierungsebene und dem Ressourcenverbrauch:

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

Klassische virtuelle Maschinen (VMs) virtualisieren die gesamte Computer-Hardware. Jede VM benötigt ein vollwertiges Gast-Betriebssystem mit eigenem Kernel, Speicher-Management und Systemdiensten. Das bindet pro Instanz mehrere Gigabyte Arbeitsspeicher und führt zu Bootzeiten im Minutenbereich.

Docker-Container teilen sich dagegen den Linux-Kernel des Host-Systems. Zur Kapselung nutzt Docker zwei Kernfunktionen des Linux-Kernels:
* **Namespaces:** Isolieren die Prozessbäume (PID), Netzwerk-Interfaces (NET), Benutzer-IDs (USER) und Dateisystem-Mounts (MNT). Ein Container sieht nur seine eigenen Prozesse und kann andere Container nicht einsehen.
* **Control Groups (cgroups):** Begrenzen und überwachen Hardwareressourcen wie Arbeitsspeicher, CPU-Zyklen und Festplatten-Durchsatz. Ein einzelner speicherintensiver Prozess kann somit nicht den gesamten Host-Server lahmlegen.

Dank dieser Konstruktion starten Container in Sekundenbruchteilen und verbrauchen nur die Ressourcen, die der Anwendungsprozess tatsächlich beansprucht.

Gleichzeitig löst Docker das klassische Problem abweichender Entwickler-Rechner ("Works on my machine"). Ob macOS, Linux oder Windows mit WSL2: Der Container kapselt Python-Laufzeit, Systempakete, Nginx-Konfiguration und Datenbankschema bitgenau ein. Die Laufzeitumgebung auf dem lokalen Arbeitsplatz gleicht dem Produktionsserver zu 100 Prozent.

---

## Die drei Container der UPA Campus App

Die Campus-Infrastruktur gliedert sich in drei spezialisierte Container, die über ein gemeinsames Docker-Netzwerk kommunizieren.

```mermaid
flowchart LR
    Browser["Client Browser"] -->|Port 8085| Nginx["Frontend Container: Nginx / PWA"]
    Nginx -->|Reverse Proxy /api/ Port 8000| FastAPI["Backend Container: FastAPI"]
    FastAPI -->|SQL Port 5432| Postgres[("Datenbank Container: PostgreSQL")]
    Postgres -->|ACID Speicherung| VolPG[("Host Volume: postgres_data")]
    FastAPI -->|Bilder & Dokumente| VolUpload[("Host Volume: uploads_data")]
```

### 1. Datenbank: Offizielles PostgreSQL Image

Für die relationale Datenhaltung dient das offizielle Image `postgres:16-alpine`. Die Wahl eines offiziellen Images bietet klare Vorzüge:
* Keine eigene Wartung von Datenbank-Kompilaten oder Sicherheits-Patches.
* Alpine Linux als schlanke Basis hält die Image-Größe bei unter 100 MB.
* Die Konfiguration erfolgt vollständig über standardisierte Umgebungsvariablen (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`).

Eigene Dockerfiles für relationale Standarddatenbanken sind in aller Regel ein Anti-Pattern, da sie unnötigen Wartungsaufwand erzeugen.

### 2. Backend: Eigenes Dockerfile mit Python und FastAPI

Das Backend verarbeitet REST-Anfragen und setzt Geschäftslogik durch. Die Basis bildet `python:3.12-slim`:

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

Wichtige Architektur-Entscheidungen im Backend-Dockerfile:
* **Effizientes Layer-Caching:** Das Kopieren und Installieren der `requirements.txt` erfolgt vor dem eigentlichen Anwendungscode. Solange sich externe Bibliotheken nicht ändern, überspringt Docker bei Code-Updates die zeitaufwendige Paketinstallation.
* **Privilegien-Minimierung:** Der Prozess läuft unter dem unprivilegierten Benutzer `appuser` statt als `root`. Selbst bei einer Sicherheitslücke in einer Drittbibliothek besitzt der Angreifer keine administrativen Rechte auf dem Container-System.
* **Integrierter Healthcheck:** Docker prüft alle 15 Sekunden über den `/health`-Endpunkt, ob die FastAPI-Instanz Anfragen verarbeitet. Scheitert die Prüfung dreimal hintereinander, markiert Docker den Container als `unhealthy`.

### 3. Frontend: Eigenes Dockerfile mit Nginx

Das Frontend liefert die Web-Admin-Oberfläche als Progressive Web App (PWA) aus und übernimmt das Request-Routing:

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

Nginx erfüllt hier eine Doppelfunktion:
* **Auslieferung statischer Dateien:** HTML, CSS, JavaScript und PWA-Manifeste werden komprimiert mit Gzip ausgeliefert.
* **Reverse Proxy:** Anfragen an `/api/` leitet Nginx transparent an den FastAPI-Container weiter. Der Web-Browser kommuniziert ausschließlich mit dem Frontend-Port, wodurch Cross-Origin Resource Sharing (CORS) Konflikte im Produktivbetrieb vermieden werden.

### 4. Orchestrierung mit Docker Compose

Docker Compose definiert das Gesamtsystem deklarativ in einer zentralen Konfigurationsdatei:

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
    build: ./web-admin
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

Ein entscheidendes Detail ist die Startkoordination:
`depends_on` mit `condition: service_healthy` stellt sicher, dass das Backend erst dann hochfährt, wenn der PostgreSQL-Dienst über `pg_isready` verifiziert bereit ist. Ohne diesen Healthcheck würde die FastAPI-Instanz beim Booten vergeblich versuchen, Datenbank-Migrationen auszuführen, und im Crash-Loop landen.

---

## Sicherheit und Ports: Isolation statt offener Schnittstellen

Ein verbreiteter Fehler im Container-Betrieb ist das wahllose Veröffentlichen interner Ports auf dem Host-System.

```mermaid
flowchart LR
    subgraph PublicInternet["Öffentliches Netz"]
        Attacker["Angreifer / Scanner"]
    end
    subgraph HostServer["Host Server"]
        NetBird["NetBird Tunnel Interface"]
        subgraph DockerBridge["Isoliertes Docker Bridge Netzwerk"]
            Frontend["Frontend (Port 80)"]
            Backend["Backend (Port 8000)"]
            DB[("PostgreSQL (Port 5432)")]
        end
    end
    Attacker -.->|Blockiert: Keine offenen Ports| HostServer
    NetBird -->|Zugriff nur via Tunnel| Frontend
    Frontend -->|Interner Proxy| Backend
    Backend -->|Interne SQL Abfrage| DB
```

In der UPA Campus Architektur gelten zwei strikte Sicherheitsregeln:

### 1. Interne Ports bleiben im Bridge-Netzwerk

Weder der Datenbank-Port 5432 noch der Backend-Port 8000 tauchen im Compose-Abschnitt `ports` auf. Sie sind ausschließlich innerhalb des virtuellen Docker-Netzwerks `campus-network` erreichbar. Die Kommunikation erfolgt über die internen DNS-Namen `postgres:5432` und `backend:8000`. Vom Host-Betriebssystem oder externen Netzwerken aus existieren diese Ports schlichtweg nicht.

### 2. Strikte Bindung an Localhost

Das Frontend exponiert Port 80 nicht mit der Standard-Syntax `8085:80`, sondern bindet ihn explizit an `127.0.0.1:8085:80`. 

Der Hintergrund: Standardmäßig bindet Docker Ports an alle Schnittstellen (`0.0.0.0`). Auf Linux-Servern manipuliert der Docker-Daemon eigenständig die Firewall-Tabellen (`iptables`). Dies führt dazu, dass Container-Ports oft an der lokalen Host-Firewall (wie UFW) vorbei direkt ins offene Internet geroutet werden.

Durch die Bindung an `127.0.0.1` lauscht der Port ausschließlich auf dem lokalen Loopback-Interface. Der Zugriff von außen erfolgt über den verschlüsselten NetBird-Mesh-Tunnel (WireGuard). Der Server besitzt damit im öffentlichen Internet null geöffnete Eingangs-Ports.

---

## Persistenz und Volumes: Flüchtige Container vs. beständiger Speicher

Container sind nach dem Design-Prinzip der 12-Factor-App konzipiert: Sie sind zustandslos und jederzeit ersetzbar.

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Container-Schicht (Flüchtig)"]
        AppCode["Anwendungscode & Bibliotheken"]
        TempFS["Temporäre Schreibschicht (UnionFS)"]
        KillAction["Container Neustart / Image Update"]
        TempFS -.->|Gelöscht beim Neuaufbau| KillAction
    end
    subgraph HostStorage["Host-System (Dauerhaft)"]
        PGVolume[("Docker Volume: postgres_data")]
        UploadVolume[("Docker Volume: uploads_data")]
    end
    ContainerLifecycle -->|Gemountet in /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Gemountet in /app/uploads| UploadVolume
```

### Das flüchtige Container-Dateisystem

Jeder Docker-Container besitzt eine beschreibbare Speicherschicht (OverlayFS / UnionFS). Dateien, die während der Laufzeit im Dateisystem des Containers erzeugt werden, existieren nur in dieser dünnen Schicht.

Wird ein Container über `docker compose down` gestoppt oder durch eine neu gebaute Image-Version ersetzt, wird die beschreibbare Schicht mitsamt ihrem Inhalt unwiderruflich gelöscht. Würde PostgreSQL seine Datenbanktabellen im regulären Container-Dateisystem ablegen, wäre nach jedem Software-Update der gesamte Datenbestand verloren.

### Benannte Docker Volumes für Datenpersistenz

Docker Volumes lösen dieses Problem, indem sie Verzeichnisse des Host-Betriebssystems direkt in den Container einbinden:
* **Trennung von Code und Daten:** Der Container enthält nur Code und Laufzeitumgebung, während alle veränderlichen Daten auf dem Host-Dateisystem (`/var/lib/docker/volumes/`) liegen.
* **Lebenszyklus-Unabhängigkeit:** Das Volume `postgres_data` existiert unabhängig vom Lebenszyklus des Datenbank-Containers. Ein Befehl wie `docker compose up -d --build` vernichtet den alten Container und startet einen neuen, der das bestehende Volume nahtlos wieder an `/var/lib/postgresql/data` bindet.
* **Performance:** Host-Volumes umgehen den Copy-on-Write-Overhead des Container-Dateisystems und erreichen die native I/O-Leistung des Host-Speichermediums.

### Zweiteilung: Relationale Tabellen und Mediendaten

In der Campus-Architektur arbeiten zwei separate Volumes:
1. `postgres_data`: Sichert binäre Datenbank-Dateien, Transaktions-Logs und Indizes.
2. `uploads_data`: Speichert vom Backend empfangene Mediendateien wie PDF-Dokumente oder Ankündigungs-Bilder.

Diese Trennung verhindert, dass große Binärdateien als BLOBs in der Datenbank landen, und hält relationale Datenbank-Backups schlank und schnell.

---

## Fazit

1. **Garantierte Reproduzierbarkeit:** Minimalistische Base-Images, Multi-Stage-Überlegungen und unprivilegierte Benutzer eliminieren Konfigurationsdrifts zwischen Entwicklung und Serverbetrieb.
2. **Defensive Netzwerkarchitektur:** Interne Dienste verbleiben im Docker-Bridge-Netzwerk, während die Bindung an `127.0.0.1` in Kombination mit Mesh-Tunneln jeden direkten Angriff aus dem Internet abwehrt.
3. **Konsequente Datentrennung:** Zustandsloser Anwendungscode in flüchtigen Containern ermöglicht risikofreie Software-Updates, während benannte Volumes relationale Daten und Medienbestände dauerhaft schützen.
