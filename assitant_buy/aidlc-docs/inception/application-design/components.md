# Componentes — Assistent Buy AI Agent

> **Versión**: 1.0 (Application Design)  
> **Fase**: INCEPTION — Application Design  

---

## Resumen de Componentes

| ID | Componente | Capa | Tipo | Descripción |
|---|---|---|---|---|
| C-01 | `IngestaConector` | Backend | Conector externo | Lee cotizaciones y PDFs desde SAB |
| C-02 | `SanitizadorZeroTrust` | Backend | Middleware/Componente | Pipeline de sanitización + Spotlighting para todos los PDFs |
| C-03 | `MotorExtraccion` | Backend | Componente IA | Extrae variables de PDFs sanitizados via Document AI + Claude |
| C-04 | `MotorCruce` | Backend | Componente de lógica | Cruza datos SAB vs. variables extraídas; clasifica discrepancias |
| C-05 | `GestorAclaraciones` | Backend | Componente HITL | Genera borradores, gestiona aprobaciones, envío SMTP, polling IMAP |
| C-06 | `GeneradorReportes` | Backend | Componente de salida | Síntesis comparativa, reporte de Comité, export PDF |
| C-07 | `ModuloAdmin` | Backend | Componente de soporte | Logs estructurados, alertas, panel CISO |
| C-08 | `AutenticadorOAuth` | Backend | Componente transversal | OAuth2/OIDC, validación de tokens, gestión de roles y sesiones |
| C-09 | `ServicioPipeline` | Backend | Servicio orquestador | Coordina la secuencia completa del análisis; gestiona estado en memoria |
| C-10 | `APIRest` | Backend | Capa de presentación API | Endpoints FastAPI que exponen todos los servicios al frontend |
| C-11 | `GatewayLLM` | Backend | Adaptador externo | Abstracción de la interacción con Anthropic Claude (interfaz `IGatewayLLM`) |
| C-12 | `RepositorioSAB` | Backend | Repositorio externo | Encapsula todas las queries read-only a la BD de SAB |
| C-13 | `RepositorioLocal` | Backend | Repositorio interno | Persistencia de análisis, aclaraciones, audit trail en PostgreSQL |
| C-14 | `GestorWebSocket` | Backend | Componente de notificación | Gestiona las conexiones WebSocket y publica actualizaciones de progreso |
| C-15 | `SPAAngular` | Frontend | SPA | Interfaz de usuario Angular organizada en NgModules por épica |
| C-16 | `GatewayDocumentAI` | Backend | Adaptador externo | Abstracción de la interacción con Document AI cloud (interfaz `IGatewayDocumentAI`) |

---

## C-01 — IngestaConector

**Propósito**: Conectar con el ecosistema SAB para extraer datos estructurados y documentos PDF de un portafolio de cotizaciones.

**Responsabilidades**:
- Conectarse a la base de datos de SAB en modo read-only (SQL/NoSQL)
- Leer todos los datos estructurados de cotizaciones del proceso activado (ítems, precios, proveedores, fechas)
- Acceder al sistema de archivos del servidor SAB y recuperar los PDFs adjuntos por ruta
- Validar la integridad de cada PDF recuperado (legible, no corrupto, tamaño válido)
- Reportar el estado de ingesta por cotización (Completa / Parcial / No procesable)
- Delegar automáticamente cada PDF recuperado al `SanitizadorZeroTrust` vía middleware

**Interfaces externas**: SAB DB (read-only), SAB filesystem (read-only)  
**Dependencias**: `RepositorioSAB`, `SanitizadorZeroTrust` (middleware)  
**RF cubiertos**: RF-01.1, RF-01.2, RF-01.3, RF-01.4, RF-01.5

---

## C-02 — SanitizadorZeroTrust

**Propósito**: Actuar como middleware de seguridad que todo PDF debe atravesar antes de llegar a cualquier componente de procesamiento posterior. Es la primera línea de defensa contra IPI y EchoLeak.

