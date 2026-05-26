# Infrastructure Design — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — Infrastructure Design  
> **Fecha**: 2026-05-26  
> **Proveedor Cloud**: AWS  
> **Ambientes**: development (local) + production (AWS)  
> **Estado**: APROBADO — 2026-05-26T22:39:00Z

---

## 1. Resumen de Componentes Lógicos Mapeados

U-02 **hereda toda la infraestructura de U-01** (App Runner, RDS, S3, CloudWatch, ECR, Secrets Manager). Las adiciones específicas de U-02 son los servicios requeridos por el pipeline de ingesta/sanitización.

| Componente Lógico | Servicio AWS | Notas |
|---|---|---|
| C-01 IngestaConector | App Runner (FastAPI) — heredado | BackgroundTask de larga duración (hasta 25 min) |
| C-02 SanitizadorZeroTrust | App Runner (FastAPI) — heredado | In-process, sin infraestructura adicional |
| C-12 RepositorioSAB (BD) | BD externa del SAB (acceso read-only) | No es un servicio AWS; se conecta via VPC + Security Group |
| C-12 GestorTokenSAB (Keycloak) | Keycloak externo del SAB | Autenticación HTTPS; credenciales en AWS Secrets Manager |
| C-16 GatewayDocumentAI (OCR) | **Amazon Textract** | OCR de PDFs escaneados; llamadas via aiobotocore |
| Almacenamiento PDFs | Amazon S3 — heredado | Nueva estructura de prefijos: `portafolios/{pid}/pdfs/` |
| Almacenamiento texto OCR | Amazon S3 — heredado (mismo bucket) | Companion files `.txt`: `portafolios/{pid}/ocr/` |
| Credenciales SAB | **AWS Secrets Manager** | Nuevos secrets: `sab/keycloak`, `sab/db` |
| Logs | Amazon CloudWatch Logs — heredado | Logs de BackgroundTask de ingesta y OCR |
| Alertas | CloudWatch Alarms — heredado | Nueva alarma para fallos de OCR |

---

## 2. Cómputo — AWS App Runner (heredado + ajuste de timeout)

### Servicio
Mismo App Runner de U-01. U-02 no añade nuevas instancias — el pipeline de ingesta corre en BackgroundTask dentro del mismo contenedor FastAPI.

### Ajuste crítico: timeout de BackgroundTask

> ⚠️ **Problema**: App Runner tiene un timeout de **20 minutos por request**. Sin embargo, el endpoint `POST /analizar` responde HTTP 202 inmediatamente y lanza una BackgroundTask. La BackgroundTask **no está sujeta al timeout del request HTTP** — corre de forma independiente en el event loop de Uvicorn.
>
> **Consecuencia**: Para portafolios grandes (200 PDFs × 25 min), la BackgroundTask puede seguir corriendo aunque el request HTTP ya respondió. Esto es el comportamiento correcto en FastAPI + asyncio.

### Ajuste de sizing para U-02

| Parámetro | U-01 (base) | U-02 (ajuste) | Razón |
|---|---|---|---|
| CPU | 1 vCPU | **2 vCPU** | El pipeline de sanitización (pypdf + regex) es CPU-intensivo por PDF |
| Memoria | 2 GB | **4 GB** | PDFs de hasta 50 MB en memoria + buffer de Textract + múltiples workers |
| Instancias mínimas | 1 | 1 | Sin cambio |
| Instancias máximas | 3 | 3 | Sin cambio |

```yaml
# apprunner.yaml (actualizado para U-02)
version: 1.0
runtime: docker
run:
  command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
  network:
    port: 8000
  env:
    - name: ENV
      value: production
  resources:
    cpu: 2 vCPU
    memory: 4 GB
```

---

## 3. Base de Datos — Amazon RDS for PostgreSQL 16 (heredado)

Sin cambios respecto a U-01. U-02 agrega los campos `es_escaneado` y `texto_ocr_s3_key` a la tabla `documento_pdf` vía migración Alembic (gestionada en la fase de Build).

### Impacto de la migración Alembic para U-02

```sql
-- Migración: add_ocr_fields_to_documento_pdf
ALTER TABLE documento_pdf
  ADD COLUMN es_escaneado BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN texto_ocr_s3_key VARCHAR(500) NULL;
```

---

## 4. Almacenamiento — Amazon S3 (heredado + prefijos nuevos)

### Estructura de prefijos actualizada

```
s3://assitant-buy-pdfs-{account_id}/
  portafolios/{portafolio_id}/
    pdfs/
      {documento_pdf_id}.pdf              ← PDF original del SAB
    ocr/
      {documento_pdf_id}.txt              ← Texto OCR (solo si es_escaneado=TRUE)
```

### Actualización de IAM Role (App Runner)

