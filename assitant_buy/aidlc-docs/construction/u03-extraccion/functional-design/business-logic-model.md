# Business Logic Model — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Functional Design
> **Versión**: v2 (revisión 2026-05-28)
> **Scope MVP**: solo BIENES
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

---

## Descripción General

U-03 implementa el núcleo inteligente del pipeline de análisis con 4 responsabilidades:

1. **GatewayLLM (C-11)** — Abstracción sobre proveedores de IA. Construye el tool schema **dinámico** desde `CatalogoVariable` (cache TTL 300s). Implementa estrategia mixta Sonnet/Opus + clasificación + extracción en una sola llamada.
2. **MotorExtraccion (C-03)** — Extrae variables contractuales y clasifica el documento en una sola pasada LLM. Valida referencias (Gate G3). Consolida resultados entre múltiples PDFs de una misma cotización por prioridad de tipo de documento.
3. **MotorCruce (C-04)** — Cruza la variable LLM `precio_total_cotizacion` contra Σ(ItemCotizado del SAB) — defensa anti-fraude. Ejecuta validaciones aritméticas determinísticas por ítem sobre datos SAB. Detecta inconsistencias internas entre variables LLM.
4. **ServicioPipeline (C-09)** — Orquesta procesamiento concurrente por PDF, filtra cotizaciones por estado SAB (`CERRADA` + `ACEPTADA_PARA_ESTUDIO`), gestiona ciclo de vida del portafolio, emite eventos WebSocket, garantiza reanudación idempotente.

---

## Módulo 1: GatewayLLM (C-11) — Abstracción de Proveedor IA

### 1.1 Interfaz `LLMProvider`

```python
class LLMProvider(ABC):
    @abstractmethod
    async def clasificar_y_extraer(
        self,
        texto_pdf: str,
        catalogo: list[CatalogoVariable],
        metadata_origen: dict,
    ) -> ResultadoClasificacionExtraccion:
        ...
```

### 1.2 Cache del Catálogo

```
C-11 mantiene:
    _catalogo_cache: list[CatalogoVariable] | None = None
    _catalogo_cached_at: datetime | None = None

C-11._refrescar_catalogo_si_expiro():
    si _catalogo_cache es None OR
       (now() - _catalogo_cached_at).seconds > CATALOGO_TTL_SEGUNDOS (300):
        _catalogo_cache = SELECT * FROM CatalogoVariable
                          WHERE activo = TRUE
                            AND tipo_adquisicion = 'BIENES'  # MVP
        _catalogo_cached_at = now()
```

### 1.3 Adaptador `AnthropicProvider` — Clasificación + Extracción + Sonnet/Opus

