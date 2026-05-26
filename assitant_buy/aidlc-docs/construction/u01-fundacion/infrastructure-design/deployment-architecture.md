# Deployment Architecture — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Proveedor Cloud**: AWS  
> **Ambientes**: development (local) + production (AWS)  
> **Fecha**: 2026-05-26  
> **Estado**: GENERADO — Pendiente aprobación

---

## 1. Arquitectura de Producción (AWS)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
└────────────────────────┬────────────────────────────────────────────┘
                         │ HTTPS / WSS
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AWS App Runner                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Load Balancer Integrado (App Runner managed)                 │  │
│  │  • TLS termination (ACM certificate)                          │  │
│  │  • Health checks → GET /health                                │  │
│  │  • Auto-scaling: 1–3 instancias                               │  │
│  └───────────────────────┬───────────────────────────────────────┘  │
│                          │                                          │
│  ┌───────────────────────▼───────────────────────────────────────┐  │
│  │  Container FastAPI (Python 3.12)                              │  │
│  │  Uvicorn — 4 workers                                          │  │
│  │  • C-08 AutenticadorOAuth  (OAuth2 PKCE + JWT HS256)          │  │
│  │  • C-10 APIRest            (FastAPI, /api/v1/, RFC 7807)      │  │
│  │  • C-13 RepositorioLocal   (SQLAlchemy 2.x async + asyncpg)   │  │
│  │  • C-14 GestorWebSocket    (FastAPI WebSocket, /ws/)          │  │
│  │  • Pipeline IA             (asyncio BackgroundTasks)           │  │
│  └──────┬──────────────────────────┬────────────────────────────┘  │
│         │ VPC Connector             │ IAM Role                     │
└─────────┼──────────────────────────┼──────────────────────────────┘
          │                          │
          ▼                          ▼
┌─────────────────────┐  ┌─────────────────────────────────────────┐
│    AWS VPC Privada  │  │         AWS Services                    │
│                     │  │                                         │
│  ┌───────────────┐  │  │  ┌─────────────────┐                   │
│  │ Amazon RDS    │  │  │  │   Amazon S3     │                   │
│  │ PostgreSQL 16 │  │  │  │ (PDFs storage)  │                   │
│  │ db.t4g.micro  │  │  │  │ Presigned URLs  │                   │
│  │ Port: 5432    │  │  │  │ SSE-S3 encrypt  │                   │
│  └───────────────┘  │  │  └─────────────────┘                   │
│                     │  │                                         │
│  Security Groups:   │  │  ┌─────────────────┐                   │
│  sg-rds ← sg-apprunner  │  │  CloudWatch Logs│                   │
│                     │  │  │ /aws/apprunner/ │                   │
└─────────────────────┘  │  │ assitant-buy/.. │                   │
                         │  └─────────────────┘                   │
                         │                                         │
                         │  ┌─────────────────┐                   │
                         │  │  Secrets Manager│                   │
                         │  │ DB, JWT, OAuth  │                   │
                         │  └─────────────────┘                   │
                         │                                         │
                         │  ┌─────────────────┐                   │
                         │  │  Amazon ECR     │                   │
                         │  │ Docker images   │                   │
                         │  └─────────────────┘                   │
                         └─────────────────────────────────────────┘
```

---

## 2. Arquitectura de Development (Local)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   docker-compose (local)                            │
│                                                                     │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐   │
│  │   app (FastAPI)      │    │    db (PostgreSQL 16)            │   │
│  │   Python 3.12        │───▶│    Port: 5432                    │   │
│  │   Port: 8000         │    │    Volume: postgres_data         │   │
│  │   uvicorn --reload   │    └──────────────────────────────────┘   │
│  └──────────────────────┘                                           │
│           │                                                         │
│           ▼                                                         │
│  ┌──────────────────────┐                                           │
│  │  LocalStack (opt)    │  ← Mock de S3 para dev sin AWS real       │
│  │  Port: 4566          │                                           │
│  └──────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼ HTTP (no HTTPS en local)
┌─────────────────────────────────────────────────────────────────────┐
│  Angular Frontend (dev server)                                      │
│  Port: 4200                                                         │
│  WS: ws://localhost:8000/ws/{portafolio_id}                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Pipeline CI/CD — GitHub Actions

```
┌──────────────────────────────────────────────────────────────────┐
│  GitHub (repositorio)                                            │
│                                                                  │
│  Push a main / PR merge                                          │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  GitHub Actions — CI/CD Pipeline                        │    │
│  │                                                         │    │
│  │  Job 1: test                                            │    │
│  │    ├── checkout                                         │    │
│  │    ├── setup Python 3.12                                │    │
│  │    ├── pip install -r requirements.txt                  │    │
│  │    ├── pytest --cov=app --cov-fail-under=80             │    │
│  │    └── ruff + mypy                                      │    │
│  │                                                         │    │
│  │  Job 2: build-and-deploy (needs: test)                  │    │
│  │    ├── configure AWS credentials                        │    │
│  │    ├── login to ECR                                     │    │
│  │    ├── docker build + tag                               │    │
│  │    ├── docker push → ECR                                │    │
│  │    └── aws apprunner start-deployment                   │    │
│  └──────────────┬──────────────────────────────────────────┘    │
└─────────────────┼────────────────────────────────────────────────┘
                  │
         ┌────────▼────────┐         ┌──────────────────┐
         │  Amazon ECR     │────────▶│  AWS App Runner  │
         │  (Docker imgs)  │  pull   │  (auto-deploy)   │
         └─────────────────┘         └──────────────────┘
