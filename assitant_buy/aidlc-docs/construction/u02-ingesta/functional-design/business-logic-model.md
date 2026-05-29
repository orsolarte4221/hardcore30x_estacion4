# Business Logic Model — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI

> ⚠️ **DELTAS PENDIENTES — APLICAR ANTES DE CODE GENERATION**
>
> Los flujos descritos abajo cambian tras la revisión de U-03 (2026-05-28). Ver [`../../u03-extraccion/functional-design/cross-unit-deltas.md`](../../u03-extraccion/functional-design/cross-unit-deltas.md) **§2** para detalle completo.
>
> **Cambios resumidos en flujos**:
> - Trigger: nuevo endpoint `POST /portafolios/from-sab` con tripleta `(version, fecha_version, id)` de `solicitud_bienes`
> - Flujo `RepositorioSAB`: queries directas a `solicitud_bienes`, `solicitud_cotizacion`, `cotizacion_item`, etc. (no se llama al endpoint Excel)
> - Flujo `ParserExcelSAB`: ❌ ELIMINADO
> - Flujo `GatewaySAB`: solo descarga de binarios por `doc_id`
> - Persistencia jerárquica en transacción única
>
> **Aplicación**: paso 1 del plan de Code Generation de U-02.

---

## Descripción General

U-02 Ingesta y Sanitización implementa el punto de entrada del pipeline de análisis de portafolios. Tiene dos responsabilidades críticas:

1. **IngestaConector (C-01 + C-12)**: Conecta al SAB (read-only), recupera las cotizaciones y sus PDFs, y los almacena de forma segura en S3.
2. **SanitizadorZeroTrust (C-02)**: Aplica el pipeline de seguridad Zero Trust (RT1-RT4) y la técnica Spotlighting sobre cada PDF antes de permitir su paso al motor de extracción (U-03).

El resultado es el objeto `PDFSanitizado`, que es la **interfaz de contrato** entre U-02 y U-03.

---

## Módulo 1: IngestaConector (C-01) — Flujos de Negocio

### 1.1 Flujo Principal: Iniciar Análisis de Portafolio (US-01)

```
POST /api/v1/procesos/{proceso_id}/analisis
    │
    ▼
[C-10 APIRest] Valida permisos (rol ANALISTA o ADMIN) → 403 si no autorizado
    │
    ▼
[C-10] Valida que no exista Portafolio activo (estado != ERROR) para proceso_id → 409 si existe (BR-U02-04)
    │
    ▼
[C-01] Verificación de disponibilidad SAB (conexión de prueba, timeout 5s)
    │
    ├─ SAB no responde → HTTP 503 (BR-U02-02)
    │
    ▼
[C-12 RepositorioSAB] Consulta sab_procesos WHERE proceso_id = {proceso_id} AND activo = TRUE
    │
    ├─ No encontrado → HTTP 404 (BR-U02-03)
    │
    ▼
[C-12] Consulta total_cotizaciones y total_pdfs del proceso
    │
    ├─ total_cotizaciones > 50 → HTTP 422 (BR-U02-01)
    ├─ total_pdfs > 200 → HTTP 422 (BR-U02-01)
    │
    ▼
[C-01] Crea Portafolio local:
    ├── id = generate_uuid_v7()
    ├── codigo_proceso = proceso_id
    ├── estado = PENDIENTE
    ├── analista_id = usuario_actual.id
    └── total_cotizaciones = N (leído del SAB)
    │
    ▼
[C-13 RepositorioLocal] INSERT Portafolio
    │
    ▼
[AuditTrail] registrar PORTAFOLIO_CREADO + ANALISIS_INICIADO
    │
    ▼
[BackgroundTask] Lanza pipeline_ingesta(portafolio_id, proceso_id, analista_id)
    │
    ▼
HTTP 202 Accepted {portafolio_id, proceso_id, estado: "PENDIENTE", ws_url: "/ws/{portafolio_id}"}
```

---

### 1.2 Flujo de la BackgroundTask: Pipeline de Ingesta Completo

