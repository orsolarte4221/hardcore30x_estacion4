# Plan de NFR Requirements — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — NFR Requirements
> **Fecha**: 2026-05-29
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM
> **Scope MVP**: solo BIENES

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta (elija A/B/C o use X para respuesta libre). Cuando termine todas, indíquelo y el workflow generará los 2 artefactos NFR de U-03 (`nfr-requirements.md` + `tech-stack-decisions.md`).

---

## Nota de Herencia

U-03 **hereda los NFRs de U-01 (Fundación) y U-02 (Ingesta)**. Este plan se enfoca únicamente en los NFRs **nuevos o específicos** del pipeline de extracción LLM, cruce determinista y orquestación:

- Performance y **costo** de las llamadas LLM (Sonnet 4.6 + fallback Opus 4.7)
- Concurrencia del pipeline por documento (semáforo)
- Gates de calidad: G3 (anti-alucinación) y G1 (tasa de escalamiento)
- Resiliencia ante fallos del proveedor LLM
- Seguridad: anti-EchoLeak y auditoría de llamadas LLM para el CISO
- Tech stack: SDK LLM, librería fuzzy, framework PBT (PBT-09)

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Functional Design aprobado (domain-entities v2, business-rules v2, business-logic-model v2, cross-unit-deltas)
- [x] Plan NFR creado con preguntas
- [x] Respuestas recibidas y analizadas (16/16, 2026-05-29)
- [x] Reconciliación de inconsistencias (Q5.3=C → delta al FD; Q2.2 ajustado a concurrencia 3; Q1.2/Q7.1 conciliadas)
- [ ] Plan aprobado (esperando aprobación final del usuario)

### PART 2 — Generation
- [x] `nfr-requirements.md` generado
- [x] `tech-stack-decisions.md` generado

---

## Sección 1 — Performance y Costo LLM

### Pregunta 1.1 — Latencia objetivo de extracción por PDF

Cada PDF dispara 1 llamada LLM primaria (Sonnet 4.6) y, si hay variables con `confianza < 0.75`, una 2ª llamada (Opus 4.7, BR-U03-06). ¿Qué SLA de latencia se define por documento?

A) < 30 s p95 por PDF (solo llamada primaria); < 60 s p95 si dispara fallback Opus — adecuado para piloto
B) < 15 s p95 por PDF — exigente, requiere prompts optimizados y límite estricto de tamaño de texto
C) Sin SLA estricto por PDF — el pipeline es asíncrono en background y el usuario sigue el progreso por WebSocket
X) Otro (describa después del tag [Answer]:)

[Answer]:C

---

### Pregunta 1.2 — Control de costo de tokens LLM

Las llamadas a Claude tienen costo por token. ¿Qué política de control de costo se aplica en el piloto?

A) Registrar tokens y costo estimado por `LlamadaLLM` (ya persistido en BD/S3), con métrica agregada por portafolio — sin tope automático
B) Igual que A + **tope configurable** `MAX_COSTO_USD_PORTAFOLIO`; al superarlo el pipeline pausa y alerta al Admin
C) Sin seguimiento de costo en el piloto — se mide solo en producción
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.3 — Manejo de rate limits / throttling del proveedor LLM

Si Anthropic responde 429 (rate limit) o 529 (overloaded), ¿cómo se comporta el sistema? (complementa el retry de BR-U03-31: 2s/4s/8s)

A) Retry con backoff exponencial (BR-U03-31) tratando 429/529 como error transitorio; respeta header `retry-after` si está presente
B) Igual que A + **limitador de tasa global** (token bucket) para no exceder el RPM/TPM contratado, reduciendo 429 proactivamente
C) Solo retry simple; si persiste, marcar variables con `ERROR_EXTRACCION` (BR-U03-31)
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 2 — Concurrencia y Throughput del Pipeline

### Pregunta 2.1 — Nivel de concurrencia por documento

BR-U03-30 define un semáforo `MAX_PDFS_CONCURRENTES` (default 5). ¿Se confirma este valor para el piloto?

A) Sí, 5 PDFs concurrentes (configurable por env var) — balance entre throughput y presión sobre rate limits del LLM
B) Más conservador: 3 concurrentes — minimiza 429 y simplifica depuración en piloto
C) Más agresivo: 8–10 concurrentes — maximiza throughput, requiere validar límites del plan de Anthropic
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

### Pregunta 2.2 — SLA de tiempo total de análisis por portafolio

Para un portafolio del piloto (referencia U-02: ≤ 50 cotizaciones, ≤ 200 PDFs LIMPIOS), ¿cuál es el tiempo objetivo de la fase de extracción + cruce de U-03?

A) ≤ 30 minutos para un portafolio de 200 PDFs (con concurrencia 5) — objetivo de piloto
B) ≤ 15 minutos — exigente, asume buena latencia LLM y concurrencia alta
C) Sin SLA total — best-effort; el valor está en la calidad, no en la velocidad
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 3 — Calidad: Gates G3 (anti-alucinación) y G1 (escalamiento)

