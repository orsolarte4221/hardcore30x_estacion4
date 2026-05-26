# Plan de Functional Design — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-25  
> **Componentes**: C-08 AutenticadorOAuth, C-10 APIRest, C-13 RepositorioLocal, C-14 GestorWebSocket  
> **Story**: US-23 (Autenticación OAuth2)  

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta. Cuando termine todas, indíquelo y el workflow generará los artefactos de diseño funcional.

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Contexto cargado (unit-of-work.md + requirements + application-design)
- [x] Plan creado con preguntas
- [x] Respuestas recibidas y analizadas
- [x] Plan aprobado

### PART 2 — Generation
- [x] `business-logic-model.md` generado
- [x] `business-rules.md` generado
- [x] `domain-entities.md` generado

---

## Sección 1 — Modelo de Dominio (Entidades)

### Pregunta 1.1 — Esquema de Portafolio y Cotización

El núcleo del dominio es un **Portafolio** de compras que contiene múltiples **Cotizaciones** de proveedores. Cada Cotización tiene **PDFs** adjuntos. ¿Qué campo identifica un portafolio en SAB?

A) ID numérico del proceso de adquisición SAB — el sistema lo recibe como parámetro al iniciar el análisis  
B) Código alfanumérico del proceso (ej: `PROC-2026-001`) — más legible para el usuario  
C) Ambos — el sistema recibe el código alfanumérico y resuelve el ID internamente desde SAB  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.2 — Estado del análisis a nivel de Portafolio

El `Portafolio` en la BD local debe tener un estado que refleje el progreso del análisis. ¿Qué estados necesitamos?

A) 3 estados simples: `PENDIENTE` → `EN_PROCESO` → `COMPLETADO` (+ `ERROR` para fallos totales)  
B) 5 estados con granularidad: `PENDIENTE` → `INGESTA` → `ANALISIS` → `REVISION_HITL` → `COMPLETADO` (+ `ERROR`)  
C) 2 estados: `EN_PROCESO` | `COMPLETADO` — el detalle de progreso se maneja en memoria, no en BD  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.3 — Modelo de Variables Extraídas

`VariableExtraida` representa un dato extraído de un PDF. Además del nombre, valor, confianza y referencia de fuente, ¿necesita el modelo alguno de estos campos adicionales?

A) Solo los campos base: `nombre`, `valor`, `confianza`, `referencia_fuente`, `requiere_revision` — es suficiente para el MVP  
B) Agregar `categoria`: `VIGENCIA | GARANTIA | PENALIDAD | CONDICION_PAGO | EXCLUSION | OTRA` — facilita el filtrado en UI  
C) Agregar `categoria` + `unidad` (ej: "días", "%" para garantías) — más semántico pero más complejo  
X) Otro (describa después del tag [Answer]:)

[Answer]: `nombre`, `valor`, `confianza`, `referencia_fuente`, `requiere_revision` , más un campo analisis que es una descripción del agente sobre el dato

---

### Pregunta 1.4 — Modelo de Discrepancia

`Discrepancia` compara un campo de SAB con lo encontrado en el PDF. ¿Qué tipos de discrepancia existen?

A) 3 tipos: `VALOR_DIFERENTE` (dato existe en ambos pero difiere) | `AUSENTE_EN_PDF` (SAB tiene el dato, PDF no) | `SOLO_EN_PDF` (PDF tiene dato que SAB no registra)  
B) Solo 2 tipos: `VALOR_DIFERENTE` | `CAMPO_FALTANTE` (agrupa "ausente en PDF" y "solo en PDF")  
C) Los tipos se derivan automáticamente de los campos nulos — sin enum explícito  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

## Sección 2 — Autenticación y Autorización (C-08)

### Pregunta 2.1 — Proveedor OAuth2 activo en el MVP

¿Con qué proveedor OAuth2 debe integrarse el MVP?

A) Solo Google Workspace — las universidades usan G Suite institucional  
B) Solo Microsoft Azure AD — las universidades usan Microsoft 365  
C) Ambos — el sistema detecta el proveedor según el dominio del email del usuario  
D) Configurable — el proveedor se define por variable de entorno; el MVP prueba con uno, producción puede cambiar  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Asignación de roles

Una vez autenticado el usuario, ¿cómo se determina su rol (Analista / Admin / CISO / Comité)?

A) Tabla de roles en PostgreSQL local — un Admin configura qué email tiene qué rol en la UI de admin  
B) Claims del token OAuth2 — el proveedor (Google/Microsoft) incluye el grupo o rol en el token  
C) Archivo de configuración `.env` — lista estática de emails → rol para el piloto  
D) Híbrido — Claims del token OAuth2 como fuente primaria, tabla local como override para casos especiales  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.3 — Duración de la sesión

¿Cuánto debe durar una sesión activa antes de requerir re-autenticación?

A) 8 horas (jornada laboral completa) — el usuario no necesita autenticarse de nuevo durante el día  
B) 4 horas con refresh automático — refresh token renueva la sesión silenciosamente si el usuario está activo  
C) 1 hora con refresh — más seguro, apropiado para el rol CISO/Admin  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 2.4 — Permisos por rol (matriz de acceso)

¿Cuál de estas matrices de permisos refleja mejor el modelo de negocio?

