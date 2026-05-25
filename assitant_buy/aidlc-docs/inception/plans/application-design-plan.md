# Plan de Application Design — Assistent Buy AI Agent

> **Instrucciones**: Responda cada pregunta llenando el campo `[Answer]:` con la letra de su opción o respuesta personalizada.

---

## Checkboxes de Ejecución

- [x] Contexto cargado (requirements.md + stories.md + PRD)
- [x] Plan creado con preguntas
- [x] Respuestas recibidas y analizadas
- [x] Diseño aprobado por usuario
- [x] `components.md` generado
- [x] `component-methods.md` generado
- [x] `services.md` generado
- [x] `component-dependency.md` generado
- [x] `application-design.md` (consolidado) generado

---

## Contexto de Diseño

El sistema tiene **6 módulos funcionales** según el PRD (M1-M12 agrupados), **4 capas** (Frontend SPA Angular, Backend FastAPI, Capa de IA, Capa de Datos), e integra **5 sistemas externos** (SAB BD, SAB filesystem, Document AI cloud, LLM Anthropic Claude, SMTP/IMAP).

Componentes candidatos identificados del PRD y los requerimientos:
1. **IngestaConector** — M1/M2: Lee SAB y recupera PDFs
2. **SanitizadorZeroTrust** — M3/M4: Pipeline de sanitización + Spotlighting
3. **MotorExtraccion** — M5: Extracción de variables via Document AI + Claude
4. **MotorCruce** — M6/M7/M8: Cruce de datos, discrepancias, confianza, escalamiento
5. **GestorAclaraciones** — M9/M10: Borradores HITL, SMTP saliente, IMAP polling
6. **GeneradorReportes** — M11/M12: Síntesis, reporte comparativo, export PDF
7. **ModuloAdmin** — M13: Logs, alertas, panel CISO
8. **AutenticadorOAuth** — Auth: OAuth2/OIDC, roles, sesiones
9. **APIGateway** — REST API FastAPI que expone todos los servicios al frontend

---

## Sección 1 — Identificación de Componentes

### Pregunta 1.1 — Granularidad del motor de IA
El motor de IA tiene dos responsabilidades distintas: extracción de variables (Document AI + Claude) y cruce de datos (lógica de negocio pura). ¿Deben ser un componente unificado o separados?

A) Separados — `MotorExtraccion` (IA) y `MotorCruce` (lógica de negocio) son componentes independientes con contratos bien definidos  
B) Unificados — un único componente `MotorAnalisis` que encapsula extracción + cruce  
C) Separados con servicio orquestador — `MotorExtraccion` y `MotorCruce` independientes, orquestados por un `ServicioAnalisis`  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.2 — Pipeline de sanitización: ¿componente o capa transversal?
La sanitización Zero Trust aplica a todos los PDFs entrantes (cotizaciones originales Y respuestas de proveedores). ¿Cómo se arquitectura?

A) Componente independiente `SanitizadorZeroTrust` — cualquier módulo que reciba PDFs llama explícitamente a este componente antes de procesarlos  
B) Middleware/interceptor — se aplica automáticamente en la capa de ingesta sin que los módulos downstream lo invoquen explícitamente  
C) Doble estrategia — middleware para la ingesta inicial (automático) + componente explícito para respuestas IMAP (manual, ya que son documentos de una fuente diferente)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.3 — Frontend: arquitectura de la SPA
¿Cómo se organiza la SPA Angular internamente?

A) Módulos por épica — un NgModule por épica del sistema (Ingesta, Análisis, Aclaración, Reportes, Admin)  
B) Módulos por persona — un NgModule por tipo de usuario (Analista, Admin/CISO, Comité)  
C) Feature modules con lazy loading — módulos de funcionalidad cargados bajo demanda según navegación  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Métodos y Contratos de Componentes

### Pregunta 2.1 — Contrato del Motor de Extracción
¿Qué debe retornar `MotorExtraccion` para cada PDF procesado?

