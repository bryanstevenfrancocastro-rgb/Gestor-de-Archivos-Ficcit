# Flujo de la Aplicación

Este documento describe la arquitectura de negocio, la jerarquía de acreditación, los flujos de aprobación por firma electrónica **PAdES-BES** y los módulos funcionales del sistema.

---

## 🎭 Roles del Sistema (7 Niveles Jerárquicos)

El sistema implementa control de acceso basado en roles (`RBAC`) jerárquico de 7 niveles, con soporte **multirrol**, **multicarrera** y asignación granular de actividades:

| Nivel | Rol | Descripción | Permisos y Capacidades |
|:---:|---|---|---|
| **1** | `RESPONSABLE_AREA` | Responsable de área | Sube evidencias/documentos PDF asignados en las actividades de acreditación. |
| **2** | `DOCENTE` | Docente universitario | Sube documentos académicos (sílabos, entregables) en sus actividades o carpetas asignadas. |
| **3** | `DIRECTOR_CARRERA` | Director de carrera | Primera firma electrónica en el flujo estándar de aprobación de su carrera. |
| **4** | `SUBDECANO` | Subdecano | Segunda firma electrónica en el flujo estándar de aprobación de la facultad. |
| **5** | `DECANO` | Decano de facultad | Tercera firma electrónica en el flujo estándar de aprobación de la facultad. |
| **6** | `RECTOR` | Rector de la universidad | Firma electrónica final (cuarta firma) y aprobación institucional definitiva. |
| **7** | `ADMINISTRADOR` | Administrador del sistema | Gestión total del sistema (usuarios, estructura, flujos de firma, auditoría). *No firma documentos.* |

---

## 🏛️ Jerarquía Documental y de Acreditación

La estructura documental sigue la parametrización institucional generada automáticamente por carrera y período académico (`Utils/catalogoAcreditacion.js`):

```text
Universidad
  └── Facultad
        └── Carrera
              └── Período Académico (ej: 2026-I)
                    └── Criterio de Acreditación (5 Criterios)
                          └── Subcriterio Oficial (8 Subcriterios; Criterio 1 agrupa directo)
                                └── Carpeta (Nivel opcional creado por Admin)
                                      └── Actividad (31 Actividades oficiales — PDFs)
```

> **Subida directa o con carpetas:** Los documentos PDF pueden subirse directamente dentro de una **Actividad** o organizarse dentro de **Carpetas opcionales** creadas dentro de subcriterios/actividades.

### 📋 Los 5 Criterios y 8 Subcriterios Oficiales

1. **1. CURRÍCULO** *(Actividades 1 a 7 directamente bajo el criterio)*
2. **2. DOCENCIA**
   - `2.1 PERSONAL ACADÉMICO` *(Actividades 8 a 10)*
   - `2.2 ESTUDIANTES` *(Actividades 11 a 17)*
3. **3. INVESTIGACIÓN E INNOVACIÓN**
   - `3.1 GESTIÓN DE LA INVESTIGACIÓN E INNOVACIÓN` *(Actividad 18)*
   - `3.2 CONTRIBUCIÓN A LA INVESTIGACIÓN E INNOVACIÓN` *(Actividades 19 y 20)*
4. **4. VINCULACIÓN CON LA SOCIEDAD**
   - `4.1 ORGANIZACIÓN DE LA VINCULACIÓN CON LA SOCIEDAD` *(Actividad 21)*
   - `4.2 TRANSFERENCIA DE TECNOLOGÍAS Y PRÁCTICAS PREPROFESIONALES` *(Actividades 22 y 23)*
5. **5. FUNCIONES ESTRATÉGICAS Y DE SOPORTE**
   - `5.1 FUNCIONES ESTRATÉGICAS` *(Actividades 24 a 27)*
   - `5.2 FUNCIONES DE SOPORTE` *(Actividades 28 a 31)*

---

## ✍️ Flujo de Firma Digital PAdES-BES

El sistema soporta tanto el **flujo jerárquico estándar de 4 niveles** como **flujos configurables** (`FlujoFirma` / `PasoFirma`) donde el administrador define pasos personalizados asignados por **rol** o por **usuario específico**.

