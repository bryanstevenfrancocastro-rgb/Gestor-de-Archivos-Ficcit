# Guía de Instalación y Configuración — Gestor de Archivos con Firma Electrónica

> Sistema de Gestión Documental y Acreditación Universitaria con Firma Electrónica **PAdES-BES** (ETSI EN 319 142 / ISO 32000-1) para la **Universidad Católica de Cuenca (UCACUE)**.

---

## 📋 Índice

1. [Prerrequisitos del Sistema](#prerrequisitos-del-sistema)
2. [Paso 1 — Clonar el Repositorio](#paso-1--clonar-el-repositorio)
3. [Paso 2 — Configurar la Base de Datos (PostgreSQL) y Almacenamiento](#paso-2--configurar-la-base-de-datos-postgresql-y-almacenamiento)
4. [Paso 3 — Configurar el Backend (API REST)](#paso-3--configurar-el-backend-api-rest)
5. [Paso 4 — Configurar el Frontend (React + Vite)](#paso-4--configurar-el-frontend-react--vite)
6. [Paso 5 — Inicializar la Base de Datos (Seeds y Migraciones)](#paso-5--inicializar-la-base-de-datos-seeds-y-migraciones)
7. [Paso 6 — Scripts de Prueba y Verificación](#paso-6--scripts-de-prueba-y-verificación)
8. [Paso 7 — Ejecutar la Aplicación](#paso-7--ejecutar-la-aplicación)
9. [Paso 8 — Credenciales de Prueba y Roles](#paso-8--credenciales-de-prueba-y-roles)
10. [Paso 9 — Solución de Problemas Frecuentes](#paso-9--solución-de-problemas-frecuentes)

---

## Prerrequisitos del Sistema

Antes de iniciar, asegúrate de tener instaladas las siguientes herramientas en tu entorno:

| Herramienta | Versión mínima | Propósito | Enlace de descarga |
|---|---|---|---|
| **Node.js** | v18 LTS (v20+ recomendado) | Runtime de JavaScript | [nodejs.org](https://nodejs.org/) |
| **npm** | v9+ *(incluido con Node.js)* | Gestor de paquetes | — |
| **PostgreSQL** | v14+ (v15/v16+ o **Neon** en la nube) | Base de datos relacional | [postgresql.org/download](https://www.postgresql.org/download/) |
| **Git** | v2.30+ | Control de versiones | [git-scm.com](https://git-scm.com/) |

### Verificación de versiones instaladas
```bash
node --version    # Debe mostrar v18.x.x o superior
npm --version     # Debe mostrar 9.x.x o superior
psql --version    # Debe mostrar PostgreSQL 14.x o superior (si usas PostgreSQL local)
git --version     # Debe mostrar git version 2.x o superior
```

---

## Paso 1 — Clonar el Repositorio

Abre tu terminal y clona el repositorio del proyecto:

```bash
git clone https://github.com/bryanstevenfrancocastro-rgb/Gestor-de-Archivos-Ficcit.git
cd Gestor-de-Archivos-Ficcit
```

### Estructura del Proyecto

```text
Gestor-de-Archivos-Ficcit/
├── back-end/                              → Servidor API REST (Express + Sequelize)
│   ├── config/
│   │   └── database.js                    → Conexión a PostgreSQL (Sequelize Pool / DATABASE_URL)
│   ├── Model/                             → Modelos ORM y asociaciones relacionales
│   │   ├── Actividad.js                   → 31 actividades oficiales (con subida directa de PDF)
│   │   ├── Carpeta.js                     → Nivel opcional de carpetas dentro de subcriterios/actividades
│   │   ├── Carrera.js
│   │   ├── Criterio.js                    → 5 criterios oficiales
│   │   ├── Documento.js                   → Metadatos, estado de flujo, hash SHA-256 y clave R2
│   │   ├── Facultad.js
│   │   ├── FlujoFirma.js                  → Flujos configurables de firma
│   │   ├── Indicador.js                   → Compatibilidad con indicadores
│   │   ├── PasoFirma.js                   → Pasos con asignación por Rol o Usuario
│   │   ├── Periodo.js
│   │   ├── Rol.js
│   │   ├── Subcriterio.js                 → 8 subcriterios oficiales
│   │   ├── Universidad.js
│   │   ├── Usuario.js
│   │   ├── UsuarioActividad.js            → Pivote M:N Usuario ↔ Actividad (visibilidad)
│   │   ├── UsuarioCarrera.js              → Pivote M:N Usuario ↔ Carrera (multicarrera)
│   │   ├── UsuarioFacultad.js             → Pivote N:M Usuario ↔ Facultad
│   │   ├── UsuarioRol.js                  → Pivote M:N Usuario ↔ Rol (multirrol)
│   │   └── index.js                       → Declaración centralizada de asociaciones
│   ├── Controller/
│   │   ├── auth.controller.js             → Autenticación, usuarios, roles, passwords
│   │   ├── documento.controller.js        → Subida, firma, rechazo y descarga de PDFs
│   │   ├── flujos.controller.js           → Configuración de flujos y pasos de firma
│   │   └── parametrizacion.controller.js  → CRUD de la jerarquía universitaria y carpetas
│   ├── Middleware/
│   │   ├── auth.middleware.js             → Validación de tokens JWT
│   │   └── rbac.middleware.js             → Control de acceso por roles y niveles
│   ├── Routes/
│   │   ├── auth.routes.js                 → /api/auth/*
│   │   ├── documento.routes.js            → /api/documentos/*
│   │   ├── flujos.routes.js               → /api/flujos/*
│   │   └── parametrizacion.routes.js      → /api/parametrizacion/*
│   ├── Services/
│   │   ├── auth.service.js                → Lógica de autenticación y tokens
│   │   ├── firma.service.js               → Motor PAdES-BES incremental, sellos QR (/AP /N) y PKCS#7
│   │   ├── r2.service.js                  → Cliente Cloudflare R2 (@aws-sdk/client-s3) y respaldo local
│   │   └── workflow.service.js            → Motor de transiciones de estados de firma
│   ├── Utils/
│   │   ├── catalogoAcreditacion.js        → Catálogo oficial (5 Criterios, 8 Subcriterios, 31 Actividades)
│   │   └── r2Keys.js                      → Generación de claves jerárquicas para R2
│   ├── tests/                             → Suite de pruebas unitarias, integración y E2E (Jest)
│   │   ├── integration/
│   │   ├── middleware/
│   │   ├── unit/
│   │   ├── docente-relaciones.test.js
│   │   ├── estructura-facultad-periodo-carrera.test.js
│   │   ├── firma-pades.test.js
│   │   ├── flujo-4firmantes-e2e.test.js
│   │   ├── flujo-firmantes-elegibles.test.js
│   │   ├── test_completo.js
│   │   └── setup.js
│   ├── generar-documento-4firmas.js       → Demostración autónoma de firma PAdES-BES en 4 niveles
│   ├── migrate-documento-r2.js            → Migración de columnas de Documentos para Cloudflare R2
│   ├── migrate-relaciones.js              → Migración segura de esquemas y tablas pivote
│   ├── seed.js                            → Seed base (7 roles, estructura inicial y 7 usuarios de prueba)
│   ├── seed-criterios.js                  → Seed de 5 Criterios, 8 Subcriterios oficiales y 31 Actividades
│   ├── seed-carreras-demo.js              → Seed de carreras de demostración
│   ├── server.js                          → Punto de entrada del servidor Express
│   ├── jest.config.js
│   ├── package.json
│   └── .env.example
├── front-end/                             → Aplicación Web SPA (React 18 + Vite + Tailwind)
│   ├── public/
│   │   └── _redirects
│   ├── src/
│   │   ├── components/
│   │   │   ├── common/                    → Badge, Breadcrumbs, MultiSelect
│   │   │   ├── flujos/
│   │   │   │   └── PasoFirmaItem.jsx      → Configuración de pasos por Rol o Usuario
│   │   │   ├── layout/
│   │   │   │   ├── Layout.jsx             → Shell principal con barra lateral
│   │   │   │   └── Sidebar.jsx            → Menú adaptativo por roles
│   │   │   └── modals/
│   │   │       ├── ModalDocenteForm.jsx   → Formulario multirrol, multicarrera y actividades
│   │   │       └── ModalFirma.jsx         → Visor PDF (pdfjs-dist), selector de posición, firma y rechazo
│   │   ├── context/
│   │   │   ├── AppContext.jsx             → Estado global y cliente API
│   │   │   ├── AppContextObject.js
│   │   │   └── useApp.js                  → Hook de acceso al contexto
│   │   ├── data/
│   │   │   └── parametrosPredefinidos.js
│   │   ├── views/
│   │   │   ├── Dashboard.jsx              → Estadísticas y métricas generales
│   │   │   ├── ExploradorDeArchivos.jsx   → Árbol jerárquico, carpetas y gestión de PDFs
│   │   │   ├── GestionUsuarios.jsx        → Administración completa de usuarios
│   │   │   ├── HistorialAuditoria.jsx     → Registro de auditoría y navegación al documento
│   │   │   ├── Login.jsx                  → Inicio de sesión
│   │   │   ├── PanelLateralAuditoria.jsx  → Detalle y trazabilidad de documentos
│   │   │   ├── ParametrizacionAdmin.jsx   → Gestión de jerarquía, subcriterios y flujos de firma
│   │   │   ├── PerfilCertificado.jsx      → Perfil de usuario y certificado digital
│   │   │   ├── RecuperarPassword.jsx      → Solicitud de recuperación de clave
│   │   │   └── ResetPassword.jsx          → Restablecimiento de clave con token
│   │   ├── App.jsx                        → Router condicional y ErrorBoundary
│   │   ├── index.css                      → TailwindCSS y estilos globales
│   │   └── main.jsx                       → Entrada de React DOM
│   ├── .env.example
│   ├── package.json
│   └── vite.config.js
├── package.json                           → Scripts globales con concurrently
├── render.yaml                            → Blueprint de despliegue en Render
├── DEPLOY.md                              → Guía de despliegue (Render + Neon/PostgreSQL + Cloudflare)
├── SETUP.md                               → Esta guía de instalación
├── README.md                              → Resumen general del repositorio
└── FLUJO.md                               → Flujo detallado del negocio y firma PAdES-BES
```

---

## Paso 2 — Configurar la Base de Datos (PostgreSQL) y Almacenamiento

### 2.1 Crear la Base de Datos

Abre **pgAdmin** o tu terminal interactiva de `psql` y crea la base de datos local (o usa una instancia en **Neon** con `DATABASE_URL`):

```sql
CREATE DATABASE gestion_archivos;
```

> **Nota:** Las tablas se crean y actualizan de forma automática cuando el servidor Express arranca gracias a `sequelize.sync({ alter: true })`.

### 2.2 Almacenamiento de Documentos (Local en Dev vs. Cloudflare R2 en Producción)

- **En desarrollo (`NODE_ENV=development`)**: si no defines las variables `CLOUDFLARE_R2_*`, el backend usa almacenamiento local automático en `back-end/uploads/`.
- **En producción (`NODE_ENV=production`)**: es obligatorio configurar **Cloudflare R2** (`CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `CLOUDFLARE_R2_BUCKET_NAME`). El backend verifica y crea automáticamente el bucket al iniciar si el token tiene permiso **Admin Read & Write**.

Si vas a usar carpetas locales en desarrollo:

**En Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "back-end\uploads"
```

**En Linux / macOS:**
```bash
mkdir -p back-end/uploads
```

---

## Paso 3 — Configurar el Backend (API REST)

### 3.1 Instalar Dependencias del Backend

```bash
cd back-end
npm install
```

### 3.2 Crear y Configurar el Archivo `.env`

Copia el archivo de plantilla `back-end/.env.example` a `back-end/.env`:

**En Windows (PowerShell):**
```powershell
Copy-Item .env.example .env
```

**En Linux / macOS:**
```bash
cp .env.example .env
```

Abre `back-end/.env` y ajusta los valores según tu entorno (revisa [back-end/.env.example](back-end/.env.example) para la referencia completa):

```env
# ─── BASE DE DATOS (PostgreSQL local; en producción usar DATABASE_URL) ────────
DB_HOST=localhost
DB_PORT=5432
DB_NAME=gestion_archivos
DB_USER=postgres
DB_PASSWORD=tu_password_seguro

# ─── AUTENTICACIÓN (JWT) ──────────────────────────────────────────────────────
JWT_SECRET=CAMBIA_ESTE_SECRETO_POR_UNO_ALEATORIO_Y_LARGO
JWT_EXPIRES_IN=8h

# ─── SERVIDOR EXPRESS ─────────────────────────────────────────────────────────
PORT=3000
NODE_ENV=development
FRONTEND_URL=http://localhost:5173

# ─── ALMACENAMIENTO EN LA NUBE — Cloudflare R2 (obligatorio en production) ─────
# En development puedes dejarlas vacías para usar almacenamiento local ./uploads
CLOUDFLARE_R2_ACCOUNT_ID=
CLOUDFLARE_R2_ACCESS_KEY_ID=
CLOUDFLARE_R2_SECRET_ACCESS_KEY=
CLOUDFLARE_R2_BUCKET_NAME=gestion-archivos-docs
MAX_UPLOAD_MB=20

# ─── FIRMA ELECTRÓNICA INSTITUCIONAL (Opcional, en memoria) ───────────────────
# Si se omiten, cada firmante sube su propio certificado .p12 al firmar
CERT_INSTITUCIONAL_P12_BASE64=
CERT_INSTITUCIONAL_PASSWORD=
```

---

## Paso 4 — Configurar el Frontend (React + Vite)

Navega a `front-end` e instala sus dependencias:

```bash
cd front-end
npm install
```

> **Atajo desde la raíz:** Puedes instalar dependencias de ambos proyectos ejecutando `npm run install:all`.

Para desarrollo local, `front-end/.env.example` apunta por defecto a `http://localhost:3000/api`. Si necesitas personalizar la URL de la API, copia `.env.example` a `.env` en `front-end/`:

```env
VITE_API_URL=http://localhost:3000/api
```

---

## Paso 5 — Inicializar la Base de Datos (Seeds y Migraciones)

### ⚡ Inicialización Rápida (Recomendada)

**Desde la raíz del proyecto:**
```bash
npm run seed
```

**O desde la carpeta `back-end/`:**
```bash
cd back-end
npm run seed:all
```

### 🔍 Detalle de los Scripts de Inicialización

#### 1. Seed Base — Estructura Institucional y Usuarios (`seed.js`)
Crea los **7 roles jerárquicos**, la estructura base institucional y los **7 usuarios de prueba** (`ADMINISTRADOR` + los 6 usuarios `*@ucacue.edu.ec`).

```bash
npm run seed:base
```

#### 2. Seed de Criterios, Subcriterios y Actividades (`seed-criterios.js`)
Inserta y sincroniza los **5 Criterios**, **8 Subcriterios oficiales** y **31 Actividades** del catálogo de acreditación (`Utils/catalogoAcreditacion.js`).

```bash
npm run seed:criterios
```

#### 3. Seed de Carreras de Demostración (`seed-carreras-demo.js`)
Registra carreras adicionales bajo la Facultad de Ingeniería para pruebas multicarrera.

```bash
npm run seed:carreras
```

#### 4. Scripts de Migración Segura (`migrate-relaciones.js` y `migrate-documento-r2.js`)
Si actualizas una base de datos existente con datos previos:

```bash
cd back-end
node migrate-relaciones.js
node migrate-documento-r2.js
```

---

## Paso 6 — Scripts de Prueba y Verificación

### 🧪 Demostración Autónoma de Firma en 4 Niveles (`generar-documento-4firmas.js`)

Valida de forma autónoma el pipeline criptográfico **PAdES-BES incremental**:
1. Genera un documento PDF de prueba y 4 certificados PKCS#12 (`.p12`) en memoria.
2. Aplica consecutivamente las 4 firmas digitales con cálculo real de `ByteRange`, contenedores PKCS#7 y sellos visuales QR progresivos mediante Appearance Stream (`/AP /N`).
3. Guarda el PDF resultante verificable en Adobe Acrobat Reader.

```bash
cd back-end
node generar-documento-4firmas.js
```

### 🧪 Ejecutar la Suite de Pruebas Automatizadas (Jest)

```bash
cd back-end
npm test                  # Todas las pruebas
npm run test:verbose      # Salida detallada
npm run test:coverage     # Reporte de cobertura
```

---

## Paso 7 — Ejecutar la Aplicación

### Opción A — Comando Único desde la Raíz *(Recomendado)*

```bash
npm run dev
```

- **Backend API:** `http://localhost:3000`
- **Frontend Web:** `http://localhost:5173`

### Opción B — Terminales Separadas

```bash
# Terminal 1
cd back-end && npm run dev

# Terminal 2
cd front-end && npm run dev
```

---

## Paso 8 — Credenciales de Prueba y Roles

El sistema implementa control de acceso basado en roles jerárquicos (**RBAC** de 7 niveles) con soporte multirrol, multicarrera y visibilidad granular por actividad. Estos son los **7 usuarios creados por `back-end/seed.js`**:

| Nivel | Rol | Correo Electrónico | Contraseña | Alcance y Funciones |
|:---:|---|---|---|---|
| **7** | `ADMINISTRADOR` | `admin@universidad.edu` | `Admin123!` | Gestión total del sistema, parametrización institucional, flujos de firma y usuarios. *(No firma documentos)* |
| **6** | `RECTOR` | `rector.prueba@ucacue.edu.ec` | `Password123!` | Máxima autoridad institucional. Firma final y aprobación definitiva. |
| **5** | `DECANO` | `decano.prueba@ucacue.edu.ec` | `Password123!` | Autoridad de Facultad. Tercera etapa del flujo estándar de firma. |
| **4** | `SUBDECANO` | `subdecano.prueba@ucacue.edu.ec` | `Password123!` | Gestión académica de facultad. Segunda etapa del flujo estándar de firma. |
| **3** | `DIRECTOR_CARRERA` | `director.prueba@ucacue.edu.ec` | `Password123!` | Director de Carrera. Primera firma técnica y revisión por carrera. |
| **2** | `DOCENTE` | `docente.prueba@ucacue.edu.ec` | `Password123!` | Carga de evidencias documentales (PDF) en actividades y carpetas asignadas. |
| **1** | `RESPONSABLE_AREA` | `responsable.prueba@ucacue.edu.ec` | `Password123!` | Carga y gestión de evidencias institucionales asignadas. |

---

## Paso 9 — Solución de Problemas Frecuentes

### ❌ `connect ECONNREFUSED 127.0.0.1:5432`
El servicio de PostgreSQL local no está ejecutándose.
- **Windows:** `Win + R` → `services.msc` → **PostgreSQL** → **Iniciar**.
- **Linux:** `sudo systemctl start postgresql`
- **macOS:** `brew services start postgresql`

### ❌ `ERROR FATAL: Faltan variables de entorno requeridas para Cloudflare R2`
En `NODE_ENV=production`, el servidor exige las 4 variables `CLOUDFLARE_R2_*`. Para desarrollo local sin R2, asegúrate de tener `NODE_ENV=development` en `back-end/.env`.

### ❌ `database "gestion_archivos" does not exist`
Crea la base en PostgreSQL:
```sql
CREATE DATABASE gestion_archivos;
```

### ❌ Error al ejecutar seeds: `relation "roles" does not exist`
Inicia primero el backend una vez (`cd back-end && npm run dev`) para que `sequelize.sync()` cree las tablas, detén con `Ctrl + C`, y vuelve a correr `npm run seed:all`.

### ❌ Error de columnas en tablas pivote o almacenamiento R2
Si actualizaste desde una versión anterior con datos previos, ejecuta:
```bash
cd back-end
node migrate-relaciones.js
node migrate-documento-r2.js
```

### ❌ Error al firmar: certificado `.p12` o contraseña inválida
Si usas firma institucional por entorno, verifica `CERT_INSTITUCIONAL_P12_BASE64` y `CERT_INSTITUCIONAL_PASSWORD`. Si firmas por usuario desde la interfaz, sube un archivo `.p12` / `.pfx` válido en el modal de firma o en tu perfil.

### ❌ Puerto 3000 o 5173 ocupado (`EADDRINUSE`)
- **Windows (PowerShell):**
  ```powershell
  Get-Process -Id (Get-NetTCPConnection -LocalPort 3000).OwningProcess | Stop-Process -Force
  Get-Process -Id (Get-NetTCPConnection -LocalPort 5173).OwningProcess | Stop-Process -Force
  ```
- **Linux / macOS:**
  ```bash
  npx kill-port 3000 5173
  ```

---

## 📚 Resumen de Comandos Principales

| Acción | Comando | Ubicación |
|---|---|---|
| **Instalar todas las dependencias** | `npm run install:all` | Raíz |
| **Inicializar todos los seeds** | `npm run seed` | Raíz |
| **Ejecutar Frontend y Backend** | `npm run dev` | Raíz |
| **Ejecutar Suite de Tests** | `npm test` | Raíz o `back-end/` |
| **Migración segura de relaciones / R2** | `node migrate-relaciones.js` / `node migrate-documento-r2.js` | `back-end/` |
| **Demostración de 4 Firmas PAdES-BES** | `node generar-documento-4firmas.js` | `back-end/` |
