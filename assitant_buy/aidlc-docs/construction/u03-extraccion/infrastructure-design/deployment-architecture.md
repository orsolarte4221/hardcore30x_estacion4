# Deployment Architecture — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Proveedor Cloud**: AWS
> **Ambientes**: development (local) + production (AWS)
> **Fecha**: 2026-05-29
> **Scope MVP**: solo BIENES

---

## 1. Arquitectura de Producción (AWS) — U-01 + U-02 + U-03

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            INTERNET                                         │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ HTTPS / WSS
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AWS App Runner                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Load Balancer Integrado — TLS (ACM) — Auto-scaling 1–3 instancias  │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │                                          │
│  ┌───────────────────────────────▼─────────────────────────────────────┐    │
│  │  Container FastAPI (Python 3.12) — 2 vCPU / 4 GB (sin cambio)      │    │
│  │  Uvicorn — 4 workers                                                │    │
│  │                                                                     │    │
│  │  ── U-01 + U-02 Components (heredados) ──────────────────────────   │    │
│  │  • Auth, API, WS, RepositorioLocal, Ingesta, Sanitizador, Textract │    │
│  │                                                                     │    │
│  │  ── U-03 Components (NUEVOS, in-process) ────────────────────────   │    │
│  │  • C-09 ServicioPipeline   (BackgroundTask asyncio, semáforo=3)     │    │
│  │  • C-03 MotorExtraccion    (clasifica+extrae por PDF)              │    │
│  │  • C-11 GatewayLLM         (AsyncAnthropic → egress HTTPS)         │    │
│  │  • C-04 MotorCruce         (Decimal, anti-fraude, sin red)         │    │
│  └──────────┬──────────────────────┬─────────────────┬───────────────┘    │
│             │ VPC Connector         │ IAM Role        │ Egress público     │
└─────────────┼──────────────────────┼─────────────────┼────────────────────┘
              │                      │                 │ HTTPS 443
              ▼                      ▼                 ▼
