# Documento de Requerimientos — Assistent Buy AI Agent

> **Versión**: 1.0 (MVP)  
> **Fecha**: 2026-05-23  
> **Estado**: Aprobado — Listo para Workflow Planning  

---

## Resumen de Análisis de Intención

| Atributo | Valor |
|---|---|
| **Solicitud del usuario** | Construir un asistente de compras que lea cotizaciones completas con sus anexos PDF, basado en el PRD de Assistent Buy AI Agent |
| **Tipo de solicitud** | Nuevo Producto — Greenfield |
| **Alcance** | Sistema completo: 6 módulos, frontend SPA, backend Python, integración con SAB, pipeline de seguridad, motor de IA |
| **Complejidad** | Alta — Agente de IA con análisis semántico de PDFs, cruce de datos estructurados vs. no estructurados, Zero Trust, HITL |
| **Profundidad de requerimientos** | Comprehensive |

---

## Decisiones Técnicas Confirmadas

| # | Área | Decisión |
|---|------|----------|
| **1.1** | Integración SAB | Acceso directo a base de datos (SQL/NoSQL) — read-only |
| **1.2** | Almacenamiento de PDFs | Sistema de archivos local / servidor de archivos accesible por ruta |
| **1.3** | Volumen MVP | Hasta 50 cotizaciones / hasta 200 PDFs por portafolio |
| **2.1** | Backend | Python (FastAPI) — ecosistema natural para IA/ML |
| **2.2** | LLM | Anthropic Claude — contexto largo, razonamiento, factualidad |
| **2.3** | Extracción PDF | Servicio cloud Document AI (Google Document AI / AWS Textract / Azure Form Recognizer) |
| **2.4** | Infraestructura | Cloud pública (AWS / GCP / Azure) |
| **3.1** | Interfaz | Aplicación web independiente (SPA) accesible desde SAB via link |
| **3.2** | Frontend | Angular — consistencia con el ecosistema tecnológico existente |
| **4.1** | Módulo Admin | Solo logs de procesamiento y alertas de error en MVP |
| **4.2** | Export de reportes | Export a PDF nativo desde la interfaz — Must Have |
| **4.3** | Envío de aclaraciones | Integración SMTP — el sistema envía emails directamente al proveedor |
| **4.4** | Recepción de respuestas | Buzón monitorizado vía IMAP polling — el sistema lee respuestas y adjuntos del proveedor automáticamente |
| **5.1** | Autenticación | OAuth2 / OIDC con proveedor externo (Google / Microsoft Azure AD) |
| **5.2** | Privacidad de datos | Sin restricciones especiales para MVP — se valida en piloto |
| **6.1** | Security Baseline | ✅ HABILITADA — todas las reglas como restricciones bloqueantes |
| **6.2** | Property-Based Testing | ✅ HABILITADA PARCIAL — PBT-02, PBT-03, PBT-07, PBT-08, PBT-09 |

---

## Requerimientos Funcionales

### RF-01: Módulo de Ingesta y Conectores (M1, M2)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-01.1 | El sistema DEBE conectarse a la base de datos de SAB en modo read-only mediante acceso directo SQL/NoSQL para extraer datos estructurados de cotizaciones (ítems, precios, proveedores, fechas) | Must Have | PRD M1 |
| RF-01.2 | El sistema DEBE acceder al sistema de archivos del servidor SAB para recuperar el 100% de los PDFs adjuntos asociados a cada cotización de un portafolio | Must Have | PRD M2 |
| RF-01.3 | El sistema DEBE procesar portafolios de hasta 50 cotizaciones con hasta 200 documentos PDF simultáneamente | Must Have | Resp. 1.3 |
| RF-01.4 | El sistema DEBE manejar fallos de ingesta por documento (PDF corrupto, PDF no encontrado) sin detener el procesamiento del portafolio completo | Must Have | PRD Journey 3 |
| RF-01.5 | El sistema DEBE marcar cada cotización con su estado de ingesta: Completa / Parcial / No procesable | Must Have | PRD Journey 3 |
| RF-01.6 | El sistema DEBE monitorizar un buzón de email dedicado (ej: `aclaraciones@dominio.com`) mediante IMAP polling para detectar respuestas entrantes de proveedores | Must Have | Resp. 4.4 |
| RF-01.7 | Al recibir un email de respuesta, el sistema DEBE correlacionarlo con la solicitud de aclaración original (por hilo de email / thread ID o asunto) y extraer automáticamente los documentos PDF adjuntos | Must Have | Resp. 4.4 |
| RF-01.8 | Los documentos adjuntos en las respuestas de proveedores DEBEN reingresar al pipeline de sanitización (Módulo 2) y luego al motor de análisis (Módulo 3) para actualizar la síntesis del proveedor | Must Have | PRD Journey 1 paso 10 |