A) Estándar según PRD:
   - **Analista**: iniciar análisis, ver resultados, gestionar aclaraciones, generar reportes
   - **Admin**: ver logs, ver alertas, configurar sistema (polling IMAP, umbral de confianza)
   - **CISO**: ver log de sanitización, ver alertas de seguridad (solo lectura)
   - **Comité**: ver reporte comparativo (solo lectura)

B) Extendido — Admin puede también ver resultados de análisis y reportes (además de logs)  
C) Simplificado — CISO y Admin son el mismo rol con nombre diferente  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 3 — Esquema de Base de Datos (C-13)

### Pregunta 3.1 — Estrategia de IDs

¿Qué tipo de ID primario se usa en todas las tablas de la BD local?

A) `UUID v4` — globalmente único, sin secuencias, adecuado para sistemas distribuidos futuros  
B) `BIGSERIAL` (autoincremental) — más simple, menor overhead en PostgreSQL, suficiente para monolito  
C) `UUID v7` — UUID ordenado por tiempo, mejor rendimiento en índices que UUID v4  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 3.2 — Estrategia de soft delete

Para las entidades principales (Portafolio, Cotizacion, Aclaracion), ¿se usa borrado lógico o físico?

A) Soft delete (`deleted_at TIMESTAMP NULL`) — los registros nunca se borran físicamente, facilita el audit trail  
B) Borrado físico — si un portafolio se cancela, se borran los datos; el audit trail es independiente  
C) No aplica para MVP — no habrá funcionalidad de borrado expuesta al usuario en v1  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 3.3 — Tabla de AuditTrail

El `AuditTrail` debe ser append-only (nunca UPDATE/DELETE). ¿Qué eventos se registran?

A) Solo eventos críticos: login/logout, inicio de análisis, aprobación de aclaraciones, generación de reportes  
B) Todos los eventos del sistema: todo lo de A + cambios de estado de portafolio, cada PDF procesado, cada variable extraída guardada  
C) Solo eventos de seguridad: login/logout + accesos denegados + sanitizaciones con elementos adversariales  
X) Otro (describe qué eventos registrarías)

[Answer]: B

---

## Sección 4 — API REST Base (C-10)

### Pregunta 4.1 — Formato de respuesta de error

¿Cuál es el formato estándar de error que devuelve la API?

A) RFC 7807 Problem Details:
   ```json
   {"type": "string", "title": "string", "status": 400, "detail": "string", "instance": "string"}
   ```

B) Formato propio simple:
   ```json
   {"error": "codigo_error", "message": "descripción legible", "timestamp": "ISO8601"}
   ```

C) FastAPI default — el framework maneja el formato de errores de validación automáticamente, sin personalización  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.2 — Paginación de listas

Las listas de cotizaciones, discrepancias, y aclaraciones pueden ser largas. ¿Qué estrategia de paginación se usa?

A) Offset/Limit clásico: `?page=1&page_size=20` — más simple, adecuado para MVP  
B) Cursor-based: `?cursor=<token>&limit=20` — más eficiente para listas grandes pero más complejo  
C) No se pagina en MVP — los portafolios son de máximo 50 cotizaciones y los resultados caben en una página  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.3 — Versioning de la API

¿Cómo se versiona la API REST?

A) URL path: `/api/v1/` — estándar, fácil de entender, ya incluido en el diseño  
B) Header: `Accept: application/vnd.assistent-buy.v1+json` — más REST-puro pero más complejo  
C) Sin versioning explícito para MVP — se agrega en v1.1 cuando haya múltiples clientes  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 5 — WebSocket (C-14)

### Pregunta 5.1 — Eventos WebSocket

¿Qué tipos de eventos publica el `GestorWebSocket`? La propuesta base es:

```json
{"tipo": "PROGRESO_ANALISIS", "payload": {"portafolio_id": "...", "cotizacion": 12, "total": 18}, "timestamp": "ISO8601"}
{"tipo": "PDF_PROCESADO", "payload": {"pdf_nombre": "...", "estado": "LIMPIO"}, "timestamp": "ISO8601"}
{"tipo": "RESPUESTA_PROVEEDOR", "payload": {"aclaracion_id": "...", "proveedor": "..."}, "timestamp": "ISO8601"}
{"tipo": "ERROR_PROCESAMIENTO", "payload": {"cotizacion_id": "...", "motivo": "..."}, "timestamp": "ISO8601"}
```

A) Aprobado — estos 4 tipos cubren el MVP  
B) Agregar `ANALISIS_COMPLETADO` como evento final que resume el resultado  
C) Agregar `ALERTA_SEGURIDAD` para notificar al CISO de elementos adversariales detectados en tiempo real  
D) B + C — agregar ambos eventos adicionales  
X) Otro (describa después del tag [Answer]:)

[Answer]: D

---

### Pregunta 5.2 — Reconexión WebSocket

Si el cliente pierde la conexión WebSocket, ¿cómo se recupera?

A) El cliente reintenta automáticamente con backoff exponencial — sin estado del lado servidor  
B) El servidor retiene los últimos N eventos (ej: 50) en memoria — el cliente los recibe al reconectarse  
C) Sin gestión de reconexión en MVP — si se pierde la conexión, el usuario refresca la página  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 3 artefactos de Functional Design de U-01.
