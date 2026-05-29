# Plan de Functional Design — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Functional Design
> **Fecha**: 2026-05-27
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM
> **Stories**: US-04 (Variables Extraídas), US-05 (Discrepancias), US-06 (Confianza y Fuente), US-07 (Escalamiento HITL), US-08 (Estado Incompleto)
> **Gates aplicables**: G1 (precisión > 96%), G3 (alucinación < 1%)

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta marcando una de las opciones (A/B/C) o usando `X` con su descripción. Cuando termine, indíquelo y el workflow generará los 3 artefactos de Functional Design de U-03 (`domain-entities.md`, `business-rules.md`, `business-logic-model.md`).

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Unit of work y stories revisados (unit-of-work.md + unit-of-work-story-map.md + stories.md)
- [x] Plan de preguntas generado
- [x] Respuestas recibidas y analizadas
- [x] Plan aprobado

### PART 2 — Generation
- [x] `domain-entities.md` generado
- [x] `business-rules.md` generado
- [x] `business-logic-model.md` generado

---

## Sección 1 — Variables a Extraer (C-03 MotorExtraccion)

### Pregunta 1.1 — Catálogo de variables

US-04 define que se extraen variables organizadas por categoría: Vigencia, Garantías, Penalidades, Condiciones de Pago, Exclusiones. ¿Cómo se define el **catálogo concreto** de variables a extraer por PDF?

A) **Catálogo fijo y cerrado** — el sistema tiene 5 categorías con un set fijo de ~15-20 variables predefinidas (ej: `vigencia_garantia`, `plazo_pago`, `monto_penalidad_retraso`, etc.). El mismo catálogo se aplica a todos los procesos del piloto.
B) **Catálogo configurable por Admin** — el catálogo de variables se define en una tabla `CatalogoVariable` que el Admin (US-22) puede ampliar/editar desde la UI. Se crean dinámicamente según el tipo de proceso.
C) **Catálogo por tipo de adquisición** — hay catálogos diferentes según el tipo de proceso SAB (bienes vs servicios vs obras). El sistema detecta el tipo y selecciona el catálogo apropiado.
X) Otro (describa después del tag [Answer]:)

[Answer]: X

[Description]: Catálogos separados por tipo de adquisición (Bienes / Servicios / Software). El tipo es seleccionado **manualmente por el Analista** al iniciar el análisis desde la UI — no se detecta automáticamente desde el SAB. Ver catálogo completo en Pregunta 1.2.

---

### Pregunta 1.2 — Variables exactas del catálogo MVP

¿Qué set de variables específicas constituye el catálogo MVP del piloto? (Marque la opción más cercana, podrá ajustarla en la respuesta X).

A) **Set mínimo (12 variables)**:
   - Vigencia: `vigencia_oferta`, `vigencia_garantia`, `fecha_entrega`
   - Garantías: `tipo_garantia`, `monto_garantia`, `cobertura_garantia`
   - Penalidades: `penalidad_retraso_porcentaje`, `monto_maximo_penalidad`
   - Pago: `plazo_pago_dias`, `forma_pago`
   - Exclusiones: `exclusiones_principales`, `condiciones_especiales`
B) **Set extendido (20 variables)** — lo de A más: `responsable_tecnico`, `tiempo_respuesta_soporte`, `nivel_servicio`, `clausula_confidencialidad`, `garantia_postventa`, `licencias_incluidas`, `costos_adicionales`, `descuentos_aplicables`
C) **Set financiero-focalizado (8 variables)** — solo lo crítico: `precio_total`, `precio_unitario`, `forma_pago`, `plazo_pago`, `monto_garantia`, `penalidad_retraso`, `vigencia_oferta`, `vigencia_entrega`
X) Otro (describa después del tag [Answer]:)

[Answer]: X

[Description]: Catálogos completos por tipo de adquisición con variables comunes + variables específicas:

**Variables Comunes (aplican a los 3 tipos — 20 variables base)**:
- Precio: `precio_total`, `costos_adicionales`, `descuentos_aplicables`
- Pago: `forma_pago`, `plazo_pago_dias`
- Vigencia: `vigencia_oferta`, `fecha_entrega`
- Garantías: `tipo_garantia`, `monto_garantia`, `cobertura_garantia`, `vigencia_garantia`, `garantia_postventa`
- Penalidades: `penalidad_retraso_porcentaje`, `monto_maximo_penalidad`
- Contractual: `clausula_confidencialidad`, `condiciones_especiales`, `exclusiones_principales`
- Soporte: `responsable_tecnico`, `tiempo_respuesta_soporte`, `nivel_servicio`

**Variables Específicas por Tipo**:

🔧 **Bienes** (20 comunes + 6 = 26 total):
`precio_unitario`, `cantidad`, `marca`, `modelo`, `especificaciones_tecnicas`, `lugar_entrega`

🛠️ **Servicios** (20 comunes + 4 = 24 total):
`costo_hora`, `vigencia_contrato`, `fecha_inicio_servicio`, `lugar_prestacion`