**Responsabilidades**:
- Interceptar automáticamente todo PDF en la capa de ingesta (middleware, no invocación explícita por módulos downstream)
- Detectar y neutralizar vectores de ataque: CSS oculto, homóglifos Unicode, instrucciones en metadatos SVG/CDATA, instrucciones multi-idioma ocultas (RT1-RT4)
- Aplicar técnica Spotlighting: insertar tokens de procedencia que separan instrucciones del sistema de contenido externo
- Clasificar el estado de sanitización: Limpio / Elementos detectados y neutralizados / No sanitizable
- Registrar cada evento de sanitización en el log del `ModuloAdmin` con tipo de ataque y fragmento neutralizado
- Bloquear el paso al `MotorExtraccion` si la sanitización es incompleta (fail closed)

**Interfaces**: Recibe `bytes` de PDF, retorna `PDFSanitizado` (contenido + estado + metadata de sanitización)  
**Dependencias**: `ModuloAdmin` (log de eventos de seguridad)  
**RF cubiertos**: RF-02.1, RF-02.2, RF-02.3, RF-02.4, RF-02.5, RF-02.6

---

## C-03 — MotorExtraccion

**Propósito**: Extraer las variables de riesgo documental de cada PDF sanitizado usando servicios de Document Intelligence en cloud y el LLM Claude para interpretación semántica.

**Responsabilidades**:
- Enviar el PDF sanitizado (con tokens Spotlighting) al servicio cloud de Document AI para extracción de texto con OCR
- Construir el prompt estructurado para Claude con el contenido del PDF y las variables a extraer
- Extraer las variables de riesgo: vigencias, garantías, penalidades, exclusiones, condiciones de pago, y otras variables definidas
- Calcular el indicador de confianza (Alta/Media/Baja) para cada variable
- Generar la referencia de fuente exacta para cada variable (PDF: nombre, página, sección)
- Escalar variables con confianza < umbral al `ServicioPipeline` para HITL
- Nunca inventar valores: variables no encontradas tienen valor nulo + flag de escalamiento

**Interfaces externas**: `IGatewayDocumentAI` (Document AI cloud), `IGatewayLLM` (Claude)  
**Contrato de salida**: `ResultadoExtraccion` — lista de variables con (nombre, valor|null, confianza, referencia_fuente, requiere_revision)  
**Dependencias**: `GatewayLLM`, `GatewayDocumentAI`, `SanitizadorZeroTrust` (recibe PDFs ya sanitizados)  
**RF cubiertos**: RF-03.1, RF-03.3, RF-03.4, RF-03.6, RF-03.7, RF-03.8

---

## C-04 — MotorCruce

**Propósito**: Cruzar los datos estructurados de SAB con las variables extraídas de los PDFs, identificar discrepancias, clasificarlas por severidad y generar los indicadores de completitud.

**Responsabilidades**:
- Recibir el `ResultadoExtraccion` del `MotorExtraccion` y los datos de SAB para la cotización
- Comparar campo a campo: valores de SAB vs. variables extraídas de PDFs
- Detectar discrepancias: valor diferente, campo ausente en PDF, campo presente en PDF pero no en SAB
- Clasificar cada discrepancia por severidad: Alta / Media / Baja (según reglas de negocio definidas en Functional Design)
- Generar referencia de fuente para cada discrepancia: campo de BD + (PDF: página/sección)
- Identificar qué variables tienen confianza baja y necesitan aclaración al proveedor
- Producir el resultado de cruce para consumo por `GeneradorReportes` y `GestorAclaraciones`

**Contrato de salida**: `ResultadoCruce` — lista de discrepancias con (campo, valor_sab, valor_pdf, severidad, referencia_fuente, requiere_aclaracion)  
**Dependencias**: `MotorExtraccion` (recibe su salida), `RepositorioLocal` (lee datos SAB vía repositorio)  
**RF cubiertos**: RF-03.2, RF-03.3, RF-03.5

