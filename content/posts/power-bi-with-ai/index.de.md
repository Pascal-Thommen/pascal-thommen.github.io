---
title: "Power BI mit KI: PBIP und MCP in der Praxis"
date: 2026-10-06T18:00:00Z
description: "Wie verbindet man Power BI mit KI-Agenten? Ein technischer Blick auf PBIP-Dateien, den Power BI Modeling MCP Server, Datenbankanbindungen und Grenzen."
summary: "Beim 3. Hackathon für Business Intelligence und KI an der Universidad Americana stand eine zentrale Frage im Raum: Wie steuert man ein Power-BI-Modell verlässlich mit KI-Agenten? Ein technischer Vergleich von PBIP, MCP und Copilot."
tags: ["Power BI", "Künstliche Intelligenz", "MCP", "Business Intelligence", "Wirtschaftsinformatik"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-mit-ki/"
  - "/de/posts/power-bi-mit-ki/"
---

Beim 3. Hackathon für Business Intelligence und KI an der Universidad Americana stand eine zentrale Frage im Raum: Wie steuert man ein Power-BI-Modell verlässlich mit KI-Agenten?

![Teilnehmer und Mentoren beim 3. Hackathon für Business Intelligence und KI an der Universidad Americana](hackathon_group.jpg)
*Teilnehmer und Mentoren beim 3. Hackathon für Business Intelligence und KI an der Universidad Americana (3. Oktober 2026).*

Drei Ansätze stehen in der Praxis zur Auswahl:
* **Dateiebene (PBIP):** Direkte Bearbeitung von TMDL-Textdateien im Dateisystem mit Coding-Agenten. Funktioniert sofort ohne jegliche Einrichtung und ohne laufendes Power BI Desktop.
* **Live-Sitzung (MCP):** Direkte Verbindung zur lokalen Analysis-Services-Engine in Power BI Desktop für interaktive Modellierung mit Live-Fehlerprüfung und DAX-Testabfragen.
* **Cloud (Microsoft Copilot):** Nativ in Microsoft Fabric integriert für die automatisierte Erstellung von Berichtsseiten und Visuals direkt auf der Canvas im Microsoft-Ökosystem.

---

## Architektur-Überblick: Wie Daten und KI zusammenspielen

Bevor man über KI spricht, muss die Datenebene geklärt sein.

Power BI übernimmt die direkte Anbindung an die Datenquellen (SQL Server, PostgreSQL, ERP-Systeme oder Data Warehouses) über Power Query, DirectQuery oder geplante Importe. Der KI-Agent benötigt **keinen direkten Zugriff auf die Produktivdatenbank** und keine Datenbank-Zugangsdaten. Stattdessen operiert die KI ausschließlich auf der semantischen Modellschicht (Tabellenbeziehungen, Kennzahlen und DAX-Logik).

![Architektur-Überblick: Dateibasierte Modellierung versus Live-Sitzung](architecture_diagram.png)
*Architektur-Überblick: Dateibasiertes Modellieren via PBIP (Pfad 1) versus Live-Sitzung via MCP (Pfad 2) und Deployment in den Power BI Service.*

Diese Trennung bietet einen wesentlichen Governance-Vorteil: Bestehende Sicherheitsrichtlinien, Rollenrechte und Firewalls verbleiben vollständig in Power BI. Die KI formt lediglich die Berechnungslogik und die visuelle Aufbereitung.

---

## Pfad 1: Dateibasierte Modellierung über PBIP (TMDL)

Klassische `.pbix`-Dateien sind binäre ZIP-Archive, die für KI-Modelle unlesbar sind. Speichert man den Bericht als `.pbip` (Power BI Project), zerlegt Power BI das Modell in Klartext:

* **Semantisches Modell:** Tabellen, Relationen und DAX-Measures liegen als TMDL-Dateien (Tabular Model Definition Language) vor.
* **Berichtsdefinition:** Diagramme, Formatierungen und Filterstrukturen liegen als strukturierte JSON-Dateien vor.

Ein Coding-Agent (wie Claude Code oder ein CLI-basierter Agent) kann direkt im Projektordner arbeiten, die TMDL-Strukturen analysieren und neue Measures in den Code schreiben. Das funktioniert sofort ohne zusätzliche Middleware, ohne API-Einrichtung und ohne laufendes Power BI Desktop.

### Reale Einschränkungen von Pfad 1

Obwohl das PBIP-Dateiformat die ideale Klartext-Grundlage bietet, hat die reine Dateibearbeitung ohne laufende Engine entscheidende Nachteile:

* **Keine Live-Syntaxprüfung:** Der Agent schreibt DAX-Formeln blind in die Textdateien. Syntaxfehler fallen erst auf, wenn das Projekt in Power BI Desktop geöffnet oder neu geladen wird.
* **Keine Testabfragen:** Der Agent kann keine `execute_dax`-Abfragen ausführen, um zu prüfen, ob die Berechnung mit den echten Daten übereinstimmt.
* **Manuelles Neuladen:** Änderungen auf der Festplatte werden nicht automatisch im geöffneten Desktop-Fenster synchronisiert.

---

## Pfad 2: Live-Sitzungssteuerung über MCP

Wenn interaktive Modellierung mit sofortiger Fehlerprüfung gefragt ist, schlägt das Model Context Protocol (MCP) die Brücke.

Die Visual Studio Code Extension **Power BI Modeling MCP Server** bündelt eine eigenständige Executable (`powerbi-modeling-mcp.exe`), die auf den Microsoft Analysis Services Bibliotheken aufsetzt.

![Power BI Modeling MCP Server in den VS Code Extensions](vscode_mcp_extension.jpg)
*Die Power BI Modeling MCP Server Extension im Visual Studio Code Marketplace.*

### Das Setup im Detail

1. **Executable lokalisieren:** Die Extension installiert die `powerbi-modeling-mcp.exe` lokal im VS Code Erweiterungsordner.
2. **KI-Client konfigurieren:** In der `claude_desktop_config.json` (oder jedem anderen MCP-kompatiblen Client) wird der Server mit dem Argument `--start` hinterlegt:

```json
{
  "mcpServers": {
    "powerbi-modeling-mcp": {
      "command": "C:\\Users\\username\\.vscode\\extensions\\analysis-services.powerbi-modeling-mcp-0.4.0-win32-x64\\server\\powerbi-modeling-mcp.exe",
      "args": ["--start"],
      "env": {}
    }
  }
}
```

3. **Verbindung zur Sitzung:** Power BI Desktop mit dem Datenmodell öffnen. Das funktioniert sowohl mit traditionellen `.pbix`-Dateien als auch mit modernen `.pbip`-Projekten. Ein Prompt an die KI genügt: *"Verbinde dich mit meiner aktiven Power BI Sitzung."*

![MCP Server aktiv in Claude Desktop](claude_mcp_running.jpg)
*Der Power BI Modeling MCP Server aktiv verbunden in Claude Desktop.*

Da Power BI Desktop für jedes geöffnete Modell im Hintergrund eine lokale Analysis-Services-Instanz startet, dockt der MCP-Server direkt an diesen lokalen Port an. Die KI erhält konkrete Werkzeuge: Schema abfragen, Measures anlegen und DAX-Abfragen direkt ausführen. Meldet die Engine einen Syntaxfehler, erhält die KI die Fehlermeldung im selben Schritt und korrigiert den Code selbstständig.

Wird ein PBIP-Projekt auf diesem Weg bearbeitet und anschließend in Power BI Desktop gespeichert, landen alle von der KI erstellten Measures sauber in den TMDL-Dateien auf der Festplatte. Man kombiniert damit die Live-Validierung der Engine mit der vollen Git-Versionskontrolle von PBIP.

---

## Pfad 3: Microsoft Copilot für Power BI

Microsoft bietet mit Copilot auch eine direkt integrierte KI-Funktion in der Cloud-Oberfläche und im Desktop an.

* **Fokus auf Ökosystem und Canvas:** Der größte Vorteil von Copilot liegt im Microsoft-Ökosystem: Governance, Compliance und die automatisierte Erstellung von Berichtsseiten und Visuals direkt auf der Canvas. Für anspruchsvolle semantische Datenmodellierung und komplexe DAX-Logik ist das System hingegen kaum ausgelegt.
* **Immer cloud-gebunden:** Auch bei der Nutzung in Power BI Desktop läuft die Berechnung niemals lokal auf dem Rechner. Sämtliche Abfragen werden in der Microsoft Cloud verarbeitet und setzen eine zugewiesene Microsoft Fabric Kapazität (mindestens F64 SKU) im Mandanten voraus.
* **Kosten und Plattform-Bindung:** Dedizierte Fabric-Kapazitäten bedeuten erhebliche monatliche Plattformkosten. Zudem bleibt Copilot ein geschlossenes System ohne Eingriffsmöglichkeiten in System-Prompts oder externe Entwickler-Tools.

---

## Können alle drei Wege alles? Der direkte Fähigkeiten-Vergleich

Kein Werkzeug deckt alle Anforderungen ab. Die drei Ansätze unterscheiden sich in ihren Stärken und Grenzen grundlegend:

| Anforderung / Funktion | Pfad 1: PBIP (Dateiebene) | Pfad 2: MCP Server (Live) | Pfad 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Unterstützte Dateiformate** | Nur PBIP (Dateisystem) | Sowohl PBIX als auch PBIP | Sowohl PBIX als auch PBIP |
| **DAX-Measures erstellen** | Ja (Stapelverarbeitung in TMDL) | Ja (direkt im Modell injiziert) | Ja (über Chat-Prompt) |
| **Live-Syntaxprüfung** | Nein (blinde Textbearbeitung) | Ja (sofortiges Engine-Feedback) | Eingeschränkt (nur Heuristik) |
| **DAX-Testabfragen (`execute_dax`)** | Nein (keine laufende Engine) | Ja (direkte Abfrage der Engine) | Nein |
| **Git-Versionskontrolle & CI/CD** | Exzellent (reine TMDL-Text-Diffs) | Bei PBIP nach Speichern voll nutzbar | Keine direkte Git-Integration |
| **Hardware-Anforderungen** | Minimal (CLI oder Code-Editor) | Hoch (Desktop-App und RAM) | Keine (Cloud-Hosting) |
| **Datenschutz & Inferenz-Ort** | Abhängig vom gewählten LLM | Abhängig vom gewählten LLM | Immer in Microsoft Cloud |
| **Visuelles Layout auf Canvas** | Eingeschränkt (blinde JSON-Bearbeitung) | Nein (reiner Modellierungsfokus) | Ja (erstellt Visuals auf der Canvas) |
| **Individuelle SVG-Karten & HTML** | Eingeschränkt (blinder DAX-Code) | Exzellent (sofortige visuelle Vorschau) | Unbrauchbar (nur Standard-Visuals) |
| **Kosten & Modell-Freiheit** | Kostenlos (jedes LLM / lokale Modelle) | Kostenlos (jeder MCP-Client) | Hoch (Fabric F64 oder User-Lizenz) |
| **Laufzeit-Voraussetzung** | Nur Code-Editor / CLI nötig | Power BI Desktop muss lokal laufen | Aktive Fabric Cloud-Kapazität |

### Klare Entscheidungshilfe für die Praxis

* **PBIP als Dateiformat wählen**, sobald Versionskontrolle in Git, Teamarbeit und CI/CD-Pipelines gefordert sind. Das ist eine Format-Entscheidung, keine Tool-Entscheidung.
* **Pfad 1 (Dateiebene) wählen**, wenn man ohne jegliche Einrichtung, ohne geöffnete Desktop-App oder in automatisierten Headless-Skripten Measures stapelweise generieren will.
* **Pfad 2 (MCP Live-Sitzung) wählen**, wenn man aktiv am Arbeitsplatz modelliert und sofortiges Feedback der Engine, Syntax-Fehlerkorrektur und DAX-Testabfragen benötigt.
* **Pfad 3 (Microsoft Copilot) wählen**, wenn fertige Berichts-Layouts direkt auf der Canvas im regulierten Microsoft-Ökosystem generiert werden sollen und Fabric-Kapazitäten bereitstehen.

---

## Praxisnutzen und Reality Check

### Wo das Hackathon-Setup überzeugt hat

* **Dynamische SVG-Measures:** Die Kombination aus dem HTML-Visual in Power BI und DAX-Measures, die dynamischen SVG-Code erzeugen. Individuelle KPI-Karten und Statusanzeigen von Hand in DAX-SVG zu codieren dauert Stunden. Eine KI liefert diese Berechnungen in Sekunden.
* **Measure-Scaffolding:** Bei einem definierten semantischen Modell erstellt der Agent Dutzende Standard-Measures (Vorjahresvergleiche, Margen, rollierende Durchschnitte) in einem Durchgang.

### Der Reality Check

* **Hoher Token-Verbrauch:** Die Übermittlung des Modells, der Tabellenschemata und des Geschäftskontexts frisst große Mengen an Kontext-Tokens. Kostenlose API-Limits sind schnell erschöpft.
* **Datenbereinigung bleibt Menschenarbeit:** Eine KI repariert kein unsauberes Datenmodell. Wenn die Datenqualität vor der Modellierung nicht stimmt, produziert die KI lediglich falsche Zahlen in höherer Geschwindigkeit.

---

## Fazit

1. **PBIP ist das Pflichtfundament:** Wer Datenmodelle professionell in Git versionieren und in CI/CD einbinden will, muss das binäre PBIX-Format verlassen. Versionskontrolle ist eine fundamentale Eigenschaft des Dateiformats, nicht des verwendeten KI-Tools.
2. **MCP entscheidet die Entwicklung für sich:** Reine Dateisystem-Agenten scheitern an fehlender Syntax-Validierung, während Copilot ein teures Komfort-Werkzeug für Standard-Visuals bleibt. Für anspruchsvolle semantische Modellierung, komplexe DAX-Logik und maßgeschneiderte SVG-Karten ist die Live-Verbindung über MCP der mit Abstand produktivste Weg.

---

Ein Dank geht an **Matías Ciancio** für die Architektur-Visualisierung und den fachlichen Austausch.
