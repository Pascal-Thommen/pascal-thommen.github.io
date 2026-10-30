---
title: "UPA Campus App: Arquitectura e Ingeniería de Sistemas (Serie)"
date: 2026-10-30T17:00:00Z
description: "Serie completa de arquitectura en 6 capítulos de la UPA Campus App: Cómo se diseñan las aplicaciones web modernas desde cero y por qué cada regla de la especificación funcional nace de una necesidad arquitectónica."
summary: "Desde túneles de ingreso cifrados y orquestación de contenedores hasta PWA, backend en FastAPI y persistencia relacional: Guía práctica de arquitectura web y diseño de especificaciones."
tags: ["Arquitectura de Software", "Docker", "FastAPI", "PostgreSQL", "NetBird", "Enterprise Systems"]
categories: ["Arquitectura", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_title: "Arquitectura UPA Campus App"
series_hub: true
aliases:
  - "/es/posts/upa-campus-app-systemhandbuch/"
---

Las aplicaciones web modernas no se construyen simplemente uniendo pantallas visuales, sino coordinando seguridad de red, aislamiento de contenedores, lógica de negocio sin estado y persistencia transaccional. En una etapa donde los asistentes de inteligencia artificial generan código sintáctico en segundos, el éxito de un proyecto ya no depende de memorizar bucles o propiedades de estilo, sino de dominar la arquitectura de sistemas: fronteras del sistema, contratos de interfaz, flujos de datos y modelos relacionales. Tomando la UPA Campus App como caso de estudio real, esta serie de seis capítulos transmite un modelo mental universal sobre cómo funcionan las aplicaciones web modernas y demuestra por qué cada requisito del pliego funcional constituye un plano lógico derivado de la arquitectura.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.12-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## Visión General del Sistema y Flujo Arquitectónico

Una sola pulsación en el teléfono móvil o en el portal administrativo recorre en milisegundos cuatro capas estrictamente aisladas, desde el túnel de ingreso cifrado hasta el motor de almacenamiento persistente:

```mermaid
flowchart LR
    Client["Cliente (PWA / App Móvil)"] -->|HTTPS 443| NetBird["Túnel Ingress NetBird"]
    NetBird -->|127.0.0.1:8085| Nginx["Contenedor Frontend (Nginx)"]
    Nginx -->|Recursos Estáticos y App Shell| Client
    Nginx -->|Proxy Inverso /api/ Puerto 8000| Backend["Contenedor Backend (FastAPI)"]
    Backend -->|SQL Puerto 5432| DB[("Contenedor PostgreSQL")]
    DB -->|Tablas Relacionales| VolDB[("Volumen: postgres_data")]
    Backend -->|Archivos Multimedia| VolMedia[("Volumen: uploads_data")]
    Nginx -->|Proxy Inverso /uploads/| VolMedia
```

---

## Por Qué la Arquitectura y la Especificación Funcional Son Inseparables

En muchos proyectos de software, la especificación funcional se percibe como una lista arbitraria de deseos. En una arquitectura web sólida ocurre exactamente lo contrario: cada regla funcional protege al sistema frente a datos inconsistentes, brechas de seguridad o saturación de recursos.

1. **Red e Identidad Zero-Trust:** El servidor de producción no abre puertos públicos en el enrutador; todo el tráfico entrante viaja por una malla cifrada WireGuard (NetBird) hacia el proxy local Nginx. En la capa de aplicación, esta defensa continúa con el inicio de sesión obligatorio mediante Google OAuth para cuentas institucionales y el principio de privilegio mínimo en la matriz de roles: las cuentas nuevas nacen con todas las casillas de permisos desmarcadas y sin botones de acceso rápido de desarrollo.
2. **Desacoplamiento Estricto de Contenedores y Clientes:** La plataforma separa la entrega estática (Nginx), la lógica de API (FastAPI) y el motor relacional (PostgreSQL) en contenedores independientes. Así, un único backend alimenta tanto al portal editorial de escritorio como a la interfaz móvil estudiantil con soporte offline y sincronización de entregas de Google Classroom.
3. **Fuente Única de Verdad en el Backend:** En lugar de dispersar noticias, eventos de calendario y avisos de emergencia en tablas separadas, un modelo unificado de publicaciones (`/api/publications`) concentra todos los formatos. De esta estructura emanan directamente las reglas del editor: fijar un aviso arriba (`is_pinned`) exige una fecha de vencimiento (`expires_at`) para evitar que alertas meteorológicas caducadas bloqueen el feed de forma permanente. Asimismo, las publicaciones sincronizadas desde Instagram permanecen en modo de solo lectura en el portal editorial porque el proceso periódico de sincronización sobrescribiría cualquier edición manual en su siguiente ciclo.
4. **Persistencia Determinista de Datos:** Los metadatos estructurados, el conteo de lecturas únicas (`PostView`) y las inscripciones en 1 clic (`EventRegistration`) se guardan con garantías ACID en PostgreSQL, mientras que las imágenes, videos y documentos PDF se alojan en un volumen dedicado del host para no saturar la memoria caché de la base de datos.

---
