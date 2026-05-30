# Plan de Infrastructure Design — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Infrastructure Design
> **Fecha**: 2026-05-29
> **Proveedor Cloud**: AWS (heredado de U-01/U-02)
> **Componentes**: C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline, C-11 GatewayLLM

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta (A/B/C o X para texto libre). Cuando termine, indíquelo y generaré los artefactos (`infrastructure-design.md` + `deployment-architecture.md`).

---

## Nota de Herencia

U-03 **hereda toda la infraestructura de U-01 y U-02** (App Runner, RDS PostgreSQL 16, S3, CloudWatch, ECR, Secrets Manager, Textract). Sus 4 componentes corren **in-process** en el mismo contenedor FastAPI/asyncio — no introducen nuevos servicios de cómputo.

**Adiciones nuevas de U-03** (lo único que diseña esta etapa):
- Integración saliente con **Anthropic API** (egress HTTPS + API key en Secrets Manager)
- Prefijo S3 nuevo para **auditoría de llamadas LLM** (`portafolios/{pid}/llm_calls/`)
- Alarmas CloudWatch para Gates de calidad (escalamiento, errores LLM)

**Evaluación de categorías de infraestructura:**

| Categoría | Estado en U-03 |
|---|---|
| Deployment Environment | Heredado (AWS, App Runner, dev local + prod) — sin cambios |
| Compute | Heredado App Runner — **Q1** (sizing) |
| Storage | RDS heredado; S3 nuevo prefijo `llm_calls/` — **Q2** (lifecycle) |
| Messaging | `asyncio.Queue` in-process (decidido en tech-stack); **sin broker** — N/A nueva infra |
| Networking | Egress a Anthropic API — **Q3** |
| Monitoring | CloudWatch heredado + alarmas U-03 — **Q4** |
| Shared | Comparte contenedor/event-loop con U-01/U-02 — **Q5** (límite de concurrencia compartida) |
| Secrets | Nuevo secret Anthropic — **Q6** |

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Functional Design + NFR Requirements aprobados
- [x] Plan Infra creado con preguntas
- [x] Respuestas recibidas y analizadas (6/6, 2026-05-29: Q1=A, Q2=A, Q3=A, Q4=B, Q5=C, Q6=A — sin inconsistencias)
- [x] Plan aprobado (2026-05-29)

### PART 2 — Generation
- [x] `infrastructure-design.md` generado
- [x] `deployment-architecture.md` generado

---

## Sección 1 — Cómputo (App Runner)

### Pregunta 1 — Sizing del contenedor para el pipeline U-03

U-02 ajustó a **2 vCPU / 4 GB** porque la sanitización (pypdf + regex) es CPU-intensiva. El pipeline de U-03 es distinto: está **dominado por I/O de red** (espera de respuestas del LLM), con poco CPU local (solo `rapidfuzz` y aritmética `Decimal`). ¿Qué sizing se usa?

A) Mantener **2 vCPU / 4 GB** heredado de U-02 — sin cambios; el contenedor es compartido y U-03 no agrega presión de CPU/memoria significativa
B) Reducir a 1 vCPU / 2 GB cuando solo corre U-03 — no aplica realmente porque el contenedor es único y compartido (descartar)
C) Aumentar a 4 GB+ por si el chunking de PDFs grandes (BR-U03-13) acumula texto en memoria
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 2 — Almacenamiento (S3 auditoría LLM)

### Pregunta 2 — Ciclo de vida del prefijo `llm_calls/` (auditoría CISO)

NFR-U03-05.2 definió **retención indefinida durante el piloto**. Para la política de lifecycle de S3 (los JSON de auditoría pueden crecer — hasta 2 por PDF):

A) Sin lifecycle automático en el piloto — todo permanece en S3 Standard; se define política post-piloto con el CISO (coherente con NFR-U03-05.2)
B) Transición a **S3 Glacier Instant Retrieval > 1 año** (igual que PDFs/OCR en U-02 §4) — balancea costo y disponibilidad de auditoría
C) Transición a Glacier > 90 días — más agresivo en costo
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 3 — Networking (egress a Anthropic API)

### Pregunta 3 — Topología de salida hacia `api.anthropic.com`

Anthropic API se consume vía HTTPS público (no requiere IP whitelist). Igual que la conexión al SAB (U-02 §7, internet público). ¿Se confirma?

A) Egress por **internet público HTTPS** desde App Runner (sin VPC connector ni NAT) — simple, suficiente para el piloto; TLS verificado
B) Egress vía **VPC Connector + NAT Gateway** para IP de salida estática — solo si una política corporativa lo exige (añade costo de NAT ~$32/mes)
C) Egress vía proxy corporativo — si la institución obliga a enrutar tráfico externo por un proxy
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

## Sección 4 — Monitoreo (CloudWatch)

### Pregunta 4 — Alarmas CloudWatch específicas de U-03

Coherente con la observabilidad mínima de NFR-U03-06 (tasa escalamiento + errores LLM). ¿Qué alarmas se crean?

A) Dos alarmas: (1) **Tasa de escalamiento > 30%** por portafolio (`ALERTA_DEGRADACION`, Gate G1) → email ADMIN+CISO; (2) **Errores LLM** (reintentos agotados ≥ 5 en 1h) → email equipo técnico
B) Solo alarma de errores LLM — la degradación de escalamiento se revisa manualmente
C) Sin alarmas nuevas en el piloto — se observa por logs/AuditTrail
X) Otro (describa después del tag [Answer]:)

[Answer]:B

---

## Sección 5 — Infraestructura Compartida (concurrencia en el contenedor)

### Pregunta 5 — Convivencia de pipelines U-02 (ingesta) y U-03 (extracción) en el mismo contenedor

Ambos pipelines corren como BackgroundTasks asyncio en el mismo App Runner. Con 1 portafolio a la vez (NFR), pero U-02 de un portafolio podría solaparse con U-03 de otro. ¿Cómo se gestiona?

A) **Procesamiento secuenciado por portafolio** (hereda "1 portafolio activo" de U-02 NFR-U02-02.1): U-03 de un portafolio arranca tras su `INGESTA_COMPLETADA`; en el piloto no se fuerza paralelismo entre portafolios → sin contención
B) Semáforos globales separados por tipo de tarea (uno para ingesta, otro para extracción) para acotar el uso total de CPU/memoria/conexiones
C) Sin gestión explícita — confiar en que la carga del piloto es baja
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

## Sección 6 — Secrets Manager

### Pregunta 6 — Organización del secret de Anthropic

¿Cómo se almacena la API key de Anthropic en AWS Secrets Manager? (patrón U-02: `assitant-buy/sab/keycloak`, `assitant-buy/sab/db`)

A) Nuevo secret dedicado **`assitant-buy/anthropic`** con `ANTHROPIC_API_KEY` (y opcionalmente nombres de modelo) — coherente con el namespacing `assitant-buy/*` ya cubierto por el IAM Role
B) Agregar `ANTHROPIC_API_KEY` a un secret existente compartido
C) Variable de entorno directa en App Runner (no recomendado — menos seguro)
X) Otro (describa después del tag [Answer]:)

[Answer]:A

---

> **Próximo paso**: Completa los `[Answer]:` y avísame. Generaré `infrastructure-design.md` + `deployment-architecture.md` de U-03. Esta es la **última etapa de diseño de U-03**; tras aprobarla, U-03 queda con todos sus diseños completos y el flujo continúa con el diseño de U-04 (según el modo de ejecución de diseños diferidos, Code Generation va en la segunda pasada).
