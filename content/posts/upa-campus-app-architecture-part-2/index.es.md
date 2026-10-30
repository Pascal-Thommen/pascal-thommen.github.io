---
title: "UPA Campus App (Capítulo 2): Servidor e Infraestructura Docker"
date: 2026-10-30T16:15:00Z
description: "Contenedores, aislamiento de puertos y persistencia: Cómo la UPA Campus App protege PostgreSQL, FastAPI y Nginx con Docker Compose y por qué el desacoplamiento de contenedores es una decisión clave de la especificación."
summary: "¿Cómo funcionan los contenedores, las cadenas de arranque con healthchecks y los volúmenes persistentes en la práctica? El capítulo 2 analiza Dockerfiles, aislamiento de puertos y desacoplamiento arquitectónico."
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

Las aplicaciones web modernas exigen entornos de ejecución reproducibles, dominios de fallo aislados y persistencia de datos segura, porque un error en la interfaz visual jamás debe derribar la base de datos ni el núcleo de la API. La UPA Campus App cumple estos requisitos mediante tres contenedores Docker desacoplados para la base de datos, la API REST y el frontend web. Este segundo capítulo examina cómo funcionan los contenedores por dentro, cómo se diseñan los Dockerfiles, cómo Docker Compose coordina el orden de arranque y cómo se separa el código efímero del almacenamiento duradero.

---

## Contenedores frente a Máquinas Virtuales

¿Por qué la UPA Campus App se ejecuta sobre Docker en lugar de máquinas virtuales tradicionales (VM) o directamente sobre el sistema operativo del servidor? La diferencia clave reside en el nivel de virtualización y el consumo de memoria:

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

Las máquinas virtuales emulan el hardware físico completo. Cada VM necesita arrancar un sistema operativo invitado entero con su propio kernel y servicios en segundo plano, reservando varios gigabytes de RAM por instancia y tardando minutos en iniciar.

Los contenedores Docker comparten el kernel Linux del servidor anfitrión y utilizan dos mecanismos nativos del kernel para aislar cada servicio:
* **Namespaces:** Aíslan los árboles de procesos (`PID`), las tarjetas de red (`NET`), los identificadores de usuario (`USER`) y los puntos de montaje de disco (`MNT`). Cada contenedor solo ve sus propios procesos.
* **Control Groups (cgroups):** Limitan y supervisan el uso de memoria RAM, ciclos de CPU y lectura/escritura en disco para impedir que un servicio descontrolado agote los recursos del servidor.

En un equipo universitario, Docker resuelve además el clásico problema de incompatibilidad entre computadoras de desarrollo ("en mi máquina sí funciona"): sin importar si un integrante programa en Windows con WSL2, macOS o Linux, la imagen del contenedor empaqueta de forma idéntica la versión exacta de Python, los controladores de PostgreSQL y la configuración de Nginx.

---

## Por Qué Frontend y Backend Operan en Contenedores Separados

En la matriz de decisiones arquitectónicas de la UPA Campus App (Decisión 3.3: *Desacoplamiento de Contenedores Frontend y Backend*), se determinó separar el portal web administrativo y el motor FastAPI en contenedores independientes en lugar de mezclarlos en una sola imagen:

```mermaid
flowchart LR
    Browser["Navegador / App Móvil"] -->|127.0.0.1:8085| Nginx["Contenedor Frontend: Nginx (UI Estática y Proxy)"]
    Nginx -->|Proxy Inverso /api/ Puerto 8000| FastAPI["Contenedor Backend: FastAPI (Lógica de Negocio)"]
    FastAPI -->|SQL Puerto 5432| Postgres[("Contenedor BD: PostgreSQL")]
    Postgres -->|Tablas ACID| VolPG[("Volumen Host: postgres_data")]
    FastAPI -->|Imágenes, Videos y PDFs| VolUpload[("Volumen Host: uploads_data")]
```

