# Cross-Unit Deltas — U-03 Functional Design

> **Origen**: Revisión del Functional Design U-03 del 2026-05-28 tras integración del schema SAB real.
> **Propósito**: Listar los cambios requeridos en otras unidades (U-01, U-02, U-06) y los TBDs operacionales pendientes.
> **Estado**: Estos deltas deben aplicarse al re-abrir las unidades respectivas para iteración posterior.

---

## 1. Delta en U-01 (Fundación) — Entidades y Estados

### 1.1 Nuevas Entidades Persistentes

| Entidad | Propósito | Definición |
|---|---|---|
| `CatalogoVariable` | Catálogo dinámico de variables, editable Admin | Ver `domain-entities.md` §1.1 |
| `LlamadaLLM` | Auditoría CISO de cada llamada al LLM | Ver `domain-entities.md` §1.2 |
| `SolicitudCotizacion` | Espejo de `solicitud_cotizacion` SAB | Ver `domain-entities.md` §3.1 |
| `ItemSolicitud` | Espejo de `solicitud_item` SAB | Ver `domain-entities.md` §3.2 |
| `ItemCotizado` | Espejo de `cotizacion_item` SAB | Ver `domain-entities.md` §3.3 |

### 1.2 Cambios en `Portafolio` (nuevos campos)

```sql
ALTER TABLE portafolio ADD COLUMN solicitud_bienes_version INTEGER NOT NULL;
ALTER TABLE portafolio ADD COLUMN solicitud_bienes_fecha_version TIMESTAMP NOT NULL;
ALTER TABLE portafolio ADD COLUMN solicitud_bienes_id BIGINT NOT NULL;
ALTER TABLE portafolio ADD COLUMN radicado_referencia NUMERIC(10,2) NULL;
ALTER TABLE portafolio ADD CONSTRAINT portafolio_solicitud_bienes_uk
    UNIQUE (solicitud_bienes_version, solicitud_bienes_fecha_version, solicitud_bienes_id);
ALTER TABLE portafolio DROP COLUMN numero_proceso;  -- si existía
```

### 1.3 Cambios en `Cotizacion`

```sql
ALTER TABLE cotizacion ADD COLUMN cotizacion_id_sab BIGINT NOT NULL;
ALTER TABLE cotizacion ADD COLUMN solicitud_cotizacion_id UUID NOT NULL
    REFERENCES solicitud_cotizacion(id);
ALTER TABLE cotizacion ADD COLUMN proveedor_nit VARCHAR(30) NOT NULL;
ALTER TABLE cotizacion ADD COLUMN proveedor_razon_social TEXT NOT NULL;
ALTER TABLE cotizacion ADD COLUMN estado_sab VARCHAR(50) NOT NULL;
ALTER TABLE cotizacion ADD CONSTRAINT cotizacion_estado_sab_check
    CHECK (estado_sab IN ('EN_PROCESO','EN_REVISION','EN_CORRECCION',
                          'ACEPTADA_PARA_ESTUDIO','SELECCIONADA','NO_SELECCIONADA'));
-- Extender estado_analisis
ALTER TABLE cotizacion DROP CONSTRAINT cotizacion_estado_analisis_check;
ALTER TABLE cotizacion ADD CONSTRAINT cotizacion_estado_analisis_check
    CHECK (estado_analisis IN ('PENDIENTE','EN_PROCESO','COMPLETADO','ERROR','OMITIDA'));
```

### 1.4 Cambios en `DocumentoPDF`

```sql
ALTER TABLE documento_pdf ADD COLUMN documento_id_sab BIGINT NOT NULL;
ALTER TABLE documento_pdf ADD COLUMN nombre_archivo_original TEXT NOT NULL;
ALTER TABLE documento_pdf ADD COLUMN s3_bucket_origen_sab VARCHAR(255) NULL;
ALTER TABLE documento_pdf ADD COLUMN s3_key_origen_sab VARCHAR(255) NULL;
ALTER TABLE documento_pdf ADD COLUMN motivo_error TEXT NULL;
ALTER TABLE documento_pdf ADD COLUMN tipo_documento_clasificado VARCHAR(30) NULL;
ALTER TABLE documento_pdf ADD CONSTRAINT documento_tipo_clasificado_check
    CHECK (tipo_documento_clasificado IS NULL OR tipo_documento_clasificado IN
           ('COTIZACION_FORMAL','FICHA_TECNICA','CERTIFICADO',
            'CARTA_PRESENTACION','ANEXO_GENERICO','NO_RECONOCIDO'));
ALTER TABLE documento_pdf ADD COLUMN confianza_clasificacion DECIMAL(5,4) NULL;
ALTER TABLE documento_pdf ADD COLUMN extraccion_omitida BOOLEAN NOT NULL DEFAULT FALSE;
```

