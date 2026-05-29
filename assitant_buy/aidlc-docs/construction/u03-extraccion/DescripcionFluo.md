# Descripción del Flujo — Sistema AssistentBuy (v3)

> **Propósito**: Documento de alineación previo a la modificación de los artefactos de Functional Design de U-03.
>
> **v3 (2026-05-28)** consolida la jerarquía real del SAB descubierta tras revisar el schema (`assitant_buy/sab/bd_sab.sql`).
>
> **Scope MVP: solo BIENES** — Servicios y Software quedan fuera del alcance de esta iteración (el catálogo dinámico permite agregarlos luego sin cambios arquitectónicos).
>
> Cambios estructurales:
> - Modelo de 3 niveles: `solicitud_bienes` → `solicitud_cotizacion` (por categoría) → `cotizacion` (por proveedor) → `cotizacion_item`
> - **Eliminación del parser Excel** — todos los datos estructurados ya están en BD SAB
> - Identificación de Portafolio por tripleta `(version, fecha_version, id)` de `solicitud_bienes` (pasada desde SAB)
> - IVA dinámico desde tabla `porcentaje_iva`
> - Sin entidad `Categoria` de nuestro lado (solo FK suelta)
>
> ⚠️ **No es un artefacto AI-DLC oficial.** Una vez aprobado, se aplicarán los cambios correspondientes a `domain-entities.md`, `business-rules.md`, `business-logic-model.md` y se creará `cross-unit-deltas.md` con los deltas para U-01, U-02 y U-06.

---

## 1. Contexto del Negocio

**Actores externos**:
- **Comprador** (ej. Universidad del Cauca) — crea la **Solicitud de Bienes** en SAB, que genera N **Solicitudes de Cotización** (RFQs por categoría).
- **Proveedor** — está registrado para ciertas **categorías**; cuando una RFQ cubre sus categorías, SAB lo invita a cotizar. Sube su Excel (procesado por SAB) + PDFs/anexos.
- **SAB** (Sistema de Adquisiciones de Bienes) — gestiona el ciclo completo de adquisición. BD PostgreSQL + storage S3 + UI propia.

**Actores del sistema AssistentBuy**:
- **Analista** — analiza un Portafolio que cubre una `solicitud_bienes` completa; revisa resultados y escalamientos.
- **Comité de Compras** — consume reporte ranqueado por ítem.
- **Admin** — administra catálogo dinámico de variables y umbrales.
- **CISO** — audita llamadas LLM y flujos de datos.

---

## 2. Jerarquía Real del SAB

```
solicitud_bienes                       ← Solicitud principal del comprador
  PK compuesta: (version, fecha_version, id)
  Identificada externamente por: radicado (numeric)
  │
  ├─ solicitud_item × N                ← Todos los ítems del proceso
  │     ├─ id_categoria                   (cada ítem tiene categoría)
  │     ├─ consecutivo_solicitud_bienes   (= numero_item)
  │     ├─ cantidad, unidad, descripcion
  │     └─ codigo_srf
  │
  ├─ radicado                          ← Código externo amigable (numeric 10,2)
  │
  └─ solicitud_cotizacion × M          ← RFQs por agrupación de categorías
       │  codigo varchar(150) — identificador propio
       │  estado: PENDIENTE | ABIERTA | CERRADA
       │  fecha_apertura, fecha_cierre
       │
       ├─ categoria_solicitud_cotizacion (M2M)  ← Qué categorías cubre esta RFQ
       │
       └─ cotizacion × K               ← Respuestas de proveedores invitados
            │  estado: EN_PROCESO | EN_REVISION | EN_CORRECCION |
            │          ACEPTADA_PARA_ESTUDIO | SELECCIONADA | NO_SELECCIONADA
            │  id_proveedor → proveedor (NIT, razon_social, tipo_regimen_iva...)
            │
            ├─ cotizacion_item × J     ← Items que el proveedor cotizó
            │     │ (PARTITION BY fecha_cotizacion — pruning por año)
            │     ├─ id_solicitud_item (FK al ítem RFQ)
            │     ├─ valor_unidad_sin_iva, valor_unidad_con_iva
            │     ├─ valor_total_item_sin_iva, valor_total_item_con_iva
            │     ├─ id_porcentaje_iva → porcentaje_iva (0%, 5%, 19%, etc.)
            │     ├─ cantidad, unidad, descripcion
            │     ├─ tiempo_entrega, observacion
            │     ├─ item_solicitado_igual_cotizado (bool)
            │     └─ es_seleccionado, razon_seleccion
            │
            └─ cotizacion_documento × L ← PDFs/anexos del proveedor
                  bucket + llave (rutas S3) + nombre + tamano + fecha_carga
```

