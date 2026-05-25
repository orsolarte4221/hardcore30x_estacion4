# Unit of Work — Dependency Matrix

> **Versión**: 1.0  
> **Fase**: INCEPTION — Units Generation  
> **Fecha**: 2026-05-25  

---

## Secuencia de Desarrollo (Paralelo Controlado)

```
Semana/Sprint 1:
  U-01 Fundación (BLOQUEANTE — todos los demás dependen de este)

Semana/Sprint 2 (en paralelo):
  U-02 Ingesta y Sanitización
  U-06 Administración y Seguridad  ← independiente, puede ir en paralelo con U-02

Semana/Sprint 3:
  U-03 Extracción, Cruce y Orquestación  ← necesita U-02 completo

Semana/Sprint 4:
  U-04 Comunicación HITL              ← necesita U-03 (ResultadoCruce)

Semana/Sprint 5:
  U-05 Reportes y Decisión            ← necesita U-03 y U-04 (datos + aclaraciones)

Semana/Sprint 6:
  U-07 Frontend SPA Angular           ← necesita API REST completa (U-01 a U-06)
```

---

## Matriz de Dependencias

| | **U-01** | **U-02** | **U-03** | **U-04** | **U-05** | **U-06** | **U-07** |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **U-01** | — | provee | provee | provee | provee | provee | provee |
| **U-02** | necesita | — | provee | — | — | — | — |
| **U-03** | necesita | necesita | — | provee | provee | — | — |
| **U-04** | necesita | necesita* | necesita | — | provee | — | — |
| **U-05** | necesita | — | necesita | necesita | — | — | — |
| **U-06** | necesita | — | — | — | — | — | — |
| **U-07** | necesita | — | — | — | — | — | — |

> `*` U-04 necesita U-02 para la reingesta de PDFs adjuntos de respuestas de proveedor.

---

## Dependencias Detalladas por Unidad

### U-01 → (es base de todo)

| Provee a | Qué provee |
|---|---|
| U-02, U-03, U-04, U-05, U-06 | Modelos SQLAlchemy compartidos (`app/core/models/`) |
| U-02, U-03, U-04, U-05, U-06 | Session de BD, configuración, logging estructurado |
| U-02, U-03, U-04, U-05, U-06 | Middleware OAuth2 (decoradores de autorización) |
| U-03, U-04 | GestorWebSocket (endpoint + publicación de eventos) |
| U-07 | API REST con OpenAPI spec y endpoints de autenticación |

---

### U-02 → depende de U-01

| Necesita de | Qué necesita |
|---|---|
| U-01 | `app/core/models/pdf.py` (PDFDocumento, EstadoSanitizacion) |
| U-01 | `app/core/models/cotizacion.py` (Cotizacion) |
| U-01 | `app/core/security/` (integración con pipeline Zero Trust) |
| U-01 | Session de BD para persistir logs de sanitización |
| U-01 | `ModuloAdmin` vía `app/core/logging/` para eventos de seguridad |

| Provee a | Qué provee |
|---|---|
| U-03 | `PDFSanitizado` (bytes + metadata de sanitización + tokens Spotlighting) |
| U-03 | `TextoExtraido` via GatewayDocumentAI |
| U-06 | Eventos de seguridad (elementos adversariales detectados) |

**Interfaz de salida**:
```python
# app/modules/ingesta/schemas.py
class PDFSanitizado:
    contenido: bytes
    estado: EstadoSanitizacion      # LIMPIO | NEUTRALIZADO | NO_SANITIZABLE
    tokens_spotlighting: list[str]
    metadata_sanitizacion: dict     # tipo_ataque, fragmento_neutralizado (si aplica)
    cotizacion_id: UUID
    nombre_archivo: str
```

---

### U-03 → depende de U-01, U-02

| Necesita de | Qué necesita |
|---|---|
| U-01 | `app/core/schemas/pipeline.py` (ResultadoExtraccion, ResultadoCruce) |
| U-01 | `app/core/models/extraccion.py` y `discrepancia.py` |
| U-01 | GestorWebSocket para publicar progreso del análisis |
| U-02 | `PDFSanitizado` como entrada al MotorExtraccion |
| U-02 | `IRepositorioSAB` para datos de SAB (vía RepositorioSAB de U-02) |

| Provee a | Qué provee |
|---|---|
| U-04 | `ResultadoCruce` — lista de discrepancias que requieren aclaración |
| U-05 | `ResultadoExtraccion` + `ResultadoCruce` — base del reporte |
| U-06 | Métricas de uso LLM (tokens, latencia) vía GatewayLLM → ModuloAdmin |

