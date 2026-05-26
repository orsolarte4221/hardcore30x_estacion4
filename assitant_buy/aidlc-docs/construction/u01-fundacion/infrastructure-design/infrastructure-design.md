# Infrastructure Design — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Infrastructure Design  
> **Fecha**: 2026-05-26  
> **Proveedor Cloud**: AWS  
> **Ambientes**: development (local) + production (AWS)  
> **Estado**: GENERADO — Pendiente aprobación

---

## 1. Resumen de Componentes Lógicos Mapeados

| Componente Lógico | Servicio AWS | Notas |
|---|---|---|
| C-08 AutenticadorOAuth | AWS App Runner (FastAPI) | OAuth2 PKCE flow integrado en la app |
| C-10 APIRest | AWS App Runner | Container serverless, escala automática |
| C-13 RepositorioLocal (BD) | Amazon RDS for PostgreSQL 16 | Managed, backups automáticos |
| C-13 RepositorioLocal (Storage) | Amazon S3 | PDFs de cotizaciones, acceso via URL firmada |
| C-14 GestorWebSocket | AWS App Runner | WebSocket nativo sobre la misma instancia FastAPI |
| Pipeline IA (async) | AWS App Runner (in-process) | asyncio + BackgroundTasks, sin infraestructura de queue separada |
| Logs | Amazon CloudWatch Logs | Recolección automática de stdout del contenedor |
| Alertas | Amazon CloudWatch Alarms | Alerta de downtime en `/health` |
| CI/CD | GitHub Actions | Pipeline automatizado: tests → build → push → deploy |

---

## 2. Cómputo — AWS App Runner

### Servicio
- **AWS App Runner** — Container as a Service (CaaS) serverless
- Gestiona automáticamente: load balancer, TLS, escalado horizontal, health checks
- **No se requiere** API Gateway ni Nginx adicional en el piloto

### Configuración del contenedor
```yaml
# apprunner.yaml (referencia)
version: 1.0
runtime: docker
build:
  commands:
    build:
      - pip install -r requirements.txt
run:
  command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
  network:
    port: 8000
    env: APP_PORT
  env:
    - name: ENV
      value: production
```

### Sizing para piloto
| Parámetro | Valor |
|---|---|
| CPU | 1 vCPU |
| Memoria | 2 GB |
| Instancias mínimas | 1 (evita cold start) |
| Instancias máximas | 3 |
| Protocolo | HTTPS (gestionado por App Runner) |
| Puerto de aplicación | 8000 |

### Justificación
- **Escala automática** según carga — adecuado para 5–15 usuarios concurrentes del piloto
- **Sin gestión de VM** — el equipo se enfoca en código, no en infraestructura
- **Integración nativa con ECR** (Elastic Container Registry) para el pipeline CI/CD
- **Load Balancer integrado** — no se necesita ALB separado; App Runner lo provee gratuitamente

---

## 3. Base de Datos — Amazon RDS for PostgreSQL 16

### Servicio
- **Amazon RDS for PostgreSQL 16** — base de datos relacional gestionada
- Instancia clase: `db.t4g.micro` (piloto) → `db.t4g.small` si la carga lo requiere

### Configuración
```
Motor: PostgreSQL 16
Clase de instancia: db.t4g.micro (2 vCPU, 1 GB RAM)
Almacenamiento: 20 GB gp3 (SSD), auto-scaling hasta 100 GB
Multi-AZ: No (piloto) — single-AZ para reducir costos
Backup: Automático diario, retención 7 días
Maintenance window: Sábados 02:00–03:00 UTC
```

### Conectividad
- RDS dentro de **VPC privada** — no expuesta a internet
- App Runner se conecta via **VPC Connector** (App Runner → VPC → RDS)
- Puerto: `5432`
- Driver: `asyncpg` (driver async para SQLAlchemy 2.x)

### Variables de entorno en App Runner
```
DATABASE_URL=postgresql+asyncpg://user:pass@rds-host:5432/assitant_buy
DB_POOL_SIZE=10
DB_MAX_OVERFLOW=20
```

