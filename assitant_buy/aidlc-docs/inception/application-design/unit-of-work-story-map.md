# Unit of Work — Story Map

> **Versión**: 1.0  
> **Fase**: INCEPTION — Units Generation  
> **Fecha**: 2026-05-25  
> **Total stories MVP**: 23 Must Have (US-01 a US-23)  
> **Stories diferidas**: 2 Should Have (US-24, US-25 — v1.1 post-piloto)  

---

## Mapa de Stories por Unidad

### U-01 — Fundación (Auth + Datos + API + WebSocket)

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-23** | Autenticación con Cuenta Institucional (OAuth2) | Todos los roles | 🔴 Must Have | E-05 |

> U-01 es la base técnica que habilita el acceso de todas las stories. Sin U-01 ninguna otra story es ejecutable.

---

### U-02 — Ingesta y Sanitización

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-01** | Iniciar Análisis de Portafolio | P-01 María (Analista) | 🔴 Must Have | E-01 |
| **US-02** | Ver Estado de Ingesta en Tiempo Real | P-01 María (Analista) | 🔴 Must Have | E-01 |
| **US-03** | Sanitización Zero Trust Automática por el Sistema | P-05 Sistema Agente | 🔴 Must Have | E-01 |

**Criterios de aceptación clave de U-02**:
- US-01: Ingesta inicia en ≤ 5 segundos; soporta hasta 50 cotizaciones / 200 PDFs
- US-02: Progreso en tiempo real via WebSocket (depende de U-01 para GestorWebSocket)
- US-03: ASR < 2% (Gate G2); 0 EchoLeak; fail closed si sanitización incompleta

---

### U-03 — Extracción, Cruce y Orquestación

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-04** | Ver Variables Extraídas por Proveedor | P-01 María (Analista) | 🔴 Must Have | E-02 |
| **US-05** | Ver Discrepancias Detectadas entre SAB y PDF | P-01 María (Analista) | 🔴 Must Have | E-02 |
| **US-06** | Consultar Indicador de Confianza y Fuente de una Variable | P-01 María (Analista) | 🔴 Must Have | E-02 |
| **US-07** | Escalamiento Automático por Baja Confianza | P-05 Sistema Agente | 🔴 Must Have | E-02 |
| **US-08** | Ver Cotizaciones con Estado Incompleto | P-01 María (Analista) | 🔴 Must Have | E-02 |

**Criterios de aceptación clave de U-03**:
- US-04: Precisión > 96% (Gate G1); alucinaciones < 1% (Gate G3)
- US-05: Discrepancias clasificadas Alta/Media/Baja con referencia de fuente
- US-06: 100% de variables con referencia verificable no vacía
- US-07: 0 valores inventados para variables escaladas
- US-08: Estados de ingesta diferenciados (Completa/Parcial/No procesable)

---

### U-04 — Comunicación HITL

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-09** | Revisar Borrador de Solicitud de Aclaración | P-01 María (Analista) | 🔴 Must Have | E-03 |
| **US-10** | Editar y Aprobar Borrador de Aclaración | P-01 María (Analista) | 🔴 Must Have | E-03 |
| **US-11** | Envío de Aclaración por SMTP | P-05 Sistema Agente | 🔴 Must Have | E-03 |
| **US-12** | Recibir Notificación de Respuesta de Proveedor | P-01 María (Analista) | 🔴 Must Have | E-03 |
| **US-13** | Monitoreo IMAP y Reingesta de Documentos de Respuesta | P-05 Sistema Agente | 🔴 Must Have | E-03 |
| **US-14** | Ver Historial de Aclaraciones | P-01 María (Analista) | 🔴 Must Have | E-03 |

**Criterios de aceptación clave de U-04**:
- US-10: Ningún email enviado sin aprobación explícita (audit trail completo)
- US-11: SMTP con retry backoff; envío en ≤ 30 segundos desde aprobación
- US-12: Notificación en < 2 minutos desde recepción del email
- US-13: Polling IMAP cada 5 minutos (configurable); audit trail de todos los emails
- US-14: Historial append-only, no editable

---

### U-05 — Reportes y Decisión

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-15** | Ver Síntesis Comparativa de Proveedores | P-01 María (Analista) | 🔴 Must Have | E-04 |
| **US-16** | Generar Reporte Comparativo para el Comité | P-01 María (Analista) | 🔴 Must Have | E-04 |
| **US-17** | Acceder al Reporte en Modo Lectura (Comité) | P-04 Comité | 🔴 Must Have | E-04 |
| **US-18** | Exportar Reporte a PDF | P-01 María (Analista) | 🔴 Must Have | E-04 |

**Criterios de aceptación clave de U-05**:
- US-15: Sin términos prohibidos (fraude, colusión, culpable, sospechoso)
- US-16: 100% de datos con referencia de fuente; reporte listo en ≤ 30 segundos
- US-17: Rol Comité solo lectura — deniega acceso a análisis/aclaraciones
- US-18: Export PDF en ≤ 15 segundos; conserva todas las referencias

---

### U-06 — Administración y Seguridad

