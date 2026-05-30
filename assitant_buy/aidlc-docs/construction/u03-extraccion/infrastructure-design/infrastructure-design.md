# Infrastructure Design — U-03 Extracción, Cruce y Orquestación

> **Unidad**: U-03 — Extracción, Cruce y Orquestación
> **Fase**: CONSTRUCTION — Infrastructure Design
> **Fecha**: 2026-05-29
> **Proveedor Cloud**: AWS
> **Ambientes**: development (local) + production (AWS)
> **Scope MVP**: solo BIENES

---

## 1. Resumen de Componentes Lógicos Mapeados

U-03 **hereda toda la infraestructura de U-01 y U-02** (App Runner, RDS PostgreSQL 16, S3, CloudWatch, ECR, Secrets Manager, Textract). Sus 4 componentes corren **in-process** en el mismo contenedor FastAPI/asyncio — no introducen servicios de cómputo nuevos.

| Componente Lógico | Servicio AWS | Notas |
|---|---|---|
| C-03 MotorExtraccion | App Runner (FastAPI) — heredado | In-process; orquestado por C-09 como tarea asyncio por PDF |
| C-04 MotorCruce | App Runner (FastAPI) — heredado | In-process; reglas deterministas (`Decimal`), sin red |
| C-09 ServicioPipeline | App Runner (FastAPI) — heredado | BackgroundTask asyncio; semáforo `MAX_PDFS_CONCURRENTES=3` |
| C-11 GatewayLLM | App Runner (FastAPI) — heredado + **egress Anthropic** | Llamadas salientes HTTPS a `api.anthropic.com` |
| Catálogo de variables | Amazon RDS — heredado | Lectura directa por extracción (sin cache — NFR-U03-05.3) |
| Auditoría de llamadas LLM | **Amazon S3 — nuevo prefijo** `portafolios/{pid}/llm_calls/` | JSON por llamada (BR-U03-37) |
| API key Anthropic | **AWS Secrets Manager — nuevo secret** `assitant-buy/anthropic` | Q6 = A |
| Logs | Amazon CloudWatch Logs — heredado | Logs de extracción/cruce (sin datos de negocio) |
| Alertas | CloudWatch Alarms — heredado + **1 alarma nueva** | Solo errores LLM (Q4 = B) |

---

## 2. Cómputo — AWS App Runner (heredado, sin cambios de sizing) — Q1 = A

U-03 **no modifica el sizing** de U-02 (**2 vCPU / 4 GB**). El pipeline de U-03 está dominado por **I/O de red** (espera de respuestas del LLM, 20–30 s/llamada), con consumo local mínimo (`rapidfuzz` + aritmética `Decimal`).

| Parámetro | Valor (heredado de U-02) | Justificación U-03 |
|---|---|---|
| CPU | 2 vCPU | Sin presión adicional — U-03 es I/O-bound, no CPU-bound |
| Memoria | 4 GB | Textos de PDFs en vuelo (3 concurrentes × ~50–200 KB) despreciable frente a 4 GB |
| Instancias min/max | 1 / 3 | Sin cambio |

El `apprunner.yaml` de U-02 se reutiliza tal cual. La BackgroundTask de U-03 corre en el mismo event loop de Uvicorn, independiente del timeout de request HTTP (igual que U-02).

---

## 3. Cómputo del LLM — Anthropic API (servicio externo gestionado)

El "cómputo de inferencia" de U-03 es **externo**: lo provee Anthropic. No hay GPU/instancias propias.

| Atributo | Valor |
|---|---|
| Endpoint | `https://api.anthropic.com` (HTTPS público) |
| Modelos | `claude-sonnet-4-6` (primario), `claude-opus-4-7` (fallback BR-U03-06) |
| SDK | `anthropic` async (`AsyncAnthropic`) |
| Tier inicial | **Tier 1** (cuenta nueva) → `MAX_PDFS_CONCURRENTES=3` (NFR-U03-02.1) |
| Reintentos | `max_retries` del SDK con `retry-after` (NFR-U03-03.1) |

> ⚠️ **GATE de Code Generation**: crear cuenta en `console.anthropic.com` + API key (NFR-U03-08.2 / cross-unit-deltas §4.6). La suscripción Claude Max NO habilita la API programática.