### 1.5 Cambios en `VariableExtraida`

```sql
ALTER TABLE variable_extraida ADD COLUMN fuente_consolidacion JSONB NULL;
-- Estructura: {tipo_documento_ganador, score_prioridad, candidatos_alternativos:[...]}
```

### 1.6 Cambios en `Discrepancia`

```sql
ALTER TABLE discrepancia DROP CONSTRAINT discrepancia_tipo_check;
ALTER TABLE discrepancia ADD CONSTRAINT discrepancia_tipo_check
    CHECK (tipo_discrepancia IN
           ('VALOR_DIFERENTE','AUSENTE_EN_PDF','SOLO_EN_PDF',
            'INCONSISTENCIA_INTERNA','INCONSISTENCIA_ARITMETICA','MONEDA_DIFERENTE'));
ALTER TABLE discrepancia ADD COLUMN item_cotizado_id UUID NULL
    REFERENCES item_cotizado(id);
```

### 1.7 Eventos AuditTrail (catálogo extendido)

Agregar los siguientes valores al catálogo `evento_tipo`:

- `ANALISIS_INICIADO_U03`
- `DOCUMENTO_CLASIFICADO`
- `EXTRACCION_COMPLETADA`
- `CONSOLIDACION_VARIABLE`
- `VARIABLE_EXTRAIDA_GUARDADA`
- `CRUCE_ARITMETICO_FALLIDO`
- `CRUCE_COMPLETADO`
- `ANALISIS_COMPLETADO`
- `COTIZACION_OMITIDA`
- `LLM_INVOCADO`
- `CATALOGO_VARIABLE_MODIFICADO`

---

## 2. Delta en U-02 (Ingesta) — Componentes y Responsabilidades

### 2.1 Componentes Eliminados

- ❌ **`ParserExcelSAB`**: Eliminado completamente. Los datos estructurados ya están en BD SAB en `cotizacion_item`.

### 2.2 Componentes Modificados

#### `RepositorioSAB` — Queries expandidas

Acceso de solo lectura a las siguientes tablas (con partition pruning donde aplica):

```sql
-- 1. Solicitud principal y radicado
SELECT * FROM solicitud_bienes
WHERE version = ? AND fecha_version = ? AND id = ?;

SELECT * FROM radicado WHERE id_solicitud = ?;

-- 2. RFQs hijas
SELECT sc.* FROM solicitud_cotizacion sc
WHERE sc.id_solicitud = ? AND sc.estado = 'CERRADA';

-- 3. Categorías que cubre cada RFQ
SELECT * FROM categoria_solicitud_cotizacion
WHERE id_solicitud_cotizacion IN (...);

-- 4. Items de la RFQ (particionado por fecha_version)
SELECT * FROM solicitud_item
WHERE id_solicitud = ?
  AND fecha_version BETWEEN ? AND ?;  -- partition pruning

-- 5. Cotizaciones aceptadas
SELECT c.*, p.nombre_razon_social, p.numero_documento_identificacion
FROM cotizacion c
JOIN proveedor p ON c.id_proveedor = p.id
WHERE c.id_solicitud_cotizacion IN (...)
  AND c.estado = 'ACEPTADA_PARA_ESTUDIO';

-- 6. Items cotizados (particionado por fecha_cotizacion)
SELECT ci.*, pi.porcentaje
FROM cotizacion_item ci
JOIN porcentaje_iva pi ON ci.id_porcentaje_iva = pi.id
WHERE ci.id_cotizacion IN (...)
  AND ci.fecha_cotizacion BETWEEN ? AND ?;  -- partition pruning

-- 7. Metadatos de documentos
SELECT * FROM cotizacion_documento WHERE id_cotizacion IN (...);

-- 8. Tarifas IVA activas (cacheable, cambia raramente)
SELECT * FROM porcentaje_iva WHERE activo = TRUE;
```

