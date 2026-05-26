# Deployment Architecture — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Proveedor Cloud**: AWS  
> **Ambientes**: development (local) + production (AWS)  
> **Fecha**: 2026-05-26  
> **Estado**: APROBADO — 2026-05-26T22:39:00Z

---

## 1. Arquitectura de Producción (AWS) — U-01 + U-02

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
│  │  Container FastAPI (Python 3.12) — 2 vCPU / 4 GB                   │    │
│  │  Uvicorn — 4 workers                                                │    │
│  │                                                                     │    │
│  │  ── U-01 Components ──────────────────────────────────────────────  │    │
│  │  • C-08 AutenticadorOAuth    (OAuth2 PKCE + JWT HS256)              │    │
│  │  • C-10 APIRest              (FastAPI, /api/v1/, RFC 7807)          │    │
│  │  • C-13 RepositorioLocal     (SQLAlchemy async + asyncpg)           │    │
│  │  • C-14 GestorWebSocket      (FastAPI WebSocket, /ws/)              │    │
│  │                                                                     │    │
│  │  ── U-02 Components ──────────────────────────────────────────────  │    │
│  │  • C-01 IngestaConector      (BackgroundTask, descarga PDFs SAB)    │    │
│  │  • C-02 SanitizadorZeroTrust (RT0-RT4 + Spotlighting + OCR)        │    │
│  │  • C-12 RepositorioSAB       (asyncpg read-only → BD SAB)          │    │
│  │  • C-16 GatewayDocumentAI    (Amazon Textract via aiobotocore)      │    │
│  │  • GestorTokenSAB            (Keycloak ROPC token manager)         │    │
│  └──────────┬──────────────────────────────┬───────────────────────────┘    │
│             │ VPC Connector                 │ IAM Role                      │
└─────────────┼──────────────────────────────┼───────────────────────────────┘
              │                              │
              ▼                              ▼
┌─────────────────────────┐    ┌─────────────────────────────────────────────┐
│    AWS VPC Privada      │    │              AWS Services                   │
│                         │    │                                             │
│  ┌─────────────────┐    │    │  ┌──────────────────────────────────────┐   │
│  │  Amazon RDS     │    │    │  │  Amazon S3  (bucket único)           │   │
│  │  PostgreSQL 16  │    │    │  │                                      │   │
│  │  db.t4g.micro   │    │    │  │  portafolios/{pid}/pdfs/{pdf_id}.pdf │   │
│  │  Port: 5432     │    │    │  │  portafolios/{pid}/ocr/{pdf_id}.txt  │   │
│  │  (BD propia)    │    │    │  │                                      │   │
│  └─────────────────┘    │    │  │  SSE-S3 (AES-256) + Block public    │   │
│                         │    │  └──────────────────────────────────────┘   │
│  Security Groups:       │    │                                             │
│  sg-rds ← sg-apprunner  │    │  ┌──────────────────────────────────────┐   │
│                         │    │  │  Amazon Textract                     │   │
└─────────────────────────┘    │  │  DetectDocumentText (sync, ≤5 págs) │   │
                               │  │  StartDocumentTextDetection (async)  │   │
                               │  └──────────────────────────────────────┘   │
                               │                                             │
                               │  ┌──────────────────────────────────────┐   │
                               │  │  AWS Secrets Manager                 │   │
                               │  │  assitant-buy/sab/keycloak           │   │
                               │  │  assitant-buy/sab/db                 │   │
                               │  │  (+ secrets de U-01)                 │   │
                               │  └──────────────────────────────────────┘   │
                               │                                             │
                               │  ┌──────────────────────────────────────┐   │
                               │  │  CloudWatch Logs + Alarms            │   │
                               │  │  Alarma: OCR failures                │   │
                               │  │  Alarma: SAB auth failures           │   │
                               │  │  Alarma: Ingesta stalled             │   │
                               │  └──────────────────────────────────────┘   │
                               │                                             │
                               │  ┌──────────────────────────────────────┐   │
                               │  │  Amazon ECR — Docker images          │   │
                               │  └──────────────────────────────────────┘   │
                               └─────────────────────────────────────────────┘