```

---

## 4. Flujo de Autenticación (OAuth2 PKCE)

```
Usuario                 Angular              FastAPI               Google
  │                       │                    │                    │
  │── Login click ────────▶│                    │                    │
  │                       │── GET /auth/login ─▶│                    │
  │                       │                    │── Redirect ────────▶│
  │◀── Redirect a Google ─│◀─ 302 redirect ────│                    │
  │── Login Google ────────────────────────────────────────────────▶│
  │◀── Authorization code ─────────────────────────────────────────│
  │── GET /auth/callback?code=... ─────────────▶│                    │
  │                       │                    │── POST /token ─────▶│
  │                       │                    │◀─ access_token ─────│
  │                       │                    │── Query BD roles ───│
  │◀── JWT (HS256, 1h) ────────────────────────│                    │
  │── API calls con Authorization: Bearer JWT ─▶│                    │
```

---

## 5. Flujo WebSocket — Progreso de Análisis

```
Angular                    FastAPI                    Claude/DocAI
  │                          │                             │
  │── WS connect ──────────▶│                             │
  │                          │ (acepta conexión)           │
  │── POST /analizar ───────▶│                             │
  │◀── 202 Accepted ─────────│                             │
  │                          │── BackgroundTask launch ───▶│
  │                          │                             │
  │◀── WS: PROGRESO_ANALISIS─│  (cotizacion 1/18)          │
  │                          │── llamada Claude ──────────▶│
  │◀── WS: PDF_PROCESADO ────│◀─ respuesta ────────────────│
  │                          │                             │
  │  ... (por cada PDF) ...  │                             │
  │                          │                             │
  │◀── WS: ANALISIS_COMPLETADO│                            │
  │◀── WS: close ────────────│ (pipeline finalizado)       │
```

---

## 6. Mapping Componentes → Infraestructura

| Componente | Módulo Python | Infraestructura | Protocolo |
|---|---|---|---|
| C-08 AutenticadorOAuth | `app/modules/auth/` | App Runner (FastAPI) | HTTPS/OAuth2 |
| C-10 APIRest | `app/core/` + `app/modules/*/routers/` | App Runner (FastAPI) | HTTPS/REST |
| C-13 RepositorioLocal (BD) | `app/core/database.py` | Amazon RDS PostgreSQL | asyncpg/TLS |
| C-13 RepositorioLocal (S3) | `app/core/storage.py` | Amazon S3 | HTTPS/Presigned |
| C-14 GestorWebSocket | `app/modules/websocket/` | App Runner (FastAPI WS) | WSS |
| Pipeline IA | `app/modules/analisis/pipeline.py` | App Runner (in-process) | Internal async |
| AuditTrail | `app/modules/audit/` + PostgreSQL trigger | Amazon RDS | asyncpg |

---

## 7. Puertos y Endpoints

| Ambiente | Protocolo | Endpoint | Descripción |
|---|---|---|---|
| Production | HTTPS | `https://{servicio}.{region}.awsapprunner.com/api/v1/` | API REST |
| Production | WSS | `wss://{servicio}.{region}.awsapprunner.com/ws/{portafolio_id}` | WebSocket |
| Production | HTTPS | `https://{servicio}.{region}.awsapprunner.com/health` | Health check |
| Development | HTTP | `http://localhost:8000/api/v1/` | API REST |
| Development | WS | `ws://localhost:8000/ws/{portafolio_id}` | WebSocket |
| Development | HTTP | `http://localhost:8000/docs` | Swagger UI (solo dev) |

---

## 8. Consideraciones de Seguridad de la Infraestructura

| Control | Implementación | Ubicación |
|---|---|---|
| Cifrado en tránsito | TLS 1.2+ (ACM en App Runner) | Networking |
| Cifrado en reposo (BD) | RDS encryption at rest (AWS KMS) | RDS |
| Cifrado en reposo (S3) | SSE-S3 (AES-256) | S3 |
| Secrets management | AWS Secrets Manager | App Runner env injection |
| Network isolation | VPC privada para RDS | VPC |
| Acceso mínimo (S3) | IAM Role con permisos específicos por prefijo | IAM |
| Acceso mínimo (RDS) | Security Group restrictivo | VPC |
| Sin acceso público RDS | No public accessibility en RDS | RDS |
| Logs sin datos sensibles | Solo UUIDs y correlation_id | CloudWatch |

---

*Generado por AIDLC Construction Workflow — 2026-05-26*
