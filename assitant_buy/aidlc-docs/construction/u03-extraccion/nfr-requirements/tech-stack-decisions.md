# Tech Stack Decisions — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — NFR Requirements
> **Fecha**: 2026-05-29
> **Scope MVP**: solo BIENES
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

---

## Herencia del Stack de U-01 / U-02

U-03 **hereda íntegramente** el tech stack de U-01 (Python 3.12, FastAPI 0.115.x, SQLAlchemy 2.x async, PostgreSQL 16, Alembic, pydantic-settings, Docker, `asyncpg`) y de U-02 (`httpx` async, `aiobotocore` para S3, `hypothesis` para PBT, `moto`). Este documento documenta **únicamente las decisiones adicionales** del núcleo inteligente: el cliente LLM, la validación fuzzy y la orquestación concurrente.

---

## TSD-U03-01: Cliente / SDK del LLM (Q6.1 = A)

| Atributo | Decisión |
|---|---|
| **Librería** | `anthropic` (SDK oficial, modo **async** — `AsyncAnthropic`) |
| **Versión** | `0.40.x` o superior compatible con tool use y Claude 4.x |
| **Modo** | Async, integrado al event loop de FastAPI/asyncio |
| **Tool use** | Nativo del SDK — `messages.create(tools=[...], tool_choice=...)` |
| **Retry** | `max_retries` del SDK (backoff exponencial + jitter, respeta `retry-after`) — alineado a NFR-U03-03.1 |
| **Modelos** | Primario `claude-sonnet-4-6`; fallback `claude-opus-4-7` (BR-U03-06) |