── Sistemas Externos (internet) ──────────────────────────────────────────────

  ┌────────────────────────────────────────────────────────────────────────┐
  │  SAB — Sistema de Administración de Bienes (Universidad del Cauca)    │
  │                                                                        │
  │  ┌──────────────────────────────────────┐                              │
  │  │  Keycloak (OAuth2 OIDC)              │◀── POST /token (ROPC)        │
  │  │  proyunicauca.docxflow.com:8443      │    grant_type=password       │
  │  └──────────────────────────────────────┘                              │
  │                                                                        │
  │  ┌──────────────────────────────────────┐                              │
  │  │  SAB API REST                        │◀── GET /documentos/descargar │
  │  │  backend.unicauca.edu.co             │    Authorization: Bearer {t} │
  │  └──────────────────────────────────────┘                              │
  │                                                                        │
  │  ┌──────────────────────────────────────┐                              │
  │  │  BD PostgreSQL SAB (read-only)       │◀── asyncpg read-only         │
  │  │  Acceso via credenciales en RDS proxy│    SELECT only               │
  │  └──────────────────────────────────────┘                              │
  └────────────────────────────────────────────────────────────────────────┘
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
│  │  Port: 8000            │    │  Volume: postgres_data               │    │
│  │  uvicorn --reload      │    └──────────────────────────────────────┘    │
│  │                        │                                                │
│  │  U-02 BackgroundTask:  │    ┌──────────────────────────────────────┐    │
│  │  • IngestaConector     │───▶│  LocalStack (mock AWS)               │    │
│  │  • SanitizadorZeroTrust│    │  Port: 4566                          │    │
│  │  • GatewayTextract     │    │  - S3 mock (PDFs + OCR .txt)         │    │
│  └────────────────────────┘    │  - Textract mock (moto en tests)     │    │
│                                └──────────────────────────────────────┘    │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SAB Dev: conecta al entorno QA del SAB real (HTTPS externo)        │   │
│  │  O usa respuestas mock de respx en tests (recomendado en local)     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ HTTP (no HTTPS en local)
┌────────────────────────────────────────────────────────────────────────────┐
│  Angular Frontend (dev server) — Port: 4200                                │
│  WS: ws://localhost:8000/ws/{portafolio_id}                                │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Flujo de Ingesta de PDF (BackgroundTask)

```
Analista                FastAPI             SAB (externo)        AWS
    │                      │                     │                 │
    │── POST /analizar ────▶│                     │                 │
    │◀── 202 Accepted ──────│                     │                 │
    │   (portafolio INGESTA)│                     │                 │
    │                      │── BackgroundTask ──────────────────────
    │                      │                     │                 │
    │                      │── obtener_token() ─▶│                 │
    │                      │                     │ Keycloak ROPC   │
    │                      │◀─ access_token ──────│                 │
    │                      │                     │                 │
    │                      │ [por cada PDF]       │                 │
    │                      │── GET /descargar ───▶│                 │
    │                      │◀─ pdf_bytes ──────────│                 │
    │                      │                     │                 │
    │                      │── sanitizar(pdf_bytes)                │
    │                      │   RT4 → RT1 → OCR? → RT2 → RT3 → Spotlighting
    │                      │                     │                 │
    │                      │── [si es_escaneado] Textract.detect──▶│
    │                      │◀─ texto_ocr ──────────────────────────│
    │                      │── Upload .txt OCR ─────────────────────▶ S3
    │                      │── Upload .pdf ──────────────────────────▶ S3
    │                      │                     │                 │
    │◀── WS: PROGRESO ──────│  (n/total PDFs)     │                 │
    │                      │                     │                 │
    │  [fin de todos PDFs]  │                     │                 │
    │                      │── Update Portafolio: LISTO_PARA_EXTRACCION
    │◀── WS: INGESTA_COMPLETADA                   │                 │
```

---

## 4. Flujo de Detección OCR (PDF escaneado)