```
pipeline_ingesta(portafolio_id, proceso_id, analista_id)
    │
    ▼
[C-01] actualizar_estado_portafolio(portafolio_id, INGESTA)
[AuditTrail] PORTAFOLIO_ESTADO_CAMBIADO {de: PENDIENTE, a: INGESTA}
[WebSocket] PROGRESO_PIPELINE {estado: "iniciando_ingesta", cotizacion_actual: 0, total: N}
    │
    ▼
[C-12] listar_cotizaciones(proceso_id) → list[CotizacionSAB]
    │
    ▼
PARA CADA cotizacion_sab EN cotizaciones_sab:
    │
    ├─ [C-01] Crear Cotizacion local:
    │      ├── id = generate_uuid_v7()
    │      ├── portafolio_id = portafolio_id
    │      ├── nombre_proveedor = cotizacion_sab.nombre_proveedor
    │      ├── email_proveedor = cotizacion_sab.email_proveedor
    │      └── estado_analisis = PENDIENTE
    │  [C-13] INSERT Cotizacion
    │
    ├─ [C-12] obtener_rutas_pdf(cotizacion_sab.id_sab) → list[RutaPDFSAB]
    │
    ├─ PARA CADA ruta_pdf EN rutas_pdf:
    │      │
    │      ├─ [C-01] recuperar_pdf(ruta_pdf)
    │      │      │
    │      │      ├─ Leer bytes desde filesystem SAB
    │      │      ├─ Calcular hash_sha256(bytes) (BR-U02-11)
    │      │      ├─ Validar tamano_bytes <= PDF_MAX_SIZE_MB (BR-U02-13)
    │      │      │      └─ Si excede → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="tamano_excedido")
    │      │      └─ Upload bytes a S3 → ruta_s3
    │      │
    │      ├─ [C-13] Crear DocumentoPDF:
    │      │      ├── estado_sanitizacion = PENDIENTE
    │      │      └── hash_sha256, tamano_bytes, ruta_almacenamiento = ruta_s3
    │      │
    │      ├─ [C-02 SanitizadorZeroTrust] sanitizar(pdf_bytes, metadata_origen)
    │      │      └─ (ver Módulo 2)
    │      │
    │      ├─ [C-13] actualizar DocumentoPDF.estado_sanitizacion = resultado.estado
    │      │
    │      └─ [WebSocket] PROGRESO_PIPELINE {cotizacion_actual, estado_pdf}
    │
    ├─ Si todos los PDFs de cotizacion son ADVERSARIAL o ERROR:
    │      └─ actualizar Cotizacion.estado_analisis = ERROR
    │
    └─ Actualizar Portafolio.cotizaciones_procesadas++
    │
FIN PARA CADA
    │
    ▼
Si al menos 1 cotizacion tiene PDFs LIMPIOS:
    [C-01] actualizar_estado_portafolio(portafolio_id, ANALISIS)
    [AuditTrail] PORTAFOLIO_ESTADO_CAMBIADO {de: INGESTA, a: ANALISIS}
    [WebSocket] INGESTA_COMPLETADA {resumen de PDFs}

Si TODAS las cotizaciones están en estado ERROR:
    [C-01] actualizar_estado_portafolio(portafolio_id, ERROR)
    [AuditTrail] PORTAFOLIO_ESTADO_CAMBIADO {de: INGESTA, a: ERROR}
    [WebSocket] INGESTA_ERROR {motivo: "todas_cotizaciones_bloqueadas"}
```

---

### 1.3 Flujo de Recuperación de PDF Individual

> ⚠️ **Diseño actualizado**: Los PDFs se descargan vía el API REST del SAB (`GET /api/v1/cotizacion/documentos/descargar/{documento_id}`), no desde el filesystem. Esto es necesario porque el Assistant Buy se ejecuta en una nube distinta a la del SAB.

