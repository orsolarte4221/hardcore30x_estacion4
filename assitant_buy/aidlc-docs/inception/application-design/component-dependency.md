# Component Dependency — Assistent Buy AI Agent

> **Versión**: 1.0 (Application Design)  
> Documenta las relaciones de dependencia, comunicación y flujo de datos entre todos los componentes del sistema.

---

## Arquitectura en Capas

```
┌─────────────────────────────────────────────────────────────────────┐
│  🖥️  FRONTEND — SPA Angular (C-15)                                  │
│  NgModules: IngestaModule | AnalisisModule | AclaracionModule |      │
│             ReportesModule | AdminModule                             │
└───────────────────────┬─────────────────────────────────────────────┘
                        │ HTTP REST + WebSocket
┌───────────────────────▼─────────────────────────────────────────────┐
│  🔌  API LAYER — FastAPI REST (C-10) + GestorWebSocket (C-14)       │
│  Middleware: AutenticadorOAuth (C-08)                                │
└───────────────────────┬─────────────────────────────────────────────┘
                        │ in-process calls
┌───────────────────────▼─────────────────────────────────────────────┐
│  ⚙️  SERVICES LAYER                                                  │
│  ServicioPipeline (C-09)  |  GestorAclaraciones (C-05)              │
│  GeneradorReportes (C-06) |  ModuloAdmin (C-07)                     │
└──────┬──────────────────────────────┬──────────────────────────────┘
       │ in-process calls             │ in-process calls
┌──────▼─────────────────────────┐  ┌─▼────────────────────────────┐
│  🧠  CORE COMPONENTS           │  │  🔒  SECURITY MIDDLEWARE       │
│  IngestaConector (C-01)        │  │  SanitizadorZeroTrust (C-02)  │
│  MotorExtraccion (C-03)        │  │  (aplicado automáticamente    │
│  MotorCruce (C-04)             │  │   antes del MotorExtraccion)  │
└──────┬─────────────────────────┘  └──────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│  🔌  ADAPTERS & REPOSITORIES                                         │
│  GatewayLLM / IGatewayLLM (C-11)   GatewayDocumentAI / IDocAI (C-16)│
│  RepositorioSAB / ISAB (C-12)      RepositorioLocal (C-13)           │
└──────┬──────────────────────────────────────────────────────────────┘
       │ I/O externo
┌──────▼──────────────────────────────────────────────────────────────┐
│  🌐  SISTEMAS EXTERNOS                                               │
│  SAB DB (read-only)  |  SAB Filesystem (read-only)                  │
│  Anthropic Claude API  |  Document AI Cloud (GCP/AWS/Azure)         │
│  SMTP Server  |  IMAP Mailbox  |  OAuth Provider (Google/Microsoft) │
│  PostgreSQL (propio)                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Matriz de Dependencias

| Componente | Depende de | Tipo de dependencia |
|---|---|---|
| **C-15** SPAAngular | C-10 APIRest, C-14 GestorWebSocket | HTTP REST, WebSocket |
| **C-10** APIRest | C-08 AutenticadorOAuth, C-09 ServicioPipeline, C-05 GestorAclaraciones, C-06 GeneradorReportes, C-07 ModuloAdmin, C-14 GestorWebSocket | In-process call |
| **C-09** ServicioPipeline | C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-03 MotorExtraccion, C-04 MotorCruce, C-05 GestorAclaraciones, C-06 GeneradorReportes, C-07 ModuloAdmin, C-13 RepositorioLocal, C-14 GestorWebSocket | In-process call |
| **C-01** IngestaConector | C-12 RepositorioSAB, C-02 SanitizadorZeroTrust (middleware) | In-process call |
| **C-02** SanitizadorZeroTrust | C-07 ModuloAdmin (logs de seguridad) | In-process call |
| **C-03** MotorExtraccion | C-11 GatewayLLM, C-16 GatewayDocumentAI, C-07 ModuloAdmin (métricas) | In-process call |
| **C-04** MotorCruce | C-13 RepositorioLocal | In-process call |
| **C-05** GestorAclaraciones | C-13 RepositorioLocal, C-14 GestorWebSocket, C-09 ServicioPipeline (reingesta) | In-process call + asyncio task |
| **C-06** GeneradorReportes | C-13 RepositorioLocal | In-process call |
| **C-07** ModuloAdmin | C-13 RepositorioLocal | In-process call |
| **C-08** AutenticadorOAuth | C-13 RepositorioLocal, OAuth Provider externo | HTTP (OAuth provider) |
| **C-11** GatewayLLM | Anthropic Claude API | HTTP externo |
| **C-12** RepositorioSAB | SAB DB, SAB Filesystem | DB Connection (read-only) |
| **C-13** RepositorioLocal | PostgreSQL | DB Connection |
| **C-14** GestorWebSocket | C-08 AutenticadorOAuth (validación de token en handshake WS) | In-process call |
| **C-16** GatewayDocumentAI | Document AI Cloud (Google/AWS/Azure) | HTTP externo |

---

## Diagrama de Flujo de Datos — Pipeline Principal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     PIPELINE PRINCIPAL DE ANÁLISIS                          │
│                                                                             │
│  SAB DB ──────────────────────────────────────────────────────────────────┐ │
│  SAB FS ────────────────────────────────────────────────────────────────┐ │ │
│                                                                         │ │ │
│  C-12 RepositorioSAB ←── IRepositorioSAB ──────────────────────────────┘ │ │
│       │ CotizacionSAB[]                                                   │ │
│       ▼                                                                   │ │
│  C-01 IngestaConector ─────── recuperar_pdf() ─────────────────────────┘ │
│       │ PDF bytes (sin sanitizar)                                         │
│       ▼                                                                   │
│  ═══ MIDDLEWARE ═══════════════════════════════════════════════════════════│
│  C-02 SanitizadorZeroTrust                                                │
│       │ PDFSanitizado (con Spotlighting tokens)                           │
│  ════════════════════════════════════════════════════════════════════════ │
│       ▼                                                                   │
│  C-11 GatewayLLM ←─── IGatewayLLM ←────────────────────────────────────┐ │
│  Document AI ───────────────────────────────────────────────────────────┐│ │
│                                                                          ││ │
│  C-03 MotorExtraccion ──────── usa GatewayLLM + DocAI ──────────────────┘│ │
│       │ ResultadoExtraccion                                               │ │
│       ▼                                                                   │ │
│  C-04 MotorCruce ───────────────────────────────────────────────────────┘ │
│       │ ResultadoCruce (discrepancias + severidades)                       │
│       ▼                                                                    │
│  C-05 GestorAclaraciones                                                   │
│       │ Borrador[] (para revisión del Analista)                            │
│       ▼                                                                    │
│  C-06 GeneradorReportes                                                    │
│       │ SintesisProveedor, ReporteComparativo                              │
│       ▼                                                                    │
│  C-13 RepositorioLocal → PostgreSQL                                        │
└─────────────────────────────────────────────────────────────────────────────┘

Estado y notificaciones (paralelo al pipeline):
C-09 ServicioPipeline ──── publicar_progreso() ────▶ C-14 GestorWebSocket ──▶ C-15 SPAAngular
```

