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

Ein Coding-Agent (wie Claude Code oder ein CLI-basierter Agent) kann direkt im Projektordner arbeiten, die TMDL-Strukturen analysieren und neue Measures in den Code schreiben. Das funktioniert sofort ohne zusätzliche Middleware oder geöffnete Software.

### Reale Einschränkungen von Pfad 1

Trotz der perfekten Eignung für Git und CI/CD hat dieser rein dateibasierte Ansatz klare Nachteile:

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

3. **Verbindung zur Sitzung:** Power BI Desktop mit dem Datenmodell öffnen, egal ob als `.pbix` oder als `.pbip` gespeichert. Ein Prompt an die KI genügt: *"Verbinde dich mit meiner aktiven Power BI Sitzung."*

![MCP Server aktiv in Claude Desktop](claude_mcp_running.jpg)
*Der Power BI Modeling MCP Server aktiv verbunden in Claude Desktop.*

Da Power BI Desktop für jedes geöffnete Modell im Hintergrund eine lokale Analysis-Services-Instanz startet, dockt der MCP-Server direkt an diesen lokalen Port an. Die KI erhält konkrete Werkzeuge: Schema abfragen, Measures anlegen und DAX-Abfragen direkt ausführen. Meldet die Engine einen Syntaxfehler, erhält die KI die Fehlermeldung im selben Schritt und korrigiert den Code selbstständig.

---

## Pfad 3: Microsoft Copilot für Power BI

Microsoft bietet mit Copilot auch eine direkt integrierte KI-Funktion in der Cloud-Oberfläche und im Desktop an.

* **Fokus auf Canvas und Layout:** Copilot erstellt fertige Visuals und Berichtsseiten direkt auf der Canvas im Microsoft-Ökosystem, eignet sich aber kaum für tiefe semantische Modellierung.
* **Cloud-gebunden:** Auch im Desktop läuft die Berechnung immer in der Microsoft Cloud und setzt eine aktive Fabric-Kapazität (mindestens F64 SKU) im Mandanten voraus.
* **Kosten und Vendor Lock-in:** Hohe Plattformkosten für dedizierte Kapazitäten und ein geschlossenes System ohne Eingriffsmöglichkeiten in System-Prompts oder externe Entwickler-Tools.

---

## Können alle drei Wege alles? Der direkte Fähigkeiten-Vergleich

Kein Werkzeug deckt alle Anforderungen ab. Die drei Ansätze unterscheiden sich in ihren Stärken und Grenzen grundlegend:

| Anforderung / Funktion | Pfad 1: PBIP (Dateiebene) | Pfad 2: MCP Server (Live) | Pfad 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Unterstützte Formate** | Nur PBIP (Dateisystem) | Sowohl PBIX als auch PBIP | Sowohl PBIX als auch PBIP |
| **DAX-Measures erstellen** | Ja (Stapelverarbeitung in TMDL) | Ja (direkt im Modell injiziert) | Ja (über Chat-Prompt) |
| **Live-Syntaxprüfung** | Nein (blinde Textbearbeitung) | Ja (sofortiges Engine-Feedback) | Eingeschränkt (nur Heuristik) |
| **DAX-Testabfragen (`execute_dax`)** | Nein (keine laufende Engine) | Ja (direkte Abfrage der Engine) | Nein |
| **Git-Versionskontrolle & CI/CD** | Exzellent (reine Text-Diffs) | Bei PBIP nach Speichern nutzbar | Keine (Cloud-gebunden) |
| **Hardware-Anforderungen** | Minimal (CLI oder Code-Editor) | Hoch (Desktop-App und RAM) | Keine (Cloud-Hosting) |
| **Datenschutz** | Abhängig vom gewählten LLM | Abhängig vom gewählten LLM | Gespeichert in Microsoft Cloud |
| **DirectQuery-Unterstützung** | Eingeschränkt (nur Metadaten) | Komplex (Abfrage-Latenz) | Ja (native Cloud-Unterstützung) |
| **Visuelles Layout & Diagramme** | Eingeschränkt (blinde JSON-Bearbeitung) | Nein (reiner Modellierungsfokus) | Ja (erstellt Visuals auf der Canvas) |
| **Individuelle SVG-Karten & HTML** | Eingeschränkt (blinder DAX-Code) | Exzellent (sofortige visuelle Vorschau) | Unbrauchbar (nur Standard-Visuals) |
| **Kosten & Modell-Freiheit** | Kostenlos (jedes LLM / lokale Modelle) | Kostenlos (jeder MCP-Client) | Hoch (Fabric F64 oder User-Lizenz) |
| **Laufzeit-Voraussetzung** | Nur Code-Editor / CLI nötig | Power BI Desktop muss lokal laufen | Aktives Fabric Cloud-Abonnement |

*Zusammenfassend eignet sich Pfad 1 für Headless-Skripte ohne Setup, Pfad 2 für die aktive Entwicklung mit Live-Validierung, und Pfad 3 für standardisierte Canvas-Berichte im Microsoft-Ökosystem.*

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

1. **PBIP ist das Pflichtfundament:** Wer Datenmodelle professionell in Git versionieren will, muss das binäre PBIX-Format verlassen. Versionskontrolle ist eine Eigenschaft des Dateiformats, nicht des KI-Tools.
2. **MCP entscheidet die Entwicklung für sich:** Reine Dateisystem-Agenten scheitern an fehlender Syntax-Validierung, während Copilot ein teures Komfort-Werkzeug für Standard-Visuals bleibt. Für anspruchsvolle semantische Modellierung, komplexe DAX-Logik und maßgeschneiderte SVG-Karten ist die Live-Verbindung über MCP der mit Abstand produktivste Weg.

---

Ein Dank geht an **Matías Ciancio** für die Architektur-Visualisierung und den fachlichen Austausch.