💻 **Software** (20 comunes + 8 = 28 total):
`precio_por_licencia`, `numero_licencias`, `tipo_licencia` (perpetua/suscripcion/open-source), `vigencia_licencia`, `incluye_soporte`, `incluye_actualizaciones`, `restricciones_uso`, `licencias_incluidas`

---

### Pregunta 1.3 — Variables ausentes en el PDF

¿Qué hace el sistema cuando una variable del catálogo **no aparece** en el PDF del proveedor?

A) Crea registro `VariableExtraida` con `valor = NULL`, `confianza = 0`, `requiere_revision = TRUE` y `analisis = "Variable no encontrada en el documento fuente"` — la variable aparece en el reporte con flag de escalamiento.
B) **No crea registro** — la variable simplemente no aparece en la síntesis del proveedor; el `MotorCruce` la detectará como `AUSENTE_EN_PDF` solo si está en SAB.
C) Crea registro con `valor = "NO_ENCONTRADO"` (string literal) y `confianza = 1.0` — se distingue entre "no encontrado" (confianza alta) y "ambiguo" (confianza baja).
X) Otro (describa después del tag [Answer]:)

[Answer]:C

---

## Sección 2 — División Document AI vs Claude (C-03 + C-11 + C-16)

### Pregunta 2.1 — Rol exacto de Document AI vs LLM en U-03

En U-02 ya se decidió que Amazon Textract hace el OCR de PDFs escaneados. En U-03, ¿cuál es el rol del LLM (Claude) y qué función adicional puede tener Document AI?

A) **Pipeline secuencial puro**: Texto del PDF (nativo + OCR companion file de U-02) → Claude extrae todas las variables en un prompt único estructurado. Document AI **no se usa más** en U-03.
B) **Pipeline híbrido**: Document AI (Textract AnalyzeDocument con FORMS+TABLES) extrae estructura tabular y key-value pairs → Claude refina e interpreta solo las variables que Document AI no pudo identificar con alta confianza.
C) **Doble pasada de validación**: Claude extrae primero, luego Document AI (Textract Queries) valida los valores extraídos preguntando "¿está X en este documento?". Discrepancias entre las dos respuestas marcan baja confianza.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 2.2 — Modelo LLM y proveedor

El PRD menciona "Anthropic Claude". ¿Qué modelo específico se usa y por qué?

A) **Claude Sonnet 4.6** (`claude-sonnet-4-6`) — balance precisión/costo. Modelo por defecto para extracción en producción.
B) **Claude Opus 4.7** (`claude-opus-4-7`) — máxima precisión, mayor costo. Justificado por gate G1 (>96% precisión) y G3 (<1% alucinación).
C) **Estrategia mixta**: Sonnet 4.6 por defecto + Opus 4.7 como fallback cuando la confianza de Sonnet sobre una variable está bajo umbral configurable.
X) Otro (describa después del tag [Answer]:)

[Answer]: X

[Description]: Estrategia mixta — Sonnet 4.6 por defecto + Opus 4.7 como fallback cuando el score de confianza de Sonnet sobre una variable está bajo umbral configurable (default: 0.75). C-11 GatewayLLM se diseña con interfaz abstracta `LLMProvider` extensible a múltiples proveedores, pero el MVP implementa solo el adaptador de Anthropic. Esto permite cambiar de proveedor en el futuro sin modificar C-03/C-04/C-09.

---

### Pregunta 2.3 — Estructura del prompt al LLM

¿Cómo se construye el prompt enviado a Claude para extraer las variables de un PDF?

A) **Un prompt por PDF, todas las variables a la vez**: El system prompt define las N variables del catálogo + reglas de output JSON; el user message contiene el texto del PDF entre delimitadores Spotlighting `[PDF_xxx_START]...[PDF_xxx_END]`. Output: JSON estructurado con N entradas.
B) **Un prompt por variable** (N llamadas por PDF): para cada variable del catálogo se hace una llamada independiente al LLM con foco específico. Más caro pero más preciso.
C) **Prompts por categoría** (5 llamadas por PDF): agrupa las variables por categoría (Vigencia, Garantías, etc.) en prompts separados para reducir distracción del modelo.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 2.4 — Tool use (function calling) vs JSON puro

¿Se usa el **tool use** de Claude (function calling) o **JSON estructurado en el output text**?

A) **Tool use con schema estricto**: definir `tool: extraer_variables` con `input_schema` JSON Schema que valida cada variable. Claude llama la tool con argumentos validados → garantiza output parseable.
B) **JSON en output text** con prompt instructivo: el system prompt instruye a Claude a responder ÚNICAMENTE con un JSON con cierto schema. Parseo con `json.loads` + validación Pydantic. Más simple, menos overhead.
C) **Tool use con extended thinking**: tool use + activar `thinking` block para que Claude razone antes de emitir el JSON. Mejor para gates G1/G3 pero más latencia y tokens.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 2.5 — Manejo de contexto excedido (chunking)

