# Domain Entities — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Functional Design
> **Versión**: v2 (revisión 2026-05-28 — alineación con schema SAB real)
> **Scope MVP**: solo BIENES (el catálogo dinámico permite agregar SERVICIOS/SOFTWARE en futuras iteraciones sin cambios arquitectónicos)
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

---

## Nota de Alineación con U-01 y U-02

Las entidades persistentes del dominio (`VariableExtraida`, `Discrepancia`, `Cotizacion`, `DocumentoPDF`, `Portafolio`, `AuditTrail`) fueron definidas en U-01 Fundación. Este documento documenta:

1. Las **extensiones** que U-03 aplica sobre entidades existentes (nuevos campos, nuevos estados ENUM).
2. Las **nuevas entidades** que U-03 introduce: `LlamadaLLM` (auditoría CISO) y `CatalogoVariable` (catálogo dinámico Admin).
3. Las **entidades requeridas como delta en U-01/U-02** que U-03 consume: `SolicitudCotizacion`, `ItemSolicitud`, `ItemCotizado` (espejo SAB).
4. Las **estructuras de datos transitorias** (Pydantic / in-memory) propias de U-03.

> Los deltas para U-01, U-02 y U-06 están consolidados en [`cross-unit-deltas.md`](./cross-unit-deltas.md).

---

## 1. Entidades Persistentes Nuevas (creadas por U-03)

### 1.1 `CatalogoVariable` — Catálogo dinámico de variables Admin-editable

Tabla persistente. Reemplaza el catálogo estático en código de la versión anterior. Admin la mantiene desde la UI (extensión de US-22).

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `nombre` | VARCHAR(80) | NOT NULL, UNIQUE per `tipo_adquisicion` | Identificador snake_case (ej: `vigencia_garantia`) |
| `descripcion_llm` | TEXT | NOT NULL | Texto que se incluye en el prompt del LLM como descripción de la variable |
| `tipo_dato` | ENUM | NOT NULL | `MONETARIO \| NUMERICO \| FECHA \| TEXTO \| BOOLEANO` |
| `criticidad` | ENUM | NOT NULL | `CRITICA \| ALTA \| MEDIA \| BAJA` |
| `tipo_adquisicion` | ENUM | NOT NULL | `BIENES \| SERVICIOS \| SOFTWARE` (MVP: solo BIENES activas) |
| `tolerancia_numerica` | DECIMAL(10,4) | NULLABLE | Tolerancia para comparación numérica (solo `tipo_dato = NUMERICO/MONETARIO`) |
| `es_cruce_sab` | BOOLEAN | NOT NULL, DEFAULT FALSE | Si la variable se cruza con datos del SAB (raro — la mayoría son solo PDF) |
| `aplica_a_tipo_documento` | TEXT[] | NULLABLE | Lista de tipos de documento donde se intenta extraer (ej: `['COTIZACION_FORMAL', 'ANEXO_GENERICO']`). NULL = todos |
| `activo` | BOOLEAN | NOT NULL, DEFAULT TRUE | Soft-delete sin borrar registro |
| `created_at` | TIMESTAMP | NOT NULL | UTC |
| `created_by` | UUID v7 | NOT NULL, FK → Usuario | Admin que creó la variable |
| `updated_at` | TIMESTAMP | NOT NULL | UTC |
| `updated_by` | UUID v7 | NOT NULL, FK → Usuario | Admin que modificó por última vez |

**Lectura en C-11 GatewayLLM** (BR-U03-19, revisado por NFR Q5.3): el catálogo se lee directamente de BD en cada extracción (sin cache). Cambios desde Admin se reflejan de inmediato.

**Variable especial `precio_total_cotizacion`**:
- `tipo_dato = MONETARIO`, `criticidad = CRITICA`, `es_cruce_sab = TRUE`
- Se cruza contra `Σ(cotizacion_item.valor_total_item_con_iva)` del SAB (anti-fraude)
- Es la **única** variable LLM con cruce SAB en el MVP

**Variables BIENES iniciales del MVP** (seed data):

