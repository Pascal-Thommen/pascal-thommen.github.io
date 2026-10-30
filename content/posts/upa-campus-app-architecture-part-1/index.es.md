---
title: "UPA Campus App (Capítulo 1): Panorama General y Arquitectura del Sistema"
date: 2026-10-30T16:00:00Z
description: "¿Cómo funciona una aplicación web moderna desde cero? El capítulo 1 explica la arquitectura en tres capas de la UPA Campus App y muestra cómo la especificación funcional nace directamente del diseño del sistema."
summary: "¿Cómo construir una plataforma universitaria en producción con infraestructura ligera? El capítulo 1 conecta la arquitectura universal en tres capas con los módulos concretos de la especificación funcional."
tags: ["Arquitectura de Software", "Docker", "FastAPI", "PostgreSQL", "Sistemas Empresariales"]
categories: ["Arquitectura", "Sistemas Empresariales"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
mermaid: true
draft: false
series: "upa-campus-app"
series_part: 1
series_part_title: "Panorama General y Arquitectura del Sistema"
hiddenInList: true
---

Toda aplicación web sólida divide la presentación visual, la lógica de negocio y la persistencia de datos en capas independientes, porque un único proceso monolítico se vuelve imposible de asegurar y mantener a medida que crece el sistema. La UPA Campus App materializa este principio mediante tres contenedores Docker aislados detrás de un túnel de ingreso cifrado con NetBird. Este primer capítulo explica cómo funciona una arquitectura web moderna desde sus cimientos y por qué cada módulo de la especificación funcional se desprende directamente de esta división técnica del trabajo.

---

## La Anatomía de una Aplicación Web Moderna

Cuando un estudiante revisa su teléfono por la mañana para comprobar si una tormenta suspendió las clases presenciales o cuándo vence la próxima entrega en Google Classroom, la aplicación se percibe como un solo bloque. Sin embargo, detrás de cada toque ocurre un relevo coordinado entre cuatro componentes especializados:

```mermaid
flowchart LR
    Client["Cliente (App Móvil / Web-Admin)"] -->|HTTPS 443| NetBird["Túnel Ingress NetBird"]
    NetBird -->|127.0.0.1:8085| Nginx["Contenedor Frontend (Nginx)"]
    Nginx -->|Recursos Estáticos y App Shell| Client
    Nginx -->|Proxy Inverso /api/ Puerto 8000| Backend["Contenedor Backend (FastAPI)"]
    Backend -->|SQL Puerto 5432| DB[("Contenedor PostgreSQL")]
    DB -->|Tablas Relacionales| VolDB[("Volumen: postgres_data")]
    Backend -->|Archivos Multimedia| VolMedia[("Volumen: uploads_data")]
    Nginx -->|Proxy Inverso /uploads/| VolMedia
```

Para visualizar el flujo, resulta útil la analogía de un restaurante universitario bien organizado:
* **Frontend (Nginx y Navegador / App):** El salón comedor y el menú. Aquí el estudiante consulta las opciones y solicita lo que necesita, pero en las mesas no se cocina ni se guarda la contabilidad.
* **Proxy Inverso (Enrutamiento Nginx):** El recepcionista en la puerta principal. Recibe cada solicitud, entrega folletos impresos (archivos estáticos e imágenes) al instante y dirige los pedidos de cocina hacia el interior.
* **Backend (FastAPI):** La cocina. Verifica que la persona tenga permiso para pedir (autenticación y control de roles), comprueba que la comanda esté completa (validación de esquemas) y prepara el plato aplicando las recetas oficiales (lógica de negocio).
* **Base de Datos (PostgreSQL y Volúmenes):** La cámara frigorífica y el libro contable. Aquí se resguardan todos los insumos y registros de forma duradera y cerrada al público. Solo la cocina tiene llave para entrar.

---

## Por Qué Cada Capa Cumple una Función Exclusiva

1. **Túnel Ingress NetBird:** Crea una red mallada WireGuard cifrada de extremo a extremo. El servidor no abre puertos públicos hacia Internet porque todo el tráfico ingresa únicamente por túneles autenticados punto a punto hacia la interfaz local loopback (`127.0.0.1:8085`).
2. **Contenedor Frontend (Nginx):** Sirve tanto la pantalla de escritorio minimalista (con un único código QR de instalación móvil y capturas de vista previa) como el portal administrativo Web-Admin para marketing y la PWA móvil. Al mismo tiempo, Nginx actúa como proxy inverso: redirige las llamadas `/api/` hacia el backend y las rutas `/uploads/` hacia el volumen de archivos multimedia.
3. **Contenedor Backend (FastAPI):** Procesa peticiones REST de forma asíncrona, valida cada entrada con Pydantic y ejecuta las reglas de la especificación funcional. Esto abarca el acceso obligatorio con Google OAuth para cuentas institucionales (`@example-campus.edu`), la sincronización con Google Classroom API y Google Workspace Directory, y la importación automática del feed oficial de Instagram.
4. **Contenedor de Base de Datos (PostgreSQL):** Almacena registros relacionales como perfiles de usuario, grupos de audiencia, publicaciones unificadas, inscripciones en 1 clic (`EventRegistration`), lecturas únicas (`PostView`) y la bitácora inmutable de auditoría (`AdminAuditLog`) con garantías transaccionales ACID completas.

---

## Del Diagrama de Arquitectura a la Especificación Funcional: El Modelo Mental Universal

En la ingeniería en informática empresarial y en el desarrollo moderno asistido por inteligencia artificial (vibe-coding), el verdadero valor no reside en memorizar la sintaxis de un bucle o márgenes CSS, sino en saber traducir cada requisito de negocio del pliego funcional en cuatro bloques arquitectónicos:
1. **Frontera del Sistema e Ingreso (Ingress):** ¿Quién puede acceder a la plataforma y a través de qué canal de red cifrado?
2. **Estado del Cliente y Presentación:** ¿Qué datos conserva el dispositivo localmente para responder sin conexión y qué acciones exigen sincronización con el servidor?
3. **Contrato de Interfaz y Lógica de Dominio:** ¿Qué reglas valida obligatoriamente la API antes de modificar datos o consultar sistemas externos (como Google Classroom o Instagram)?
4. **Persistencia Relacional:** ¿En qué tablas y bajo qué restricciones de integridad (como bloqueos contra inscripciones duplicadas) se resguarda la información de forma duradera?

Al aplicar este modelo mental a la UPA Campus App, se comprende de inmediato por qué los cuatro módulos principales de la especificación funcional fueron diseñados exactamente de esa manera:

```mermaid
flowchart TD
    subgraph Spec["Módulos de la Especificación Funcional"]
        M1["Módulo Inicio (Feed): Avisos de Emergencia, Noticias, RSVP en 1 Clic e Instagram"]
        M2["Módulo Agenda: Lista Cronológica, Calendario Mensual y Classroom"]
        M3["Módulo Perfil: Identidad Universitaria (@example-campus.edu) y Almacenamiento Offline"]
        M4["Módulo Web-Admin: Editor Unificado, Grupos Alfabéticos y Matriz de Roles"]
    end
    subgraph Arch["Implementación Arquitectónica en las Capas"]
        A1["Modelo Post Unificado en PostgreSQL + Bloqueo de Solo Lectura para Sync de Instagram"]
        A2["Fuente Dual: Eventos Internos en PostgreSQL + Entregas en Vivo vía Google Classroom OAuth"]
        A3["Validación de Dominio Google OAuth 2.0 en Backend + Cache Storage / Room en Cliente"]
        A4["Caché de Google Directory API en Backend + Roles bajo Privilegio Mínimo sin Preselección"]
    end
    M1 --> A1
    M2 --> A2
    M3 --> A3
    M4 --> A4
```

* **¿Por qué existe un modelo unificado de publicaciones en lugar de tablas separadas para noticias y eventos?** En la vida universitaria, un mismo anuncio suele ser simultáneamente una noticia en el feed y un evento en el calendario (por ejemplo, una conferencia magistral con texto explicativo, afiche, auditorio, horario e inscripción RSVP en 1 clic). Como el backend alimenta ambas vistas desde la misma tabla relacional (`posts` mediante `/api/publications`), existe una verdadera *Fuente Única de Verdad (Single Source of Truth)*: si marketing cambia el aula de una charla, la tarjeta del feed, el modal a pantalla completa y el punto en el calendario mensual se actualizan en el mismo instante.
* **¿Por qué el feed distingue entre modo público y modo autenticado?** Antes de iniciar sesión, el endpoint `/api/v1/feed` devuelve exclusivamente las publicaciones públicas de Instagram de la universidad. Solo cuando el estudiante envía un token JWT válido tras autenticarse con Google OAuth, el backend entrega noticias internas, eventos segmentados por carrera y banners superiores de emergencia (`EMERGENCY_BANNER`). Así, la pantalla inicial resulta atractiva sin exponer información institucional privada.
* **¿Por qué el módulo Agenda se conecta directamente a Google Classroom mediante tokens OAuth?** Los docentes ya cargan las fechas límite de tareas y entregas en Google Classroom. Obligar a los profesores a duplicar esos plazos manualmente en el portal del campus provocaría datos desactualizados en pocas semanas. Por ello, el backend resguarda el token OAuth de Classroom al iniciar sesión y combina en tiempo real las entregas académicas oficiales con los eventos propios de la universidad.

---

## Conceptos Fundamentales de Infraestructura

<details open>
<summary>Glosario de conceptos técnicos</summary>

* **Contenedor:** Unidad de ejecución aislada que empaqueta el código de la aplicación junto con todas sus dependencias de sistema. Al compartir el kernel Linux del servidor anfitrión, los contenedores inician en fracciones de segundo y consumen mucha menos memoria RAM que las máquinas virtuales.
* **Puerto:** Canal numérico lógico que identifica un servicio de red específico (como el puerto 5432 para PostgreSQL o el 8000 para FastAPI). En una arquitectura segura, los puertos internos solo son accesibles dentro de la red privada de contenedores.
* **Proxy Inverso:** Servidor web perimetral (Nginx) que recibe todas las solicitudes entrantes de los clientes y las distribuye según el prefijo de la ruta (`/api/`, `/uploads/`, `/dashboard/`) hacia el contenedor interno correspondiente.
* **Volumen:** Directorio persistente en el disco del servidor anfitrión conectado al contenedor para garantizar que las tablas de la base de datos y los archivos subidos sobrevivan a cualquier reinicio o actualización de versión.

</details>

---

## Orquestación de Contenedores y Estrategia de Almacenamiento

En un equipo universitario, estudiantes y desarrolladores trabajan sobre distintos sistemas operativos (Windows, macOS, Linux). Sin contenedores, las diferencias en versiones de Python o librerías instaladas generan fallos constantes al desplegar en el servidor. Docker Compose integra los tres servicios de forma declarativa dentro de una red bridge aislada:

```yaml
# docker-compose.yml (extracto con marcadores neutrales anonimizados)
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
      ALLOWED_EMAIL_DOMAIN: example-campus.edu
      APP_ENV: production
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

Dado que los contenedores son efímeros por diseño y se reconstruyen en cada despliegue, el estado persistente se divide en dos volúmenes con nombre en el host:
1. **Datos Relacionales (`postgres_data`):** Resguarda tablas, índices y registros transaccionales en `/var/lib/postgresql/data`.
2. **Archivos Multimedia (`uploads_data`):** Almacena afiches, videos (`MP4`, `WEBM`) y documentos PDF en `/app/uploads`. Los archivos binarios pesados no se guardan como BLOB dentro de PostgreSQL porque saturarían la memoria caché de la base de datos y multiplicarían el peso de las copias de seguridad diarias.

---

## Conclusiones Clave

1. **Separación de Responsabilidades frente a Monolitos:** Dividir el sistema en proxy Nginx, backend FastAPI y base de datos PostgreSQL permite actualizar, probar y blindar cada capa de forma independiente.
2. **La Especificación Funcional como Consecuencia de la Arquitectura:** Reglas como el modelo unificado de publicaciones (`Single Source of Truth`), la sincronización en vivo con Google Classroom y el filtrado autenticado del feed surgen directamente de evitar duplicidad de datos y brechas de seguridad.
3. **Contenedores sin Estado y Volúmenes Persistentes:** Los contenedores de aplicación pueden reemplazarse en cualquier momento sin riesgo, mientras los registros relacionales y archivos multimedia permanecen protegidos en volúmenes dedicados del servidor.
