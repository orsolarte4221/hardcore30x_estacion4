# Business Rules — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Functional Design
> **Versión**: v2 (revisión 2026-05-28)
> **Scope MVP**: solo BIENES
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

---

## Convenciones

- **BR-U03-XX**: Identificador único de regla de negocio de U-03
- **Severidad**: `CRITICA` (fallo bloquea el flujo) | `ALTA` (impacta la integridad) | `MEDIA` (afecta UX) | `BAJA` (validación de datos)
- **Refs**: Decisión del plan que originó la regla (las decisiones 21-30 son de la revisión 2026-05-28)

---

## Sección 1 — Catálogo Dinámico de Variables (C-03 + C-11)

### BR-U03-01: Catálogo dinámico desde BD — editable por Admin
- **Severidad**: ALTA
- **Regla**: El catálogo de variables vive en la tabla `CatalogoVariable` (persistente, editable por Admin desde la UI). El sistema NO usa catálogos hardcoded en código. En el MVP solo están activas variables con `tipo_adquisicion = BIENES`.
- **Acción**: C-11 lee el catálogo directamente desde BD en cada extracción (BR-U03-19, sin cache).
- **Refs**: Decisión 21 (Portafolio = `solicitud_bienes`), Decisión 1 (Catálogo dinámico)

### BR-U03-02: Selección de tipo de adquisición — MVP fijo a BIENES
- **Severidad**: ALTA
- **Regla**: El sistema solo soporta `tipo_adquisicion = BIENES` en el MVP. La estructura permite agregar `SERVICIOS` y `SOFTWARE` insertando variables en `CatalogoVariable` con esos tipos en el futuro sin cambios de código.
- **Refs**: Scope MVP 2026-05-28

### BR-U03-03: Variables ausentes en el PDF — convención `NO_ENCONTRADO`
- **Severidad**: MEDIA
- **Regla**: Cuando una variable del catálogo no aparece en el PDF, el LLM devuelve `valor = "NO_ENCONTRADO"` con `confianza = 1.0`. Distingue "no encontrada" (certeza alta) de "encontrada pero ambigua" (confianza baja).
- **Consecuencia**: Variables `NO_ENCONTRADO` con criticidad CRITICA o ALTA marcan la cotización como `requiere_revision_hitl = TRUE`.
- **Refs**: Decisión 1.3-C

### BR-U03-04: Atributo `criticidad` del catálogo no se modifica en runtime
- **Severidad**: ALTA
- **Regla**: La `criticidad` de cada `CatalogoVariable` solo cambia mediante edición Admin (con evento `CATALOGO_VARIABLE_MODIFICADO`). No se ajusta dinámicamente durante el procesamiento.
- **Refs**: Decisión 4.4-A

---

## Sección 2 — Clasificación + Extracción LLM (C-03 + C-11)

### BR-U03-05: Pipeline sin Document AI en U-03
- **Severidad**: ALTA
- **Regla**: U-03 usa únicamente el texto del PDF (nativo o OCR companion de U-02 en S3). No realiza llamadas adicionales a Amazon Textract.
- **Refs**: Decisión 2.1-A

### BR-U03-06: Estrategia mixta Sonnet/Opus — umbral de fallback 0.75
- **Severidad**: ALTA
- **Regla**: Extracción primaria con Claude Sonnet 4.6. Si alguna variable tiene `confianza < UMBRAL_FALLBACK_OPUS` (default 0.75), C-11 hace una segunda llamada con Claude Opus 4.7 para re-extraer **solo** esas variables. El resultado de Opus reemplaza al de Sonnet para esas variables.
- **Persistencia**: hasta 2 `LlamadaLLM` por PDF (PRIMARIA + FALLBACK).
- **Refs**: Decisión 2.2-X

### BR-U03-07: C-11 GatewayLLM — interfaz LLMProvider abstracta
- **Severidad**: ALTA
- **Regla**: C-11 implementa `LLMProvider` que desacopla a C-03/C-04/C-09 del proveedor. MVP implementa solo `AnthropicProvider`. La interfaz permite agregar OpenAI/Gemini sin cambios en componentes de negocio.
- **Refs**: Decisión 2.2-X