```
C-01.recuperar_pdf(doc_pdf_sab: DocumentoPDFSAB, portafolio_id, pdf_id_local)
    │
    ▼
HTTP GET {SAB_BASE_URL}/api/v1/cotizacion/documentos/descargar/{doc_pdf_sab.documento_id}
    Authorization: Bearer {SAB_API_TOKEN}
    Timeout: SAB_DOWNLOAD_TIMEOUT_S (default: 30s)
    │
    ├─ HTTP 404 → ResultadoRecuperacionPDF(estado=NO_ENCONTRADO)
    ├─ HTTP 401/403 → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_auth_error") + log ERROR
    ├─ Timeout / Connection Error → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_timeout")
    ├─ HTTP 5xx → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_server_error")
    │
    ▼ (HTTP 200 OK)
Recibir bytes del PDF (streaming si > 10MB para no acumular en memoria)
    │
    ▼
Validar tamano_bytes <= PDF_MAX_SIZE_MB
    │
    ├─ Excede límite → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="tamano_excedido")
    │
    ▼
Calcular hash_sha256(bytes)
    │
    ▼
Upload a S3 con key: f"portafolios/{portafolio_id}/pdfs/{pdf_id_local}.pdf"
    │
    ├─ Upload falla → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="upload_s3_fallido")
    │
    ▼
ResultadoRecuperacionPDF(
    estado = OK,
    pdf_bytes = bytes,
    nombre_archivo = doc_pdf_sab.nombre_archivo,
    hash_sha256 = hash,
    tamano_bytes = len(bytes),
    ruta_s3 = s3_key
)
```

---

## Módulo 2: SanitizadorZeroTrust (C-02) — Pipeline Zero Trust

### 2.1 Flujo Principal de Sanitización

> **Cambio de orden**: RT4 (validación de estructura) se ejecuta **primero** para evitar desperdiciar llamadas a Document AI en PDFs corruptos o sobredimensionados. Después, si el PDF es escaneado, se hace OCR antes de aplicar RT1/RT2/RT3.