---

## 3. Modelo de Integración con SAB

### 3.1 BD SAB (solo lectura, conexión directa)

Componente: **`RepositorioSAB`** (existente en application-design, sus queries se expanden).

**Tablas/views consultadas** (todas las nombres confirmados del schema):

| Tabla | Propósito |
|---|---|
| `solicitud_bienes` | Dato raíz del Portafolio (composite key + radicado) |
| `radicado` | Lookup auxiliar por código externo |
| `solicitud_cotizacion` | Lista de RFQs hijas + estado |
| `categoria_solicitud_cotizacion` | M2M para saber qué categoría cubre cada RFQ |
| `solicitud_item` (particionada por año) | Items del proceso (RFQ) |
| `cotizacion` | Respuestas de proveedores con estado + id_proveedor |
| `cotizacion_item` (particionada por año) | Items cotizados con todos los valores monetarios y FK IVA |
| `cotizacion_documento` | Lista de docs con `bucket` + `llave` S3 + metadata |
| `proveedor` | NIT, razón social, régimen IVA |
| `porcentaje_iva` | Tarifas IVA configurables (con flag `activo`) |
| `vista_cuadro_comparativo` | (Opcional, U-05) — referencia comparativa nativa SAB |

**Configuración propuesta** (TBD con DBA SAB):
- Read replica (no BD primaria)
- Usuario read-only (`assistentbuy_reader`)
- Acceso vía VPN o VPC peering
- Credenciales en AWS Secrets Manager con rotación

### 3.2 API REST SAB (OAuth2 Keycloak)

Componente: **`GatewaySAB`** (existente en U-02). **Único uso**: descargar binarios de documentos.

| Endpoint | Propósito |
|---|---|
| `GET /sab-bienes/api/v1/cotizacion/documentos/descargar/{doc_id}` | Descargar PDF/anexo individual |

> **NO** se usa el endpoint `/cotizacion_item/formato/generar` — los datos estructurados se leen directo de `cotizacion_item` en BD.

### 3.3 ❌ `ParserExcelSAB` ELIMINADO

Decisión arquitectónica de v3: SAB ya parsea el Excel internamente y persiste todos los valores en `cotizacion_item`. No hay valor agregado en re-parsearlo.

---

## 4. Origen de los Datos por Categoría

### 4.1 BD SAB → Datos estructurados (sin LLM)

- Jerarquía completa: `solicitud_bienes`, `solicitud_cotizacion`, `cotizacion`, `cotizacion_item`, `solicitud_item`
- Metadatos de documentos (sin descargar binarios)
- Información de proveedores (NIT, razón social)
- Tarifas IVA activas

### 4.2 PDFs y anexos vía REST API (clasificados por contenido en U-03)

El proveedor sube cualquier archivo. SAB no clasifica por tipo — solo guarda nombre y tamaño. **La clasificación es responsabilidad de U-03** vía LLM:

| `tipo_documento_clasificado` | Variables que aporta |
|---|---|
| `COTIZACION_FORMAL` | Vigencia, garantías, formas de pago, exclusiones, condiciones, penalidades, `precio_total_cotizacion` (cruce) |
| `FICHA_TECNICA` | Marca, modelo, especificaciones técnicas, nivel_servicio |
| `CERTIFICADO` | Sin extracción (solo metadata) |
| `CARTA_PRESENTACION` | Sin extracción (solo metadata) |
| `ANEXO_GENERICO` | Intenta catálogo completo con confianza penalizada (−0.1) |
| `NO_RECONOCIDO` | Sin extracción, `extraccion_omitida = TRUE` |

### 4.3 Catálogo dinámico de variables (BD nuestra)

Tabla `CatalogoVariable` editable por Admin (US-22 extendida). Define qué variables se extraen del PDF a nivel Cotización. Solo aplica a variables **cualitativas/contractuales** (las cuantitativas vienen de SAB).