```
SanitizadorZeroTrust
    │
    ├─ RT4: pypdf.open(pdf_bytes) → válido → OK
    │
    ├─ pypdf.extract_text() → "" (sin capa de texto)
    │         es_escaneado = TRUE
    │
    ▼
GatewayDocumentAI (C-16) → Amazon Textract
    │
    ├─ n_paginas ≤ 5:
    │       aiobotocore.textract.detect_document_text(
    │           Document={'Bytes': pdf_bytes}
    │       )
    │       → respuesta inmediata con Blocks de texto
    │
    └─ n_paginas > 5:
            aiobotocore.textract.start_document_text_detection(
                DocumentLocation={'S3Object': {'Bucket': BUCKET, 'Name': pdf_s3_key}}
            ) → job_id
            │
            Polling (cada 2s, max 120s):
            aiobotocore.textract.get_document_text_detection(JobId=job_id)
            │
            ├─ Status: IN_PROGRESS → esperar
            ├─ Status: FAILED → ERROR "ocr_fallido"
            └─ Status: SUCCEEDED → texto_ocr extraído
    │
    ▼
texto_ocr (string) → Upload S3: portafolios/{pid}/ocr/{pdf_id}.txt
DocumentoPDF.texto_ocr_s3_key = "portafolios/{pid}/ocr/{pdf_id}.txt"
    │
    ▼
Continúa a RT1 → RT2(texto_ocr) → RT3(texto_ocr) → Spotlighting
```

---

## 5. Mapping Componentes → Infraestructura (U-02)

| Componente | Módulo Python | Infraestructura | Protocolo |
|---|---|---|---|
| C-01 IngestaConector | `app/modules/ingesta/conector.py` | App Runner (BackgroundTask) | Internal async |
| C-02 SanitizadorZeroTrust | `app/modules/ingesta/sanitizador.py` | App Runner (in-process) | Internal |
| C-12 RepositorioSAB (BD) | `app/modules/ingesta/repositorio_sab.py` | BD PostgreSQL externa del SAB | asyncpg/TLS |
| C-12 GestorTokenSAB | `app/modules/ingesta/gestor_token_sab.py` | Keycloak SAB externo | HTTPS/OAuth2 |
| C-16 GatewayDocumentAI | `app/modules/ingesta/gateway_textract.py` | Amazon Textract | aiobotocore |
| Almacenamiento PDF | `app/core/storage.py` (extendido) | Amazon S3 `portafolios/*/pdfs/` | HTTPS |
| Almacenamiento OCR | `app/core/storage.py` (extendido) | Amazon S3 `portafolios/*/ocr/` | HTTPS |
| AuditTrail ingesta | `app/modules/audit/` | Amazon RDS PostgreSQL | asyncpg |

---

## 6. Puertos y Endpoints de U-02

| Ambiente | Protocolo | Endpoint | Descripción |
|---|---|---|---|
| Production | HTTPS | `POST /api/v1/procesos/{id}/analisis` | Inicia análisis (lanza BackgroundTask) |
| Production | WSS | `wss://{dominio}/ws/{portafolio_id}` | Progreso de ingesta en tiempo real |
| Development | HTTP | `POST http://localhost:8000/api/v1/procesos/{id}/analisis` | Igual |
| Development | WS | `ws://localhost:8000/ws/{portafolio_id}` | Igual |

---

## 7. Consideraciones de Seguridad de la Infraestructura (U-02)

| Control | Implementación | Alcance |
|---|---|---|
| Credenciales SAB | AWS Secrets Manager (nunca en env vars planas) | Keycloak + BD SAB |
| TLS con SAB externo | httpx verify=True, sin bypass SSL | App Runner → SAB |
| Textract IAM | Solo permisos `detect` / `start` / `get`; sin permisos de admin | IAM Role |
| S3 companion files OCR | Mismo bucket privado; Block public access | S3 |
| Texto OCR en logs | Prohibido loggear texto extraído (solo s3_key y n_chars) | CloudWatch |
| Credentials no en logs | Validado por PBT-U02-03 (hypothesis) | Tests + Code |
| PDF max size en Textract | RT4 valida ANTES de llamar a Textract (no se procesa si > 50 MB) | SanitizadorZeroTrust |

---

*Generado por AIDLC Construction Workflow — 2026-05-26*