```
C-02.sanitizar(pdf_bytes, metadata_origen)
    │
    ▼
╔═══ ETAPA 0: Regla RT4 — Validación de Estructura (PRIMERO) ══╗
│                                                               │
│ [C-02] Validar integridad del PDF:                            │
│     │                                                         │
│     ├─ PDF inválido/corrupto → RT4: FAIL → estado=ERROR       │
│     ├─ Tamaño > PDF_MAX_SIZE_MB → RT4: FAIL → estado=ERROR    │
│     └─ Estructura válida → RT4: OK                            │
╚═══════════════════════════════════════════════════════════════╝
    │
    ▼ (solo si RT4 OK)
╔═══ ETAPA 1: Detección de capa de texto + OCR si necesario ═══╗
│                                                               │
│ [C-02] texto = pypdf.extract_text(pdf_bytes)                  │
│     │                                                         │
│     ├─ len(texto.strip()) > 0 → texto NATIVO disponible       │
│     │      es_escaneado = FALSE                               │
│     │                                                         │
│     └─ len(texto.strip()) == 0 → PDF escaneado (imagen)       │
│            es_escaneado = TRUE                                │
│            │                                                  │
│            ▼ [C-16] GatewayDocumentAI → Amazon Textract       │
│              aiobotocore: textract.detect_document_text()      │
│              (o start_document_text_detection para > 5 págs)  │
│            │                                                  │
│            ├─ OCR falla → estado=ERROR, motivo="ocr_fallido"  │
│            │                                                  │
│            └─ OCR OK → texto = texto_ocr                      │
│                 Upload S3: portafolios/{pid}/ocr/{pdf_id}.txt  │
│                 DocumentoPDF.texto_ocr_s3_key = s3_key         │
╚═══════════════════════════════════════════════════════════════╝
    │
    ▼ (solo si RT4 OK y texto disponible)
╔═══ ETAPA 2: Regla RT1 — Metadatos Maliciosos ════════════════╗
│                                                               │
│ [C-02] Parsear estructura PDF (sin ejecutar)                  │
│     │                                                         │
│     ├─ Detectar /JavaScript en metadata → RT1: FAIL → ERROR   │
│     ├─ Detectar /AA (auto-action) → RT1: FAIL → ERROR         │
│     ├─ Detectar /URI con dominio externo → RT1: FAIL → ERROR  │
│     └─ Sin elementos maliciosos → RT1: OK                     │
╚═══════════════════════════════════════════════════════════════╝
    │
    ▼ (solo si RT4+RT1 OK)
╔═══ ETAPA 3: Regla RT2 — Inyección de Prompt ══════════════════╗
│                                                                │
│ [C-02] Analizar texto (nativo o OCR) normalizado:             │
│     texto_normalizado = unicodedata.normalize('NFKC', texto)  │
│     │                                                          │
│     ├─ Detectar patrones de instrucción al LLM:               │
│     │    - "Ignore previous instructions" (multi-idioma)      │
│     │    - "System:", "Assistant:", "Human:"                  │
│     │    - Caracteres de control especiales                   │
│     │                                                          │
│     ├─ Patrón detectado → RT2: FAIL → estado=ADVERSARIAL      │
│     └─ Sin patrones → RT2: OK                                 │
╚════════════════════════════════════════════════════════════════╝
    │
    ▼ (solo si RT4+RT1+RT2 OK)
╔═══ ETAPA 4: Regla RT3 — Contenido Ofuscado ═══════════════════╗
│                                                                │
│ [C-02] Análisis de rendering y texto (nativo o OCR):          │
│     │                                                          │
│     ├─ Detectar texto invisible (color = fondo)               │
│     ├─ Detectar caracteres homoglíficos Unicode               │
│     ├─ Detectar texto en capas no visibles                    │
│     ├─ Detectar texto con tamaño 0pt o 1pt                    │
│     │                                                          │
│     ├─ Ofuscación detectada → RT3: FAIL → estado=ADVERSARIAL  │
│     └─ Sin ofuscación → RT3: OK                               │
╚════════════════════════════════════════════════════════════════╝
    │
    ▼ (solo si RT4+RT1+RT2+RT3 OK)
╔═══ ETAPA 5: Spotlighting ═════════════════════════════════════╗
│                                                               │
│ [C-02] aplicar_spotlighting(texto, metadata_origen)           │
│     │                                                         │
│     ├─ Generar token: token = uuid4().hex[:8]                 │
│     ├─ Envolver texto:                                        │
│     │    "[PDF_{token}_START]\n{texto}\n[PDF_{token}_END]"    │
│     └─ Retornar: texto_con_spotlighting, [token]              │
╚═══════════════════════════════════════════════════════════════╝
    │
    ▼
╔═══ RESULTADO FINAL ════════════════════════════════════════════╗
│                                                               │
│ Construir PDFSanitizado:                                      │
│   - estado: LIMPIO / ADVERSARIAL / ERROR                      │
│   - contenido_bytes: bytes si LIMPIO, None si no              │
│   - tokens_spotlighting: [token] si LIMPIO        │
│   - elementos_detectados: [] o [ElementoAdversarial]
│                                                   │
│ Registrar en AuditTrail (BR-U02-09)               │
│ Si ADVERSARIAL → WebSocket ALERTA_SEGURIDAD (BR-U02-10)
╚═══════════════════════════════════════════════════╝
    │
    ▼
Retornar PDFSanitizado
```

---

### 2.2 Flujo de Detección de RT2 (Inyección de Prompt)

```
C-02._detectar_inyeccion_prompt(texto_extraido) → list[ElementoAdversarial]
    │
    ▼
Compilar patrones de detección multi-idioma:
    - Español: "ignora las instrucciones anteriores", "eres ahora"
    - Inglés: "ignore previous instructions", "you are now", "system:"
    - Árabe/Chino: patrones equivalentes (normalización Unicode)
    - Patrones de roleplay: "act as", "pretend you are"
    - Patrones de delimitadores maliciosos: "[SYSTEM]", "###", "---"
    │
    ▼
Aplicar regex en modo case-insensitive sobre texto_normalizado
    │
    ├─ Sin matches → retornar []
    │
    ├─ Match encontrado:
    │      ├─ Extraer fragmento (máx 500 chars alrededor del match)
    │      └─ retornar [ElementoAdversarial(regla="RT2", tipo="inyeccion_prompt", fragmento)]
    │
    ▼
IMPORTANTE: Los patrones de detección se cargan desde configuración,
no están hardcodeados, para permitir actualización sin re-deploy.
```

---

### 2.3 Flujo de Spotlighting con Delimitadores Dinámicos

