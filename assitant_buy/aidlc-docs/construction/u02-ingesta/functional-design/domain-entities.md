# Domain Entities — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI

> ⚠️ **DELTAS PENDIENTES — APLICAR ANTES DE CODE GENERATION**
>
> Tras la revisión de U-03 Functional Design (2026-05-28), U-02 absorbe cambios significativos. Documentados en [`../../u03-extraccion/functional-design/cross-unit-deltas.md`](../../u03-extraccion/functional-design/cross-unit-deltas.md) **§2**.
>
> **Resumen de deltas**:
> - ❌ Eliminar `ParserExcelSAB` (los datos estructurados ya están en BD SAB en `cotizacion_item`)
> - `RepositorioSAB`: expandir con 8 queries directas a tablas SAB reales (`solicitud_bienes`, `solicitud_cotizacion`, `cotizacion_item`, `cotizacion_documento`, `porcentaje_iva`, etc.)
> - `GatewaySAB`: reducir a solo descarga de binarios
> - Nuevo endpoint `POST /portafolios/from-sab` invocado desde SAB UI con tripleta composite key
> - Persistencia jerárquica transaccional: Portafolio → SolicitudCotizacion → ItemSolicitud → Cotizacion → ItemCotizado → DocumentoPDF
> - Nuevo estado `EXCEL_INVALIDO` reemplazado por validación BD `SIN_ITEMS`
>
> **Aplicación**: al iniciar Code Generation de U-02 (paso 1 del plan).
>
> Durante el diseño de U-04..U-07, asumir el modelo refinado.

---

## Nota de Alineación con U-01

Las entidades persistentes de este módulo (`DocumentoPDF`, `Cotizacion`, `Portafolio`, `AuditTrail`) fueron definidas en U-01 Fundación. Este documento documenta:
1. Las **extensiones o restricciones** aplicadas por U-02 sobre esas entidades.
2. Las **estructuras de datos transitorias** (in-memory / schema Pydantic) propias de U-02 que no se persisten directamente en la BD.
3. Las **columnas de la BD** que U-02 escribe por primera vez (las que U-01 definió pero que solo U-02 popula).

---

## 1. Entidades Persistentes — Escritas por U-02

### 1.1 `DocumentoPDF` (definida en U-01 §4)

U-02 es la **única unidad que crea** registros `DocumentoPDF` y actualiza el campo `estado_sanitizacion`. Con la inclusión del OCR en U-02, también escribe `es_escaneado` y `texto_ocr_s3_key`.

| Campo | Escrito por U-02 | Valor inicial | Valores finales posibles |
|-------|:---:|---|---|
| `id` | ✅ | `generate_uuid_v7()` | — |
| `cotizacion_id` | ✅ | ID de `Cotizacion` creada en el mismo pipeline | — |
| `nombre_archivo` | ✅ | Nombre original del archivo en SAB | — |
| `ruta_almacenamiento` | ✅ | S3 key: `portafolios/{pid}/pdfs/{pdf_id}.pdf` | — |
| `tamano_bytes` | ✅ | Calculado de los bytes crudos recibidos | — |
| `hash_sha256` | ✅ | `sha256(pdf_bytes).hexdigest()` | — |
| `estado_sanitizacion` | ✅ | `PENDIENTE` | `LIMPIO` / `ADVERSARIAL` / `ERROR` |
| `es_escaneado` | ✅ | `FALSE` (se detecta en el pipeline) | `TRUE` si no tiene capa de texto nativa |
| `texto_ocr_s3_key` | ✅ | `None` | `portafolios/{pid}/ocr/{pdf_id}.txt` (solo si `es_escaneado=TRUE` y OCR exitoso) |
| `procesado_por_ia` | ✅ (default) | `FALSE` | `TRUE` lo escribe U-03 |
| `created_at` / `updated_at` | ✅ | `NOW()` | — |

> **Nota sobre `texto_ocr_s3_key`**: El texto OCR se almacena como archivo companion en S3 (no en la BD) para evitar inflar la tabla con grandes TEXT. U-03 descarga el `.txt` desde S3 reutilizando el OCR ya calculado, sin llamar a Document AI de nuevo.

