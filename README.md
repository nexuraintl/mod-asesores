# ms_web_asesores

Frontend de la consola de **asesores** de la plataforma NEXURA. Interfaz web con la que los agentes de soporte gestionan conversaciones del chatbot, visualizan métricas y administran casos en tiempo real.

---

## Descripción

Aplicación web construida con **React** (Create React App), compilada y servida mediante **Nginx** sobre **Cloud Run**. Es la interfaz operativa que usan los asesores internos para atender a los clientes finales que interactúan con el chatbot (`ms_web_clientes`).

A diferencia de `ms_web_clientes` (que es multi-tenant con una página por empresa), esta aplicación expone una única SPA que gestiona la sesión del asesor y se conecta al backend mediante el API Gateway de la plataforma.

---

## Stack

| Componente | Tecnología |
|---|---|
| Framework | React (Create React App) |
| Servidor web | Nginx Alpine |
| Contenedor | Docker |
| Compute | Cloud Run (GCP) |
| Puerto expuesto | 8080 |
| CI/CD | Azure DevOps → GitHub → Cloud Build |
| Iconografía | Font Awesome 4.7, Material Design Icons, Themify Icons |
| Gráficos | Chart.js, C3, Chartist, Morris |
| UI base | Bootstrap 4 + Circular Std font |

---

## Estructura del repositorio

```
ms_web_asesores/
├── build/                          # Artefactos compilados (servidos por Nginx)
│   ├── index.html                  # SPA principal — consola del asesor
│   ├── assets/
│   │   ├── icons/                  # Íconos de UI (adjuntar, enviar, estados, redes)
│   │   ├── images/                 # Avatares, logos, imágenes de la interfaz
│   │   ├── libs/                   # JS de dashboards (ecommerce, finance, sales)
│   │   └── vendor/                 # Librerías de terceros
│   │       ├── bootstrap/          # Bootstrap 4
│   │       ├── datatables/         # Tablas con paginación y filtros
│   │       ├── charts/             # Chart.js, C3, Chartist, Morris
│   │       ├── fonts/              # Font Awesome 4.7, Circular Std, Material Icons
│   │       ├── datepicker/         # Selector de fechas con Moment.js
│   │       ├── full-calendar/      # Calendario de eventos
│   │       └── select2/            # Dropdowns avanzados
│   ├── static/
│   │   ├── css/                    # Estilos compilados de la app
│   │   ├── js/                     # Bundles JS de la app (4 chunks + runtime)
│   │   └── media/                  # Fuentes, audio (beep, classic_phone), CSV de prueba
│   ├── favicon.ico
│   └── manifest.json
├── nginx.conf
├── Dockerfile
├── .azure-pipelines.yml
└── README.md
```

---

## Infraestructura y despliegue

### Arquitectura de ejecución

```
Asesor (navegador interno)
    │  HTTPS
    ▼
Cloud Load Balancer  (pre-mscloud.nexura.com.co)
    ▼
API Gateway ESPv2
    ▼
Cloud Run  (qam-web-asesores / prem-web-asesores)
    │  Nginx escuchando en :8080
    └── Sirve build/ como SPA (fallback a index.html)
```

### Servidor web — nginx.conf

Nginx está configurado para:
- Escuchar en el puerto **8080** (requerido por Cloud Run).
- Servir el directorio `/usr/share/nginx/html` (contenido de `build/`).
- Fallback a `index.html` para cualquier ruta no encontrada (`try_files`) — necesario para el enrutamiento de la SPA.
- Caché de 30 días para assets estáticos (JS, CSS, imágenes, SVG).
- `server_tokens off` — no expone la versión de Nginx en headers.

### Dockerfile

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -rf /usr/share/nginx/html/*
COPY build/ /usr/share/nginx/html
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

### Ramas y ambientes

| Rama | Ambiente | Trigger Cloud Build | Nombre Cloud Run |
|---|---|---|---|
| `qa` | QAM | Automático al push | `qam-web-asesores` |
| `master` | PREM | Automático al push | `prem-web-asesores` |
| `main` | PROD | Requiere aprobación | `prod-web-asesores` |

### Pipeline CI/CD

El archivo `.azure-pipelines.yml` sincroniza las ramas `dev`, `qa` y `master` desde Azure DevOps hacia el repositorio espejo en GitHub (`nexuraintl/ms_web_asesores`). Cloud Build detecta el push en GitHub y ejecuta la build + despliegue en Cloud Run.

**Requisito:** variable de pipeline `GITHUB_TOKEN_NEXURAINTL` configurada en ADO con scope `repo`.

---

## Ejecución local

```bash
# Build local
docker build -t ms-web-asesores:local .

# Ejecutar
docker run --rm -p 8080:8080 ms-web-asesores:local

# Verificar
curl http://localhost:8080/
# Abrir en el navegador: http://localhost:8080
```

> No requiere variables de entorno — es un servidor de archivos estáticos puro. La configuración del backend (URLs del API Gateway, Firebase) está embebida en los bundles JS compilados.

---

## Assets de audio

La aplicación incluye dos archivos de audio en `build/static/media/` usados para notificaciones al asesor:

| Archivo | Uso |
|---|---|
| `beep.0bd3827c.mp3` | Notificación de nuevo mensaje |
| `classic_phone.ad11bb53.mp3` | Notificación de nueva conversación entrante |

---

## Dependencias externas

| Dependencia | Tipo | Descripción |
|---|---|---|
| Firebase | Backend/Auth | Autenticación de asesores y sincronización de conversaciones en tiempo real (`chat-asesores-3` / `cliente-chat-desa`) |
| `ms_ia_chatbot` | Microservicio | Lógica del chatbot que los asesores supervisan y a la que pueden escalar |
| API Gateway ESPv2 | Infraestructura | Punto de entrada autenticado para las llamadas del frontend al backend |
| Cloud Build | CI/CD | Build y despliegue de la imagen Docker |
| Artifact Registry | Registro | Almacenamiento de la imagen Docker |

---

## Diferencias con ms_web_clientes

| Aspecto | ms_web_asesores | ms_web_clientes |
|---|---|---|
| Audiencia | Asesores internos de NEXURA | Clientes finales de las empresas |
| Tipo de app | SPA (Single Page Application) | Multi-página estática (una HTML por tenant) |
| Multi-tenant | No — una sola app para todos los asesores | Sí — una página HTML por empresa (`data-company`) |
| UI | Dashboard completo con tablas, gráficos, calendario | Widget de chatbot embebido |
| Audio | Sí (notificaciones al asesor) | No |

---

## Responsable técnico

Santiago Valenzuela López — Equipo de Plataforma NEXURA
