# Infraestructura Compartida — Assistent Buy AI Agent

> **Definida en**: U-01 Fundación  
> **Aplicable a**: U-01, U-02, U-03, U-04, U-05, U-06, U-07  
> **Proveedor Cloud**: AWS  
> **Fecha**: 2026-05-26  
> **Estado**: GENERADO — Pendiente aprobación

---

## Propósito

Este documento define la infraestructura base compartida por todas las unidades de trabajo (U-01 a U-07) del sistema Assistent Buy AI Agent.

El sistema es un **monolito modular**: todas las unidades comparten el mismo proceso FastAPI y la misma base de datos PostgreSQL. Las unidades NO tienen infraestructura independiente — se construyen sobre esta base.

**Regla**: Cada unidad (U-02 a U-07) **solo documenta sus adiciones específicas** a esta infraestructura (nuevas tablas, nuevos endpoints, nuevos servicios externos). Los componentes listados aquí **no se repiten** en los documentos de infraestructura de cada unidad.

---

## 1. Plataforma Base Compartida

| Componente | Servicio AWS | Compartido por |
|---|---|---|
| Servidor de aplicación | AWS App Runner (FastAPI/Python 3.12) | U-01…U-07 |
| Base de datos relacional | Amazon RDS for PostgreSQL 16 | U-01…U-07 |
| Almacenamiento de archivos | Amazon S3 | U-01…U-07 |
| Logs centralizados | Amazon CloudWatch Logs | U-01…U-07 |
| Secrets | AWS Secrets Manager | U-01…U-07 |
| Registry de imágenes | Amazon ECR | U-01…U-07 |
| CI/CD | GitHub Actions | U-01…U-07 |
| Autenticación | Google Workspace OAuth2 (via FastAPI) | U-01…U-07 |

---

## 2. Cómputo Compartido — AWS App Runner

- **Servicio**: AWS App Runner (CaaS serverless)
- **Runtime**: Python 3.12 + FastAPI 0.115.x + Uvicorn 4 workers
- **Sizing**: 1 vCPU, 2 GB RAM, 1–3 instancias
- **Puerto**: 8000 (interno), HTTPS/WSS expuesto por App Runner
- **Health check**: `GET /health` → 200 OK

Todas las unidades agregan sus módulos (`app/modules/{dominio}/`) al mismo proceso FastAPI. No hay microservicios separados en el piloto.

---

## 3. Base de Datos Compartida — Amazon RDS PostgreSQL 16

- **Servicio**: Amazon RDS for PostgreSQL 16
- **Instancia**: db.t4g.micro (piloto)
- **Base de datos**: `assitant_buy` (única, compartida por todas las unidades)
- **Migraciones**: Alembic — una cadena de migraciones única para todo el sistema
- **Conexión**: asyncpg via VPC Connector (App Runner → VPC → RDS)

### Esquema compartido (definido en U-01)
Las siguientes tablas son infraestructura base creada por U-01:
- `usuarios` — gestión de identidades y roles
- `audit_trail` — registro append-only de todos los eventos del sistema

Las demás tablas son definidas por cada unidad en su respectiva fase de Code Generation.

---

## 4. Almacenamiento Compartido — Amazon S3

- **Bucket**: `assitant-buy-pdfs-{account_id}` (único, compartido)
- **Acceso**: Presigned URLs (15 min de expiración)
- **Cifrado**: SSE-S3 en reposo
- **Acceso público**: Bloqueado completamente

### Estructura de prefijos (responsabilidad por unidad)
```
s3://assitant-buy-pdfs-{account_id}/
  portafolios/         ← U-02 (gestión de portafolios)
  cotizaciones/        ← U-03 (procesamiento de PDFs)
  reportes/            ← U-06 (generación de reportes)
```

---

## 5. Monitoreo Compartido

### Logs (CloudWatch)
- Log group: `/aws/apprunner/assitant-buy/{service-id}/application`
- Retención: 90 días activos
- Formato: texto plano con `correlation_id` UUID
- **Todos los módulos** escriben a este mismo log group via `structlog` o `logging`

### Alerta compartida
- **Alarma de downtime**: ≥ 3 healthcheck failures consecutivos → SNS → Email al equipo
- Aplicable a toda la aplicación (no por unidad)

---

## 6. Autenticación Compartida

- **Proveedor**: Google Workspace OAuth2 (PKCE flow)
- **Tokens**: JWT HS256, expiración 1 hora, refresh automático
- **Roles**: `ANALISTA`, `ADMIN`, `CISO`, `COMITE` — almacenados en tabla `usuarios` PostgreSQL
- **Bootstrap**: Email ADMIN configurado en AWS Secrets Manager

La autenticación y autorización es transversal — todos los endpoints de todas las unidades la usan.

---

## 7. CI/CD Compartido — GitHub Actions

- **Pipeline único** para todo el monolito
- Stages: test → lint → build → push ECR → deploy App Runner
- **Tests**: `pytest --cov=app --cov-fail-under=80` (cobertura mínima 80% global)
- Cada unidad agrega sus tests al mismo suite de `pytest`

---

## 8. Ambientes

| Ambiente | Infraestructura | Propósito |
|---|---|---|
| `development` | Docker Compose local (app + PostgreSQL + LocalStack) | Desarrollo y pruebas locales de los devs |
| `production` | AWS (App Runner + RDS + S3 + CloudWatch) | Piloto con usuarios reales |

No hay ambiente `staging` en el piloto.

---

## 9. Adiciones Específicas por Unidad

Las unidades U-02 a U-07 documentan **solo** lo que agregan a esta base:

| Unidad | Adiciones esperadas |
|---|---|
| U-01 Fundación | *Esta infraestructura* (base completa) |
| U-02 Portafolios | Tablas: `portafolios`, `cotizaciones` |
| U-03 Procesamiento PDFs | Integración: Document AI API; tablas: `documentos_pdf`, `variables_extraidas` |
| U-04 Análisis IA | Integración: Claude API (Anthropic); tablas: `discrepancias` |
| U-05 HITL / Aclaraciones | Tablas: `aclaraciones`, `discrepancia_aclaracion` |
| U-06 Reportes | Prefijo S3: `reportes/`; integración: PDF generation library |
| U-07 Dashboard | Sin infraestructura nueva (usa datos existentes via API) |

---

*Generado por AIDLC Construction Workflow — 2026-05-26*