#### `GatewaySAB` — Reducido a descarga de binarios

Solo se usa para:
```
GET /sab-bienes/api/v1/cotizacion/documentos/descargar/{doc_id}
→ binario (PDF/Excel/otros)
```

### 2.3 Componente Nuevo: Endpoint `POST /portafolios/from-sab`

```python
POST /portafolios/from-sab
Body:
{
    "solicitud_bienes_version": 1,
    "solicitud_bienes_fecha_version": "2026-05-20T10:00:00Z",
    "solicitud_bienes_id": 42
}

Response 202 Accepted:
{
    "portafolio_id": "uuid",
    "estado": "INGESTA",
    "websocket_subscribe": "/ws/portafolios/{id}"
}
```

Flujo del endpoint:
1. Valida composite key contra BD SAB (`solicitud_bienes` debe existir)
2. Crea `Portafolio` con la tripleta + `radicado_referencia`
3. Encola ingesta en BackgroundTask
4. Retorna 202 inmediatamente para WebSocket subscribe

### 2.4 Persistencia Jerárquica (transacción única por portafolio)

Al ingerir, U-02 persiste en este orden dentro de una sola transacción:

```
1. Portafolio (composite key + radicado)
2. SolicitudCotizacion × M (todas las RFQs hijas con estado CERRADA)
3. ItemSolicitud × N (1×Portafolio, normalizado, con FK a SolicitudCotizacion)
4. Cotizacion × K (con snapshot de NIT/razón social)
5. ItemCotizado × J (uno por ItemSolicitud × cotización; con cotizado=FALSE si no hay row en SAB)
6. DocumentoPDF × L (metadata; binarios se descargan en paso siguiente)
```

Luego, fuera de la transacción:
- Descarga de binarios PDF vía GatewaySAB
- SanitizadorZeroTrust por cada PDF
- OCR Textract si aplica

### 2.5 Validaciones de U-02

- Cada `Cotizacion` debe tener al menos 1 row en `cotizacion_item` SAB. Si no → `Cotizacion.estado_ingesta = SIN_ITEMS` (nuevo valor ENUM en `estado_ingesta`).
- Los items con `accion_cotizar = "No"` se manifiestan como ausencia de row en `cotizacion_item` → `ItemCotizado.cotizado = FALSE` con campos numéricos NULL.

---

## 3. Delta en U-06 (Administración) o Extensión de US-22

### 3.1 Nuevo CRUD: CatalogoVariable

UI Admin para gestionar variables del catálogo:

- **CREATE**: agregar variable nueva al catálogo. Campos: nombre, descripcion_llm, tipo_dato, criticidad, tipo_adquisicion, tolerancia_numerica, es_cruce_sab, aplica_a_tipo_documento.
- **UPDATE**: editar descripción LLM, criticidad, tolerancia, aplica_a_tipo_documento. No se permite cambiar `nombre` o `tipo_dato` (rompería compatibilidad de extracciones previas).
- **DEACTIVATE**: marcar `activo = FALSE`. La variable deja de extraerse pero las `VariableExtraida` históricas se preservan.

### 3.2 Auditoría

Cada operación CRUD genera evento `CATALOGO_VARIABLE_MODIFICADO` en `AuditTrail`:

```json
{
  "accion": "CREATE | UPDATE | DEACTIVATE",
  "variable_nombre": "vigencia_garantia",
  "before": { ... } | null,
  "after": { ... } | null,
  "admin_id": "uuid"
}
```

### 3.3 Seed Data MVP

Al desplegar U-06 por primera vez, ejecutar migración seed con las 24 variables iniciales de BIENES (ver `domain-entities.md` §1.1).

---

## 4. TBDs Operacionales — Pendientes de Coordinación

Estos no bloquean la aprobación de U-03 Functional Design, pero deben resolverse antes de Code Generation:

