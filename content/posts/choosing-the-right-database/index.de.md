---
title: "Welche Datenbank ist die richtige?"
date: 2026-10-09T10:00:00Z
description: "PostgreSQL als Standardarchitektur, seine Stärken gegen Nischensysteme und die wenigen Szenarien, in denen spezialisierte Datenbanken wirklich berechtigt sind."
summary: "PostgreSQL ist für moderne Softwareprojekte die Standardantwort. Dieser Leitfaden analysiert funktionale und betriebliche Kriterien und zeigt, wann Abweichungen wirklich Sinn ergeben."
tags: ["Datenbanken", "PostgreSQL", "Softwarearchitektur", "Wirtschaftsinformatik"]
categories: ["Enterprise Systems", "Architektur"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
draft: false
cover:
  image: "cover.png"
  alt: "Architektur-Überblick: Der PostgreSQL-Standard und das Datenbank-Ökosystem"
  caption: "Der PostgreSQL-Standard: Single Source of Truth plus spezialisierte Beschleuniger"
  relative: true
---

Welche Datenbank passt zu einem neuen Projekt? In den meisten Fällen PostgreSQL, auch für Dokumente, Geodaten, Volltextsuche und KI-Embeddings. Alles andere braucht eine starke Begründung.

---

## Der Bewertungsmaßstab

Man muss jede Datenbank anhand zweier Kriterien beurteilen:

1. **Funktionale Anforderungen (FA):** Was kann das System fachlich leisten (ACID-Transaktionen, Relationen, flexible Abfragen)?
2. **Nicht-funktionale Anforderungen (NFA):** Was kosten Betrieb, Wartung, Personal und Lizenzen, und wie gut ist das System gegen Vendor Lock-in geschützt?

Funktionale Anforderungen sind heute bei vielen Systemen schnell erfüllt. Die langfristige Wirtschaftlichkeit entscheidet sich jedoch bei den nicht-funktionalen Anforderungen: Wartbarkeit, niedrige Betriebskosten (TCO) und Unabhängigkeit haben in einer nachhaltigen Architektur stets Vorrang vor kurzfristigen Hype-Features.

Das Architektur-Prinzip "Choose Boring Technology" (geprägt bei Etsy) bringt es auf den Punkt: Jedes Team besitzt nur ein kleines Budget an "Innovation Tokens", meist nicht mehr als drei. Wer diese Tokens für experimentelle Datenbanken verbrennt, hat keine Ressourcen mehr für das eigentliche Kerngeschäft. Die Grundregel lautet daher: Wähle die "langweiligste" Technologie, die das Problem zuverlässig löst.

---

## Mit gutem Grund ist PostgreSQL der Standard

PostgreSQL bildet heute das Fundament moderner Softwarearchitektur. Dank seines modularen Aufbaus deckt es Anforderungen ab, für die man früher separate Spezialsysteme betreiben musste, und löst damit viele teure Nischenprodukte ab.

<details open>
<summary>Was PostgreSQL an Spezialsystemen verdrängt hat</summary>

* **Dokumenten-Datenbanken (z. B. MongoDB):** PostgreSQL speichert JSONB binär und durchsucht verschachtelte Attribute über GIN-Indizes extrem schnell.
* **Vektor-Datenbanken (z. B. Pinecone):** Führt hochdimensionale Vektorsuchen mit der Erweiterung `pgvector` und HNSW-Indizes direkt per SQL aus.
* **Geodatenbanken:** Bietet mit `PostGIS` den weltweiten Industriestandard für räumliche Abfragen.
* **Zeitreihen- & IoT-Datenbanken (z. B. InfluxDB):** Schreibt Millionen Sensor-Messwerte über die Erweiterung `TimescaleDB` oder natives Tabellen-Partitioning effizient weg.
* **Graph-Datenbanken:** Löst flache Beziehungsbäume, Organigramme oder Stücklisten über rekursive CTEs direkt in SQL, ganz ohne separate Graph-Engine.
* **Volltext-Suchmaschinen:** Liefert eingebaute linguistische Analysen für Standard-Suchfelder ohne externe Suchinfrastruktur.

</details>

Dazu kommt die Lizenzsicherheit: PostgreSQL unterliegt einer echten Open-Source-Lizenz ohne Eigentümerkonzern, ohne Audit-Fallen und ohne restriktive Klauseln wie die SSPL. Personal mit SQL-Kenntnissen ist am Markt sofort verfügbar.

---

## Die versteckten Kosten von Spezial-Datenbanken

Wer leichtfertig mehrere Spezial-Datenbanken parallel einführt, zahlt die Zeche im laufenden Betrieb:

* **Die Dual-Write-Katastrophe:** Liegen Kunden in PostgreSQL, Produkte in MongoDB und der Suchindex in Elasticsearch, muss die Anwendung jede Änderung in drei Systeme gleichzeitig schreiben. Schlägt der dritte Schreibvorgang fehl, laufen die Datenbestände auseinander. Solche asynchronen Zustände nachträglich zu reparieren, kostet Unsummen.
* **Day-2 Operations:** Eine Datenbank per Docker zu starten dauert fünf Minuten (Day 1). Der eigentliche Aufwand beginnt an Tag 2: Wie funktionieren Backups im laufenden Betrieb? Beherrscht das System Point-in-Time-Recovery (sekundengenaues Zurückspringen bei Fehlern)? Wie aufwendig sind Upgrades und Sicherheitspatches? Für PostgreSQL existieren dafür seit Jahrzehnten ausgereifte Werkzeuge, bei Nischen-Datenbanken baut man den Betrieb oft selbst.

---

## Die Ausnahmen: Wann man ein anderes System wählt

Man unterscheidet strikt zwischen **eigenständigen Primärdatenbanken** (die PostgreSQL ersetzen) und **Zweitdatenbanken** (die PostgreSQL als Hilfssystem entlasten).

### Teil 1: Eigenständige Primärdatenbanken

Hier ersetzt eine andere Datenbank PostgreSQL als zentrale Quelle der Wahrheit im regulären Projektalltag. Extreme Rand- und Hyperscale-Fälle folgen separat in Teil 3.

#### 1. Dateibasiertes SQL (SQLite)

Wenn keine Server-Infrastruktur existiert oder gewollt ist: Mobile-Apps, Desktop-Programme, Edge-Geräte und CLI-Werkzeuge.

SQLite speichert die gesamte Datenbank in einer einzigen Datei. Es benötigt keinen Server-Prozess, verursacht null Betriebskosten und läuft wartungsfrei. Die Grenze ist rein technischer Natur: SQLite erlaubt bauartbedingt immer nur einen schreibenden Prozess zur selben Zeit. Sobald viele Nutzer über das Netzwerk gleichzeitig schreiben, scheidet es aus.

#### 2. Die Hosting-Ausnahme: MariaDB

PostgreSQL verbraucht im Leerlauf etwas mehr Arbeitsspeicher als MariaDB. Bei modernen Cloud-Preisen (ein Server mit 4 GB RAM kostet bei Anbietern wie Hetzner rund 6 Euro im Monat) macht dieser Unterschied betriebswirtschaftlich nur wenige Cent aus. Für diese minimale Ersparnis lohnt sich der funktionale Downgrade bei Eigenentwicklungen nicht.

MariaDB bleibt dennoch berechtigt, wenn äußere Zwänge entscheiden: Standard-Software wie WordPress, restriktive Shared-Hosting-Pakete oder bestehende Betriebsteams, die auf MariaDB spezialisiert sind. MySQL scheidet wegen des Eigentümers Oracle aus, MariaDB ist die offene Alternative.

---

### Teil 2: Zweitdatenbanken zur Entlastung von PostgreSQL

Diese Systeme lösen PostgreSQL niemals ab, sondern arbeiten als spezialisiertes Hilfssystem daneben. Es gibt hier kein Risiko inkonsistenter Daten: PostgreSQL bleibt die alleinige Quelle der Wahrheit (Single Source of Truth), während das Zweitsystem rein zur Leistungssteigerung dient.

#### 3. In-Memory-Cache (Valkey, Redis)

Wenn sehr häufige Lesezugriffe auf identische Daten Antworten in unter einer Millisekunde verlangen.

Das Zusammenspiel aus **PostgreSQL als persistentem Primärspeicher** und **Valkey oder Redis als In-Memory-Datenbank** ist der globale Industriestandard für Hochlast-Anwendungen. Die Anwendung liest Daten bevorzugt aus dem extrem schnellen Arbeitsspeicher (Cache-Aside-Muster). Erst bei einem Cache-Miss wird PostgreSQL angefragt. Invalidiert wird der Cache gezielt bei Datenänderungen.

Beispiel: Eine Produktseite wird 5.000 Mal pro Sekunde aufgerufen, der Inhalt ändert sich jedoch nur einmal pro Stunde. Valkey liefert die Antwort in Mikrosekunden direkt aus dem RAM, ohne Festplattenzugriff und ohne SQL-Parser.

Valkey (unter dem Dach der Linux Foundation) ist für Neuprojekte die lizenzsichere Wahl, nachdem Redis 2024 auf restriktivere Lizenzen umgestellt wurde.

#### 4. Spaltenbasierte Analyse / OLAP (ClickHouse, DuckDB)

Wenn analytische Aggregationen über zweistellige Millionen Zeilen in relationalen Zeilenspeichern zu träge werden.

Spaltendatenbanken können Transaktionsdatenbanken nicht ersetzen: Sie sind auf das blockweise Hinzufügen (Append-Only) optimiert und extrem ineffizient bei einzelnen Zeilen-Updates. Sie dienen daher rein als Analyselager, das asynchron (etwa über periodische ETL-Batches oder Event-Streaming) mit Daten aus PostgreSQL befüllt wird.

Beispiel: Eine Umsatzanalyse über 50 Millionen Verkaufszeilen, aggregiert nach Monat und Region. PostgreSQL liest als zeilenorientierte Datenbank jede Zeile mit allen Spalten von der Festplatte. Ein spaltenorientiertes System wie ClickHouse liest ausschließlich die zwei relevanten Kennzahl-Spalten und verarbeitet Milliarden Datenpunkte in Sekunden. DuckDB bietet denselben Vorteil dateibasiert direkt im Prozess der Anwendung.

---

### Teil 3: Echte Extremfälle und Hyper-Scale

Erst ab extremen physikalischen Schwellenwerten lohnt sich die Einführung eigenständiger Spezialisten:

* **Massiver IoT-Schreibdurchsatz (Apache Cassandra, ScyllaDB):** Dauerhaft mehr als 100.000 Schreibvorgänge pro Sekunde von weltweit verteilten Sensoren. Der Preis dafür ist Eventual Consistency und massive administrative Komplexität. Für normale IoT-Projekte genügt PostgreSQL mit TimescaleDB vollkommen.
* **Horizontales Multi-Node-Sharding (Distributed SQL wie CockroachDB, TiDB):** PostgreSQL skaliert für 99,9 Prozent aller Unternehmen mühelos vertikal (größere CPU/RAM-Server) sowie über Lese-Replikate (Read Replicas für Ausfallsicherheit und Lastverteilung). Echtes horizontales Sharding, bei dem Schreibvorgänge über Dutzende Server weltweit verteilt werden müssen, betreiben fast nur Technologiekonzerne wie Uber, Netflix oder Amazon.
* **Tiefe Netzwerkanalysen (Neo4j):** Nur für Pfadanalysen über mehr als drei Stationen (Geldwäsche-Erkennung, Betrugsnetzwerke).
* **Riesige KI-Plattformen (Qdrant, Milvus):** Erst ab mehr als 20 bis 50 Millionen hochdimensionalen Embeddings notwendig.
* **Proprietäre Legacy-Systeme (Oracle Database, IBM DB2):** Historische Altlasten. Für Neuprojekte wegen horrenden Lizenzkosten, Audit-Risiken und Vendor Lock-in betriebswirtschaftlich nicht mehr zu rechtfertigen.

---

## Fazit

1. **PostgreSQL ist der Standard:** Wer davon abweicht, muss dies technisch oder betriebswirtschaftlich belegen, nicht mit Bauchgefühl oder Hype.
2. **Hilfssysteme ergänzen, Spezialisten sind Verbindlichkeiten:** Die Kombination aus PostgreSQL und einem In-Memory-Cache (Valkey) deckt fast jedes Skalierungsproblem ab. Zusätzliche Spezialdatenbanken werden erst eingeführt, wenn das Primärsystem messbar kapituliert.
3. **Betriebskosten schlagen Lizenzkosten:** Jede zusätzliche Datenbank erfordert eigenes Monitoring, Backup-Prozesse und Patch-Routinen. PostgreSQL deckt den Großteil aller Softwareprojekte mit einem einzigen, beherrschbaren Betriebskonzept ab.
