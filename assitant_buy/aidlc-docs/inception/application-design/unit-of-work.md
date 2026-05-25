# Unit of Work — Assistent Buy AI Agent

> **Versión**: 1.0  
> **Fase**: INCEPTION — Units Generation  
> **Fecha**: 2026-05-25  
> **Tipo de sistema**: Monolito Modular FastAPI (in-process)  

---

## Decisiones de Arquitectura Confirmadas

| Decisión | Elección |
|---|---|
| **Agrupación de unidades** | 7 unidades (U-01 a U-07) tal como se propusieron |
| **Frontend** | U-07 separado — proyecto Angular independiente que consume la API REST |
| **Secuencia de desarrollo** | Paralelo controlado — U-01 primero, luego U-02 y U-06 en paralelo, luego U-03 → U-04 → U-05, finalmente U-07 |
| **Organización del código** | Híbrido: `app/core/` para transversales + `app/modules/{dominio}/` para módulos de negocio |
| **Modelo de dominio** | Unificado — modelos compartidos en `app/core/models/` (Cotizacion, VariableExtraida, Discrepancia, etc.) |
| **Gestión de secrets** | `.env` con pydantic-settings (`BaseSettings`) |
| **Bounded contexts** | Modelo unificado — las unidades son etapas de un pipeline, no dominios independientes |

---

## Resumen de Unidades

| Unit | Nombre | Componentes | Stories MVP |
|---|---|---|---|
| **U-01** | Fundación (Auth + Datos + API + WebSocket) | C-08, C-10, C-13, C-14 | US-23 |
| **U-02** | Ingesta y Sanitización | C-01, C-02, C-12, C-16 | US-01, US-02, US-03 |
| **U-03** | Extracción, Cruce y Orquestación | C-03, C-04, C-09, C-11 | US-04, US-05, US-06, US-07, US-08 |
| **U-04** | Comunicación HITL | C-05 | US-09, US-10, US-11, US-12, US-13, US-14 |
| **U-05** | Reportes y Decisión | C-06 | US-15, US-16, US-17, US-18 |
| **U-06** | Administración y Seguridad | C-07 | US-19, US-20, US-21, US-22 |
| **U-07** | Frontend SPA Angular | C-15 | Todas las US (capa UI) |

---

## Estructura de Código — Monolito Modular (Backend)