### BR-U03-08: Una sola llamada LLM por PDF: clasifica + extrae
- **Severidad**: CRITICA
- **Regla**: El tool use de Claude integra **clasificación de `tipo_documento` + extracción de variables aplicables** en una sola invocación. Se evita la doble llamada (clasificar primero, extraer después).
  - El tool schema dinámico contiene primero `tipo_documento_clasificado` (ENUM con 6 valores), luego `variables` (lista de objetos con `nombre`/`valor`/`confianza`/`referencia_fuente`/`analisis`).
  - El LLM decide qué variables intentar según el tipo que reconoce.
- **Refs**: Decisión 15 (Clasificación + extracción en una llamada), Decisión 2.4-A

### BR-U03-09: Tool schema dinámico desde CatalogoVariable
- **Severidad**: CRITICA
- **Regla**: El `input_schema` del tool use se construye dinámicamente desde el catálogo leído directo de BD en C-11 (BR-U03-19, sin cache). Cambios en CatalogoVariable (via Admin) se reflejan **inmediatamente** en la siguiente extracción, sin redeploy.
- **Refs**: Decisión 1 (catálogo dinámico), BR-U03-19 (lectura directa, revisado por NFR Q5.3)

### BR-U03-10: Prompt con texto entre delimitadores Spotlighting
- **Severidad**: MEDIA
- **Regla**: El user message contiene el texto del PDF entre delimitadores `[PDF_{token}_START]...[PDF_{token}_END]` aplicados por U-02. El system prompt define la tool, las reglas de extracción y los tipos de documento posibles.
- **Refs**: Decisión 2.3-A

### BR-U03-11: Confianza self-reported por el LLM
- **Severidad**: MEDIA
- **Regla**: El score de confianza de cada variable es el reportado por el LLM. Solo se modifica si la validación fuzzy falla (BR-U03-15).
- **Refs**: Decisión 3.1-A

### BR-U03-12: Umbrales de confianza fijos
- **Severidad**: ALTA
- **Regla**: Umbrales hardcoded:
  - `confianza >= 0.85` → 🟢 Alta
  - `0.60 <= confianza < 0.85` → 🟡 Media
  - `confianza < 0.60` → 🔴 Baja → escalamiento HITL
- **Refs**: Decisión 3.2-A

### BR-U03-13: Chunking por páginas si excede contexto
- **Severidad**: ALTA
- **Regla**: Si el texto excede la ventana del modelo, U-03 divide en chunks de `CHUNK_PAGINAS` páginas (default 20) con overlap de 1 página. Cada chunk produce un resultado parcial; la consolidación intra-PDF toma el valor de mayor confianza por variable.
- **Refs**: Decisión 2.5-A

### BR-U03-14: Anti-EchoLeak — una sesión LLM por documento
- **Severidad**: CRITICA
- **Regla**: Cada llamada al LLM es completamente independiente (sin conversation history). El array `messages` contiene exclusivamente system prompt + user message del documento actual entre delimitadores Spotlighting. **Por construcción arquitectónica**, no por instrucción en prompt.
- **Refs**: Decisión 8.4-A

---

## Sección 3 — Validación Anti-Alucinación (Gate G3)

### BR-U03-15: Validación fuzzy post-LLM
- **Severidad**: CRITICA
- **Regla**: Tras recibir la respuesta del LLM, el sistema verifica con `rapidfuzz` que `referencia_fuente.fragmento_texto` exista en la página reportada del PDF. Mínimo ratio 0.85.
- **Si ratio < 0.85**: `VariableExtraida.confianza = min(confianza_llm, 0.3)` + `requiere_revision = TRUE`.
- **Propósito**: Gate G3 — tasa alucinación < 1%.
- **Refs**: Decisión 3.4-B