Si un PDF (texto + OCR) excede el contexto del modelo, ¿qué estrategia se aplica?

A) **Chunking por páginas con overlap**: dividir el PDF en chunks de N páginas con overlap de 1 página. Cada chunk produce un JSON parcial; un paso final consolida las extracciones (toma la de mayor confianza por variable).
B) **Sumario previo + extracción**: una primera llamada al LLM produce un resumen de cada sección del PDF; una segunda llamada extrae variables sobre el sumario consolidado. Riesgo: pérdida de detalle.
C) **Rechazar con escalamiento**: si el PDF excede el contexto del modelo seleccionado, se marca la cotización como `requiere_revision_hitl = TRUE` con motivo "PDF excede límite de procesamiento automático" y se le pide al Analista revisión manual.
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 3 — Modelo de Confianza y Referencias (US-06)

### Pregunta 3.1 — Cálculo del score de confianza

`VariableExtraida.confianza` es DECIMAL(5,4) entre 0.0000 y 1.0000. ¿Cómo se calcula este valor?

A) **Confianza reportada por el LLM**: Claude devuelve el score en el JSON (`{"valor": "...", "confianza": 0.92, ...}`). Es self-reported por el modelo basado en su propia certeza.
B) **Confianza compuesta**: media ponderada entre (1) score del LLM self-reported (50%), (2) si la referencia de fuente coincide textualmente con el valor en el PDF (30%), (3) si Document AI también extrajo el mismo valor (20%).
C) **Confianza por reglas**: heurística determinística — `1.0` si la referencia exacta del valor aparece en el PDF; `0.7` si aparece parafraseada; `0.5` si requiere interpretación; `0.0` si no se encuentra. Sin involucrar al LLM en el scoring.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 3.2 — Umbrales de confianza (Alta / Media / Baja)

US-06 muestra indicadores 🟢 Alta / 🟡 Media / 🔴 Baja. ¿Cuáles son los umbrales numéricos exactos sobre el score de confianza?

A) **Umbrales fijos en código** (no configurables):
   - Alta: `confianza >= 0.85`
   - Media: `0.60 <= confianza < 0.85`
   - Baja: `confianza < 0.60` → dispara escalamiento HITL (US-07)
B) **Umbrales configurables por Admin** en tabla `ConfiguracionSistema`:
   - `umbral_confianza_alta` (default: 0.85)
   - `umbral_confianza_baja` (default: 0.60) — bajo este umbral se dispara escalamiento
   - Cualquier cambio queda registrado en `AuditTrail` con evento `CONFIGURACION_CAMBIADA`
C) **Umbrales por categoría de variable** — distintos según criticidad financiera. Ej: variables de precio tienen umbral más estricto (Alta >= 0.92) que las de garantía (Alta >= 0.80).
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 3.3 — Estructura del campo `referencia_fuente`

El campo `VariableExtraida.referencia_fuente` (TEXT) debe permitir la navegación al fragmento exacto del PDF (US-06 escenario 1). ¿Qué estructura tiene este campo?

A) **JSON con coordenadas**: `{"documento_id": "...", "pagina": 7, "seccion": "Cláusula 4.2", "fragmento_texto": "El plazo de garantía será de 12 meses...", "bbox": [x1, y1, x2, y2]}` — permite resaltar el fragmento en el visor PDF con coordenadas físicas.
B) **JSON sin coordenadas**: `{"documento_id": "...", "pagina": 7, "fragmento_texto": "El plazo de garantía será de 12 meses..."}` — el visor busca el fragmento textual en la página para resaltarlo (sin bbox).
C) **Texto libre** (formato fijo): `"PDF: cotizacion_proveedor_x.pdf | Pág 7 | Sección 4.2 | \"El plazo de garantía será de 12 meses...\""` — sin parseo, solo visualización.
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 3.4 — Validación de la referencia (anti-alucinación, Gate G3)

¿Cómo garantiza el sistema que `referencia_fuente.fragmento_texto` **realmente existe** en el PDF (para prevenir alucinaciones)?

A) **Validación textual post-LLM**: después de recibir la respuesta del LLM, una función verifica que el `fragmento_texto` aparece literalmente en el texto extraído del PDF. Si no aparece, la `confianza` se baja a 0.3 y se marca `requiere_revision = TRUE`.
B) **Validación con fuzzy matching**: usa una distancia de Levenshtein o similar (ratio > 0.85) para verificar que el fragmento esté presente con tolerancia a normalización de espacios/encoding. Más robusto pero menos estricto.
C) **Sin validación post-LLM** — se confía en el LLM. La validación se hace solo en testing/QA contra el corpus D2 para medir Gate G3.
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

## Sección 4 — Cruce SAB vs PDF (C-04 MotorCruce)

### Pregunta 4.1 — Datos del SAB que se cruzan

¿Qué datos estructurados del SAB se cruzan con las variables extraídas del PDF? (recordar: en U-02 sabemos que el SAB provee metadatos + PDFs vía API REST).

