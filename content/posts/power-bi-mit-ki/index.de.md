---
title: "Power BI mit KI steuern: Wie das Setup über MCP und PBIP in der Praxis funktioniert"
date: 2026-10-06T18:00:00Z
description: "Wie verbindet man Claude mit Power BI? Kein Hype, sondern das echte Setup vom Hackathon: PBIP-Dateisystem vs. Power BI Modeling MCP Server."
summary: "Am Hackathon wollten viele Teilnehmer und Mentoren wissen: Wie steuert ihr Power BI live mit Claude? Hier ist das reale Setup: PBIP-Dateien, Microsofts MCP-Server und warum die Vorarbeit an den Daten trotzdem entscheidend bleibt."
tags: ["Power BI", "Künstliche Intelligenz", "MCP", "Business Intelligence", "Wirtschaftsinformatik"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: true
ShowBreadCrumbs: true
---

Am Hackathon in Asunción kam nach unserer Präsentation immer wieder dieselbe Frage auf, sowohl von Teilnehmern als auch von Mentoren: *Wie habt ihr Power BI eigentlich an Claude angebunden? Läuft das über MCP oder wie funktioniert die Verbindung?*

Die Antwort ist simpel, wenn man die Architektur versteht. Es gibt keine Zauberei, sondern genau zwei praxistaugliche Wege: den direkten Dateizugriff über das PBIP-Format und die Live-Verbindung über das Model Context Protocol (MCP).

Hier ist der Überblick, wie das Setup in der Realität aufgebaut ist, wo die Werkzeuge glänzen und wo die Grenzen liegen.

---

## Weg 1: Der Dateisystem-Weg (Claude Code auf PBIP)

Der einfachste Weg kommt komplett ohne MCP aus. Die Voraussetzung dafür ist das moderne Projektformat von Microsoft: `.pbip` (Power BI Project).

Klassische `.pbix`-Dateien sind binäre Archive. Eine KI kann damit nichts anfangen. Speichert man den Bericht jedoch als `.pbip`, zerlegt Power BI das gesamte Projekt in lesbare Klartextdateien:

1. **Das semantische Modell:** Tabellen, Beziehungen und DAX-Measures liegen als TMDL (Tabular Model Definition Language) in separaten Textdateien.
2. **Die Berichtsdefinition:** Diagramme, Filter und visuelle Layouts liegen als JSON-Dateien vor.

Wenn dieses Projekt auf der Festplatte liegt, braucht man kein spezielles Protokoll. Man startet ein Terminal mit Claude Code, verweist auf den Projektordner und lässt das Modell analysieren. Die KI liest die TMDL-Dateien, versteht das Schema und schreibt neue DAX-Measures direkt in den Quelltext. Git versioniert jeden Schritt sauber mit.

---

## Weg 2: Der Live-Session-Weg (Claude Desktop + Power BI Modeling MCP)

Möchte man interaktiv arbeiten und direkt in ein geöffnetes Power BI Desktop hineingreifen, kommt das Model Context Protocol (MCP) ins Spiel. 

Microsoft stellt dafür eine Brücke bereit, die viele noch gar nicht auf dem Schirm haben: die Visual Studio Code Extension **Power BI Modeling MCP Server**.

### Das Setup unter der Haube

1. **Server-Executable:** Mit der VS Code Extension liefert Microsoft eine eigenständige Binärdatei mit (`powerbi-modeling-mcp.exe`).
2. **Claude Desktop Konfiguration:** In der Datei `claude_desktop_config.json` wird dieser Server unter dem Schlüssel `mcpServers` eingetragen. Man übergibt den Pfad zur Executable und das Start-Argument `--start`.
3. **Aktive Sitzung:** Sobald Power BI Desktop mit einem Datenmodell geöffnet ist, läuft im Hintergrund eine lokale Analysis-Services-Instanz.
4. **Befehl an Claude:** Ein Prompt wie *"Verbinde dich mit meiner aktiven Power BI Sitzung"* reicht aus. Der MCP-Server klinkt sich an den lokalen Port an.

Ab diesem Moment verfügt Claude über Werkzeuge: Das Modell abfragen, Tabellen inspizieren, DAX-Measures erstellen und Abfragen direkt gegen die Engine ausführen. Fehler in einer DAX-Formel werden sofort von der Engine zurückgemeldet, sodass die KI die Syntax eigenständig korrigiert.

---

## Wo das Setup glänzt: Maßgeschneiderte SVGs und Scaffolding

Der größte Hebel liegt nicht darin, Standard-Balkendiagramme zu bauen. Die kann man in Power BI auch schnell zusammenklicken.

Richtig stark wird die Kombination bei komplexen, individuellen Anforderungen:
* **Dynamische SVG-Karten:** Kombiniert man das HTML-Visual in Power BI mit DAX-Measures, die dynamischen SVG-Code generieren, lassen sich maßgeschneiderte KPI-Karten und Fortschrittsanzeigen bauen. Das von Hand zu schreiben dauert ewig. Eine KI generiert den DAX-SVG-Code in Sekunden.
* **Scaffolding:** Hat man ein klares semantisches Modell, baut der MCP-Server Dutzende Standard-Kennzahlen (Time Intelligence, YoY, Margen) in einem einzigen Durchlauf auf.

---

## Der Reality Check: Token-Hunger und Datenhygiene

In der Praxis muss man zwei Faktoren realistisch einordnen:

1. **Hoher Token-Verbrauch:** Damit die KI sinnvolle Berechnungen anstellt, muss sie das semantische Modell, Tabellenschemata und den Geschäftskontext verarbeiten. Das schluckt massiv Kontext-Tokens. Ein kleiner kostenloser API-Zugang stößt hier schnell an Limits.
2. **Die Vorarbeit bleibt menschliche Arbeit:** Die meiste Zeit eines BI-Projekts fließt in Datenbereinigung, Validierung und das Verständnis der Geschäftsprozesse. Wenn das Datenmodell Müll ist, hilft auch der beste MCP-Server nichts.

Das Werkzeug nimmt einem das Tippen und die Syntax-Arbeit ab, aber nicht das logische Verständnis.

---

## Fazit: Business Intelligence as Code

Die Kombination aus PBIP und MCP verändert die Arbeitsweise grundlegend. Power BI verwandelt sich von einer geschlossenen Klick-Umgebung in ein quelltextbasiertes System, das sich nahtlos in moderne Entwickler-Workflows und KI-Pipelines einfügt.

Für die Wirtschaftsinformatik ist das genau die richtige Richtung: Mehr Automatisierung bei Routinen, volle Nachvollziehbarkeit im Code und die volle Kontrolle über die Business-Logik.