### BR-U03-16: Estructura de `referencia_fuente`
- **Severidad**: MEDIA
- **Regla**:
  ```json
  {
    "documento_id": "uuid",
    "pagina": 7,
    "seccion": "Cláusula 4.2",
    "fragmento_texto": "El plazo de garantía será de 12 meses..."
  }
  ```
  `seccion` es opcional.
- **Refs**: Decisión 3.3-B

### BR-U03-17: Gate G3 — valor LLM conservado, excluido del reporte
- **Severidad**: CRITICA
- **Regla**: Si `requiere_revision = TRUE`, `valor` mantiene el sugerido por el LLM (no se nullea). UI muestra "Valor sugerido por IA — pendiente verificación". El **reporte para Comité** excluye variables sin `valor_revisado`. Resultado: 0% de invención en reporte final.
- **Refs**: Decisión 6.3-B

---

## Sección 4 — Consolidación Multi-PDF por Cotización

### BR-U03-18: Consolidación de variables entre múltiples PDFs de una cotización
- **Severidad**: ALTA
- **Regla**: Cuando una cotización tiene varios `DocumentoPDF`, una misma variable puede extraerse de varios. Para elegir el valor ganador:
  ```
  score_consolidacion = PRIORIDAD_TIPO_DOCUMENTO × confianza_variable

  PRIORIDAD_TIPO_DOCUMENTO:
    COTIZACION_FORMAL = 1.0
    FICHA_TECNICA     = 0.9
    ANEXO_GENERICO    = 0.7
    (CERTIFICADO, CARTA_PRESENTACION, NO_RECONOCIDO no aportan variables)
  ```
  Gana el candidato con mayor `score_consolidacion`. Los demás se persisten en `fuente_consolidacion.candidatos_alternativos` para auditoría.
- **Caso especial**: la variable `precio_total_cotizacion` solo se acepta de `COTIZACION_FORMAL` (no de anexos).
- **Refs**: Decisión 17 (Prioridad por tipo), Decisión 15

### BR-U03-19: Lectura directa del catálogo en C-11 (sin cache) — revisado por NFR Q5.3
- **Severidad**: MEDIA
- **Regla**: C-11 lee `CatalogoVariable` directamente desde BD en cada extracción de PDF (`WHERE activo = TRUE AND tipo_adquisicion = 'BIENES'`). **No hay cache en memoria.** El catálogo es pequeño (~24 variables), por lo que la carga de BD es despreciable incluso con concurrencia.
- **Implicación**: cambios del Admin se reflejan **inmediatamente** (sin ventana de 300s). No se requiere `asyncio.Lock` ni invalidación push.
- **Refs**: Decisión NFR Q5.3 (2026-05-29) — **anula** la Decisión 19 original (cache TTL 300s)

---

## Sección 5 — Cruce SAB ↔ PDF (C-04 MotorCruce)

### BR-U03-20: Cruce LLM ↔ SAB único — `precio_total_cotizacion`
- **Severidad**: ALTA (anti-fraude)
- **Regla**: La única variable LLM con cruce SAB es `precio_total_cotizacion`. Se compara contra:
  ```sql
  SELECT SUM(valor_total_item_con_iva)
  FROM cotizacion_item
  WHERE id_cotizacion = ? AND fecha_cotizacion BETWEEN ? AND ?
  ```
  Si `|valor_pdf - valor_sab| > 1.00 COP` → Discrepancia `INCONSISTENCIA_ARITMETICA` severidad **ALTA**.
- **Propósito**: detectar fraude tipo "Excel barato en SAB + PDF caro al Comité".
- **Refs**: Decisión 6 (cruce precio_total), Decisión 20 (INCONSISTENCIA_ARITMETICA)