A) **SAB tiene los campos completos** — el `RepositorioSAB` expone una tabla `sab_cotizaciones_detalle` con todos los campos del catálogo de variables (`vigencia`, `monto_garantia`, `forma_pago`, etc.) que el proveedor registró al cotizar. El cruce es 1:1 contra el PDF.
B) **SAB tiene solo lo financiero**: precio_total, precio_unitario, forma_pago, plazo_pago. Las variables de garantías/penalidades/exclusiones solo existen en el PDF (no hay cruce posible para ellas, solo extracción).
C) **SAB tiene un subset configurable** — el Admin define en `MapeoVariableSAB` qué variable del catálogo corresponde a qué campo SAB. Las no mapeadas no se cruzan.
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 4.2 — Normalización antes de comparar

Para comparar `valor_sab` con `valor_pdf` (US-05), ¿cómo se normalizan los valores?

A) **Normalización por tipo de variable**:
   - Monetario: convertir a Decimal, normalizar moneda (COP/USD), tolerancia 0.01
   - Fecha: parsear a ISO 8601, comparar exacto
   - Numérico: convertir a Decimal, tolerancia configurable por variable
   - Texto: lowercase, strip whitespace, comparar con fuzzy matching (ratio > 0.90)
B) **Comparación literal**: `valor_sab.strip().lower() == valor_pdf.strip().lower()` — sin normalización por tipo. Más simple pero genera falsos positivos.
C) **Comparación delegada al LLM**: un segundo prompt al LLM evalúa "¿son estos dos valores equivalentes en contexto de negocio?" — el LLM decide si hay discrepancia.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 4.3 — Clasificación de severidad (Alta / Media / Baja)

US-05 muestra severidad 🔴 Alta / 🟡 Media / 🟢 Baja, y la entidad `Discrepancia` definida en U-01 tiene ENUM `BAJA / MEDIA / ALTA / CRITICA`. ¿Cómo se clasifica cada discrepancia?

A) **Por tipo de discrepancia + tipo de variable** (reglas fijas):
   - `VALOR_DIFERENTE` en variable monetaria con diferencia > 10% → ALTA
   - `VALOR_DIFERENTE` en variable monetaria con diferencia 1-10% → MEDIA
   - `VALOR_DIFERENTE` en variable de texto → MEDIA
   - `AUSENTE_EN_PDF` en variable crítica (precio, vigencia, garantía) → ALTA
   - `AUSENTE_EN_PDF` en variable secundaria → BAJA
   - `SOLO_EN_PDF` (información extra del proveedor) → BAJA
   - Inconsistencias internas críticas (ej: vigencia oferta < fecha entrega) → CRITICA
B) **Por impacto financiero** (calculado): la severidad se deriva de un score `impacto = |diferencia_monetaria| / monto_referencia` para variables financieras, con thresholds configurables. Variables no monetarias siempre son MEDIA por defecto.
C) **Definida por el LLM**: para cada discrepancia, una llamada al LLM evalúa el contexto y asigna severidad con justificación textual. Más flexible pero más costoso y menos determinista.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 4.4 — Criticidad de variable (tagging)

Para que el cruce sepa qué variables son "críticas" vs "secundarias", ¿cómo se etiqueta cada variable del catálogo?

A) **Atributo `criticidad` en el catálogo**: cada variable del catálogo MVP tiene un atributo `criticidad: CRITICA | ALTA | MEDIA | BAJA` predefinido (ej: `vigencia_garantia: CRITICA`, `descuentos_aplicables: BAJA`).
B) **Lista hardcoded de variables críticas**: una constante `VARIABLES_CRITICAS = {"vigencia_oferta", "monto_garantia", "plazo_pago", "penalidad_retraso"}` define las críticas; el resto es secundario.
C) **Sin distinción**: todas las variables tienen el mismo peso. La severidad se calcula solo por tipo de discrepancia (sin considerar criticidad de la variable).
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 5 — Orquestación del Pipeline (C-09 ServicioPipeline)

### Pregunta 5.1 — Trigger del pipeline de U-03

¿Cómo arranca el pipeline de U-03? (recordar: U-02 termina cambiando `Portafolio.estado` de `INGESTA` a `ANALISIS`).

A) **Trigger síncrono en proceso**: la BackgroundTask de U-02, justo antes de cambiar el estado a `ANALISIS`, invoca directamente `ServicioPipeline.iniciar_extraccion(portafolio_id)` en la misma tarea asyncio. No hay cola intermedia.
B) **Trigger por evento (Pub/Sub interno)**: U-02 publica un evento `INGESTA_COMPLETADA` en un bus interno (asyncio Queue o similar); el `ServicioPipeline` está suscrito y arranca otro BackgroundTask para U-03.
C) **Trigger por endpoint explícito**: el cambio de estado del portafolio a `ANALISIS` queda visible, pero el Analista debe presionar "Iniciar Análisis" (segundo botón) para arrancar U-03. Da más control humano sobre cuándo se gasta dinero en LLM/Document AI.
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 5.2 — Granularidad de procesamiento (concurrencia)