```
AnthropicProvider.clasificar_y_extraer(texto_pdf, catalogo, metadata_origen)
    │
    ▼
╔═══ LLAMADA PRIMARIA — Sonnet 4.6 ═══════════════════════════════════╗
│                                                                      │
│ Construir tool definition dinámico:                                  │
│   tool = {                                                           │
│     "name": "clasificar_y_extraer",                                  │
│     "input_schema": {                                                │
│       "type": "object",                                              │
│       "properties": {                                                │
│         "tipo_documento_clasificado": {                              │
│           "type": "string",                                          │
│           "enum": ["COTIZACION_FORMAL", "FICHA_TECNICA",             │
│                    "CERTIFICADO", "CARTA_PRESENTACION",              │
│                    "ANEXO_GENERICO", "NO_RECONOCIDO"]                │
│         },                                                           │
│         "confianza_clasificacion": {"type":"number", "min":0,"max":1}│
│         "variables": {                                               │
│           "type": "array",                                           │
│           "items": {                                                 │
│             "type": "object",                                        │
│             "properties": {                                          │
│               "nombre": {"enum": [v.nombre for v in catalogo]},      │
│               "valor": {"type": "string"},                           │
│               "confianza": {"type": "number", "min":0, "max":1},     │
│               "referencia_fuente": {                                 │
│                 "type": "object",                                    │
│                 "properties": {                                      │
│                   "pagina": {"type": "integer"},                     │
│                   "seccion": {"type": "string"},                     │
│                   "fragmento_texto": {"type": "string"}              │
│                 }                                                    │
│               },                                                     │
│               "analisis": {"type": "string"}                         │
│             }                                                        │
│           }                                                          │
│         }                                                            │
│       },                                                             │
│       "required": ["tipo_documento_clasificado",                     │
│                    "confianza_clasificacion", "variables"]           │
│     }                                                                │
│   }                                                                  │
│                                                                      │
│ Construir messages:                                                  │
│   system = (                                                         │
│     "Eres un extractor de variables contractuales. Primero clasifica │
│      el documento por contenido (no por nombre). Luego extrae las    │
│      variables que apliquen al tipo de documento.\n"                 │
│     "Tipos:\n"                                                       │
│     "- COTIZACION_FORMAL: documento formal de cotización del         │
│       proveedor con términos, vigencia, garantías, formas de pago.\n"│
│     "- FICHA_TECNICA: datasheet con especificaciones, marca, modelo.\n"
│     "- CERTIFICADO: certificación de calidad/cumplimiento.\n"        │
│     "- CARTA_PRESENTACION: carta institucional sin variables.\n"     │
│     "- ANEXO_GENERICO: otros documentos relevantes.\n"               │
│     "- NO_RECONOCIDO: no puedes determinar el tipo.\n\n"             │
│     "Para CERTIFICADO/CARTA_PRESENTACION/NO_RECONOCIDO: deja         │
│      variables=[] (no extraigas).\n"                                 │
│     "Si una variable aplica pero no aparece, usa                     │
│      valor='NO_ENCONTRADO' con confianza=1.0.\n\n"                   │
│     f"Catálogo de variables a buscar:\n{catalogo_descripcion}\n"     │
│   )                                                                  │
│   user = texto_pdf  # Ya delimitado con Spotlighting por U-02        │
│                                                                      │
│ Llamar API: anthropic.messages.create(                              │
│   model="claude-sonnet-4-6",                                         │
│   tools=[tool], tool_choice={"type":"tool", "name":"clasificar..."}, │
│   messages=[{"role":"user", "content": user}],                       │
│   system=system                                                      │
│ )                                                                    │
│                                                                      │
│ Extraer tool_use block; parsear input → ResultadoClasificacion...    │
│ Registrar LlamadaLLM(tipo=PRIMARIA, modelo, tipo_documento_*)        │
│ Subir JSON a S3: portafolios/{pid}/llm_calls/{call_id}.json          │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
Si tipo_documento_clasificado IN [CERTIFICADO, CARTA_PRESENTACION, NO_RECONOCIDO]:
    → Retornar resultado (sin fallback Opus — no hay variables que mejorar)
    │
    ▼
Filtrar variables con confianza < UMBRAL_FALLBACK_OPUS (0.75)
    │
    ├─ Sin variables bajo umbral → retornar resultado Sonnet
    │
    └─ Hay variables bajo umbral:
╔═══ LLAMADA FALLBACK — Opus 4.7 ════════════════════════════════════╗
│                                                                      │
│ Construir tool SOLO con las variables de baja confianza             │
│ (subconjunto del catálogo). Mantiene tipo_documento_clasificado     │
│ del resultado primario (no se re-clasifica).                        │
│                                                                      │
│ Llamar API: anthropic.messages.create(                              │
│   model="claude-opus-4-7",                                           │
│   tools=[tool_subset], tool_choice=...,                              │
│   messages=[{"role":"user", "content": user}],                       │
│   system=system_extraccion_solo  # sin instrucciones de clasificación│
│ )                                                                    │
│                                                                      │
│ Registrar LlamadaLLM(tipo=FALLBACK, modelo="claude-opus-4-7")        │
│                                                                      │
│ Merge: para cada variable fallback, reemplazar el resultado Sonnet  │
│ con el resultado Opus                                                │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
Retornar ResultadoClasificacionExtraccion (Sonnet + Opus merged)
```

---

## Módulo 2: MotorExtraccion (C-03)

### 2.1 Flujo Principal de Extracción por PDF