| Nombre | Tipo | Criticidad | Notas |
|---|---|---|---|
| `precio_total_cotizacion` | MONETARIO | CRITICA | Cruce con SAB (anti-fraude) |
| `costos_adicionales` | MONETARIO | ALTA | Solo extracción |
| `descuentos_aplicables` | MONETARIO | BAJA | Solo extracción |
| `forma_pago` | TEXTO | ALTA | Solo extracción |
| `plazo_pago_dias` | NUMERICO | ALTA | Solo extracción |
| `vigencia_oferta` | FECHA | CRITICA | Solo extracción + inconsistencia interna |
| `fecha_entrega` | FECHA | CRITICA | Solo extracción + inconsistencia interna |
| `tipo_garantia` | TEXTO | ALTA | Solo extracción |
| `monto_garantia` | MONETARIO | CRITICA | Solo extracción |
| `cobertura_garantia` | TEXTO | ALTA | Solo extracción |
| `vigencia_garantia` | FECHA | CRITICA | Solo extracción + inconsistencia interna |
| `garantia_postventa` | TEXTO | MEDIA | Solo extracción |
| `penalidad_retraso_porcentaje` | NUMERICO | ALTA | Solo extracción |
| `monto_maximo_penalidad` | MONETARIO | ALTA | Inconsistencia interna |
| `clausula_confidencialidad` | TEXTO | MEDIA | Solo extracción |
| `condiciones_especiales` | TEXTO | MEDIA | Solo extracción |
| `exclusiones_principales` | TEXTO | MEDIA | Solo extracción |
| `responsable_tecnico` | TEXTO | MEDIA | Solo extracción |
| `tiempo_respuesta_soporte` | TEXTO | MEDIA | Solo extracción |
| `nivel_servicio` | TEXTO | MEDIA | Solo extracción |
| `marca` | TEXTO | MEDIA | Específica BIENES |
| `modelo` | TEXTO | MEDIA | Específica BIENES |
| `especificaciones_tecnicas` | TEXTO | ALTA | Específica BIENES (típicamente en FICHA_TECNICA) |
| `lugar_entrega` | TEXTO | MEDIA | Específica BIENES |

> 24 variables MVP. El Admin puede agregar/modificar/desactivar desde la UI sin redeploy.

---

### 1.2 `LlamadaLLM` — Auditoría CISO de cada llamada al LLM

Tabla persistente (sin cambios respecto a v1).

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID v7 | PK, NOT NULL | Identificador único |
| `documento_pdf_id` | UUID v7 | FK → DocumentoPDF, NOT NULL | PDF procesado |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL | Cotización |
| `tipo_llamada` | ENUM | NOT NULL | `PRIMARIA \| FALLBACK \| CLASIFICACION` |
| `modelo` | VARCHAR(50) | NOT NULL | `claude-sonnet-4-6` \| `claude-opus-4-7` |
| `s3_key` | VARCHAR(1000) | NOT NULL | `portafolios/{pid}/llm_calls/{call_id}.json` |
| `prompt_tokens` | INTEGER | NOT NULL | Tokens del prompt |
| `completion_tokens` | INTEGER | NOT NULL | Tokens de la respuesta |
| `latencia_ms` | INTEGER | NOT NULL | Latencia API |
| `success` | BOOLEAN | NOT NULL | TRUE si la llamada retornó respuesta válida |
| `tipo_documento_clasificado` | ENUM | NULLABLE | Resultado de clasificación (si la llamada incluyó clasificación) |
| `created_at` | TIMESTAMP | NOT NULL | UTC |

**Objeto S3** (`portafolios/{pid}/llm_calls/{call_id}.json`):
```json
{
  "system_prompt": "...",
  "user_message": "...",
  "tool_definition": { "name": "clasificar_y_extraer", "input_schema": {...} },
  "response": { "tool_use": {...} },
  "model": "claude-sonnet-4-6",
  "prompt_tokens": 4500,
  "completion_tokens": 800,
  "latencia_ms": 2300,
  "timestamp": "2026-05-28T14:22:00Z"
}
```

---

## 2. Entidades Persistentes Existentes — Extensiones por U-03

### 2.1 `VariableExtraida` (definida en U-01 §5)

