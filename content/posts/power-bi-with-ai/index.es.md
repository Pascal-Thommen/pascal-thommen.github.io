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
* **Nivel de archivos (PBIP):** Editar archivos TMDL integrados con Git mediante agentes de código.
* **Sesión en vivo (MCP):** Interactuar con la instancia activa de Power BI Desktop a través del servidor MCP de Analysis Services.
* **Nube (Microsoft Copilot):** Utilizar las capacidades integradas de IA en Microsoft Fabric.

Para el desarrollo activo del modelo, el Camino 2 (MCP) es el método más confiable y económico, ya que permite pruebas en vivo inmediatas con un consumo mínimo de tokens. El Camino 1 destaca en la integración continua con Git y CI/CD, mientras que el Camino 3 está pensado principalmente para usuarios finales de negocio.

---

## Panorama de arquitectura: Cómo interactúan los datos y la IA

Antes de hablar de IA, es indispensable entender la capa de datos.

Power BI se encarga de la conexión directa a los orígenes de datos (SQL Server, PostgreSQL, sistemas ERP o data warehouses) mediante Power Query, DirectQuery o importación programada. El agente de IA **no necesita acceso directo a la base de datos de producción** ni requiere credenciales de la misma. En su lugar, la IA opera exclusivamente sobre la capa del modelo semántico (tablas, relaciones y medidas DAX).

![Panorama de arquitectura: Modelado basado en archivos versus sesión en vivo](architecture_diagram.png)
*Resumen de arquitectura: Modelado basado en archivos vía PBIP (Camino 1) versus sesión en vivo vía MCP (Camino 2) y despliegue a Power BI Service.*

Esta separación aporta una ventaja fundamental de gobernanza: las políticas de seguridad, permisos de usuario y firewalls permanecen dentro de Power BI. La IA solo interactúa con la lógica de cálculo y la estructura visual.

---

## Camino 1: Modelado basado en archivos con PBIP (TMDL)

El formato tradicional `.pbix` es un archivo binario comprimido ilegible para modelos de lenguaje. Al guardar el informe como `.pbip` (Power BI Project), Power BI descompone el modelo en texto plano:

* **Modelo semántico:** Las tablas, relaciones y medidas DAX se guardan como archivos TMDL (Tabular Model Definition Language).
* **Definición de informes:** Los gráficos, formatos y filtros se guardan en archivos JSON estructurados.

Un agente de desarrollo (como Claude Code o cualquier herramienta de terminal) puede trabajar directamente sobre la carpeta, analizar el esquema TMDL y escribir nuevas medidas en el código fuente.

### Limitaciones reales del Camino 1

Aunque este método se integra perfectamente con Git y flujos de CI/CD, presenta restricciones claras:

* **Sin validación de sintaxis en tiempo real:** El agente escribe fórmulas DAX directamente en los archivos de texto. Si hay un error, solo se detecta al abrir o recargar el proyecto en Power BI Desktop.
* **Sin ejecución de consultas de prueba:** El agente no puede ejecutar consultas de prueba (`execute_dax`) contra el motor para comprobar si los números calculados son correctos.
* **Recarga manual:** Las modificaciones realizadas en el disco no se reflejan automáticamente en la ventana abierta de Power BI Desktop sin reiniciar o recargar.

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

Microsoft ofrece también funciones integradas de IA directamente en el servicio en la nube y en Power BI Desktop mediante Copilot.

* **Costos y licenciamiento:** Copilot está vinculado a la nube de Microsoft y requiere licencias de pago o capacidades de Fabric. Esto genera costos continuos de plataforma y bloqueo de proveedor (vendor lock-in).
* **Sistema cerrado:** Funciona como un servicio administrado en la nube sin opciones para personalizar prompts del sistema, encadenar agentes o conectar herramientas externas de desarrollo.
* **Enfoque principal:** Diseñado para usuarios de negocio que buscan resúmenes rápidos y diseños visuales estándar, no para ingeniería profunda del modelo de datos.

---

## ¿Pueden las tres opciones hacer todo por igual? Comparativa directa de capacidades

Ninguna herramienta cubre la totalidad del flujo. Cada alternativa presenta ventajas y limitaciones evidentes:

| Capacidad / Requerimiento | Camino 1: PBIP (Archivos TMDL) | Camino 2: Servidor MCP (En vivo) | Camino 3: Microsoft Copilot |
| :--- | :--- | :--- | :--- |
| **Formatos compatibles** | Solo PBIP (sistema de archivos) | Tanto PBIX como PBIP | Tanto PBIX como PBIP |
| **Creación de medidas DAX** | Sí (por lotes en texto plano) | Sí (inyección directa al modelo) | Sí (prompt en chat) |
| **Validación de sintaxis en vivo** | No (edición ciega de texto) | Sí (respuesta inmediata del motor) | Parcial (solo heurística) |
| **Prueba de consultas DAX (`execute_dax`)** | No (sin motor en ejecución) | Sí (consulta directa al motor) | No |
| **Control de versiones Git y CI/CD** | Excelente (diffs de texto nativos) | Limitado (requiere guardar el modelo) | Nula (bloqueado en la nube) |
| **Requisitos de hardware** | Mínimos (CLI o editor de código) | Altos (aplicación Desktop y RAM) | Nulos (alojado en la nube) |
| **Privacidad de datos** | Depende del LLM elegido | Depende del LLM elegido | Almacenado en la nube de Microsoft |
| **Soporte para DirectQuery** | Limitado (solo metadatos) | Complejo (latencia de consulta) | Sí (soporte nativo en la nube) |
| **Diseño visual y generación de gráficos** | Limitado (edición ciega de JSON) | No (enfoque exclusivo en modelado) | Sí (crea visuales en el lienzo) |
| **Tarjetas SVG a medida y HTML** | Limitado (cadena DAX a ciegas) | Excelente (vista previa en vivo) | Inadecuado (solo visuales estándar) |
| **Costo y flexibilidad de modelos** | Gratuito (cualquier LLM o modelo local) | Gratuito (cualquier cliente MCP) | Alto (licencia de usuario o Fabric F64) |
| **Requisito de ejecución** | Solo editor de código / CLI | Power BI Desktop abierto localmente | Suscripción activa a Fabric |

### Resumen de aplicación

* **Usar PBIP** cuando se requiera control de versiones en Git, integración continua CI/CD y generación masiva de medidas sin abrir Power BI Desktop.
* **Usar MCP** durante el modelado activo frente a la máquina, cuando se necesite validación en tiempo real del motor, autocorrección de errores y visuales avanzados con SVG.
* **Usar Copilot** cuando la organización disponga del presupuesto para Microsoft Fabric y busque crear páginas de informe rápidas y genéricas para usuarios finales.

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

Integrar Power BI con IA no consiste en buscar una herramienta mágica que lo haga todo, sino en elegir la vía adecuada para cada necesidad. PBIP aporta la disciplina del desarrollo de software en Git, MCP brinda un asistente interactivo con validación real del motor, y las soluciones en la nube cubren reportes generales. El juicio analítico permanece en manos humanas mientras la velocidad de ejecución se multiplica notablemente.

---

Un agradecimiento especial a **Matías Ciancio** por la visualización de la arquitectura y el intercambio técnico.