### Consideraciones de seguridad
- Credenciales almacenadas en **AWS Secrets Manager** (no en variables de entorno planas)
- Security Group de RDS: solo acepta tráfico desde el Security Group de App Runner
- SSL/TLS habilitado en la conexión (parámetro `sslmode=require`)

---

## 4. Almacenamiento — Amazon S3

### Uso
- Almacenamiento de **PDFs de cotizaciones** subidos por los analistas
- Acceso controlado via **Presigned URLs** (expiración 15 minutos) — no acceso público directo

### Configuración del bucket
```
Nombre: assitant-buy-pdfs-{account_id}  (nombre único global)
Región: us-east-1 (misma que App Runner y RDS)
Versionado: Habilitado (para auditoría y recuperación)
Cifrado: SSE-S3 (AES-256, cifrado en reposo)
Acceso público: BLOQUEADO (Block all public access = true)
```

### Estructura de prefijos en S3
```
s3://assitant-buy-pdfs-{account_id}/
  portafolios/{portafolio_id}/
    cotizaciones/{cotizacion_id}/
      {pdf_nombre}_{timestamp}.pdf
```

### Política de ciclo de vida
```
- Archivos > 1 año → mover a S3 Glacier Instant Retrieval (reducción de costos)
- No expiración automática (los PDFs son registros de negocio permanentes)
```

### IAM Role para App Runner
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*"
}
```

---

## 5. Procesamiento Asíncrono — asyncio + BackgroundTasks

### Estrategia
- **Sin infraestructura de queue separada** en U-01
- El pipeline de IA corre como `BackgroundTask` de FastAPI, dentro del mismo proceso App Runner
- Implementado con `asyncio` nativo de Python 3.12

### Flujo
```
POST /api/v1/portafolios/{id}/analizar
  → FastAPI recibe request
  → Responde 202 Accepted inmediatamente
  → Lanza BackgroundTask: pipeline_analisis_ia()
  → pipeline_analisis_ia() emite eventos WebSocket por progreso
  → Cliente Angular recibe eventos en tiempo real
```

### Limitaciones conocidas (U-01)
- Si App Runner reinicia la instancia, los BackgroundTasks en vuelo se pierden
- Mitigación: campo `estado` en `Portafolio` permite reintentar análisis desde el frontend
- Retry automático: 3 intentos con backoff exponencial por cada llamada a Claude/Document AI (BR-20)

---

## 6. Networking — AWS App Runner Integrado

### SSL/TLS
- App Runner provee **certificado TLS gestionado** automáticamente (sin Let's Encrypt manual)
- HTTPS habilitado por defecto en la URL `https://{service-id}.{region}.awsapprunner.com`
- **Dominio custom**: Si el proyecto tiene dominio propio, se configura en App Runner → Custom Domain → verificación DNS
  - Certificado: App Runner solicita automáticamente via AWS Certificate Manager (ACM)
  - Let's Encrypt (certbot) **NO aplica** en App Runner — ACM lo reemplaza sin gestión manual