### RF-02: Módulo de Sanitización y Seguridad (M3, M4)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-02.1 | El sistema DEBE pasar cada PDF por un pipeline de sanitización Zero Trust antes de enviarlo al LLM — ningún PDF externo puede llegar al LLM sin ser sanitizado | Must Have | PRD Principio 3 |
| RF-02.2 | El pipeline de sanitización DEBE neutralizar: CSS oculto, homóglifos Unicode, instrucciones en metadatos SVG/CDATA, instrucciones multi-idioma ocultas | Must Have | PRD RT1-RT4 |
| RF-02.3 | El sistema DEBE implementar Spotlighting — señales de procedencia que diferencian instrucciones del sistema vs. contenido de documentos externos | Must Have | PRD M4 |
| RF-02.4 | El sistema DEBE registrar el estado de sanitización de cada documento y alertar si se detectan elementos adversariales | Must Have | PRD Principio 3 |
| RF-02.5 | La tasa de ataque exitoso (ASR) de Inyección Indirecta de Prompts DEBE ser < 2% sobre el corpus adversarial D3 | Must Have | PRD §10 |
| RF-02.6 | La tasa de exfiltraciones de datos (EchoLeak) DEBE ser 0 incidentes en producción | Must Have | PRD §10 |

### RF-03: Módulo de Análisis y Extracción (M5, M6, M7, M8)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-03.1 | El sistema DEBE extraer de cada PDF: vigencias de oferta, garantías, penalidades, exclusiones, condiciones de pago, y otras variables de riesgo documental | Must Have | PRD M5 |
| RF-03.2 | El sistema DEBE cruzar automáticamente los datos estructurados de SAB (ítems, precios) contra las variables extraídas de los PDFs e identificar discrepancias | Must Have | PRD M6 |
| RF-03.3 | Cada variable extraída DEBE tener un indicador de confianza (alta / media / baja) y una referencia verificable a su fuente (PDF: página, sección; o campo de BD) | Must Have | PRD M7 / Principio 4 |
| RF-03.4 | El sistema DEBE escalar automáticamente al humano cuando la confianza de una variable cae por debajo del umbral configurado — NUNCA debe inventar valores | Must Have | PRD M8 / Principio 2 |
| RF-03.5 | Las discrepancias detectadas DEBEN clasificarse por severidad (alta / media / baja) | Must Have | PRD CU2 |
| RF-03.6 | La precisión de extracción DEBE superar el 96% contra el corpus anotado D2 como gate de lanzamiento | Must Have | PRD §11 |
| RF-03.7 | La tasa de alucinación (datos no presentes en los documentos fuente) DEBE ser < 1% | Must Have | PRD §10 |
| RF-03.8 | El sistema DEBE usar Google Document AI / AWS Textract / Azure Form Recognizer para extracción de texto, incluyendo PDFs escaneados con OCR | Must Have | Resp. 2.3 |

