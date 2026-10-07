---
title: "How to Use Power BI with AI: PBIP, MCP, and Practical Architecture"
date: 2026-10-06T18:00:00Z
description: "How do you connect Power BI with AI agents? A hands-on look at PBIP files, the Power BI Modeling MCP Server, database connections, and limitations."
summary: "One of the key questions at the hackathon was: How do you connect Power BI with AI? Here is the actual architecture: PBIP files, the MCP server, and how data layers interact."
tags: ["Power BI", "Artificial Intelligence", "MCP", "Business Intelligence", "Business Informatics"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: true
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-mit-ki/"
  - "/en/posts/power-bi-mit-ki/"
---

One of the key questions at the hackathon was: How do you connect Power BI with AI? More specifically: How do you use AI agents to model and automate Power BI?

![Participants and mentors at the 3rd BI Hackathon by Comunidad Data Platform Paraguay](hackathon_group.jpg)
*Participants and mentors at the 3rd BI Hackathon by Comunidad Data Platform Paraguay (Asuncion, October 3, 2026).*

In practice, there are three primary paths to bring AI into Power BI:
1. File-based editing via the PBIP format (offline, Git-integrated).
2. Live session control via the Model Context Protocol (MCP).
3. Native cloud features via Microsoft Copilot.

None of these three options can do everything, and they are not equally suited for every task. Understanding where data connections sit, what each approach can actually deliver, and where its limits lie determines whether AI automation succeeds in practice.

---

## Architecture Overview: How Data and AI Connect

Before examining AI tools, the data connection layer needs clarity.

Power BI connects directly to your data sources (SQL Server, PostgreSQL, ERP systems, or cloud data warehouses) through Power Query, DirectQuery, or scheduled imports. The AI agent does not require direct access to your production database. Instead, the agent operates entirely on the semantic model layer (tables, relationships, and DAX calculations).

```mermaid
flowchart TD
    DB[(Production Database)] -->|Power Query / DirectQuery| PBI[Power BI Desktop / Engine]
    
    subgraph Path 1: File System
        PBIP[PBIP Project Folder] -->|TMDL & JSON Files| Agent1[Coding Agent / Claude Code]
        Agent1 -->|Git Commit| Repo[Git Repository]
    end
    
    subgraph Path 2: Live Session
        PBI <-->|Local Analysis Services Port| MCP[Power BI Modeling MCP Server]
        MCP <-->|Tools: execute_dax, create_measure| Agent2[AI Assistant / Claude Desktop]
    end
    
    PBI -.->|Save / Export| PBIP
```

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

![Configuration in claude_desktop_config.json](claude_desktop_config_json.jpg)
*Configuration snippet in claude_desktop_config.json pointing to the MCP executable.*

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

---

## Path 3: Microsoft Copilot for Power BI

Microsoft also offers built-in AI capabilities directly inside the Power BI service and desktop through Copilot.

* **Cost and Licensing:** Microsoft Copilot requires paid Microsoft Fabric capacity (minimum F64 SKU) or Premium capacity. This creates significant recurring cost barriers for individual developers or mid-sized teams.
* **Closed Ecosystem:** Copilot is a closed cloud feature. It does not allow custom prompt engineering, agentic chaining, or external tool execution.
* **Primary Focus:** Copilot is designed primarily for non-technical users to generate summary descriptions and basic visual layouts rather than deep data model engineering.

---

## Can All Three Approaches Do Everything? Direct Capability Comparison

No single tool covers the entire workflow. Each approach has distinct strengths and clear limitations:

| Capability / Requirement | Path 1: PBIP Files (TMDL) | Path 2: MCP Server (Live) | Path 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **DAX Measure Creation** | Yes (batch plain text) | Yes (direct injection) | Yes (chat prompt) |
| **Live Syntax Validation** | No (blind text edits) | Yes (instant engine feedback) | Partial (heuristics only) |
| **DAX Query Testing (`execute_dax`)** | No (no running engine) | Yes (direct engine query) | No |
| **Git Version Control & CI/CD** | Excellent (native text diffs) | Manual (must save model first) | None (cloud-locked) |
| **Visual Layout & Chart Generation** | Limited (blind JSON edits) | No (modeling focus only) | Yes (creates canvas visuals) |
| **Custom SVG Cards & HTML Visuals** | Limited (blind DAX string) | Excellent (live visual preview) | Poor (standard visuals only) |
| **Cost & Model Flexibility** | Free (any LLM or local model) | Free (any MCP client) | Very expensive (Fabric F64) |
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
