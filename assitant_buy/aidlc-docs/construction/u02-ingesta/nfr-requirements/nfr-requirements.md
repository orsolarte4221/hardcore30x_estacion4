# NFR Requirements — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — NFR Requirements  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI

---

## Nota de Herencia desde U-01

U-02 **hereda todos los NFRs de U-01** (Fundación). Este documento documenta únicamente los **NFRs adicionales o modificados** específicos del pipeline de ingesta y sanitización, con especial atención a:
- La integración cross-cloud con el SAB (BD read-only + API REST con Keycloak)
- El procesamiento de PDFs como archivos binarios (tamaño, throughput, memoria)
- El pipeline Zero Trust con sus requisitos de seguridad específicos
- La BackgroundTask asíncrona de larga duración

---

## NFR-U02-01 — Performance del Pipeline de Ingesta

### NFR-U02-01.1: SLA del Endpoint de Inicio (US-01)

El endpoint `POST /api/v1/procesos/{proceso_id}/analisis` es síncrono solo en la creación del registro `Portafolio`. El pipeline real corre en BackgroundTask.

| Fase | SLA |
|------|-----|
| Validación de permisos + creación `Portafolio` (síncrono) | < 500ms (p95) |
| Primer evento WebSocket `PROGRESO_PIPELINE` | < 10 segundos desde el 202 |
| Timeout de conexión de prueba al SAB | 5 segundos |

### NFR-U02-01.2: Throughput del Pipeline (BackgroundTask)

| Atributo | Valor objetivo | Condición |
|---|---|---|
| PDFs procesados por minuto | ≥ 10 PDFs/min | Con red estable al SAB y S3 |
| Tiempo estimado para portafolio completo (200 PDFs) | ≤ 25 minutos | Carga máxima piloto |
| Tiempo de sanitización por PDF (solo procesamiento local) | < 5 segundos/PDF | Excluye descarga y upload |

**Nota**: El tiempo dominante en el pipeline es la descarga de PDFs del SAB (red) y el upload a S3, no el procesamiento local de sanitización. El SLA de 25 minutos se define en función de la latencia de red observada en el piloto.

### NFR-U02-01.3: Descarga de PDFs del SAB

| Atributo | Valor |
|---|---|
| Timeout por documento | 30 segundos (configurable: `SAB_DOWNLOAD_TIMEOUT_S`) |
| Tamaño máximo de PDF | 50 MB (configurable: `PDF_MAX_SIZE_MB`) |
| Modo de descarga para PDFs > 10 MB | Streaming (no acumular en memoria) |
| Conexiones HTTP concurrentes al SAB | 1 (secuencial por portafolio, sin paralelismo en piloto) |

**Rationale**: El procesamiento secuencial simplifica el manejo de errores y protege el SAB de sobrecarga durante el piloto. El paralelismo se considerará en producción si el throughput es insuficiente.

---

## NFR-U02-02 — Escalabilidad del Pipeline de Ingesta

### NFR-U02-02.1: Límites de Escala (Hereda BR-U02-01)

| Atributo | Piloto | Producción (target) |
|---|---|---|
| Portafolios en ingesta simultánea | 1 | 5 |
| Cotizaciones por portafolio | ≤ 50 | ≤ 200 |
| PDFs totales por portafolio | ≤ 200 | ≤ 500 |
| Tamaño máximo por PDF | 50 MB | 100 MB |

**Implicación de diseño**: En piloto, solo se procesa **1 portafolio en ingesta a la vez por defecto**. Si el Analista intenta iniciar un segundo portafolio mientras hay uno en `INGESTA`, el sistema lo acepta (crea el `Portafolio` en estado `PENDIENTE`) pero la BackgroundTask lo encola internamente.

### NFR-U02-02.2: Uso de Memoria durante la Ingesta

El pipeline debe ser cuidadoso con el uso de memoria al manejar PDFs grandes.

| Escenario | Política |
|---|---|
| PDF ≤ 10 MB | Carga completa en memoria para procesamiento |
| PDF > 10 MB | Descarga en streaming; procesamiento por chunks si aplica |
| Máxima memoria simultánea por BackgroundTask | ≤ 500 MB (estimado: 200 PDFs × 50 MB × factor de eficiencia) |

**Protección**: RT4 (tamaño máximo) se valida primero en el pipeline de sanitización, antes de intentar cargar el PDF completo en memoria para RT2/RT3.

---

## NFR-U02-03 — Disponibilidad y Resiliencia de la Integración SAB

### NFR-U02-03.1: Comportamiento cuando el SAB está no disponible