---

## Diagrama de Flujo de Datos — Ciclo HITL

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CICLO HITL COMPLETO                                 │
│                                                                             │
│  C-05 GestorAclaraciones                                                   │
│       │ genera Borrador[]                                                   │
│       ▼                                                                     │
│  C-13 RepositorioLocal (estado: Pendiente)                                 │
│       │                                                                     │
│       ▼                                                                     │
│  C-10 APIRest ──── GET /aclaraciones ────▶ C-15 SPAAngular (Analista ve)   │
│                                                                             │
│  Analista aprueba ──── POST /aclaraciones/{id}/aprobar                     │
│       │                                                                     │
│       ▼                                                                     │
│  C-05 GestorAclaraciones                                                   │
│       ├── C-13 RepositorioLocal (audit trail: aprobado por, timestamp)     │
│       └── SMTP Server ──── email ──────────────────────────────────────▶   │
│                                                  Proveedor (externo)       │
│                                                       │ responde           │
│                                                       ▼                    │
│  IMAP Mailbox ◀─────────────────────────────── email de respuesta          │
│       │                                                                     │
│       ▼ (asyncio background task, cada 5 min)                              │
│  C-05 GestorAclaraciones.poll_imap_iteration()                             │
│       ├── correlacionar_respuesta() → Aclaracion                           │
│       ├── PDFs adjuntos → C-09 ServicioPipeline.reingestar()               │
│       ├── C-14 GestorWebSocket.publicar_notificacion() → C-15 SPAAngular  │
│       └── C-13 RepositorioLocal (audit trail: email recibido)              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Comunicación WebSocket — Tipos de Mensajes

