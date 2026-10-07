---
title: "Power BI con IA: PBIP y MCP en la práctica"
date: 2026-10-06T18:00:00Z
description: "¿Cómo conectar Power BI con agentes de IA? Un análisis técnico de archivos PBIP, el servidor MCP de modelado, conexiones a bases de datos y limitaciones."
summary: "En la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana, el debate central fue concreto: ¿Cómo controlar un modelo de Power BI de forma confiable mediante agentes de IA? Una comparativa técnica de PBIP, MCP y Copilot."
tags: ["Power BI", "Inteligencia Artificial", "MCP", "Business Intelligence", "Informática Empresarial"]
categories: ["Business Intelligence", "AI Engineering"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
aliases:
  - "/posts/power-bi-mit-ki/"
  - "/es/posts/power-bi-mit-ki/"
---

En la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana, el debate central fue concreto: ¿Cómo controlar un modelo de Power BI de forma confiable mediante agentes de IA?

![Participantes y mentores en la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana](hackathon_group.jpg)
*Participantes y mentores en la 3ra Edición de la Hackathon de Inteligencia de Negocios e IA en la Universidad Americana (3 de octubre de 2026).*

En la práctica existen tres opciones principales:
* **Agentes en sistema de archivos (Nivel de archivos):** Edición directa de archivos de texto TMDL (PBIP) con agentes de código. Funciona de inmediato sin abrir Power BI Desktop y sin herramientas intermedias.
* **Control de sesión en vivo (MCP):** Conexión directa al motor local mediante el servidor MCP de Analysis Services para modelado interactivo con verificación de errores en tiempo real (funciona con PBIX y PBIP).
* **Asistentes integrados en la plataforma (Microsoft Copilot):** Asistentes en la nube dentro del ecosistema Microsoft para generar páginas de informe y elementos visuales en el lienzo.

Para el modelado semántico activo, el Camino 2 (MCP) es con diferencia la opción más sólida, ya que el motor en ejecución valida la sintaxis DAX y los resultados de consulta al instante. El Camino 1 aporta total independencia sin necesidad de instalación previa, mientras que el Camino 3 destaca por el diseño automático de páginas de informe dentro del ecosistema corporativo de Microsoft.

---

## Panorama de arquitectura: Cómo interactúan los datos y la IA

Antes de hablar de IA, es indispensable entender la capa de datos.

Power BI se encarga de la conexión directa a los orígenes de datos (SQL Server, PostgreSQL, sistemas ERP o data warehouses) mediante Power Query, DirectQuery o importación programada. El agente de IA **no necesita acceso directo a la base de datos de producción** ni requiere credenciales de la misma. En su lugar, la IA opera exclusivamente sobre la capa del modelo semántico (tablas, relaciones y medidas DAX).

![Panorama de arquitectura: Modelado basado en archivos versus sesión en vivo](architecture_diagram.png)
*Resumen de arquitectura: Modelado basado en archivos vía PBIP (Camino 1) versus sesión en vivo vía MCP (Camino 2) y despliegue a Power BI Service.*

Esta separación aporta una ventaja fundamental de gobernanza: las políticas de seguridad, permisos de usuario y firewalls permanecen dentro de Power BI. La IA solo interactúa con la lógica de cálculo y la estructura visual.

---

## Camino 1: Modelado directo en el sistema de archivos (PBIP / TMDL)

Los archivos clásicos `.pbix` son paquetes binarios comprimidos que resultan ilegibles para los modelos de IA. Al guardar el informe como `.pbip` (Power BI Project), Power BI descompone el modelo en texto plano:

* **Modelo semántico:** Las tablas, relaciones y medidas DAX se guardan como archivos TMDL (Tabular Model Definition Language).
* **Definición de informes:** Los gráficos, formatos y filtros se guardan en archivos JSON estructurados.

La ventaja decisiva del Camino 1 es la ausencia total de configuración: agentes de código (como Claude Code, Cursor o scripts CLI) pueden trabajar de inmediato en cualquier sistema operativo, incluso en entornos Linux sin interfaz gráfica. Analizan archivos TMDL y agregan medidas sin requerir que Power BI Desktop esté instalado ni abierto.

### Limitaciones operativas del Camino 1

Trabajar exclusivamente sobre el disco sin un motor activo presenta desventajas evidentes:

* **Generación a ciegas:** El agente escribe fórmulas DAX sin validación de sintaxis. Los errores solo se detectan al abrir el proyecto en Power BI Desktop.
* **Sin consultas de prueba:** El agente no puede ejecutar consultas `execute_dax` para contrastar cálculos con datos reales.
* **Recarga manual:** Las modificaciones realizadas en los archivos no se reflejan automáticamente en una sesión abierta de Power BI Desktop.

---

## Camino 2: Control en sesión viva mediante MCP

Cuando se requiere modelado interactivo con validación inmediata, el Model Context Protocol (MCP) proporciona el enlace necesario.

La extensión de Visual Studio Code **Power BI Modeling MCP Server** empaqueta un ejecutable independiente (`powerbi-modeling-mcp.exe`) basado en las librerías de Microsoft Analysis Services.

![Extensión Power BI Modeling MCP Server en VS Code](vscode_mcp_extension.jpg)
*La extensión Power BI Modeling MCP Server en el Marketplace de Visual Studio Code.*

### Configuración del entorno

1. **Localizar el ejecutable:** La extensión instala `powerbi-modeling-mcp.exe` en la carpeta local de extensiones de VS Code.
2. **Configurar el cliente de IA:** En el archivo `claude_desktop_config.json` (o cualquier cliente compatible con MCP), se registra el servidor con el parámetro `--start`:

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

3. **Conexión a la sesión:** Abrir Power BI Desktop con el modelo de datos, guardado como `.pbix` tradicional o como `.pbip`. Basta con indicar a la IA: *"Conéctate a mi sesión activa de Power BI."*

![Servidor MCP activo en Claude Desktop](claude_mcp_running.jpg)
*El servidor MCP conectado y en ejecución dentro de Claude Desktop.*

Dado que Power BI Desktop levanta una instancia local de Analysis Services en segundo plano para cualquier informe abierto, el servidor MCP se conecta directamente a ese puerto local. La IA obtiene herramientas funcionales: consultar el esquema, crear medidas y ejecutar consultas DAX de prueba contra el motor. Si una fórmula tiene errores de sintaxis, el motor responde de inmediato y la IA corrige el código de manera autónoma.

---

## Camino 3: Microsoft Copilot para Power BI

Microsoft ofrece Copilot como asistente integrado tanto en Power BI Desktop como en Power BI Service.

* **Dependiente de la nube incluso en Desktop:** Aunque Copilot está disponible en el panel lateral de Power BI Desktop tanto para archivos `.pbix` como `.pbip`, el procesamiento nunca es local. Las instrucciones y metadatos se transfieren a la nube de Microsoft. Sin una conexión activa y capacidad de Fabric asignada (mínimo SKU F64) en el tenant, la función queda deshabilitada.
* **Ventaja central: Ecosistema y diseño en el lienzo:** Copilot no está diseñado para un modelado semántico minucioso. Su verdadero valor reside en la gobernanza empresarial y su capacidad para generar páginas de informe completas y gráficos directamente sobre el lienzo, algo que ni el Camino 1 ni el Camino 2 pueden realizar.
* **Bloqueo de plataforma y costos elevados:** Las capacidades de Fabric exigen una inversión notable en comparación con APIs abiertas de LLM, sin acceso a prompts de sistema ni herramientas externas de desarrollo.

---

## ¿Pueden las tres opciones hacer todo por igual? Comparativa directa de capacidades

Ninguna herramienta cubre la totalidad del flujo. Cada alternativa presenta ventajas y limitaciones evidentes:

| Requisito / Capacidad | Camino 1: Sistema de archivos | Camino 2: Servidor MCP (En vivo) | Camino 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Formatos compatibles** | Exclusivamente PBIP (TMDL texto) | Tanto PBIX como PBIP | Tanto PBIX como PBIP |
| **Esfuerzo de configuración inicial** | Nulo (funciona en cualquier editor) | Medio (extensión VS Code y config) | Bajo con licencia, prohibitivo sin ella |
| **Creación de medidas DAX** | Sí (edición masiva en TMDL) | Sí (inyectado directo en el motor) | Sí (mediante prompt de chat) |
| **Validación de sintaxis en vivo** | No (edición ciega de texto) | Sí (respuesta inmediata del motor) | Limitada (solo heurística de chat) |
| **Consultas de prueba (`execute_dax`)** | No (sin motor en ejecución) | Sí (consultas directas al motor) | No |
| **Integración con Git y control de versiones** | Nativo del formato PBIP | Disponible con PBIP tras guardar | Ninguna (ligado a la nube) |
| **Requisitos de hardware** | Mínimos (CLI o editor de código) | Altos (Power BI Desktop y RAM) | Ninguno (hospedaje en nube) |
| **Privacidad de datos** | Controlada por el LLM elegido | Controlada por el LLM elegido | Almacenado en la nube de Microsoft |
| **Soporte para DirectQuery** | Limitado (solo metadatos) | Complejo (latencia de consulta) | Sí (soporte nativo en la nube) |
| **Diseño visual en el lienzo** | Limitado (edición ciega de JSON) | No (enfoque exclusivo en modelado) | Sí (crea visuales en el lienzo) |
| **Tarjetas SVG a medida y HTML** | Limitado (cadena DAX a ciegas) | Excelente (vista previa en vivo) | Inadecuado (solo visuales estándar) |
| **Costo y flexibilidad de modelos** | Gratuito (cualquier LLM o local) | Gratuito (cualquier cliente MCP) | Alto (licencia de usuario o Fabric F64) |
| **Requisito de ejecución** | Solo editor de código / CLI | Power BI Desktop abierto localmente | Suscripción activa a Fabric |

### Resumen de aplicación

* **Elegir agentes en sistema de archivos (Camino 1)** cuando se busque trabajar de inmediato sin configuración previa, en entornos Linux o CI/CD, generando medidas en lote sobre archivos TMDL.
* **Elegir MCP (Camino 2)** durante el modelado activo, cuando la validación instantánea del motor, la autocorrección de errores y las tarjetas visuales SVG sean prioritarias.
* **Elegir Copilot (Camino 3)** cuando la organización cuente con infraestructura en Microsoft Fabric y busque crear páginas de informe completas de forma automatizada sobre el lienzo.

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

La ingeniería de BI profesional con IA exige diferenciar con claridad el formato de archivo y el método de trabajo:

1. **PBIP es la base obligatoria:** Controlar versiones con Git requiere abandonar el formato binario PBIX. La integración con Git es una propiedad del formato de archivo, no de la herramienta de IA.
2. **MCP supera con claridad al resto en el modelado activo:** Los agentes de archivos carecen de validación de sintaxis y Copilot es una herramienta de conveniencia costosa para visuales genéricos. Para el modelado semántico riguroso, lógica DAX compleja y tarjetas SVG dinámicas, la conexión en vivo con MCP es la vía más productiva.
3. **El criterio analítico sigue siendo humano:** La IA acelera la generación de código, pero no resuelve la higiene de datos deficiente ni reemplaza el diseño de arquitectura de negocio.

---

Un agradecimiento especial a **Matías Ciancio** por la visualización de la arquitectura y el intercambio técnico.