### RF-04: Módulo de Comunicación HITL (M9, M10)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-04.1 | El sistema DEBE generar borradores de solicitudes de aclaración para ambigüedades e inconsistencias detectadas, en lenguaje formal orientado a proveedores | Must Have | PRD M9 |
| RF-04.2 | El analista DEBE poder revisar, editar y aprobar o descartar cada borrador antes de cualquier envío | Must Have | PRD M10 / Principio 2 |
| RF-04.3 | El sistema DEBE enviar las solicitudes aprobadas directamente por email vía SMTP al proveedor, usando como dirección remitente el buzón monitorizado (ej: `aclaraciones@dominio.com`) para garantizar que las respuestas lleguen al canal IMAP | Must Have | Resp. 4.3 / 4.4 |
| RF-04.4 | El sistema NUNCA DEBE enviar comunicaciones externas sin aprobación explícita del analista | Must Have | PRD Principio 2 / W1 |
| RF-04.5 | El sistema DEBE mantener un histórico de borradores con su estado (pendiente / aprobado / descartado / respondido) y la identidad del analista que tomó la decisión | Must Have | PRD §9 |
| RF-04.6 | El flujo de aprobación DEBE completarse en < 2 minutos por solicitud de aclaración (criterio de usabilidad del piloto) | Must Have | PRD Días 31-60 |
| RF-04.7 | El sistema DEBE notificar al analista cuando llega una respuesta de proveedor al buzón IMAP, indicando a qué solicitud de aclaración corresponde y si incluye documentos adjuntos | Must Have | Resp. 4.4 |
| RF-04.8 | El intervalo de polling IMAP DEBE ser configurable (default: cada 5 minutos) y el sistema DEBE registrar en el audit trail cada email recibido, con timestamp y número de adjuntos detectados | Must Have | SECURITY-03 / Resp. 4.4 |

### RF-05: Módulo de Reportes y Síntesis (M11, M12)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-05.1 | El sistema DEBE generar una síntesis por proveedor que incluya: variables extraídas, discrepancias detectadas, indicadores de confianza, y referencias verificables a fuentes | Must Have | PRD M11 |
| RF-05.2 | El sistema DEBE generar un reporte comparativo para el Comité de Contratación con: ranking de proveedores, matriz de cumplimiento, riesgos identificados, y trazabilidad completa | Must Have | PRD M12 |
| RF-05.3 | Los reportes DEBEN poder exportarse en formato PDF directamente desde la interfaz | Must Have | Resp. 4.2 |
| RF-05.4 | Toda afirmación en los reportes DEBE citar su fuente verificable (PDF página/sección o campo de BD) — ningún dato sin referencia | Must Have | PRD Principio 4 |
| RF-05.5 | Los reportes NUNCA DEBEN usar términos como "fraude", "colusión" o "culpable" — solo: "discrepancia", "riesgo documental", "anomalía" | Must Have | PRD Principio 5 |

### RF-06: Módulo de Administración y Monitoring (Básico MVP)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-06.1 | El sistema DEBE registrar logs estructurados de cada operación de procesamiento con timestamps, IDs de correlación y resultado | Must Have | Resp. 4.1 / SECURITY-03 |
| RF-06.2 | El sistema DEBE generar alertas cuando ocurran errores de procesamiento, fallos de sanitización, o documentos no procesables | Must Have | Resp. 4.1 |
| RF-06.3 | El sistema DEBE mostrar el log de sanitización de PDFs a usuarios con rol Admin | Must Have | PRD Módulo 6 |

### RF-07: Interfaz de Usuario (SPA Angular)

| ID | Requerimiento | Prioridad | Fuente |
|----|--------------|-----------|--------|
| RF-07.1 | La interfaz DEBE ser una SPA Angular accesible mediante un link desde SAB | Must Have | Resp. 3.1 / 3.2 |
| RF-07.2 | La interfaz DEBE implementar autenticación OAuth2/OIDC con Google Workspace o Microsoft Azure AD | Must Have | Resp. 5.1 |
| RF-07.3 | La interfaz DEBE implementar los 4 roles definidos en el PRD: Analista, Admin, Sistema, Comité | Must Have | PRD §9 |
| RF-07.4 | La interfaz DEBE mostrar el progreso del análisis con indicador en tiempo real: "Procesando X de Y cotizaciones" | Must Have | PRD Principio 1 / Journey 1 |
| RF-07.5 | La interfaz DEBE permitir navegar desde cualquier dato de la síntesis directamente al fragmento fuente en el documento original | Must Have | PRD Principio 4 |

