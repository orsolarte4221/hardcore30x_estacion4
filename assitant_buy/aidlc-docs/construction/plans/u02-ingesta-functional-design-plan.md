# Plan de Functional Design — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI  
> **Stories**: US-01 (Iniciar Análisis), US-02 (Estado en Tiempo Real), US-03 (Sanitización Zero Trust)

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta. Cuando termine todas, indíquelo y el workflow generará los 3 artefactos de Functional Design de U-02.

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Unit of work y stories revisados (unit-of-work.md + unit-of-work-story-map.md)
- [x] Plan de preguntas generado
- [x] Respuestas recibidas y analizadas
- [x] Plan aprobado

### PART 2 — Generation
- [x] `domain-entities.md` generado
- [x] `business-rules.md` generado
- [x] `business-logic-model.md` generado

---

## Sección 1 — Integración con SAB (C-01 IngestaConector + C-12 RepositorioSAB)

### Pregunta 1.1 — Fuente de datos SAB

El `IngestaConector` debe leer los datos del sistema SAB. ¿Cómo accede al SAB en el piloto?

A) Conexión read-only directa a la BD PostgreSQL/MySQL del SAB — query directa a las tablas del sistema  
B) API REST del SAB — el SAB expone endpoints para consultar portafolios y cotizaciones  
C) Archivos exportados del SAB — el SAB genera CSV/Excel periódicamente que el sistema consume  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.2 — Datos disponibles en SAB

¿Qué información del portafolio está disponible en el SAB para iniciar el análisis?

A) SAB provee: lista de proveedores, cotizaciones asociadas y **rutas/URLs a los PDFs** — el sistema solo lee metadatos y localiza los archivos  
B) SAB provee: solo metadatos del proceso (número de proceso, proveedores, montos esperados) — los PDFs se suben manualmente al sistema  
C) SAB provee: todo lo de A + **datos estructurados de las cotizaciones** (tablas de precios, especificaciones) que se cruzan con los PDFs  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.3 — Identificador del portafolio

¿Cómo identifica el Analista el portafolio que quiere analizar al iniciar el proceso?

A) Número de proceso SAB — el analista ingresa el identificador del proceso de contratación, el sistema lo busca en SAB  
B) Selección desde lista — el sistema lista los portafolios activos del SAB y el analista selecciona uno  
C) Carga manual — el analista sube un archivo de configuración (Excel/JSON) con la lista de cotizaciones y sus PDFs  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Pipeline Zero Trust (C-02 SanitizadorZeroTrust)

### Pregunta 2.1 — Reglas de sanitización RT1-RT4

El pipeline Zero Trust define 4 reglas de sanitización (RT1-RT4). ¿Cuáles son exactamente estas 4 reglas?

A) Definición estándar del PRD:  
   - **RT1** — Detección de metadatos maliciosos en PDF (scripts, JavaScript embebido, enlaces externos)  
   - **RT2** — Detección de inyección de prompt (instrucciones al LLM dentro del texto del PDF)  
   - **RT3** — Detección de contenido ofuscado (texto invisible, colores matching background, caracteres homoglíficos)  
   - **RT4** — Validación de estructura del PDF (PDF válido, no corrupto, dentro del límite de tamaño)  
B) Diferente definición — describa las 4 reglas después del tag [Answer]:  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Técnica Spotlighting

El `SanitizadorZeroTrust` usa **Spotlighting** para marcar el contenido del PDF antes de enviarlo al LLM. ¿Cómo funciona esta técnica en el sistema?

A) Envolturas de delimitadores — el texto del PDF se encierra en delimitadores especiales `[PDF_CONTENT_START]...[PDF_CONTENT_END]` para que el LLM distinga el contenido del documento de las instrucciones del sistema  
B) Codificación base64 — el texto del PDF se codifica en base64 antes de enviarlo al LLM para aislar instrucciones adversariales  
C) Resaltado de roles — se usa el sistema de roles del LLM (system/user) para aislar el contenido del PDF en el mensaje `user` y las instrucciones en `system`  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.3 — Política "fail closed" para PDFs rechazados

