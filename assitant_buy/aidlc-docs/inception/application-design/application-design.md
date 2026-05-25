# Application Design — Assistent Buy AI Agent

> **Versión**: 1.0 (MVP)  
> **Fecha**: 2026-05-24  
> **Fase**: INCEPTION — Application Design  
> **Tipo de Proyecto**: Greenfield  
> Este documento consolida los cuatro artefactos de Application Design en una referencia única.

---

## Resumen Ejecutivo

**Assistent Buy AI Agent** es un sistema de análisis agéntico de cotizaciones de compras construido sobre:

| Capa | Tecnología | Componentes |
|---|---|---|
| **Frontend** | Angular SPA (NgModules por épica) | C-15 SPAAngular |
| **API** | FastAPI (Python 3.12+) | C-10 APIRest, C-14 GestorWebSocket |
| **Seguridad transversal** | Middleware FastAPI | C-02 SanitizadorZeroTrust, C-08 AutenticadorOAuth |
| **Servicios** | Python in-process | C-09 ServicioPipeline, C-05 GestorAclaraciones, C-06 GeneradorReportes, C-07 ModuloAdmin |
| **Componentes core** | Python in-process | C-01 IngestaConector, C-03 MotorExtraccion, C-04 MotorCruce |
| **Adaptadores** | Python (interfaces) | C-11 GatewayLLM, C-16 GatewayDocumentAI, C-12 RepositorioSAB |
| **Persistencia** | PostgreSQL + SQLAlchemy | C-13 RepositorioLocal |
| **Comunicación RT** | WebSocket (asyncio) | C-14 GestorWebSocket |

**Total**: 16 componentes, 5 servicios, 3 patrones de comunicación (in-process, WebSocket, HTTP)

---

## Decisiones Arquitectónicas Clave

| Decisión | Elección | Rationale |
|---|---|---|
| Granularidad del motor IA | **Separados**: MotorExtraccion + MotorCruce | Responsabilidades distintas (IA vs. lógica de negocio); más fácil de testear con PBT |
| Pipeline de sanitización | **Middleware automático** | Zero Trust — ningún PDF llega al MotorExtraccion sin sanitizar; fail closed |
| Estado de progreso | **En memoria + WebSocket** | Baja latencia para actualizaciones en tiempo real al frontend |
| Comunicación entre componentes | **In-process** (mismo proceso FastAPI) | MVP con volumen controlado; sin overhead de serialización; WebSocket compartido |
| Polling IMAP | **asyncio background task** | In-process, comparte estado y memoria; no requiere worker externo en MVP |
| Abstracción LLM | **Interfaz `IGatewayLLM`** | Permite swap de LLM sin cambiar MotorExtraccion; facilita mocking en tests |
| Abstracción Document AI | **Interfaz `IGatewayDocumentAI`** | Permite swap de proveedor OCR (Google/AWS/Azure); facilita mocking en PBT |
| Acceso a SAB | **Repository pattern** | Desacopla lógica de negocio de la BD de SAB; facilita tests y cambio futuro |
| Frontend | **NgModules por épica** | Alineado con los journeys del usuario; lazy loading natural por funcionalidad |

---

## Componentes (Resumen)

> Documento completo: [`components.md`](./components.md)

| ID | Componente | Capa | RF cubiertos |
|---|---|---|---|
| C-01 | `IngestaConector` | Backend Core | RF-01.1 a RF-01.5 |
| C-02 | `SanitizadorZeroTrust` | Middleware | RF-02.1 a RF-02.6 |
| C-03 | `MotorExtraccion` | Backend Core + IA | RF-03.1, RF-03.3, RF-03.4, RF-03.6, RF-03.7, RF-03.8 |
| C-04 | `MotorCruce` | Backend Core | RF-03.2, RF-03.3, RF-03.5 |
| C-05 | `GestorAclaraciones` | Servicio HITL | RF-04.1 a RF-04.8, RF-01.6 a RF-01.8 |
| C-06 | `GeneradorReportes` | Backend Core | RF-05.1 a RF-05.5 |
| C-07 | `ModuloAdmin` | Soporte | RF-06.1 a RF-06.3 |
| C-08 | `AutenticadorOAuth` | Transversal | RF-07.2, RF-07.3, SECURITY-12 |
| C-09 | `ServicioPipeline` | Orquestador | RF-01 a RF-04 (coordinación) |
| C-10 | `APIRest` | Presentación API | RF-07.1 a RF-07.5, SECURITY-04, 05, 08, 11 |
| C-11 | `GatewayLLM` | Adaptador | Motor de extracción (LLM) |
| C-12 | `RepositorioSAB` | Repositorio externo | RF-01.1, RF-01.2 |
| C-13 | `RepositorioLocal` | Repositorio interno | Persistencia de todos los módulos |
| C-14 | `GestorWebSocket` | Notificación RT | RF-07.4 (progreso en tiempo real) |
| C-15 | `SPAAngular` | Frontend | Todas las US (UI) |
| C-16 | `GatewayDocumentAI` | Adaptador | Motor de extracción (OCR/Document AI) |

---

## Flujos Principales de Datos

> Documentos completos: [`services.md`](./services.md) · [`component-dependency.md`](./component-dependency.md)

### Pipeline de Análisis

