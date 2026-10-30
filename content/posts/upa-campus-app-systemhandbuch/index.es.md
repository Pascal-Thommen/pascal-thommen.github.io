---
title: "UPA Campus App: Manual Completo del Sistema y Libro de Arquitectura"
date: 2026-10-30T17:00:00Z
description: "Manual completo del sistema y libro de arquitectura de UPA Campus App: Contenedores Docker, Ingress Zero-Trust con NetBird, backend FastAPI y persistencia PostgreSQL."
summary: "La guía arquitectónica integral de UPA Campus App: mapeo de especificaciones funcionales, modelos de roles, arquitectura de contenedores y flujos de datos de extremo a extremo."
tags: ["Arquitectura de Software", "Docker", "FastAPI", "PostgreSQL", "NetBird", "Enterprise Systems"]
categories: ["Arquitectura", "Enterprise Systems"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowToc: true
TocOpen: true
mermaid: true
draft: false
aliases:
  - "/posts/upa-campus-app-architecture-part-1/"
  - "/posts/upa-campus-app-architecture-part-2/"
  - "/posts/upa-campus-app-architecture-part-3/"
  - "/posts/upa-campus-app-architecture-part-4/"
  - "/posts/upa-campus-app-architecture-part-5/"
  - "/posts/upa-campus-app-architecture-part-6/"
---

La UPA Campus App consolida avisos del campus, fechas académicas e inscripciones a eventos dentro de una infraestructura modular de tres contenedores. Este manual maestro sirve a estudiantes e ingenieros como referencia central de arquitectura: vincula las especificaciones funcionales con los módulos de código y documenta el flujo de datos integral desde la interfaz web hasta la persistencia relacional.

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#)
[![NetBird](https://img.shields.io/badge/NetBird-WireGuard%20Ingress-1A202C?style=flat-square&logo=wireguard&logoColor=white)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)](#)
[![Python](https://img.shields.io/badge/Python%203.11-3776AB?style=flat-square&logo=python&logoColor=white)](#)

---

<details open>
<summary><strong>Índice Interactivo</strong></summary>

1. [Parte 1: Sección Obligatoria para Compañeros de Estudio](#parte-1-sección-obligatoria-para-compañeros-de-estudio)
   - [1.1 Visión General y Sistema Integral](#11-visión-general-y-sistema-integral)
   - [1.2 Mapeo del Pliego de Requisitos: Funcionalidades y Código](#12-mapeo-del-pliego-de-requisitos-funcionalidades-y-código)
   - [1.3 Roles y Permisos: Principio de Privilegio Mínimo](#13-roles-y-permisos-principio-de-privilegio-mínimo)
   - [1.4 El Recorrido en Vivo: Desde el Clic hasta el Commit en PostgreSQL](#14-el-recorrido-en-vivo-desde-el-clic-hasta-el-commit-en-postgresql)
2. [Parte 2: Profundización Técnica Voluntaria](#parte-2-profundización-técnica-voluntaria)
   - [2.1 Volúmenes Host de Docker vs Almacenamiento Efímero](#21-volúmenes-host-de-docker-vs-almacenamiento-efímero)
   - [2.2 Detalles de Ingress WireGuard con NetBird](#22-detalles-de-ingress-wireguard-con-netbird)
   - [2.3 Validación de Esquemas con Pydantic y Migraciones con Alembic](#23-validación-de-esquemas-con-pydantic-y-migraciones-con-alembic)
3. [Conclusión](#conclusión)

</details>

---

## Parte 1: Sección Obligatoria para Compañeros de Estudio

### 1.1 Visión General y Sistema Integral

La arquitectura de la UPA Campus App desacopla la presentación visual, la lógica de negocio y el almacenamiento de datos en tres contenedores ligeros. El acceso externo se realiza sin abrir puertos en el router mediante una malla privada de WireGuard (NetBird Ingress):

```mermaid
flowchart TD
    Client["Cliente: Navegador / PWA / Android"] -->|Túnel cifrado WireGuard| NetBird["Capa NetBird Ingress"]
    NetBird -->|Puerto 8085| Nginx["Contenedor 1: upa-frontend (Nginx :80)"]
    
    subgraph Docker_Bridge["Red Aislada Docker (upa-network)"]
        Nginx -->|Archivos estáticos PWA y Media| Client
        Nginx -->|Reverse Proxy /api/ -> Puerto 8000| FastAPI["Contenedor 2: upa-backend (FastAPI / Uvicorn :8000)"]
        FastAPI -->|SQL TCP Puerto 5432| Postgres["Contenedor 3: upa-postgres (PostgreSQL 16 :5432)"]
    end
    
    subgraph Storage_Volumes["Volúmenes Persistentes de Docker"]
        Postgres -->|Tablas relacionales ACID| VolDB[("Volumen: postgres_data")]
        FastAPI -->|Subida de imágenes y medios| VolMedia[("Volumen: uploads_data")]
        Nginx -.->|Lectura directa /uploads/| VolMedia
    end
```

Distribución operativa de los tres contenedores:

1. **Contenedor Frontend (`upa-frontend`):** Entrega la Single Page Application precompilada (Vue.js / Vite) mediante Nginx en el puerto 80. Nginx actúa como reverse proxy: las solicitudes bajo la ruta `/api/` se reenvían al contenedor backend. Los archivos subidos en `/uploads/` se sirven directamente desde el volumen compartido sin sobrecargar el proceso de Python.
2. **Contenedor Backend (`upa-backend`):** API REST implementada con FastAPI y Python 3.11 en el puerto 8000. Gestiona la lógica de negocio, valida datos de entrada con Pydantic, verifica tokens de Google OAuth restringidos al dominio `@upa.edu.py` y coordina tareas en segundo plano (notificaciones push, sincronización).
3. **Contenedor de Base de Datos (`upa-postgres`):** Núcleo relacional con PostgreSQL 16 Alpine en el puerto 5432. Los datos estructurados residen en el volumen nombrado `postgres_data`. El contenedor opera de forma aislada dentro del bridge interno `upa-network`.

---

### 1.2 Mapeo del Pliego de Requisitos: Funcionalidades y Código

Los requerimientos del pliego funcional universitario se corresponden directamente con los módulos de código del backend:

| Requisito Funcional | Función Didáctica Principal | Módulo de Código / Endpoint | Modelo de Datos y Tablas |
| :--- | :--- | :--- | :--- |
| **Aviso de Emergencia** | Notificación destacada superior ante imprevistos climáticos o suspensiones | `src/api/publications.py` | `Post.is_emergency`, `Post.emergency_level` |
| **Fijar Publicaciones** | Fijación prioritaria de eventos clave con fecha de vigencia obligatoria | `src/services/scheduler.py`, `src/api/publications.py` | `Post.pinned`, `Post.pinned_until` |
| **Sincronización Instagram** | Importación automática de publicaciones con bloqueo de edición en campos sincronizados | `src/services/instagram.py`, `src/api/publications.py` | `Post.instagram_id`, `Post.media_type` |
| **Inscripción en 1 Clic** | Registro y cancelación instantánea de cupos para conferencias sin formularios repetitivos | `src/api/events.py` (`/api/events/{id}/register`) | `EventRegistration` (`post_id`, `user_id`) |
| **Fechas Académicas** | Sincronización de entregas y plazos formativos desde Google Classroom | `src/api/classroom.py`, `src/schemas/classroom.py` | Caché local de Classroom |
| **Control de Roles** | Asignación granular basada en el principio de menor privilegio | `src/permissions.py`, `src/api/roles.py` | `UserRole` (array JSON `permissions`) |
| **Segmentación de Audiencias** | Directorio ordenado alfabéticamente de grupos de Google Workspace con filtro en tiempo real | `src/services/google_groups.py`, `src/api/groups.py` | `AudienceGroup`, caché `@upa.edu.py` |
| **Fuente Única de Verdad** | Esquema unificado para noticias, fechas de calendario y publicaciones multimedia | `src/api/publications.py` | `Post` (Modelo polimórfico) |
| **Registro de Auditoría** | Trazabilidad completa de modificaciones administrativas y editoriales | `src/services/audit.py`, `src/api/history.py` | `AdminAuditLog` (`action`, `payload`) |
| **Notificaciones Push** | Recordatorios programados con anticipación (minutos, horas, días antes del evento) | `src/services/push.py`, `src/services/scheduler.py` | `DeviceToken` (Tokens FCM, OS) |

---

### 1.3 Roles y Permisos: Principio de Privilegio Mínimo

El modelo de seguridad aplica el principio de menor privilegio. Al registrar una nueva cuenta en el panel administrativo, todas las casillas de verificación de permisos comienzan desmarcadas.

```mermaid
flowchart TD
    subgraph ModeloRoles["Modelo de Accesos UPA Campus App"]
        direction TB
        Admin["Administrador (Control Total)"]
        Editor["Editor / Redactor (Permisos Modulares)"]
        Student["Estudiante (PWA y App Móvil)"]
    end

    subgraph Modulos["Módulos de Permisos Canónicos"]
        M1["publicaciones: Crear y editar noticias"]
        M2["grupos: Consultar audiencias y listas"]
        M3["roles: Administrar permisos de usuarios"]
        M4["apis: Diagnóstico y estado del sistema"]
        M5["historial: Auditoría de cambios"]
    end

    Admin --> M1 & M2 & M3 & M4 & M5
    Editor --> M1
    Editor -.->|Solo con concesión explícita| M2 & M3 & M4 & M5
    Student --> ReadOnly["Acceso de solo lectura a publicaciones vía OAuth"]
```

Los tres niveles de acceso:

1. **Estudiante:** Acceso validado mediante Google OAuth con dirección de correo `@upa.edu.py`. Posee permisos de solo lectura sobre publicaciones dirigidas a su carrera, puede inscribirse o cancelar su asistencia a eventos con un clic y gestionar sus preferencias personales.
2. **Editor:** Personal encargado de la redacción. Por defecto, solo cuenta con el permiso `publicaciones`. Las secciones adicionales (`grupos`, `roles`, `apis`, `historial`) permanecen bloqueadas hasta que un administrador las habilita individualmente.
3. **Administrador:** Acceso irrestricto. Es el único perfil facultado para crear cuentas de usuario, configurar permisos de roles, ejecutar diagnósticos de APIs e inspeccionar el historial de auditoría.

El backend aplica estas restricciones mediante inyección de dependencias en FastAPI:

```python
# src/permissions.py
PERMISSION_KEYS: tuple[str, ...] = (
    "publicaciones", "grupos", "roles", "apis", "historial"
)

def require_permission(permission: str):
    def dependency(
        user: User = Depends(get_current_user),
        db: Session = Depends(get_db)
    ):
        if user.role == "admin":
            return user
        perms = effective_permissions(db, user)
        if permission not in perms:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permiso requerido no asignado: {permission}"
            )
        return user
    return dependency
```

---

### 1.4 El Recorrido en Vivo: Desde el Clic hasta el Commit en PostgreSQL

Proceso técnico que se desencadena cuando un estudiante pulsa "Inscribirme al Evento" en la PWA:

```mermaid
sequenceDiagram
    autonumber
    actor Student as Estudiante (Navegador PWA)
    participant NetBird as NetBird Ingress (:8085)
    participant Nginx as Nginx Proxy (:80)
    participant FastAPI as FastAPI Backend (:8000)
    participant Postgres as PostgreSQL DB (:5432)

    Student->>NetBird: POST /api/events/42/register (Bearer JWT)
    NetBird->>Nginx: Reenvío mediante túnel WireGuard
    Nginx->>FastAPI: Proxy pass http://backend:8000/api/events/42/register
    FastAPI->>FastAPI: Validar firma del token y dominio @upa.edu.py
    FastAPI->>Postgres: SELECT * FROM event_registrations WHERE post_id=42 AND user_id=7
    Postgres-->>FastAPI: 0 registros (Sin inscripción previa)
    FastAPI->>Postgres: INSERT INTO event_registrations (post_id, user_id, registered_at)
    Postgres-->>FastAPI: Commit exitoso (ID generado: 301)
    FastAPI-->>Nginx: HTTP 201 Created (Esquema: RegistrationOut)
    Nginx-->>NetBird: Retorno de carga útil HTTP 201
    NetBird-->>Student: Interfaz web cambia estado a 'Inscrito'
```

Etapas de la ejecución:

1. **Cliente e Ingress:** La aplicación web emite una petición HTTPS incluyendo el token JWT de Google OAuth en los encabezados. La solicitud viaja por el overlay de WireGuard hasta el puerto 8085 del host.
2. **Enrutamiento por Reverse Proxy:** Nginx detecta el prefijo `/api/` y canaliza la conexión dentro de la red Docker hacia `http://backend:8000`.
3. **Validación y Autenticación:** El middleware de FastAPI analiza el JWT, verifica la firma criptográfica contra los servidores de Google y comprueba que el correo pertenezca al dominio `@upa.edu.py`.
4. **Lógica de Dominio:** El servicio comprueba la disponibilidad de plazas y certifica que el usuario no cuente con una inscripción previa activa.
5. **Persistencia Transaccional:** SQLAlchemy ejecuta la instrucción `INSERT`. La restricción de unicidad sobre las columnas `(post_id, user_id)` previene registros duplicados en condiciones de concurrencia.
6. **Confirmación al Cliente:** Tras el `db.commit()`, FastAPI retorna el código HTTP 201 con el esquema Pydantic correspondiente. Nginx retorna la respuesta a través del túnel y la interfaz actualiza el botón de inmediato.

---

## Parte 2: Profundización Técnica Voluntaria

### 2.1 Volúmenes Host de Docker vs Almacenamiento Efímero

Los contenedores ejecutan sus procesos sobre un sistema de archivos en capas (OverlayFS). Cualquier modificación en la capa de escritura del contenedor añade sobrecarga de copia en escritura y se destruye si el contenedor se recrea.

El entorno de producción separa el almacenamiento persistente del código:

```mermaid
flowchart LR
    subgraph Ephemeral["Runtime del Contenedor (OverlayFS: Efímero)"]
        Binaries["Binarios Python FastAPI"]
        NginxBin["Configuración de Nginx"]
        OSLibs["Librerías del Sistema Alpine"]
    end

    subgraph Persistent["Volúmenes Persistentes de Docker (En el Host)"]
        V1[("postgres_data: /var/lib/postgresql/data")]
        V2[("uploads_data: /app/uploads")]
    end

    FastAPIApp["Contenedor upa-backend"] -->|Ejecución sin escritura en código| Binaries
    FastAPIApp -->|Registros relacionales transaccionales| V1
    FastAPIApp -->|Guardado de imágenes y documentos| V2
    NginxApp["Contenedor upa-frontend"] -.->|Lectura directa de alto rendimiento| V2
```

Se definen dos volúmenes específicos:

* `postgres_data`: Asocia `/var/lib/postgresql/data` al almacenamiento del host. Al actualizar la imagen de la base de datos o reiniciar el servicio, los datos no sufren pérdidas.
* `uploads_data`: Almacena archivos binarios como afiches de actividades o manuales PDF. El backend tiene permisos de escritura y Nginx acceso de lectura directa para distribuir `/uploads/`.

---

### 2.2 Detalles de Ingress WireGuard con NetBird

Las implementaciones habituales exigen abrir puertos en el enrutador y poseer una dirección IP pública fija. En entornos académicos protegidos, los cortafuegos bloquean puertos entrantes y emplean NAT simétrico.

NetBird resuelve este escenario construyendo una red mallada de WireGuard gestionada por software:

```mermaid
flowchart TD
    subgraph External["Red Externa"]
        ClientDevice["Dispositivo de Estudiante o Desarrollador"]
    end

    subgraph Coordination["Plano de Coordinación de NetBird"]
        STUN["Servidores STUN / TURN (NAT Traversal)"]
    end

    subgraph Campus_Server["Servidor del Campus (Sin puertos de entrada abiertos)"]
        NetBirdDaemon["NetBird Daemon (Punto final WireGuard)"]
        NginxService["Frontend Nginx (:8085)"]
    end

    ClientDevice -->|1. Descubrimiento y autenticación| Coordination
    NetBirdDaemon -->|2. Conexión saliente y UDP hole punching| Coordination
    ClientDevice <==|3. Túnel directo cifrado WireGuard peer-to-peer|==> NetBirdDaemon
    NetBirdDaemon -->|4. Reenvío a puerto local 127.0.0.1:8085| NginxService
```

Principios técnicos de operación:

1. **Conexiones Salientes:** El servicio NetBird en el servidor inicia comunicaciones hacia afuera hacia los servidores de control. Los cortafuegos del campus permiten el tráfico saliente por defecto.
2. **ICE y UDP Hole Punching:** Los clientes descubren sus trayectorias mediante STUN y TURN para negociar enlaces directos punto a punto sin requerir apertura de puertos.
3. **Cifrado de Nivel de Kernel:** Las tramas se aseguran con ChaCha20-Poly1305. Ningún tercero conectado a la red local puede interceptar o alterar el tráfico.

---

### 2.3 Validación de Esquemas con Pydantic y Migraciones con Alembic

La integridad operativa requiere validación estricta de datos y evolución controlada del modelo de datos.

#### Validación de Esquemas con Pydantic

FastAPI utiliza modelos de Pydantic para la serialización y la entrada de datos. Cualquier parámetro fuera de formato produce una respuesta HTTP 422 Unprocessable Entity antes de tocar la base de datos:

```python
# src/schemas/publication.py
from datetime import datetime
from pydantic import BaseModel, Field

class PublicationCreate(BaseModel):
    title: str = Field(..., min_length=3, max_length=255)
    content: str = Field(..., min_length=1)
    is_emergency: bool = False
    pinned: bool = False
    pinned_until: datetime | None = None
    media_url: str | None = None

class PublicationOut(PublicationCreate):
    id: int
    created_at: datetime
    views_count: int = 0

    class Config:
        from_attributes = True
```

#### Migraciones con Alembic

Las estructuras de tablas evolucionan a lo largo del tiempo. Ejecutar sentencias `ALTER TABLE` manuales genera inconsistencias entre entornos. Alembic versiona estos cambios mediante archivos de migración:

```mermaid
flowchart LR
    A["001_initial_schema.py"] -->|alembic upgrade| B["002_add_emergency_banner.py"]
    B -->|alembic upgrade| C["003_add_instagram_sync_fields.py"]
    C -->|alembic upgrade| D["004_add_event_registrations.py"]
```

Al inicializarse el contenedor, el sistema ejecuta de forma automática `alembic upgrade head`, sincronizando el esquema con la versión esperada.

---

## Conclusión

Dos principios definen la arquitectura de la UPA Campus App:

1. **La simplicidad modular supera la complejidad:** Separar tareas entre Nginx, FastAPI y PostgreSQL proporciona alta resiliencia y mínimo esfuerzo operativo.
2. **Seguridad integrada en el diseño:** El ingreso mediante NetBird y la división rigurosa de roles resguardan la infraestructura universitaria frente a fallas o accesos no autorizados.