U-03 es la única unidad que crea registros `VariableExtraida`. Solo variables a **nivel Cotización** (los ítems vienen estructurados del SAB en `ItemCotizado`).

| Campo | Escrito por U-03 | Notas |
|---|:---:|---|
| `id` | ✅ | `generate_uuid_v7()` |
| `cotizacion_id` | ✅ | FK |
| `documento_id` | ✅ | FK → DocumentoPDF (el documento ganador de la consolidación multi-PDF) |
| `nombre` | ✅ | Del catálogo `CatalogoVariable.nombre` |
| `valor` | ✅ | Extraído por el LLM. `"NO_ENCONTRADO"` si no aparece (confianza 1.0) |
| `confianza` | ✅ | DECIMAL(5,4). Self-reported por LLM. Ajustado a `min(actual, 0.3)` si fuzzy falla |
| `referencia_fuente` | ✅ | JSON: `{"documento_id", "pagina", "seccion", "fragmento_texto"}` |
| `analisis` | ✅ | Razonamiento del LLM |
| `requiere_revision` | ✅ | TRUE si `confianza < 0.60` OR fuzzy ratio < 0.85 |
| `fuente_consolidacion` | ✅ | Nuevo: `{tipo_documento_ganador, score_prioridad, candidatos_alternativos: [...]}`. Soporta auditoría de por qué se eligió esa fuente cuando hay varios PDFs |
| `valor_revisado`, `revisado_por`, `revisado_en` | ❌ | Lo escribe U-04 (HITL) |

### 2.2 `Discrepancia` (definida en U-01 §6)

| Campo | Notas |
|---|---|
| `tipo_discrepancia` | ENUM extendido: `VALOR_DIFERENTE \| AUSENTE_EN_PDF \| SOLO_EN_PDF \| INCONSISTENCIA_INTERNA \| INCONSISTENCIA_ARITMETICA \| MONEDA_DIFERENTE` |
| `nombre_campo_sab` | NULLABLE — solo para discrepancias con cruce SAB |
| `item_cotizado_id` | UUID NULLABLE (NUEVO) — referencia al `ItemCotizado` cuando la discrepancia es aritmética a nivel ítem |
| `severidad` | Calculada por reglas (ver `business-rules.md` BR-U03-18) |

> El tipo `INCONSISTENCIA_ARITMETICA` se introduce en esta versión para separar los errores aritméticos sobre `cotizacion_item` (IVA, totales) de las inconsistencias entre variables LLM (`INCONSISTENCIA_INTERNA`).

### 2.3 `Cotizacion` (definida en U-01 §3)

| Campo | Escrito por | Notas |
|---|---|---|
| `cotizacion_id_sab` | U-02 (NUEVO) | BIGINT — `cotizacion.id` en BD SAB |
| `solicitud_cotizacion_id` | U-02 (NUEVO) | FK → `SolicitudCotizacion` (entidad nueva) |
| `proveedor_nit` | U-02 (NUEVO) | VARCHAR — snapshot de `proveedor.numero_documento_identificacion` |
| `proveedor_razon_social` | U-02 (NUEVO) | TEXT — snapshot de `proveedor.nombre_razon_social` |
| `estado_sab` | U-02 (NUEVO) | ENUM espejo: `EN_PROCESO \| EN_REVISION \| EN_CORRECCION \| ACEPTADA_PARA_ESTUDIO \| SELECCIONADA \| NO_SELECCIONADA` |
| `estado_analisis` | U-03 | ENUM extendido: `PENDIENTE \| EN_PROCESO \| COMPLETADO \| ERROR \| OMITIDA` |
| `puntaje_global` | U-03 | DECIMAL(5,2) calculado al completar análisis |
| `requiere_revision_hitl` | U-03 | TRUE si ≥1 VariableExtraida con `requiere_revision = TRUE` |

### 2.4 `DocumentoPDF` (definida en U-01 §4)

