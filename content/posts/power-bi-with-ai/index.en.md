---
title: "Power BI with AI: PBIP and MCP in Practice"
date: 2026-10-06T18:00:00Z
description: "How do you connect Power BI with AI agents? A hands-on look at PBIP files, the Power BI Modeling MCP Server, database connections, and limitations."
summary: "At the 3rd Business Intelligence & AI Hackathon, one question took center stage: How do you reliably control a Power BI model with AI agents? A technical comparison of PBIP, MCP, and Copilot."
tags: ["Power BI", "Artificial Intelligence", "MCP", "Business Intelligence", "Business Informatics"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-mit-ki/"
  - "/en/posts/power-bi-mit-ki/"
---

At the 3rd Business Intelligence & AI Hackathon at Universidad Americana, one question took center stage: How do you reliably control a Power BI model with AI agents?

![Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana](hackathon_group.jpg)
*Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana (October 3, 2026).*

Three distinct paths exist in practice:
* **Headless Filesystem Agents (File Level):** Directly editing TMDL text files (PBIP) on disk with coding agents. Runs instantly without opening Power BI Desktop and without extra middleware.
* **Live Session Control (MCP):** Connecting directly to the local engine via the Analysis Services MCP server for interactive modeling with real-time error checking (works with both PBIX and PBIP).
* **Platform-Integrated Assistants (Microsoft Copilot):** Cloud-based assistants within the Microsoft ecosystem for generating report pages and canvas visuals.

For active semantic modeling, Path 2 (MCP) is by far the strongest approach because the running engine validates DAX syntax and query results immediately. Path 1 offers complete headless independence without any local setup, while Path 3 delivers its primary value through automated report page layout inside the corporate Microsoft ecosystem.

---

## Architecture Overview: How Data and AI Connect

Before examining AI tools, the data connection layer needs clarity.

Power BI connects directly to your data sources (SQL Server, PostgreSQL, ERP systems, or cloud data warehouses) through Power Query, DirectQuery, or scheduled imports. The AI agent does not require direct access to your production database. Instead, the agent operates entirely on the semantic model layer (tables, relationships, and DAX calculations).

![Architecture Overview: File-Based Modeling vs. Live Session](architecture_diagram.png)
*Architecture overview: File-based modeling via PBIP (Path 1) versus live session modeling via MCP (Path 2) and deployment to Power BI Service.*

This separation provides a crucial governance advantage: existing enterprise access controls, firewalls, and data source permissions remain enforced by Power BI. The AI only shapes how data is modeled and aggregated.

---

## Path 1: Headless Filesystem Modeling on TMDL Files

The standard `.pbix` format is a compressed binary archive. An AI model cannot inspect or modify binary blobs. Saving a project as `.pbip` (Power BI Project) splits the report into human-readable plain text:

* **Semantic Model:** Tables, relationships, and DAX measures are stored in TMDL (Tabular Model Definition Language) files.
* **Report Definition:** Visuals, layouts, and filters are stored in JSON format.

The decisive advantage of Path 1 is zero setup: coding agents (such as Claude Code, Cursor, or CLI scripts) can work immediately on any operating system, including headless Linux environments. They analyze TMDL files and append measures without requiring Power BI Desktop to be installed or open.

### Practical Limitations of Path 1

Operating purely on disk without a live engine introduces clear constraints:

* **Blind syntax generation:** The agent writes DAX formulas into text files blindly. Syntax errors are only caught when you open or reload the project in Power BI Desktop.
* **No test execution:** The agent cannot run `execute_dax` queries to verify calculation results against actual data.
* **Manual reload required:** Changes made to TMDL files on disk require reloading or reopening Power BI Desktop to reflect in the UI.

---

## Path 2: Live Session Control via MCP

When you need interactive modeling with instant validation, the Model Context Protocol (MCP) bridges the gap.

Microsoft provides the **Power BI Modeling MCP Server** extension for Visual Studio Code. This extension packages a dedicated executable (`powerbi-modeling-mcp.exe`) built on Microsoft Analysis Services libraries.

![Power BI Modeling MCP Server in VS Code Extensions](vscode_mcp_extension.jpg)
*The Power BI Modeling MCP Server extension in the Visual Studio Code Marketplace.*

### How the Setup Works

1. **Locate the Executable:** The extension installs `powerbi-modeling-mcp.exe` inside your local VS Code extension folder.
2. **Configure the AI Client:** In `claude_desktop_config.json` (or any MCP-compatible client), register the server path with the `--start` argument:

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

3. **Connect to the Active Session:** Open Power BI Desktop with your data model, saved as either `.pbix` or `.pbip`. Then prompt the AI: *"Connect to my active Power BI session."*

![MCP Server running in Claude Desktop](claude_mcp_running.jpg)
*The Power BI Modeling MCP Server connected and running inside Claude Desktop.*

Because Power BI Desktop runs a local Analysis Services instance in the background for any open report, the MCP server attaches directly to its local port. The AI receives functional tools: inspecting tables, creating measures, and running DAX test queries. If a formula fails, the engine returns the error message immediately, allowing the AI to self-correct in real time.

---

## Path 3: Microsoft Copilot for Power BI

Microsoft offers Copilot as a built-in AI assistant across Power BI Desktop and the Power BI Service.

* **Cloud-Bound Even on Desktop:** While Copilot is accessible as a side pane in Power BI Desktop for both `.pbix` and `.pbip` files, processing never happens locally. Prompts and metadata travel to the Microsoft Cloud. Without an active connection and assigned Fabric capacity (F64 SKU minimum) in the tenant, the feature is disabled.
* **Core Advantage: Ecosystem Integration and Canvas Layout:** Copilot is not built for granular semantic modeling. Its true value lies in enterprise compliance and the ability to generate complete report pages and canvas visuals automatically, something neither Path 1 nor Path 2 can do.
* **Platform Lock-In and High Costs:** Dedicated Fabric capacities represent a significant financial barrier compared to open LLM APIs, with zero control over underlying system prompts or developer tooling.

---

## Can All Three Approaches Do Everything? Direct Capability Comparison

No single tool covers the entire workflow. Each approach has distinct strengths and clear limitations:

| Capability / Requirement | Path 1: Headless Filesystem | Path 2: MCP Server (Live) | Path 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Supported File Formats** | Strictly PBIP (TMDL plain text) | Both PBIX and PBIP | Both PBIX and PBIP |
| **Initial Setup Effort** | Zero (works instantly in any editor) | Medium (VS Code extension & config) | Low if licensed, otherwise prohibitive |
| **DAX Measure Creation** | Yes (batch plain text in TMDL) | Yes (direct injection into engine) | Yes (chat prompt) |
| **Live Syntax Validation** | No (blind text edits) | Yes (instant engine feedback) | Partial (heuristics only) |
| **DAX Query Testing (`execute_dax`)** | No (no running engine) | Yes (direct engine query) | No |
| **Git & Version Control** | Native to PBIP files | Available with PBIP after saving | None (tied to Microsoft cloud) |
| **Hardware Requirements** | Minimal (CLI or code editor) | High (Desktop app and RAM) | Zero (cloud-hosted) |
| **Data Privacy** | Depends on chosen LLM | Depends on chosen LLM | Stored in Microsoft Cloud |
| **DirectQuery Support** | Limited (metadata only) | Complex (query latency) | Yes (native cloud support) |
| **Visual Canvas Layout** | Limited (blind JSON edits) | No (modeling focus only) | Yes (generates canvas visuals) |
| **Custom SVG Cards & HTML Visuals** | Limited (blind DAX string) | Excellent (live visual preview) | Poor (standard visuals only) |
| **Cost & Model Flexibility** | Free (any LLM or local model) | Free (any MCP client) | High (Fabric F64 or user license) |
| **Runtime Requirement** | Code editor only (CLI) | Power BI Desktop open locally | Active Fabric cloud subscription |

*In summary: Path 1 excels at headless scripting without setup, Path 2 is the clear choice for active modeling with live validation, and Path 3 automates canvas layout within the Microsoft ecosystem.*

---

## Practical Strengths and Reality Check

### Where the Hackathon Setup Excelled

* **Dynamic SVG Measures:** Building custom KPI cards by combining Power BI's HTML visual with DAX measures that generate dynamic SVG code. Crafting complex SVG strings manually takes hours. An AI generates working DAX SVG measures in seconds.
* **Measure Scaffolding:** Generating batches of standard measures (YoY growth, moving averages, period-to-date) across an established semantic model in one pass.

### The Reality Check

* **High Token Usage:** Passing the entire semantic model, relationship graph, and business context consumes a large volume of tokens. Free API tiers run out quickly.
* **Data Hygiene Precedes Automation:** An AI cannot fix a broken data model. If upstream data cleaning and business definitions are unclear, automated measures will only produce faster incorrect numbers.

---

## Conclusion

Two central findings emerge for practical engineering:

1. **PBIP is the mandatory foundation:** Versioning data models in Git requires moving away from binary PBIX files. Version control is a property of the file format, not of the AI tool.
2. **MCP decisively wins active modeling:** Plain filesystem agents lack syntax validation, while Copilot remains an expensive convenience tool for generic canvas visuals. For precise semantic modeling, complex DAX logic, and dynamic SVG cards, live engine connection via MCP is the most productive approach by far.

---

Special thanks to **Matías Ciancio** for the architecture visualization and technical exchange.