```
C-02.aplicar_spotlighting(contenido, metadata_origen) → str
    │
    ▼
Generar token único por request:
    token = uuid4().hex[:8]  # Ej: "a8f9c21d"
    │
    ▼
Construir texto con delimitadores:
    resultado = (
        f"[PDF_{token}_START]\n"
        f"FUENTE: Cotización de {metadata_origen.nombre_proveedor}\n"
        f"PROCESO: {metadata_origen.proceso_id}\n"
        f"ARCHIVO: {metadata_origen.nombre_archivo}\n"
        f"---\n"
        f"{contenido}\n"
        f"[PDF_{token}_END]"
    )
    │
    ▼
Registrar token en PDFSanitizado.tokens_spotlighting = [f"PDF_{token}"]
    │
    ▼
Retornar resultado
```

---

### 2.4 Flujo de Registro en AuditTrail y Alerta WebSocket

```
C-02._registrar_resultado(pdf_id, cotizacion_id, portafolio_id, estado, elementos)
    │
    ▼
Si estado == LIMPIO:
    AuditTrail.registrar({
        evento: "SANITIZACION_APLICADA",
        entidad_tipo: "DocumentoPDF",
        entidad_id: pdf_id,
        metadata: {
            resultado: "limpio",
            rt1: "OK", rt2: "OK", rt3: "OK", rt4: "OK"
        }
    })

Si estado == ADVERSARIAL:
    elem = elementos[0]  # El primer elemento detectado es el más crítico
    AuditTrail.registrar({
        evento: "PDF_ADVERSARIAL_DETECTADO",
        entidad_tipo: "DocumentoPDF",
        entidad_id: pdf_id,
        metadata: {
            resultado: "adversarial",
            regla_fallida: elem.regla,
            tipo_amenaza: elem.tipo_amenaza,
            fragmento_neutralizado: elem.fragmento_neutralizado,
            nombre_proveedor: metadata_origen.nombre_proveedor
        }
    })
    
    GestorWebSocket.publicar(portafolio_id, {
        tipo: "ALERTA_SEGURIDAD",
        payload: {
            portafolio_id, cotizacion_id, pdf_nombre,
            tipo_amenaza: elem.tipo_amenaza, timestamp
        }
    })
    # Solo a usuarios ADMIN y CISO (BR-U02-10)

Si estado == ERROR:
    AuditTrail.registrar({
        evento: "SANITIZACION_APLICADA",
        entidad_tipo: "DocumentoPDF",
        entidad_id: pdf_id,
        metadata: {
            resultado: "error",
            regla_fallida: elementos[0].regla if elementos else "DESCONOCIDO",
            motivo: elementos[0].tipo_amenaza if elementos else "pdf_corrupto"
        }
    })
```

---

## Módulo 3: RepositorioSAB (C-12) — Acceso Read-Only + Autenticación Keycloak

### 3.0 GestorTokenSAB — Autenticación OAuth2 ROPC contra Keycloak

> **Contexto**: El SAB usa Keycloak como Identity Provider. El login se realiza con `grant_type=password` (OAuth2 Resource Owner Password Credentials). El `access_token` dura **1200 segundos (20 min)**, por lo que el sistema debe gestionarlo proactivamente.
>
> **Solución**: Se crea un `GestorTokenSAB` (singleton en memoria, parte de C-12) que mantiene la sesión activa y renueva el token automáticamente antes de que expire, sin intervención manual.