**Rationale**: el SDK oficial maneja nativamente tool use, reintentos con `retry-after` y serialización de respuestas. Implementa la interfaz `LLMProvider` (BR-U03-07) como `AnthropicProvider`, desacoplando C-03/C-04/C-09 del proveedor.

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic(
    api_key=settings.ANTHROPIC_API_KEY,   # desde Secrets Manager (NFR-U03-08.2)
    max_retries=settings.ANTHROPIC_MAX_RETRIES,  # default 3 (BR-U03-31)
)
```

> ⚠️ **Gate**: requiere cuenta en `console.anthropic.com` + API key (NFR-U03-08.2). La suscripción Claude Max NO habilita la API.

---

## TSD-U03-02: Validación Fuzzy — Gate G3 (Q6.2 = A)

| Atributo | Decisión |
|---|---|
| **Librería** | `rapidfuzz` 3.x |
| **Uso** | Verificar que `referencia_fuente.fragmento_texto` exista en la página del PDF (BR-U03-15) |
| **Función** | `rapidfuzz.fuzz.partial_ratio` (normalizado 0–1, umbral `FUZZY_RATIO_MIN` = 0.85) |
| **Backend** | C++ nativo — órdenes de magnitud más rápido que `thefuzz`/`fuzzywuzzy` |

**Rationale**: `rapidfuzz` es la opción más rápida y mantenida; su backend C++ permite validar miles de fragmentos sin penalizar el pipeline. Licencia MIT (vs GPL de `fuzzywuzzy`).

---

## TSD-U03-03: Manejo de Valores Monetarios y Aritmética (C-04)

| Atributo | Decisión |
|---|---|
| **Tipo** | `decimal.Decimal` (stdlib) — **nunca `float`** para montos |
| **Uso** | Validaciones aritméticas IVA/total unitario/total ítem (BR-U03-22), cruce anti-fraude (BR-U03-20) |
| **Tolerancia** | 0.01 COP en comparaciones aritméticas; 1.00 COP en cruce `precio_total_cotizacion` |
| **Normalización** | `Decimal` con `quantize` para separadores; parseo de fechas a ISO 8601 (BR-U03-27) |

**Rationale**: la integridad financiera anti-fraude exige aritmética exacta. `float` introduce errores de redondeo inaceptables en validaciones con tolerancia de centavos.

---

## TSD-U03-04: Orquestación Concurrente (C-09)

| Atributo | Decisión |
|---|---|
| **Concurrencia** | `asyncio.Semaphore(MAX_PDFS_CONCURRENTES)` — default **3** (NFR-U03-02.1) |
| **Distribución** | `asyncio.gather` sobre PDFs LIMPIOS de cada cotización (BR-U03-30) |
| **Bus de eventos** | `asyncio.Queue` interno para `INGESTA_COMPLETADA` (BR-U03-28) — sin broker externo en piloto |
| **Idempotencia** | Verificación en BD (`VariableExtraida` / `tipo_documento_clasificado != NULL`) antes de reprocesar (BR-U03-33) |
| **Sin circuit breaker** | Q4.2 = B — reintento manual + idempotencia (NFR-U03-03.3) |

**Rationale**: asyncio nativo es suficiente para la concurrencia del piloto (1 portafolio a la vez). No se introduce Celery/RQ ni broker externo — coherente con el enfoque "sin infraestructura pesada" de U-01/U-02.

---

## TSD-U03-05: Catálogo sin Cache (Q5.3 = C)

| Atributo | Decisión |
|---|---|
| **Acceso** | Lectura directa de `CatalogoVariable` desde BD (SQLAlchemy async) en cada extracción (BR-U03-19, revisado) |
| **Sin cache en memoria** | No se mantiene estado `_catalogo_cache` ni `asyncio.Lock` |
| **Query** | `SELECT * FROM catalogo_variable WHERE activo = TRUE AND tipo_adquisicion = 'BIENES'` |

**Rationale**: catálogo pequeño (~24 variables) → carga de BD despreciable. Elimina complejidad de invalidación/locking y refleja cambios del Admin de inmediato.

---

## TSD-U03-06: Variables de Entorno Nuevas (U-03)

```env
# ─── Anthropic LLM ───────────────────────────────────────────────
ANTHROPIC_API_KEY=...                 # SECRETO — desde AWS Secrets Manager
ANTHROPIC_MODEL_PRIMARY=claude-sonnet-4-6
ANTHROPIC_MODEL_FALLBACK=claude-opus-4-7
ANTHROPIC_MAX_RETRIES=3               # BR-U03-31 (2s/4s/8s con backoff del SDK)
UMBRAL_FALLBACK_OPUS=0.75             # BR-U03-06: confianza < umbral → re-extrae con Opus

# ─── Pipeline de extracción ──────────────────────────────────────
MAX_PDFS_CONCURRENTES=3               # NFR-U03-02.1 (Tier 1 Anthropic; subir al crecer tier)
CHUNK_PAGINAS=20                      # BR-U03-13: chunking si excede contexto
FUZZY_RATIO_MIN=0.85                  # BR-U03-15 / Gate G3