Dentro del pipeline de U-03, ¿cuál es la unidad de procesamiento concurrente?

A) **Por cotización (proveedor)**: el pipeline procesa cada `Cotizacion` en paralelo (asyncio.gather con semáforo configurable, ej: 4 cotizaciones simultáneas). Dentro de cada cotización, los PDFs se procesan secuencialmente.
B) **Por PDF**: cada `DocumentoPDF` con estado `LIMPIO` se procesa concurrentemente con un semáforo global (ej: 8 PDFs simultáneos). La consolidación por cotización se hace al final.
C) **Procesamiento serializado**: cotizaciones y PDFs en serie (sin concurrencia) para evitar saturar Claude/Textract y simplificar manejo de errores. Trade-off: más lento pero más predecible.
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

### Pregunta 5.3 — Manejo de errores por etapa

Si una llamada al LLM falla (timeout, rate limit, error API) para un PDF específico, ¿cuál es el comportamiento?

A) **Retry con backoff exponencial + fallback**: 3 reintentos con 2s, 4s, 8s. Si todos fallan, la `Cotizacion` queda en `EN_PROCESO` con un `VariableExtraida` por cada variable del catálogo (todas con `confianza=0` y `analisis="Error en extracción IA"`) — el Analista las verá como pendientes de revisión.
B) **Marcar cotización como ERROR**: el primer fallo después de retries marca `Cotizacion.estado_analisis = ERROR` y registra el motivo en `AuditTrail`. El pipeline continúa con las demás cotizaciones del portafolio.
C) **Fallback a modelo alternativo**: si Sonnet falla 3 veces, se intenta con Opus (mayor capacidad y rate limit independiente). Si Opus también falla, se marca como ERROR.
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 5.4 — Estado de progreso WebSocket

¿Qué eventos WebSocket emite el `ServicioPipeline` durante U-03?

A) **Eventos granulares por etapa**:
   - `EXTRACCION_INICIADA` (al arrancar U-03)
   - `EXTRACCION_PROGRESO` (por cada PDF terminado): `{cotizacion_actual, pdfs_procesados, pdfs_totales, variables_extraidas, variables_baja_confianza}`
   - `CRUCE_INICIADO` (al pasar a fase de cruce)
   - `CRUCE_PROGRESO` (por cada cotización cruzada)
   - `ANALISIS_COMPLETADO` (al final): `{total_discrepancias, total_escalamientos, total_errores}`
B) **Eventos compactos**: solo `EXTRACCION_PROGRESO` con un payload que incluye toda la información (fase actual, % global, métricas). Menos mensajes pero más datos por mensaje.
C) **Solo eventos críticos**: `EXTRACCION_INICIADA`, `ANALISIS_COMPLETADO`, `ANALISIS_ERROR`. Sin progreso intermedio (el Analista actualiza la UI manualmente cuando vea que el pipeline está corriendo).
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 5.5 — Idempotencia y reanudación

Si el pipeline de U-03 se interrumpe a la mitad (crash del worker, deploy, etc.), ¿qué pasa al reiniciar?

A) **Reanudación idempotente**: al arrancar, el sistema busca portafolios en estado `ANALISIS` con cotizaciones `PENDIENTE` o `EN_PROCESO`; recalcula qué PDFs ya tienen `VariableExtraida` registradas y reanuda desde donde quedó. Las extracciones ya completadas no se reprocesan.
B) **Sin reanudación automática**: los portafolios interrumpidos quedan en `ERROR` con motivo `pipeline_interrumpido`; el Analista debe reiniciar manualmente desde la UI (botón "Reintentar análisis"). El reintento procesa solo lo pendiente.
C) **Reproceso completo**: cualquier interrupción dispara un reproceso total del portafolio al reiniciar (borrando las extracciones previas). Garantiza consistencia pero es caro.
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 6 — Escalamiento HITL (US-07)

### Pregunta 6.1 — Trigger del escalamiento

¿Qué condiciones marcan una variable como `requiere_revision = TRUE` y a su `Cotizacion` como `requiere_revision_hitl = TRUE`?

A) **Solo por confianza baja**: `confianza < umbral_baja` (default 0.60) → `VariableExtraida.requiere_revision = TRUE`. Si la cotización tiene **al menos una** variable con `requiere_revision = TRUE`, entonces `Cotizacion.requiere_revision_hitl = TRUE`.
B) **Por confianza baja OR validación textual fallida** (ver pregunta 3.4): cualquiera de las dos condiciones dispara el escalamiento.
C) **Por confianza baja OR fragmento ausente en PDF OR discrepancia ALTA/CRITICA**: tres condiciones disparan escalamiento. Es la más estricta — minimiza riesgo de error pasando al Comité, maximiza intervención del Analista.
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 6.2 — Notificación del escalamiento al Analista

