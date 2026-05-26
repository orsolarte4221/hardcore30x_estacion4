# Domain Entities — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-25  

---

## Principios del Modelo de Dominio

- **IDs**: UUID v7 (ordenados por tiempo, mejor rendimiento en índices PostgreSQL)
- **Timestamps**: Todas las entidades tienen `created_at` y `updated_at` (UTC, ISO 8601)
- **Borrado**: Físico (`DELETE`) — no hay funcionalidad de borrado expuesta al usuario en MVP; la trazabilidad es responsabilidad del `AuditTrail`
- **Convención de nombres**: `snake_case` para campos, `PascalCase` para entidades

---

## Entidades del Dominio

### 1. `Usuario`

Representa a un usuario autenticado en el sistema.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Email de Google Workspace |
| `nombre` | VARCHAR(255) | NOT NULL | Nombre completo (desde token Google) |
| `rol` | ENUM | NOT NULL | `ANALISTA \| ADMIN \| CISO \| COMITE` |
| `activo` | BOOLEAN | NOT NULL, DEFAULT TRUE | Si el usuario puede acceder al sistema |
| `ultimo_login` | TIMESTAMP | NULL | Último acceso registrado |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de creación del registro |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación del registro |

**Notas de negocio:**
- El rol es asignado por un `ADMIN` desde la UI de administración (tabla local PostgreSQL)
- Un email bootstrap inicial de `ADMIN` se define en `.env` como semilla
- El proveedor OAuth2 es exclusivamente Google Workspace para el MVP
- El `email` proviene del token de identidad de Google y es inmutable

---

### 2. `Portafolio`

Agrupa el conjunto de cotizaciones de un proceso de adquisición SAB.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `codigo_proceso` | VARCHAR(50) | UNIQUE, NOT NULL | Código alfanumérico del proceso SAB (ej: `PROC-2026-001`) |
| `nombre_proceso` | VARCHAR(500) | NOT NULL | Descripción legible del proceso de compra |
| `estado` | ENUM | NOT NULL | `PENDIENTE \| INGESTA \| ANALISIS \| REVISION_HITL \| COMPLETADO \| ERROR` |
| `analista_id` | UUID v7 | FK → Usuario, NOT NULL | Analista responsable del proceso |
| `total_cotizaciones` | INTEGER | NOT NULL, DEFAULT 0 | Número total de cotizaciones en el portafolio |
| `cotizaciones_procesadas` | INTEGER | NOT NULL, DEFAULT 0 | Número de cotizaciones ya analizadas |
| `metadata_sab` | JSONB | NULL | Datos adicionales del proceso extraídos de SAB |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Ciclo de vida del estado:**
```
PENDIENTE → INGESTA → ANALISIS → REVISION_HITL → COMPLETADO
                                               ↘ ERROR (puede ocurrir en cualquier etapa)
```

**Notas de negocio:**
- El `codigo_proceso` es el identificador de negocio (legible por el usuario)
- No hay funcionalidad de borrado expuesta en MVP; un portafolio puede quedar en estado `ERROR`
- `total_cotizaciones` y `cotizaciones_procesadas` se actualizan como el pipeline avanza

---

### 3. `Cotizacion`

Representa la oferta de un proveedor dentro de un portafolio.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `portafolio_id` | UUID v7 | FK → Portafolio, NOT NULL | Portafolio al que pertenece |
| `nombre_proveedor` | VARCHAR(500) | NOT NULL | Nombre del proveedor que presentó la oferta |
| `email_proveedor` | VARCHAR(255) | NULL | Email del proveedor para comunicaciones |
| `numero_oferta` | VARCHAR(100) | NULL | Número de referencia de la oferta del proveedor |
| `estado_analisis` | ENUM | NOT NULL | `PENDIENTE \| EN_PROCESO \| COMPLETADO \| ERROR` |
| `puntaje_global` | DECIMAL(5,2) | NULL | Puntaje de confianza global (0.00–100.00) |
| `requiere_revision_hitl` | BOOLEAN | NOT NULL, DEFAULT FALSE | Si el analista debe revisar manualmente |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Notas de negocio:**
- Una `Cotizacion` pertenece a exactamente un `Portafolio`
- El `puntaje_global` es calculado por el motor de IA al finalizar el análisis
- `requiere_revision_hitl` se activa automáticamente si alguna variable tiene `confianza` baja o contiene elementos adversariales

---

### 4. `DocumentoPDF`