### BR-U03-21: Variables PDF sin cruce SAB
- **Severidad**: MEDIA
- **Regla**: El resto de variables del catálogo (vigencia, garantías, forma_pago, plazo_pago, etc.) se extraen del PDF y se persisten en `VariableExtraida`, **sin cruce SAB** (no existen como campos a nivel cotización en el SAB). Solo se evalúan inconsistencias internas (BR-U03-26).
- **Refs**: Decisión revisada — SAB no tiene `forma_pago` ni `plazo_pago` a nivel cotización

### BR-U03-22: Validaciones aritméticas por ítem (sin LLM, sobre BD SAB)
- **Severidad**: ALTA (detección de errores en datos SAB)
- **Regla**: Por cada `ItemCotizado` con `cotizado = TRUE`, el MotorCruce valida (todos con tolerancia 0.01 COP):
  | Check | Fórmula | Severidad si falla |
  |---|---|---|
  | `iva_unitario` | `valor_iva ≈ precio_sin_iva × porcentaje_iva / 100` | ALTA |
  | `total_unitario` | `precio_con_iva ≈ precio_sin_iva × (1 + porcentaje_iva/100)` | ALTA |
  | `total_item` | `valor_total_item_con_iva ≈ precio_con_iva × cantidad` | ALTA |

  Cada falla crea Discrepancia `INCONSISTENCIA_ARITMETICA` con `item_cotizado_id` poblado.
- **Refs**: Decisión 7 (aritmética sin LLM), schema SAB con `porcentaje_iva` table

### BR-U03-23: Validación de % IVA contra tabla `porcentaje_iva`
- **Severidad**: MEDIA
- **Regla**: Para cada `ItemCotizado`, validar:
  ```sql
  porcentaje_iva ∈ (SELECT porcentaje FROM porcentaje_iva WHERE activo = TRUE)
  ```
  Si el porcentaje no está en la lista activa SAB → Discrepancia `INCONSISTENCIA_ARITMETICA` severidad MEDIA con `check = "iva_valido"`.
- **Refs**: Decisión 5 (IVA dinámico desde tabla)

### BR-U03-24: Validación cantidad cotizada vs solicitada
- **Severidad**: ALTA
- **Regla**: Para cada `ItemCotizado` con `cotizado = TRUE`:
  ```
  if ItemCotizado.cantidad_cotizada != ItemSolicitud.cantidad_solicitada:
      Discrepancia INCONSISTENCIA_ARITMETICA, severidad ALTA, check = "cantidad_rfq"
  ```
- **Refs**: Decisión 7 (validaciones aritméticas)

### BR-U03-25: Ítems no cotizados
- **Severidad**: BAJA (informativo)
- **Regla**: Si para una `Cotizacion` existe un `ItemSolicitud` sin `cotizacion_item` row en BD SAB, U-02 ya creó `ItemCotizado` con `cotizado = FALSE`. U-03 no genera discrepancias para estos — son legítimos (proveedor decide no cotizar ítems de categorías no relevantes).
- **Refs**: Decisión 10 (ausencia de row = no cotizó)

### BR-U03-26: Inconsistencias internas entre variables LLM
- **Severidad**: CRITICA
- **Regla**: El MotorCruce verifica inconsistencias lógicas entre variables LLM-extraídas del mismo PDF/cotización:
  - `vigencia_oferta < fecha_entrega` → CRITICA (oferta vence antes de entrega)
  - `vigencia_garantia < fecha_entrega` → CRITICA (garantía expira antes de entrega)
  - `monto_maximo_penalidad > precio_total_cotizacion` → ALTA (penalidad excede el contrato)
  - Tipo de discrepancia: `INCONSISTENCIA_INTERNA`.
- **Refs**: Decisión 4.3-A

### BR-U03-27: Normalización antes de comparar
- **Severidad**: ALTA
- **Regla**: Antes de comparar valores:
  - **MONETARIO**: convertir a `Decimal`, normalizar separadores. Si las monedas difieren (COP vs USD) → Discrepancia `MONEDA_DIFERENTE` severidad ALTA, sin conversión automática.
  - **FECHA**: parsear a ISO 8601, comparar exacto.
  - **NUMERICO**: convertir a `Decimal`, tolerancia configurable (default 0).
  - **TEXTO**: lowercase + strip, fuzzy ratio > 0.90.