| Campo | Escrito por | Notas |
|---|---|---|
| `documento_id_sab` | U-02 (NUEVO) | BIGINT — `cotizacion_documento.id` en BD SAB |
| `nombre_archivo_original` | U-02 (NUEVO) | TEXT — `cotizacion_documento.nombre`. **No es confiable para clasificar** |
| `s3_bucket_origen_sab` | U-02 (NUEVO) | VARCHAR — `cotizacion_documento.bucket` (referencia) |
| `s3_key_origen_sab` | U-02 (NUEVO) | VARCHAR — `cotizacion_documento.llave` (referencia) |
| `motivo_error` | U-02 (NUEVO) | TEXT NULLABLE — descrito por U-02 si sanitización falla |
| `procesado_por_ia` | U-03 | TRUE al completar la extracción |
| `tipo_documento_clasificado` | U-03 (NUEVO) | ENUM: `COTIZACION_FORMAL \| FICHA_TECNICA \| CERTIFICADO \| CARTA_PRESENTACION \| ANEXO_GENERICO \| NO_RECONOCIDO` |
| `confianza_clasificacion` | U-03 (NUEVO) | DECIMAL(5,4) — self-reported por LLM |
| `extraccion_omitida` | U-03 (NUEVO) | BOOLEAN — TRUE si `NO_RECONOCIDO` o el LLM rechazó extraer |

### 2.5 `Portafolio` (definida en U-01 §2)

| Campo | Escrito por | Notas |
|---|---|---|
| `solicitud_bienes_version` | U-02 (NUEVO) | INTEGER — parte de la composite key del SAB |
| `solicitud_bienes_fecha_version` | U-02 (NUEVO) | TIMESTAMP — parte de la composite key |
| `solicitud_bienes_id` | U-02 (NUEVO) | BIGINT — parte de la composite key |
| `radicado_referencia` | U-02 (NUEVO) | NUMERIC(10,2) NULLABLE — `radicado.radicado` para display |
| Constraint | UNIQUE (version, fecha_version, id) | Evita Portafolios duplicados para el mismo `solicitud_bienes` |

### 2.6 `AuditTrail` — Eventos generados por U-03

| Evento | Trigger | `metadata` JSONB |
|---|---|---|
| `ANALISIS_INICIADO_U03` | Pipeline inicia | `{portafolio_id, n_solicitudes_cotizacion, n_cotizaciones, n_pdfs_limpios}` |
| `DOCUMENTO_CLASIFICADO` | LLM clasifica un PDF | `{documento_pdf_id, tipo_documento_clasificado, confianza_clasificacion}` |
| `EXTRACCION_COMPLETADA` | MotorExtraccion termina un PDF | `{documento_pdf_id, cotizacion_id, n_variables_extraidas, n_variables_baja_confianza, modelo_usado}` |
| `CONSOLIDACION_VARIABLE` | Variable consolidada de multi-PDF | `{cotizacion_id, nombre_variable, documento_ganador_id, n_candidatos}` |
| `VARIABLE_EXTRAIDA_GUARDADA` | VariableExtraida persistida | `{variable_nombre, confianza, requiere_revision}` |
| `CRUCE_ARITMETICO_FALLIDO` | Validación aritmética ítem falla | `{item_cotizado_id, check, valor_esperado, valor_actual, diferencia}` |
| `CRUCE_COMPLETADO` | MotorCruce termina una cotización | `{cotizacion_id, n_discrepancias, n_alta, n_critica, n_aritmeticas}` |
| `ANALISIS_COMPLETADO` | Pipeline termina | `{portafolio_id, total_variables, total_escalamientos, total_errores, total_discrepancias}` |
| `COTIZACION_OMITIDA` | Cotización sin PDFs LIMPIOS o sin items | `{cotizacion_id, motivo: "sin_pdfs_limpios" \| "sin_items_sab"}` |
| `LLM_INVOCADO` | Llamada LLM completada | `{modelo, tipo_llamada, prompt_tokens, completion_tokens, latencia_ms, success, s3_key}` |
| `CATALOGO_VARIABLE_MODIFICADO` | Admin edita CatalogoVariable | `{accion: CREATE \| UPDATE \| DEACTIVATE, variable_nombre, before, after, admin_id}` |

---

## 3. Entidades Persistentes Requeridas como Delta de U-01

Estas entidades NO las crea U-03, pero U-03 las consume. Su definición persistente debe agregarse a U-01 (ver `cross-unit-deltas.md`).