---

## Requerimientos No Funcionales

### RNF-01: Rendimiento

| ID | Requerimiento | Meta | Fuente |
|----|--------------|------|--------|
| RNF-01.1 | Latencia máxima de análisis para un portafolio denso (50 cotizaciones / 200 PDFs) | ≤ 60 segundos (o indicar tiempo estimado en UI) | PRD Principio 1 |
| RNF-01.2 | Precisión de extracción de variables | > 96% | PRD §10 |
| RNF-01.3 | Tasa de alucinación | < 1% | PRD §10 |
| RNF-01.4 | Tasa de recall de variables críticas | > 95% | PRD §11 |
| RNF-01.5 | Consistencia entre ejecuciones repetidas sobre el mismo corpus | > 95% | PRD §11 |

### RNF-02: Seguridad (Security Baseline — HABILITADA)

| ID | Requerimiento | Regla SEC |
|----|--------------|-----------|
| RNF-02.1 | Cifrado en reposo (AES-256 o equivalente) y en tránsito (TLS 1.2+) para todos los datos | SECURITY-01 |
| RNF-02.2 | Logs de acceso habilitados en todos los intermediarios de red (API Gateway, Load Balancer) | SECURITY-02 |
| RNF-02.3 | Logging estructurado en la aplicación — sin datos sensibles en logs | SECURITY-03 |
| RNF-02.4 | Headers HTTP de seguridad obligatorios: CSP, HSTS, X-Frame-Options, X-Content-Type-Options | SECURITY-04 |
| RNF-02.5 | Validación de todos los inputs de API — sin concatenación SQL, longitudes máximas definidas | SECURITY-05 |
| RNF-02.6 | Permisos mínimos necesarios — sin wildcards en políticas IAM | SECURITY-06 |
| RNF-02.7 | Red restrictiva — deny-by-default, sin acceso inbound 0.0.0.0/0 excepto puertos 80/443 en LB | SECURITY-07 |
| RNF-02.8 | Autorización a nivel de aplicación en cada endpoint — deny-by-default, prevención de IDOR | SECURITY-08 |
| RNF-02.9 | Hardening: sin credenciales por defecto, sin stack traces en producción | SECURITY-09 |
| RNF-02.10 | Dependencias con versiones exactas (lock file), escaneo de vulnerabilidades en CI | SECURITY-10 |
| RNF-02.11 | Lógica de auth en módulo dedicado, rate limiting en endpoints públicos | SECURITY-11 |
| RNF-02.12 | OAuth2/OIDC con MFA para admins, sin credenciales hardcodeadas, sessions invalidadas en logout | SECURITY-12 |
| RNF-02.13 | Sin deserialización insegura, audit trail de cambios críticos, integridad de pipeline CI/CD | SECURITY-13 |
| RNF-02.14 | Alertas para fallos de auth repetidos, retención de logs mínimo 90 días | SECURITY-14 |
| RNF-02.15 | Manejo explícito de todas las excepciones — fail closed, sin recursos no liberados | SECURITY-15 |

### RNF-03: Testing (Property-Based Testing — PARCIAL)

Las siguientes reglas PBT son **bloqueantes** en modo Parcial:

| Regla | Aplicación en Assistent Buy |
|-------|----------------------------|
| **PBT-02** (Round-trip) | Serialización/deserialización de resultados de análisis, modelos de datos de cotizaciones |
| **PBT-03** (Invariantes) | Motor de extracción: precios siempre positivos; cruce: cada discrepancia tiene fuente; confianza en [0,1] |
| **PBT-07** (Generadores de calidad) | Generadores de dominio para: Cotización, DocumentoPDF, VariableExtraída, Discrepancia |
| **PBT-08** (Shrinking y reproducibilidad) | Todos los tests PBT con seed logging en CI |
| **PBT-09** (Framework) | **Hypothesis** (Python) para el backend; configurado en CI pipeline |

