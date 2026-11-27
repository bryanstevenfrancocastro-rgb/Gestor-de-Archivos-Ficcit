# 🚀 Guía de Despliegue

## Stack de producción

| Componente | Plataforma | URL de producción |
|---|---|---|
| **Backend** (Node.js/Express) | [Render](https://render.com) | `https://gestor-archivos-api-1t07.onrender.com` |
| **Base de datos** (PostgreSQL) | [Neon](https://neon.tech) *(o Render Managed PostgreSQL)* | *(conexión mediante `DATABASE_URL`)* |
| **Almacenamiento PDF** | [Cloudflare R2](https://developers.cloudflare.com/r2/) | *(bucket `gestion-archivos-docs`)* |
| **Frontend** (React/Vite) | [Cloudflare Pages](https://pages.cloudflare.com) | `https://gestor-de-archivos-ficcit.pages.dev` |

---

## 1 · Base de datos en Neon (PostgreSQL)

1. Crea un proyecto en **<https://console.neon.tech>** (ej. `gestion-archivos-db`).
2. En **Connect**, copia la **Connection String** completa (`postgresql://...?sslmode=require`).
3. Esa cadena será tu variable `DATABASE_URL` en Render.

> [!NOTE]
> Si prefieres usar PostgreSQL administrado dentro de Render en lugar de Neon, puedes usar el Blueprint [render.yaml](render.yaml) (`New Blueprint Instance` en <https://dashboard.render.com/blueprints>).

---

## 2 · Almacenamiento de PDFs en Cloudflare R2

1. En **<https://dash.cloudflare.com>** → **R2 Object Storage**, activa R2.
2. Ve a **Manage R2 API Tokens** → **Create API Token** con permiso **Admin Read & Write**.
3. Guarda:
   - `CLOUDFLARE_R2_ACCOUNT_ID` *(visible en R2 Overview)*
   - `CLOUDFLARE_R2_ACCESS_KEY_ID`
   - `CLOUDFLARE_R2_SECRET_ACCESS_KEY`
4. Define `CLOUDFLARE_R2_BUCKET_NAME` (ej. `gestion-archivos-docs`). El backend verifica y crea el bucket automáticamente al iniciar si no existe.

---

## 3 · Backend en Render

### 3.1 Crear el servicio web

1. Dashboard de Render → **New** → **Web Service**
2. Conecta el repositorio `bryanstevenfrancocastro-rgb/Gestor-de-Archivos-Ficcit` (rama `main`)
3. Configura:
   - **Root Directory**: `back-end`
   - **Runtime**: `Node`
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
   - **Plan**: `Free` (desarrollo) / `Starter` (producción)

### 3.2 Variables de entorno del backend

En **Environment** del servicio web, añade:

| Variable | Valor |
|---|---|
| `NODE_ENV` | `production` |
| `DATABASE_URL` | *(Connection String de **Neon** o Internal Database URL de Render)* |
| `JWT_SECRET` | *(cadena aleatoria larga)* |
| `JWT_EXPIRES_IN` | `8h` |
| `FRONTEND_URL` | `https://gestor-de-archivos-ficcit.pages.dev` *(tu URL de Cloudflare Pages)* |
| `CLOUDFLARE_R2_ACCOUNT_ID` | *(Account ID hexadecimal de Cloudflare)* |
| `CLOUDFLARE_R2_ACCESS_KEY_ID` | *(Access Key ID del token R2)* |
| `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | *(Secret Access Key del token R2)* |
| `CLOUDFLARE_R2_BUCKET_NAME` | `gestion-archivos-docs` |
| `MAX_UPLOAD_MB` | `20` |
| `CERT_INSTITUCIONAL_P12_BASE64` | *(opcional, certificado `.p12` en Base64 si usas firma institucional)* |
| `CERT_INSTITUCIONAL_PASSWORD` | *(opcional, contraseña del certificado institucional)* |

> [!NOTE]
> El servicio **Free** de Render hiberna tras 15 minutos de inactividad; la primera petición tras hibernar puede tardar ~30 s. Los archivos PDF no se pierden porque se almacenan en **Cloudflare R2**.

---

## 4 · Frontend en Cloudflare Pages

### 4.1 Conectar el repositorio

1. Ve a **<https://dash.cloudflare.com>** → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**
2. Selecciona `bryanstevenfrancocastro-rgb/Gestor-de-Archivos-Ficcit` (rama `main`)
3. Configura el build:

| Campo | Valor |
|---|---|
| **Framework preset** | Vite |
| **Root directory** | `front-end` |
| **Build command** | `npm run build` |
| **Build output directory** | `dist` |

### 4.2 Variables de entorno del frontend

En **Settings → Variables and Secrets** de tu proyecto de Pages, añade:

| Variable | Valor (Production) |
|---|---|
| `VITE_API_URL` | `https://gestor-archivos-api-1t07.onrender.com/api` |

> [!IMPORTANT]
> El prefijo `VITE_` es obligatorio para que Vite exponga la variable al código del navegador. Después de añadirla o modificarla, haz **Retry deployment** en el último deploy.

---

## 5 · Inicialización de la base de datos

Render ejecuta solo `npm start`. Para correr los seeds en producción (desde la **Shell** de Render o localmente apuntando a `DATABASE_URL`):

```bash
npm run seed:base       # Roles, estructura inicial y 7 usuarios de prueba
npm run seed:criterios  # 5 Criterios, 8 Subcriterios oficiales y 31 Actividades
npm run seed:carreras   # Carreras de demo (opcional)

# O todo de una vez:
npm run seed:all
```

---

## 6 · Verificar el despliegue

```bash
curl https://gestor-archivos-api-1t07.onrender.com/api/health
# Respuesta esperada:
# {"estado":"OK","almacenamiento":"Cloudflare R2","bucket":"gestion-archivos-docs","timestamp":"..."}
```

---

## 7 · URLs y Accesos del Sistema Desplegado

| Servicio | URL Pública | Estado |
|---|---|---|
| **Frontend (Cloudflare Pages)** | [https://gestor-de-archivos-ficcit.pages.dev](https://gestor-de-archivos-ficcit.pages.dev) | 🟢 Activo |
| **Backend API (Render)** | [https://gestor-archivos-api-1t07.onrender.com](https://gestor-archivos-api-1t07.onrender.com) | 🟢 Activo |
| **Health Check API** | [https://gestor-archivos-api-1t07.onrender.com/api/health](https://gestor-archivos-api-1t07.onrender.com/api/health) | 🟢 Activo |

### Credenciales de acceso de prueba (`seed.js`)

| Rol | Correo | Contraseña |
|---|---|---|
| `ADMINISTRADOR` | `admin@universidad.edu` | `Admin123!` |
| `RECTOR` | `rector.prueba@ucacue.edu.ec` | `Password123!` |
| `DECANO` | `decano.prueba@ucacue.edu.ec` | `Password123!` |
| `SUBDECANO` | `subdecano.prueba@ucacue.edu.ec` | `Password123!` |
| `DIRECTOR_CARRERA` | `director.prueba@ucacue.edu.ec` | `Password123!` |
| `DOCENTE` | `docente.prueba@ucacue.edu.ec` | `Password123!` |
| `RESPONSABLE_AREA` | `responsable.prueba@ucacue.edu.ec` | `Password123!` |