```
Analista activa análisis
    → ServicioPipeline
    → IngestaConector (SAB DB + Filesystem)
    → [SanitizadorZeroTrust middleware]
    → MotorExtraccion (Document AI + Claude via IGatewayLLM)
    → MotorCruce (discrepancias + severidades)
    → GestorAclaraciones (borradores HITL)
    → GeneradorReportes (síntesis por proveedor)
    + GestorWebSocket (progreso en tiempo real → SPAAngular)
```

### Ciclo HITL Completo

```
MotorCruce detecta discrepancias
    → GestorAclaraciones.generar_borrador()
    → Analista aprueba borrador
    → SMTP saliente (remitente: buzon monitorizado)
    → Proveedor responde
    → IMAP polling (asyncio task, cada 5 min)
    → Correlación con solicitud original
    → ServicioPipeline.reingestar_respuesta_proveedor()
    → [SanitizadorZeroTrust] → MotorExtraccion → MotorCruce
    → Notificación al Analista via WebSocket
```

---

## Interfaz de Componentes Clave

> Documento completo: [`component-methods.md`](./component-methods.md)

```python
# Contrato del Motor de Extracción
ResultadoExtraccion:
  variables: list[VariableExtraida(nombre, valor|None, confianza, referencia_fuente, requiere_revision)]
  estado: Exitoso | Parcial | Fallido
  metadata: {documento, paginas_procesadas, duracion_ms, tokens_usados}

# Contrato del Motor de Cruce
ResultadoCruce:
  discrepancias: list[Discrepancia(campo, valor_sab, valor_pdf, severidad, referencia_fuente, requiere_aclaracion)]
  cobertura: float  # % campos SAB encontrados en PDF
  variables_sin_fuente: list[str]

# Abstracción del LLM
class IGatewayLLM(Protocol):
    async def completar(prompt_sistema, prompt_usuario, schema_respuesta) → RespuestaLLM

# Abstracción del acceso a SAB (read-only enforced)
class IRepositorioSAB(Protocol):
    async def listar_cotizaciones(proceso_id) → list[CotizacionSAB]
    async def obtener_items_cotizacion(cotizacion_id) → list[ItemSAB]
    async def obtener_rutas_pdf(cotizacion_id) → list[RutaPDF]
    # NUNCA métodos write
```

---

## Endpoints REST (Resumen)

```
AUTH:        GET /auth/login/{proveedor}  |  GET /auth/callback  |  POST /auth/logout
PIPELINE:    GET /procesos  |  POST /procesos/{id}/analisis  |  GET .../estado  |  WS .../ws
ANÁLISIS:    GET /cotizaciones/{id}/variables  |  .../discrepancias  |  .../estado
ACLAR.:      GET /aclaraciones  |  POST .../aprobar  |  POST .../descartar  |  GET .../historial
REPORTES:    GET /reportes/{id}/sintesis  |  POST .../generar  |  GET .../pdf
ADMIN:       GET /admin/logs  |  GET .../alertas  |  POST .../revisar  |  GET .../sanitizacion/{id}
PDF VIEWER:  GET /pdfs/{id}/fragmento?pagina=N&seccion=X  |  GET /pdfs/{id}/metadata
```

---

## Restricciones de Diseño Obligatorias

| Restricción | Impacto |
|---|---|
| ❌ Ningún PDF llega al MotorExtraccion sin pasar por SanitizadorZeroTrust | Zero Trust — enforced por middleware |
| ❌ IngestaConector nunca escribe en SAB | Read-only a nivel de usuario de BD |
| ❌ GestorAclaraciones nunca envía email sin registro de aprobación | Principio HITL del PRD |
| ❌ GeneradorReportes nunca genera datos sin referencia de fuente | Trazabilidad obligatoria |
| ❌ RepositorioLocal no tiene UPDATE/DELETE en tablas de audit trail | Inmutabilidad del audit trail |
| ❌ MotorExtraccion no usa Anthropic SDK directamente | Siempre vía IGatewayLLM |
| ❌ MotorExtraccion no usa SDKs de Document AI directamente | Siempre vía IGatewayDocumentAI |

---

## Trazabilidad a Requerimientos

| RF | Componente(s) responsables |
|---|---|
| RF-01.x (Ingesta) | C-01, C-09, C-12 |
| RF-02.x (Sanitización) | C-02 |
| RF-03.x (Extracción y cruce) | C-03, C-04, C-11, C-16 |
| RF-04.x (HITL y Aclaraciones) | C-05 |
| RF-05.x (Reportes) | C-06 |
| RF-06.x (Admin) | C-07 |
| RF-07.x (Autenticación y roles) | C-08, C-10 |
| RNF-01 (Rendimiento) | C-09 (asyncio), C-03 (timeout control) |
| RNF-02 (Seguridad) | C-02, C-08, C-10 |
| Security Baseline | C-02 (SECURITY-02,03), C-08 (SECURITY-12), C-10 (SECURITY-04,05,08) |
| PBT (Hypothesis) | C-03 y C-04 (motores — funciones puras) |

---

## Próximos Pasos

Con Application Design completo, la siguiente fase es **Units Planning**, donde se definirán:
1. El esquema de BD de `RepositorioLocal` (tablas, índices, relaciones)
2. Los modelos de datos Python (dataclasses / Pydantic)
3. La estructura de directorios del proyecto (backend FastAPI + frontend Angular)
4. Los tests unitarios y de PBT para los motores de extracción y cruce