| Categoría | Variables ejemplo |
|---|---|
| Vigencia | `vigencia_oferta`, `fecha_entrega` |
| Garantías | `tipo_garantia`, `monto_garantia`, `vigencia_garantia`, `cobertura_garantia`, `garantia_postventa` |
| Penalidades | `penalidad_retraso_porcentaje`, `monto_maximo_penalidad` |
| Pago | `forma_pago`, `plazo_pago_dias` |
| Cláusulas | `clausula_confidencialidad`, `condiciones_especiales`, `exclusiones_principales` |
| Soporte | `responsable_tecnico`, `tiempo_respuesta_soporte`, `nivel_servicio` |
| Específicas de Bienes (MVP) | `marca`, `modelo`, `especificaciones_tecnicas`, `lugar_entrega` |
| Cruce anti-fraude | `precio_total_cotizacion` (cruza vs `Σ(cotizacion_item.valor_total_item_con_iva)`) |

---

## 5. Modelo de Datos (Nuestro Lado)

```
Portafolio                                              ◄── 1×solicitud_bienes
  ├─ solicitud_bienes_version (INTEGER)
  ├─ solicitud_bienes_fecha_version (TIMESTAMP)
  ├─ solicitud_bienes_id (BIGINT)
  │   ─ UNIQUE (version, fecha_version, id)
  ├─ radicado_referencia (NUMERIC) — para display
  ├─ analista_id, fecha_creacion, estado
  │
  ├─ ItemSolicitud × N                                  ◄── 1×Portafolio (todos)
  │   ├─ solicitud_item_id_sab (BIGINT)
  │   ├─ numero_item (= consecutivo_solicitud_bienes)
  │   ├─ categoria_id_sab (INTEGER, sin entidad propia)
  │   ├─ cantidad_solicitada, unidad_medida, descripcion
  │   ├─ codigo_srf
  │   └─ FK → SolicitudCotizacion que la cubre
  │
  └─ SolicitudCotizacion × M                            ◄── NUEVA entidad
        ├─ solicitud_cotizacion_id_sab (BIGINT)
        ├─ codigo (VARCHAR — el código SAB)
        ├─ estado_sab (ENUM espejo)
        ├─ fecha_apertura, fecha_cierre
        │
        └─ Cotizacion × K                               ◄── Por proveedor
              ├─ cotizacion_id_sab (BIGINT)
              ├─ proveedor_nit, proveedor_razon_social  ◄── snapshot
              ├─ estado_sab (ENUM espejo)
              ├─ estado_analisis (PENDIENTE | EN_PROCESO | COMPLETADO | ERROR | OMITIDA)
              ├─ requiere_revision_hitl (BOOL)
              ├─ puntaje_global (DECIMAL)
              │
              ├─ ItemCotizado × J                       ◄── Una por ItemSolicitud cotizado
              │   ├─ cotizacion_item_id_sab (BIGINT)
              │   ├─ FK → ItemSolicitud
              │   ├─ cotizado (BOOL) — derivado: existe row en BD SAB
              │   ├─ valor_unidad_sin_iva, valor_unidad_con_iva
              │   ├─ valor_total_item_sin_iva, valor_total_item_con_iva
              │   ├─ porcentaje_iva (snapshot de porcentaje_iva.porcentaje)
              │   ├─ cantidad_cotizada, unidad
              │   ├─ tiempo_entrega_dias, observacion
              │   ├─ descripcion_cotizada
              │   └─ item_solicitado_igual_cotizado (BOOL, espejo SAB)
              │
              ├─ DocumentoPDF × L
              │   ├─ documento_id_sab (BIGINT)
              │   ├─ nombre_archivo_original (TEXT — no estandarizado)
              │   ├─ s3_bucket_origen_sab, s3_key_origen_sab  ◄── desde cotizacion_documento
              │   ├─ s3_bucket_nuestro, s3_key_nuestro        ◄── copia local sanitizada
              │   ├─ tipo_documento_clasificado (ENUM, U-03)
              │   ├─ confianza_clasificacion (DECIMAL)
              │   └─ extraccion_omitida (BOOL)
              │
              ├─ VariableExtraida × P (LLM, nivel Cotización)
              │   ├─ FK documento_pdf_id, FK Cotizacion
              │   ├─ nombre (del CatalogoVariable)
              │   ├─ valor / confianza
              │   └─ referencia_fuente JSON
              │
              ├─ Discrepancia × Q (MotorCruce)
              │   └─ tipo (incl. INCONSISTENCIA_ARITMETICA), severidad
              │
              └─ LlamadaLLM × R (audit CISO, S3)

CatalogoVariable (global, editable Admin) ──────── consumido por C-11 con cache TTL 300s
```

