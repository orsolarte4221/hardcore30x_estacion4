# NFR Requirements — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — NFR Requirements  
> **Fecha**: 2026-05-26  
> **Componentes**: C-08 AutenticadorOAuth, C-10 APIRest, C-13 RepositorioLocal, C-14 GestorWebSocket

---

## NFR-01 — Escalabilidad

### NFR-01.1: Volumen de Portafolios y Cotizaciones

| Atributo | Piloto | Producción (target) |
|----------|--------|---------------------|
| Portafolios activos simultáneos | 10–20 | 50–100 |
| Cotizaciones por portafolio | Hasta 100 | Hasta 200 |
| PDFs por cotización | 1–10 | 1–15 |
| Total registros estimados en BD | ~50,000 | ~500,000 |

**Implicaciones de diseño:**
- Pool de conexiones PostgreSQL requerido desde v1: `min_size=5`, `max_size=20`
- Los endpoints de lista deben usar paginación siempre (ver BR-14)
- El pipeline de IA opera en background (async) — nunca bloquea requests HTTP

### NFR-01.2: Usuarios Concurrentes

| Atributo | Piloto | Producción (target) |
|----------|--------|---------------------|
| Usuarios concurrentes | 5–15 | 50–100 |
| Conexiones WebSocket simultáneas | 5–15 | 50–100 |
| Requests HTTP/segundo (pico) | ~10 | ~50 |

**Implicaciones de diseño:**
- Uvicorn con múltiples workers (`--workers 4`) desde el inicio para manejar concurrencia HTTP
- El `GestorWebSocket` debe usar un mapa thread-safe de conexiones activas
- Rate limiting de 100 req/min por usuario (BR-16) previene abuso en producción

---

## NFR-02 — Performance

### NFR-02.1: Tiempo de Respuesta API REST

| Tipo de Endpoint | SLA (p95) | SLA (p99) |
|-----------------|-----------|-----------|
| Endpoints GET (lectura simple) | < 1,000ms | < 2,000ms |
| Endpoints POST (escritura simple) | < 1,000ms | < 2,000ms |
| Endpoints que inician pipeline IA | < 500ms (solo dispatch) | < 1,000ms |
| Endpoint `/health` | < 100ms | < 200ms |

**Nota:** Los endpoints que inician análisis de IA solo despachan el trabajo al background; el progreso se notifica vía WebSocket. El tiempo de procesamiento de IA queda fuera del SLA HTTP.

### NFR-02.2: Performance del Flujo de Autenticación (OAuth2)

| Etapa | SLA |
|-------|-----|
| Inicio flujo (redirect a Google) | < 200ms |
| Procesamiento del callback de Google | < 500ms |
| Flujo completo (incluyendo Google) | < 5 segundos |

**Nota:** El tiempo de respuesta de Google está fuera del control del sistema. El SLA de 5 segundos es una experiencia de usuario objetivo — fallos de red de Google no se contabilizan en el SLA interno.

### NFR-02.3: Latencia WebSocket

| Tipo de Evento | Latencia Máxima (origen → cliente) |
|---------------|-------------------------------------|
| `PROGRESO_ANALISIS` | < 2,000ms |
| `PDF_PROCESADO` | < 2,000ms |
| `ANALISIS_COMPLETADO` | < 2,000ms |
| `ALERTA_SEGURIDAD` | < 2,000ms |
| `ERROR_PROCESAMIENTO` | < 2,000ms |
| `RESPUESTA_PROVEEDOR` | < 2,000ms |

**Nota:** WebSocket es "near real-time" en el contexto de este sistema — análisis de IA toma segundos/minutos, por lo que latencia de 2 segundos en notificaciones es imperceptible para el usuario.

---

## NFR-03 — Disponibilidad y Confiabilidad

### NFR-03.1: Uptime Requerido