- **Refs**: Decisión 4.2-A

---

## Sección 6 — Orquestación del Pipeline (C-09 ServicioPipeline)

### BR-U03-28: Trigger por evento Pub/Sub interno
- **Severidad**: ALTA
- **Regla**: C-09 arranca al recibir `INGESTA_COMPLETADA` del bus interno (asyncio Queue). U-02 publica este evento al terminar la ingesta. No hay invocación directa U-02 → U-03.
- **Refs**: Decisión 5.1-B

### BR-U03-29: Filtro por estado SAB al cargar cotizaciones
- **Severidad**: ALTA
- **Regla**: El pipeline procesa solo cotizaciones con:
  - `SolicitudCotizacion.estado_sab = 'CERRADA'` (proceso cerrado en SAB)
  - `Cotizacion.estado_sab = 'ACEPTADA_PARA_ESTUDIO'` (cotización aprobada para análisis)
- **Cotizaciones que no cumplan**: se ignoran (no se marca OMITIDA, simplemente no entran al pipeline).
- **Refs**: Decisión 26 (trigger estado SAB)

### BR-U03-30: Concurrencia por documento con semáforo
- **Severidad**: MEDIA
- **Regla**: Unidad de concurrencia = `DocumentoPDF` individual. Semáforo `MAX_PDFS_CONCURRENTES` (default 5). Al completar todos los PDFs de una cotización, MotorCruce arranca para esa cotización.
- **Refs**: Decisión 5.2-B

### BR-U03-31: Retry con backoff exponencial + registros de error
- **Severidad**: ALTA
- **Regla**: Si la llamada LLM falla (timeout, rate limit, 5xx):
  - Reintento 1: espera 2s
  - Reintento 2: 4s
  - Reintento 3: 8s
  - Si todos fallan: crea `VariableExtraida` con `confianza=0.0`, `valor="ERROR_EXTRACCION"`, `analisis="Error en llamada al LLM"`, `requiere_revision=TRUE` por cada variable del catálogo.
- **Refs**: Decisión 5.3-A

### BR-U03-32: Cotizaciones OMITIDAS — sin PDFs LIMPIOS
- **Severidad**: ALTA
- **Regla**: Cotizaciones con cero `DocumentoPDF` en estado `LIMPIO` se marcan `estado_analisis = OMITIDA` sin procesamiento. Evento `COTIZACION_OMITIDA` en AuditTrail.
- **Refs**: Decisión 7.3-A

### BR-U03-33: Reanudación idempotente del pipeline
- **Severidad**: ALTA
- **Regla**: Al arrancar, C-09 busca portafolios en estado `ANALISIS` con cotizaciones `PENDIENTE`/`EN_PROCESO`. Por cada PDF verifica si ya tiene `VariableExtraida` o `tipo_documento_clasificado != NULL`; si los tiene, omite reprocesamiento.
- **Refs**: Decisión 5.5-A

### BR-U03-34: ServicioPipeline invocable por PDF individual
- **Severidad**: ALTA
- **Regla**: C-09 expone `extraer_pdf(documento_pdf_id, cotizacion_id)` invocable independientemente. Contrato para U-04 (reingesta tras aclaraciones).
- **Refs**: Decisión 8.2-C

---

## Sección 7 — Eventos WebSocket y Notificaciones

### BR-U03-35: Eventos WebSocket granulares
- **Severidad**: MEDIA
- **Regla**: C-09 emite:
  - `EXTRACCION_INICIADA` (al arrancar)
  - `DOCUMENTO_CLASIFICADO` (por cada PDF clasificado, payload con `tipo_documento_clasificado`)
  - `EXTRACCION_PROGRESO` (por PDF completado)
  - `CRUCE_INICIADO`, `CRUCE_PROGRESO`
  - `ANALISIS_COMPLETADO` (al terminar)