A) Un objeto `ResultadoExtraccion` con: lista de variables (nombre, valor, confianza, referencia_fuente), estado (exitoso/parcial/fallido), y metadata del documento  
B) Solo las variables con confianza Alta — las de confianza Media/Baja se marcan como "requieren revisión" sin valor  
C) Un objeto con variables + un flag de "listo para cruce" que el orquestador usa para encadenar al MotorCruce  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Contrato del componente IMAP
El GestorAclaraciones necesita monitorizar el buzón IMAP. ¿Cómo se implementa el polling?

A) Background job/worker independiente — un proceso separado del servidor FastAPI que hace polling cada N minutos (ej: Celery Beat, APScheduler, cron interno)  
B) Endpoint en la API que el scheduler del sistema invoca (ej: llamada HTTP interna programada)  
C) Hilo de background dentro del proceso FastAPI (asyncio background task con loop)  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 2.3 — Persistencia de estado del análisis
¿Cómo se gestiona el estado del análisis mientras corre (ej: "cotización 12 de 18 procesada")?

A) Estado en PostgreSQL — cada cotización tiene un registro de estado actualizable en tiempo real; el frontend hace polling a la API para mostrar progreso  
B) Estado en memoria + WebSocket — el backend mantiene el estado en RAM y notifica al frontend via WebSocket para actualizaciones en tiempo real  
C) Estado en PostgreSQL + Server-Sent Events (SSE) — persistencia duradera con notificaciones push al frontend sin WebSocket completo  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 3 — Capa de Servicios

### Pregunta 3.1 — Servicio orquestador principal
¿Existe un servicio que coordine el pipeline completo de análisis (Ingesta → Sanitización → Extracción → Cruce → Notificación)?

A) Sí — un `ServicioPipeline` (o `ServicioAnalisis`) que orquesta la secuencia completa y gestiona el estado  
B) No — cada componente llama al siguiente directamente (encadenamiento directo entre componentes)  
C) Sí, pero distribuido — un `ServicioPipeline` para la secuencia principal, y el `GestorAclaraciones` maneja su propio sub-pipeline independiente  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 3.2 — Patrón de comunicación entre servicios
¿Cómo se comunican los componentes del backend entre sí?

A) Llamadas directas en proceso (in-process) — todos los componentes corren en el mismo proceso FastAPI y se llaman directamente  
B) Cola de mensajes (ej: Redis, RabbitMQ) — los componentes se comunican via mensajes asíncronos para mayor resiliencia  
C) Híbrido — el pipeline principal es in-process para minimizar latencia, pero el GestorAclaraciones/IMAP usa un scheduler asíncrono (Celery/APScheduler) para el polling  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 4 — Dependencias y Patrones

### Pregunta 4.1 — Patrón de acceso a datos (SAB)
La integración con SAB es read-only vía BD directa. ¿Cómo se abstrae este acceso?

A) Repository pattern — un `RepositorioSAB` que encapsula todas las queries a SAB; si en el futuro SAB expone una API, solo cambia el repositorio  
B) Acceso directo — el `IngestaConector` contiene las queries directamente sin una capa de abstracción adicional  
C) Adaptador con interfaz — un `AdaptadorSAB` que implementa una interfaz `IFuenteCotizaciones`; facilita mocking en tests y cambio de fuente en el futuro  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.2 — Patrón para el LLM (Claude)
¿Cómo se abstrae la interacción con Anthropic Claude?

A) Cliente directo — usar el SDK oficial de Anthropic directamente en el `MotorExtraccion`  
B) Capa de abstracción `IGatewayLLM` — una interfaz que el `MotorExtraccion` consume; la implementación concreta es Claude; facilita swap a otro LLM sin cambiar el motor  
C) Proxy con circuit breaker — capa de abstracción + manejo de fallos (si Claude no responde, registrar el error y escalar al humano)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 5 artefactos de Application Design.