US-07 dice "notifica al Analista con el contexto: variable, PDF fuente, fragmento ambiguo". ¿Cuándo y cómo se notifica?

A) **Notificación al final del pipeline (no por variable)**: cuando termina U-03, el evento WebSocket `ANALISIS_COMPLETADO` incluye `{total_escalamientos: N, cotizaciones_con_revision: [...]}`. La UI muestra un badge con el número. No hay notificación push por cada variable.
B) **Notificación granular en tiempo real**: cada vez que se detecta una variable que requiere revisión durante el pipeline, se emite un evento WebSocket `ESCALAMIENTO_DETECTADO` con el contexto completo. La UI muestra una lista que va creciendo.
C) **Notificación al final + email opcional**: igual que A, pero adicionalmente si la cotización tiene > N variables escaladas (configurable), se envía un email al Analista con el resumen para que actúe aunque no esté en la UI.
X) Otro (describa después del tag [Answer]:)

[Answer]:C

---

### Pregunta 6.3 — Garantía de no-invención (Gate G3)

US-07 escenario 2 exige "tasa de valores inventados es 0% para estas variables". ¿Cómo se garantiza?

A) **Combinación de validación textual + valor NULL**: cuando se dispara escalamiento por baja confianza, `VariableExtraida.valor` se setea explícitamente a `NULL` (sobrescribiendo lo que el LLM devolvió). Solo el campo `analisis` queda con el contexto del LLM. El Analista solo verá `NULL` hasta que ingrese `valor_revisado`.
B) **Conservar el valor del LLM con flag visible**: `VariableExtraida.valor` mantiene lo que el LLM devolvió, pero la UI muestra un indicador 🔴 grande "Valor sugerido por IA — no verificado". El reporte para Comité **no incluye** valores con `requiere_revision = TRUE` hasta que haya `valor_revisado`.
C) **Validación post-pipeline**: al terminar U-03, un proceso de auditoría verifica que cada `VariableExtraida.valor IS NOT NULL AND requires_revision = FALSE` tiene su `referencia_fuente.fragmento_texto` presente en el PDF. Si falla, se baja a `requiere_revision = TRUE` y `valor = NULL`.
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 7 — Estado de Cotización Incompleto (US-08)

### Pregunta 7.1 — Definición de los estados Completa / Parcial / No procesable

US-08 muestra 3 secciones: ✅ Completas / ⚠️ Parciales / ❌ No procesables. ¿Cómo se mapean estos estados a los datos persistidos?

A) **Atributo derivado en runtime** sin nueva columna: el estado se calcula al vuelo desde:
   - **Completa**: `Cotizacion.estado_analisis = COMPLETADO` AND todos los PDFs tienen `estado_sanitizacion = LIMPIO`
   - **Parcial**: `estado_analisis = COMPLETADO` AND al menos 1 PDF con `estado_sanitizacion IN (ADVERSARIAL, ERROR)` (pero al menos uno LIMPIO)
   - **No procesable**: `estado_analisis = ERROR` OR todos los PDFs en (ADVERSARIAL, ERROR) OR cero PDFs
B) **Nueva columna `estado_ingesta`** en `Cotizacion`: ENUM `COMPLETA | PARCIAL | NO_PROCESABLE` actualizada por U-02/U-03 según corresponda. Se persiste y se consulta directamente.
C) **Vista materializada** `vw_cotizacion_estado_ingesta` que calcula los 3 estados con joins; las consultas leen esa vista en lugar de calcular cada vez.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 7.2 — Motivo del estado Parcial / No procesable

US-08 dice "cada cotización en las secciones Parcial y No procesable tiene el motivo listado (PDF corrupto, PDF no encontrado, Sin anexos, etc.)". ¿Dónde se almacena el motivo?

A) **Derivado de los eventos `AuditTrail`** de la cotización: se consultan los eventos relacionados al `cotizacion_id` (ej: `PDF_ADVERSARIAL_DETECTADO`, `SANITIZACION_APLICADA` con estado `ERROR`) y se muestran como motivos. Sin nuevo campo en BD.
B) **Nuevo campo `motivo_estado_ingesta`** TEXT NULLABLE en `Cotizacion`: actualizado por U-02/U-03 con el motivo principal (ej: `"2 de 3 PDFs no sanitizables: 1 corrupto, 1 con inyección de prompt"`). Construcción del texto en el backend.
C) **Tabla nueva `EventoCotizacion`** con (`cotizacion_id`, `tipo`, `mensaje`, `created_at`) — registra cada evento de ingesta de forma estructurada, la UI los muestra como timeline de la cotización.
X) Otro (describa después del tag [Answer]:)

[Answer]: X