```
assitant_buy/                          # Workspace root (APLICACIÓN aquí)
│
├── backend/                           # Proyecto FastAPI
│   ├── app/
│   │   ├── core/                      ← U-01 (Fundación transversal)
│   │   │   ├── models/                # SQLAlchemy ORM — modelos de dominio unificados
│   │   │   │   ├── portafolio.py      # Portafolio, EstadoPortafolio
│   │   │   │   ├── cotizacion.py      # Cotizacion, EstadoCotizacion
│   │   │   │   ├── pdf.py             # PDFDocumento, EstadoSanitizacion
│   │   │   │   ├── extraccion.py      # VariableExtraida, NivelConfianza
│   │   │   │   ├── discrepancia.py    # Discrepancia, SeveridadDiscrepancia
│   │   │   │   ├── aclaracion.py      # SolicitudAclaracion, EstadoAclaracion
│   │   │   │   ├── log_evento.py      # LogEvento, TipoEvento
│   │   │   │   ├── audit_trail.py     # AuditTrail (append-only)
│   │   │   │   └── sesion.py          # Sesion de usuario
│   │   │   ├── schemas/               # Pydantic schemas compartidos entre módulos
│   │   │   │   ├── pipeline.py        # ResultadoExtraccion, ResultadoCruce
│   │   │   │   └── websocket.py       # Formato JSON WebSocket {tipo, payload, timestamp}
│   │   │   ├── auth/                  # OAuth2/OIDC, JWT, roles, middleware de autorización
│   │   │   ├── config/                # pydantic BaseSettings, carga de .env
│   │   │   ├── database/              # SQLAlchemy engine, session factory, Alembic
│   │   │   ├── security/              # Zero Trust helpers, rate limiting, headers HTTP
│   │   │   └── logging/              # Logger estructurado con ID de correlación
│   │   │
│   │   ├── modules/
│   │   │   ├── ingesta/               ← U-02 (C-01, C-02, C-12, C-16)
│   │   │   │   ├── conector.py        # IngestaConector — lectura SAB
│   │   │   │   ├── sanitizador.py     # SanitizadorZeroTrust — pipeline + Spotlighting
│   │   │   │   ├── gateways/
│   │   │   │   │   ├── sab_repo.py    # RepositorioSAB (IRepositorioSAB)
│   │   │   │   │   └── document_ai.py # GatewayDocumentAI (IGatewayDocumentAI)
│   │   │   │   └── schemas.py         # Schemas internos de ingesta
│   │   │   │
│   │   │   ├── extraccion/            ← U-03 (C-03, C-04, C-09, C-11)
│   │   │   │   ├── motor_extraccion.py # MotorExtraccion — Document AI + Claude
│   │   │   │   ├── motor_cruce.py      # MotorCruce — comparación SAB vs PDF
│   │   │   │   ├── pipeline_service.py # ServicioPipeline — orquestador
│   │   │   │   ├── gateways/
│   │   │   │   │   └── llm_gateway.py  # GatewayLLM (IGatewayLLM)
│   │   │   │   └── schemas.py
│   │   │   │
│   │   │   ├── aclaraciones/          ← U-04 (C-05)
│   │   │   │   ├── gestor_aclaraciones.py # GestorAclaraciones — HITL completo
│   │   │   │   ├── smtp_client.py         # Cliente SMTP
│   │   │   │   ├── imap_poller.py         # Polling IMAP asyncio background task
│   │   │   │   └── schemas.py
│   │   │   │
│   │   │   ├── reportes/              ← U-05 (C-06)
│   │   │   │   ├── generador_reportes.py  # GeneradorReportes — síntesis + PDF
│   │   │   │   └── schemas.py
│   │   │   │
│   │   │   └── admin/                 ← U-06 (C-07)
│   │   │       ├── modulo_admin.py        # ModuloAdmin — logs + alertas + panel CISO
│   │   │       └── schemas.py
│   │   │
│   │   ├── api/                       ← C-10 APIRest (routers delegando a módulos)
│   │   │   ├── v1/
│   │   │   │   ├── router.py          # Router principal con prefijo /api/v1
│   │   │   │   ├── portafolios.py     # Endpoints de portafolios y análisis
│   │   │   │   ├── aclaraciones.py    # Endpoints de aclaraciones
│   │   │   │   ├── reportes.py        # Endpoints de reportes
│   │   │   │   ├── admin.py           # Endpoints de administración y CISO
│   │   │   │   └── websockets.py      # Endpoints WebSocket (C-14 GestorWebSocket)
│   │   │   └── deps.py                # Dependencias FastAPI (get_current_user, etc.)
│   │   │
│   │   └── main.py                    # FastAPI app factory, startup, middleware
│   │
│   ├── alembic/                       # Migraciones de base de datos
│   ├── tests/                         # Tests por módulo
│   │   ├── unit/
│   │   ├── integration/
│   │   └── pbt/                       # Property-Based Tests (Hypothesis)
│   ├── .env.example                   # Template de variables de entorno
│   ├── pyproject.toml                 # Dependencias Python
│   └── Dockerfile
│
└── frontend/                          ← U-07 (C-15 SPAAngular)
    ├── src/
    │   ├── app/
    │   │   ├── modules/
    │   │   │   ├── ingesta/           # IngestaModule — vistas E-01
    │   │   │   ├── analisis/          # AnalisisModule — vistas E-02
    │   │   │   ├── aclaraciones/      # AclaracionModule — vistas E-03
    │   │   │   ├── reportes/          # ReportesModule — vistas E-04
    │   │   │   └── admin/             # AdminModule — vistas E-05
    │   │   ├── core/                  # Auth PKCE, interceptores HTTP, guards
    │   │   └── shared/                # Componentes reutilizables
    │   └── environments/
    ├── angular.json
    └── package.json
```

---

## U-01 — Fundación (Auth + Datos + API + WebSocket)

**Propósito**: Establecer el esqueleto del sistema: autenticación, esquema de BD, endpoints base y WebSocket. Todo lo demás depende de esta unidad.

