---
title: "Controlar Power BI con IA: Cómo funciona la arquitectura con MCP y PBIP en la práctica"
date: 2026-10-06T18:00:00Z
description: "¿Cómo conectar Claude con Power BI? Sin rodeos teóricos, sino la arquitectura real del hackathon: archivos PBIP vs. Power BI Modeling MCP Server."
summary: "En el hackathon, mentores y participantes preguntaban: ¿Cómo conectaron Power BI en vivo con Claude? Aquí está la arquitectura real: archivos PBIP, el servidor MCP de Microsoft y por qué el trabajo previo de datos sigue siendo clave."
tags: ["Power BI", "Inteligencia Artificial", "MCP", "Business Intelligence", "Informática Empresarial"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: true
ShowBreadCrumbs: true
---

En el hackathon de Asunción, tras nuestra presentación surgió una y otra vez la misma pregunta, tanto de participantes como de mentores: *¿Cómo hicieron la conexión entre Power BI y Claude? ¿Fue con MCP o cómo se pide eso?*

La respuesta es simple cuando se entiende la arquitectura. No hay magia, sino exactamente dos caminos prácticos: el acceso directo a archivos mediante el formato PBIP y la conexión en vivo mediante el Model Context Protocol (MCP).

Aquí comparto cómo funciona este entorno en la realidad, dónde destacan estas herramientas y cuáles son sus límites reales.

---

## Camino 1: Vía sistema de archivos (Claude Code sobre PBIP)

El método más sencillo no requiere configurar MCP. El único requisito es el formato moderno de Microsoft: `.pbip` (Power BI Project).

Los archivos tradicionales `.pbix` son paquetes binarios comprimidos. Una IA no puede leerlos directamente. Al guardar el informe como `.pbip`, Power BI desglosa el proyecto en texto plano legible:

1. **El modelo semántico:** Las tablas, relaciones y medidas DAX se guardan como TMDL (Tabular Model Definition Language) en archivos de texto independientes.
2. **La definición del informe:** Los gráficos, filtros y diseños visuales se guardan como archivos JSON.

Con este proyecto guardado en disco, no hace falta ningún protocolo especial. Abres una terminal con Claude Code, le indicas la carpeta del proyecto y le pides analizar el modelo. La IA lee los archivos TMDL, comprende la estructura y escribe nuevas medidas DAX directamente en el código fuente. Git registra cada cambio con diffs limpios.

---

## Camino 2: Conexión en vivo (Claude Desktop + Power BI Modeling MCP)

Si buscas trabajar de forma interactiva y manipular un modelo abierto en Power BI Desktop en tiempo real, entra en juego el Model Context Protocol (MCP).

Microsoft ofrece un puente que muchos aún desconocen: la extensión de Visual Studio Code **Power BI Modeling MCP Server**.

### La configuración técnica

1. **Ejecutable del servidor:** Con la extensión de VS Code, Microsoft incluye un ejecutable independiente (`powerbi-modeling-mcp.exe`).
2. **Configuración en Claude Desktop:** En el archivo `claude_desktop_config.json`, se registra este servidor bajo la clave `mcpServers`. Se indica la ruta al ejecutable y el argumento `--start`.
3. **Sesión activa:** Al abrir Power BI Desktop con un modelo, en segundo plano se ejecuta una instancia local de Analysis Services.
4. **Instrucción a Claude:** Un prompt simple como *"Conéctate a mi sesión activa de Power BI"* es suficiente. El servidor MCP se enlaza al puerto local.

A partir de ahí, Claude dispone de herramientas operativas: consultar el modelo, inspeccionar tablas, crear medidas DAX y ejecutar consultas de prueba directamente contra el motor. Si una fórmula DAX tiene un error de sintaxis, el motor responde de inmediato y la IA corrige el código de forma autónoma.

---

## Dónde destaca la herramienta: SVGs a medida y scaffolding rápido

El verdadero valor no está en crear gráficos de barras estándar, los cuales se arman en segundos manualmente.

El potencial real surge en requerimientos complejos y personalizados:
* **Tarjetas KPI dinámicas con SVG:** Al combinar el visual HTML de Power BI con medidas DAX que generan código SVG dinámico, es posible crear tarjetas KPI e indicadores de progreso personalizados. Escribir eso a mano toma horas. Una IA genera la medida DAX con SVG en cuestión de segundos.
* **Scaffolding de medidas:** Con un modelo semántico definido, el servidor MCP puede estructurar decenas de medidas estándar (Time Intelligence, variaciones YoY, márgenes) en una sola interacción.

---

## La realidad operativa: Consumo de tokens e higiene de datos

En la práctica existen dos factores clave que no se deben pasar por alto:

1. **Alto consumo de tokens:** Para generar cálculos correctos, la IA debe cargar en su contexto el modelo semántico, los esquemas de tablas y las reglas de negocio. Esto consume una gran cantidad de tokens por interacción.
2. **El trabajo previo sigue siendo humano:** La mayor parte del tiempo en un proyecto de BI se invierte en limpieza de datos, validación y comprensión de los procesos del negocio. Si el modelo de datos tiene errores de diseño, ninguna IA podrá compensarlo.

La herramienta automatiza la escritura de sintaxis y acelera el desarrollo, pero no reemplaza el criterio analítico.

---

## Conclusión: Business Intelligence as Code

La combinación de PBIP y MCP transforma la forma de trabajar. Power BI pasa de ser una herramienta visual cerrada a un sistema basado en código fuente, perfectamente integrable en flujos de desarrollo modernos y automatizaciones con IA.

Para la informática empresarial, este es el camino adecuado: automatizar tareas repetitivas, mantener trazabilidad total mediante control de versiones y conservar el control sobre la lógica del negocio.