**No se crean entidades**: `Categoria`, `RadicadoSAB`, `ProveedorMaestro` — la información se snapshotea en `Cotizacion`/`ItemSolicitud` al ingestar.

---

## 6. Flujo End-to-End

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Inicio de análisis                                                   │
│    SAB UI presenta botón "Analizar con AssistentBuy"                    │
│    SAB invoca: POST /portafolios/from-sab                               │
│       body: {version, fecha_version, id} de solicitud_bienes            │
│    Nuestro sistema:                                                     │
│      ├─ Valida composite key contra BD SAB                              │
│      ├─ Crea Portafolio (1:1 con solicitud_bienes)                      │
│      └─ Encola pipeline U-02                                            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. U-02 Ingesta                                                         │
│                                                                         │
│ 2a. RepositorioSAB queries (todas en BD SAB):                           │
│     ├─ JOIN solicitud_bienes ← composite key                            │
│     ├─ JOIN solicitud_item por id_solicitud (todos los ítems del PAA)   │
│     ├─ JOIN solicitud_cotizacion WHERE id_solicitud =                   │
│     │   y estado = 'CERRADA'                                            │
│     ├─ JOIN categoria_solicitud_cotizacion (para mapear item→RFQ)       │
│     ├─ JOIN cotizacion WHERE id_solicitud_cotizacion IN (...)           │
│     │   AND estado = 'ACEPTADA_PARA_ESTUDIO'                            │
│     ├─ JOIN cotizacion_item (filtro por año para partition pruning)     │
│     ├─ JOIN porcentaje_iva para resolver tarifas                        │
│     ├─ JOIN proveedor para snapshot de NIT/razón social                 │
│     └─ JOIN cotizacion_documento para lista de docs (no descarga aún)   │
│                                                                         │
│ 2b. Persistencia (BD nuestra):                                          │
│     ├─ Portafolio (composite key + radicado)                            │
│     ├─ ItemSolicitud × N (con FK a SolicitudCotizacion)                 │
│     ├─ SolicitudCotizacion × M                                          │
│     ├─ Cotizacion × K (estado_analisis=PENDIENTE)                       │
│     ├─ ItemCotizado × J (datos del SAB ya estructurados — sin LLM)      │
│     │     - Si solicitud_item NO tiene cotizacion_item en SAB:          │
│     │       crear ItemCotizado con cotizado=FALSE y valores NULL        │
│     └─ DocumentoPDF × L (metadata, S3 origen)                           │
│                                                                         │
│ 2c. GatewaySAB descarga binarios (REST):                                │
│     └─ Por cada DocumentoPDF: GET .../documentos/descargar/{doc_id}     │
│         → SanitizadorZeroTrust → OCR si escaneado → copiar a S3 nuestro │
│                                                                         │
│ 2d. Si cotización tiene 0 PDFs LIMPIO:                                  │
│       estado_analisis = OMITIDA                                         │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼  (INGESTA_COMPLETADA)
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. U-03 Extracción + Cruce                                              │
│                                                                         │
│ 3a. MotorExtraccion (LLM Sonnet 4.6 + Opus 4.7 fallback):               │
│     ├─ Lee CatalogoVariable de BD (cache TTL 300s)                      │
│     ├─ Por cada DocumentoPDF (procesados en paralelo, semáforo 5):      │
│     │   1. LLM clasifica tipo_documento + extrae variables relevantes   │
│     │      en UNA SOLA llamada (tool use con schema dinámico)           │
│     │   2. Anti-EchoLeak: una sesión LLM por documento                  │
│     │   3. Fuzzy validation post-LLM (ratio ≥ 0.85)                     │
│     │   4. Persiste VariableExtraida + LlamadaLLM (S3 audit CISO)       │
│     │                                                                   │
│     └─ Consolidación multi-documento por variable:                      │
│        ganador = max(prioridad_por_tipo × confianza)                    │
│        prioridades: COTIZACION_FORMAL=1.0, FICHA_TECNICA=0.9,           │
│        ANEXO_GENERICO=0.7                                               │
│                                                                         │
│ 3b. MotorCruce (sin LLM — reglas deterministas):                        │
│     • Cruce: precio_total_cotizacion (LLM, COTIZACION_FORMAL)           │
│       vs Σ(ItemCotizado.valor_total_item_con_iva) ⚠️ anti-fraude        │
│     • Validaciones aritméticas por ítem (sobre datos SAB):              │
│       - IVA: valor_iva ≈ precio_sin_iva × porcentaje / 100              │
│       - Total unitario: precio_con_iva ≈ precio_sin_iva + IVA           │
│       - Total ítem: valor_total ≈ precio_con_iva × cantidad             │
│       - % IVA ∈ (SELECT porcentaje FROM porcentaje_iva WHERE activo)    │
│       - cantidad_cotizada == cantidad_solicitada (vs ItemSolicitud)     │
│     • Inconsistencias internas (vigencia_oferta < fecha_entrega...)     │
│     • Persiste Discrepancia con severidad BAJA/MEDIA/ALTA/CRITICA       │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼  (ANALISIS_COMPLETADO)
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. U-04 Revisión HITL (Analista)                                        │
│    • Variables con requiere_revision = TRUE                             │
│    • Discrepancias ALTA/CRITICA                                         │
│    • Cotizaciones OMITIDA / sin items → solicitar aclaración            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. U-05 Comparación + Reporte Comité                                    │
│    Por cada ItemSolicitud en el Portafolio:                             │
│      ├─ Encontrar todas las Cotizaciones (de cualquier                  │
│      │   SolicitudCotizacion del mismo Portafolio) que                  │
│      │   tienen ItemCotizado para ese ítem                              │
│      ├─ Ranquear por valor_total_item_con_iva                           │
│      └─ Resaltar mejor opción (con consideración de tiempo_entrega)     │
│    + Comparación lado a lado de términos contractuales                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Comparaciones que Ejecuta el Sistema