Esta separación aporta tres ventajas operativas directas para el campus:
1. **Actualizaciones visuales sin interrumpir la API:** Si el equipo de marketing solicita un ajuste en el editor de publicaciones (`web-admin`) o en la pantalla de inicio con código QR, solo se reconstruye y reinicia el contenedor ligero de Nginx en menos de un segundo. El contenedor FastAPI y las conexiones activas de la aplicación Android continúan operando sin cortes.
2. **Fronteras de seguridad aisladas:** El contenedor Nginx no contiene el intérprete de Python, ni contraseñas de base de datos, ni las llaves de la cuenta de servicio de Google. Incluso ante un fallo en la capa de presentación, los secretos del backend permanecen inaccesibles.
3. **Entorno Zero-Mock para el equipo móvil:** Como el contenedor del backend incluye migraciones automáticas y datos iniciales representativos del campus, el equipo de Android puede levantar todo el sistema en su propia computadora con un solo comando (`docker compose up -d`) y programar contra endpoints reales desde el primer día.

---

## Anatomía de los Tres Contenedores

### 1. Base de Datos: Imagen Oficial de PostgreSQL

Para la persistencia relacional se emplea la imagen oficial `postgres:16-alpine`. Mantener un Dockerfile propio para una base de datos estándar solo añadiría carga de mantenimiento. La edición sobre Alpine Linux ocupa menos de 100 MB, incorpora los parches oficiales de seguridad y se configura íntegramente mediante variables de entorno (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`).

### 2. Backend: Dockerfile Personalizado con Python y FastAPI

El backend procesa todas las llamadas REST y tareas programadas sobre la imagen base `python:3.12-slim`:

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

Tres decisiones dentro de este Dockerfile garantizan un despliegue rápido y seguro:
* **Caché de capas (Layer Caching):** Docker guarda cada instrucción como una capa inmutable. Al colocar `COPY requirements.txt .` y `pip install` antes de `COPY src/ ./src`, Docker reutiliza las librerías ya instaladas cuando solo cambia el código fuente, reduciendo el tiempo de construcción de minutos a segundos.
* **Principio de privilegio mínimo (`USER appuser`):** Por omisión, los contenedores ejecutan sus procesos como `root`. Al cambiar al usuario de sistema `appuser`, el proceso de Python carece de permisos administrativos dentro del contenedor ante cualquier vulnerabilidad externa.
* **Diagnóstico automático (`HEALTHCHECK`):** Cada 15 segundos, Docker consulta internamente el endpoint `/health`. Si FastAPI falla tres veces seguidas, Docker marca el contenedor como `unhealthy`.

### 3. Frontend: Dockerfile Personalizado con Nginx

El contenedor frontend empaqueta las vistas estáticas (portal editorial Web-Admin y PWA del campus) junto con las reglas de enrutamiento:

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

### 4. Coordinación de Arranque con Docker Compose

¿Qué ocurre cuando los tres contenedores arrancan al mismo tiempo tras reiniciar el servidor? PostgreSQL necesita varios segundos para verificar sus registros transaccionales, mientras que Python inicia casi al instante. Sin coordinación, FastAPI intentaría ejecutar migraciones sobre una base de datos que aún no acepta conexiones y se cerraría con error.

Docker Compose evita esta condición de carrera mediante `condition: service_healthy`:

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
    build:
      context: .
      dockerfile: web-admin/Dockerfile
    container_name: campus-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "127.0.0.1:8085:80"
    networks:
      - campus-network
```

Solo cuando la utilidad `pg_isready` confirma que PostgreSQL está listo para recibir consultas SQL, Docker inicia el contenedor del backend.

---

## Seguridad mediante Aislamiento de Puertos

Un error frecuente al configurar contenedores consiste en exponer todos los puertos internos hacia el servidor anfitrión (por ejemplo, publicar `5432:5432` en producción).

```mermaid
flowchart LR
    subgraph PublicInternet["Internet Pública"]
        Attacker["Escáneres Automatizados de Puertos"]
    end
    subgraph HostServer["Servidor Host"]
        NetBird["Interfaz de Túnel NetBird"]
        subgraph DockerBridge["Red Docker Bridge Aislada"]
            Frontend["Frontend (Puerto 80)"]
            Backend["Backend (Puerto 8000)"]
            DB[("PostgreSQL (Puerto 5432)")]
        end
    end
    Attacker -.->|Bloqueado: 0 Puertos Públicos Abiertos| HostServer
    NetBird -->|Acceso en 127.0.0.1:8085| Frontend
    Frontend -->|DNS Interno: backend:8000| Backend
    Backend -->|DNS Interno: postgres:5432| DB
```

En producción, la UPA Campus App aplica dos reglas de red estrictas:
1. **Sin puertos expuestos en el host para Base de Datos y API:** Ni el puerto 5432 ni el puerto 8000 se publican hacia el exterior en producción. Los contenedores se comunican a través del servidor DNS interno de Docker usando sus nombres de servicio (`postgres:5432` y `backend:8000`). Fuera de la red bridge `campus-network`, esos puertos no existen.
2. **Vinculación exclusiva a Loopback (`127.0.0.1:8085:80`):** Si se declarara `8085:80` a secas, Docker publicaría el puerto en todas las tarjetas de red (`0.0.0.0`) y modificaría automáticamente las tablas `iptables` de Linux, saltándose cortafuegos locales como UFW. Al anteponer `127.0.0.1:`, Nginx escucha únicamente en la interfaz local loopback del propio servidor, donde solo el túnel cifrado de NetBird puede entregarle tráfico.

---

## Contenedores Efímeros frente a Volúmenes Persistentes

¿Por qué se perderían los datos dentro de un contenedor normal cada vez que se actualiza el código?

```mermaid
flowchart TD
    subgraph ContainerLifecycle["Capa del Contenedor (Efímera)"]
        AppCode["Imagen Inmutable (Código y Librerías)"]
        TempFS["Capa de Escritura Temporal (OverlayFS)"]
        KillAction["Actualización (docker compose up -d --build)"]
        TempFS -.->|Se destruye por completo al actualizar| KillAction
    end
    subgraph HostStorage["Sistema de Archivos del Host (Duradero)"]
        PGVolume[("Volumen Docker: postgres_data")]
        UploadVolume[("Volumen Docker: uploads_data")]
    end
    ContainerLifecycle -->|Montado en /var/lib/postgresql/data| PGVolume
    ContainerLifecycle -->|Montado en /app/uploads| UploadVolume
```

Todo contenedor en ejecución combina capas de imagen de solo lectura con una delgada capa superior de escritura (`OverlayFS`). Cuando se despliega una mejora mediante `docker compose up -d --build`, Docker elimina el contenedor antiguo junto con su capa temporal y crea uno nuevo a partir de la imagen actualizada.

Para garantizar que ni las inscripciones de los estudiantes ni los videos de eventos desaparezcan en cada actualización, Docker Compose conecta dos **volúmenes con nombre** del disco del servidor (`/var/lib/docker/volumes/`) dentro de los contenedores:
* **`postgres_data`:** Sobrevive a cualquier recreación del contenedor y conserva intactas las tablas, índices y registros de transacciones.
* **`uploads_data`:** Conserva todas las imágenes (`JPG`, `PNG`, `WEBP`), videos (`MP4`, `WEBM`) y documentos (`PDF`) cargados desde el campo multimedia único del editor.

---

## Conclusiones

1. **El Desacoplamiento Garantiza Continuidad:** Ejecutar Nginx, FastAPI y PostgreSQL en contenedores separados permite actualizar la interfaz visual al instante sin cortar las conexiones activas de la API.
2. **Arranque Determinista:** La combinación de `pg_isready` con `condition: service_healthy` asegura que el backend jamás inicie antes de que la base de datos esté operativa.
3. **Puertos Cerrados y Volúmenes Duraderos:** Los servicios internos dialogan exclusivamente dentro de la red bridge privada de Docker, mientras los volúmenes en el host permiten actualizar el software sin perder un solo dato.