```
C-03.extraer(contexto: ContextoExtraccion) → ResultadoClasificacionExtraccion
    │
    ▼
╔═══ PASO 1: Preparar texto del PDF ══════════════════════════════════╗
│                                                                      │
│ Si DocumentoPDF.texto_ocr_s3_key IS NOT NULL:                       │
│     texto_pdf = S3.download(texto_ocr_s3_key)                        │
│ Else:                                                                │
│     texto_pdf = pypdf.extract_text(pdf_bytes)                        │
│                                                                      │
│ Verificar tokens Spotlighting (BR-U03-10)                            │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 2: Evaluar tamaño del contexto ═════════════════════════════╗
│                                                                      │
│ tokens_estimados = len(texto_pdf) / 4                                │
│                                                                      │
│ Si tokens_estimados <= LIMITE_CONTEXTO_MODELO:                       │
│     → procesamiento directo                                          │
│ Else:                                                                │
│     → chunking 20 páginas + overlap 1 (§2.2)                         │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 3: Llamada al GatewayLLM ═══════════════════════════════════╗
│                                                                      │
│ catalogo = await C-11.obtener_catalogo()  # Cache TTL 300s          │
│                                                                      │
│ resultado_llm = await C-11.clasificar_y_extraer(                    │
│     texto_pdf = texto_pdf,                                           │
│     catalogo = catalogo,                                             │
│     metadata_origen = {                                              │
│         "proveedor": contexto.nombre_proveedor,                      │
│         "portafolio_id": contexto.portafolio_id,                     │
│         "tipo_adquisicion": "BIENES"                                 │
│     }                                                                │
│ )                                                                    │
│                                                                      │
│ # Retry con backoff: 2s, 4s, 8s (BR-U03-31)                         │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 4: Persistir clasificación en DocumentoPDF ═════════════════╗
│                                                                      │
│ UPDATE DocumentoPDF SET                                              │
│     tipo_documento_clasificado = resultado_llm.tipo_documento_*,     │
│     confianza_clasificacion = resultado_llm.confianza_clasificacion, │
│     extraccion_omitida = (tipo_documento_* IN                        │
│                            [CERTIFICADO, CARTA_PRESENTACION,         │
│                             NO_RECONOCIDO]),                         │
│     procesado_por_ia = TRUE                                          │
│ WHERE id = documento_pdf_id                                          │
│                                                                      │
│ AuditTrail → DOCUMENTO_CLASIFICADO                                   │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
Si extraccion_omitida == TRUE:
    return resultado_llm  # Sin Paso 5/6/7 — variables vacías
    │
    ▼
╔═══ PASO 5: Validación Fuzzy Anti-Alucinación (Gate G3) ═════════════╗
│                                                                      │
│ PARA CADA variable EN resultado_llm.variables:                       │
│     Si variable.valor == "NO_ENCONTRADO": saltar                     │
│                                                                      │
│     Si variable.referencia_fuente.fragmento_texto IS NOT NULL:       │
│         texto_pagina = extraer_pagina(texto_pdf, variable.ref.pagina)│
│         ratio = rapidfuzz.fuzz.partial_ratio(                        │
│             variable.referencia_fuente.fragmento_texto,              │
│             texto_pagina                                             │
│         )                                                            │
│         │                                                            │
│         ├─ ratio >= 0.85 → referencia válida                         │
│         │                                                            │
│         └─ ratio < 0.85 → POSIBLE ALUCINACIÓN:                       │
│              variable.confianza = min(variable.confianza, 0.3)       │
│              variable.requiere_revision = TRUE                       │
│                                                                      │
│     Si referencia_fuente IS NULL:                                    │
│         variable.confianza = min(actual, 0.5)                        │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 6: Umbrales de confianza (BR-U03-12) ═══════════════════════╗
│                                                                      │
│ PARA CADA variable:                                                  │
│     Si variable.confianza < 0.60:                                    │
│         variable.requiere_revision = TRUE                            │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
Retornar resultado_llm  (con variables enriquecidas)
```

### 2.2 Flujo de Chunking por Páginas (intra-PDF)

```
C-03._extraer_con_chunking(texto_pdf, catalogo) → ResultadoClasificacionExtraccion
    │
    ▼
Dividir texto_pdf en páginas (separador: [PAGINA_N])
    │
    ▼
Crear chunks de CHUNK_PAGINAS (default 20) con overlap de 1 página
    │
    ▼
PARA CADA chunk:
    resultado_chunk = await C-11.clasificar_y_extraer(texto_chunk, catalogo)
    resultados_por_chunk.append(resultado_chunk)
    │
    ▼
CLASIFICACION final: tipo_documento_clasificado con mayor confianza_clasificacion
                     entre los chunks (vota la mayoría con desempate por confianza)
    │
    ▼
CONSOLIDACIÓN INTRA-PDF — por variable, mayor confianza entre chunks:
    variables_finales = {}
    PARA CADA chunk_result:
        PARA CADA variable EN chunk_result.variables:
            si nombre NOT IN variables_finales OR
               variable.confianza > variables_finales[nombre].confianza:
                variables_finales[nombre] = variable
    │
    ▼
Retornar ResultadoClasificacionExtraccion consolidado
```