### 4.1 Acceso a BD SAB

| Pendiente | Responsable | Impacto si no se resuelve |
|---|---|---|
| Confirmar disponibilidad de **read replica** vs BD primaria | DBA SAB + DevOps | Performance — risk de carga en SAB primaria |
| **Usuario read-only** `assistentbuy_reader` con GRANT SELECT sobre tablas listadas en §2.2 | DBA SAB | Seguridad |
| **Conexión segura**: VPN site-to-site vs VPC peering vs PrivateLink | DevOps + Networking | Latencia + seguridad |
| **Credenciales** en AWS Secrets Manager con rotación periódica | DevOps | Operaciones |
| **Schema sync**: política para detectar cambios DDL en SAB que rompan nuestras queries | DBA SAB + nosotros | Riesgo continuidad |

### 4.2 Mecanismo SAB UI → AssistentBuy

| Pendiente | Responsable | Impacto |
|---|---|---|
| Definir si SAB invoca `POST /portafolios/from-sab` con token JWT compartido, OAuth2, o IP whitelist | Equipos SAB + AssistentBuy | Seguridad endpoint |
| Definir si la invocación es síncrona (espera respuesta) o async (fire-and-forget con webhook de progreso) | Equipos SAB + AssistentBuy | UX en SAB UI |
| Manejo de errores: ¿qué hace SAB si recibe 4xx/5xx? Retry, mostrar mensaje al usuario? | Equipos SAB | UX en SAB UI |

### 4.3 Sincronización de Estado SAB ↔ Nuestra BD

- ¿Qué pasa si una `cotizacion.estado` cambia en SAB (ej. de `ACEPTADA_PARA_ESTUDIO` a `EN_CORRECCION`) **después** de que iniciamos análisis?
- Opciones:
  - **A) Polling** desde nuestro lado a `cotizacion.estado` cada N minutos (simple, latencia ~N)
  - **B) Webhook** desde SAB cuando cambian estados relevantes (real-time, requiere desarrollo en SAB)
  - **C) Snapshot inmutable**: una vez ingerido, no monitoreamos cambios. El Analista decide si reiniciar.
- **Recomendación tentativa**: **C** para MVP. Re-evaluar si el flujo del negocio lo exige.

### 4.4 Vista `vista_cuadro_comparativo` de SAB

- ¿Su lógica nos sirve como base para U-05 (comparación entre proveedores) o solo como cross-check?
- Acción: leer la definición de la vista (`bd_sab.sql` línea 3191+) y evaluar al diseñar U-05.

### 4.5 Concurrencia Multi-Portafolio en C-11

- Cache de catálogo en C-11 es global (no por portafolio). Con concurrencia alta, evaluar si necesita lock o si el `if-expired-then-refresh` es seguro con asyncio. **Recomendación**: usar `asyncio.Lock` en la verificación de expiración para evitar refreshes simultáneos.

### 4.6 Otros

- **Política de retención** del bucket S3 `llm_calls/` (CISO audit): ¿1 año? ¿5 años? ¿lifecycle a Glacier?
- **Cuota Anthropic API**: capacity planning para peak (varios portafolios concurrentes con N PDFs cada uno).
- **Rotación credenciales SAB Keycloak**: política y herramienta.

---

## 5. Resumen de Impacto

| Unidad | Tipo | Esfuerzo estimado |
|---|---|---|
| U-01 | Schema migrations + nuevas entidades | Alto (5 entidades nuevas + 4 ALTER TABLE) |
| U-02 | Refactor RepositorioSAB + eliminar ParserExcel + nuevo endpoint | Alto (rediseño parcial de ingesta) |
| U-06 | Nuevo CRUD CatalogoVariable + UI Admin + seed data | Medio |
| U-03 | Aplicado en esta iteración del Functional Design | Completado |
| U-04, U-05, U-07 | Sin impacto directo (consumen del modelo de U-03 ya alineado) | Ninguno |

> Estos deltas se reflejarán al re-abrir las unidades respectivas. Mientras tanto, U-03 queda en estado **COMPLETADO (revisado)** asumiendo que los deltas se cumplirán.