┌─────────────────────┐  ┌──────────────────────┐  ┌──────────────────────────┐
│   AWS VPC Privada   │  │     AWS Services     │  │  Anthropic API (externo) │
│                     │  │                      │  │                          │
│  ┌───────────────┐  │  │ ┌──────────────────┐ │  │  api.anthropic.com:443   │
│  │ Amazon RDS    │  │  │ │ Amazon S3        │ │  │  ┌────────────────────┐  │
│  │ PostgreSQL 16 │  │  │ │ portafolios/{p}/ │ │  │  │ claude-sonnet-4-6  │  │
│  │ db.t4g.micro  │  │  │ │  pdfs/    (U-02) │ │  │  │  (primaria)        │  │
│  │ (BD propia)   │  │  │ │  ocr/     (U-02) │ │  │  ├────────────────────┤  │
│  │               │  │  │ │  llm_calls/(U-03)│ │  │  │ claude-opus-4-7    │  │
│  │ CatalogoVar.  │◀─┼──┼─┤  ← auditoría LLM │ │  │  │  (fallback <0.75)  │  │
│  │ (lee por PDF) │  │  │ │ SSE-S3 (AES-256) │ │  │  └────────────────────┘  │
│  └───────────────┘  │  │ └──────────────────┘ │  │   tool use (1 llamada:   │
│                     │  │                      │  │   clasifica + extrae)    │
│  Security Groups:   │  │ ┌──────────────────┐ │  └──────────────────────────┘
│  sg-rds←sg-apprunner│  │ │ Secrets Manager  │ │
│                     │  │ │ assitant-buy/sab/*│ │  ── SAB (externo, via U-02) ──
└─────────────────────┘  │ │ assitant-buy/    │ │  (U-03 NO accede al SAB:
                         │ │   anthropic ←NUEVO│ │   lee de nuestra BD — BR-U03-39)
                         │ └──────────────────┘ │
                         │                      │
                         │ ┌──────────────────┐ │
                         │ │ CloudWatch       │ │
                         │ │ Alarma: LLM      │ │
                         │ │  Errors (U-03)   │ │
                         │ │ (escalamiento →  │ │
                         │ │  AuditTrail/manual)│ │
                         │ └──────────────────┘ │
                         │ ┌──────────────────┐ │
                         │ │ Amazon ECR       │ │
                         │ └──────────────────┘ │
                         └──────────────────────┘
```

---

## 2. Arquitectura de Development (Local)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     docker-compose (local)                                 │
│                                                                            │
│  ┌────────────────────────┐    ┌──────────────────────────────────────┐    │
│  │  app (FastAPI)         │    │  db (PostgreSQL 16)                  │    │
│  │  Python 3.12           │───▶│  Port: 5432                          │    │
│  │  Port: 8000            │    │  CatalogoVariable (seed BIENES)      │    │
│  │  uvicorn --reload      │    └──────────────────────────────────────┘    │
│  │                        │                                                │
│  │  U-03 BackgroundTask:  │    ┌──────────────────────────────────────┐    │
│  │  • ServicioPipeline    │───▶│  LocalStack (mock AWS)               │    │
│  │  • MotorExtraccion     │    │  - S3 mock (llm_calls/ JSON)         │    │
│  │  • GatewayLLM ─────────┼─┐  └──────────────────────────────────────┘    │
│  │  • MotorCruce          │ │                                              │
│  └────────────────────────┘ │  ┌──────────────────────────────────────┐    │
│                             └─▶│  Anthropic API (real, vía API key)   │    │
│                                │  O mock del AnthropicProvider en tests│    │
│                                └──────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ HTTP/WS
┌────────────────────────────────────────────────────────────────────────────┐
│  Angular Frontend (dev server) — Port: 4200                                │
│  WS: eventos EXTRACCION_PROGRESO / CRUCE_PROGRESO / ANALISIS_COMPLETADO     │
└────────────────────────────────────────────────────────────────────────────┘
```

> En tests, `AnthropicProvider` se reemplaza por un doble de prueba (respuestas tool use simuladas) — **sin llamadas reales** ni costo. Para integración manual en dev se usa la API key real.

---

## 3. Flujo de Extracción + Cruce (BackgroundTask U-03)

```
U-02              C-09 Pipeline        C-03/C-11           Anthropic        AWS
  │                    │                   │                  │              │
  │─ INGESTA_COMPLETADA ▶│ (asyncio.Queue)  │                  │              │
  │                    │                   │                  │              │
  │                    │ filtra cotizaciones por estado SAB    │              │
  │                    │ (CERRADA + ACEPTADA_PARA_ESTUDIO)     │              │
  │                    │                   │                  │              │
  │                    │ semáforo(3) ─ por cada PDF LIMPIO ──▶ │              │
  │                    │                   │ lee CatalogoVariable (BD, s/cache)│
  │                    │                   │── clasificar_y_extraer ─▶│       │
  │                    │                   │   (Sonnet 4.6, tool use) │       │
  │                    │                   │◀─ tipo_doc + variables ──│       │
  │                    │                   │  [si conf<0.75: Opus 4.7]│       │
  │                    │                   │── fuzzy validation (rapidfuzz)   │
  │                    │                   │── persiste VariableExtraida ─────▶ RDS
  │                    │                   │── LlamadaLLM JSON ───────────────▶ S3 llm_calls/
  │◀─ WS EXTRACCION_PROGRESO ──────────────│                  │              │
  │                    │                   │                  │              │
  │                    │ [todos PDFs de la cotización listos]  │              │
  │                    │── C-04 MotorCruce ─▶│                 │              │
  │                    │   • precio_total vs Σ items (anti-fraude)            │
  │                    │   • aritmética IVA/total/cantidad (Decimal)          │
  │                    │   • inconsistencias internas                        │
  │                    │── persiste Discrepancia ─────────────────────────────▶ RDS
  │◀─ WS ANALISIS_COMPLETADO ──────────────│  {escalamientos} │              │
```

---

## 4. Mapping Componentes → Infraestructura (U-03)

| Componente | Módulo Python | Infraestructura | Protocolo |
|---|---|---|---|
| C-09 ServicioPipeline | `app/modules/extraccion/pipeline.py` | App Runner (BackgroundTask) | Internal async |
| C-03 MotorExtraccion | `app/modules/extraccion/motor_extraccion.py` | App Runner (in-process) | Internal |
| C-11 GatewayLLM | `app/modules/extraccion/gateway_llm.py` | Anthropic API (externo) | HTTPS/`anthropic` SDK |
| C-04 MotorCruce | `app/modules/extraccion/motor_cruce.py` | App Runner (in-process) | Internal |
| Catálogo (lectura) | `app/modules/extraccion/repositorio_catalogo.py` | Amazon RDS (BD propia) | asyncpg |
| Auditoría LLM | `app/core/storage.py` (extendido) | Amazon S3 `portafolios/*/llm_calls/` | HTTPS |
| VariableExtraida / Discrepancia / LlamadaLLM | `app/modules/extraccion/repositorio.py` | Amazon RDS PostgreSQL | asyncpg |

---

## 5. Puertos y Endpoints de U-03

U-03 no expone endpoints REST nuevos para usuario final — se dispara por evento interno (`INGESTA_COMPLETADA`, BR-U03-28). Expone eventos WebSocket sobre la conexión heredada de U-01.

| Ambiente | Protocolo | Canal | Eventos |
|---|---|---|---|
| Production | WSS | `wss://{dominio}/ws/{portafolio_id}` | `EXTRACCION_INICIADA`, `DOCUMENTO_CLASIFICADO`, `EXTRACCION_PROGRESO`, `CRUCE_INICIADO`, `CRUCE_PROGRESO`, `ANALISIS_COMPLETADO` |
| Development | WS | `ws://localhost:8000/ws/{portafolio_id}` | Igual |

> Contrato interno expuesto para U-04 (BR-U03-34): `C-09.extraer_pdf(documento_pdf_id, cotizacion_id)` invocable por PDF individual (reingesta tras aclaraciones).

---

## 6. Consideraciones de Seguridad de la Infraestructura (U-03)

| Control | Implementación | Alcance |
|---|---|---|
| API key Anthropic | AWS Secrets Manager `assitant-buy/anthropic` (nunca en env plana ni logs) | App Runner → Anthropic |
| TLS con Anthropic | SDK `anthropic` con verificación TLS (sin bypass) | Egress HTTPS |
| Anti-EchoLeak | Una sesión LLM por documento, sin history (BR-U03-14) — por construcción | C-11 |
| Auditoría LLM en S3 | Bucket privado, SSE-S3 (AES-256), Block public access (heredado U-02) | S3 `llm_calls/` |
| Texto/montos en logs | Prohibido — solo IDs y métricas; contenido solo en auditoría cifrada | CloudWatch |
| Aislamiento del SAB | U-03 NO accede a la BD/API del SAB (lee de nuestra BD — BR-U03-39) | Separación U-02/U-03 |
| Egress sin IP fija | Internet público; Anthropic no requiere whitelist (Q3 = A) | Networking |

---

*Generado por AIDLC Construction Workflow — 2026-05-29*