---

## Módulo 3: Consolidación Multi-PDF de una Cotización (C-03)

Cuando una cotización tiene varios `DocumentoPDF`, una misma variable puede aparecer en varios. C-03 ejecuta consolidación TRAS terminar la extracción de todos los PDFs de una cotización.

### 3.1 Flujo de Consolidación Multi-PDF

```
C-03.consolidar_cotizacion(cotizacion_id) → list[VariableExtraida]
    │
    ▼
documentos = SELECT * FROM DocumentoPDF
              WHERE cotizacion_id = :id
                AND procesado_por_ia = TRUE
                AND extraccion_omitida = FALSE
    │
    ▼
candidatos_por_variable = {}  # dict[nombre_variable, list[CandidatoConsolidacion]]
    │
    ▼
PARA CADA documento EN documentos:
    PARA CADA variable_extraida_pdf EN documento.variables_pdf_temp:
        candidato = CandidatoConsolidacion(
            documento_pdf_id = documento.id,
            tipo_documento = documento.tipo_documento_clasificado,
            variable = variable_extraida_pdf,
            score_consolidacion =
              PRIORIDAD_TIPO_DOCUMENTO[documento.tipo_documento_clasificado]
              * variable_extraida_pdf.confianza
        )
        candidatos_por_variable[variable_extraida_pdf.nombre].append(candidato)
    │
    ▼
PARA CADA (nombre_variable, candidatos) EN candidatos_por_variable:
    │
    ├─ Caso especial: nombre_variable == "precio_total_cotizacion"
    │     Filtrar candidatos → solo de tipo_documento = COTIZACION_FORMAL
    │     Si no hay candidatos COTIZACION_FORMAL:
    │         variable_extraida = None (no se persiste)
    │         continue
    │
    └─ Caso general:
          ganador = max(candidatos, key=lambda c: c.score_consolidacion)
          alternativos = [c for c in candidatos if c != ganador]
    │
    ▼
    INSERT VariableExtraida:
        cotizacion_id, documento_id = ganador.documento_pdf_id,
        nombre = nombre_variable,
        valor = ganador.variable.valor,
        confianza = ganador.variable.confianza,
        referencia_fuente = ganador.variable.referencia_fuente,
        analisis = ganador.variable.analisis,
        requiere_revision = ganador.variable.requiere_revision,
        fuente_consolidacion = {
            "tipo_documento_ganador": ganador.tipo_documento,
            "score_prioridad": ganador.score_consolidacion,
            "candidatos_alternativos": [
                {"documento_id": a.documento_pdf_id,
                 "tipo": a.tipo_documento,
                 "score": a.score_consolidacion}
                for a in alternativos
            ]
        }
    │
    ▼
    AuditTrail → CONSOLIDACION_VARIABLE {cotizacion_id, nombre, ganador_id, n_candidatos}
```

---

## Módulo 4: MotorCruce (C-04)

### 4.1 Flujo Principal de Cruce