---

## C-05 — GestorAclaraciones

**Propósito**: Gestionar el ciclo completo de comunicación HITL con proveedores: generación de borradores, aprobación por el Analista, envío SMTP, y recepción de respuestas vía IMAP.

**Responsabilidades**:
- Generar borradores de solicitudes de aclaración para discrepancias e inconsistencias detectadas por el `MotorCruce`
- Exponer los borradores al Analista para revisión, edición y aprobación/descarte
- Enviar las solicitudes aprobadas vía SMTP usando el buzón monitorizado como remitente
- Ejecutar el polling IMAP como asyncio background task (default: cada 5 minutos)
- Correlacionar emails de respuesta con solicitudes originales (thread ID / asunto con ID)
- Extraer PDFs adjuntos de las respuestas y devolverlos al `ServicioPipeline` para reingesta
- Notificar al Analista vía `GestorWebSocket` cuando llega una respuesta de proveedor
- Mantener el historial de aclaraciones en `RepositorioLocal` con audit trail completo

**Dependencias**: `RepositorioLocal`, `GestorWebSocket`, `ServicioPipeline` (reingesta de respuestas)  
**RF cubiertos**: RF-04.1 al RF-04.8, RF-01.6, RF-01.7, RF-01.8

---

## C-06 — GeneradorReportes

**Propósito**: Consolidar los resultados del análisis y producir la síntesis comparativa de proveedores y el reporte formal para el Comité de Contratación.

**Responsabilidades**:
- Generar la síntesis por proveedor: variables extraídas, discrepancias, confianza, referencias de fuente
- Generar el reporte comparativo consolidado: ranking, matriz de cumplimiento, riesgos por severidad, trazabilidad completa
- Garantizar que toda afirmación en el reporte tenga una cita de fuente verificable (nunca dato sin referencia)
- Aplicar las restricciones de lenguaje del PRD (sin "fraude", "colusión", "culpable")
- Exportar el reporte a PDF via librería de generación PDF
- Proveer la vista del reporte en modo solo lectura para el Comité

**Dependencias**: `RepositorioLocal` (lee datos de análisis y aclaraciones)  
**RF cubiertos**: RF-05.1 al RF-05.5

---

## C-07 — ModuloAdmin

**Propósito**: Proveer observabilidad operacional y de seguridad del sistema al Admin (Director) y al CISO.

**Responsabilidades**:
- Registrar logs estructurados de cada operación del pipeline (timestamp, ID correlación, componente, resultado)
- Registrar eventos de seguridad del `SanitizadorZeroTrust` (tipo de elemento, fragmento, PDF)
- Generar y almacenar alertas de error de procesamiento (PDF corrupto, timeout LLM, fallo SMTP)
- Exponer el panel de logs al Admin y el log de sanitización al CISO (con control de rol)
- Asegurar que los logs sean append-only (inmutables) y retenidos por mínimo 90 días (SECURITY-14)

**Dependencias**: `RepositorioLocal` (persistencia de logs y alertas), `AutenticadorOAuth` (control de roles)  
**RF cubiertos**: RF-06.1, RF-06.2, RF-06.3

---

## C-08 — AutenticadorOAuth

**Propósito**: Gestionar el ciclo completo de autenticación OAuth2/OIDC, validación de tokens, asignación de roles y gestión de sesiones.

**Responsabilidades**:
- Implementar el flujo OAuth2/OIDC con Google Workspace y/o Microsoft Azure AD
- Validar tokens en cada request entrante (servidor, no solo en login) — SECURITY-12
- Asignar roles según configuración: Analista / Admin / CISO / Comité
- Gestionar expiración de sesiones y revocación de tokens en logout
- Proveer middleware de autorización para todos los endpoints de la `APIRest`
- Enforcer MFA para cuentas con rol Admin y CISO

