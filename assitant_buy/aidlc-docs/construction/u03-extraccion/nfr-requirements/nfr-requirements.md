# NFR Requirements — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — NFR Requirements
> **Fecha**: 2026-05-29
> **Scope MVP**: solo BIENES
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

---

## Nota de Herencia

U-03 **hereda todos los NFRs de U-01 (Fundación) y U-02 (Ingesta)**. Este documento define únicamente los NFRs **nuevos o específicos** del núcleo inteligente: extracción LLM, cruce determinista y orquestación del pipeline. En particular:

- Performance y **costo** de las llamadas LLM (Sonnet 4.6 + fallback Opus 4.7)
- Concurrencia por documento y throughput del análisis
- Gates de calidad G3 (anti-alucinación) y G1 (tasa de escalamiento)
- Resiliencia ante fallos/throttling del proveedor LLM
- Seguridad: anti-EchoLeak y auditoría de llamadas LLM para el CISO

> **Decisiones tomadas en**: plan `construction/plans/u03-extraccion-nfr-requirements-plan.md` (Q1.1–Q8.1, 2026-05-29).

---

## NFR-U03-01 — Performance y Costo del Pipeline LLM

### NFR-U03-01.1: SLA de latencia (Q1.1 = C)

**No se define SLA estricto de latencia por PDF.** El pipeline de U-03 corre asíncrono en background (disparado por `INGESTA_COMPLETADA`, BR-U03-28) y el usuario sigue el avance vía eventos WebSocket (BR-U03-35). La prioridad es la **calidad de extracción**, no la latencia individual.

| Atributo | Política |
|---|---|
| Latencia por PDF | Sin SLA duro; se observa p50/p95 como métrica informativa |
| Visibilidad de progreso | Eventos WebSocket `EXTRACCION_PROGRESO` por PDF completado |
| Latencia percibida | Mitigada por feedback en tiempo casi real (no por velocidad bruta) |

### NFR-U03-01.2: Registro de costo de tokens (Q1.2 = A)

| Atributo | Política |
|---|---|
| Persistencia de costo | Tokens (input/output) y costo estimado se registran por cada `LlamadaLLM` (BR-U03-37) |
| Agregación | Costo por portafolio **calculable on-demand** sumando `LlamadaLLM` (no panel permanente — ver NFR-U03-06) |
| Tope automático | ❌ Sin tope en el piloto. El consumo se observa para calibrar topes en producción |
| Doble llamada | Cada PDF puede generar hasta 2 `LlamadaLLM` (PRIMARIA Sonnet + FALLBACK Opus, BR-U03-06) — ambas contabilizadas |

**Rationale**: en un piloto supervisado con 1 portafolio a la vez, un tope automático con pausa de pipeline añade complejidad de estado sin valor proporcional. Se prioriza medir el costo real para dimensionar producción.

---

## NFR-U03-02 — Concurrencia y Throughput

### NFR-U03-02.1: Concurrencia por documento (Q2.1 = B)

| Atributo | Valor | Nota |
|---|---|---|
| `MAX_PDFS_CONCURRENTES` | **3** (configurable por env var) | Reducido de 5 → 3 por límites de Anthropic Tier 1 (ver NFR-U03-08) |
| Unidad de concurrencia | `DocumentoPDF` individual (BR-U03-30) | Semáforo asyncio |
| Cruce por cotización | MotorCruce arranca al completar todos los PDFs de la cotización | BR-U03-30 |

**Rationale**: con cuenta Anthropic nueva (Tier 1, ~40–50K TPM), 3 PDFs concurrentes × ~10K tokens ≈ 30K TPM deja margen seguro y reduce 429 proactivamente. Subir a 5 cuando el tier crezca (solo cambio de env var).

### NFR-U03-02.2: SLA de tiempo total por portafolio (Q2.2 = A)

| Atributo | Valor objetivo | Condición |
|---|---|---|
| Tiempo de extracción + cruce | **≤ 30 minutos** (objetivo blando) | Portafolio piloto: ≤ 200 PDFs LIMPIOS, concurrencia 3 |

**Nota de coherencia**: el objetivo de 30 min se mantiene con concurrencia **3** (≈ 200 PDFs ÷ 3 × ~20–25 s ≈ 22–28 min, más MotorCruce determinista que es rápido). Es un objetivo orientativo, no un SLA bloqueante (coherente con NFR-U03-01.1). El fallback Opus y el chunking (BR-U03-13) pueden extenderlo en portafolios con PDFs densos.