#### Ciclo de vida de `estado_sanitizacion` en U-02

```
PENDIENTE
    │
    ▼ ETAPA 0: RT4 — Validación de estructura (primero, antes de gastar en OCR)
    │
    ├─ PDF corrupto, inválido o > PDF_MAX_SIZE_MB → ERROR
    │
    ▼ ETAPA 1: Detección de capa de texto
    │
    ├─ Tiene texto nativo (pypdf extrae texto) → texto_nativo
    │
    └─ Sin texto nativo (PDF escaneado / imagen) → es_escaneado = TRUE
           │
           ▼ GatewayDocumentAI.ocr(pdf_bytes) → texto_ocr
           │
           ├─ OCR falla → ERROR (motivo: "ocr_fallido")
           └─ OCR OK → Upload texto_ocr a S3 como companion file
                    DocumentoPDF.texto_ocr_s3_key = "portafolios/{pid}/ocr/{pdf_id}.txt"
    │
    ▼ ETAPA 2: RT1 — Metadatos maliciosos (sobre estructura PDF)
    │
    ├─ JavaScript, auto-actions, URIs externos detectados → ERROR
    │
    ▼ ETAPA 3: RT2 — Inyección de prompt (sobre texto_nativo o texto_ocr)
    │
    ├─ Patrones de instrucción al LLM detectados → ADVERSARIAL
    │
    ▼ ETAPA 4: RT3 — Contenido ofuscado (sobre texto_nativo o texto_ocr)
    │
    ├─ Texto invisible, homoglíficos, capas ocultas detectados → ADVERSARIAL
    │
    ▼ ETAPA 5: Spotlighting (delimitadores dinámicos sobre el texto final)
    │
    └─ Todas las etapas OK → LIMPIO
```

**Regla crítica**: Solo los PDFs en estado `LIMPIO` pasan al `MotorExtraccion` (U-03). Los PDFs `ADVERSARIAL` y `ERROR` quedan bloqueados permanentemente (*fail closed*).

---

### 1.2 `Cotizacion` — Campos escritos por U-02 (definida en U-01 §3)

U-02 **crea** el registro `Cotizacion` local a partir de los datos leídos del SAB.

| Campo | Escrito por U-02 | Notas |
|-------|:---:|---|
| `id` | ✅ | UUID v7 local (distinto del ID en SAB) |
| `portafolio_id` | ✅ | FK al `Portafolio` creado en el mismo job |
| `nombre_proveedor` | ✅ | Leído del SAB read-only |
| `email_proveedor` | ✅ | Leído del SAB (puede ser NULL si SAB no lo provee) |
| `numero_oferta` | ✅ | Leído del SAB |
| `estado_analisis` | ✅ (default) | Inicial: `PENDIENTE`; U-03 lo actualiza a `EN_PROCESO`/`COMPLETADO`/`ERROR` |
| `puntaje_global` | ❌ | Lo escribe U-03 |
| `requiere_revision_hitl` | ❌ | Lo escribe U-03 |

---

### 1.3 `Portafolio` — Campos escritos por U-02 (definida en U-01 §2)

U-02 **transiciona el estado** del `Portafolio` durante el pipeline de ingesta.

| Transición | Trigger | Acción sobre `Portafolio` |
|---|---|---|
| `PENDIENTE` → `INGESTA` | BackgroundTask inicia conexión a SAB | `actualizar_estado(id, INGESTA)` |
| `INGESTA` → `ANALISIS` | Todas las `Cotizacion` procesadas (al menos parcialmente) | `actualizar_estado(id, ANALISIS)` |
| Cualquier estado → `ERROR` | Fallo catastrófico (SAB inaccesible, BD inaccesible) | `actualizar_estado(id, ERROR)` |

**Nota**: `Portafolio.total_cotizaciones` y `cotizaciones_procesadas` son actualizados por U-02 durante el pipeline como métricas de progreso para el WebSocket.

---

### 1.4 `AuditTrail` — Eventos generados por U-02 (definida en U-01 §8)