Se añaden los permisos para el prefijo `ocr/` (el prefijo `pdfs/` ya estaba en U-01):

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": [
    "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*/pdfs/*",
    "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*/ocr/*"
  ]
}
```

### Política de ciclo de vida (actualizada)

```
- PDFs (portafolios/*/pdfs/): > 1 año → S3 Glacier Instant Retrieval
- Textos OCR (portafolios/*/ocr/): > 1 año → S3 Glacier Instant Retrieval
- No expiración automática (registros de negocio permanentes)
```

---

## 5. OCR — Amazon Textract

### Servicio

**Amazon Textract** — servicio gestionado de AWS para extracción de texto de documentos (PDFs e imágenes).

| Atributo | Valor |
|---|---|
| **API usada (< 5 páginas)** | `DetectDocumentText` — síncrona, respuesta directa |
| **API usada (≥ 5 páginas)** | `StartDocumentTextDetection` + `GetDocumentTextDetection` — asíncrona con polling |
| **SDK** | `aiobotocore` (ya en el stack para S3) — sin librería adicional |
| **Formato de entrada** | Bytes del PDF directamente (sin subir a S3 primero para Textract) |
| **Precio estimado** | ~$1.50 por 1,000 páginas (piloto: bajo volumen) |
| **Región** | `us-east-1` — misma que S3 y App Runner (sin costo de transferencia entre servicios) |

### Integración con IAM Role

Textract usa el mismo IAM Role de App Runner. Se añade el permiso:

```json
{
  "Effect": "Allow",
  "Action": [
    "textract:DetectDocumentText",
    "textract:StartDocumentTextDetection",
    "textract:GetDocumentTextDetection"
  ],
  "Resource": "*"
}
```

> **Nota**: Textract no tiene ARN de recurso específico — `"Resource": "*"` es el estándar de AWS para este servicio.

### Flujo de llamada

```
BackgroundTask (App Runner)
    │
    ├─ PDF ≤ 5 páginas:
    │      aiobotocore.textract.detect_document_text(
    │          Document={'Bytes': pdf_bytes}
    │      ) → respuesta síncrona con texto
    │
    └─ PDF > 5 páginas:
           aiobotocore.textract.start_document_text_detection(
               DocumentLocation={'S3Object': {'Bucket': bucket, 'Name': pdf_s3_key}}
           ) → job_id
           
           Polling cada 2s hasta que job_id Status = SUCCEEDED
           aiobotocore.textract.get_document_text_detection(JobId=job_id)
           → texto extraído página por página
```

---

## 6. Gestión de Credenciales SAB — AWS Secrets Manager

### Nuevos Secrets de U-02

| Secret Name | Contenido | Rotación |
|---|---|---|
| `assitant-buy/sab/keycloak` | `SAB_KEYCLOAK_URL`, `SAB_CLIENT_ID`, `SAB_CLIENT_SECRET`, `SAB_SERVICE_USERNAME`, `SAB_SERVICE_PASSWORD` | Manual (trimestral, coordinado con TI del SAB) |
| `assitant-buy/sab/db` | `SAB_DB_URL` (incluyendo usuario y contraseña read-only de la BD del SAB) | Manual |

### Inyección en App Runner

```yaml
# En la configuración de App Runner — Variables de entorno desde Secrets Manager
env:
  - name: SAB_SERVICE_PASSWORD
    valueFrom: arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/sab/keycloak:SAB_SERVICE_PASSWORD::
  - name: SAB_CLIENT_SECRET
    valueFrom: arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/sab/keycloak:SAB_CLIENT_SECRET::
  # ... resto de variables del secret
```

### IAM Role — Permiso para leer Secrets Manager

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": [
    "arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/sab/*"
  ]
}
```

---

## 7. Conectividad al SAB Externo

### Topología de red

El SAB es un sistema externo (Universidad del Cauca) accesible vía internet público HTTPS. No se requiere VPN ni Direct Connect para el piloto.

```
App Runner (AWS us-east-1)
    │
    │ HTTPS (port 443) — internet público
    │
    ├─▶ SAB Keycloak: https://proyunicauca.docxflow.com:8443
    │       OAuth2 ROPC token endpoint
    │
    └─▶ SAB API REST: https://backend.unicauca.edu.co
            GET /sab-bienes/api/v1/cotizacion/documentos/descargar/{id}
```

### Consideraciones de seguridad para conexión al SAB externo

| Control | Implementación |
|---|---|
| TLS | Verificación SSL habilitada (`verify=True` en httpx); no se deshabilita |
| Credenciales | Nunca en logs; siempre desde Secrets Manager |
| Timeout | 30 segundos por descarga; 5 segundos para conexión Keycloak |
| IP estática (opcional) | App Runner NO provee IP estática nativa. Si el SAB requiere whitelist de IPs: usar NAT Gateway en la VPC + VPC Connector. Para el piloto: no se implementa (el SAB acepta cualquier IP autenticada) |

---

## 8. Monitoreo — CloudWatch (heredado + alarmas U-02)

### Nuevas alarmas específicas de U-02