Si un PDF no pasa la sanitización (RT1-RT4 detectan problemas), ¿qué ocurre con la cotización asociada?

A) La cotización queda en estado `PENDIENTE_REVISION` — el Analista puede decidir continuar con los PDFs restantes o detener el análisis  
B) La cotización queda en estado `NO_PROCESABLE` — automáticamente y el análisis continúa con las demás cotizaciones del portafolio sin ella  
C) El análisis completo del portafolio se detiene — si cualquier PDF falla, todo el portafolio queda en `ERROR` hasta que el Analista resuelva  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 2.4 — Niveles de severidad de sanitización

¿Qué estados puede tener un PDF después de pasar por el `SanitizadorZeroTrust`?

A) 2 estados: `LIMPIO` (pasó todas las reglas) / `RECHAZADO` (falló al menos una regla)  
B) 3 estados: `LIMPIO` / `SOSPECHOSO` (anomalías menores, continúa con alerta) / `ADVERSARIAL` (bloqueado, no llega al LLM)  
C) 4 estados: `LIMPIO` / `ADVERTENCIA_METADATOS` / `INYECCION_PROMPT` / `CORRUPTO` — un estado por tipo de fallo  
X) Otro (describa después del tag [Answer]:)

[Answer]: X
[Description]: De acuerdo con la entidad de dominio `DocumentoPDF` aprobada en U-01, el PDF puede quedar en uno de los siguientes 3 estados post-sanitización:
1. `LIMPIO` (pasó todas las reglas RT1-RT4).
2. `ADVERSARIAL` (bloqueado por inyección de prompt activa o texto ofuscado, dispara alerta de seguridad).
3. `ERROR` (bloqueado por PDF corrupto, malformado o con scripts en metadatos imposibles de sanitizar).

---

## Sección 3 — Gateway Document AI (C-16 GatewayDocumentAI)

### Pregunta 3.1 — Rol de Document AI en U-02

¿En qué momento del pipeline de U-02 interviene el `GatewayDocumentAI`?

A) Solo en U-02 para **detección de PDFs escaneados/imagen** — Document AI convierte PDF imagen a texto OCR antes de que el `SanitizadorZeroTrust` analice el contenido textual  
B) No interviene en U-02 — Document AI se usa en U-03 (extracción de variables). En U-02 solo se hace sanitización del PDF crudo (sin OCR)  
C) En U-02 Document AI hace el OCR Y la sanitización — el gateway transforma el PDF a texto estructurado y el sanitizador analiza ese texto  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 3.2 — Manejo de PDFs escaneados (sin capa de texto)

Algunos PDFs de cotizaciones pueden ser documentos escaneados (imágenes). ¿Cómo los maneja el sistema?

A) Document AI (OCR) los convierte automáticamente — si el PDF no tiene capa de texto, se envía a Document AI para OCR antes de la sanitización  
B) Se rechazan automáticamente — los PDFs sin capa de texto son marcados como `NO_PROCESABLE` con motivo "PDF sin texto extraíble"  
C) Se procesan con advertencia — se intenta la extracción con Document AI; si falla, se marca la variable como confianza `BAJA` en U-03  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 4 — Modelo de Dominio U-02

### Pregunta 4.1 — Entidad `DocumentoPDF`

¿Qué datos específicos almacena la entidad `DocumentoPDF` generada durante la ingesta?

A) Mínimo: `id`, `cotizacion_id`, `nombre_archivo`, `s3_key`, `estado_sanitizacion`, `hash_sha256`, `tamaño_bytes`, `created_at`  
B) Lo de A + datos de sanitización: `regla_fallida` (RT1-RT4), `motivo_rechazo`, `es_escaneado`, `texto_extraido_ocr`  
C) Lo de A + metadatos del documento: `num_paginas`, `tiene_javascript`, `tiene_metadata_sospechosa`, `score_sanitizacion` (0-100)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.2 — Entidad de log de sanitización