**Dependencias**: Proveedor OAuth externo (Google/Microsoft), `RepositorioLocal` (sesiones)  
**RF cubiertos**: RF-07.2, RF-07.3, RNF-02 (SECURITY-12)

---

## C-09 — ServicioPipeline

**Propósito**: Orquestador central del análisis. Coordina la secuencia completa desde la activación hasta la notificación final, gestiona el estado en memoria y lo publica vía WebSocket.

**Responsabilidades**:
- Recibir la solicitud de análisis de un portafolio desde la `APIRest`
- Orquestar la secuencia: IngestaConector → [SanitizadorZeroTrust middleware] → MotorExtraccion → MotorCruce → GeneradorReportes (síntesis preliminar)
- Gestionar el estado del análisis en memoria (estado por cotización, progreso global)
- Publicar actualizaciones de progreso al `GestorWebSocket` en tiempo real
- Manejar errores de cada etapa sin detener el pipeline completo (continúa con las cotizaciones restantes)
- Triggerear la generación de borradores de aclaración en el `GestorAclaraciones` tras el cruce
- Recibir documentos de respuesta de proveedor del `GestorAclaraciones` y reiniciar el pipeline parcial para esos documentos

**Dependencias**: Todos los componentes C-01 a C-07, `GestorWebSocket`, `RepositorioLocal`  
**RF cubiertos**: RF-01 al RF-04 (coordinación), RF-07.4

---

## C-10 — APIRest

**Propósito**: Capa FastAPI que expone todos los endpoints REST del sistema al frontend SPA Angular.

**Responsabilidades**:
- Definir y exponer todos los endpoints del sistema (ver `component-methods.md`)
- Aplicar el middleware de autenticación (`AutenticadorOAuth`) en todos los endpoints no públicos
- Aplicar validación de inputs (tipo, longitud, formato) en todos los parámetros — SECURITY-05
- Configurar CORS restrictivo (solo orígenes permitidos explícitamente) — SECURITY-08
- Configurar headers HTTP de seguridad — SECURITY-04
- Aplicar rate limiting en endpoints públicos y de autenticación — SECURITY-11
- Gestionar los endpoints WebSocket para actualizaciones de progreso
- Servir fragmentos de PDFs almacenados para la funcionalidad de visor con navegación a fuente (RF-07.5)

**Dependencias**: `ServicioPipeline`, `GestorAclaraciones`, `GeneradorReportes`, `ModuloAdmin`, `AutenticadorOAuth`, `GestorWebSocket`, `RepositorioLocal`  
**RF cubiertos**: RF-07.1 al RF-07.5, RNF-02 (SECURITY-04, SECURITY-05, SECURITY-08, SECURITY-11)

---

## C-11 — GatewayLLM

**Propósito**: Abstracción de la interacción con Anthropic Claude. Implementa la interfaz `IGatewayLLM` que el `MotorExtraccion` consume, desacoplando la lógica de extracción del proveedor concreto de LLM.

**Responsabilidades**:
- Implementar `IGatewayLLM.completar(prompt: str, contexto: dict) → RespuestaLLM`
- Gestionar la autenticación con la API de Anthropic (API key via secrets manager)
- Implementar manejo de errores: timeout, rate limit, error de API → escalar al humano
- Aplicar los límites de contexto del modelo (chunking si el PDF excede el contexto)
- Registrar métricas de uso (tokens, latencia) en el `ModuloAdmin`

**Interfaz**: `IGatewayLLM` (abstracción), `AnthropicClaudeGateway` (implementación concreta)  
**Dependencias**: SDK oficial de Anthropic, `ModuloAdmin` (métricas)

---

## C-12 — RepositorioSAB

**Propósito**: Repository pattern que encapsula todas las queries read-only a la base de datos de SAB. Si en el futuro SAB expone una API REST, solo este componente cambia.

