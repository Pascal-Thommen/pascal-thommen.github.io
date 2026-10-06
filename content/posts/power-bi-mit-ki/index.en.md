---
title: "Controlling Power BI with AI: How the Setup Works in Practice via MCP and PBIP"
date: 2026-10-06T18:00:00Z
description: "How do you connect Claude with Power BI? No theoretical fluff, but the real setup from the hackathon: PBIP file system vs. Power BI Modeling MCP Server."
summary: "At the hackathon, mentors and participants kept asking: How did you connect Power BI to Claude in real time? Here is the actual architecture: PBIP files, Microsoft's MCP server, and why upfront data preparation still matters."
tags: ["Power BI", "Artificial Intelligence", "MCP", "Business Intelligence", "Business Informatics"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: true
ShowBreadCrumbs: true
---

At the hackathon in Asuncion, the same question came up repeatedly after our presentation, from both participants and mentors: *How did you actually connect Power BI to Claude? Was it through MCP, or how did you establish that link?*

The answer is straightforward once you understand the architecture. There is no magic involved, just two practical approaches: direct file access via the PBIP format, and a live session link through the Model Context Protocol (MCP).

Here is how the setup actually works, where the tools excel, and where their limits lie.

---

## Approach 1: The File System Route (Claude Code on PBIP)

The simplest approach does not require MCP at all. The only prerequisite is Microsoft's modern project format: `.pbip` (Power BI Project).

Traditional `.pbix` files are binary archives. An AI cannot process them. When you save your report as `.pbip`, however, Power BI splits the entire project into human-readable plain text files:

1. **The semantic model:** Tables, relationships, and DAX measures are stored as TMDL (Tabular Model Definition Language) in dedicated text files.
2. **The report definition:** Visuals, filters, and layouts are stored as JSON files.

Once this project lives on your disk, you do not need any special protocol. Open a terminal with Claude Code, point it to the project folder, and let it inspect the model. The AI reads the TMDL files, understands the schema, and writes new DAX measures directly into the source code. Git tracks every change with clean diffs.

---

## Approach 2: The Live Session Route (Claude Desktop + Power BI Modeling MCP)

If you want to work interactively and touch an open Power BI Desktop instance in real time, the Model Context Protocol (MCP) comes into play.

Microsoft provides a bridge that many developers have not noticed yet: the Visual Studio Code extension **Power BI Modeling MCP Server**.

### The Setup Under the Hood

1. **Server executable:** The VS Code extension includes a standalone executable (`powerbi-modeling-mcp.exe`).
2. **Claude Desktop configuration:** In `claude_desktop_config.json`, register this server under `mcpServers`. Point the command to the executable path and pass the `--start` argument.
3. **Active session:** Whenever Power BI Desktop runs with a model open, a local Analysis Services instance is running in the background.
4. **Command to Claude:** A prompt like *"Connect to my active Power BI session"* is all it takes. The MCP server binds to the local port.

From this point on, Claude has tools: query the model, inspect tables, create DAX measures, and execute queries directly against the engine. Any DAX syntax error gets returned immediately, allowing the AI to fix its code autonomously.

---

## Where the Setup Shines: Custom SVGs and Rapid Scaffolding

The real leverage is not creating standard bar charts. You can build those manually in seconds.

The combination becomes powerful for complex, custom requirements:
* **Dynamic SVG cards:** Combining Power BI's HTML visual with DAX measures that generate dynamic SVG code allows you to build custom KPI cards and progress indicators. Writing those by hand takes hours. An AI generates the DAX SVG code in seconds.
* **Scaffolding:** Once your semantic model is clear, the MCP server can scaffold dozens of standard measures (Time Intelligence, YoY, margins) in a single run.

---

## The Reality Check: Token Consumption and Data Hygiene

In practical use, two factors must be kept in mind:

1. **Heavy token usage:** For the AI to generate meaningful calculations, it must process the semantic model, table schemas, and business context. That consumes a massive amount of context tokens. Free tier limits are reached quickly.
2. **Upfront work remains human work:** Most of the time in any BI project is spent cleaning data, validating numbers, and understanding business processes. If the data model is flawed, even the best MCP server cannot fix it.

The tool eliminates repetitive typing and syntax friction, but it does not replace domain knowledge.

---

## Conclusion: Business Intelligence as Code

Combining PBIP and MCP changes the workflow fundamentally. Power BI transforms from a closed visual desktop app into a code-based system that integrates smoothly into modern developer workflows and AI pipelines.

For business informatics, this is the right direction: automating routine work, maintaining full version control in code, and keeping humans in charge of core business logic.