---

## NFR-U03-03 — Resiliencia del Pipeline LLM

### NFR-U03-03.1: Rate limits / throttling (Q1.3 = A)

| Escenario | Comportamiento |
|---|---|
| HTTP 429 (rate limit) | Tratado como transitorio → retry con backoff (NFR-U03-03.2); respeta header `retry-after` si está presente (manejado nativamente por el SDK `anthropic`) |
| HTTP 529 (overloaded) | Igual tratamiento que 429 |
| Sin token bucket global | El SDK con jitter evita thundering herd; no se implementa limitador de tasa propio en el piloto (tier aún desconocido) |
| Log operacional | Al agotar reintentos por 429: `WARNING` explícito sugiriendo reducir `MAX_PDFS_CONCURRENTES` o subir tier |

### NFR-U03-03.2: Política de retry (Q4.1 = A)

Confirma BR-U03-31:

| Intento | Espera |
|---|---|
| Reintento 1 | 2 s |
| Reintento 2 | 4 s |
| Reintento 3 | 8 s |
| Tras 3 fallos | `VariableExtraida` con `confianza=0.0`, `valor="ERROR_EXTRACCION"`, `requiere_revision=TRUE` por cada variable del catálogo |

### NFR-U03-03.3: Fallo sostenido del proveedor LLM (Q4.2 = B — sin circuit breaker)

| Atributo | Política |
|---|---|
| Circuit breaker | ❌ No implementado en el piloto |
| Comportamiento ante caída sostenida | Cada PDF agota sus 3 reintentos y queda con variables `ERROR_EXTRACCION`; el pipeline continúa con los demás PDFs |
| Recuperación | El Analista reintenta el portafolio manualmente; la **reanudación idempotente** (BR-U03-33) evita reprocesar PDFs ya completados |
| Failover a proveedor alterno | ❌ Fuera de MVP (solo `AnthropicProvider`, BR-U03-07) |

**Rationale**: piloto supervisado con 1 portafolio a la vez. La idempotencia de BR-U03-33 hace que el reintento manual sea barato. Un circuit breaker se reconsiderará en producción multi-portafolio.

---

## NFR-U03-04 — Calidad: Gates G3 y G1

### NFR-U03-04.1: Anti-alucinación — Gate G3 (Q3.1 = A)

| Atributo | Valor |
|---|---|
| Meta de tasa de alucinación | **< 1%** en el reporte final |
| Validación fuzzy (`rapidfuzz`) | Ratio mínimo **≥ 0.85** entre `referencia_fuente.fragmento_texto` y el texto de la página reportada (BR-U03-15) |
| Acción si ratio < 0.85 | `confianza = min(confianza_llm, 0.3)` + `requiere_revision = TRUE` |
| Garantía de reporte | Variables sin `valor_revisado` se **excluyen** del reporte para Comité (BR-U03-17) → 0% de invención en el reporte final |

### NFR-U03-04.2: Tasa de escalamiento — Gate G1 (Q3.2 = A)

| Atributo | Valor |
|---|---|
| `UMBRAL_ALERTA_ESCALAMIENTO` | **30%** (configurable) |
| Acción al superar | Evento `ALERTA_DEGRADACION` para ADMIN y CISO (BR-U03-38) |
| Umbrales de confianza | 🟢 ≥ 0.85 · 🟡 0.60–0.85 · 🔴 < 0.60 → escalamiento HITL (BR-U03-12) |

---

## NFR-U03-05 — Seguridad Específica de U-03

### NFR-U03-05.1: Anti-EchoLeak (heredado + reforzado)

| Regla | NFR |
|---|---|
| Una sesión LLM por documento | Cada llamada es independiente, sin conversation history — **por construcción arquitectónica** (BR-U03-14) |
| Spotlighting | El texto del PDF llega entre delimitadores dinámicos aplicados por U-02 (BR-U03-10) |
| 0 EchoLeak | Ningún output del LLM debe contener system prompts ni datos de otros competidores |

### NFR-U03-05.2: Auditoría de llamadas LLM para CISO (Q5.1 = A, Q5.2 = A)