Representa un archivo PDF adjunto a una cotización.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL | Cotización a la que pertenece |
| `nombre_archivo` | VARCHAR(500) | NOT NULL | Nombre original del archivo |
| `ruta_almacenamiento` | VARCHAR(1000) | NOT NULL | Ruta o URL donde está almacenado el PDF |
| `tamano_bytes` | BIGINT | NOT NULL | Tamaño del archivo en bytes |
| `hash_sha256` | VARCHAR(64) | NOT NULL | Hash SHA-256 para verificación de integridad |
| `estado_sanitizacion` | ENUM | NOT NULL | `PENDIENTE \| LIMPIO \| ADVERSARIAL \| ERROR` |
| `procesado_por_ia` | BOOLEAN | NOT NULL, DEFAULT FALSE | Si el motor IA ya procesó este PDF |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de carga |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Notas de negocio:**
- El `hash_sha256` se calcula al recibir el PDF y se verifica antes de procesar
- `estado_sanitizacion = ADVERSARIAL` dispara un evento `ALERTA_SEGURIDAD` vía WebSocket
- Un PDF con estado `ADVERSARIAL` no se procesa por el motor IA; se registra en `AuditTrail`

---

### 5. `VariableExtraida`

Representa un dato específico extraído de un `DocumentoPDF` por el motor de IA.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL | Cotización a la que pertenece |
| `documento_id` | UUID v7 | FK → DocumentoPDF, NOT NULL | PDF del que fue extraída |
| `nombre` | VARCHAR(255) | NOT NULL | Nombre del campo extraído (ej: `vigencia_garantia`, `penalidad_retraso`) |
| `valor` | TEXT | NULL | Valor del dato tal como aparece en el PDF |
| `confianza` | DECIMAL(5,4) | NOT NULL | Score de confianza de la extracción (0.0000–1.0000) |
| `referencia_fuente` | TEXT | NULL | Fragmento de texto del PDF que respalda el valor |
| `analisis` | TEXT | NULL | Descripción del agente IA sobre el dato extraído (por qué lo extrajo, observaciones) |
| `requiere_revision` | BOOLEAN | NOT NULL, DEFAULT FALSE | Si el analista debe revisar esta variable manualmente |
| `valor_revisado` | TEXT | NULL | Valor corregido por el analista en HITL (NULL si no fue corregido) |
| `revisado_por` | UUID v7 | FK → Usuario, NULL | Analista que realizó la revisión HITL |
| `revisado_en` | TIMESTAMP | NULL | Timestamp de la revisión HITL |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de extracción |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Notas de negocio:**
- El campo `analisis` es texto libre generado por el agente IA; documenta el razonamiento detrás de la extracción
- `requiere_revision = TRUE` cuando `confianza < umbral_configurado` (umbral configurable por `ADMIN`)
- Si `valor_revisado IS NOT NULL`, el valor corregido tiene precedencia sobre `valor` en reportes
- Los tipos de variable no usan enum explícito; se derivan por convención de nombres

---

### 6. `Discrepancia`

Compara un campo del sistema SAB con la variable extraída del PDF.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL | Cotización analizada |
| `variable_extraida_id` | UUID v7 | FK → VariableExtraida, NULL | Variable del PDF (NULL si ausente en PDF) |
| `nombre_campo_sab` | VARCHAR(255) | NOT NULL | Nombre del campo en SAB |
| `valor_sab` | TEXT | NULL | Valor registrado en SAB |
| `valor_pdf` | TEXT | NULL | Valor encontrado en el PDF |
| `tipo_discrepancia` | VARCHAR(50) | NOT NULL | Derivado lógicamente: `VALOR_DIFERENTE \| AUSENTE_EN_PDF \| SOLO_EN_PDF` |
| `severidad` | ENUM | NOT NULL | `BAJA \| MEDIA \| ALTA \| CRITICA` |
| `resuelta` | BOOLEAN | NOT NULL, DEFAULT FALSE | Si la discrepancia fue resuelta vía aclaración |
| `aclaracion_id` | UUID v7 | FK → Aclaracion, NULL | Aclaración que resolvió esta discrepancia |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de detección |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Lógica de derivación de `tipo_discrepancia`:**
- `valor_sab IS NOT NULL AND valor_pdf IS NOT NULL AND valor_sab ≠ valor_pdf` → `VALOR_DIFERENTE`
- `valor_sab IS NOT NULL AND valor_pdf IS NULL` → `AUSENTE_EN_PDF`
- `valor_sab IS NULL AND valor_pdf IS NOT NULL` → `SOLO_EN_PDF`

**Notas de negocio:**
- No se usa enum explícito en BD; el tipo se deriva y almacena como `VARCHAR` calculado
- La `severidad` es asignada por el motor IA basándose en el impacto financiero del campo

---

### 7. `Aclaracion`