| Métrica | Objetivo |
|---------|----------|
| Uptime mensual | ≥ 99% |
| Downtime máximo mensual | ≤ 7.2 horas |
| Ventanas de mantenimiento | Planificadas, comunicadas con 24h de antelación |
| RTO (Recovery Time Objective) | < 1 hora |
| RPO (Recovery Point Objective) | < 24 horas (dado backup diario) |

### NFR-03.2: Backup de PostgreSQL

| Atributo | Valor |
|----------|-------|
| Frecuencia | Diaria (automática, horario nocturno) |
| Retención de backups | 30 días |
| Tipo de backup | pg_dump completo (full dump) |
| Destino | Storage externo al servidor de aplicación |
| Prueba de restore | Mensual (manual) |

**Máxima pérdida de datos aceptable:** 24 horas (coincide con frecuencia de backup).

### NFR-03.3: Manejo de Fallos del Pipeline IA

| Fase | Comportamiento |
|------|---------------|
| Fallo en llamada a Claude/Document AI | Retry automático 3 veces con backoff exponencial (`1s → 2s → 4s`) |
| 3 reintentos fallidos | Cotización → estado `ERROR`; evento `ERROR_PROCESAMIENTO` vía WebSocket |
| Cotizaciones restantes del portafolio | Continúan procesándose (fallo no detiene el portafolio completo) |
| Portafolio con todas las cotizaciones en `ERROR` | Portafolio → estado `ERROR`; evento `ANALISIS_COMPLETADO` con resumen de errores |
| Reintento manual | El analista puede reintentar cotizaciones en `ERROR` desde la UI |

---

## NFR-04 — Seguridad

### NFR-04.1: Cifrado de Datos en Reposo

**Decisión:** Cifrado a nivel de disco (filesystem encryption del servidor). Sin cifrado adicional a nivel de columna en piloto.

| Componente | Cifrado |
|------------|---------|
| Volumen del servidor (BD) | Cifrado de disco obligatorio en servidor cloud |
| Archivos PDF almacenados | En ruta cifrada por filesystem encryption |
| Backups de PostgreSQL | Cifrados en storage externo (AES-256 por el proveedor cloud) |
| Datos en tránsito (HTTPS/WSS) | TLS 1.2 mínimo, TLS 1.3 preferido |
| Variables de entorno / secrets | Almacenados en `.env` con permisos restrictivos (`chmod 600`) |

**Revisión post-piloto:** Evaluar cifrado a nivel de columna (`pgcrypto`) para `email_proveedor` y valores financieros si se requiere cumplimiento de normativas en producción.

### NFR-04.2: Datos Sensibles en Logs

**Política:** Los logs del sistema solo contienen IDs (UUIDs) y metadatos. **Nunca** se almacenan valores de negocio (nombres de proveedores, montos, emails, contenido de cotizaciones) en los logs.

| Dato | Permitido en logs |
|------|------------------|
| UUID de portafolio/cotización | ✅ Sí |
| Nombre del proveedor | ❌ No |
| Email del proveedor | ❌ No |
| Montos y valores de cotización | ❌ No |
| Email del usuario | ❌ No (solo UUID del usuario) |
| Código de proceso SAB | ✅ Sí (identificador de negocio no sensible) |
| Tipo de error / stack trace | ✅ Sí (sin datos de payload) |

**Implementación:** El logger estructurado debe tener una función `sanitize_log_context()` que filtre los campos prohibidos antes de escribir al log.

### NFR-04.3: Retención del AuditTrail

| Atributo | Valor |
|----------|-------|
| Retención activa en tabla `audit_trail` | 1 año |
| Retención en tabla histórica `audit_trail_archive` | 5 años (mínimo) |
| Proceso de archivado | Automático mensual: registros > 1 año → `audit_trail_archive` |
| Acceso a datos archivados | Solo `ADMIN` y `CISO` vía consulta directa |

**Trigger de archivado:** Proceso programado que corre el primer día de cada mes; mueve registros con `created_at < NOW() - INTERVAL '1 year'` a la tabla de archivo.

---

## NFR-05 — Entorno de Despliegue

### NFR-05.1: Infraestructura Cloud