```
C-04.cruzar(cotizacion_id) → ResultadoCruce
    │
    ▼
╔═══ PASO 1: Cargar datos consolidados ═══════════════════════════════╗
│                                                                      │
│ variables_pdf = SELECT * FROM VariableExtraida                       │
│                 WHERE cotizacion_id = :id                            │
│                                                                      │
│ items_cotizados = SELECT ic.*, isol.cantidad_solicitada              │
│                    FROM ItemCotizado ic                              │
│                    JOIN ItemSolicitud isol ON ic.item_solicitud_id = isol.id
│                    WHERE ic.cotizacion_id = :id                      │
│                                                                      │
│ # NO se consulta BD SAB aquí — los items ya están en nuestra BD     │
│ # (snapshoteados por U-02 en la ingesta — BR-U03-39)                │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 2: Cruce LLM ↔ SAB — precio_total_cotizacion ═══════════════╗
│ (BR-U03-20)                                                          │
│                                                                      │
│ var_precio = variables_pdf.get("precio_total_cotizacion")            │
│ precio_total_pdf = parsear_monetario(var_precio.valor) if var_precio │
│                    else None                                         │
│                                                                      │
│ precio_total_sab = SUM([ic.valor_total_item_con_iva                 │
│                          for ic in items_cotizados if ic.cotizado])  │
│                                                                      │
│ Si precio_total_pdf IS None:                                        │
│     → Discrepancia AUSENTE_EN_PDF, severidad ALTA                    │
│                                                                      │
│ Si precio_total_pdf IS NOT None:                                    │
│     diferencia = abs(precio_total_pdf - precio_total_sab)            │
│     Si diferencia > Decimal("1.00"):                                 │
│         → Discrepancia INCONSISTENCIA_ARITMETICA, severidad ALTA     │
│         → check = "cruce_total_pdf_sab"                              │
│         → AuditTrail CRUCE_ARITMETICO_FALLIDO                        │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 3: Validaciones Aritméticas por Ítem (BR-U03-22, 23, 24) ═══╗
│                                                                      │
│ porcentajes_iva_activos = SELECT porcentaje FROM porcentaje_iva_sab │
│                            WHERE activo = TRUE                       │
│   (consultado vía RepositorioSAB — query corta cacheable)            │
│                                                                      │
│ PARA CADA ic EN items_cotizados WHERE cotizado = TRUE:               │
│                                                                      │
│   # Check 1: IVA unitario                                            │
│   iva_esperado = ic.valor_unidad_sin_iva * ic.porcentaje_iva / 100   │
│   iva_real = ic.valor_unidad_con_iva - ic.valor_unidad_sin_iva       │
│   Si abs(iva_esperado - iva_real) > Decimal("0.01"):                 │
│     → Discrepancia INCONSISTENCIA_ARITMETICA, ALTA, "iva_unitario"   │
│                                                                      │
│   # Check 2: Total unitario con IVA                                  │
│   precio_con_iva_esperado = ic.valor_unidad_sin_iva                  │
│                              * (1 + ic.porcentaje_iva / 100)         │
│   Si abs(precio_con_iva_esperado - ic.valor_unidad_con_iva)          │
│      > Decimal("0.01"):                                              │
│     → Discrepancia INCONSISTENCIA_ARITMETICA, ALTA, "total_unitario" │
│                                                                      │
│   # Check 3: Total ítem                                              │
│   total_esperado = ic.valor_unidad_con_iva * ic.cantidad_cotizada    │
│   Si abs(total_esperado - ic.valor_total_item_con_iva)               │
│      > Decimal("0.01"):                                              │
│     → Discrepancia INCONSISTENCIA_ARITMETICA, ALTA, "total_item"     │
│                                                                      │
│   # Check 4: % IVA en lista activa SAB                               │
│   Si ic.porcentaje_iva NOT IN porcentajes_iva_activos:               │
│     → Discrepancia INCONSISTENCIA_ARITMETICA, MEDIA, "iva_valido"    │
│                                                                      │
│   # Check 5: Cantidad cotizada vs solicitada                         │
│   Si ic.cantidad_cotizada != ic.cantidad_solicitada:                 │
│     → Discrepancia INCONSISTENCIA_ARITMETICA, ALTA, "cantidad_rfq"   │
│                                                                      │
│ # Items con cotizado=FALSE no generan discrepancias (BR-U03-25)     │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 4: Inconsistencias Internas entre Variables LLM ════════════╗
│ (BR-U03-26)                                                          │
│                                                                      │
│ vigencia_oferta = variables_pdf.get("vigencia_oferta")               │
│ fecha_entrega = variables_pdf.get("fecha_entrega")                   │
│ vigencia_garantia = variables_pdf.get("vigencia_garantia")           │
│ monto_max_penalidad = variables_pdf.get("monto_maximo_penalidad")    │
│ precio_total = variables_pdf.get("precio_total_cotizacion")          │
│                                                                      │
│ Si vigencia_oferta < fecha_entrega:                                  │
│     → Discrepancia INCONSISTENCIA_INTERNA, CRITICA                   │
│                                                                      │
│ Si vigencia_garantia < fecha_entrega:                                │
│     → Discrepancia INCONSISTENCIA_INTERNA, CRITICA                   │
│                                                                      │
│ Si monto_max_penalidad > precio_total:                               │
│     → Discrepancia INCONSISTENCIA_INTERNA, ALTA                      │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PASO 5: Persistir Discrepancias ═════════════════════════════════╗
│                                                                      │
│ INSERT INTO Discrepancia × M                                         │
│                                                                      │
│ AuditTrail → CRUCE_COMPLETADO                                        │
│   {cotizacion_id, n_discrepancias, n_alta, n_critica, n_aritmeticas} │
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.2 Flujo de Normalización por Tipo

```
C-04._normalizar(valor: str, tipo: TipoVariable) → NormalizedValue
    │
    ▼