### Flujo Jerárquico Estándar (4 Niveles)

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                 1. DOCENTE / RESPONSABLE DE ÁREA                         │
│                                                                          │
│  - Navega en el Explorador hasta la Actividad o Carpeta correspondiente. │
│  - Sube el documento PDF (almacenado en Cloudflare R2 o ./uploads en dev)│
│  - El documento ingresa en estado: PENDIENTE                             │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                 2. DIRECTOR DE CARRERA (Paso 1/4)                        │
│                                                                          │
│  - Previsualiza el PDF en el visor interactivo y selecciona posición.    │
│  - ✅ Firmar: Aplica firma PAdES-BES + sello QR → FIRMADO_DIRECTOR       │
│  - ❌ Rechazar: Estado: RECHAZADO (habilita re-subida)                   │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                 3. SUBDECANO (Paso 2/4)                                  │
│                                                                          │
│  - Revisa el documento con la firma 1 estampada.                         │
│  - ✅ Firmar: Aplica firma incremental → FIRMADO_SUBDECANO               │
│  - ❌ Rechazar: Estado: RECHAZADO                                        │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                 4. DECANO (Paso 3/4)                                     │
│                                                                          │
│  - Revisa el documento con las firmas 1 y 2 estampadas.                  │
│  - ✅ Firmar: Aplica firma incremental → FIRMADO_DECANO                  │
│  - ❌ Rechazar: Estado: RECHAZADO                                        │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                 5. RECTOR (Paso 4/4 - Firma Final)                       │
│                                                                          │
│  - Revisa el documento con las firmas 1, 2 y 3 estampadas.               │
│  - ✅ Firmar: Aplica firma incremental final → COMPLETADO                │
│  - ❌ Rechazar: Estado: RECHAZADO                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

### 📊 Estados del Documento

| Estado | Significado | Siguiente Acción / Rol |
|---|---|---|
| `PENDIENTE` | Recién cargado por Docente / Responsable | Pendiente firma de `DIRECTOR_CARRERA` (o paso 1 del flujo) |
| `FIRMADO_DIRECTOR` | Firmado por Director de Carrera | Pendiente firma de `SUBDECANO` |
| `FIRMADO_SUBDECANO` | Firmado por Subdecano | Pendiente firma de `DECANO` |
| `FIRMADO_DECANO` | Firmado por Decano | Pendiente firma final de `RECTOR` |
| `COMPLETADO` | Todas las firmas del flujo completadas | Proceso finalizado exitosamente |
| `RECHAZADO` | Rechazado en cualquier etapa del flujo | Habilitado para corrección y re-subida |

---

## 🔏 Proceso Técnico de Firma PAdES-BES Incremental

Al firmar un documento desde `ModalFirma`:

1. **Visor interactivo (`pdfjs-dist`)**: el firmante visualiza el PDF real y selecciona gráficamente la página y posición donde desea estampar su firma.
2. **Carga de certificado `.p12`**: se utiliza el certificado `.p12` provisto por el firmante (o el certificado institucional en memoria vía `CERT_INSTITUCIONAL_P12_BASE64` si está configurado) y se valida con `node-forge` / `pkijs` / `asn1js`.
3. **Sello visual QR progresivo (`/AP /N`)**: cada firma estampa únicamente su propio sello visual con código QR (`qrcode` + `pdf-lib`) mediante Appearance Stream incremental, sin pre-estampar firmas futuras.
4. **Firma criptográfica incremental**: `@signpdf/signpdf` y `@signpdf/placeholder-plain` incrustan la firma PKCS#7 preservando intactas las firmas anteriores.
5. **Integridad y persistencia**: se recalcula el hash **SHA-256** del PDF firmado, se sube la nueva versión a **Cloudflare R2** (`r2.service.js`) y se registra el evento en auditoría.

---

## 🖥️ Módulos de la Aplicación

1. **Dashboard**: panel estadístico interactivo con métricas de estado, desglose por criterio, alertas de flujo y actividad reciente.
2. **Explorador de Archivos**: navegación en árbol jerárquico (Universidad → Facultad → Carrera → Período → Criterio → Subcriterio → Carpeta → Actividad), subida directa de PDFs, descarga y apertura del modal de firma.
3. **Parametrización Admin**: gestión de la jerarquía institucional, creación de carpetas opcionales y configuración de flujos dinámicos de firma (`FlujoFirma` / `PasoFirma`).
4. **Gestión de Usuarios**: administración de cuentas, asignación multirrol, multicarrera, actividades visibles y restablecimiento administrativo de contraseñas.
5. **Historial de Auditoría**: bitácora de eventos (cargas, firmas, rechazos, cambios de estructura y autenticaciones) con navegación directa al documento auditado.
6. **Perfil y Certificado**: gestión del perfil del usuario y configuración de firma digital.
