---
title: "Power BI con IA: PBIP y MCP en la práctica"
date: 2026-10-06T18:00:00Z
description: "¿Cómo conectar Power BI con agentes de IA? Un análisis técnico de archivos PBIP, Power BI Modeling MCP Server, conexión a bases de datos y límites reales."
summary: "En la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana, el debate central fue concreto: ¿Cómo controlar un modelo de Power BI de forma confiable mediante agentes de IA? Comparativa técnica de PBIP, MCP y Copilot."
tags: ["Power BI", "Inteligencia Artificial", "MCP", "Business Intelligence", "Informática Empresarial"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-con-ia/"
  - "/es/posts/power-bi-con-ia/"
---

En la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana, el debate central fue concreto: ¿Cómo controlar un modelo de Power BI de forma confiable mediante agentes de IA?

![Participantes y mentores en la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana](hackathon_group.jpg)
*Participantes y mentores en la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana (3 de octubre de 2026).*

En la práctica existen tres opciones principales:
* **Nivel de archivos (PBIP):** Edición directa de archivos de texto TMDL en el sistema de archivos con agentes de código. Funciona de inmediato sin necesidad de abrir Power BI Desktop y sin ninguna configuración previa.
* **Sesión en vivo (MCP):** Conexión directa al motor local de Analysis Services en Power BI Desktop para modelado interactivo con verificación de errores en tiempo real y consultas DAX de prueba (funciona tanto con PBIX como con PBIP).
* **Nube (Microsoft Copilot):** Integración nativa en Microsoft Fabric para la creación automatizada de páginas de informe y elementos visuales directamente en el lienzo dentro del ecosistema Microsoft.

---

## Panorama de arquitectura: Cómo interactúan los datos y la IA

Antes de hablar de IA, es indispensable entender la capa de datos.

Power BI se encarga de la conexión directa a los orígenes de datos (SQL Server, PostgreSQL, sistemas ERP o data warehouses) mediante Power Query, DirectQuery o importación programada. El agente de IA **no necesita acceso directo a la base de datos de producción** ni requiere credenciales de la misma. En su lugar, la IA opera exclusivamente sobre la capa del modelo semántico (tablas, relaciones y medidas DAX).

![Panorama de arquitectura: Modelado basado en archivos versus sesión en vivo](architecture_diagram.png)
*Resumen de arquitectura: Modelado basado en archivos vía PBIP (Camino 1) versus sesión en vivo vía MCP (Camino 2) y despliegue a Power BI Service.*

Esta separación aporta una ventaja fundamental de gobernanza: las políticas de seguridad, permisos de usuario y firewalls permanecen dentro de Power BI. La IA solo interactúa con la lógica de cálculo y la estructura visual.

---

## Camino 1: Modelado basado en archivos con PBIP (TMDL)

Los archivos clásicos `.pbix` son paquetes binarios comprimidos que resultan ilegibles para los modelos de IA. Al guardar el informe como `.pbip` (Power BI Project), Power BI descompone el modelo en texto plano:

* **Modelo semántico:** Las tablas, relaciones y medidas DAX se guardan como archivos TMDL (Tabular Model Definition Language).
* **Definición de informes:** Los gráficos, formatos y filtros se guardan en archivos JSON estructurados.

La ventaja decisiva del Camino 1 es la configuración cero: Los agentes de código (como Claude Code, Cursor o scripts CLI) pueden trabajar de inmediato en cualquier sistema operativo, incluso en servidores Linux o entornos CI sin interfaz gráfica. Analizan los archivos TMDL directamente en la carpeta del proyecto y escriben nuevas medidas en el código sin necesidad de tener Power BI Desktop instalado o abierto.

### Limitaciones reales del Camino 1

Sin un motor en ejecución en segundo plano, este enfoque basado exclusivamente en archivos presenta limitaciones evidentes:

* **Sin validación de sintaxis en tiempo real:** El agente escribe fórmulas DAX directamente en los archivos de texto. Si hay un error, solo se detecta al abrir o recargar el proyecto en Power BI Desktop.
* **Sin ejecución de consultas de prueba:** El agente no puede ejecutar consultas de prueba (`execute_dax`) contra el motor para comprobar si los números calculados son correctos.
* **Recarga manual:** Las modificaciones realizadas en el disco no se reflejan automáticamente en la ventana abierta de Power BI Desktop sin reiniciar o recargar.

---

## Camino 2: Control en sesión viva mediante MCP

Cuando se requiere modelado interactivo con validación inmediata, el Model Context Protocol (MCP) proporciona el enlace necesario.

La extensión para Visual Studio Code **Power BI Modeling MCP Server** empaqueta un ejecutable independiente (`powerbi-modeling-mcp.exe`) basado en las librerías de Microsoft Analysis Services.

![Power BI Modeling MCP Server en las extensiones de VS Code](vscode_mcp_extension.jpg)
*La extensión Power BI Modeling MCP Server en Visual Studio Code Marketplace.*

### Pasos de configuración

1. **Ubicar el ejecutable:** La extensión instala `powerbi-modeling-mcp.exe` localmente en la carpeta de extensiones de VS Code.
2. **Configurar el cliente de IA:** En `claude_desktop_config.json` (o cualquier cliente compatible con MCP), registrar el servidor con el parámetro `--start`:

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

3. **Conectar a la sesión activa:** Abrir Power BI Desktop con el modelo de datos. Esto funciona tanto con archivos tradicionales `.pbix` como con proyectos modernos `.pbip`. Enviar una instrucción a la IA: *"Conéctate a mi sesión activa de Power BI."*

![Servidor MCP activo en Claude Desktop](claude_mcp_running.jpg)
*El servidor Power BI Modeling MCP conectado activamente en Claude Desktop.*

Dado que Power BI Desktop levanta una instancia local de Analysis Services en segundo plano para cada modelo abierto, el servidor MCP se conecta directamente a ese puerto local. La IA obtiene herramientas concretas: consultar esquemas, generar medidas y ejecutar consultas DAX de prueba. Si el motor devuelve un error de sintaxis, la IA recibe la notificación en el mismo paso y corrige el código de inmediato.

Cuando se edita un proyecto PBIP mediante este método y luego se guarda en Power BI Desktop, todas las medidas creadas por la IA se guardan directamente en los archivos TMDL en el disco. De esta manera, se combina la validación en tiempo real del motor con el control total de versiones en Git de PBIP.

---

## Camino 3: Microsoft Copilot para Power BI

Microsoft ofrece también funciones integradas de IA directamente en el servicio en la nube y en Power BI Desktop mediante Copilot.

* **Ligado a la nube a pesar de Desktop:** Incluso al utilizar Copilot como panel lateral en Power BI Desktop tanto para `.pbix` como para `.pbip`, el procesamiento nunca se ejecuta localmente. Cada instrucción se envía a la nube de Microsoft. Sin una conexión activa y sin una capacidad asignada de Fabric (mínimo SKU F64) en el tenant, la función permanece inactiva.
* **Verdadera ventaja: Ecosistema y generación en el lienzo:** El valor genuino de Copilot no radica en el modelado semántico profundo, sino en la integración transparente con Microsoft Fabric y en su capacidad para generar páginas de informe completas y elementos visuales directamente en el lienzo. Esto no lo pueden hacer ni el Camino 1 ni el Camino 2.
* **Costos de plataforma y dependencia del proveedor:** Las capacidades dedicadas de Fabric representan una barrera financiera considerable. Además, Copilot sigue siendo una plataforma cerrada sin acceso a prompts de sistema personalizados ni herramientas de desarrollo externas.

---

## ¿Pueden las tres opciones hacer todo por igual? Comparativa directa de capacidades

Ninguna herramienta cubre la totalidad del flujo. Cada alternativa presenta ventajas y limitaciones evidentes:

| Requisito / Capacidad | Camino 1: Sistema de archivos Headless | Camino 2: Servidor MCP (En vivo) | Camino 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Formatos compatibles** | Estrictamente PBIP (TMDL texto plano) | Tanto PBIX como PBIP | Tanto PBIX como PBIP |
| **Carga de configuración** | Cero (utilizable en cualquier editor) | Media (extensión VS Code y config) | Baja si ya cuenta con licencia |
| **Creación de medidas DAX** | Sí (por lotes en TMDL) | Sí (inyección directa al motor) | Sí (prompt en chat) |
| **Validación de sintaxis en vivo** | No (edición ciega de texto) | Sí (respuesta inmediata del motor) | Parcial (solo heurística) |
| **Consultas de prueba (`execute_dax`)** | No (sin motor en ejecución) | Sí (consulta directa al motor) | No |
| **Control de versiones en Git y CI/CD** | Nativo mediante archivos PBIP | Disponible con PBIP tras guardar | Sin integración directa con Git |
| **Requisitos de hardware** | Mínimos (CLI o editor de código) | Altos (Power BI Desktop y RAM) | Ninguno (hospedaje en nube) |
| **Privacidad de datos y ubicación de inferencia** | Controlada por el LLM elegido | Controlada por el LLM elegido | Siempre en la nube de Microsoft |
| **Soporte para DirectQuery** | Limitado (solo metadatos) | Complejo (latencia de consulta) | Sí (soporte nativo en la nube) |
| **Diseño visual en el lienzo** | Limitado (edición ciega de JSON) | No (enfoque exclusivo en modelado) | Sí (crea visuales en el lienzo) |
| **Tarjetas SVG a medida y HTML** | Limitado (cadena DAX a ciegas) | Excelente (vista previa en vivo) | Inadecuado (solo visuales estándar) |
| **Costo y flexibilidad de modelos** | Gratuito (cualquier LLM o local) | Gratuito (cualquier cliente MCP) | Alto (licencia de usuario o Fabric F64) |
| **Requisito de ejecución** | Solo editor de código / CLI | Power BI Desktop abierto localmente | Capacidad Fabric activa en la nube |

### Guía de decisión clara para la práctica

* **Elegir PBIP como formato de archivo** en cuanto se requiera control de versiones en Git, trabajo colaborativo y flujos de CI/CD. Esta es una decisión de formato, no de herramienta.
* **Elegir Camino 1 (Nivel de archivos)** cuando se deseen generar medidas por lotes sin ninguna configuración, sin abrir Power BI Desktop o mediante scripts automatizados.
* **Elegir Camino 2 (Sesión viva con MCP)** cuando se modele interactivamente en el puesto de trabajo y se necesite respuesta inmediata del motor, corrección de errores de sintaxis y consultas DAX de prueba.
* **Elegir Camino 3 (Microsoft Copilot)** cuando se requiera generar automáticamente el diseño de páginas de informe directamente en el lienzo dentro del ecosistema regulado de Microsoft y se cuente con capacidad Fabric asignada.

---

## Utilidad práctica y realidad operativa

### Dónde destacó la solución del hackathon

* **Medidas dinámicas con SVG:** Combinar el visual HTML de Power BI con medidas DAX que generan código SVG dinámico permite crear tarjetas KPI e indicadores personalizados. Programar código SVG en DAX a mano toma horas; la IA genera la medida en segundos.
* **Scaffolding de medidas:** Con un modelo semántico definido, el agente genera decenas de medidas estándar (comparativas año contra año, márgenes, promedios móviles) en una sola operación.

### Realidad operativa

* **Alto consumo de tokens:** Transferir el modelo semántico, las tablas y el contexto de negocio consume una gran cantidad de tokens de contexto. Los límites de uso en planes gratuitos se agotan con rapidez.
* **La limpieza de datos sigue siendo trabajo humano:** La IA no puede corregir un modelo de datos deficiente. Si la calidad de los datos de origen no está asegurada, la IA solo producirá cálculos incorrectos a mayor velocidad.

---

## Conclusión

Para la práctica quedan dos conclusiones esenciales:

1. **PBIP es la base obligatoria:** Controlar versiones en Git e integrar flujos de CI/CD exige abandonar el formato binario PBIX. El control de versiones es una propiedad del formato de archivo, no de la herramienta de IA utilizada.
2. **MCP domina el desarrollo activo:** Los agentes sobre el sistema de archivos fallan por falta de validación de sintaxis, mientras que Copilot sigue siendo una costosa herramienta de conveniencia para visuales genéricos. Para el modelado semántico exigente, lógica DAX compleja y tarjetas SVG dinámicas, la conexión en vivo mediante MCP es con diferencia la vía más productiva.

---

Un agradecimiento especial a **Matías Ciancio** por la visualización de la arquitectura y el intercambio técnico.