### 7.1 Cruce LLM (PDF) ↔ SAB a nivel Cotización (en U-03)

| Variable | Fuente PDF | Fuente SAB | Severidad si discrepa |
|---|---|---|---|
| `precio_total_cotizacion` | LLM (de `COTIZACION_FORMAL`) | `Σ(cotizacion_item.valor_total_item_con_iva)` | **ALTA** si \|Δ\| > 1.00 COP — anti-fraude |
| `forma_pago` | LLM | (no existe directo en SAB) | Solo extracción, sin cruce |
| `plazo_pago_dias` | LLM | (no existe directo en SAB) | Solo extracción, sin cruce |

> A diferencia de v2, **el SAB no tiene `forma_pago` ni `plazo_pago` a nivel cotización** — solo los financieros agregados de items. Estas variables salen 100% del PDF.

### 7.2 Validaciones Aritméticas por Ítem (sin LLM, sobre BD SAB)

| Check | Fórmula | Tolerancia | Severidad |
|---|---|---|---|
| IVA unitario | `valor_iva ≈ precio_sin_iva × porcentaje / 100` | 0.01 COP | ALTA |
| Total unitario | `precio_con_iva ≈ precio_sin_iva × (1 + porcentaje/100)` | 0.01 COP | ALTA |
| Total ítem | `valor_total ≈ precio_con_iva × cantidad` | 0.01 COP | ALTA |
| % IVA válido | `porcentaje ∈ (SELECT porcentaje FROM porcentaje_iva WHERE activo)` | exacto | MEDIA |
| Cantidad coincide RFQ | `cotizacion_item.cantidad == solicitud_item.cantidad` | exacto | ALTA |

### 7.3 Inconsistencias Internas (variables del PDF, en U-03)

| Check | Severidad |
|---|---|
| `vigencia_oferta < fecha_entrega` | CRITICA |
| `vigencia_garantia < fecha_entrega` | CRITICA |
| `monto_maximo_penalidad > precio_total_cotizacion` | ALTA |

### 7.4 ❌ Validación cruzada del bloque RFQ entre Excels — ELIMINADA

Razón: cada proveedor solo recibe Excel con los ítems de sus categorías. Distintos proveedores pueden cubrir distintas RFQs del mismo `solicitud_bienes`. La integridad la garantiza SAB en su BD.

### 7.5 Comparación entre Proveedores (en U-05, fuera del alcance de U-03)

- Por cada `ItemSolicitud` del Portafolio:
  - Recoger todos los `ItemCotizado` que apuntan a él (de cualquier `Cotizacion` del Portafolio)
  - Ranquear por `valor_total_item_con_iva` ascendente
  - Resaltar mejor opción