**Interfaces de salida**:
```python
# app/core/schemas/pipeline.py
class VariableExtraida:
    nombre: str
    valor: str | None                # None si no encontrado (NUNCA inventado)
    confianza: NivelConfianza        # ALTA | MEDIA | BAJA
    referencia_fuente: str           # "PDF: nombre.pdf, pág. 3, §2.1"
    requiere_revision: bool          # True si confianza < umbral

class ResultadoExtraccion:
    cotizacion_id: UUID
    variables: list[VariableExtraida]
    timestamp: datetime

class Discrepancia:
    campo: str
    valor_sab: str | None
    valor_pdf: str | None
    severidad: SeveridadDiscrepancia # ALTA | MEDIA | BAJA
    referencia_fuente: str
    requiere_aclaracion: bool

class ResultadoCruce:
    cotizacion_id: UUID
    discrepancias: list[Discrepancia]
    timestamp: datetime
```

---

### U-04 → depende de U-01, U-02, U-03

| Necesita de | Qué necesita |
|---|---|
| U-01 | `app/core/models/aclaracion.py` (SolicitudAclaracion, EstadoAclaracion) |
| U-01 | GestorWebSocket para notificar al Analista respuestas entrantes |
| U-01 | Configuración SMTP/IMAP desde pydantic-settings |
| U-02 | Pipeline de sanitización (reingesta de PDFs adjuntos de respuestas) |
| U-03 | `ResultadoCruce` como fuente de los borradores de aclaración |
| U-03 | `ServicioPipeline` para disparar reingesta parcial con PDFs de respuesta |

| Provee a | Qué provee |
|---|---|
| U-05 | Aclaraciones respondidas e incorporadas al análisis (enriquece el reporte) |

---

### U-05 → depende de U-01, U-03, U-04

| Necesita de | Qué necesita |
|---|---|
| U-01 | `app/core/models/` completo para leer datos del análisis desde la BD |
| U-03 | `ResultadoExtraccion` y `ResultadoCruce` persistidos |
| U-04 | Aclaraciones respondidas incorporadas al estado de análisis |

---

### U-06 → depende de U-01 (solo)

> Puede desarrollarse en paralelo con U-02 al no depender del pipeline de procesamiento.

| Necesita de | Qué necesita |
|---|---|
| U-01 | `app/core/models/log_evento.py` y `audit_trail.py` |
| U-01 | `app/core/auth/` para control de roles (Admin vs CISO) |

---

### U-07 → depende de U-01 a U-06

| Necesita de | Qué necesita |
|---|---|
| U-01 | Endpoints OAuth2, OpenAPI spec completa, WebSocket endpoint |
| U-02 | Endpoints de inicio de análisis y progreso de ingesta |
| U-03 | Endpoints de variables, discrepancias y estado de análisis |
| U-04 | Endpoints de borradores, historial de aclaraciones |
| U-05 | Endpoints de reportes, síntesis, export PDF, vista Comité |
| U-06 | Endpoints de logs, alertas, panel CISO |

---

## Dependencias de Componentes Transversales

| Componente | Usado por | Unidad |
|---|---|---|
| `RepositorioLocal (C-13)` | C-05, C-06, C-07, C-08, C-09 | U-01 (definido), todos lo usan |
| `GestorWebSocket (C-14)` | C-05, C-09 | U-01 (definido), U-03 y U-04 lo usan |
| `AutenticadorOAuth (C-08)` | C-07, C-10, C-14 | U-01 (definido), todos los endpoints lo usan |
| `ModuloAdmin (C-07)` | C-02, C-11, C-16 | U-06 (definido), U-02 y U-03 envían eventos |

---

## Riesgos de Integración

| Riesgo | Entre unidades | Mitigación |
|---|---|---|
| Cambio en schemas de pipeline | U-03 ↔ U-04, U-05 | Schemas definidos en `app/core/schemas/pipeline.py` (U-01) — cambio centralizado |
| Latencia de GatewayLLM | U-03 → U-07 (UX) | Progreso vía WebSocket; análisis asíncrono; GestorWebSocket en U-01 |
| Correlación IMAP fallida | U-04 → U-03 | Thread ID en asunto del email como identificador primario; logs de no-correlación |
| API no estable para U-07 | U-01→U-06 → U-07 | U-07 se desarrolla después de API estable; usar OpenAPI spec como contrato |
| Sanitización bloqueante | U-02 → U-03 | `fail closed` por diseño; PDFs no sanitizables no entran al pipeline (Gate G2) |