```
GestorTokenSAB — Ciclo de vida del token

Al iniciar la BackgroundTask (primera llamada a la API del SAB):
    │
    ▼
_session = None  # No hay token aún
    │
    ▼
GestorTokenSAB.obtener_token() → str (access_token vigente)
    │
    ├─ Si _session es None o _session.expires_at - now() < 60s:
    │      │
    │      ├─ Si _session existe Y _session.refresh_expires_at > now():
    │      │      └─ Intentar renovar con refresh_token (más eficiente)
    │      │             POST {SAB_KEYCLOAK_URL}/openid-connect/token
    │      │               grant_type=refresh_token
    │      │               refresh_token={_session.refresh_token}
    │      │               client_id={SAB_CLIENT_ID}
    │      │               client_secret={SAB_CLIENT_SECRET}
    │      │
    │      └─ Si no hay refresh_token válido:
    │             POST {SAB_KEYCLOAK_URL}/openid-connect/token
    │               grant_type=password
    │               client_id={SAB_CLIENT_ID}
    │               client_secret={SAB_CLIENT_SECRET}
    │               username={SAB_SERVICE_USERNAME}
    │               password={SAB_SERVICE_PASSWORD}
    │             │
    │             ├─ HTTP 401 → ERROR: credenciales inválidas
    │             │              Log CRÍTICO (sin revelar password)
    │             │              Portafolio → ERROR
    │             ├─ HTTP 5xx o timeout → ERROR: Keycloak no disponible
    │             └─ HTTP 200 → Construir TokenSABSession:
    │                    expires_at = now() + timedelta(seconds=1200)
    │                    refresh_expires_at = now() + timedelta(seconds=1800)
    │                    _session = TokenSABSession(...)
    │
    └─ Si _session.expires_at - now() >= 60s:
           └─ Retornar _session.access_token (vigente, sin renovar)
```

**Seguridad crítica**:
- Las credenciales (`SAB_SERVICE_USERNAME`, `SAB_SERVICE_PASSWORD`, `SAB_CLIENT_SECRET`) **nunca se registran en logs, AuditTrail ni trazas de error**.
- El `access_token` y `refresh_token` se mantienen **solo en memoria** durante la BackgroundTask. No se persisten en BD ni en S3.
- Si el `GestorTokenSAB` falla en obtener un token válido, la BackgroundTask termina con error y el `Portafolio` queda en estado `ERROR`.

---

### 3.1 Consultas al SAB

```
C-12.listar_cotizaciones(proceso_id) → list[CotizacionSAB]
    │
    ▼
SELECT
    c.id_sab,
    p.nombre AS nombre_proveedor,
    p.email AS email_proveedor,
    c.numero_oferta
FROM sab_cotizaciones c
JOIN sab_proveedores p ON c.proveedor_id = p.id
WHERE c.proceso_id = :proceso_id
ORDER BY p.nombre ASC
    │
    ▼
PARA CADA cotizacion:
    SELECT documento_id, nombre_archivo
    FROM sab_documentos_pdf
    WHERE cotizacion_id = :cotizacion_id
    ORDER BY nombre_archivo
    │
    ▼
Construir list[CotizacionSAB] con documentos_pdf anidados
    (url_descarga = f"{SAB_BASE_URL}/api/v1/cotizacion/documentos/descargar/{documento_id}")
```

**Restricción**: Todas las consultas usan un usuario de BD con permisos solo de `SELECT`. No se emiten sentencias de escritura. La **descarga del contenido** del PDF se realiza vía HTTP contra el API REST del SAB (ver §1.3), autenticada con el token Keycloak obtenido por el `GestorTokenSAB` (ver §3.0).

### 3.2 Flujo integrado: obtener token + descargar PDF

```
C-01.recuperar_pdf(doc_pdf_sab, portafolio_id, pdf_id_local)
    │
    ▼
access_token = GestorTokenSAB.obtener_token()  # §3.0: renueva si expira en < 60s
    │
    ▼
HTTP GET {doc_pdf_sab.url_descarga}
  Authorization: Bearer {access_token}
  Timeout: SAB_DOWNLOAD_TIMEOUT_S
    │
    ├─ HTTP 401 → Token expirado/inválido:
    │      GestorTokenSAB.invalidar_sesion()
    │      Reintentar UNA vez (obtener_token() forzará re-login)
    │      Si vuelve a fallar → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_auth_error")
    │
    ├─ HTTP 404 → ResultadoRecuperacionPDF(estado=NO_ENCONTRADO)
    ├─ Timeout   → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_timeout")
    ├─ HTTP 5xx  → ResultadoRecuperacionPDF(estado=CORRUPTO, motivo="sab_server_error")
    │
    └─ HTTP 200 → continúa pipeline (ver §1.3)
```

---


## Módulo 4: Interacciones entre Módulos (Diagrama de Flujo Completo)