### 3.1 `SolicitudCotizacion` (NUEVA, espejo del SAB)

| Campo | Tipo | Restricciones |
|---|---|---|
| `id` | UUID v7 | PK |
| `portafolio_id` | UUID v7 | FK → Portafolio, NOT NULL |
| `solicitud_cotizacion_id_sab` | BIGINT | NOT NULL, UNIQUE per portafolio |
| `codigo_sab` | VARCHAR(150) | NOT NULL |
| `estado_sab` | ENUM | NOT NULL: `PENDIENTE \| ABIERTA \| CERRADA` |
| `fecha_apertura` | TIMESTAMP | NULLABLE |
| `fecha_cierre` | TIMESTAMP | NULLABLE |
| `categoria_ids_sab` | INTEGER[] | NOT NULL — array de IDs de categoría SAB que cubre |

### 3.2 `ItemSolicitud` (NUEVA, espejo de `solicitud_item` SAB)

| Campo | Tipo | Restricciones |
|---|---|---|
| `id` | UUID v7 | PK |
| `portafolio_id` | UUID v7 | FK → Portafolio, NOT NULL (1×Portafolio normalizado) |
| `solicitud_cotizacion_id` | UUID v7 | FK → SolicitudCotizacion, NOT NULL (denormalizado para queries) |
| `solicitud_item_id_sab` | BIGINT | NOT NULL, UNIQUE |
| `numero_item` | INTEGER | NOT NULL — `consecutivo_solicitud_bienes` del SAB |
| `categoria_id_sab` | INTEGER | NOT NULL — FK suelta, sin entidad propia |
| `cantidad_solicitada` | INTEGER | NOT NULL |
| `unidad_medida` | VARCHAR(50) | NULLABLE |
| `descripcion` | TEXT | NULLABLE |
| `codigo_srf` | VARCHAR(20) | NULLABLE |

### 3.3 `ItemCotizado` (NUEVA, espejo de `cotizacion_item` SAB)

| Campo | Tipo | Restricciones |
|---|---|---|
| `id` | UUID v7 | PK |
| `cotizacion_id` | UUID v7 | FK → Cotizacion, NOT NULL |
| `item_solicitud_id` | UUID v7 | FK → ItemSolicitud, NOT NULL |
| `cotizacion_item_id_sab` | BIGINT | NULLABLE — NULL si el proveedor no cotizó ese ítem |
| `cotizado` | BOOLEAN | NOT NULL — FALSE si no hay row en `cotizacion_item` SAB |
| `valor_unidad_sin_iva` | DECIMAL(15,2) | NULLABLE (NULL si `cotizado=FALSE`) |
| `valor_unidad_con_iva` | DECIMAL(15,2) | NULLABLE |
| `valor_total_item_sin_iva` | DECIMAL(15,2) | NULLABLE |
| `valor_total_item_con_iva` | DECIMAL(15,2) | NULLABLE |
| `porcentaje_iva` | DECIMAL(5,2) | NULLABLE — snapshot de `porcentaje_iva.porcentaje` |
| `cantidad_cotizada` | INTEGER | NULLABLE |
| `unidad` | VARCHAR(40) | NULLABLE |
| `descripcion_cotizada` | TEXT | NULLABLE |
| `tiempo_entrega_dias` | INTEGER | NULLABLE |
| `observacion` | VARCHAR(300) | NULLABLE |
| `item_solicitado_igual_cotizado` | BOOLEAN | NOT NULL DEFAULT FALSE — espejo SAB |
| `fecha_cotizacion_sab` | DATE | NOT NULL — para trazabilidad de partition SAB |

---

## 4. Estructuras de Datos Transitorias (Pydantic / In-Memory)

### 4.1 `ContextoExtraccion`

```python
class TipoAdquisicion(str, Enum):
    BIENES = "BIENES"
    # SERVICIOS, SOFTWARE pendientes de futuras iteraciones

class ContextoExtraccion:
    documento_pdf_id: UUID
    cotizacion_id: UUID
    portafolio_id: UUID
    nombre_proveedor: str
    tipo_adquisicion: TipoAdquisicion  # MVP: siempre BIENES
    texto_pdf: str            # Texto nativo o OCR companion desde S3
    tokens_spotlighting: list[str]  # Tokens del SanitizadorZeroTrust (U-02)
    catalogo_variables: list[CatalogoVariable]  # Cache C-11, filtrado por tipo_adquisicion + activo
```