¿Cómo se registra el resultado de cada regla de sanitización (RT1-RT4) para auditoría del CISO (US-19)?

A) En la misma tabla `DocumentoPDF` — campo `detalle_sanitizacion` tipo JSONB con el resultado de cada regla RT1-RT4  
B) Tabla separada `SanitizacionLog` — una fila por PDF con las 4 columnas: `rt1_resultado`, `rt2_resultado`, `rt3_resultado`, `rt4_resultado` + `timestamp`  
C) En `AuditTrail` — cada regla fallida genera su propio evento `PDF_SANITIZACION_FALLIDA` con el detalle en el campo `detalle`  
X) Otro (describa después del tag [Answer]:)

[Answer]: X
[Description]: En el `AuditTrail` (aprobado en U-01), utilizando la columna `metadata` (JSONB) para no alterar la estructura de base de datos. Se registran bajo dos eventos del catálogo:
1. `SANITIZACION_APLICADA`: Cuando el PDF es procesado con éxito, guardando en el JSONB el resultado de éxito de cada regla (RT1-RT4).
2. `PDF_ADVERSARIAL_DETECTADO`: Cuando el PDF falla por inyección o anomalías severas, guardando en el JSONB la regla fallida específica y el fragmento neutralizado para la auditoría del CISO.
---

### Pregunta 4.3 — Estado del Portafolio durante la ingesta

¿Qué estados puede tener un `Portafolio` durante el proceso de U-02?

A) 3 estados básicos: `PENDIENTE` → `PROCESANDO` → `LISTO_PARA_ANALISIS` (más `ERROR` si algo falla)  
B) 4 estados: `BORRADOR` → `INGESTA_EN_PROGRESO` → `SANITIZACION_EN_PROGRESO` → `LISTO_PARA_EXTRACCION` (más `ERROR` y `CANCELADO`)  
C) 5 estados completos del pipeline: `PENDIENTE` → `INGESTA` → `SANITIZACION` → `EXTRACCION` → `ANALISIS_COMPLETO` (más `ERROR` y `CANCELADO`)  
X) Otro (describa después del tag [Answer]:)

[Answer]: X
[Description]: Alineado con el ENUM `Portafolio.estado` aprobado en U-01, los estados involucrados durante el ciclo de vida de U-02 son:
1. `PENDIENTE`: Estado inicial al registrar el portafolio.
2. `INGESTA`: Transición activa mientras el pipeline realiza la descarga desde SAB y la sanitización Zero Trust (U-02).
3. `ANALISIS`: Estado al que transiciona al completar con éxito U-02, disparando el inicio de la extracción en U-03.
4. `ERROR`: Si el proceso de ingesta general falla catastróficamente.

---

## Sección 5 — Flujos de Negocio US-01, US-02, US-03

### Pregunta 5.1 — Flujo de inicio de análisis (US-01)

Cuando el Analista hace click en "Iniciar Análisis", ¿qué pasos ocurren exactamente antes de que el primer WebSocket empiece a llegar?

A) Flujo síncrono: `POST /api/v1/portafolios/{id}/analizar` → FastAPI conecta SAB → lee lista de cotizaciones → lanza BackgroundTask → responde 202  
B) Flujo con validación previa: `POST /analizar` → FastAPI valida permisos → valida que portafolio esté en estado `PENDIENTE` → conecta SAB → lanza BackgroundTask → 202  
C) Flujo con creación de portafolio: `POST /portafolios` (crea portafolio) → `POST /portafolios/{id}/analizar` (inicia análisis) → BackgroundTask → 202  
X) Otro (describa después del tag [Answer]:)