U-02 genera los siguientes eventos en la tabla `AuditTrail`. El campo `metadata` (JSONB) contiene el detalle forense de cada regla RT1-RT4 para la auditoría del CISO (US-19).

| Evento | Trigger | `metadata` JSONB |
|---|---|---|
| `SANITIZACION_APLICADA` | PDF procesado correctamente (estado `LIMPIO`) | `{rt1: "OK", rt2: "OK", rt3: "OK", rt4: "OK", pdf_id, tamano_bytes, hash_sha256}` |
| `PDF_ADVERSARIAL_DETECTADO` | PDF bloqueado por RT2 o RT3 (estado `ADVERSARIAL`) | `{regla_fallida: "RT2", tipo_amenaza: "inyeccion_prompt", fragmento_neutralizado: "...", pdf_id, nombre_proveedor}` |
| `SANITIZACION_APLICADA` con estado `ERROR` | PDF bloqueado por RT1 o RT4 (estado `ERROR`) | `{regla_fallida: "RT4", motivo: "pdf_corrupto", pdf_id}` |
| `PORTAFOLIO_ESTADO_CAMBIADO` | Transición de estado del portafolio | `{estado_anterior: "PENDIENTE", estado_nuevo: "INGESTA", proceso_id}` |
| `ANALISIS_INICIADO` | BackgroundTask arranca el pipeline de ingesta | `{proceso_id, analista_id, n_cotizaciones: N}` |

---

## 2. Estructuras de Datos Transitorias (Pydantic / In-Memory)

Estas estructuras existen durante la ejecución del pipeline de ingesta pero **no se persisten directamente** como tablas propias en la BD.

### 2.1 `CotizacionSAB`

Representa los datos leídos del SAB (read-only). Es la fuente de verdad para crear `Cotizacion` local.

```python
class CotizacionSAB:
    id_sab: str                       # ID en el sistema SAB (externo)
    nombre_proveedor: str
    email_proveedor: str | None
    numero_oferta: str | None
    documentos_pdf: list[DocumentoPDFSAB]  # Documentos descargables vía API del SAB
    metadata_adicional: dict               # Datos adicionales del SAB (JSONB)
```

### 2.2 `DocumentoPDFSAB`

> ⚠️ **Corrección arquitectónica**: El sistema SAB expone un endpoint REST para descargar PDFs: `GET /api/v1/cotizacion/documentos/descargar/{documento_id}`. Como el Assistant Buy se despliega en una nube distinta a la del SAB, el acceso directo al filesystem del SAB no es viable. La descarga se realiza vía HTTP sobre este endpoint.

```python
class DocumentoPDFSAB:
    documento_id: str                 # ID del documento en el SAB (usado en la URL de descarga)
    cotizacion_id_sab: str           # ID de la cotización en el SAB a la que pertenece
    nombre_archivo: str               # Nombre original del archivo
    tamano_bytes: int | None          # Tamaño en bytes si el SAB lo provee en el listado
    url_descarga: str                 # URL construida: {SAB_BASE_URL}/api/v1/cotizacion/documentos/descargar/{documento_id}
```

### 2.3 `ResultadoRecuperacionPDF`

Retornado por `IngestaConector.recuperar_pdf()` al terminar la lectura y subida del PDF.

```python
class EstadoRecuperacionPDF(str, Enum):
    OK = "OK"
    CORRUPTO = "CORRUPTO"
    NO_ENCONTRADO = "NO_ENCONTRADO"

class ResultadoRecuperacionPDF:
    estado: EstadoRecuperacionPDF
    pdf_bytes: bytes | None           # Solo cuando estado == OK
    nombre_archivo: str
    tamano_bytes: int
    hash_sha256: str
    ruta_s3: str | None               # Key en S3 tras upload exitoso
```

### 2.4 `MetadataOrigen`

Contexto de procedencia que acompaña al PDF durante todo el pipeline de sanitización.

```python
class MetadataOrigen:
    proveedor_id: UUID                # ID local de la Cotizacion
    nombre_proveedor: str
    cotizacion_id: UUID
    portafolio_id: UUID
    nombre_archivo: str
    proceso_id: str                   # Código del proceso SAB
```