### 4.2 `ResultadoClasificacionExtraccion`

Respuesta parseada del tool use de Claude (clasificación + extracción en una sola llamada).

```python
class TipoDocumentoClasificado(str, Enum):
    COTIZACION_FORMAL = "COTIZACION_FORMAL"
    FICHA_TECNICA = "FICHA_TECNICA"
    CERTIFICADO = "CERTIFICADO"
    CARTA_PRESENTACION = "CARTA_PRESENTACION"
    ANEXO_GENERICO = "ANEXO_GENERICO"
    NO_RECONOCIDO = "NO_RECONOCIDO"

class VariableExtraidaLLM:
    nombre: str
    valor: str  # "NO_ENCONTRADO" si no aparece
    confianza: float  # 0.0–1.0, self-reported
    referencia_fuente: dict  # {pagina, seccion, fragmento_texto}
    analisis: str

class ResultadoClasificacionExtraccion:
    tipo_documento_clasificado: TipoDocumentoClasificado
    confianza_clasificacion: float
    variables: list[VariableExtraidaLLM]  # Vacío si tipo == CERTIFICADO/CARTA/NO_RECONOCIDO
    modelo_usado: str
    prompt_tokens: int
    completion_tokens: int
    latencia_ms: int
    llm_call_id: UUID
```

### 4.3 `CandidatoConsolidacion` y `ResultadoConsolidacion`

Para una misma variable extraída de varios PDFs de la misma cotización:

```python
PRIORIDAD_TIPO_DOCUMENTO = {
    TipoDocumentoClasificado.COTIZACION_FORMAL: 1.0,
    TipoDocumentoClasificado.FICHA_TECNICA:     0.9,
    TipoDocumentoClasificado.ANEXO_GENERICO:    0.7,
    # CERTIFICADO, CARTA_PRESENTACION, NO_RECONOCIDO: no aportan variables
}

class CandidatoConsolidacion:
    documento_pdf_id: UUID
    tipo_documento: TipoDocumentoClasificado
    variable: VariableExtraidaLLM
    score_consolidacion: float  # prioridad_tipo × confianza_variable

class ResultadoConsolidacion:
    nombre_variable: str
    ganador: CandidatoConsolidacion
    candidatos_alternativos: list[CandidatoConsolidacion]  # Para auditoría
```

### 4.4 `ResultadoValidacionFuzzy`

```python
class ResultadoValidacionFuzzy:
    variable_nombre: str
    fragmento_llm: str
    ratio_similitud: float  # rapidfuzz 0.0–1.0
    fragmento_encontrado: str | None
    es_valido: bool  # ratio >= 0.85
    pagina_verificada: int | None
```

### 4.5 `ResultadoCruce` y `ValidacionAritmetica`

```python
class ValidacionAritmetica:
    item_cotizado_id: UUID
    check: str  # "iva_unitario" | "total_unitario" | "total_item" | "iva_valido" | "cantidad_rfq"
    valor_esperado: Decimal
    valor_actual: Decimal
    diferencia_absoluta: Decimal
    tolerancia: Decimal
    paso: bool
    severidad_si_falla: str  # "ALTA" | "MEDIA" | "CRITICA"

class DiscrepanciaDetectada:
    nombre_campo_sab: str | None  # NULL si es aritmetica
    item_cotizado_id: UUID | None  # NULL si es nivel cotización
    variable_extraida_id: UUID | None
    valor_sab: str
    valor_pdf: str | None
    tipo_discrepancia: str  # VALOR_DIFERENTE | AUSENTE_EN_PDF | SOLO_EN_PDF | INCONSISTENCIA_INTERNA | INCONSISTENCIA_ARITMETICA | MONEDA_DIFERENTE
    severidad: str
    diferencia_porcentual: float | None

class ResultadoCruce:
    cotizacion_id: UUID
    discrepancias: list[DiscrepanciaDetectada]
    validaciones_aritmeticas: list[ValidacionAritmetica]
    tiene_inconsistencia_interna: bool
    cruce_total_pdf_vs_sab: dict  # {valor_pdf, valor_sab, diferencia, paso}
```