| Tipo de mensaje | Emisor | Receptor | Cuando |
|---|---|---|---|
| `progreso_cotizacion` | ServicioPipeline | SPAAngular (Analista) | Durante el análisis, por cada cotización procesada |
| `analisis_completo` | ServicioPipeline | SPAAngular (Analista) | Al finalizar el pipeline de un portafolio |
| `pdf_no_sanitizable` | ServicioPipeline | SPAAngular (Analista/CISO) | Cuando el SanitizadorZeroTrust falla |
| `respuesta_proveedor` | GestorAclaraciones | SPAAngular (Analista) | Cuando llega un email de respuesta via IMAP |
| `sintesis_actualizada` | ServicioPipeline | SPAAngular (Analista) | Cuando se completa la reingesta de una respuesta |

---

## Patrones de Comunicación

### In-Process (mismo proceso FastAPI)
**Usado por**: Todos los componentes del backend entre sí  
**Justificación**: MVP con volumen de 50 cotizaciones/200 PDFs; simplicidad operacional; estado compartido en memoria para WebSocket.

### WebSocket
**Usado por**: SPAAngular ↔ GestorWebSocket  
**Justificación**: Actualizaciones de progreso en tiempo real; el polling HTTP generaría latencia innecesaria y overhead de requests.

### HTTP REST
**Usado por**: SPAAngular → APIRest; AutenticadorOAuth → OAuth Provider; MotorExtraccion → Document AI y Claude API  
**Justificación**: Estándar para APIs externas y comunicación cliente-servidor.

### Asyncio Background Task (in-process)
**Usado por**: GestorAclaraciones (IMAP polling)  
**Justificación**: El polling IMAP corre en el event loop de FastAPI; comparte estado y conexiones con el proceso principal sin requerir un worker externo (suficiente para el MVP).

### Repository Pattern
**Usado por**: RepositorioSAB, RepositorioLocal  
**Justificación**: Desacopla la lógica de negocio del motor de persistencia; facilita mocking en tests (PBT-02); permite cambio de fuente de datos sin modificar los componentes de negocio.

---

## Restricciones de Dependencia (Never Have)

| Restricción | Razón |
|---|---|
| ❌ MotorExtraccion NO depende directamente de Anthropic SDK | Debe pasar siempre por `IGatewayLLM` para permitir swap de LLM |
| ❌ MotorExtraccion NO depende directamente de SDKs de Document AI | Debe pasar siempre por `IGatewayDocumentAI` para permitir swap de proveedor |
| ❌ IngestaConector NO escribe en SAB | El usuario de BD de SAB es read-only a nivel de BD |
| ❌ Ningún componente accede a PDFs sin pasar por SanitizadorZeroTrust | Zero Trust — fail closed |
| ❌ GestorAclaraciones NO envía emails sin registro de aprobación en RepositorioLocal | Principio HITL — nunca sin aprobación humana |
| ❌ GeneradorReportes NO genera datos sin referencia de fuente | Trazabilidad obligatoria del PRD Principio 4 |
| ❌ RepositorioLocal NO tiene métodos UPDATE/DELETE en tablas de audit trail | Inmutabilidad del audit trail |
