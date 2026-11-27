# Gestor de Archivos con Firma Electrónica (PAdES-BES)

Sistema web para la gestión documental institucional y acreditación universitaria con firma electrónica **PAdES-BES** (ETSI EN 319 142 / ISO 32000-1) para la **Universidad Católica de Cuenca (UCACUE)**. Permite a docentes y responsables de área subir evidencias en PDF que recorren flujos de aprobación y firma digital secuencial multi-nivel (flujo jerárquico estándar de 4 niveles o flujos dinámicos configurables por rol o usuario).

## Características principales

- **Jerarquía institucional y de acreditación**: **Universidad → Facultad → Carrera → Período → Criterio (5) → Subcriterio (8 oficiales) → Carpeta (opcional) → Actividad (31 oficiales)**, con generación automática de plantilla por período.
- **Subida directa de PDFs y carpetas opcionales**: carga con arrastrar y soltar directamente en la actividad o organizada dentro de carpetas personalizadas.
- **Almacenamiento en la nube con Cloudflare R2**: persistencia de PDFs originales y firmados mediante `@aws-sdk/client-s3` (y respaldo local en `./uploads` en desarrollo cuando no se configuran credenciales R2).
- **Flujos de firma digital PAdES-BES**:
  - Flujo jerárquico estándar de 4 niveles (`DIRECTOR_CARRERA` → `SUBDECANO` → `DECANO` → `RECTOR`).
  - Flujos configurables (`FlujoFirma` / `PasoFirma`) asignables por **rol** o **usuario específico**.
  - Sello visual QR progresivo por cada firma mediante Appearance Stream (`/AP /N`) incremental PAdES-BES y hash de integridad **SHA-256**.
  - Visor PDF interactivo (`pdfjs-dist`) en el modal de firma con selector visual de posición de firma.
- **Control de acceso (`RBAC`) de 7 niveles**: soporte multirrol, multicarrera y visibilidad granular por actividad.
- **Autenticación y seguridad**: JWT (sesión de 8 horas), recuperación y restablecimiento de contraseña, `helmet` y `cors`.
- **Dashboard y auditoría**: métricas por estado/criterio en tiempo real y trazabilidad completa en `HistorialAuditoria` con navegación directa al documento.

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| **Backend** | Node.js · Express · Sequelize |
| **Base de datos** | PostgreSQL (local o **Neon** en producción) |
| **Almacenamiento** | **Cloudflare R2** (`@aws-sdk/client-s3`) · respaldo local `./uploads` en dev |
| **Frontend** | React 18 · Vite · Tailwind CSS · `lucide-react` |
| **Visor PDF** | `pdfjs-dist` |
| **Firma digital** | `@signpdf/signpdf` · `@signpdf/placeholder-plain` · `pkijs` · `asn1js` · `node-forge` · `pdf-lib` · `qrcode` |
| **Autenticación** | JWT (`jsonwebtoken`) · `bcryptjs` |
| **Pruebas** | Jest · Supertest |

## Inicio rápido

Consulta [SETUP.md](SETUP.md) para la guía completa de instalación paso a paso y [DEPLOY.md](DEPLOY.md) para el despliegue en producción (**Render + PostgreSQL/Neon + Cloudflare Pages + Cloudflare R2**).

```bash
# 1. Instalar dependencias (backend y frontend desde la raíz)
npm run install:all

# 2. Configurar .env en back-end (ver back-end/.env.example y SETUP.md)

# 3. Inicializar base de datos (roles, 7 usuarios de prueba y catálogo oficial)
cd back-end
npm run seed:base
npm run seed:criterios
# Opcional: carreras demo
npm run seed:carreras
# O todo desde la raíz: npm run seed

# 4. Ejecutar backend y frontend juntos desde la raíz
cd ..
npm run dev
# Backend:  http://localhost:3000
# Frontend: http://localhost:5173

# 5. Ejecutar pruebas automatizadas
npm test
```

## Flujo de aprobación y firma digital