Las reglas PBT-01, PBT-04, PBT-05, PBT-06, PBT-10 son **advisory** (no bloqueantes) en modo Parcial.

### RNF-04: Disponibilidad y Confiabilidad

| ID | Requerimiento | Meta |
|----|--------------|------|
| RNF-04.1 | Disponibilidad del servicio en horario laboral (6am - 10pm hora Colombia) | ≥ 99% |
| RNF-04.2 | Procesamiento de documentos fallidos no debe interrumpir el resto del portafolio | 100% |
| RNF-04.3 | Trazabilidad completa: 100% de las afirmaciones con cita verificable | 100% |

### RNF-05: Privacidad de Datos

| ID | Requerimiento |
|----|--------------|
| RNF-05.1 | Sin restricciones de residencia de datos para el MVP — se valida en piloto con cada universidad |
| RNF-05.2 | Los datos de cotizaciones NO deben ser usados para entrenar modelos externos |
| RNF-05.3 | El sistema es capa de solo lectura sobre SAB — NUNCA modifica registros en SAB |

---

## Restricciones Explícitas (Won't Have)

Las siguientes funcionalidades están **explícitamente excluidas** del MVP:

| ID | Excluido | Razón |
|----|----------|-------|
| W1 | Comunicación autónoma sin aprobación humana | Principio HITL + riesgo EchoLeak |
| W2 | Detección de fraude o colusión | Posicionamiento: riesgo documental, no anti-fraude |
| W3 | Adjudicación automática | Responsabilidad legal + Principio HITL |
| W4 | Integración directa con ERPs externos (SAP, Oracle) | Fase 2 |
| W5 | Escritura en SAB (modificar registros) | Principio de capa analítica read-only |
| W6 | Negociación automática de precios | W1 + W3 |
| W7 | Dashboard de métricas completo (S3) | Diferido a v1.1 post-piloto |
| W8 | Circuit-breakers de diversidad de proveedores (S2) | Requiere datos históricos de S1 |

---

## Gates de Calidad (Go / No-Go)

El sistema **no puede lanzarse a producción** hasta superar todos estos criterios:

| Gate | Criterio | Umbral |
|------|----------|--------|
| G1 | Precisión de extracción sobre corpus D2 | > 96% |
| G2 | ASR de IPI sobre corpus D3 adversarial | < 2% |
| G3 | Tasa de alucinación | < 1% |
| G4 | Exfiltraciones EchoLeak | 0 incidentes |
| G5 | Consistencia entre ejecuciones | > 95% |
| G6 | Cumplimiento Security Baseline (todas las reglas) | 100% compliant |

---

## Stack Tecnológico Definido

| Capa | Tecnología |
|------|-----------|
| **Backend** | Python 3.12+ / FastAPI |
| **Frontend** | Angular (versión LTS actual) |
| **LLM** | Anthropic Claude (API) |
| **Extracción PDF** | Google Document AI / AWS Textract / Azure Form Recognizer |
| **Base de datos propia** | PostgreSQL (estado de análisis, audit trail, histórico) |
| **Autenticación** | OAuth2/OIDC (Google Workspace o Microsoft Azure AD) |
| **Email saliente (HITL)** | SMTP (configurable: SendGrid / AWS SES / servidor corporativo) |
| **Email entrante (respuestas)** | IMAP polling sobre buzón dedicado — librería `imaplib` (Python stdlib) o `IMAPClient` |
| **Infraestructura** | Cloud pública (AWS / GCP / Azure) con contenedores Docker |
| **Testing PBT** | Hypothesis (Python) |
| **CI/CD** | GitHub Actions o equivalente con PBT + security scanning |

---

## Referencia al PRD

Este documento de requerimientos está basado en y es trazable al PRD completo:  
📄 [`assitant_buy_prd.md`](../../../assitant_buy_prd.md) — versión 1.0 (Mayo 2026)

Todos los Must Have del PRD (M1-M12) están cubiertos en este documento.  
Los Should Have (S1-S5) están diferidos a v1.1 como está definido en el PRD.