Registra una consulta enviada a un proveedor para resolver discrepancias.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL | Cotización sobre la que se hace la aclaración |
| `solicitado_por` | UUID v7 | FK → Usuario, NOT NULL | Analista que solicita la aclaración |
| `estado` | ENUM | NOT NULL | `ENVIADA \| RESPONDIDA \| VENCIDA \| CERRADA` |
| `pregunta` | TEXT | NOT NULL | Texto de la consulta enviada al proveedor |
| `respuesta_proveedor` | TEXT | NULL | Respuesta recibida del proveedor |
| `fecha_limite` | TIMESTAMP | NULL | Fecha límite para recibir la respuesta |
| `respondido_en` | TIMESTAMP | NULL | Timestamp de recepción de la respuesta |
| `created_at` | TIMESTAMP | NOT NULL | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL | Última modificación |

**Notas de negocio:**
- Una `Aclaracion` puede estar vinculada a múltiples `Discrepancia`s (relación N:M vía tabla pivot)
- La llegada de una respuesta dispara el evento WebSocket `RESPUESTA_PROVEEDOR`
- No hay funcionalidad de borrado expuesta; el estado `CERRADA` marca la conclusión del proceso

---

### 8. `AuditTrail`

Registro append-only de todos los eventos del sistema. **Nunca se modifica ni se elimina.**

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `usuario_id` | UUID v7 | FK → Usuario, NULL | Usuario que generó el evento (NULL para eventos de sistema) |
| `evento` | VARCHAR(100) | NOT NULL | Tipo de evento (ver catálogo abajo) |
| `entidad_tipo` | VARCHAR(50) | NULL | Tipo de entidad afectada (ej: `Portafolio`, `Cotizacion`) |
| `entidad_id` | UUID v7 | NULL | ID de la entidad afectada |
| `descripcion` | TEXT | NOT NULL | Descripción detallada del evento |
| `metadata` | JSONB | NULL | Datos adicionales del evento (estado anterior/nuevo, IPs, etc.) |
| `ip_origen` | INET | NULL | IP del usuario que generó el evento |
| `created_at` | TIMESTAMP | NOT NULL | Timestamp del evento (UTC) |

**Catálogo de eventos registrados (`evento` field):**

| Categoría | Evento |
|-----------|--------|
| **Seguridad** | `LOGIN_EXITOSO`, `LOGIN_FALLIDO`, `LOGOUT`, `ACCESO_DENEGADO` |
| **Seguridad** | `PDF_ADVERSARIAL_DETECTADO`, `SANITIZACION_APLICADA` |
| **Portafolio** | `PORTAFOLIO_CREADO`, `PORTAFOLIO_ESTADO_CAMBIADO` |
| **Análisis** | `ANALISIS_INICIADO`, `PDF_PROCESADO`, `VARIABLE_EXTRAIDA_GUARDADA` |
| **HITL** | `VARIABLE_REVISADA`, `REVISION_APROBADA` |
| **Aclaración** | `ACLARACION_ENVIADA`, `RESPUESTA_PROVEEDOR_RECIBIDA`, `ACLARACION_CERRADA` |
| **Reporte** | `REPORTE_GENERADO`, `REPORTE_DESCARGADO` |
| **Admin** | `ROL_ASIGNADO`, `CONFIGURACION_CAMBIADA` |

**Restricciones de integridad:**
- `INSERT` solamente; prohibidos `UPDATE` y `DELETE` a nivel de aplicación
- Se recomienda protección adicional a nivel PostgreSQL (trigger que rechace UPDATE/DELETE)
- `created_at` es inmutable; usa `DEFAULT NOW()` en BD, no modificable por aplicación

---

## Diagrama de Relaciones

```
Usuario ──────────────────────────────────────────────────────┐
    │                                                          │
    │ (analista_id)                                            │ (revisado_por)
    ▼                                                          ▼
Portafolio ────────────────── Cotizacion ──────── VariableExtraida
    (1:N)                          (1:N)               │  (1:N)
                                    │                  │
                                    │ (1:N)            │ (variable_extraida_id)
                                    ▼                  ▼
                               DocumentoPDF        Discrepancia ────────── Aclaracion
                                                                  (N:M via pivot)

AuditTrail ← registra eventos de todas las entidades (no FK directa, referencia por UUID)
```

---

## Tabla Pivot: `DiscrepanciaAclaracion`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `discrepancia_id` | UUID v7 (FK) | Discrepancia vinculada |
| `aclaracion_id` | UUID v7 (FK) | Aclaración que la aborda |
| `created_at` | TIMESTAMP | Fecha de vinculación |

---