[Answer]: X
[Description]: El flujo es atómico y asíncrono:
1. El Analista llama a `POST /api/v1/procesos/{proceso_id}/analisis`.
2. El API Router de FastAPI valida permisos (rol `ANALISTA` o `ADMIN`) y verifica que no exista un análisis en curso para ese `proceso_id`.
3. Crea de forma síncrona el registro `Portafolio` local en estado `PENDIENTE` y responde inmediatamente un `202 Accepted` con el ID del portafolio.
4. Lanza una `BackgroundTask` (o tarea de asyncio) que corre en segundo plano.
5. En segundo plano, la tarea conecta a SAB (lectura read-only), descarga la lista de cotizaciones, cambia el estado del portafolio local a `INGESTA`, y dispara el primer evento WebSocket con el progreso inicial (ej: "Ingestando 0/N cotizaciones").

---

### Pregunta 5.2 — Límites de escala de U-02

Los criterios dicen "hasta 50 cotizaciones / 200 PDFs" (US-01). ¿Qué pasa si se supera ese límite?

A) Validación antes de iniciar — si el portafolio tiene más de 50 cotizaciones o 200 PDFs, el sistema rechaza el inicio con error 422 y un mensaje al Analista  
B) Procesamiento por lotes — el sistema acepta cualquier cantidad pero procesa en lotes de 50 cotizaciones, con pausas entre lotes  
C) Sin límite duro en MVP — 50/200 es una guía de capacidad esperada, no un límite técnico. Si el sistema puede, procesa todo  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 5.3 — Manejo del hash SHA-256 de PDFs

¿Para qué se usa el `hash_sha256` del `DocumentoPDF`?

A) Solo trazabilidad — es un campo de auditoría para verificar integridad del archivo en cualquier momento futuro  
B) Deduplicación activa — si un PDF con el mismo hash ya fue procesado en el mismo portafolio, se reutiliza el resultado y no se reprocesa  
C) Deduplicación global — si el mismo PDF ya fue procesado en CUALQUIER portafolio anterior, se reutiliza el resultado de sanitización  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 6 — Seguridad y Errores

### Pregunta 6.1 — EchoLeak: definición y detección

El criterio "0 EchoLeak" (Gate G4) es fundamental. ¿Cómo se define un EchoLeak en este sistema?

A) EchoLeak = cualquier output del LLM que **reproduce texto literal del PDF** que fue marcado como `ADVERSARIAL` — el PDF adversarial "filtró" su contenido a través del LLM al output final  
B) EchoLeak = cualquier output del LLM que **incluye instrucciones de sistema** (el prompt del sistema fue "leakeado" al proveedor)  
C) EchoLeak = cualquier respuesta del LLM que **contradice los datos del SAB** de forma sospechosa, sugiriendo manipulación del contexto  
X) Otro (describa después del tag [Answer]:)

[Answer]: X
[Description]: EchoLeak se define como la exfiltración o fuga de información confidencial interna en los outputs finales generados por el LLM (especialmente en los borradores de correo dirigidos a los proveedores). Esto incluye:
1. La fuga de las instrucciones y prompts de sistema (System Prompts).
2. La fuga de datos confidenciales del negocio (como precios, nombres o datos de cotizaciones de competidores o datos internos del SAB).
Se previene en U-02/U-03 mediante context isolation (nunca mezclar cotizaciones en el mismo prompt) y el pipeline Zero Trust.

---

### Pregunta 6.2 — Qué hacer cuando el SAB no está disponible

Si la conexión con el SAB falla al iniciar el análisis, ¿cuál es el comportamiento del sistema?

A) Retry con backoff exponencial — 3 reintentos con espera 1s, 2s, 4s; si todos fallan, portafolio queda en `ERROR` con evento en AuditTrail  
B) Error inmediato — `POST /analizar` devuelve 503 Service Unavailable al Analista con mensaje descriptivo  
C) Cola de reintentos — el análisis queda en `PENDIENTE_SAB` y el sistema reintenta automáticamente cada 5 minutos hasta conectar  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 3 artefactos de Functional Design de U-02.