- **Refs**: Decisión 5.4-A

### BR-U03-36: Notificación HITL al final + email opcional
- **Severidad**: MEDIA
- **Regla**: Sin notificación push por variable. Al terminar, `ANALISIS_COMPLETADO` incluye `{total_escalamientos, cotizaciones_con_revision}`. Adicionalmente: si una cotización supera `UMBRAL_EMAIL_ESCALAMIENTO` (default 5), email al Analista vía SMTP compartido con U-04.
- **Refs**: Decisión 6.2-C

---

## Sección 8 — Persistencia y Auditoría

### BR-U03-37: Persistencia llamadas LLM en S3 + tabla LlamadaLLM
- **Severidad**: ALTA
- **Regla**: Cada llamada LLM persiste:
  1. JSON completo `{system_prompt, user_message, tool_definition, response, model, tokens, latencia, timestamp}` en S3 (`portafolios/{pid}/llm_calls/{call_id}.json`)
  2. Registro `LlamadaLLM` en BD con S3 key + métricas + `tipo_documento_clasificado`
- **Propósito**: auditoría CISO (US-19), reproducibilidad.
- **Refs**: Decisión 8.1-A

### BR-U03-38: Métricas runtime Gate G1/G3
- **Severidad**: MEDIA
- **Regla**: Si `tasa_escalamiento_portafolio > UMBRAL_ALERTA_ESCALAMIENTO` (default 30%) → evento `ALERTA_DEGRADACION` para ADMIN y CISO. Adicionalmente, `valor_revisado != valor` (HITL) implica error de extracción registrado para `tasa_correcciones_llm` (panel Admin U-06).
- **Refs**: Decisión 8.3-A

---

## Sección 9 — Reglas Cross-Unit (acoplamiento U-02 ↔ U-03)

### BR-U03-39: Lectura de ItemCotizado/ItemSolicitud desde BD nuestra
- **Severidad**: ALTA
- **Regla**: C-04 MotorCruce lee `ItemCotizado` e `ItemSolicitud` de **nuestra BD** (ya snapshoteados por U-02 al ingestar). No consulta BD SAB durante U-03 — esa interacción es exclusiva de U-02.
- **Refs**: Separación de responsabilidades U-02 (ingesta) vs U-03 (procesamiento)

### BR-U03-40: Trigger de inicio Portafolio vía endpoint
- **Severidad**: ALTA
- **Regla**: SAB invoca `POST /portafolios/from-sab` con body `{solicitud_bienes_version, solicitud_bienes_fecha_version, solicitud_bienes_id}` (la tripleta de la composite key). Este endpoint vive en U-02; al recibirlo, U-02 valida contra BD SAB, crea Portafolio + jerarquía, e inicia ingesta. Cuando ingesta termina, dispara `INGESTA_COMPLETADA` y U-03 arranca (BR-U03-28).
- **Refs**: Decisión 27 (endpoint `POST /portafolios/from-sab`)

---

## Resumen de Reglas por Componente

| Componente | Reglas |
|---|---|
| C-03 MotorExtraccion | BR-U03-01, BR-U03-02, BR-U03-03, BR-U03-04, BR-U03-05, BR-U03-08, BR-U03-10, BR-U03-11, BR-U03-12, BR-U03-13, BR-U03-15, BR-U03-16, BR-U03-17, BR-U03-18 |
| C-04 MotorCruce | BR-U03-20, BR-U03-21, BR-U03-22, BR-U03-23, BR-U03-24, BR-U03-25, BR-U03-26, BR-U03-27, BR-U03-39 |
| C-09 ServicioPipeline | BR-U03-28, BR-U03-29, BR-U03-30, BR-U03-31, BR-U03-32, BR-U03-33, BR-U03-34, BR-U03-35, BR-U03-36, BR-U03-38, BR-U03-40 |
| C-11 GatewayLLM | BR-U03-06, BR-U03-07, BR-U03-09, BR-U03-14, BR-U03-19, BR-U03-37 |