[Description]: Derivado en runtime desde `DocumentoPDF` records (no desde AuditTrail). Cada `DocumentoPDF` tiene un campo `motivo_error` TEXT NULLABLE (poblado por U-02 cuando el estado es ADVERSARIAL o ERROR con texto legible: "PDF corrupto", "Inyección de prompt detectada", "PDF protegido con contraseña"). El motivo de la cotización se construye como `@property` agregando los motivos de sus PDFs problemáticos. Sin nueva columna en `Cotizacion`. Consistente con la filosofía de 7.1 (derivar sin persistir estado redundante). Implicación: U-02 es responsable de poblar `DocumentoPDF.motivo_error`.

---

### Pregunta 7.3 — Impacto del estado en U-03

Cuando una cotización es **No procesable** en U-02, ¿qué hace U-03?

A) **U-03 ignora cotizaciones No procesable**: el `ServicioPipeline` filtra cotizaciones con 0 PDFs `LIMPIO` y no las procesa. Su `estado_analisis` queda en `PENDIENTE` (o se actualiza a un estado especial `OMITIDA`).
B) **U-03 procesa con datos parciales**: si una cotización tiene 0 PDFs LIMPIOS pero tiene datos en SAB, U-03 ejecuta el `MotorCruce` reportando "PDF no disponible" en todas las variables. Permite ver al menos los datos del SAB.
C) **U-03 marca como ERROR**: cualquier cotización sin PDFs procesables queda en `estado_analisis = ERROR` con motivo, sin intentar ningún procesamiento.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 8 — Topics Transversales

### Pregunta 8.1 — Persistencia del prompt usado (auditoría y reproducibilidad)

Para auditoría (US-19 CISO) y reproducibilidad de extracciones, ¿se persiste el prompt exacto enviado al LLM por cada PDF?

A) **Persistir en S3 (no en BD)**: cada llamada al LLM guarda en S3 (`portafolios/{pid}/llm_calls/{call_id}.json`) el objeto `{system_prompt, user_message, response, model, tokens, latency, timestamp}`. La BD solo guarda `llm_call_s3_key` en una tabla nueva `LlamadaLLM` que referencia la `VariableExtraida` (o el batch de variables).
B) **Persistir en `AuditTrail` con metadata limitada**: evento `LLM_INVOCADO` con `metadata = {model, prompt_tokens, completion_tokens, latency_ms, success}` — sin el contenido del prompt (para no inflar la BD).
C) **No persistir** — solo registrar métricas agregadas (tokens/día, latencia promedio) para control de costos. Reproducibilidad no es requisito.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 8.2 — Reingesta desde aclaraciones (interacción con U-04)

Cuando llega una respuesta de proveedor con PDFs (US-13, U-04), esos PDFs entran al pipeline U-02 → U-03 con un PDF marcado como respuesta de aclaración. ¿Cómo afecta U-03?

A) **U-03 reextrae solo el PDF nuevo y consolida**: las `VariableExtraida` previas se mantienen; se ejecuta extracción sobre el PDF nuevo; se reemplazan las variables solo si la nueva extracción tiene mayor confianza que la anterior O si responde a una variable que tenía `requiere_revision = TRUE`.
B) **U-03 reprocesa toda la cotización**: la llegada de un PDF de aclaración invalida las extracciones previas; U-03 vuelve a extraer todas las variables incluyendo el PDF nuevo + los originales. Más simple, más caro.
C) **No es responsabilidad de U-03 en este Functional Design** — la lógica de reingesta se define en U-04 (HITL) y solo entra al alcance de U-03 cuando U-04 lo invoque explícitamente. Esta unidad asume un único pipeline de extracción por cotización.
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 8.3 — Métricas de Gate G1 / G3 (cómo se miden en runtime)

Los gates G1 (precisión > 96%) y G3 (alucinación < 1%) se miden contra corpus D2/D3 en testing. ¿Hay alguna métrica en runtime (producción) que monitoree degradación?

A) **Métrica en runtime: tasa de escalamiento HITL** — si el % de variables con `requiere_revision = TRUE` excede un umbral configurable (ej: > 30%), se emite una alerta al Admin/CISO ("posible degradación de extracción"). Sin reevaluación de precisión exacta en producción.
B) **Muestreo manual periódico**: el Analista marca con un botón "validado/correcto" las variables que verifica manualmente en su flujo normal; el sistema calcula precisión empírica mensual y la muestra en panel Admin.
C) **Sin medición en runtime** — los gates se miden solo en CI/CD contra corpus D2/D3 antes de cada deploy. En producción no se reevalúa.
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 8.4 — Aislamiento de contexto entre cotizaciones (anti-EchoLeak)

En U-02 (pregunta 6.1) se definió que EchoLeak incluye fuga de "datos de cotizaciones de competidores". ¿Cómo garantiza U-03 que el LLM no mezcle contexto entre cotizaciones?

A) **Una sesión LLM por PDF, sin contexto persistente**: cada llamada al LLM es completamente independiente (sin conversation history). El prompt nunca contiene datos de otras cotizaciones del mismo portafolio. Garantía por construcción.
B) **Una sesión LLM por cotización (no por PDF)**: para optimizar caché de prompt, una cotización puede procesarse en varios mensajes dentro de una sesión, pero distintas cotizaciones nunca comparten sesión. El system prompt valida explícitamente este aislamiento.
C) **Validación post-respuesta**: cada respuesta del LLM pasa por un check que verifica si menciona nombres de otros proveedores del mismo portafolio. Si los menciona, la respuesta se rechaza y se marca como `posible_echoleak` para revisión.
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