| Atributo | Política |
|---|---|
| Contenido persistido | **Payload completo** en S3: `{system_prompt, user_message, tool_definition, response, model, tokens, latencia, timestamp}` (BR-U03-37) |
| Ubicación | `portafolios/{pid}/llm_calls/{call_id}.json` + registro `LlamadaLLM` en BD |
| Retención | **Indefinida durante el piloto** (igual que PDFs en U-02); política post-piloto según CISO |
| Cifrado | AES-256 en reposo + TLS en tránsito + acceso vía IAM Role (heredado de U-02 NFR-U02-04.4) |
| Reproducibilidad | El payload completo permite re-ejecutar y auditar cada extracción (US-19) |

### NFR-U03-05.3: Lectura del catálogo sin cache (Q5.3 = C)

| Atributo | Política |
|---|---|
| Cache de `CatalogoVariable` | ❌ Eliminado. C-11 lee directo de BD en cada extracción (BR-U03-19, revisado) |
| Concurrencia | Sin condición de carrera (sin estado compartido) → no requiere `asyncio.Lock` |
| Carga de BD | Despreciable (~24 variables, query trivial indexada) |
| Beneficio de seguridad/UX | Cambios del Admin (y `criticidad`) se aplican **de inmediato**, sin ventana de inconsistencia de 300s |

### NFR-U03-05.4: Logs sin datos de negocio (heredado de U-01/U-02)

Nunca se registran valores de negocio (proveedor, montos, contenido de cotizaciones) en logs de aplicación. La única persistencia del contenido es la auditoría CISO cifrada en S3 (NFR-U03-05.2).

---

## NFR-U03-06 — Observabilidad (Q7.1 = B)

Set de métricas **mínimo** para el piloto (hereda enfoque "logs + AuditTrail, sin Prometheus/Grafana" de U-02):

| Métrica expuesta | Fuente |
|---|---|
| **Tasa de escalamiento** por portafolio | `VariableExtraida.requiere_revision` + AuditTrail |
| **Errores LLM** (reintentos agotados, `ERROR_EXTRACCION`) | Logs `WARNING`/`ERROR` + AuditTrail |

**Disponible on-demand (no como métrica estándar)**: tokens/costo por portafolio, latencia LLM p50/p95, tasa de fallback Opus, tasa de alucinación detectada — todo consultable desde `LlamadaLLM` y `Discrepancia` cuando se requiera, sin dashboard permanente en el piloto.

| Evento | Nivel | Qué registra (sin datos de negocio) |
|---|---|---|
| Inicio extracción portafolio | `INFO` | `portafolio_id`, `n_cotizaciones` |
| PDF clasificado | `INFO` | `documento_pdf_id`, `tipo_documento_clasificado`, `confianza_clasificacion` |
| Reintento LLM | `WARNING` | `documento_pdf_id`, `intento`, `http_status` |
| Reintentos agotados (429) | `WARNING` | `documento_pdf_id` + sugerencia operacional (NFR-U03-03.1) |
| Cotización OMITIDA | `WARNING` | `cotizacion_id` (sin PDFs LIMPIOS) |
| `ALERTA_DEGRADACION` | `ERROR` | `portafolio_id`, `tasa_escalamiento` |
| Análisis completado | `INFO` | `portafolio_id`, `total_escalamientos`, `duracion_s` |

---

## NFR-U03-07 — Cobertura de Tests (Q7.2 = A)

| Módulo | Cobertura mínima | Prioridad |
|---|---|---|
| **Global U-03** | **80%** | Obligatorio antes de Build |
| `MotorCruce` (C-04) — aritmética + anti-fraude | **≥ 90%** | CRÍTICO — integridad financiera |
| Validación fuzzy Gate G3 (BR-U03-15) | **≥ 90%** | CRÍTICO — anti-alucinación |
| `MotorExtraccion` (C-03) — consolidación multi-PDF | 85% | ALTA |
| `GatewayLLM` (C-11) — tool schema dinámico, fallback | 85% | ALTA |
| `ServicioPipeline` (C-09) — orquestación, idempotencia | 80% | MEDIA |

**Tests PBT obligatorios** (extensión PBT parcial — PBT-02, 03, 07, 08, 09):

| ID | Propiedad |
|---|---|
| **PBT-U03-01** | Para cualquier `ItemCotizado`, las validaciones aritméticas (IVA, total unitario, total ítem — BR-U03-22) son consistentes: si los valores cumplen las fórmulas con tolerancia 0.01, no se genera `INCONSISTENCIA_ARITMETICA`; si no, sí se genera. |
| **PBT-U03-02** | Para cualquier par (fragmento, texto de página), la validación fuzzy (BR-U03-15) es monótona respecto al umbral: ratio ≥ 0.85 ⇒ no se penaliza confianza; ratio < 0.85 ⇒ `confianza ≤ 0.3` y `requiere_revision = TRUE`. |
| **PBT-U03-03** | Para cualquier conjunto de candidatos multi-PDF de una variable, la consolidación (BR-U03-18) selecciona siempre el de mayor `prioridad_tipo × confianza`, y `precio_total_cotizacion` solo se acepta de `COTIZACION_FORMAL`. |