**Decisión:** Cloud managed (AWS/GCP/Azure) con instancias básicas.

| Componente | Especificación Mínima Piloto |
|------------|------------------------------|
| Instancia de aplicación (FastAPI) | 2 vCPU, 4GB RAM |
| Instancia PostgreSQL | Managed DB service (ej: Cloud SQL, RDS) — 2 vCPU, 4GB RAM |
| Storage para PDFs | Object storage (ej: Cloud Storage, S3) |
| Networking | HTTPS con certificado SSL/TLS gestionado |

**Modelo de despliegue:** Contenedor Docker (`Dockerfile`) corriendo en la instancia cloud. Docker Compose para orquestación local de desarrollo. Sin Kubernetes en piloto.

---

## NFR-06 — Observabilidad y Mantenibilidad

### NFR-06.1: Estrategia de Logging

**Decisión:** Logs de texto plano con nivel INFO/ERROR. Simple, legible, suficiente para piloto.

| Atributo | Valor |
|----------|-------|
| Formato | Texto plano: `[TIMESTAMP] [LEVEL] [MODULE] message` |
| Niveles en producción | `INFO` y `ERROR` (sin DEBUG en producción) |
| Niveles en desarrollo | `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| Correlation ID | Incluido en cada línea de log (UUID del request) |
| Destino | stdout (recolectado por el proveedor cloud) |
| Retención de logs | Gestionada por el proveedor cloud (mínimo 30 días) |

**Regla crítica:** Los logs siguen la política NFR-04.2 — nunca datos de negocio sensibles.

### NFR-06.2: Healthcheck y Monitoreo

**Decisión:** Solo endpoint `/health` con estado de BD.

```json
GET /health → 200 OK
{
  "status": "healthy",
  "database": "connected",
  "timestamp": "2026-05-26T01:00:00Z",
  "version": "1.0.0"
}
```

| Verificación | Incluida |
|-------------|----------|
| Disponibilidad del proceso | ✅ |
| Conexión a PostgreSQL | ✅ |
| Versión de la aplicación | ✅ |
| Métricas de performance | ❌ (post-piloto) |
| Estado de workers | ❌ (post-piloto) |

**Uso:** El proveedor cloud configura el healthcheck automático apuntando a `/health` con intervalo de 30 segundos y umbral de 3 fallos para reiniciar el contenedor.

### NFR-06.3: Cobertura de Tests

| Categoría | Cobertura Mínima | Prioridad |
|-----------|-----------------|-----------|
| **Global U-01** | **80%** | Obligatorio antes de Build |
| Flujos de autenticación OAuth2 | 90%+ | CRÍTICO — seguridad |
| Lógica de autorización (permisos) | 90%+ | CRÍTICO — seguridad |
| Repositorios (CRUD + queries) | 80% | ALTA |
| Endpoints API (happy path + errores) | 80% | ALTA |
| GestorWebSocket (conexión, eventos) | 70% | MEDIA |
| Middlewares (rate limit, auth) | 85% | ALTA |

**Herramientas de test:**
- Framework: `pytest` + `pytest-asyncio`
- Coverage: `pytest-cov`
- Mock de BD: `pytest-postgresql` (BD real en tests, no mock)
- Mock de Google OAuth: respuestas HTTP simuladas con `httpx` + `respx`

---

## Resumen de NFRs por Categoría

| Categoría | Requisito Clave |
|-----------|----------------|
| **Escalabilidad** | 10–20 portafolios piloto; 5–15 usuarios concurrentes |
| **Performance** | < 1s API REST p95; < 5s OAuth2 completo; < 2s WebSocket |
| **Disponibilidad** | 99% uptime mensual; backup diario; retry 3x en pipeline IA |
| **Seguridad** | Cifrado en disco; logs sin datos sensibles; AuditTrail 1 año activo + 5 años archivo |
| **Despliegue** | Cloud managed; Docker; instancia 2vCPU/4GB |
| **Observabilidad** | Logs texto plano con correlation ID; `/health` básico; 80% cobertura tests |