> **Próximo paso**: Una vez completadas todas las respuestas, indíquelo y el workflow generará los 3 artefactos de Functional Design de U-03:
> - `aidlc-docs/construction/u03-extraccion/functional-design/domain-entities.md`
> - `aidlc-docs/construction/u03-extraccion/functional-design/business-rules.md`
> - `aidlc-docs/construction/u03-extraccion/functional-design/business-logic-model.md`

---

## Revisión 2026-05-28 — Re-alineación con schema SAB real

Tras aprobar la primera versión, se revisó el schema completo de SAB (`assitant_buy/sab/bd_sab.sql`) y se identificaron 10 decisiones adicionales que **rectifican** el modelo. La revisión está documentada en [`../u03-extraccion/DescripcionFluo.md`](../u03-extraccion/DescripcionFluo.md) v3.

### Decisiones adicionales (21-30)

- **D21**: Portafolio = `solicitud_bienes` completa (1:1 con composite key + radicado de referencia).
- **D22**: Identificación por tripleta `(version, fecha_version, id)` pasada desde SAB.
- **D23**: Nueva entidad `SolicitudCotizacion` (espejo SAB, agrupa cotizaciones por categoría).
- **D24**: Sin entidad `Categoria` de nuestro lado — solo FK suelta `categoria_id_sab`.
- **D25**: Sin entidad `Proveedor` maestra — snapshot (NIT + razón social) en `Cotizacion` al ingestar.
- **D26**: Trigger SAB = `solicitud_cotizacion.estado='CERRADA'` AND `cotizacion.estado='ACEPTADA_PARA_ESTUDIO'`.
- **D27**: Endpoint nuevo `POST /portafolios/from-sab` invocado desde SAB UI con la tripleta.
- **D28**: Comparación U-05 por ítem cruzando todas las `SolicitudCotizacion` del Portafolio.
- **D29**: Queries con partition pruning sobre `cotizacion_item.fecha_cotizacion` y `solicitud_item.fecha_version`.
- **D30**: Opcional — `vista_cuadro_comparativo` SAB como referencia auxiliar para U-05.

### Cambios técnicos derivados

- ❌ **Eliminado** `ParserExcelSAB` — los datos estructurados ya están en `cotizacion_item` SAB.
- ✅ **IVA dinámico** desde tabla `porcentaje_iva` (no hardcoded {0, 5, 19}).
- ✅ **Catálogo de variables dinámico** en BD (`CatalogoVariable`) — Admin editable con TTL cache 300s.
- ✅ **Clasificación + extracción** en **una sola** llamada LLM (tool use con `tipo_documento_clasificado` como primer campo).
- ✅ **Consolidación multi-PDF** por `prioridad_tipo × confianza` (COTIZACION_FORMAL=1.0, FICHA_TECNICA=0.9, ANEXO_GENERICO=0.7).
- ✅ **Validaciones aritméticas** sobre datos SAB sin LLM (IVA unitario, total unitario, total ítem, cantidad RFQ).
- ✅ **Scope MVP reducido a BIENES** — Servicios y Software se agregan luego sin cambios arquitectónicos.

### Deltas requeridos en otras unidades

Documentados en [`../u03-extraccion/functional-design/cross-unit-deltas.md`](../u03-extraccion/functional-design/cross-unit-deltas.md):

- **U-01**: 5 entidades nuevas (`CatalogoVariable`, `LlamadaLLM`, `SolicitudCotizacion`, `ItemSolicitud`, `ItemCotizado`) + 4 ALTER TABLE.
- **U-02**: Eliminar `ParserExcelSAB`, expandir `RepositorioSAB` con queries reales SAB, nuevo endpoint `POST /portafolios/from-sab`.
- **U-06**: CRUD `CatalogoVariable` + seed data 24 variables BIENES.

### TBDs operacionales

- Acceso BD SAB (read replica, credenciales, conexión segura)
- Mecanismo de invocación SAB UI → endpoint (auth, retry)
- Sincronización de estado SAB ↔ nuestra BD (recomendado: snapshot inmutable para MVP)
- Política de retención S3 para llamadas LLM

### Artefactos actualizados

- [x] `domain-entities.md` — v2 con nuevas entidades, sin catálogo estático
- [x] `business-rules.md` — v2 con BR 01-40 incluyendo validaciones aritméticas y clasificación
- [x] `business-logic-model.md` — v2 con modelo de 4 módulos refinado
- [x] `cross-unit-deltas.md` — NUEVO con deltas detallados

---

> **Estado final**: U-03 Functional Design **COMPLETADO (revisado)** el 2026-05-28. Aprobación pendiente del usuario.