### 2.5 `PDFSanitizado`

Estructura de salida del `SanitizadorZeroTrust`. Es la interfaz de contrato entre U-02 y U-03.

```python
class EstadoSanitizacion(str, Enum):
    LIMPIO = "LIMPIO"
    ADVERSARIAL = "ADVERSARIAL"
    ERROR = "ERROR"

class ElementoAdversarial:
    regla: str                        # "RT1" | "RT2" | "RT3" | "RT4"
    tipo_amenaza: str                 # "javascript_embebido" | "inyeccion_prompt" | "homoglifo" | "pdf_corrupto"
    fragmento_neutralizado: str | None # Fragmento de texto (máx 500 chars) para logging

class PDFSanitizado:
    documento_pdf_id: UUID            # ID del DocumentoPDF persistido
    cotizacion_id: UUID
    portafolio_id: UUID
    contenido_bytes: bytes | None     # Bytes del PDF limpio; None si estado != LIMPIO
    estado: EstadoSanitizacion
    tokens_spotlighting: list[str]    # Delimitadores dinámicos aplicados (ej: ["PDF_abc123"])
    elementos_detectados: list[ElementoAdversarial]  # Vacío si LIMPIO
    metadata_origen: MetadataOrigen
```

### 2.6 `EstadoIngesta`

Estructura de progreso publicada por WebSocket durante el pipeline.

```python
class EstadoIngesta:
    portafolio_id: UUID
    proceso_id: str
    total_cotizaciones: int
    cotizaciones_procesadas: int
    pdfs_total: int
    pdfs_limpios: int
    pdfs_adversariales: int
    pdfs_error: int
    estado_actual: str                # "leyendo_sab" | "sanitizando" | "completado" | "error"
    cotizacion_actual: str | None     # Nombre del proveedor en proceso
```

### 2.7 `TokenSABSession`

Representa la sesión OAuth2 activa del service account contra el Keycloak del SAB. El `GestorTokenSAB` (parte de C-12) mantiene esta estructura en memoria y la renueva proactivamente antes de que expire.

> ⚠️ **Contexto**: El SAB usa Keycloak como Identity Provider. El login del SAB (`POST .../openid-connect/token`) con `grant_type=password` devuelve un `access_token` JWT con `expires_in=1200s` (20 min). El Assistant Buy debe usar este flujo con un **service account de solo consulta** creado exclusivamente para la integración.

```python
class TokenSABSession:
    access_token: str           # JWT Bearer para llamadas a la API del SAB
    refresh_token: str          # Para renovar sin re-autenticar
    expires_at: datetime        # UTC: now() + timedelta(seconds=expires_in)
    refresh_expires_at: datetime # UTC: now() + timedelta(seconds=refresh_expires_in)
    token_type: str             # Siempre "Bearer"
```

---

## 3. Integración con SAB (Read-Only — Dos Canales)

El sistema SAB es externo y se integra mediante **dos canales independientes**:

### 3.1 Canal 1 — BD del SAB (lectura de metadatos)

El `RepositorioSABSQL` accede directamente a la base de datos del SAB con un usuario read-only para obtener metadatos de procesos, cotizaciones y proveedores.

| Tabla SAB | Acceso | Propósito |
|---|---|---|
| `sab_procesos` | SELECT | Validar que el proceso existe y está activo |
| `sab_cotizaciones` | SELECT | Listar cotizaciones del proceso con sus IDs de documentos |
| `sab_proveedores` | SELECT | Obtener nombre y email del proveedor |
| `sab_documentos_pdf` | SELECT | Obtener `documento_id` y `nombre_archivo` de los PDFs de cada cotización |

**Restricción crítica**: El usuario de BD del SAB tiene **permisos exclusivos de SELECT**. No se emiten sentencias INSERT, UPDATE ni DELETE contra la BD del SAB.

### 3.2 Canal 2 — API REST del SAB (descarga de PDFs) + Autenticación Keycloak