```
Analista → POST /api/v1/procesos/{proceso_id}/analisis
                │
                │ (síncrono, < 500ms)
                ▼
        ┌───────────────────┐
        │   C-10 APIRest    │ Valida permisos + crea Portafolio(PENDIENTE)
        │                   │ Lanza BackgroundTask
        └─────────┬─────────┘
                  │
                  │ HTTP 202 ←─────────────────────── Analista
                  │
                  │ (asíncrono, en background)
                  ▼
        ┌──────────────────────────────────────────────────────────┐
        │                    BackgroundTask                        │
        │                                                          │
        │  C-01 ────────→ C-12 (SAB read-only)                   │
        │  IngestaConector    RepositorioSAB                      │
        │       │                │                                 │
        │       │←── CotizacionSAB[]                              │
        │       │                                                  │
        │  PARA CADA cotizacion:                                  │
        │       │                                                  │
        │       ├─ Crear Cotizacion local (C-13)                  │
        │       │                                                  │
        │       ├─ Recuperar PDF bytes (C-01 → SAB filesystem)    │
        │       │                                                  │
        │       ├─ Upload S3                                       │
        │       │                                                  │
        │       ├─ Crear DocumentoPDF(PENDIENTE) (C-13)           │
        │       │                                                  │
        │       ├─ C-02 SanitizadorZeroTrust.sanitizar()          │
        │       │       │                                          │
        │       │       ├─ RT1 → RT2 → RT3 → RT4 → Spotlighting   │
        │       │       │                                          │
        │       │       └─ PDFSanitizado (LIMPIO | ADVERSARIAL | ERROR)
        │       │                                                  │
        │       ├─ Actualizar DocumentoPDF.estado_sanitizacion     │
        │       │                                                  │
        │       ├─ AuditTrail (BR-U02-09)                         │
        │       │                                                  │
        │       └─ WebSocket PROGRESO_PIPELINE (BR-U02-16)        │
        │                                                          │
        │  FIN PARA CADA                                           │
        │                                                          │
        │  Portafolio(INGESTA → ANALISIS)                         │
        │  WebSocket INGESTA_COMPLETADA (BR-U02-17)               │
        └──────────────────────────────────────────────────────────┘
                  │
                  │ (señal para U-03)
                  ▼
        Portafolio.estado = ANALISIS → U-03 MotorExtraccion puede comenzar
```

---

## Módulo 5: Interfaz de Salida — Contrato con U-03

U-02 entrega a U-03 los siguientes objetos por cada cotización procesada:

| Objeto | Tipo | Descripción |
|---|---|---|
| `PDFSanitizado` | Estructura in-memory | PDF con estado `LIMPIO`, bytes del contenido, tokens de Spotlighting |
| `DocumentoPDF` | Entidad persistida en BD | Con `estado_sanitizacion = LIMPIO` y `ruta_almacenamiento` en S3 |
| `Cotizacion` | Entidad persistida en BD | Con todos los datos del proveedor y `estado_analisis = PENDIENTE` |
| `Portafolio` | Entidad persistida en BD | Con `estado = ANALISIS` |

**Solo los PDFs con `estado = LIMPIO` son entregados a U-03.** Los PDFs `ADVERSARIAL` y `ERROR` nunca llegan al `MotorExtraccion`.

---

## Resumen de Flujos por Historia de Usuario

| Historia | Flujo clave |
|---|---|
| **US-01** (Iniciar Análisis) | §1.1: Validaciones + creación Portafolio + 202 + BackgroundTask |
| **US-02** (Estado en Tiempo Real) | §1.2: WebSocket PROGRESO_PIPELINE + INGESTA_COMPLETADA |
| **US-03** (Sanitización Zero Trust) | §2.1: Pipeline RT1→RT2→RT3→RT4→Spotlighting; §2.2: Detección RT2; §2.3: Spotlighting |
| **US-19** (CISO: Log Sanitización) | §2.4: AuditTrail con metadata JSONB forense |
| **US-20/21** (Alertas Seguridad) | §2.4: WebSocket ALERTA_SEGURIDAD solo a ADMIN y CISO |