- Side-by-side de variables nivel Cotización (LLM-extraídas) entre proveedores

> Opcional: usar `vista_cuadro_comparativo` de SAB como cross-check o como fuente complementaria.

---

## 8. Decisiones de Diseño Confirmadas

| # | Decisión | Estado |
|---|---|---|
| 1 | Catálogo de variables dinámico (`CatalogoVariable` BD + Admin CRUD) | ✅ |
| 2 | Modelo de 2 niveles de variables (Cotización LLM + Ítem desde SAB) | ✅ |
| 3 | Items vienen 100% de BD SAB (`cotizacion_item`) — sin parser Excel | ✅ |
| 4 | **MVP: solo BIENES**. Catálogo dinámico permite agregar SERVICIOS y SOFTWARE en futuras iteraciones sin cambios arquitectónicos | ✅ |
| 5 | IVA dinámico desde tabla `porcentaje_iva` (no hardcoded {0,5,19}) | ✅ |
| 6 | Cruce LLM↔SAB de `precio_total_cotizacion` vs Σ items (anti-fraude) | ✅ |
| 7 | Validaciones aritméticas sin LLM sobre datos SAB | ✅ |
| 8 | Excel obligatorio en SAB (validación: ≥1 row en `cotizacion_item`) | ✅ |
| 9 | `ItemSolicitud` 1×Portafolio (normalizado) | ✅ |
| 10 | Ausencia de `cotizacion_item` row = `cotizado=FALSE` (sin flag explícito) | ✅ |
| 11 | ❌ Obsoleto (no hay parser) — reemplazado por validación a nivel BD | ✅ |
| 12 | Documentos: N llamadas REST por `doc_id` (no S3 directo en MVP) | ✅ |
| 13 | Sin metadata SAB de tipo documento — clasificación por LLM | ✅ |
| 14 | Listado de documentos vía BD SAB (`cotizacion_documento`) | ✅ |
| 15 | Clasificación + extracción en la **misma** llamada LLM (tool use) | ✅ |
| 16 | Modelo híbrido REST (binarios) + BD SAB (estructurado) | ✅ |
| 17 | Prioridad por `tipo_documento` en consolidación multi-PDF | ✅ |
| 18 | ❌ Obsoleto — validación cruzada entre Excels ya no aplica | ✅ |
| 19 | Cache TTL 300s para `CatalogoVariable` en C-11 | ✅ |
| 20 | Tipo `INCONSISTENCIA_ARITMETICA` separado de `INCONSISTENCIA_INTERNA` | ✅ |
| 21 | **Portafolio = `solicitud_bienes` completa (1:1)** | ✅ NUEVO |
| 22 | **Identificación por tripleta (version, fecha_version, id) pasada desde SAB** | ✅ NUEVO |
| 23 | **Nueva entidad `SolicitudCotizacion`** (espejo SAB) | ✅ NUEVO |
| 24 | **Sin entidad `Categoria`** — solo `categoria_id_sab` como FK suelta | ✅ NUEVO |
| 25 | **Sin entidad `Proveedor` maestra** — snapshot en `Cotizacion` al ingestar | ✅ NUEVO |
| 26 | **Trigger SAB**: `solicitud_cotizacion.estado=CERRADA + cotizacion.estado=ACEPTADA_PARA_ESTUDIO` | ✅ NUEVO |
| 27 | **Endpoint nuevo**: `POST /portafolios/from-sab` invocado por SAB UI | ✅ NUEVO |
| 28 | Comparación U-05 por ítem cruzando todas las SolicitudCotizacion del Portafolio | ✅ NUEVO |
| 29 | Queries con partition pruning (`fecha_cotizacion` y `fecha_version` en WHERE) | ✅ NUEVO |
| 30 | Opcional: `vista_cuadro_comparativo` SAB como referencia auxiliar U-05 | ✅ NUEVO |

---

## 9. Deltas Requeridos en Otras Unidades

### Delta en U-01 (Fundación)

**Nuevas entidades**:
- ➕ `CatalogoVariable` (Admin-editable)
- ➕ `SolicitudCotizacion` (espejo SAB)
- ➕ `ItemSolicitud` (FK a Portafolio y a SolicitudCotizacion)
- ➕ `ItemCotizado` (FK a Cotizacion, FK a ItemSolicitud)