### Pregunta 3.1 — Meta de tasa de alucinación (Gate G3)

BR-U03-15 valida con fuzzy (`rapidfuzz`, ratio ≥ 0.85) que el fragmento citado por el LLM exista en el PDF. ¿Cuál es la meta de tasa de alucinación y el umbral fuzzy?

A) Tasa de alucinación < 1% con ratio fuzzy ≥ 0.85 (valores del Functional Design) — confirmar
B) Más estricto: ratio fuzzy ≥ 0.90 (reduce falsos positivos del LLM a costa de más escalamientos HITL)
C) Más permisivo: ratio fuzzy ≥ 0.80 (menos escalamientos, mayor riesgo de aceptar citas imprecisas)
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 3.2 — Umbral de alerta de tasa de escalamiento (Gate G1)

BR-U03-38 emite `ALERTA_DEGRADACION` si la tasa de escalamiento de un portafolio supera `UMBRAL_ALERTA_ESCALAMIENTO` (default 30%). ¿Se confirma?

A) Sí, 30% (configurable) — por encima de eso se asume degradación del modelo o PDFs de baja calidad
B) Más estricto: 20% — alerta temprana de calidad
C) Más laxo: 40% — esperamos muchos escalamientos legítimos en el piloto inicial
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 4 — Resiliencia del Pipeline

### Pregunta 4.1 — Política de retry LLM

BR-U03-31 define 3 reintentos con backoff 2s/4s/8s y, si todos fallan, `VariableExtraida` con `ERROR_EXTRACCION`. ¿Se confirma?

A) Sí, tal cual (3 reintentos 2/4/8s → ERROR_EXTRACCION + `requiere_revision=TRUE`)
B) Aumentar a 5 reintentos para mayor robustez frente a 529 overloaded
C) Confirmar reintentos pero **además** degradar el documento completo a estado de reintento manual (no marcar todas las variables como ERROR)
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 4.2 — Fallo sostenido del proveedor LLM (circuit breaker)

Si el proveedor LLM está caído o devuelve errores de forma sostenida durante un portafolio, ¿qué comportamiento global se espera?

A) Circuit breaker: tras N fallos consecutivos (default 10) en una ventana corta, pausar el pipeline del portafolio y alertar al Admin; reanudable (idempotencia BR-U03-33)
B) Sin circuit breaker — cada PDF agota sus reintentos y queda en ERROR; el Analista reintenta el portafolio manualmente
C) Circuit breaker + failover automático a proveedor alterno vía interfaz `LLMProvider` (BR-U03-07) — fuera de MVP (solo AnthropicProvider)
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

## Sección 5 — Seguridad específica de U-03

### Pregunta 5.1 — Retención de la auditoría de llamadas LLM (CISO)

BR-U03-37 persiste cada llamada LLM completa en S3 (`portafolios/{pid}/llm_calls/{call_id}.json`) + registro `LlamadaLLM`. ¿Cuánto tiempo se retiene esta auditoría?

A) Retención indefinida durante el piloto (igual que PDFs en U-02); política post-piloto según CISO
B) Retención 1 año + archivado — alineado con la política de AuditTrail de U-01
C) Retención 90 días — suficiente para auditorías del piloto
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 5.2 — Contenido de los payloads LLM persistidos

El JSON de `LlamadaLLM` en S3 incluye el texto del PDF (system prompt + user message) y la respuesta. ¿Hay restricciones sobre qué se almacena?

A) Almacenar payload completo (texto del PDF + respuesta) — necesario para reproducibilidad y auditoría CISO; el bucket es privado y cifrado (hereda AES-256 + IAM de U-02)
B) Almacenar payload completo pero con **enmascaramiento** de identificadores de proveedor (NIT/razón social) en el JSON de auditoría
C) Almacenar solo metadata (tokens, modelo, latencia, hash del prompt) sin el texto — minimiza exposición, sacrifica reproducibilidad exacta
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 5.3 — Concurrencia del cache del catálogo (C-11)

BR-U03-19 cachea `CatalogoVariable` en memoria con TTL 300s. Con concurrencia (varios PDFs a la vez), ¿qué garantía se requiere?

A) Cache thread-safe / async-safe con refresco único (lock) para evitar lecturas duplicadas a BD; lecturas concurrentes ven una vista consistente del catálogo durante cada ciclo de 300s
B) Cache simple sin lock — aceptable que un cambio Admin tarde hasta 300s y que dos refrescos simultáneos golpeen BD ocasionalmente
C) Sin cache en piloto — leer catálogo de BD en cada PDF (más simple, más carga a BD)
X) Otro (describa después del tag [Answer]:)

[Answer]:C

---

## Sección 6 — Tech Stack específico de U-03

### Pregunta 6.1 — Cliente / SDK del LLM

¿Qué cliente se usa para invocar Claude desde C-11 GatewayLLM?