> **Nota**: La respuesta 5.1 (Let's Encrypt) fue seleccionada antes de elegir CaaS/App Runner (2.1=A). En App Runner, el certificado SSL es gestionado automáticamente por ACM, lo cual es equivalente o superior a Let's Encrypt sin la complejidad de certbot.

### API Gateway / Load Balancer
- **Sin API Gateway adicional** — App Runner incluye load balancer HTTP/HTTPS integrado
- Rate limiting implementado en FastAPI (BR-16: 100 req/min por usuario)
- Autenticación gestionada por FastAPI (OAuth2/JWT)

### VPC Configuration
```
VPC: assitant-buy-vpc (CIDR: 10.0.0.0/16)
  Subnets privadas: 10.0.1.0/24, 10.0.2.0/24 (para RDS)
  Subnets públicas: 10.0.10.0/24, 10.0.11.0/24 (para VPC Connector de App Runner)

Security Groups:
  sg-apprunner: Outbound 5432 → sg-rds
  sg-rds: Inbound 5432 desde sg-apprunner
```

### WebSocket
- WebSocket nativo soportado en App Runner (protocolo Upgrade HTTP → WS)
- Endpoint: `wss://{dominio}/ws/{portafolio_id}`
- Timeout de conexión WS: 60 segundos idle (configurable en App Runner)

---

## 7. Monitoreo y Observabilidad

### Logs — Amazon CloudWatch Logs
- App Runner envía automáticamente stdout/stderr a CloudWatch Logs
- Log group: `/aws/apprunner/assitant-buy/{service-id}/application`
- Retención de logs: 90 días (configurado en CloudWatch)
- Formato de log: texto plano con `correlation_id` (sin datos sensibles — solo UUIDs)

### Alertas — CloudWatch Alarms
```
Alarma 1: HealthCheck Failure
  Métrica: App Runner HealthCheck consecutive failures
  Umbral: ≥ 3 fallos consecutivos en /health
  Acción: Notificación SNS → Email al equipo
```

### Dashboard
- CloudWatch Dashboard básico: CPU, memoria, request count, error rate (4xx/5xx), latencia p95

---

## 8. CI/CD — GitHub Actions

### Pipeline automatizado
```
Trigger: push a rama main / merge de Pull Request

Stages:
  1. Test      → pytest con cobertura 80% mínima
  2. Lint      → ruff + mypy
  3. Build     → docker build
  4. Push      → docker push a Amazon ECR
  5. Deploy    → aws apprunner start-deployment (CLI)
```

### Archivo de pipeline (referencia)
```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - run: pytest --cov=app --cov-fail-under=80

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
      - run: |
          docker build -t $ECR_REPO:$GITHUB_SHA .
          docker push $ECR_REPO:$GITHUB_SHA
      - run: |
          aws apprunner start-deployment \
            --service-arn ${{ secrets.APP_RUNNER_ARN }}
```

### Secrets en GitHub
```
AWS_ACCESS_KEY_ID       → IAM user con permisos ECR + App Runner
AWS_SECRET_ACCESS_KEY   → Credencial IAM
APP_RUNNER_ARN          → ARN del servicio App Runner
ECR_REPO                → URL del repositorio ECR
```

---

## 9. Variables de Entorno por Ambiente

### Development (local — docker-compose)
```env
ENV=development
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/assitant_buy_dev
AWS_ACCESS_KEY_ID=local-mock (o LocalStack)
AWS_SECRET_ACCESS_KEY=local-mock
S3_BUCKET=assitant-buy-local
GOOGLE_CLIENT_ID={id de OAuth app de desarrollo}
GOOGLE_CLIENT_SECRET={secret de OAuth app de desarrollo}
JWT_SECRET={secret local aleatorio}
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=60
ADMIN_EMAIL={email bootstrap del ADMIN local}
```

### Production (App Runner — AWS Secrets Manager)
```env
ENV=production
DATABASE_URL=          ← desde Secrets Manager
GOOGLE_CLIENT_ID=      ← desde Secrets Manager
GOOGLE_CLIENT_SECRET=  ← desde Secrets Manager
JWT_SECRET=            ← desde Secrets Manager
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=60
S3_BUCKET=assitant-buy-pdfs-{account_id}
ADMIN_EMAIL=           ← desde Secrets Manager
```

---

## 10. Estimación de Costos (Piloto)

| Servicio | Configuración | Estimado mensual (USD) |
|---|---|---|
| AWS App Runner | 1 vCPU, 2 GB, ~1 instancia | ~$25–40 |
| Amazon RDS | db.t4g.micro, 20 GB gp3 | ~$15–20 |
| Amazon S3 | <10 GB estimado piloto | <$1 |
| CloudWatch Logs | ~5 GB/mes logs | ~$2–5 |
| ECR | Almacenamiento imágenes Docker | ~$1–2 |
| AWS Secrets Manager | 5–10 secrets | ~$2 |
| **Total estimado** | | **~$45–70/mes** |

---

*Generado por AIDLC Construction Workflow — 2026-05-26*