La descarga de bytes de PDFs se realiza vía el endpoint HTTP del SAB, usando como autorización el `access_token` obtenido del **Keycloak del SAB**:

```
# 1. Obtener token (OAuth2 ROPC contra Keycloak del SAB)
POST {SAB_KEYCLOAK_URL}/protocol/openid-connect/token
  Body: grant_type=password
        client_id={SAB_CLIENT_ID}
        client_secret={SAB_CLIENT_SECRET}
        username={SAB_SERVICE_USERNAME}
        password={SAB_SERVICE_PASSWORD}

# 2. Descargar PDF con el token obtenido
GET {SAB_BASE_URL}/api/v1/cotizacion/documentos/descargar/{documento_id}
  Authorization: Bearer {access_token}
  Respuesta: application/pdf
```

**Motivación**: El SAB usa Keycloak con autenticación `grant_type=password`. Para que el Assistant Buy acceda de forma no-interactiva (sin UI de login), se crea un **SAB service account** (usuario funcionario de solo consulta) cuyas credenciales se almacenan como secretos en el entorno del Assistant Buy.

**Configuración requerida** (todas variables de entorno / secretos, nunca en código):

| Variable | Descripción |
|---|---|
| `SAB_BASE_URL` | URL base del API SAB (ej: `https://backend.unicauca.edu.co/sab-bienes`) |
| `SAB_KEYCLOAK_URL` | URL base del realm Keycloak (ej: `https://proyunicauca.docxflow.com:8443/realms/unicauca-funcionarios-qa`) |
| `SAB_CLIENT_ID` | `client_id` del cliente Keycloak |
| `SAB_CLIENT_SECRET` | `client_secret` del cliente Keycloak |
| `SAB_SERVICE_USERNAME` | Usuario del service account (funcionario de solo consulta) |
| `SAB_SERVICE_PASSWORD` | Contraseña del service account |
| `SAB_DOWNLOAD_TIMEOUT_S` | Timeout de descarga HTTP en segundos (default: 30) |

**Prerrequisito operacional**: El equipo de TI del SAB debe crear el usuario service account en Keycloak con rol de **solo lectura** (sin permisos de creación, edición ni aprobación de procesos). Esta es una precondición de despliegue, no algo que el Assistant Buy gestione.

---

## 4. Diagrama de Relaciones de U-02

```
SAB (externo, read-only)
    │
    │ RepositorioSABSQL (SELECT only)
    ▼
CotizacionSAB[] ─────────────────────────────────────────────┐
    │                                                          │
    │ IngestaConector.iniciar_ingesta_portafolio()             │
    ▼                                                          ▼
Portafolio (PENDIENTE→INGESTA)          Cotizacion (PENDIENTE) [persistida en BD local]
    │                                          │
    │                                          │ recuperar_pdf() × N PDFs
    │                                          ▼
    │                                    DocumentoPDFSAB
    │                                          │
    │                                          │ GET SAB_API/cotizacion/documentos/descargar/{id}
    │                                          │ (HTTP, cross-cloud) + hash + upload S3
    │                                          ▼
    │                                    DocumentoPDF (PENDIENTE) [persistida en BD local]
    │                                          │
    │                                          │ SanitizadorZeroTrust.sanitizar()
    │                                          ▼
    │                                    RT1 → RT2 → RT3 → RT4 → Spotlighting
    │                                          │
    │                          ┌───────────────┼───────────────┐
    │                          ▼               ▼               ▼
    │                       LIMPIO         ADVERSARIAL       ERROR
    │                          │               │               │
    │                          │         AuditTrail      AuditTrail
    │                          │         ALERTA_SEGURIDAD  SANITIZACION_APLICADA
    │                          ▼
    │                    PDFSanitizado
    │                    (interfaz hacia U-03)
    │
    ▼ (todas las cotizaciones procesadas)
Portafolio (INGESTA→ANALISIS)
    │
    ▼ (señal para U-03)
WebSocket: INGESTA_COMPLETADA
```

---

> **Próximo artefacto**: `business-rules.md` — Reglas de negocio detalladas de U-02.