Si tipo == MONETARIO:
    Eliminar símbolos: "$", "COP", "USD", ".", ","
    Extraer moneda si presente: "COP" | "USD" | None
    Convertir a Decimal
    Si moneda detectada distinta a COP → flag MONEDA_DIFERENTE

Si tipo == FECHA:
    Parsear con dateutil + locale español: "dd/mm/yyyy" | "yyyy-mm-dd" | "dd de mes de yyyy"
    Retornar date ISO 8601

Si tipo == NUMERICO:
    Extraer número de strings como "30 días" → Decimal(30)

Si tipo == TEXTO:
    valor.strip().lower()

Si tipo == BOOLEANO:
    "sí|si|yes|true|1|incluye" → True
    "no|false|0|no incluye" → False
```

---

## Módulo 5: ServicioPipeline (C-09) — Orquestación

### 5.1 Trigger por Evento INGESTA_COMPLETADA

```
ServicioPipeline.__init__():
    bus_interno.subscribe("INGESTA_COMPLETADA", self._on_ingesta_completada)

async def _on_ingesta_completada(evento):
    portafolio_id = evento["portafolio_id"]
    asyncio.create_task(self.iniciar_analisis(portafolio_id))
```

### 5.2 Flujo Principal del Pipeline de Análisis

```
ServicioPipeline.iniciar_analisis(portafolio_id)
    │
    ▼
╔═══ INICIO ══════════════════════════════════════════════════════════╗
│                                                                      │
│ portafolio = SELECT * FROM Portafolio WHERE id = :id                 │
│ tipo_adquisicion = "BIENES"  # MVP                                   │
│                                                                      │
│ # Filtrar cotizaciones por estado SAB (BR-U03-29)                   │
│ cotizaciones = SELECT c.*                                            │
│                FROM Cotizacion c                                     │
│                JOIN SolicitudCotizacion sc                           │
│                  ON c.solicitud_cotizacion_id = sc.id                │
│                WHERE sc.portafolio_id = :portafolio_id               │
│                  AND sc.estado_sab = 'CERRADA'                       │
│                  AND c.estado_sab = 'ACEPTADA_PARA_ESTUDIO'          │
│                  AND c.estado_analisis IN ('PENDIENTE','EN_PROCESO') │
│                                                                      │
│ AuditTrail → ANALISIS_INICIADO_U03 {                                 │
│   portafolio_id, n_solicitudes_cotizacion, n_cotizaciones,           │
│   n_pdfs_limpios                                                     │
│ }                                                                    │
│ WebSocket → EXTRACCION_INICIADA                                      │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ CLASIFICAR COTIZACIONES (PDFs disponibles) ══════════════════════╗
│                                                                      │
│ PARA CADA cotizacion:                                                │
│     pdfs_limpios = [p for p in cotizacion.documentos                 │
│                     if p.estado_sanitizacion == "LIMPIO"]            │
│     │                                                                │
│     ├─ len(pdfs_limpios) == 0:                                       │
│     │      UPDATE Cotizacion.estado_analisis = "OMITIDA"             │
│     │      AuditTrail → COTIZACION_OMITIDA (motivo: sin_pdfs_limpios)│
│     │                                                                │
│     └─ len(pdfs_limpios) > 0:                                        │
│            pdfs_a_procesar.extend(pdfs_limpios)                      │
│            UPDATE Cotizacion.estado_analisis = "EN_PROCESO"          │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ PROCESAMIENTO CONCURRENTE POR PDF (BR-U03-30) ═══════════════════╗
│                                                                      │
│ semaforo = asyncio.Semaphore(MAX_PDFS_CONCURRENTES)  # default 5    │
│                                                                      │
│ tareas = [_procesar_pdf(pdf, semaforo) for pdf in pdfs_a_procesar]   │
│                                                                      │
│ await asyncio.gather(*tareas, return_exceptions=True)                │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ CIERRE DEL PORTAFOLIO ═══════════════════════════════════════════╗
│                                                                      │
│ Calcular métricas finales:                                           │
│   total_discrepancias = COUNT(Discrepancia WHERE portafolio)         │
│   total_aritmeticas = COUNT WHERE tipo = INCONSISTENCIA_ARITMETICA  │
│   total_escalamientos = COUNT(VariableExtraida WHERE requiere_rev)   │
│   total_errores = COUNT(Cotizacion WHERE estado_analisis = ERROR)    │
│   cotizaciones_con_revision = [c.id WHERE requiere_revision_hitl]    │
│                                                                      │
│ WebSocket → ANALISIS_COMPLETADO                                      │
│ AuditTrail → ANALISIS_COMPLETADO                                     │
│                                                                      │
│ Si total_escalamientos_max_por_cotizacion > UMBRAL_EMAIL (5):        │
│     EmailService.enviar_resumen_escalamientos(analista, resumen)     │
│     (reutiliza SMTP de C-05 GestorAclaraciones)                      │
│                                                                      │
│ Si tasa_escalamiento > UMBRAL_ALERTA (30%):                          │
│     WebSocket → ALERTA_DEGRADACION (a ADMIN + CISO)                  │
╚══════════════════════════════════════════════════════════════════════╝
```

### 5.3 Flujo de Procesamiento Individual de un PDF

```
async _procesar_pdf(pdf: DocumentoPDF, semaforo) → None
    │
    ▼