---

## 4. Almacenamiento — Amazon S3 (heredado + prefijo `llm_calls/`) — Q2 = A

### Estructura de prefijos actualizada

```
s3://assitant-buy-pdfs-{account_id}/
  portafolios/{portafolio_id}/
    pdfs/{documento_pdf_id}.pdf          ← U-02 (PDF original)
    ocr/{documento_pdf_id}.txt           ← U-02 (texto OCR)
    llm_calls/{call_id}.json             ← U-03 (auditoría LLM — NUEVO)
```

### Política de ciclo de vida (Q2 = A)

| Prefijo | Lifecycle |
|---|---|
| `llm_calls/` | **Sin transición automática en el piloto** — permanece en S3 Standard. Política de retención/archivado se define **post-piloto con el CISO** (coherente con NFR-U03-05.2: retención indefinida en piloto) |

**Rationale**: la auditoría LLM es evidencia de cumplimiento (US-19); no se arriesga su disponibilidad con transiciones a Glacier durante el piloto. El volumen es bajo (JSON de texto, no binarios).

### Actualización de IAM Role — prefijo `llm_calls/`

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": [
    "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*/llm_calls/*"
  ]
}
```

> El `Resource` consolidado `portafolios/*` del IAM Role de U-02 (§10) ya cubre este prefijo; la entrada explícita se documenta por claridad.

---

## 5. Base de Datos — Amazon RDS PostgreSQL 16 (heredado)

U-03 lee el catálogo y entidades de la BD propia. **No agrega engine nuevo** (a diferencia de U-02, que añadió el engine read-only del SAB — U-03 lee de *nuestra* BD por BR-U03-39).

### Migraciones Alembic relevantes para U-03

Las entidades nuevas (`CatalogoVariable`, `LlamadaLLM`, `SolicitudCotizacion`, `ItemSolicitud`, `ItemCotizado`) y los `ALTER TABLE` están en `cross-unit-deltas.md` §1 (delta a U-01). **Se aplican en el Code Generation** (estrategia Opción C), no en esta etapa de diseño.

| Aspecto | Nota |
|---|---|
| Lectura del catálogo | `SELECT ... FROM catalogo_variable WHERE activo=TRUE AND tipo_adquisicion='BIENES'` por extracción (sin cache) |
| Pool de conexiones | Heredado de U-01; las lecturas de catálogo (pequeñas, frecuentes) no requieren ajuste |
| Persistencia | `VariableExtraida`, `Discrepancia`, `LlamadaLLM` (metadata) vía `local_engine` |

---

## 6. Networking — Egress a Anthropic (Q3 = A)

| Atributo | Decisión |
|---|---|
| Topología | **Internet público HTTPS** desde App Runner — sin VPC Connector ni NAT Gateway |
| Destino | `api.anthropic.com:443` |
| TLS | Verificación habilitada (sin bypass), igual que la conexión al SAB (U-02 §7) |
| IP estática | No requerida — Anthropic no exige IP whitelist |

**Rationale**: idéntico patrón al egress hacia el SAB. Evita el costo de NAT Gateway (~$32/mes) y la complejidad de VPC Connector. Si una política corporativa futura exige IP de salida fija, se migraría a VPC Connector + NAT (documentado como opción B del plan, no activada).

---

## 7. Gestión de Credenciales — AWS Secrets Manager (Q6 = A)

### Nuevo Secret de U-03

| Secret Name | Contenido | Rotación |
|---|---|---|
| `assitant-buy/anthropic` | `ANTHROPIC_API_KEY` (+ opcional `ANTHROPIC_MODEL_PRIMARY`, `ANTHROPIC_MODEL_FALLBACK`) | Manual (regenerar key en console.anthropic.com) |

### Inyección en App Runner

```yaml
env:
  - name: ANTHROPIC_API_KEY
    valueFrom: arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/anthropic:ANTHROPIC_API_KEY::
```

El IAM Role de U-02 ya concede `secretsmanager:GetSecretValue` sobre `assitant-buy/*` (§10), por lo que **no requiere cambio de política** para leer el nuevo secret.

---

## 8. Monitoreo — CloudWatch (heredado + 1 alarma) — Q4 = B

Coherente con la observabilidad mínima de NFR-U03-06. Se crea **una sola alarma nueva**; la degradación por tasa de escalamiento (Gate G1) se revisa manualmente desde logs/AuditTrail.

```
Alarma 5: LLM Errors
  Métrica: Custom metric "AssitantBuy/U03/LLMErrorCount"
           (publicada al agotar reintentos → ERROR_EXTRACCION)
  Umbral: ≥ 5 errores LLM en 1 hora
  Acción: SNS → Email al equipo técnico
```

```python
# Publicado desde C-09/C-11 al agotar reintentos del LLM
cloudwatch.put_metric_data(
    Namespace='AssitantBuy/U03',
    MetricData=[{'MetricName': 'LLMErrorCount', 'Value': 1, 'Unit': 'Count'}]
)
```

**No se crea alarma de tasa de escalamiento** (Q4 = B): el evento `ALERTA_DEGRADACION` (BR-U03-38) se sigue emitiendo al AuditTrail y se revisa manualmente, sin disparar email automático en el piloto.

---

## 9. Infraestructura Compartida — convivencia de pipelines (Q5 = C)

| Atributo | Decisión |
|---|---|
| Gestión de concurrencia entre U-02 y U-03 | **Ninguna explícita** — se confía en el intercalado natural de asyncio |
| Justificación | Ambos pipelines son I/O-bound (U-02: red SAB/S3; U-03: red Anthropic). El event loop intercala sus `await` sin bloqueo. La contención de CPU/memoria es despreciable para la carga del piloto |
| Semáforos | Cada pipeline mantiene el suyo (U-02: descargas; U-03: `MAX_PDFS_CONCURRENTES=3`), pero no hay semáforo global cruzado |

> Si en producción se procesan múltiples portafolios en paralelo de forma sostenida, se reconsiderará un semáforo global por tipo de tarea (opción B del plan).

---

## 10. IAM Role Consolidado (U-01 + U-02 + U-03)

El IAM Role de App Runner no requiere permisos nuevos: los recursos de U-03 (`portafolios/*/llm_calls/*` en S3 y `assitant-buy/anthropic` en Secrets Manager) ya están cubiertos por los wildcards de U-02 (§10). Se documenta el alcance efectivo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "S3Portafolios", "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::assitant-buy-pdfs-{account_id}/portafolios/*" },
    { "Sid": "TextractOCR", "Effect": "Allow",
      "Action": ["textract:DetectDocumentText","textract:StartDocumentTextDetection","textract:GetDocumentTextDetection"],
      "Resource": "*" },
    { "Sid": "SecretsManager", "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:{account_id}:secret:assitant-buy/*" },
    { "Sid": "CloudWatchMetrics", "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"], "Resource": "*" }
  ]
}
```

> **Nota**: Anthropic API NO es un servicio AWS — no requiere permisos IAM. El control de acceso es la API key en Secrets Manager.

---

## 11. Estimación de Costos (incremento de U-03 sobre U-01+U-02)

El costo de infraestructura AWS que **agrega U-03** es marginal; el costo dominante es el **consumo de tokens de Anthropic** (no es infra AWS — ver NFR-U03-01.2).

| Concepto | Configuración | Estimado mensual (USD) |
|---|---|---|
| Secrets Manager | +1 secret (`assitant-buy/anthropic`) | +$0.40 |
| S3 `llm_calls/` | JSON de auditoría, bajo volumen en piloto | < $0.50 |
| CloudWatch | +1 alarma + custom metric `LLMErrorCount` | ~$0.30 |
| App Runner | Sin cambio de sizing | $0 |
| **Subtotal infra AWS U-03** | | **~$1.20/mes** |
| **Anthropic API (tokens)** | Pay-per-token — variable según volumen | **No es infra AWS** (ver NFR-U03-01.2) |

> El costo total de infraestructura del piloto (U-01+U-02+U-03) sigue en **~$76–106/mes** de AWS. El gasto de Anthropic se mide aparte por portafolio vía `LlamadaLLM`.

---

*Generado por AIDLC Construction Workflow — 2026-05-29*
