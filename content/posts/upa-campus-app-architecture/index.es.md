---
title: "UPA Campus App: Arquitectura e Ingeniería de Sistemas (Serie)"
date: 2026-10-30T17:00:00Z
description: "Serie completa de arquitectura de 6 capítulos de la UPA Campus App: Ingress Zero-Trust con NetBird, proxy inverso Nginx, PWA, backend FastAPI y persistencia en PostgreSQL."
summary: "Serie de arquitectura de 6 capítulos de la UPA Campus App: Desde túneles de ingress cifrados y orquestación de contenedores hasta PWA, backend FastAPI y persistencia relacional."
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

La UPA Campus App centraliza avisos académicos, calendarios y registros de eventos dentro de una infraestructura modular de tres contenedores. Esta serie técnica de seis capítulos documenta la arquitectura integral del sistema: desde la seguridad de red y la orquestación de contenedores hasta el procesamiento asíncrono de APIs y la persistencia relacional.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.11-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

## Visión General del Sistema y Flujo Arquitectónico

El flujo de datos conecta directamente al cliente a través de una red superpuesta privada con el entorno aislado de contenedores:

```mermaid
flowchart LR
    Client["Cliente (PWA / Navegador)"] -->|HTTPS 443| NetBird["Túnel Ingress NetBird"]
    NetBird -->|Puerto 8085| Nginx["Contenedor Frontend (Nginx / Web-Admin)"]
    Nginx -->|Recursos Estáticos| Client
    Nginx -->|Proxy Inverso /api/ Puerto 8000| Backend["Contenedor Backend (FastAPI)"]
    Backend -->|SQL Puerto 5432| DB[("Contenedor PostgreSQL")]
    DB -->|Tablas Persistentes| VolDB[("Volumen: postgres_data")]
    Backend -->|Subidas / Medios| VolMedia[("Volumen: uploads_data")]
    Nginx -->|Proxy Inverso /uploads/| VolMedia
```

---

## Principios Arquitectónicos Fundamentales

1. **Integridad de Red Zero-Trust:** El servidor de producción no expone puertos públicos en el enrutador. Todo el tráfico de entrada transita por una red mallada WireGuard privada mediante NetBird.
2. **Aislamiento Estricto de Contenedores:** Frontend, capa de API y base de datos operan en contenedores separados dentro de una red bridge dedicada de Docker.
3. **Backend Sin Estado con E/S Asíncrona:** FastAPI garantiza validación estricta de tipos mediante Pydantic y procesamiento sin estado de solicitudes.
4. **Persistencia Determinista de Datos:** Clara separación entre datos relacionales estructurados en PostgreSQL y archivos multimedia no estructurados en volúmenes dedicados del host.

---
