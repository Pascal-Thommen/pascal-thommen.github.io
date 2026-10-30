---
title: "UPA Campus App (Capítulo 2): Servidor e Infraestructura Docker"
date: 2026-10-30T16:15:00Z
description: "Contenedores, aislamiento de puertos y persistencia: Cómo la UPA Campus App asegura PostgreSQL, FastAPI y Nginx con Docker Compose."
summary: "¿Cómo funcionan la contenerización, el aislamiento de puertos y la persistencia en producción? El capítulo 2 profundiza en Dockerfiles y Docker Compose."
tags: ["Docker", "Arquitectura de Software", "PostgreSQL", "FastAPI", "Sistemas Empresariales"]
categories: ["Arquitectura", "Sistemas Empresariales"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 2
series_part_title: "Servidor e Infraestructura Docker"
hiddenInList: true
---

Las aplicaciones web modernas exigen entornos de ejecución reproducibles, aislamiento de fallos y persistencia de datos fiable. La UPA Campus App implementa estos requisitos mediante tres contenedores modulares para base de datos, API REST y frontend web. Esta segunda entrega analiza el diseño de los Dockerfiles, la orquestación declarativa con Docker Compose, los mecanismos de aislamiento de puertos y la estrategia de almacenamiento persistente.

---

## Contenedores frente a máquinas virtuales

¿Por qué la UPA Campus App utiliza Docker en lugar de máquinas virtuales tradicionales? La diferencia fundamental radica en el nivel de virtualización y en el consumo de recursos:

```mermaid
flowchart TD
    subgraph VM_Arch["Máquinas Virtuales (Virtualización de Hardware)"]
        direction TB
        AppVM["Aplicación y Librerías"]
        GuestOS["Sistema Operativo Huésped Completo"]
        Hypervisor["Hipervisor"]
        HostHW1["Sistema Host y Hardware"]
        AppVM --> GuestOS --> Hypervisor --> HostHW1
    end
    subgraph Docker_Arch["Contenedores (Virtualización de Sistema Operativo)"]
        direction TB
        AppDocker["Aplicación y Librerías"]
        Engine["Motor Docker"]
        SharedKernel["Kernel Linux Compartido"]
        HostHW2["Sistema Host y Hardware"]
        AppDocker --> Engine --> SharedKernel --> HostHW2
    end
```

Las máquinas virtuales (VM) tradicionales virtualizan el hardware físico. Cada máquina requiere un sistema operativo huésped completo con su propio kernel, gestión de memoria y servicios en segundo plano. Esto reserva varios gigabytes de memoria por instancia y requiere tiempos de arranque de varios minutos.

Los contenedores Docker comparten el kernel Linux del sistema host. Docker aprovecha dos capacidades fundamentales del kernel:
* **Namespaces:** Aíslan los árboles de procesos (PID), interfaces de red (NET), identificadores de usuario (USER) y puntos de montaje del sistema de archivos (MNT). Cada contenedor observa únicamente sus propios procesos sin acceso al resto.
* **Control Groups (cgroups):** Restringen y monitorizan el uso de hardware como memoria RAM, ciclos de CPU y lectura o escritura de disco. Un proceso con consumo excesivo no compromete la estabilidad del servidor host.

Gracias a esta arquitectura, los contenedores arrancan en fracciones de segundo y consumen únicamente los recursos requeridos por el proceso en ejecución.

Asimismo, Docker elimina las discrepancias entre equipos de desarrollo. Tanto en macOS como en Linux o Windows con WSL2, el contenedor encapsula el entorno Python, paquetes del sistema, configuración de Nginx y esquema de base de datos con total paridad. El entorno local reproduce con exactitud el servidor de producción.

---

## Los tres contenedores de la UPA Campus App

La infraestructura del campus se divide en tres contenedores especializados comunicados mediante una red Docker privada.

```mermaid
flowchart LR
    Browser["Navegador Cliente"] -->|Puerto 8085| Nginx["Contenedor Frontend: Nginx / PWA"]
    Nginx -->|Reverse Proxy /api/ Puerto 8000| FastAPI["Contenedor Backend: FastAPI"]
    FastAPI -->|SQL Puerto 5432| Postgres[("Contenedor BD: PostgreSQL")]
    Postgres -->|Persistencia ACID| VolPG[("Volumen Host: postgres_data")]
    FastAPI -->|Archivos y Multimedia| VolUpload[("Volumen Host: uploads_data")]
```

### 1. Base de datos: Imagen oficial de PostgreSQL

Para el almacenamiento relacional se utiliza la imagen oficial `postgres:16-alpine`. La selección de una imagen oficial aporta ventajas operativas directas:
* No requiere mantenimiento de compilaciones ni parches manuales de seguridad.
* Alpine Linux mantiene el tamaño total de la imagen por debajo de 100 MB.
* La configuración se define de forma puramente declarativa mediante variables de entorno (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`).

Escribir Dockerfiles propios para motores relacionales estándar representa habitualmente un antipatrón que incrementa la deuda técnica sin aportar valor arquitectónico.

### 2. Backend: Dockerfile personalizado con Python y FastAPI

El backend procesa la lógica de negocio y expone la API REST. La imagen base es `python:3.12-slim`:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app \
    UPLOAD_DIR=/app/uploads

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN addgroup --system appgroup && adduser --system --group appuser

COPY src/ ./src
COPY alembic/ ./alembic
COPY alembic.ini .

RUN mkdir -p /app/uploads && chown -R appuser:appgroup /app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=15s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Decisiones de diseño en el Dockerfile del backend:
* **Caché óptima de capas:** La copia e instalación de `requirements.txt` se realiza antes de copiar el código fuente. Al modificar la lógica de la aplicación, Docker reutiliza la capa de dependencias en caché y acelera los tiempos de compilación.
* **Principio de menor privilegio:** La aplicación se ejecuta bajo el usuario sin privilegios `appuser` en lugar de `root`. Ante cualquier vulnerabilidad en una librería externa, el atacante carece de privilegios administrativos en el sistema.
* **Healthcheck integrado:** Docker verifica el endpoint `/health` cada 15 segundos. Si la comprobación falla tres veces consecutivas, el contenedor queda catalogado como `unhealthy`.

### 3. Frontend: Dockerfile personalizado con Nginx

El frontend distribuye la interfaz de administración como Progressive Web App (PWA) y gestiona el enrutamiento interno:

```dockerfile
FROM nginx:alpine

RUN rm -rf /usr/share/nginx/html/*

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=15s --timeout=5s --start-period=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1:80/frontend-health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

Nginx desempeña dos roles clave:
* **Distribución de estáticos:** Archivos HTML, CSS, JavaScript y manifiestos PWA se sirven comprimidos mediante Gzip.
* **Reverse Proxy:** Nginx redirige solicitudes bajo `/api/` hacia el contenedor FastAPI. El navegador web interactúa exclusivamente con el puerto del frontend, suprimiendo conflictos de CORS (Cross-Origin Resource Sharing) en producción.

### 4. Orquestación con Docker Compose

Docker Compose describe el ecosistema completo en un manifiesto centralizado:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: campus-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: db_user
      POSTGRES_PASSWORD: secret_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U db_user -d app_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - campus-network

  backend:
    build: ./backend
    container_name: campus-backend
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      APP_ENV: production
      DATABASE_URL: postgresql://db_user:secret_password@postgres:5432/app_db
    volumes:
      - uploads_data:/app/uploads
    networks:
      - campus-network

  frontend:
    build: ./web-admin
    container_name: campus-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "127.0.0.1:8085:80"
    networks:
      - campus-network

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

Un mecanismo crítico de estabilidad es la coordinación de arranque:
`depends_on` con `condition: service_healthy` asegura que el backend inicie únicamente cuando PostgreSQL responde positivamente a `pg_isready`. Sin esta validación, la API fallaría al intentar ejecutar migraciones de base de datos durante el inicio del sistema.

---

## Seguridad y puertos: Aislamiento estricto de red

Un fallo habitual en despliegues con contenedores consiste en exponer puertos internos al host sin restricción.

```mermaid
flowchart LR
    subgraph PublicInternet["Internet Pública"]
        Attacker["Escáneres Automatizados"]
    end
    subgraph HostServer["Servidor Host"]
        NetBird["Túnel Mesh NetBird"]
        subgraph DockerBridge["Red Docker Bridge Aislada"]
            Frontend["Frontend (Puerto 80)"]
            Backend["Backend (Puerto 8000)"]
            DB[("PostgreSQL (Puerto 5432)")]
        end
    end
    Attacker -.->|Bloqueado: Sin puertos públicos| HostServer
    NetBird -->|Acceso exclusivo por túnel| Frontend
    Frontend -->|Proxy interno| Backend
    Backend -->|Consulta SQL interna| DB
```

La arquitectura de la UPA Campus App aplica dos reglas estrictas:

### 1. Puertos internos confinados a la red Bridge

Ni el puerto 5432 de PostgreSQL ni el puerto 8000 de FastAPI se declaran en la directiva `ports` de Compose. Operan exclusivamente dentro de la red virtual `campus-network`. La resolución se efectúa mediante nombres internos del DNS de Docker (`postgres:5432`, `backend:8000`). Ni el sistema operativo del host ni redes externas pueden acceder a dichos puertos.

### 2. Vinculación estricta a Localhost

El contenedor frontend vincula el puerto 80 mediante `127.0.0.1:8085:80` en lugar de la sintaxis por defecto `8085:80`.

Por omisión, Docker publica puertos en `0.0.0.0`. En sistemas Linux, el demonio Docker altera tablas de `iptables`, ignorando frecuentemente reglas del cortafuegos local (como UFW). La vinculación directa a `127.0.0.1` restringe el tráfico a la interfaz loopback. El acceso externo se canaliza a través de un túnel privado NetBird con cifrado WireGuard, eliminando cualquier puerto expuesto directamente a internet.

---

## Persistencia y volúmenes: Contenedores efímeros y almacenamiento duradero

Los contenedores siguen los principios de la metodología 12-Factor App: carecen de estado persistente interno y resultan fácilmente reemplazables.

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Capa del Contenedor (Efímera)"]
        AppCode["Código de Aplicación y Librerías"]
        TempFS["Capa de Escritura Temporal (UnionFS)"]
        KillAction["Reinicio o Reconstrucción"]
        TempFS -.->|Destruida al reconstruir| KillAction
    end
    subgraph HostStorage["Almacenamiento Host (Duradero)"]
        PGVolume[("Volumen Docker: postgres_data")]
        UploadVolume[("Volumen Docker: uploads_data")]
    end
    ContainerLifecycle -->|Montado en /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Montado en /app/uploads| UploadVolume
```

### El sistema de archivos efímero del contenedor

Cada contenedor posee una capa de escritura temporal (OverlayFS / UnionFS). Cualquier fichero generado durante la ejecución reside en esa capa volátil.

Al detener o reemplazar un contenedor con `docker compose down` y `docker compose up -d`, la capa de escritura se elimina de forma definitiva. Si PostgreSQL almacenase tablas en el sistema de ficheros interno del contenedor, cada actualización causaría la pérdida total de los datos.

### Volúmenes con nombre para persistencia de datos

Los volúmenes de Docker desacoplan el almacenamiento del ciclo de vida del software:
* **Separación de responsabilidades:** El contenedor preserva código inmutable, mientras los datos residen en el almacenamiento del host (`/var/lib/docker/volumes/`).
* **Independencia del ciclo de vida:** El volumen con nombre `postgres_data` sobrevive a la destrucción del contenedor. Un despliegue con `docker compose up -d --build` sustituye el contenedor sin alterar el contenido montado en `/var/lib/postgresql/data`.
* **Rendimiento nativo:** Los volúmenes omiten la sobrecarga Copy-on-Write del sistema de ficheros del contenedor y entregan velocidad nativa de disco.

### Separación entre registros relacionales y archivos multimedia

La arquitectura divide el almacenamiento en dos volúmenes:
1. `postgres_data`: Custodia tablas relacionales, registros WAL de transacciones e índices.
2. `uploads_data`: Almacena documentos PDF e imágenes recibidos por la API.

Esta segregación evita almacenar objetos binarios grandes (BLOBs) en PostgreSQL, garantizando copias de seguridad compactas y alta eficiencia en la memoria caché de la base de datos.

---

## Conclusiones

1. **Paridad técnica garantizada:** Imágenes base reducidas, usuarios sin privilegios y capas optimizadas eliminan discrepancias entre entornos locales y servidores en la nube.
2. **Topología de red defensiva:** Los servicios internos permanecen aislados en la red bridge, mientras la vinculación a `127.0.0.1` combinada con túneles mesh neutraliza vectores de ataque directo.
3. **Cómputo sin estado y persistencia desacoplada:** El uso de contenedores efímeros simplifica actualizaciones sin riesgo, apoyado en volúmenes dedicados que protegen la información corporativa.
