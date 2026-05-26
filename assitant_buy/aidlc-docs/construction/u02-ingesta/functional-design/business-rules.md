# Business Rules — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI

---

## Convenciones

- **BR-U02-XX**: Identificador único de regla de negocio de U-02 (continúa la numeración global después de BR-24 de U-01)
- **Severidad**: `CRITICA` (fallo bloquea el flujo) | `ALTA` (impacta la integridad) | `MEDIA` (afecta UX) | `BAJA` (validación de datos)
- **Refs**: Historia de usuario y requisito funcional del PRD que originó la regla

---

## Sección 1 — Límites y Validaciones de Entrada (C-01 IngestaConector)

### BR-U02-01: Límite de escala del portafolio (50 cotizaciones / 200 PDFs)
- **Severidad**: ALTA
- **Regla**: Antes de iniciar el pipeline de ingesta, el sistema valida que el proceso SAB no supere los límites del piloto:
  - Máximo **50 cotizaciones** por proceso.
  - Máximo **200 PDFs** en total entre todas las cotizaciones.
- **Acción**: Si se supera cualquier límite, el endpoint retorna HTTP 422 con cuerpo RFC 7807: `{"type": "/errors/limite-portafolio-excedido", "detail": "El proceso SAB-XXXX supera el límite de 50 cotizaciones (actual: N)"}`. El `Portafolio` no se crea.
- **Refs**: US-01 criterio de escala

### BR-U02-02: SAB accesible antes de iniciar
- **Severidad**: CRITICA
- **Regla**: Si la conexión read-only al SAB falla al momento de iniciar el análisis, el sistema retorna HTTP 503 al Analista con mensaje descriptivo. El registro `Portafolio` **no se crea** (no hay portafolios en estado `ERROR` sin haber podido leer el SAB).
- **Acción**: HTTP 503 `{"type": "/errors/sab-no-disponible", "detail": "Sistema SAB no disponible. Reintente en unos minutos."}`.
- **Nota**: No hay cola de reintentos automática. El Analista debe reintentar manualmente.
- **Refs**: 6.2-B, RF-01.1

### BR-U02-03: Proceso SAB debe existir y estar activo
- **Severidad**: CRITICA
- **Regla**: El `proceso_id` ingresado por el Analista debe existir en el SAB y tener estado activo. Si no existe o está cerrado, se retorna HTTP 404.
- **Acción**: HTTP 404 `{"type": "/errors/proceso-no-encontrado", "detail": "El proceso SAB-XXXX no existe o no está activo."}`.
- **Refs**: 1.3-A, RF-01.2

### BR-U02-04: No crear portafolio duplicado para el mismo proceso
- **Severidad**: ALTA
- **Regla**: Si ya existe un `Portafolio` local con el mismo `codigo_proceso` en estado distinto a `ERROR`, el sistema rechaza la creación con HTTP 409.
- **Excepción**: Si el portafolio anterior está en `ERROR`, el Analista puede forzar un nuevo análisis que sobrescriba el anterior (en MVP: crear uno nuevo, el anterior queda en estado `ERROR`).
- **Refs**: RF-01.3

---

## Sección 2 — Pipeline Zero Trust (C-02 SanitizadorZeroTrust)

### BR-U02-05: El sanitizador es middleware obligatorio — sin bypass
- **Severidad**: CRITICA
- **Regla**: **Ningún PDF puede llegar al `MotorExtraccion` (U-03) sin haber pasado por el `SanitizadorZeroTrust`**. El componente actúa como middleware a nivel de arquitectura, no como función opcional. No existe ninguna ruta de código que invoque al extractor sin pasar por sanitización.
- **Acción**: Violación de esta regla es un defecto de seguridad crítico (bloqueante en CI).
- **Refs**: RF-02.1, PRD Principio #3 (Zero Trust)

### BR-U02-06: Reglas de sanitización RT1-RT4 (definición exacta)
- **Severidad**: CRITICA
- Las 4 reglas del pipeline Zero Trust son **no negociables y no configurables**:

| Regla | Nombre | Qué detecta | Estado resultante si detecta |
|---|---|---|---|
| **RT1** | Metadatos maliciosos | Scripts JavaScript embebidos en metadatos PDF, enlaces externos activos, acciones automáticas de PDF | `ERROR` |
| **RT2** | Inyección de prompt | Texto que contenga patrones de instrucción al LLM (ej: "Ignore las instrucciones anteriores", patrones multi-idioma) | `ADVERSARIAL` |
| **RT3** | Contenido ofuscado | Texto invisible (color igual al fondo), caracteres homoglíficos Unicode, texto en capas no visibles, CSS oculto | `ADVERSARIAL` |
| **RT4** | Validación de estructura | PDF inválido o corrupto, tamaño > límite configurado, número de páginas excesivo | `ERROR` |

- **Refs**: US-03, RF-02.2, PRD §RT

### BR-U02-07: Política fail-closed por PDF individual
- **Severidad**: CRITICA
- **Regla**: Si un PDF no pasa la sanitización (estado `ADVERSARIAL` o `ERROR`), **la cotización asociada** queda con `estado_analisis = ERROR` y el pipeline continúa con las demás cotizaciones del portafolio. El portafolio **no se detiene** por PDFs individuales fallidos.
- **Excepción**: Si **todas** las cotizaciones de un portafolio tienen PDFs fallidos, el portafolio transiciona a `ERROR`.
- **Acción**: El WebSocket publica `PROGRESO_PIPELINE` con `{cotizacion_estado: "PDF_BLOQUEADO", motivo: "ADVERSARIAL|ERROR"}`.
- **Refs**: 2.3-B, RF-02.4

### BR-U02-08: Técnica Spotlighting con delimitadores dinámicos
- **Severidad**: ALTA
- **Regla**: Después de pasar las reglas RT1-RT4, **todo** PDF en estado `LIMPIO` recibe la técnica Spotlighting antes de que su texto sea enviado a cualquier LLM (en U-03). El proceso consiste en:
  1. Generar un token aleatorio único por request: `token = uuid4().hex[:8]` (ej: `a8f9c21d`).
  2. Envolver el texto extraído: `f"[PDF_{token}_START]\n{texto}\n[PDF_{token}_END]"`.
  3. Registrar el token en `PDFSanitizado.tokens_spotlighting`.
- **Propósito**: Impide ataques de escape donde el texto del PDF trate de "cerrar" los delimitadores (el atacante no puede conocer el token aleatorio).
- **Aplicación**: Siempre. No hay PDFs limpios sin Spotlighting.
- **Refs**: 2.2-A, RF-02.3, PRD M4

### BR-U02-09: Registro forense obligatorio en AuditTrail
- **Severidad**: ALTA
- **Regla**: Cada PDF procesado por el `SanitizadorZeroTrust` genera exactamente **un evento** en `AuditTrail`:
  - Estado `LIMPIO` → evento `SANITIZACION_APLICADA` con `metadata.resultado = "limpio"`.
  - Estado `ADVERSARIAL` → evento `PDF_ADVERSARIAL_DETECTADO` con `metadata.regla_fallida`, `metadata.fragmento_neutralizado` (máx 500 chars).
  - Estado `ERROR` → evento `SANITIZACION_APLICADA` con `metadata.resultado = "error"` y `metadata.motivo`.
- **Refs**: 4.2-X, US-19, RF-02.5

### BR-U02-10: Alerta de seguridad para PDFs ADVERSARIALES
- **Severidad**: CRITICA
- **Regla**: Cuando un PDF resulta en estado `ADVERSARIAL`, se publica **en tiempo real** un evento WebSocket de tipo `ALERTA_SEGURIDAD` dirigido únicamente a usuarios con rol `ADMIN` y `CISO` conectados.
- **Formato del evento**: `{tipo: "ALERTA_SEGURIDAD", payload: {portafolio_id, cotizacion_id, pdf_nombre, tipo_amenaza, timestamp}}`.
- **No hay notificación al Analista**: El Analista ve el estado del PDF como `bloqueado` en el dashboard, pero no recibe la alerta de seguridad forense (que es para el CISO).
- **Refs**: BR de U-01 (WebSocket), US-20, US-21

---

## Sección 3 — Manejo de PDFs del SAB (C-01 + C-12)

### BR-U02-11: Verificación de integridad SHA-256
- **Severidad**: ALTA
- **Regla**: Al recuperar un PDF del SAB, el sistema calcula el `hash_sha256` de los bytes recibidos y lo persiste en `DocumentoPDF.hash_sha256`. Este hash sirve como campo de auditoría de integridad y **no se usa para deduplicación** en MVP.
- **Propósito**: Permite verificar en cualquier momento futuro que el PDF almacenado no fue alterado tras su ingesta.
- **Refs**: 5.3-A, RF-01.5