### 4.6 `EstadoProgresoPipeline`

Publicado por WebSocket durante U-03 (BR-U03-26).

```python
class EstadoProgresoPipeline:
    # EXTRACCION_INICIADA
    portafolio_id: UUID
    n_solicitudes_cotizacion: int
    n_cotizaciones: int
    n_pdfs_a_procesar: int

    # EXTRACCION_PROGRESO
    cotizacion_id: UUID
    proveedor_razon_social: str
    documento_id: UUID
    tipo_documento_clasificado: str
    pdfs_procesados: int
    pdfs_totales: int
    variables_extraidas: int
    variables_baja_confianza: int

    # ANALISIS_COMPLETADO
    total_discrepancias: int
    total_aritmeticas: int
    total_escalamientos: int
    total_errores: int
    cotizaciones_con_revision: list[UUID]
```

---

## 5. Diagrama de Relaciones de U-03

```
CatalogoVariable (BD, Admin-editable)
    │
    ▼ (lectura directa BD, sin cache)
C-11 GatewayLLM  ◄── LLMProvider abstracto, MVP solo AnthropicProvider
    │
DocumentoPDF (LIMPIO, de U-02) ──────────────────────────────────────┐
    │                                                                 │
    │ ContextoExtraccion                                              │
    ▼                                                                 │
LLM clasifica + extrae (una sola llamada, tool use)                  │
    │                                                                 │
    ├─ tipo_documento_clasificado → DocumentoPDF.tipo_documento_*    │
    │                                                                 │
    ├─ Variables candidatas (segun tipo_documento)                   │
    │                                                                 │
    └─ LlamadaLLM × 1-2 (BD + S3 audit CISO)                          │
                                                                      │
Por cada Cotización con múltiples DocumentoPDF:                      │
    Consolidación multi-PDF (prioridad × confianza)                   │
        │                                                             │
        ▼                                                             │
    VariableExtraida × N (única por variable, con fuente_consolidacion)
        │                                                             │
        ├─ Fuzzy validation post-LLM (ratio >= 0.85)                  │
        │     │                                                       │
        │     └─ confianza = min(actual, 0.3), requiere_revision=TRUE │
        │                                                             │
        └─ Si requiere_revision=TRUE → ESCALAMIENTO_DETECTADO         │
                                                                      │
ItemCotizado × J (datos SAB, sin LLM)  ◄──────────────────────────────┤
    │                                                                 │
    ▼                                                                 ▼
C-04 MotorCruce
    │
    ├─ Cruce precio_total_cotizacion (LLM) vs Σ ItemCotizado SAB
    │   └─ Discrepancia INCONSISTENCIA_ARITMETICA si Δ > 1 COP
    │
    ├─ Validaciones aritméticas por ítem (sobre SAB, sin LLM)
    │   ├─ IVA: valor_iva ≈ precio_sin_iva × porcentaje / 100
    │   ├─ Total unitario, total ítem
    │   ├─ % IVA ∈ porcentaje_iva activos
    │   └─ cantidad_cotizada == cantidad_solicitada
    │
    └─ Inconsistencias internas (vigencia_oferta < fecha_entrega...)
       └─ Discrepancia INCONSISTENCIA_INTERNA, severidad CRITICA

Cotizacion
    ├─ estado_analisis: PENDIENTE → EN_PROCESO → COMPLETADO | ERROR | OMITIDA
    ├─ puntaje_global: DECIMAL(5,2)
    └─ requiere_revision_hitl: BOOLEAN

ANALISIS_COMPLETADO (WebSocket) + Email si escalamientos > UMBRAL_EMAIL
```

---

> **Próximo artefacto**: [`business-rules.md`](./business-rules.md) — Reglas de negocio detalladas.
> **Deltas cross-unit**: [`cross-unit-deltas.md`](./cross-unit-deltas.md).
