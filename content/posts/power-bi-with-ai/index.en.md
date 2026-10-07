---
title: "Power BI with AI: PBIP and MCP in Practice"
date: 2026-10-06T18:00:00Z
description: "How do you connect Power BI with AI agents? A technical comparison of PBIP files, the Power BI Modeling MCP Server, database connections, and real limits."
summary: "At the 3rd Business Intelligence & AI Hackathon at Universidad Americana, one question took center stage: How do you reliably control a Power BI model with AI agents? A technical comparison of PBIP, MCP, and Copilot."
tags: ["Power BI", "Artificial Intelligence", "MCP", "Business Intelligence", "Business Informatics"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-with-ai/"
---

At the 3rd Business Intelligence & AI Hackathon at Universidad Americana, one question took center stage: How do you reliably control a Power BI model with AI agents?

![Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana](hackathon_group.jpg)
*Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana (October 3, 2026).*

Three distinct paths exist in practice:
* **File Level (PBIP):** Direct editing of TMDL text files on disk with coding agents. Works immediately without any setup and without opening Power BI Desktop.
* **Live Session (MCP):** Direct connection to the local Analysis Services engine in Power BI Desktop for interactive modeling with real-time error checking and DAX test queries.
* **Cloud (Microsoft Copilot):** Natively integrated within Microsoft Fabric for automated report page layout and canvas visuals inside the Microsoft ecosystem.

---

## Architecture Overview: How Data and AI Connect

Before examining AI tools, the data connection layer needs clarity.

Power BI connects directly to your data sources (SQL Server, PostgreSQL, ERP systems, or cloud data warehouses) through Power Query, DirectQuery, or scheduled imports. The AI agent does not require direct access to your production database. Instead, the agent operates entirely on the semantic model layer (tables, relationships, and DAX calculations).

![Architecture Overview: File-Based Modeling vs. Live Session](architecture_diagram.png)
*Architecture overview: File-based modeling via PBIP (Path 1) versus live session modeling via MCP (Path 2) and deployment to Power BI Service.*

This separation provides a crucial governance advantage: existing enterprise access controls, firewalls, and data source permissions remain enforced by Power BI. The AI only shapes how data is modeled and aggregated.

---

## Path 1: File-Based Modeling via PBIP (TMDL)

The standard `.pbix` format is a compressed binary archive. An AI model cannot inspect or modify binary blobs. Saving a project as `.pbip` (Power BI Project) splits the report into human-readable plain text:

* **Semantic Model:** Tables, relationships, and DAX measures are stored in TMDL (Tabular Model Definition Language) files.
* **Report Definition:** Visuals, layouts, and filters are stored in JSON format.

A coding agent (such as Claude Code or any terminal-based agent) can open this directory, read the TMDL files, understand the schema, and write new measures directly into the source code. This works immediately without extra middleware or open software.

### Practical Limitations of Path 1

While file-based editing integrates directly with Git workflows, it comes with clear constraints:

* **No live syntax validation:** The agent writes DAX formulas into text files blindly. Syntax errors are only caught when you open or reload the project in Power BI Desktop.
* **No test execution:** The agent cannot run `execute_dax` queries to verify calculation results against actual data.
* **Manual reload required:** Changes made to TMDL files on disk require reloading or reopening Power BI Desktop to reflect in the UI.

---

## Path 2: Live Session Control via MCP

When you need interactive modeling with instant validation, the Model Context Protocol (MCP) bridges the gap.

The Visual Studio Code Extension **Power BI Modeling MCP Server** bundles a standalone executable (`powerbi-modeling-mcp.exe`) built on Microsoft Analysis Services client libraries.

![Power BI Modeling MCP Server in VS Code Extensions](vscode_mcp_extension.jpg)
*The Power BI Modeling MCP Server extension in the Visual Studio Code Marketplace.*

### Setup Details

1. **Locate the executable:** The extension installs `powerbi-modeling-mcp.exe` locally within the VS Code extension folder.
2. **Configure the AI client:** In `claude_desktop_config.json` (or any MCP-compliant client), register the server with the `--start` flag:

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

3. **Connect to the session:** Open Power BI Desktop with your data model, whether saved as `.pbix` or `.pbip`. Prompt the AI: *"Connect to my active Power BI session."*

![MCP Server Active in Claude Desktop](claude_mcp_running.jpg)
*The Power BI Modeling MCP Server actively connected in Claude Desktop.*

Because Power BI Desktop starts a local Analysis Services instance in the background for every open model, the MCP server connects directly to that local port. The AI gains structured tools: inspect schema, create measures, and run DAX queries directly. If the engine returns a syntax error, the AI receives that feedback in the same turn and self-corrects the code.

---

## Path 3: Microsoft Copilot for Power BI

Microsoft also offers built-in AI capabilities directly inside the Power BI service and desktop through Copilot.

* **Focus on Canvas and Layout:** Copilot creates report pages and standard visuals directly on the canvas in the Microsoft ecosystem, but is rarely suited for deep semantic modeling.
* **Cloud-Bound:** Processing always takes place in the Microsoft Cloud, requiring an active connection and assigned Fabric capacity (F64 SKU minimum) in the tenant.
* **Platform Costs and Vendor Lock-in:** High dedicated capacity fees and a closed architecture without custom system prompts or external developer tool access.

---

## Can All Three Approaches Do Everything? Direct Capability Comparison

No single tool covers the entire workflow. Each approach has distinct strengths and clear limitations:

| Capability / Requirement | Path 1: PBIP (File Level) | Path 2: MCP Server (Live) | Path 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Supported File Formats** | Only PBIP (file system) | Both PBIX and PBIP | Both PBIX and PBIP |
| **DAX Measure Creation** | Yes (batch plain text in TMDL) | Yes (direct injection into engine) | Yes (chat prompt) |
| **Live Syntax Validation** | No (blind text edits) | Yes (instant engine feedback) | Partial (heuristics only) |
| **DAX Query Testing (`execute_dax`)** | No (no running engine) | Yes (direct engine query) | No |
| **Git Version Control & CI/CD** | Excellent (native text diffs) | Available with PBIP after saving | None (tied to Microsoft cloud) |
| **Hardware Requirements** | Minimal (CLI or code editor) | High (Desktop app and RAM) | None (cloud hosting) |
| **Data Privacy** | Controlled by chosen LLM | Controlled by chosen LLM | Stored in Microsoft Cloud |
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

1. **PBIP is the mandatory foundation:** Versioning data models in Git requires moving away from binary PBIX files. Version control is a property of the file format, not of the AI tool.
2. **MCP decisively wins active modeling:** Plain filesystem agents lack syntax validation, while Copilot remains an expensive convenience tool for generic canvas visuals. For precise semantic modeling, complex DAX logic, and dynamic SVG cards, live engine connection via MCP is the most productive approach by far.

---

Special thanks to **Matías Ciancio** for the architecture visualization and technical exchange.