El SAB es un sistema externo sobre el que no tenemos control de disponibilidad.

| Escenario | Comportamiento del sistema |
|---|---|
| SAB no disponible al inicio del análisis | HTTP 503 al Analista; `Portafolio` no se crea (BR-U02-02) |
| SAB no disponible durante la BackgroundTask (descarga) | `DocumentoPDF` → `ERROR` (motivo: `sab_timeout`); pipeline continúa con los demás PDFs |
| Keycloak del SAB no disponible | BackgroundTask falla con `Portafolio` → `ERROR`; evento `INGESTA_ERROR` en WebSocket |
| Token Keycloak expirado durante descarga | Reintentar UNA vez con re-login; si falla → PDF → `ERROR` |

### NFR-U02-03.2: Resiliencia del Upload a S3

| Escenario | Comportamiento |
|---|---|
| Upload S3 falla (primer intento) | Reintento automático con backoff: 1s, 2s |
| 2 reintentos fallidos | `DocumentoPDF` → `ERROR` (motivo: `upload_s3_fallido`) |
| S3 no disponible globalmente | BackgroundTask falla; `Portafolio` → `ERROR` |

### NFR-U02-03.3: Idempotencia ante reinicios

Si la BackgroundTask es interrumpida (restart del servidor, crash), el `Portafolio` queda en estado `INGESTA`. El Analista puede:
- Ver el estado actual en el dashboard.
- Opcionalmente reintentar el análisis (en MVP: crear un nuevo `Portafolio` para el mismo proceso).

**MVP**: No hay recuperación automática de BackgroundTasks interrumpidas. Esta capacidad se considera para producción.

---

## NFR-U02-04 — Seguridad Específica de U-02

### NFR-U02-04.1: Protección de Credenciales del SAB

Las credenciales para acceder al SAB (Keycloak + BD) son de la más alta sensibilidad.

| Atributo | Política |
|---|---|
| Almacenamiento | Exclusivamente en variables de entorno / secretos del proveedor cloud. **Nunca en código ni en logs.** |
| `SAB_SERVICE_PASSWORD` en logs | ❌ Prohibido absolutamente. Si se registra por error: incidente de seguridad. |
| `access_token` y `refresh_token` | Solo en memoria durante la BackgroundTask. No se persisten en BD ni S3. |
| Rotación de credenciales | El service account de SAB debe tener contraseña con rotación trimestral (política del equipo de TI). |

### NFR-U02-04.2: Gestión del Token Keycloak

| Atributo | Valor |
|---|---|
| Renovación proactiva | Si `expires_at - now() < 60 segundos` |
| Uso de `refresh_token` | Prioritario sobre re-login completo |
| `refresh_token` expirado | Re-login completo con `grant_type=password` |
| Fallo en re-login | Portafolio → `ERROR`; log CRITICO con contexto (sin credenciales) |

### NFR-U02-04.3: Aislamiento del Contenido de PDFs (Zero Trust)

Todo PDF externo se trata como potencialmente adversarial hasta pasar el pipeline RT1-RT4.

| Regla | NFR |
|---|---|
| Nunca ejecutar contenido de un PDF | Los bytes del PDF se parsean solo para análisis estructural, no se ejecutan |
| Spotlighting obligatorio | Todo PDF `LIMPIO` lleva delimitadores dinámicos antes de enviarse a cualquier LLM |
| 0 EchoLeak | Meta de seguridad Gate G4: ningún output del LLM debe contener System Prompts ni datos de competidores |
| Logs del `SanitizadorZeroTrust` | Solo se registra la regla fallida y el tipo de amenaza. **Nunca** el fragmento completo del PDF (solo fragmento neutralizado ≤ 500 chars para el `AuditTrail` del CISO). |

### NFR-U02-04.4: Cifrado de PDFs en S3

| Atributo | Valor |
|---|---|
| Cifrado en reposo (S3) | AES-256 gestionado por el proveedor cloud (Server-Side Encryption) |
| Cifrado en tránsito (upload/download S3) | TLS 1.2 mínimo (forzado por la librería boto3/aiobotocore) |
| Acceso al bucket S3 | Solo el backend; sin URLs públicas. Acceso via IAM Role. |
| Retención de PDFs en S3 | Indefinida durante el piloto; política de retención post-piloto según CISO |

---

## NFR-U02-05 — Observabilidad del Pipeline de Ingesta

### NFR-U02-05.1: Logs de la BackgroundTask

La BackgroundTask de ingesta corre en background, por lo que los logs son la única fuente de observabilidad directa durante el procesamiento.