```
Alarma 2: OCR Failures
  Métrica: Custom metric "U02/OCR/FailureCount" (publicada desde app)
  Umbral: ≥ 5 fallos de OCR en 1 hora
  Acción: SNS → Email al equipo técnico

Alarma 3: Ingesta Stalled
  Métrica: Custom metric "U02/Pipeline/IngestaStalledCount"
  Umbral: Portafolio en estado INGESTA por > 30 minutos
  Acción: SNS → Email al equipo técnico

Alarma 4: SAB Authentication Failures
  Métrica: Custom metric "U02/SAB/AuthFailureCount"
  Umbral: ≥ 1 fallo de autenticación Keycloak
  Acción: SNS → Email URGENTE (indica credenciales expiradas)
```

### Métricas custom (publicadas desde el código con `boto3.client('cloudwatch')`)

```python
# Publicar desde la BackgroundTask al finalizar la ingesta de un PDF
cloudwatch.put_metric_data(
    Namespace='AssitantBuy/U02',
    MetricData=[{
        'MetricName': 'OCRFailureCount',
        'Value': 1 if ocr_fallo else 0,
        'Unit': 'Count'
    }]
)
```

---

## 9. Variables de Entorno por Ambiente (U-02)

### Development (docker-compose local)

```env
# SAB — BD (para dev, puede ser BD de prueba o mock)
SAB_DB_URL=postgresql+asyncpg://readonly:password@sab-host-dev:5432/sab_dev

# SAB — API REST + Keycloak (entorno QA del SAB)
SAB_BASE_URL=https://backend.unicauca.edu.co/sab-bienes
SAB_KEYCLOAK_URL=https://proyunicauca.docxflow.com:8443/realms/unicauca-funcionarios-qa
SAB_CLIENT_ID=dev-client-id
SAB_CLIENT_SECRET=dev-secret
SAB_SERVICE_USERNAME=assistant_buy_dev
SAB_SERVICE_PASSWORD=dev-password

# Límites de pipeline
SAB_DOWNLOAD_TIMEOUT_S=30
PDF_MAX_SIZE_MB=50

# S3 — LocalStack para dev sin AWS real
S3_BUCKET_NAME=assitant-buy-pdfs-local
AWS_ENDPOINT_URL=http://localhost:4566  # LocalStack
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_REGION=us-east-1

# Textract — LocalStack (moto en tests; LocalStack para dev manual)
# En local, Textract se puede mockear con moto en tests
# Para integración manual en dev, se puede usar AWS real con credenciales personales
```

### Production (App Runner — valores desde AWS Secrets Manager)

```env
# Inyectados automáticamente desde Secrets Manager:
SAB_DB_URL             ← assitant-buy/sab/db
SAB_KEYCLOAK_URL       ← assitant-buy/sab/keycloak
SAB_CLIENT_ID          ← assitant-buy/sab/keycloak
SAB_CLIENT_SECRET      ← assitant-buy/sab/keycloak  [SECRETO]
SAB_SERVICE_USERNAME   ← assitant-buy/sab/keycloak
SAB_SERVICE_PASSWORD   ← assitant-buy/sab/keycloak  [SECRETO]

# Variables de entorno directas (no secretas):
SAB_BASE_URL=https://backend.unicauca.edu.co/sab-bienes
SAB_DOWNLOAD_TIMEOUT_S=30
PDF_MAX_SIZE_MB=50
S3_BUCKET_NAME=assitant-buy-pdfs-{account_id}
AWS_REGION=us-east-1
```

---

## 10. IAM Role Consolidado para App Runner (U-01 + U-02)

El IAM Role de App Runner acumula todos los permisos de U-01 y U-02:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3PDFs",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*"
    },
    {
      "Sid": "TextractOCR",
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText",
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SecretsManagerSAB",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/*"
    },
    {
      "Sid": "CloudWatchMetrics",
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*"
    }
  ]
}
```

---

## 11. Estimación de Costos (Piloto — U-01 + U-02 combinado)

| Servicio | Configuración | Estimado mensual (USD) |
|---|---|---|
| AWS App Runner | 2 vCPU, 4 GB, ~1 instancia | ~$50–70 |
| Amazon RDS | db.t4g.micro, 20 GB gp3 | ~$15–20 |
| Amazon S3 | PDFs + textos OCR, < 20 GB piloto | < $2 |
| **Amazon Textract** | ~500 páginas OCR/mes (estimado piloto) | **~$0.75** |
| CloudWatch Logs + Metrics | ~5 GB/mes logs + custom metrics | ~$5–8 |
| ECR | Almacenamiento imágenes Docker | ~$1–2 |
| AWS Secrets Manager | 8–12 secrets (U-01 + U-02) | ~$3 |
| **Total estimado** | | **~$75–105/mes** |

> **Nota**: El costo de Textract para el piloto es mínimo (~$0.75/mes) asumiendo 500 páginas OCR procesadas. A $1.50/1,000 páginas, solo se convertiría en costo significativo en producción con volúmenes altos.

---

*Generado por AIDLC Construction Workflow — 2026-05-26*