**Responsabilidades**:
- Implementar `IRepositorioSAB` con métodos para: listar cotizaciones de un proceso, obtener ítems/precios de una cotización, listar proveedores, obtener rutas de PDFs
- Gestionar la conexión a la BD de SAB (pool de conexiones, timeout, read-only enforced)
- Abstraer el tipo de BD (SQL o NoSQL) internamente
- Nunca escribir en SAB bajo ninguna circunstancia

**Interfaz**: `IRepositorioSAB`  
**Dependencias**: Driver de BD de SAB (SQLAlchemy u equivalente)

---

## C-13 — RepositorioLocal

**Propósito**: Capa de acceso a datos de la base de datos PostgreSQL propia del sistema (no SAB). Persiste el estado de análisis, aclaraciones, audit trail, logs y sesiones.

**Responsabilidades**:
- Implementar acceso a las entidades del sistema: Portafolio, Cotizacion, PDF, VariableExtraida, Discrepancia, Aclaracion, LogEvento, AuditTrail, Sesion
- Garantizar que el audit trail sea append-only (insert, no update/delete)
- Gestionar el esquema de BD con migraciones versionadas (Alembic)
- Proveer transacciones con rollback en paths de error

**Dependencias**: PostgreSQL, SQLAlchemy + Alembic

---

## C-14 — GestorWebSocket

**Propósito**: Gestionar las conexiones WebSocket activas y publicar actualizaciones de progreso del análisis al frontend en tiempo real.

**Responsabilidades**:
- Mantener el registro de conexiones WebSocket activas por sesión de usuario
- Recibir eventos de progreso del `ServicioPipeline` y retransmitirlos al cliente correspondiente
- Recibir notificaciones del `GestorAclaraciones` (respuesta de proveedor recibida) y publicarlas al Analista conectado
- Gestionar desconexiones y reconexiones de forma transparente
- Los mensajes WebSocket tienen formato JSON estructurado: `{tipo, payload, timestamp}`

**Dependencias**: FastAPI WebSocket, `AutenticadorOAuth` (validar token en el handshake WS)

---

## C-15 — SPAAngular

**Propósito**: Interfaz de usuario Angular que implementa las vistas de todas las épicas del sistema.

**Responsabilidades**:
- Autenticar al usuario vía OAuth2/OIDC (flujo PKCE) y gestionar tokens en memoria (no localStorage)
- Organizar las vistas en NgModules por épica: IngestaModule, AnalisisModule, AclaracionModule, ReportesModule, AdminModule
- Implementar todas las vistas de las US-01 a US-23 (Must Have)
- Gestionar la conexión WebSocket para actualizaciones de progreso en tiempo real
- Aplicar las restricciones de roles en el routing (guards Angular)
- Configurar los HTTP Security Headers en el servidor que sirve la SPA (CSP, HSTS, etc.) — SECURITY-04

**Dependencias**: `APIRest` (HTTP), WebSocket (`GestorWebSocket`), Proveedor OAuth externo

---

## C-16 — GatewayDocumentAI

**Propósito**: Abstracción de la interacción con servicios de Document Intelligence en la nube. Implementa la interfaz `IGatewayDocumentAI` que el `MotorExtraccion` consume, desacoplando la lógica de extracción del proveedor concreto de OCR/Document AI.

**Responsabilidades**:
- Implementar `IGatewayDocumentAI.extraer_texto(pdf_bytes) → TextoExtraido`
- Gestionar la autenticación con el proveedor cloud (Google Document AI / AWS Textract / Azure Form Recognizer)
- Implementar manejo de errores: timeout, cuota excedida, error de API → fallback a extracción básica o escalamiento
- Registrar métricas de uso (páginas procesadas, latencia, costos estimados) en el `ModuloAdmin`
- Retornar texto estructurado con referencias de página y sección para trazabilidad

**Interfaz**: `IGatewayDocumentAI` (abstracción), `GoogleDocumentAIGateway` (implementación concreta por defecto)  
**Dependencias**: SDK del proveedor cloud (google-cloud-documentai o equivalente), `ModuloAdmin` (métricas)