| Evento | Nivel de Log | Qué se registra |
|---|---|---|
| Inicio de pipeline | `INFO` | `portafolio_id`, `proceso_id`, `n_cotizaciones` |
| Inicio de descarga de PDF | `INFO` | `cotizacion_id`, `documento_id`, `nombre_archivo` |
| PDF descargado OK | `INFO` | `cotizacion_id`, `tamano_bytes`, `hash_sha256` |
| Descarga fallida | `WARNING` | `cotizacion_id`, `documento_id`, `motivo`, `http_status` |
| PDF sanitizado OK (LIMPIO) | `INFO` | `documento_pdf_id`, `reglas_pasadas: [RT1,RT2,RT3,RT4]` |
| PDF bloqueado (ADVERSARIAL) | `WARNING` | `documento_pdf_id`, `regla_fallida`, `tipo_amenaza` (sin fragmento del PDF) |
| PDF bloqueado (ERROR) | `WARNING` | `documento_pdf_id`, `regla_fallida`, `motivo` |
| Keycloak: renovación de token | `DEBUG` | Solo en modo development |
| Keycloak: fallo de autenticación | `CRITICAL` | `motivo` (sin credenciales) |
| Pipeline completado | `INFO` | `portafolio_id`, `pdfs_limpios`, `pdfs_adversariales`, `pdfs_error`, `duracion_s` |

**Regla heredada de NFR-04.2 de U-01**: Nunca se registran valores de negocio (nombre de proveedor, montos, contenido de cotizaciones) en logs.

### NFR-U02-05.2: Métricas de Observabilidad (Piloto)

Para el piloto, las métricas se obtienen de los logs y del `AuditTrail` (sin Prometheus/Grafana en piloto).

| Métrica | Fuente |
|---|---|
| Tiempo total de ingesta por portafolio | Log `Pipeline completado` (campo `duracion_s`) |
| Tasa de PDFs bloqueados (adversariales) | `AuditTrail` eventos `PDF_ADVERSARIAL_DETECTADO` |
| Tiempo promedio de descarga por PDF | Logs `PDF descargado OK` con timestamp |
| Errores de conectividad SAB | Logs `CRITICAL` y `WARNING` de Keycloak/HTTP |

---

## NFR-U02-06 — Cobertura de Tests Específica de U-02

| Módulo | Cobertura Mínima | Prioridad |
|---|---|---|
| **Global U-02** | **80%** | Obligatorio antes de Build |
| `SanitizadorZeroTrust` (RT1-RT4) | 95%+ | CRÍTICO — seguridad |
| Técnica Spotlighting | 90%+ | CRÍTICO — seguridad |
| `GestorTokenSAB` (Keycloak ROPC) | 90%+ | CRÍTICO — integración |
| `IngestaConector` (flujo principal) | 85% | ALTA |
| `RepositorioSABSQL` (queries) | 80% | ALTA |
| Manejo de errores de red (SAB/S3) | 85% | ALTA |
| Flujo BackgroundTask completo | 75% | MEDIA |

**Tests de seguridad obligatorios** (PBT — Property-Based Testing):
- **PBT-U02-01**: Para cualquier PDF con inyección de prompt en el texto, el `SanitizadorZeroTrust` debe retornar estado `ADVERSARIAL` (nunca `LIMPIO`).
- **PBT-U02-02**: Para cualquier PDF en estado `LIMPIO`, el `PDFSanitizado.contenido_bytes` siempre contiene delimitadores de Spotlighting con token dinámico.
- **PBT-U02-03**: Para cualquier credencial SAB, el `GestorTokenSAB` nunca registra el valor de `password` o `access_token` en ningún log.

---

## Resumen de NFRs por Categoría (U-02)

| Categoría | Requisito Clave |
|---|---|
| **Performance** | < 500ms dispatch de 202; ≥ 10 PDFs/min throughput; ≤ 25 min para 200 PDFs |
| **Escalabilidad** | 1 portafolio simultáneo en piloto; máx 200 PDFs × 50 MB; streaming para > 10 MB |
| **Disponibilidad SAB** | Fail-fast en inicio (503); fail-PDF en background (continúa pipeline) |
| **Seguridad** | 0 credenciales en logs; token Keycloak solo en memoria; AES-256 en S3; 0 EchoLeak |
| **Observabilidad** | Logs estructurados por etapa; métricas via AuditTrail; 80% cobertura mínima |
| **Tests de Seguridad** | 95% cobertura en SanitizadorZeroTrust; 3 PBTs obligatorios (Zero Trust + credenciales) |