# ─── Gates de calidad ────────────────────────────────────────────
UMBRAL_ALERTA_ESCALAMIENTO=0.30       # BR-U03-38 / Gate G1
UMBRAL_EMAIL_ESCALAMIENTO=5           # BR-U03-36: email al Analista si supera N escalamientos
```

**Validación al startup** (`pydantic-settings`): `ANTHROPIC_API_KEY` requerida; si falta, fail-fast con mensaje claro.

---

## TSD-U03-07: Testing Específico de U-03 (Q6.3 = A, PBT-09)

| Atributo | Decisión |
|---|---|
| **Framework PBT** | `hypothesis` 6.x (ya en stack U-02) — cumple **PBT-09** (shrinking + reproducibilidad por seed, PBT-08) |
| **Mock del LLM** | Doble de prueba de `AnthropicProvider` (respuestas tool use simuladas) — sin llamadas reales en tests |
| **Generadores PBT** | Dominio-específicos (PBT-07): `ItemCotizado` con valores monetarios `Decimal` realistas, fragmentos de texto, candidatos multi-PDF |
| **Cobertura** | 80% global; ≥ 90% MotorCruce y Gate G3 (NFR-U03-07) |

**Estructura de tests de U-03:**
```
backend/tests/
├── unit/
│   ├── test_gateway_llm/
│   │   ├── test_tool_schema_dinamico.py    # schema desde CatalogoVariable (sin cache)
│   │   ├── test_clasificar_y_extraer.py    # una llamada: clasifica + extrae
│   │   └── test_fallback_opus.py           # confianza < 0.75 → re-extrae con Opus
│   ├── test_motor_extraccion/
│   │   ├── test_fuzzy_validation.py        # PBT-U03-02: Gate G3 (rapidfuzz)
│   │   ├── test_consolidacion_multi_pdf.py # PBT-U03-03: prioridad × confianza
│   │   └── test_no_encontrado.py           # convención NO_ENCONTRADO
│   ├── test_motor_cruce/
│   │   ├── test_aritmetica_iva.py          # PBT-U03-01: IVA/total/cantidad (Decimal)
│   │   ├── test_cruce_precio_total.py      # anti-fraude precio_total vs Σ items
│   │   └── test_inconsistencias_internas.py# vigencia < entrega, etc.
│   └── test_servicio_pipeline/
│       ├── test_concurrencia_semaforo.py   # MAX_PDFS_CONCURRENTES
│       ├── test_idempotencia.py            # reanudación sin reprocesar
│       └── test_filtro_estado_sab.py       # CERRADA + ACEPTADA_PARA_ESTUDIO
└── integration/
    └── test_pipeline_u03_completo.py       # INGESTA_COMPLETADA → ANALISIS_COMPLETADO (mocks LLM)
```

---

## TSD-U03-08: Dependencias Nuevas (U-03)

| Paquete | Versión | Uso |
|---|---|---|
| `anthropic` | `0.40.x`+ | SDK oficial async — tool use, clasificación + extracción |
| `rapidfuzz` | `3.x` | Validación fuzzy Gate G3 (BR-U03-15) |

**Dependencias reutilizadas sin cambios** (U-01/U-02):
- `sqlalchemy[asyncio]` + `asyncpg` — lectura de `CatalogoVariable` y entidades U-03
- `hypothesis` — PBT (ya presente desde U-02)
- `pydantic-settings` — nuevas env vars
- `aiobotocore` — persistencia de `LlamadaLLM` JSON en S3 (auditoría CISO)

`decimal` y `asyncio` son stdlib — sin dependencia adicional.

---

## Resumen Ejecutivo de Decisiones Nuevas (U-03)

| Categoría | Tecnología | Versión |
|---|---|---|
| Cliente LLM | anthropic (AsyncAnthropic) | 0.40.x+ |
| Modelos | claude-sonnet-4-6 / claude-opus-4-7 | — |
| Validación fuzzy | rapidfuzz | 3.x |
| Aritmética monetaria | decimal.Decimal (stdlib) | — |
| Concurrencia | asyncio.Semaphore (3) | stdlib |
| Catálogo | Lectura directa BD (sin cache) | — |
| PBT framework | hypothesis (reutilizado) | 6.x |

---

## Dependencias Prohibidas (U-03)

| Librería | Razón |
|---|---|
| `openai` / SDKs de otros proveedores | MVP solo `AnthropicProvider`; la abstracción `LLMProvider` los permitiría a futuro, no ahora |
| `float` para montos | Errores de redondeo inaceptables en validación anti-fraude; usar `Decimal` |
| `thefuzz` / `fuzzywuzzy` | Más lentas; licencia GPL; usar `rapidfuzz` (MIT) |
| `requests` (sync al LLM) | Bloqueante; usar `AsyncAnthropic` |
| Cache en memoria del catálogo (TTL/lock) | Eliminado por NFR Q5.3 — lectura directa de BD |
| Celery / RQ / broker externo | Innecesario para piloto de 1 portafolio; usar asyncio nativo |