**Componentes incluidos**:
- `C-08` AutenticadorOAuth — OAuth2/OIDC, JWT, roles, MFA
- `C-10` APIRest — estructura FastAPI, middleware, CORS, rate limiting, seguridad
- `C-13` RepositorioLocal — esquema PostgreSQL completo + Alembic + modelos SQLAlchemy
- `C-14` GestorWebSocket — conexiones WebSocket, formato de mensajes

**Alcance de código**:
- `app/core/` completo (models, config, auth, database, security, logging)
- `app/api/` estructura base
- `app/main.py`
- `alembic/` con migración inicial

**Prerrequisito de**: U-02, U-03, U-04, U-05, U-06, U-07

**Criterios de finalización**:
- [ ] Todos los modelos SQLAlchemy definidos y migración inicial ejecutable
- [ ] Flujo OAuth2 funcional con Google/Microsoft (test con mock)
- [ ] FastAPI arranca con `/health`, `/api/v1` y documentación OpenAPI
- [ ] WebSocket endpoint básico funcional
- [ ] Security Baseline: CORS, headers HTTP, rate limiting configurados
- [ ] pydantic-settings cargando `.env` correctamente

---

## U-02 — Ingesta y Sanitización

**Propósito**: Conectar con SAB, leer cotizaciones y PDFs, y sanitizar cada documento con el pipeline Zero Trust antes de cualquier procesamiento posterior.

**Componentes incluidos**:
- `C-01` IngestaConector — lectura SAB (DB read-only + filesystem)
- `C-02` SanitizadorZeroTrust — pipeline RT1-RT4 + Spotlighting
- `C-12` RepositorioSAB — repository pattern para SAB
- `C-16` GatewayDocumentAI — abstracción Document AI cloud

**Alcance de código**:
- `app/modules/ingesta/` completo
- Integración con `app/core/models/` (PDFDocumento, EstadoSanitizacion)
- API endpoints de inicio de análisis y progreso

**Prerrequisito de**: U-03

**Criterios de finalización**:
- [ ] IngestaConector lee datos de SAB (con mock de BD SAB para tests)
- [ ] SanitizadorZeroTrust detecta RT1-RT4 en corpus de prueba D3
- [ ] GatewayDocumentAI implementa IGatewayDocumentAI con mock/stub
- [ ] Pipeline Zero Trust: `fail closed` si sanitización incompleta
- [ ] Logs de sanitización persistidos en `app/core/models/log_evento.py`
- [ ] **PBT**: propiedad "ningún PDF sin sanitizar llega al MotorExtraccion" (PBT-02)

---

## U-03 — Extracción, Cruce y Orquestación

**Propósito**: Núcleo del valor del sistema. Extrae variables de los PDFs sanitizados, cruza con datos SAB, clasifica discrepancias, y orquesta el pipeline completo.

**Componentes incluidos**:
- `C-03` MotorExtraccion — Document AI + Claude, confianza, referencias
- `C-04` MotorCruce — comparación SAB vs PDF, clasificación de discrepancias
- `C-09` ServicioPipeline — orquestador central del análisis
- `C-11` GatewayLLM — abstracción Anthropic Claude

**Alcance de código**:
- `app/modules/extraccion/` completo
- Integración con `app/core/schemas/` (ResultadoExtraccion, ResultadoCruce)
- API endpoints de variables, discrepancias y estado de análisis

**Prerrequisito de**: U-04, U-05

**Criterios de finalización**:
- [ ] MotorExtraccion extrae variables con confianza y referencias (test con corpus D1/D2)
- [ ] MotorCruce detecta discrepancias Alta/Media/Baja correctamente
- [ ] ServicioPipeline orquesta la secuencia completa sin bloquear el hilo principal
- [ ] Variables de confianza baja generan flag de escalamiento (nunca valores inventados)
- [ ] GatewayLLM implementa IGatewayLLM con manejo de errores y chunking
- [ ] **Gate G1**: precisión extracción > 96% (corpus D2)
- [ ] **Gate G2**: ASR < 2% (corpus D3)
- [ ] **Gate G3**: tasa de alucinación < 1%

---

## U-04 — Comunicación HITL

**Propósito**: Gestionar el ciclo completo de aclaración con proveedores: borradores, aprobación, SMTP, polling IMAP, correlación y reingesta.

**Componentes incluidos**:
- `C-05` GestorAclaraciones — HITL completo (borrador → aprobación → SMTP → IMAP → reingesta)