### BR-U02-12: OCR en U-02 para PDFs escaneados — previo a sanitización
- **Severidad**: ALTA
- **Regla**: U-02 **sí distingue** entre PDFs con texto nativo y PDFs escaneados (imágenes). El flujo es:
  1. **Detección**: tras RT4, se comprueba si el PDF tiene capa de texto nativa (`pypdf`).
  2. **OCR** (solo si no hay texto nativo): `GatewayDocumentAI.ocr(pdf_bytes)` convierte la imagen a texto.
  3. **Almacenamiento del texto OCR**: el texto resultante se sube a S3 como companion file (`portafolios/{pid}/ocr/{pdf_id}.txt`). Se guarda solo la S3 key en `DocumentoPDF.texto_ocr_s3_key`. El texto OCR **no** se almacena en la BD para no inflar las filas.
  4. **Sanitización**: RT2 y RT3 se aplican sobre el texto OCR (igual que sobre el texto nativo).
- **Consecuencia si el OCR falla**: `DocumentoPDF` → `ERROR` (motivo: `"ocr_fallido"`). La cotización queda bloqueada. No se puede sanitizar un PDF del que no se puede extraer texto.
- **Reutilización en U-03**: U-03 no llama a Document AI para PDFs ya procesados; descarga el texto desde `texto_ocr_s3_key` directamente.
- **Campos nuevos en `DocumentoPDF`**: `es_escaneado: bool`, `texto_ocr_s3_key: str | None`.
- **Refs**: 3.2-A (corrige 3.1-B que era contradictorio)

### BR-U02-13: Tamaño máximo de PDF
- **Severidad**: ALTA
- **Regla**: La regla RT4 valida que ningún PDF supere el tamaño máximo configurado en la variable de entorno `PDF_MAX_SIZE_MB` (default: 50 MB). PDFs que superen este límite resultan en estado `ERROR` con motivo `"pdf_tamano_excedido"`.
- **Refs**: BR-U02-06 (RT4)

### BR-U02-14: Integración SAB — read-only en dos canales (BD y API REST con Keycloak)
- **Severidad**: CRITICA
- **Canal 1 — BD del SAB**: El usuario de base de datos configurado en `SAB_DB_URL` tiene permisos **exclusivamente de SELECT**. El sistema nunca emite sentencias `INSERT`, `UPDATE` o `DELETE` contra la BD del SAB.
- **Canal 2 — API REST del SAB con autenticación Keycloak**: La descarga de PDFs usa el flujo **OAuth2 Resource Owner Password Credentials (ROPC)** contra el Keycloak del SAB:
  1. El `GestorTokenSAB` obtiene un `access_token` haciendo `POST {SAB_KEYCLOAK_URL}/openid-connect/token` con `grant_type=password` y las credenciales del service account.
  2. El token se usa como `Authorization: Bearer {access_token}` en cada llamada de descarga.
  3. El token se renueva proactivamente si quedan menos de 60 segundos para su expiración (`expires_in=1200s`), usando el `refresh_token` cuando esté disponible.
- **Prerrequisito de despliegue**: El equipo de TI del SAB debe crear un **usuario service account en Keycloak** (funcionario) con privilegios de **solo lectura** (sin capacidad de crear, editar o aprobar procesos de compra). Esta cuenta es exclusiva para la integración con el Assistant Buy.
- **Gestión de secretos**:
  - `SAB_SERVICE_USERNAME`, `SAB_SERVICE_PASSWORD`, `SAB_CLIENT_SECRET` se almacenan exclusivamente como secretos de entorno. **Nunca se hardcodean en el código fuente.**
  - Estas credenciales **nunca se registran en logs, AuditTrail ni trazas de error**.
  - El `access_token` y `refresh_token` se mantienen solo en memoria durante la BackgroundTask (no se persisten).
- **Refs**: 1.1-A, RF-01.4, PRD §Seguridad


---

## Sección 4 — Flujo Asíncrono y WebSocket (US-01, US-02)

