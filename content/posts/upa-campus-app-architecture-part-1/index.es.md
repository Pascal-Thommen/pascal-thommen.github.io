---
title: "UPA Campus App: Arquitectura e Infraestructura de Contenedores"
date: 2026-10-30T16:00:00Z
description: "Arquitectura de producción de la UPA Campus App: Del túnel de ingreso NetBird pasando por Nginx y FastAPI hasta la persistencia en PostgreSQL."
summary: "¿Cómo se estructura una plataforma de campus moderna? La parte 1 analiza la interacción entre la PWA, el proxy inverso Nginx, el backend FastAPI y los volúmenes de PostgreSQL."
tags: ["Arquitectura de Software", "Docker", "FastAPI", "PostgreSQL", "Sistemas Empresariales"]
categories: ["Arquitectura", "Sistemas Empresariales"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
draft: false
---

¿Cómo construir una plataforma de campus confiable para estudiantes y docentes con una infraestructura ágil? La UPA Campus App separa la interfaz de usuario, la lógica de la API y la persistencia de datos en tres contenedores Docker independientes, protegidos mediante un túnel de ingreso cifrado con NetBird. Esta guía detalla la arquitectura global, la interacción entre contenedores y la estrategia de almacenamiento de datos relacionales y archivos multimedia.

---

## Visión General de la Arquitectura

El flujo de información avanza de forma directa desde el cliente a través de una red privada mallada hasta el entorno de contenedores:

```mermaid
flowchart LR
    Client["Cliente (PWA / Navegador)"] -->|HTTPS 443| NetBird["Túnel Ingress NetBird"]
    NetBird -->|Puerto 8085| Nginx["Contenedor Frontend (Nginx / Web-Admin)"]
    Nginx -->|Recursos Estáticos| Client
    Nginx -->|Proxy Inverso /api/ Puerto 8000| Backend["Contenedor Backend (FastAPI)"]
    Backend -->|SQL Puerto 5432| DB[("Contenedor PostgreSQL")]
    DB -->|Tablas Persistentes| VolDB[("Volumen: postgres_data")]
    Backend -->|Subidas / Multimedia| VolMedia[("Volumen: uploads_data")]
    Nginx -->|Proxy Inverso /uploads/| VolMedia
```

---

## Roles y Responsabilidades de los Componentes

Cada capa del sistema resuelve una tarea técnica delimitada:

1. **Túnel Ingress NetBird:** Establece una red mallada WireGuard con cifrado de extremo a extremo. El servidor no expone puertos públicos a Internet, dado que todo el tráfico transita exclusivamente por túneles autorizados punto a punto.
2. **Contenedor Frontend (Nginx):** Entrega la aplicación web administrativa (PWA) como archivos HTML, CSS y JavaScript precompilados. Además, Nginx opera como proxy inverso: canaliza solicitudes dirigidas a `/api/` hacia el backend y despacha imágenes desde `/uploads/`. El cliente interactúa con un único dominio unificado.
3. **Contenedor Backend (FastAPI):** Procesa peticiones REST, valida esquemas con Pydantic y ejecuta la lógica de negocio. Gestiona la autenticación mediante Google OAuth 2.0 para cuentas institucionales (`@example-campus.edu`), la verificación de permisos y la sincronización de actividades académicas.
4. **Contenedor de Base de Datos (PostgreSQL):** Almacena registros relacionales como perfiles, roles, horarios y avisos con garantías transaccionales ACID completas sobre una imagen base Alpine Linux.

---

## Conceptos Fundamentales de Infraestructura

<details open>
<summary>Glosario de conceptos técnicos</summary>

* **Contenedor:** Unidad de ejecución aislada que empaqueta código y dependencias. Al compartir el kernel del sistema operativo anfitrión, los contenedores arrancan en fracciones de segundo y consumen una fracción de los recursos de una máquina virtual.
* **Puerto:** Punto de conexión lógico para enrutar el tráfico de red. La redirección de puertos mapea un puerto externo del host (por ejemplo, 8085) hacia un puerto interno del contenedor (por ejemplo, 80).
* **Proxy Inverso:** Servidor intermedio que recibe tráfico externo y lo distribuye hacia servicios internos. Oculta la topología de la red, gestiona certificados SSL y centraliza las rutas bajo un origen común.
* **Volumen:** Espacio de almacenamiento gestionado por el host fuera del sistema de archivos volátil del contenedor, garantizando la preservación de los datos entre reinicios y actualizaciones.

</details>

---

## Infraestructura de Contenedores en Detalle

¿Por qué utilizar Docker para la UPA Campus App? En entornos universitarios, los desarrolladores utilizan diversos sistemas operativos (Linux, macOS, Windows). Sin aislamiento de contenedores surgen discrepancias de dependencias, versiones incompatibles de Python y desalineación en configuraciones de bases de datos. Docker asegura paridad absoluta entre el entorno local y el servidor institucional.

Los tres servicios operan integrados en una red puente (bridge) aislada:

```yaml
# docker-compose.yml (extracto simplificado con marcadores neutrales)
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
    networks:
      - campus-network

  backend:
    build: ./backend
    container_name: campus-backend
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DATABASE_URL: postgresql://db_user:secret_password@postgres:5432/app_db
      APP_ENV: production
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
      - "8085:80"
    networks:
      - campus-network

networks:
  campus-network:
    driver: bridge

volumes:
  postgres_data:
  uploads_data:
```

La comunicación interna utiliza resolución DNS nativa de Docker (`postgres:5432`, `backend:8000`). Ni la base de datos ni FastAPI exponen sus puertos al exterior, manteniendo blindado el perímetro de red.

---

## Estrategia de Persistencia de Datos

Los contenedores son efímeros por diseño. Al recrear una imagen, cualquier dato guardado en su capa interna se elimina. Un entorno de producción demanda una separación rigurosa:

1. **Datos Estructurados mediante Volúmenes de Base de Datos (`postgres_data`):** PostgreSQL escribe tablas, índices y registros en `/var/lib/postgresql/data`. Al montar este directorio en un volumen con nombre de Docker, la base de datos preserva su contenido durante migraciones o actualizaciones de versión.
2. **Archivos Multimedia en Volúmenes Dedicados (`uploads_data`):** Las imágenes y afiches de publicaciones se depositan en `/app/uploads`. Evitar el almacenamiento binario (BLOBs) dentro de PostgreSQL previene la sobrecarga de copias de seguridad y mantiene óptimo el uso de memoria caché. PostgreSQL guarda únicamente la ruta relativa, mientras que Nginx despacha los archivos estáticos de forma eficiente.

---

## Conclusiones Clave

1. **Desacoplamiento Estricto:** Separar proxy frontend, API y base de datos previene dependencias monolíticas y permite despliegues modulares sin interrupciones.
2. **Cómputo Efímero y Volúmenes Persistentes:** Los contenedores pueden destruirse y reemplazarse en cualquier momento sin comprometer los datos relacionales ni las subidas de archivos.
3. **Mínima Superficie de Exposición:** La comunicación interna transcurre en la red puente de Docker, mientras que NetBird protege el acceso sin puertos públicos abiertos.