**Alcance de código**:
- `app/modules/aclaraciones/` completo
- `imap_poller.py` como asyncio background task
- Integración con GestorWebSocket para notificaciones
- API endpoints de aclaraciones

**Prerrequisito de**: U-05 (aclaraciones alimentan reportes)

**Criterios de finalización**:
- [ ] Generación de borradores a partir de ResultadoCruce
- [ ] Flujo aprobación/descarte con audit trail completo
- [ ] SMTP: envío con retry backoff exponencial (3 reintentos)
- [ ] IMAP polling asyncio configurable (default: 5 minutos)
- [ ] Correlación de respuestas por thread ID / ID en asunto
- [ ] Reingesta de PDFs adjuntos al pipeline de U-02/U-03
- [ ] Notificación WebSocket al Analista en < 2 minutos
- [ ] **Seguridad**: ningún envío sin aprobación explícita registrada

---

## U-05 — Reportes y Decisión

**Propósito**: Consolidar resultados del análisis y generar el reporte comparativo formal con trazabilidad total para el Comité de Contratación.

**Componentes incluidos**:
- `C-06` GeneradorReportes — síntesis por proveedor, reporte Comité, export PDF

**Alcance de código**:
- `app/modules/reportes/` completo
- API endpoints de reportes y vista solo-lectura para Comité
- Integración con librería PDF (WeasyPrint o equivalente)

**Prerrequisito de**: U-07 (el frontend consume el reporte)

**Criterios de finalización**:
- [ ] Síntesis comparativa de todos los proveedores generada
- [ ] Reporte Comité con: ranking, matriz cumplimiento, riesgos, trazabilidad
- [ ] Export PDF generado en ≤ 30 segundos
- [ ] 100% de datos en el reporte tienen referencia de fuente verificable
- [ ] Ningún término prohibido en ningún output del reporte
- [ ] Vista solo-lectura para rol Comité con control de acceso

---

## U-06 — Administración y Seguridad

**Propósito**: Proveer observabilidad operacional y de seguridad al Admin (Director) y al CISO. Desarrollado en paralelo con U-02 al ser independiente del pipeline.

**Componentes incluidos**:
- `C-07` ModuloAdmin — logs estructurados, alertas, panel CISO

**Alcance de código**:
- `app/modules/admin/` completo
- API endpoints de logs, alertas y panel CISO
- Integración con `app/core/models/log_evento.py` y `audit_trail.py`

**Desarrollable en paralelo con**: U-02

**Criterios de finalización**:
- [ ] Logs estructurados por operación con ID de correlación
- [ ] Logs de sanitización del SanitizadorZeroTrust persistidos y visibles por CISO
- [ ] Alertas de seguridad (IPI detectado) visibles en ≤ 2 minutos
- [ ] Panel Admin con filtros (fecha, proveedor, tipo de evento)
- [ ] Logs append-only (ningún endpoint permite edición o borrado)
- [ ] Retención de logs: mínimo 90 días (SECURITY-14)
- [ ] Control de roles: Admin ve logs operacionales, CISO ve logs de seguridad

---

## U-07 — Frontend SPA Angular

**Propósito**: Interfaz de usuario completa que consume la API REST y WebSocket del backend. Desarrollada como proyecto Angular independiente.

**Componentes incluidos**:
- `C-15` SPAAngular — vistas de todas las épicas

**Alcance de código**:
- `frontend/` proyecto Angular independiente
- 5 NgModules: IngestaModule, AnalisisModule, AclaracionModule, ReportesModule, AdminModule
- Auth PKCE, guards de roles, interceptores HTTP, WebSocket client

**Desarrollada después de**: Backend completo (U-01 a U-06) con API estable

**Criterios de finalización**:
- [ ] Flujo OAuth2 PKCE con Google/Microsoft funcional
- [ ] Tokens JWT en memoria (no localStorage — SECURITY-01)
- [ ] Todas las vistas de US-01 a US-23 implementadas
- [ ] WebSocket client para progreso en tiempo real
- [ ] Route guards por rol (Analista, Admin, CISO, Comité)
- [ ] Sin llamadas directas a BD (todo via API REST)
- [ ] Visor de PDF con navegación a fuente (RF-07.5)