```text
DOCENTE / RESPONSABLE_AREA sube PDF (en Actividad o Carpeta)
                 ↓
           Estado: PENDIENTE
                 ↓ (Paso 1: DIRECTOR_CARRERA firma)
      Estado: FIRMADO_DIRECTOR
                 ↓ (Paso 2: SUBDECANO firma)
      Estado: FIRMADO_SUBDECANO
                 ↓ (Paso 3: DECANO firma)
      Estado: FIRMADO_DECANO
                 ↓ (Paso 4: RECTOR firma)
          Estado: COMPLETADO
```

Además del flujo estándar de 4 niveles, el administrador puede definir flujos personalizados (`FlujoFirma` / `PasoFirma`) por rol o usuario. En cualquier etapa, un revisor autorizado puede rechazar el documento (estado: `RECHAZADO`), habilitando su re-subida.

Consulta [FLUJO.md](FLUJO.md) para el detalle completo del modelo de datos, estados y proceso criptográfico PAdES-BES.

## Estructura del proyecto

```text
├── back-end/
│   ├── config/                 # Conexión PostgreSQL (database.js)
│   ├── Controller/             # auth, documento, flujos, parametrizacion
│   ├── Middleware/             # auth.middleware.js (JWT) y rbac.middleware.js
│   ├── Model/                  # Modelos Sequelize (Criterio, Subcriterio, Carpeta, Actividad, FlujoFirma, etc.)
│   ├── Routes/                 # /api/auth, /api/documentos, /api/flujos, /api/parametrizacion
│   ├── Services/               # auth.service.js, firma.service.js, r2.service.js, workflow.service.js
│   ├── Utils/                  # catalogoAcreditacion.js, r2Keys.js
│   ├── tests/                  # Pruebas unitarias, integración y E2E (Jest + Supertest)
│   ├── seed.js                 # Seed base (roles, estructura inicial y 7 usuarios de prueba)
│   ├── seed-criterios.js       # 5 Criterios, 8 Subcriterios oficiales y 31 Actividades
│   ├── seed-carreras-demo.js   # Carreras adicionales de demostración
│   ├── migrate-documento-r2.js # Migración de columnas para almacenamiento en R2
│   ├── migrate-relaciones.js   # Migración segura de tablas pivote
│   └── server.js               # Punto de entrada Express
├── front-end/
│   └── src/
│       ├── components/         # common/ (Badge, Breadcrumbs, MultiSelect), flujos/, layout/, modals/ (ModalFirma, ModalDocenteForm)
│       ├── context/            # AppContext y hooks
│       ├── data/               # parametrosPredefinidos.js
│       └── views/              # Login, Dashboard, ExploradorDeArchivos, ParametrizacionAdmin, GestionUsuarios, HistorialAuditoria, PerfilCertificado, etc.
├── DEPLOY.md                   # Guía de despliegue (Render + PostgreSQL + Cloudflare Pages/R2)
├── FLUJO.md                    # Flujo de negocio, estados y firma PAdES-BES
├── SETUP.md                    # Guía de instalación local y solución de problemas
└── render.yaml                 # Blueprint de Render para el backend
```

## Usuarios de prueba (seed)

| Rol | Correo electrónico | Contraseña |
|---|---|---|
| `ADMINISTRADOR` | `admin@universidad.edu` | `Admin123!` |
| `RECTOR` | `rector.prueba@ucacue.edu.ec` | `Password123!` |
| `DECANO` | `decano.prueba@ucacue.edu.ec` | `Password123!` |
| `SUBDECANO` | `subdecano.prueba@ucacue.edu.ec` | `Password123!` |
| `DIRECTOR_CARRERA` | `director.prueba@ucacue.edu.ec` | `Password123!` |
| `DOCENTE` | `docente.prueba@ucacue.edu.ec` | `Password123!` |
| `RESPONSABLE_AREA` | `responsable.prueba@ucacue.edu.ec` | `Password123!` |