**Cambios en `Portafolio`**:
- 🔧 Reemplazar `numero_proceso` por tripleta `(solicitud_bienes_version, solicitud_bienes_fecha_version, solicitud_bienes_id)`
- 🔧 Añadir `radicado_referencia` (NUMERIC) para display

**Cambios en `Cotizacion`**:
- 🔧 `cotizacion_id_sab` (BIGINT), `solicitud_cotizacion_id` (FK)
- 🔧 `estado_sab` (ENUM espejo)
- 🔧 `proveedor_nit`, `proveedor_razon_social` (snapshot)
- 🔧 `estado_analisis` ENUM extendido con `OMITIDA`

**Cambios en `DocumentoPDF`**:
- 🔧 `documento_id_sab` (BIGINT), `nombre_archivo_original` (TEXT)
- 🔧 `s3_bucket_origen_sab`, `s3_key_origen_sab` (referencia)
- 🔧 `tipo_documento_clasificado` (ENUM, llenado por U-03)
- 🔧 `confianza_clasificacion` (DECIMAL), `extraccion_omitida` (BOOL)

**Cambios en `Discrepancia`**:
- 🔧 `tipo` extendido con `INCONSISTENCIA_ARITMETICA`

### Delta en U-02 (Ingesta)

**Nuevos componentes/responsabilidades**:
- ➕ `RepositorioSAB`: queries directas a 10 tablas SAB (ver §3.1)
- ❌ `ParserExcelSAB`: ELIMINADO (decisión v3)
- ➕ Endpoint `POST /portafolios/from-sab` para inicio invocado desde SAB UI

**Cambios en componentes existentes**:
- 🔧 `GatewaySAB`: solo descarga binarios de documentos por `doc_id` (REST)
- 🔧 Persistencia jerárquica: Portafolio → SolicitudCotizacion → Cotizacion → ItemCotizado en una transacción
- 🔧 Validación: cotizaciones procesadas solo si estado SAB cumple §6 paso 2a
- 🔧 Si `cotizacion_item` row falta para un ítem → crear `ItemCotizado` con `cotizado=FALSE`

### Delta en U-06 (Administración) o extensión de US-22

- ➕ CRUD de `CatalogoVariable` desde Admin UI
- 🔧 Evento auditoría `CATALOGO_VARIABLE_MODIFICADO` en `AuditTrail`

---

## 10. Pendientes para Resolver Después (TBD)

Quedarán documentados en `cross-unit-deltas.md`:

- **Conexión BD SAB**: read replica vs primaria, VPN vs VPC peering, usuario `assistentbuy_reader`
- **Mecanismo exacto SAB UI → AssistentBuy**: ¿botón con POST directo o deep link con token de autenticación?
- **Sincronización de estado**: ¿cómo notifica SAB cuando una cotización pasa a `ACEPTADA_PARA_ESTUDIO` después de creado el Portafolio? (webhook desde SAB, polling, o reanudación manual)
- **`vista_cuadro_comparativo` de SAB**: explorar si su lógica nos sirve como base para U-05 o solo como cross-check
- **Rotación de credenciales SAB Keycloak**: política y herramienta (AWS Secrets Manager rotation)

---

## 11. Siguiente Paso

Si apruebas esta **v3**, procedo en este orden:

1. **`domain-entities.md`** — nuevas entidades (`CatalogoVariable`, `SolicitudCotizacion`, `ItemSolicitud`, `ItemCotizado`), Portafolio con composite key, eliminación de catálogo estático, campos nuevos en `DocumentoPDF` y `Cotizacion`
2. **`business-rules.md`** — BRs aritméticas (con IVA dinámico), BR cruce total LLM↔SAB, BR clasificación documento, BR consolidación multi-PDF por prioridad, BR trigger SAB, eliminación de BR obsoletas (parser Excel, validación cruzada Excels)
3. **`business-logic-model.md`** — `RepositorioSAB` con queries detalladas, `GatewaySAB` solo binarios, `GatewayLLM` con schema dinámico desde cache, consolidación multi-PDF, validaciones MotorCruce
4. **`cross-unit-deltas.md`** — TBDs operacionales + deltas U-01, U-02, U-06
5. **`plan/u03-extraccion-functional-design-plan.md`** — sección "Revisión 2026-05-28" con las 30 decisiones
6. **`aidlc-state.md`** + **`audit.md`** — registro de la revisión

> **GATE**: Aprueba (✅) o pide ajustes (🔧). No tocaré ningún otro archivo hasta tu confirmación.