---

## NFR-U03-08 — Capacity Planning Anthropic API (Q8.1 = A)

### NFR-U03-08.1: Dimensionamiento del piloto

| Atributo | Valor |
|---|---|
| Portafolios activos simultáneos | **1** (hereda U-02 NFR-U02-02.1) |
| Concurrencia interna | 3 PDFs (NFR-U03-02.1) |
| TPM estimado en pico | ~30K TPM (3 × ~10K tokens) |
| Escalado multi-portafolio | **TBD de producción** — requiere subir tier y reconsiderar token bucket (Q1.3-B) |

### NFR-U03-08.2: ⚠️ GATE de Code Generation — Cuenta Anthropic API

El usuario **no tiene cuenta** en `console.anthropic.com` (la suscripción Claude Max **no** habilita la API programática). Prerequisitos antes de Code Generation de U-03:

1. Crear cuenta en `console.anthropic.com`
2. Configurar método de pago (activa Tier 1)
3. Generar API key → almacenar en AWS Secrets Manager (nunca en código/logs)
4. Verificar tier real y recalibrar `MAX_PDFS_CONCURRENTES` / `ANTHROPIC_MAX_RETRIES`

> Documentado también en `cross-unit-deltas.md` §4.6.

---

## Resumen de NFRs por Categoría (U-03)

| Categoría | Requisito Clave |
|---|---|
| **Performance** | Sin SLA duro por PDF (async + WebSocket); ≤ 30 min/portafolio (objetivo blando, concurrencia 3) |
| **Costo** | Tokens/costo registrados por `LlamadaLLM`; sin tope automático en piloto |
| **Concurrencia** | `MAX_PDFS_CONCURRENTES=3` (Tier 1); subible por env var |
| **Resiliencia** | Retry 2/4/8s + `retry-after`; sin circuit breaker (reintento manual + idempotencia) |
| **Calidad** | Alucinación < 1% (fuzzy ≥ 0.85); alerta escalamiento > 30% |
| **Seguridad** | Anti-EchoLeak por construcción; auditoría LLM completa cifrada en S3 (retención indefinida piloto); catálogo sin cache |
| **Observabilidad** | Mínima: tasa escalamiento + errores LLM; resto on-demand desde `LlamadaLLM` |
| **Tests** | 80% global; ≥ 90% MotorCruce y Gate G3; 3 PBTs obligatorios |
| **Capacity** | 1 portafolio a la vez; **GATE**: crear cuenta Anthropic API antes de Code Gen |

---

## Compliance de Extensiones

### Security Baseline (✅ habilitado — todas las reglas bloqueantes)

| Aspecto | Estado | Nota |
|---|---|---|
| Secretos fuera de código/logs | ✅ Compliant | API key Anthropic en Secrets Manager (NFR-U03-08.2); 0 credenciales en logs (NFR-U03-05.4) |
| Cifrado en reposo/tránsito | ✅ Compliant | Auditoría LLM en S3 AES-256 + TLS (NFR-U03-05.2, heredado U-02) |
| Aislamiento de contenido adversarial | ✅ Compliant | Anti-EchoLeak por construcción + Spotlighting (NFR-U03-05.1) |
| Mínimo privilegio en logs | ✅ Compliant | Sin datos de negocio en logs (NFR-U03-05.4) |
| Auditoría/trazabilidad | ✅ Compliant | `LlamadaLLM` + AuditTrail para CISO (NFR-U03-05.2) |

### Property-Based Testing (✅ parcial — PBT-02, 03, 07, 08, 09)

| Regla | Aplicabilidad en NFR Requirements | Estado |
|---|---|---|
| **PBT-09** (selección de framework) | ✅ Aplica en esta etapa | `hypothesis` — ver `tech-stack-decisions.md` |
| PBT-02, 03 (round-trip / invariantes) | Se materializan en Code Generation | 3 PBTs identificados (NFR-U03-07) |
| PBT-07, 08 (calidad generadores / shrinking) | Se materializan en Code Generation | N/A en esta etapa |