async with semaforo:  # Límite concurrencia
    │
    ▼
╔═══ Idempotencia (BR-U03-33) ════════════════════════════════════════╗
│                                                                      │
│ Si pdf.procesado_por_ia == TRUE OR                                   │
│    pdf.tipo_documento_clasificado IS NOT NULL:                       │
│     return  # Ya procesado en ejecución previa                       │
╚══════════════════════════════════════════════════════════════════════╝
    │
    ▼
contexto = ContextoExtraccion(
    documento_pdf_id = pdf.id,
    cotizacion_id = pdf.cotizacion_id,
    portafolio_id = portafolio_id,
    nombre_proveedor = cotizacion.proveedor_razon_social,
    tipo_adquisicion = "BIENES",
    texto_pdf = _cargar_texto_pdf(pdf),
    tokens_spotlighting = pdf.tokens_spotlighting,
    catalogo_variables = await C-11.obtener_catalogo()
)
    │
    ▼
try:
    resultado = await MotorExtraccion.extraer(contexto)
    │
    ▼
    WebSocket → DOCUMENTO_CLASIFICADO {documento_id, tipo_documento_*}
    WebSocket → EXTRACCION_PROGRESO {cotizacion_id, documento_id, n_variables...}
    │
    ▼
    # Verificar si todos los PDFs de la cotización están procesados
    Si _todos_pdfs_cotizacion_procesados(pdf.cotizacion_id):
        # Consolidación multi-PDF (§3.1)
        await MotorExtraccion.consolidar_cotizacion(pdf.cotizacion_id)
        │
        ▼
        # Cruce
        await MotorCruce.cruzar(pdf.cotizacion_id)
        │
        ▼
        UPDATE Cotizacion.estado_analisis = "COMPLETADO"
        UPDATE Cotizacion.puntaje_global = calcular_puntaje(...)
        UPDATE Cotizacion.requiere_revision_hitl = (∃ var con requiere_revision)
        WebSocket → CRUCE_PROGRESO

except LLMCallError after retries (BR-U03-31):
    _crear_variables_error(pdf, contexto.catalogo_variables)
    UPDATE Cotizacion.estado_analisis = "ERROR"
```

### 5.4 Reanudación Idempotente (BR-U03-33)

```
ServicioPipeline._reanudar_en_startup()
    │
    ▼  (FastAPI lifespan event al iniciar la aplicación)
    │
portafolios_pendientes = SELECT * FROM Portafolio
                          WHERE estado = "ANALISIS"
    │
    ▼
PARA CADA portafolio:
    cotizaciones_incompletas = SELECT * FROM Cotizacion c
                                JOIN SolicitudCotizacion sc ON ...
                                WHERE sc.portafolio_id = :id
                                  AND c.estado_analisis IN ("PENDIENTE","EN_PROCESO")
    │
    ├─ Vacío: skip
    │
    └─ Con incompletas:
           asyncio.create_task(iniciar_analisis(portafolio.id))
           # _procesar_pdf verifica idempotencia → solo lo no procesado
```

---

## Módulo 6: Interacciones Entre Módulos (Diagrama Completo)

```
[SAB UI]
    │ usuario presiona "Analizar con AssistentBuy"
    │ POST /portafolios/from-sab
    │ body: {solicitud_bienes_version, fecha_version, id}
    ▼
