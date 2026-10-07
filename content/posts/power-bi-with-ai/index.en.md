---
title: "Power BI with AI: PBIP and MCP in Practice"
date: 2026-10-06T18:00:00Z
description: "How do you connect Power BI with AI agents? A hands-on look at PBIP files, the Power BI Modeling MCP Server, database connections, and limitations."
summary: "At the 3rd Business Intelligence & AI Hackathon, one question took center stage: How do you reliably control a Power BI model with AI agents without operating blindly? A breakdown of PBIP, MCP, and Copilot."
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

At the 3rd Business Intelligence & AI Hackathon at Universidad Americana, one question took center stage: How do you reliably control a Power BI model with AI agents without operating blindly?

![Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana](hackathon_group.jpg)
*Participants and mentors at the 3rd Business Intelligence & AI Hackathon at Universidad Americana (October 3, 2026).*

Three distinct paths exist in practice:
* **File Level (PBIP):** Editing Git-integrated TMDL files offline with coding agents.
* **Live Session (MCP):** Driving the active Power BI Desktop instance through Microsoft's Analysis Services MCP server.
* **Cloud (Microsoft Copilot):** Using built-in AI capabilities within Microsoft Fabric.

Each path solves a different problem. None can do everything.

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

A coding agent (such as Claude Code or any terminal-based agent) can open this directory, read the TMDL files, understand the schema, and write new measures directly into the source code.

### Limitations of Path 1

While file-based editing integrates directly with Git workflows, it comes with clear constraints:

* **No live syntax validation:** The agent writes DAX formulas into text files blindly. Syntax errors are only caught when you open or reload the project in Power BI Desktop.
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

3. **Connect to the Active Session:** Open Power BI Desktop with your data model. Then prompt the AI: *"Connect to my active Power BI session."*

![MCP Server running in Claude Desktop](claude_mcp_running.jpg)
*The Power BI Modeling MCP Server connected and running inside Claude Desktop.*

Because Power BI Desktop runs a local Analysis Services instance in the background, the MCP server attaches to its local port. The AI receives functional tools: inspecting tables, creating measures, and running DAX test queries. If a formula fails, the engine returns the error message immediately, allowing the AI to self-correct in real time.

### Works with Both PBIX and PBIP

A frequent question is whether MCP requires the new PBIP format. The answer is no:
* **PBIP is only mandatory for Path 1**, where an AI agent reads and edits TMDL text files on disk without running Power BI Desktop.
* **MCP attaches to the active Analysis Services instance** that Power BI Desktop spins up locally whenever any report is open. Whether that file was saved as `.pbip` or as a legacy `.pbix` binary archive does not matter to the MCP server.

---

## Path 3: Microsoft Copilot for Power BI

Microsoft also offers built-in AI capabilities directly inside the Power BI service and desktop through Copilot.

* **Cost and Licensing:** Microsoft Copilot in Power BI requires paid capacity. While individual chat assistance in Microsoft 365 or Power BI Desktop requires per-user add-on licenses ($30 per user per month), full organizational Copilot integration in Power BI workspaces requires dedicated Microsoft Fabric capacity (minimum F64 SKU or pay-as-you-go capacity units). This creates significant recurring cost barriers and vendor lock-in compared to open LLM APIs.
* **Closed Ecosystem:** Copilot is a closed cloud feature. It does not allow custom prompt engineering, agentic chaining, or external tool execution.
* **Primary Focus:** Copilot is designed primarily for non-technical users to generate summary descriptions and basic visual layouts rather than deep data model engineering.

---

## Can All Three Approaches Do Everything? Direct Capability Comparison

No single tool covers the entire workflow. Each approach has distinct strengths and clear limitations:

| Capability / Requirement | Path 1: PBIP Files (TMDL) | Path 2: MCP Server (Live) | Path 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Compatible Formats** | Only PBIP (folder structure) | Both PBIX and PBIP | PBIX and published models |
| **DAX Measure Creation** | Yes (batch plain text) | Yes (direct injection) | Yes (chat prompt) |
| **Live Syntax Validation** | No (blind text edits) | Yes (instant engine feedback) | Partial (heuristics only) |
| **DAX Query Testing (`execute_dax`)** | No (no running engine) | Yes (direct engine query) | No |
| **Git Version Control & CI/CD** | Excellent (native text diffs) | Limited (requires desktop save) | None (cloud-locked) |
| **Hardware Requirements** | Minimal (CLI or code editor) | High (Desktop app and RAM) | Zero (cloud-hosted) |
| **Data Privacy** | Depends on chosen LLM | Depends on chosen LLM | Stored in Microsoft Cloud |
| **DirectQuery Support** | Limited (metadata only) | Complex (query latency) | Yes (native cloud support) |
| **Visual Layout & Chart Generation** | Limited (blind JSON edits) | No (modeling focus only) | Yes (creates canvas visuals) |
| **Custom SVG Cards & HTML Visuals** | Limited (blind DAX string) | Excellent (live visual preview) | Poor (standard visuals only) |
| **Cost & Model Flexibility** | Free (any LLM or local model) | Free (any MCP client) | High (Fabric F64 or user license) |
| **Runtime Requirement** | Code editor only (CLI) | Power BI Desktop open locally | Active Fabric cloud subscription |

### Summary of Strengths

* **Use PBIP** when you want version-controlled data models in Git, automated CI/CD pipelines, and bulk measure scaffolding without opening Power BI Desktop.
* **Use MCP** when you are actively modeling at your workstation and need real-time engine feedback, error self-correction, and custom SVG visual measures.
* **Use Copilot** when you have an enterprise Fabric budget and want quick, generic report pages generated for end users.

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

Connecting Power BI with AI is not about finding a magic tool that does everything: it is about selecting the right path for the job. File-based PBIP enables disciplined software engineering in Git, live MCP gives developers an interactive co-pilot with real engine validation, and corporate cloud tools serve general reporting needs. The analytical judgment remains in human hands, while the execution speed scales significantly.
