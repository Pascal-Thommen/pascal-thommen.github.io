---
title: "UPA Campus App (Capítulo 2): Servidor y Arquitectura de Contenedores"
date: 2026-10-30T16:15:00Z
description: "¿Por qué la UPA Campus App se ejecuta en tres contenedores Docker independientes? El Capítulo 2 explica la virtualización a nivel de sistema operativo, Dockerfiles, healthchecks y volúmenes persistentes."
summary: "¿Cómo garantizan los contenedores Docker, los healthchecks y los volúmenes una operación estable sin pérdida de datos? El Capítulo 2 detalla el diseño de la infraestructura del servidor."
tags: ["Docker", "Arquitectura de Software", "PostgreSQL", "FastAPI", "Enterprise Systems"]
categories: ["Arquitectura", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 2
series_part_title: "Servidor y Arquitectura de Contenedores"
hiddenInList: true
---

¿Cómo se garantiza que una aplicación web funcione exactamente igual en la computadora de cada estudiante desarrollador y en el servidor de producción, evitando que un fallo en la interfaz afecte a la base de datos? Empaquetando el servidor web, la lógica de negocio y el motor de base de datos en tres contenedores Linux aislados con áreas de almacenamiento independientes. Este segundo capítulo analiza el funcionamiento de Docker, la construcción de Dockerfiles listos para producción, la cadena de arranque coordinada tras un reinicio y la separación entre contenedores efímeros y datos persistentes.

---

## Virtualización a Nivel de Sistema Operativo frente a Máquinas Virtuales

En los equipos de desarrollo universitarios es habitual trabajar con distintos sistemas operativos (Windows, macOS, Linux). Cuando Python o PostgreSQL se instalan directamente sobre el sistema anfitrión, las diferencias de versiones y librerías generan fallos que solo ocurren en determinados equipos.

Las máquinas virtuales (VM) tradicionales resuelven esto emulando el hardware completo, pero obligan a levantar un sistema operativo invitado entero con su propio kernel por cada servicio, consumiendo varios gigabytes de memoria RAM y demorando minutos en arrancar. Los contenedores Docker comparten en cambio el kernel Linux del servidor anfitrión:

```mermaid
flowchart TD
    subgraph VM_Stack["Máquina Virtual Tradicional"]
        direction TB
        V1["Aplicación y Librerías"] --> V2["Sistema Operativo Invitado Completo (Kernel Propio)"]
        V2 --> V3["Hipervisor"] --> V4["Hardware del Servidor"]
    end
    subgraph Container_Stack["Contenedorización con Docker"]
        direction TB
        C1["Aplicación y Librerías"] --> C2["Motor Docker"]
        C2 --> C3["Kernel Linux Compartido (Namespaces y cgroups)"] --> C4["Hardware del Servidor"]
    end
```

Dos funciones nativas del kernel Linux hacen posible este aislamiento ligero:
* **Namespaces:** Proporcionan a cada contenedor una vista aislada de procesos (`PID`), interfaces de red (`NET`) y puntos de montaje (`MNT`). Un contenedor no puede ver los procesos de otro.
* **Control Groups (cgroups):** Regulan el consumo de memoria RAM y procesador para que una consulta pesada no bloquee el resto del servidor.

---

## Por Qué la Especificación Exige Tres Contenedores Separados

La especificación técnica de la UPA Campus App establece que el portal web (`campus-frontend`), el servidor API (`campus-backend`) y la base de datos (`campus-postgres`) deben ejecutarse como tres contenedores independientes en lugar de agruparse en una sola imagen:

```mermaid
flowchart LR
    UI["Contenedor 1: campus-frontend\nNginx (Web-Admin y PWA)"] -->|Llamadas API Internas| API["Contenedor 2: campus-backend\nPython 3.12 y FastAPI"]
    API -->|Transacciones SQL| DB[("Contenedor 3: campus-postgres\nPostgreSQL 16 Alpine")]
```

Esta separación aporta tres ventajas operativas directas:
1. **Actualizaciones Visuales sin Interrumpir la API:** Si el equipo de marketing solicita un ajuste visual en el panel administrativo, solo se reconstruye y reemplaza el contenedor ligero de Nginx en menos de un segundo. El contenedor de FastAPI y las conexiones activas de la aplicación móvil continúan operando sin cortes.
2. **Reducción de la Superficie de Exposición:** El contenedor de frontend no incluye intérprete de Python, credenciales de base de datos ni llaves de servicios de Google.
3. **Entorno Local Real para el Equipo Móvil:** Como el contenedor de backend incluye migraciones automáticas y datos iniciales de prueba, los desarrolladores de Android pueden levantar un servidor completo en su equipo con un solo comando (`docker compose up -d`) y programar contra endpoints reales en vez de usar simulaciones estáticas.

---

## Diseño del Dockerfile de Backend: Caché de Capas, Permisos y Autodiagnóstico

Mientras la base de datos utiliza la imagen oficial `postgres:16-alpine`, un archivo `Dockerfile` propio define el entorno de ejecución del backend en FastAPI:

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

Tres decisiones técnicas en este archivo resultan clave en producción:
* **Aprovechamiento del Caché de Capas (Layer Caching):** Docker guarda cada instrucción como una capa reutilizable. Al ubicar `COPY requirements.txt .` y `pip install` antes de copiar el código fuente (`COPY src/ ./src`), Docker evita reinstalar los paquetes de Python cuando solo cambia la lógica del programa, reduciendo el tiempo de compilación de minutos a segundos.
* **Ejecución sin Privilegios de Administrador (`USER appuser`):** Por defecto, los procesos en Docker corren como `root`. El cambio al usuario restringido `appuser` impide que el proceso de Python modifique archivos del sistema dentro del contenedor.
* **Verificación Continua de Salud (`HEALTHCHECK`):** Que un proceso figure encendido no garantiza que responda peticiones. Cada 15 segundos, Docker consulta internamente la ruta `/health` y marca el contenedor como `unhealthy` si deja de responder.

---

## Arranque Coordinado y Volúmenes Persistentes en Docker Compose

Cuando un servidor se reinicia tras un corte eléctrico o mantenimiento, todos los contenedores arrancan al mismo tiempo. Como PostgreSQL necesita unos segundos para verificar sus registros de transacciones mientras Python inicia casi al instante, un backend sin coordinación intentaría conectarse prematuramente y caería en un bucle de reinicios.

En `docker-compose.yml`, una condición de salud explícita (`condition: service_healthy`) ordena la secuencia de encendido:

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
      ALLOWED_EMAIL_DOMAIN: example-campus.edu
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

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

Docker espera hasta que la herramienta `pg_isready` confirma que PostgreSQL acepta conexiones SQL antes de iniciar el contenedor del backend.

### Contenedores Efímeros frente a Volúmenes Persistentes

Al desplegar una nueva versión del software (`docker compose up -d --build`), Docker elimina el contenedor anterior y crea uno nuevo a partir de la imagen actualizada. Todo archivo guardado en la capa interna de escritura del contenedor (`OverlayFS`) desaparece en ese reemplazo:

```mermaid
flowchart TD
    subgraph Efimero["Tiempo de Ejecución Efímero (Se Reemplaza en Cada Actualización)"]
        Code["Imagen de Solo Lectura (Python, FastAPI, Nginx)"]
        Overlay["Capa Temporal del Contenedor (OverlayFS)"]
    end
    subgraph Persistente["Almacenamiento Persistente en el Host (Sobrevive a las Actualizaciones)"]
        VolDB[("Volumen: postgres_data\nTablas, Índices y Logs Transaccionales")]
        VolMedia[("Volumen: uploads_data\nAfiches, Videos MP4/WEBM y PDFs")]
    end
    Efimero -->|Montado en /var/lib/postgresql/data| VolDB
    Efimero -->|Montado en /app/uploads| VolMedia
```

Para conservar tanto las cuentas de estudiantes como el material multimedia subido por administración, la arquitectura conecta dos **Volúmenes de Docker** alojados en el disco físico del servidor:
1. **`postgres_data`:** Resguarda todas las tablas relacionales e índices fuera del ciclo de vida del contenedor.
2. **`uploads_data`:** Conserva en `/app/uploads` las imágenes, videos y documentos PDF cargados desde el panel editorial.

---

## Conclusiones

1. **Entornos Reproducibles mediante Contenedores Desacoplados:** Separar frontend, backend y base de datos permite actualizar la interfaz sin interrumpir el servicio API y asegura el mismo comportamiento en cualquier computadora del equipo.
2. **Secuencia de Arranque Determinista y Datos Seguros:** Las pruebas `pg_isready` evitan fallos de conexión al iniciar el servidor, mientras que los volúmenes `postgres_data` y `uploads_data` permiten reconstruir contenedores en cualquier momento sin perder información.