[U-02 endpoint /portafolios/from-sab]
    │
    ├─ RepositorioSAB queries BD SAB:
    │      solicitud_bienes → radicado → solicitud_cotizacion ×M (CERRADA)
    │      → cotizacion ×K (ACEPTADA_PARA_ESTUDIO)
    │      → cotizacion_item ×J (partition pruning por fecha)
    │      → cotizacion_documento metadata
    │      → proveedor + porcentaje_iva
    │
    ├─ GatewaySAB descarga binarios de PDFs (REST)
    │
    ├─ SanitizadorZeroTrust + OCR Textract si necesario
    │
    └─ Persiste en BD nuestra:
        Portafolio + SolicitudCotizacion ×M + ItemSolicitud ×N (1×Portafolio)
        + Cotizacion ×K + ItemCotizado ×J + DocumentoPDF ×L
    │
    │ publica INGESTA_COMPLETADA {portafolio_id}
    ▼
[Bus Interno asyncio.Queue]
    │ ServicioPipeline suscrito
    ▼
[C-09 ServicioPipeline]
    │ filtra cotizaciones por estado SAB (CERRADA + ACEPTADA_PARA_ESTUDIO)
    │
    ├─ Cotización sin PDFs LIMPIOS → OMITIDA
    │
    └─ PDFs LIMPIOS → semáforo(5) → asyncio.gather
           │
           ▼
    [C-03 MotorExtraccion]  ←→  [C-11 GatewayLLM]
           │                           │
           │                    Cache CatalogoVariable (TTL 300s)
           │                           │
           │                    AnthropicProvider — UNA llamada por PDF:
           │                    ├─ Sonnet 4.6 (primaria, clasifica + extrae)
           │                    └─ Opus 4.7 (fallback si conf < 0.75)
           │                           │
           │                    LlamadaLLM → S3 + BD (CISO)
           │
           ├─ UPDATE DocumentoPDF.tipo_documento_clasificado, extraccion_omitida
           │
           ├─ Validación Fuzzy (rapidfuzz, ratio ≥ 0.85)
           │   └─ Alucinación → confianza = min(actual, 0.3)
           │
           ▼ (todos PDFs de cotización procesados)
    [C-03.consolidar_cotizacion]
           │
           ├─ Por variable: ganador = max(prioridad_tipo × confianza)
           ├─ precio_total_cotizacion: SOLO desde COTIZACION_FORMAL
           └─ VariableExtraida ×N consolidada → BD (con fuente_consolidacion)
           │
           ▼
    [C-04 MotorCruce]
           │
           ├─ Cruce precio_total_cotizacion (PDF) vs Σ ItemCotizado SAB
           │   └─ Δ > 1.00 COP → INCONSISTENCIA_ARITMETICA ALTA
           │
           ├─ Validaciones aritméticas por ItemCotizado
           │   ├─ IVA unitario, total unitario, total ítem
           │   ├─ % IVA ∈ porcentaje_iva activos SAB
           │   └─ cantidad_cotizada == cantidad_solicitada
           │
           └─ Inconsistencias internas variables LLM
               └─ vigencia_oferta < fecha_entrega → CRITICA, etc.
           │
           ▼
    Discrepancia ×M → BD
    Cotizacion: COMPLETADO | puntaje_global | requiere_revision_hitl
           │
           ▼ (todas cotizaciones completas)
    WebSocket → ANALISIS_COMPLETADO
    Email opcional → Analista (si escalamientos > UMBRAL_EMAIL)
    Si tasa_escalamiento > 30%: ALERTA_DEGRADACION → ADMIN + CISO
```

---

## Resumen de Flujos por Historia de Usuario

| Historia | Flujo clave |
|---|---|
| **US-04** (Variables Extraídas) | §1.3 GatewayLLM clasificar+extraer; §2.1 MotorExtraccion; §3.1 Consolidación multi-PDF |
| **US-05** (Discrepancias) | §4.1 MotorCruce — cruce precio total + aritméticas + inconsistencias |
| **US-06** (Confianza y Fuente) | §2.1 Paso 5 fuzzy validation; `referencia_fuente` con `fragmento_texto` |
| **US-07** (Escalamiento HITL) | §5.2 ANALISIS_COMPLETADO + email; Gate G3 conserva valor LLM con flag |
| **US-08** (Estado Incompleto) | §5.2 OMITIDA si sin PDFs LIMPIOS; `motivo_error` derivado de DocumentoPDF |
| **US-19** (CISO Auditoría) | §1.3 LlamadaLLM → S3 + BD; AuditTrail con eventos LLM_INVOCADO + DOCUMENTO_CLASIFICADO |
| **US-22** (Admin) | §1.2 cache CatalogoVariable refresca cambios Admin en máximo 5 min |