| Story | Título | Persona | Prioridad | Épica |
|---|---|---|---|---|
| **US-19** | Ver Log de Sanitización de PDFs (CISO) | P-03 Alejandro (CISO) | 🔴 Must Have | E-05 |
| **US-20** | Recibir Alertas de Seguridad por Elemento Adversarial | P-03 Alejandro (CISO) | 🔴 Must Have | E-05 |
| **US-21** | Rechazo Automático de PDFs que No Pasan Sanitización | P-05 Sistema Agente | 🔴 Must Have | E-05 |
| **US-22** | Ver Logs de Procesamiento y Alertas (Admin) | P-02 Carlos (Admin) | 🔴 Must Have | E-05 |

**Criterios de aceptación clave de U-06**:
- US-19: Logs inmutables, retención mínima 90 días (SECURITY-14)
- US-20: Alertas en ≤ 2 minutos; EchoLeak = 0 en producción
- US-21: Gate G4 (0 EchoLeak); fail closed — no sanitizable = no llega al LLM
- US-22: Filtros por fecha, proveedor, tipo de evento; alertas persistentes hasta revisión

---

### U-07 — Frontend SPA Angular

| Story | Título | Módulo Angular | Épica |
|---|---|---|---|
| **US-01** | Iniciar Análisis de Portafolio | IngestaModule | E-01 |
| **US-02** | Ver Estado de Ingesta en Tiempo Real | IngestaModule | E-01 |
| **US-03** | Sanitización (log visible en Panel CISO) | IngestaModule | E-01 |
| **US-04** | Ver Variables Extraídas por Proveedor | AnalisisModule | E-02 |
| **US-05** | Ver Discrepancias Detectadas | AnalisisModule | E-02 |
| **US-06** | Consultar Indicador de Confianza y Fuente | AnalisisModule | E-02 |
| **US-07** | Escalamiento visible al Analista | AnalisisModule | E-02 |
| **US-08** | Ver Cotizaciones con Estado Incompleto | IngestaModule | E-01 |
| **US-09** | Revisar Borrador de Aclaración | AclaracionModule | E-03 |
| **US-10** | Editar y Aprobar Borrador | AclaracionModule | E-03 |
| **US-11** | Estado de envío SMTP visible | AclaracionModule | E-03 |
| **US-12** | Notificación de Respuesta (WebSocket) | AclaracionModule | E-03 |
| **US-13** | Estado de reingesta visible | AclaracionModule | E-03 |
| **US-14** | Historial de Aclaraciones | AclaracionModule | E-03 |
| **US-15** | Síntesis Comparativa de Proveedores | ReportesModule | E-04 |
| **US-16** | Generar Reporte para Comité | ReportesModule | E-04 |
| **US-17** | Vista solo-lectura para Comité | ReportesModule | E-04 |
| **US-18** | Exportar Reporte a PDF | ReportesModule | E-04 |
| **US-19** | Panel de Sanitización CISO | AdminModule | E-05 |
| **US-20** | Panel de Alertas de Seguridad | AdminModule | E-05 |
| **US-21** | Indicador visual de rechazo de PDF | IngestaModule | E-05 |
| **US-22** | Panel de Logs Admin | AdminModule | E-05 |
| **US-23** | Login OAuth2 / logout | SharedModule (Auth) | E-05 |

---

## Stories Diferidas (v1.1 — post-piloto)

| Story | Título | Persona | Prioridad | Unidad futura |
|---|---|---|---|---|
| **US-24** | Dashboard de Métricas de Desempeño (Admin) | P-02 Carlos (Admin) | 🟡 Should Have | U-06 (extensión) |
| **US-25** | Análisis Multi-Portafolio Comparativo (Admin) | P-02 Carlos (Admin) | 🟡 Should Have | U-03 / U-05 (extensión) |

> Estas stories están documentadas en E-06 y no forman parte del MVP. Se construyen en el ciclo S2/S3 según el PRD.

---

## Cobertura de Stories por Épica

| Épica | Stories MVP | Unidad(es) responsable(s) |
|---|---|---|
| E-01 — Activación y Pre-análisis | US-01, US-02, US-03 | U-02 (backend), U-07 (frontend) |
| E-02 — Análisis y Extracción | US-04, US-05, US-06, US-07, US-08 | U-03 (backend), U-07 (frontend) |
| E-03 — Comunicación y Aclaración HITL | US-09, US-10, US-11, US-12, US-13, US-14 | U-04 (backend), U-07 (frontend) |
| E-04 — Reportes y Decisión | US-15, US-16, US-17, US-18 | U-05 (backend), U-07 (frontend) |
| E-05 — Seguridad y Administración | US-19, US-20, US-21, US-22, US-23 | U-06 + U-01 (backend), U-07 (frontend) |
| E-06 — Supervisión Avanzada (v1.1) | US-24, US-25 | Diferido |

**Total stories cubiertos por el MVP**: 23/25 (100% Must Have)  
**Cobertura de Unidades**: 7/7 unidades con stories asignadas  
**Stories sin asignar**: 0  