A) SDK oficial `anthropic` (async) con tool use nativo — alineado con AnthropicProvider (BR-U03-07)
B) SDK oficial `anthropic` (sync) envuelto en threadpool — más simple, sin async en la capa LLM
C) Cliente HTTP propio sobre la API REST de Anthropic — máximo control, más código a mantener
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 6.2 — Librería de validación fuzzy (Gate G3)

BR-U03-15 menciona `rapidfuzz` para validar la cita del LLM contra el texto del PDF. ¿Se confirma?

A) `rapidfuzz` (rápido, C++ backend, ratio configurable) — confirmar
B) `thefuzz`/`fuzzywuzzy` — más conocido pero más lento
C) Validación de substring exacto + normalización, sin librería fuzzy — más estricto y predecible
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

### Pregunta 6.3 — Framework de Property-Based Testing (PBT-09)

La extensión PBT (parcial: PBT-02, 03, 07, 08, 09) exige seleccionar framework PBT en las decisiones de tech stack. Para Python:

A) `hypothesis` — estándar de facto en Python, soporta shrinking y reproducibilidad por seed (cumple PBT-08)
B) Otro framework PBT de Python
C) (No aplica — reconsiderar habilitación de PBT)
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 7 — Observabilidad y Tests

### Pregunta 7.1 — Métricas de observabilidad del pipeline LLM

¿Qué métricas se exponen para el pipeline de U-03 en el piloto? (hereda enfoque "logs + AuditTrail, sin Prometheus" de U-02)

A) Derivadas de logs + `LlamadaLLM` + AuditTrail: tokens/costo por portafolio, latencia LLM p50/p95, tasa de fallback Opus, tasa de escalamiento, tasa de alucinación detectada
B) Subconjunto mínimo: solo tasa de escalamiento y errores LLM
C) Observabilidad completa con Prometheus/Grafana — fuera del alcance del piloto
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

### Pregunta 7.2 — Cobertura mínima de tests para U-03

¿Qué cobertura se exige antes de considerar U-03 "lista para build"? (U-02 usó 80% global con módulos de seguridad 90–95%)

A) 80% global; ≥ 90% en MotorCruce (lógica determinista aritmética/anti-fraude) y en validación fuzzy Gate G3; PBTs obligatorios en aritmética IVA/total y en validación fuzzy
B) 70% global — más laxo para acelerar el piloto
C) 85% global con 95% en MotorCruce y Gate G3 — alta exigencia dada la criticidad anti-fraude
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 8 — Trazabilidad con `cross-unit-deltas.md` §4 (TBDs Operacionales)

Los TBDs operacionales del Functional Design de U-03 se mapean a esta etapa NFR así:

| TBD (cross-unit-deltas §4) | Scope | Dónde se resuelve |
|---|---|---|
| §4.5 Concurrencia cache C-11 (`asyncio.Lock`) | **NFR U-03** | Q5.3 (este plan) → `tech-stack-decisions.md` |
| §4.6 Retención bucket S3 `llm_calls/` (CISO) | **NFR U-03** | Q5.1 (este plan) |
| §4.6 Cuota Anthropic API / capacity planning pico | **NFR U-03** | Q8.1 (este plan) |
| §4.1 Acceso BD SAB (read replica, usuario read-only, VPN) | NFR **U-02** | Se cierra al re-abrir U-02 (U-03 lee de *nuestra* BD — BR-U03-39) |
| §4.2 Seguridad endpoint `POST /portafolios/from-sab` | NFR/Infra **U-02** | El endpoint vive en U-02 |
| §4.3 Sincronización estado SAB ↔ nuestra BD | Diseño **U-02** | Rec. tentativa: snapshot inmutable en MVP |
| §4.4 `vista_cuadro_comparativo` SAB | Diseño **U-05** | Se evalúa al diseñar U-05 |

### Pregunta 8.1 — Capacity planning de la cuota Anthropic API (§4.6)

Considerando que en producción pueden correr varios portafolios concurrentes, cada uno con N PDFs (y posibles llamadas de fallback Opus), ¿qué política de capacidad se asume?

A) Dimensionar para el piloto con **1 portafolio activo a la vez** (hereda límite de U-02 NFR-U02-02.1) y concurrencia interna de 5 PDFs; documentar el TPM/RPM requerido y dejar el escalado multi-portafolio como TBD de producción
B) Dimensionar desde ya para **5 portafolios concurrentes** (target producción U-02): requiere validar/ampliar el tier de Anthropic y aplicar el limitador de tasa global (Q1.3-B)
C) Sin capacity planning explícito en el piloto — se observa el consumo real y se ajusta reactivamente
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

> **Próximo paso**: Una vez completadas todas las respuestas, indíquelo. Analizaré ambigüedades (si las hay) y generaré los 2 artefactos de NFR Requirements de U-03. Tras tu aprobación, la siguiente etapa es **Infrastructure Design** de U-03 (igual que en U-01 y U-02, la etapa opcional *NFR Design* no se usa en este proyecto).