### BR-U02-15: Respuesta 202 inmediata al Analista
- **Severidad**: ALTA
- **Regla**: El endpoint `POST /api/v1/procesos/{proceso_id}/analisis` debe responder HTTP 202 en menos de 500ms tras crear el registro `Portafolio` local. Toda la lógica de conexión a SAB, descarga de PDFs y sanitización ocurre en la `BackgroundTask`.
- **Respuesta 202**: `{portafolio_id: UUID, proceso_id: str, estado: "PENDIENTE", mensaje: "Análisis iniciado. Conéctese al WebSocket para seguir el progreso."}`.
- **Refs**: 5.1-X, US-01, RF-01.1

### BR-U02-16: Publicación de progreso en WebSocket
- **Severidad**: MEDIA
- **Regla**: Durante el pipeline de ingesta, la `BackgroundTask` publica eventos de progreso al WebSocket en cada cotización procesada con el formato:
  ```json
  {
    "tipo": "PROGRESO_PIPELINE",
    "payload": {
      "portafolio_id": "...",
      "cotizacion_nombre": "Proveedor X",
      "cotizacion_actual": 3,
      "total_cotizaciones": 18,
      "pdfs_procesados": 7,
      "pdfs_total": 45,
      "estado_pdf": "LIMPIO|ADVERSARIAL|ERROR"
    }
  }
  ```
- **Refs**: US-02, RF-03.1

### BR-U02-17: Publicación de INGESTA_COMPLETADA al finalizar
- **Severidad**: ALTA
- **Regla**: Al terminar la ingesta de todas las cotizaciones, la `BackgroundTask` publica un evento final `INGESTA_COMPLETADA` y transiciona el `Portafolio` a estado `ANALISIS`:
  ```json
  {
    "tipo": "INGESTA_COMPLETADA",
    "payload": {
      "portafolio_id": "...",
      "total_cotizaciones": 18,
      "cotizaciones_con_pdfs_limpios": 16,
      "cotizaciones_bloqueadas": 2,
      "pdfs_limpios": 43,
      "pdfs_adversariales": 1,
      "pdfs_error": 1
    }
  }
  ```
- **Refs**: US-02, RF-03.2

---

## Sección 5 — Seguridad Zero Trust y EchoLeak

### BR-U02-18: Definición operacional de EchoLeak
- **Severidad**: CRITICA
- **Regla**: Un **EchoLeak** es cualquier fuga de información confidencial interna en los outputs generados por el LLM (especialmente borradores de correo). Incluye:
  1. Fuga de **System Prompts** del sistema (instrucciones de análisis internas).
  2. Fuga de **datos de otras cotizaciones** o del SAB hacia el exterior (un proveedor recibe datos de sus competidores).
- **Prevención en U-02**: El `SanitizadorZeroTrust` aplica Spotlighting (BR-U02-08) para marcar el contenido externo. La prevención completa de EchoLeak también requiere **aislamiento de contexto** en U-03 (nunca mezclar datos de diferentes cotizaciones en el mismo prompt).
- **Tasa objetivo**: 0 incidentes en producción (Gate G4).
- **Refs**: 6.1-X, RF-02.6, PRD §W1

### BR-U02-19: Aislamiento de contexto por cotización
- **Severidad**: CRITICA
- **Regla**: Cada PDF se procesa de forma completamente **aislada**. El pipeline nunca incluye datos de una cotización en el procesamiento de otra. Cada instancia de `MetadataOrigen` contiene solo los IDs de la cotización en proceso.
- **Refs**: BR-U02-18, RF-02.6

---

## Resumen de Reglas por Componente

| Componente | Reglas |
|---|---|
| C-01 IngestaConector | BR-U02-01, BR-U02-02, BR-U02-03, BR-U02-04, BR-U02-11, BR-U02-12, BR-U02-13, BR-U02-14, BR-U02-15, BR-U02-16, BR-U02-17 |
| C-02 SanitizadorZeroTrust | BR-U02-05, BR-U02-06, BR-U02-07, BR-U02-08, BR-U02-09, BR-U02-10, BR-U02-18, BR-U02-19 |
| C-12 RepositorioSAB | BR-U02-14 |
| C-16 GatewayDocumentAI | BR-U02-12 (✅ **activo en U-02**: OCR de PDFs escaneados antes de sanitización; el texto OCR se guarda en S3 para reutilizarlo en U-03) |
