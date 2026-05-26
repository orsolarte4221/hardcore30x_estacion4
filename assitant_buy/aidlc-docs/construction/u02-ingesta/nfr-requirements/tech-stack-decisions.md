# Tech Stack Decisions — U-02 Ingesta y Sanitización

> **Unidad**: U-02 — Ingesta y Sanitización  
> **Fase**: CONSTRUCTION — NFR Requirements  
> **Fecha**: 2026-05-26  
> **Componentes**: C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-12 RepositorioSAB, C-16 GatewayDocumentAI

---

## Herencia del Stack de U-01

U-02 **hereda íntegramente** el tech stack de U-01 (Python 3.12, FastAPI 0.115.x, SQLAlchemy 2.x async, PostgreSQL 16, Alembic, pydantic-settings, Docker). Este documento documenta **únicamente las decisiones adicionales** requeridas por las responsabilidades específicas de U-02: integración SAB (HTTP async + Keycloak), procesamiento de PDFs, y almacenamiento en S3.

---

## TSD-U02-01: Cliente HTTP Async (Integración SAB y Keycloak)

| Atributo | Decisión |
|---|---|
| **Librería** | `httpx` 0.27.x con `AsyncClient` |
| **Modo** | Async (`async with httpx.AsyncClient()`) para no bloquear el event loop |
| **Timeouts** | Configurados via `httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0)` |
| **Retry** | Manual con backoff (no se usa librería de retry externa en piloto) |
| **Certificados SSL** | Verificación SSL habilitada por defecto (`verify=True`) — no se deshabilita en producción |

**Usos en U-02:**
1. `GestorTokenSAB`: `POST {SAB_KEYCLOAK_URL}/openid-connect/token` (obtener/renovar token)
2. `IngestaConector.recuperar_pdf()`: `GET {SAB_BASE_URL}/api/v1/cotizacion/documentos/descargar/{id}`

**Rationale:** `httpx` es el cliente HTTP async estándar en el ecosistema FastAPI/async Python. Es la misma librería usada en los tests de U-01 (`respx` para mocking de `httpx`). Evita introducir una segunda librería HTTP (`aiohttp`) al stack.

**Configuración de sesión reutilizable:**
```python
# El GestorTokenSAB y el IngestaConector comparten un AsyncClient reutilizable
# (no se crea un nuevo cliente por cada descarga)
async_client = httpx.AsyncClient(
    timeout=httpx.Timeout(connect=5.0, read=SAB_DOWNLOAD_TIMEOUT_S),
    verify=True,
    follow_redirects=False,
)
```

---

## TSD-U02-02: Parsing y Análisis de PDFs

| Atributo | Decisión |
|---|---|
| **Librería principal** | `pypdf` 4.x (antes `PyPDF2`, ahora mantenido activamente) |
| **Uso** | RT1 (metadatos maliciosos), RT2 (extracción de texto para detección de inyección), RT3 (análisis de rendering básico), RT4 (validación de estructura) |
| **Parsing de bajo nivel** | `pypdf` con manejo de PDFs malformados (excepciones controladas) |
| **NO se ejecuta JavaScript** | `pypdf` parsea estructura, no ejecuta contenido activo |

**Rationale:** `pypdf` es la librería Python más usada para parsing de PDFs sin dependencias nativas. Es puro Python, se instala fácilmente en contenedores, y provee acceso a metadatos, estructura interna y texto extraído. No depende de instalaciones de sistema (como `pdfium` o `poppler`).

**Limitaciones conocidas:**
- RT3 (ofuscación por texto invisible) tiene cobertura parcial con `pypdf`; la detección completa de texto renderizado requeriría `pdfplumber` o análisis de colores. Para el piloto, se detecta texto con tamaño de fuente 0 o 1pt y texto en capas marcadas como ocultas.
- La detección de homoglíficos (RT3) se realiza sobre el texto extraído con normalización Unicode, no sobre el rendering.

**Detección adicional RT2 — Análisis de texto:**
```python
# Normalización Unicode para detección de homoglíficos y patrones multi-idioma
import unicodedata

def normalizar_texto(texto: str) -> str:
    return unicodedata.normalize('NFKC', texto)
```

---

## TSD-U02-03: Almacenamiento de PDFs en S3

| Atributo | Decisión |
|---|---|
| **Librería AWS** | `aiobotocore` 2.x (wrapper async de `botocore`) |
| **Alternativa GCP/Azure** | Si el cloud es GCP: `google-cloud-storage` con `asyncio`. Si Azure: `azure-storage-blob` async |
| **Modo de upload** | `upload_fileobj` con streaming para archivos > 10 MB |
| **Server-Side Encryption** | `SSE-S3` (AES-256 gestionado por AWS, sin costo adicional) |
| **Bucket** | 1 bucket dedicado para PDFs de Assistant Buy (no compartido con otros sistemas) |
| **Política de acceso** | Sin acceso público; acceso solo desde el backend via IAM Role/Service Account |

**Key de almacenamiento en S3:**
```
portafolios/{portafolio_id}/pdfs/{documento_pdf_id}.pdf
```

**Rationale:** `aiobotocore` permite uploads a S3 sin bloquear el event loop. El streaming para archivos grandes evita que la BackgroundTask consuma memoria excesiva. La política de no-acceso-público asegura que los PDFs solo son accesibles por el backend autorizado.

---

## TSD-U02-04: Conexión Read-Only a la BD del SAB

| Atributo | Decisión |
|---|---|
| **Librería** | `SQLAlchemy 2.x async` (misma que en U-01, con engine adicional para SAB) |
| **Driver** | `asyncpg` (el SAB usa PostgreSQL) |
| **Engine** | `sab_engine` — engine separado del `local_engine` de la BD propia del sistema |
| **Pool de conexiones** | `min_size=1`, `max_size=5` (bajo porque el acceso es solo al inicio de cada pipeline) |
| **URL** | `SAB_DB_URL` = `postgresql+asyncpg://readonly_user:pass@sab-host:5432/sab_db` |

**Rationale:** Reutilizar SQLAlchemy evita introducir una segunda librería de BD. El pool pequeño es suficiente dado que las queries al SAB se hacen solo al inicio del pipeline (listar cotizaciones y documentos), no durante todo el proceso.

**Configuración de motores separados:**
```python
# engine.py — dos engines, completamente separados
local_engine = create_async_engine(settings.DATABASE_URL, ...)     # BD propia del sistema
sab_engine   = create_async_engine(settings.SAB_DB_URL,
                                   pool_size=1, max_overflow=4,
                                   execution_options={"readonly": True})
```

---

## TSD-U02-05: Variables de Entorno Nuevas (U-02)

U-02 añade las siguientes variables de entorno al `.env` (además de las definidas en U-01):

```env
# ─── Integración SAB — BD (read-only) ───────────────────────────
SAB_DB_URL=postgresql+asyncpg://readonly:password@sab-host:5432/sab_db

# ─── Integración SAB — API REST + Keycloak ───────────────────────
SAB_BASE_URL=https://backend.unicauca.edu.co/sab-bienes
SAB_KEYCLOAK_URL=https://proyunicauca.docxflow.com:8443/realms/unicauca-funcionarios-qa
SAB_CLIENT_ID=un-client-id
SAB_CLIENT_SECRET=6Sc...  # SECRETO — nunca en código
SAB_SERVICE_USERNAME=assistant_buy_service
SAB_SERVICE_PASSWORD=...  # SECRETO — nunca en código

# ─── Límites de pipeline ─────────────────────────────────────────
SAB_DOWNLOAD_TIMEOUT_S=30
PDF_MAX_SIZE_MB=50

# ─── S3 / Object Storage ─────────────────────────────────────────
S3_BUCKET_NAME=assistent-buy-pdfs
S3_REGION=us-east-1
# Si se usa IAM Role (recomendado en producción), no se necesitan AWS_ACCESS_KEY_ID
# Para desarrollo local:
AWS_ACCESS_KEY_ID=...      # SECRETO
AWS_SECRET_ACCESS_KEY=...  # SECRETO
```

**Validación al startup** (via `pydantic-settings`): Si cualquiera de las variables marcadas como requeridas falta, la aplicación falla al iniciar con mensaje claro (fail-fast).

---

## TSD-U02-06: Testing Específico de U-02

| Atributo | Decisión |
|---|---|
| **Mock de llamadas HTTP al SAB** | `respx` (mismo que U-01 para Google OAuth) — mock de `httpx.AsyncClient` |
| **Mock de Keycloak** | `respx` interceptando `POST {SAB_KEYCLOAK_URL}/...` |
| **Mock de S3** | `moto` 5.x (mock de AWS S3 para tests unitarios e integración) |
| **Fixtures de PDFs** | PDFs de prueba en `tests/fixtures/pdfs/` (limpios, adversariales, corruptos) |
| **Tests de seguridad PBT** | `hypothesis` (Property-Based Testing para RT1-RT4 y Spotlighting) |

**Estructura de tests de U-02:**
```
backend/tests/
├── unit/
│   ├── test_sanitizador/
│   │   ├── test_rt1_metadatos.py       # PDFs con JavaScript, auto-actions, URIs
│   │   ├── test_rt2_inyeccion.py       # PDFs con prompts en múltiples idiomas
│   │   ├── test_rt3_ofuscacion.py      # PDFs con texto invisible, homoglíficos
│   │   ├── test_rt4_estructura.py      # PDFs corruptos, oversized, inválidos
│   │   └── test_spotlighting.py        # Delimitadores dinámicos, unicidad de tokens
│   ├── test_gestor_token_sab/
│   │   ├── test_obtener_token.py       # Login exitoso, 401, timeout, refresh
│   │   └── test_no_leak_credenciales.py # PBT: credentials never in logs/responses
│   └── test_ingesta_conector/
│       ├── test_recuperar_pdf.py       # Descarga OK, 404, timeout, oversized
│       └── test_pipeline_completo.py   # End-to-end con mocks SAB + S3
├── test_gateway_document_ai/
│   ├── test_ocr_pdf_escaneado.py       # OCR exitoso, fallo, companion S3
│   └── test_ocr_no_duplicate.py        # U-03 reutiliza texto_ocr_s3_key sin re-llamar
├── integration/
│   └── test_api_analisis.py            # POST /analisis → 202 + WebSocket events
└── fixtures/
    └── pdfs/
        ├── clean_simple.pdf
        ├── clean_scanned.pdf              # PDF escaneado (imagen) sin texto nativo
        ├── adversarial_injection_es.pdf
        ├── adversarial_injection_en.pdf
        ├── adversarial_injection_scanned.pdf  # Inyección embebida en imagen escaneada
        ├── adversarial_invisible_text.pdf
        ├── adversarial_javascript.pdf
        └── corrupted.pdf
```

---

## TSD-U02-07: Dependencias Nuevas (U-02)

Las siguientes dependencias se agregan a `requirements.txt` para U-02:

| Paquete | Versión | Uso |
|---|---|---|
| `httpx` | `0.27.x` | Cliente HTTP async para SAB API y Keycloak |
| `pypdf` | `4.x` | Parsing y análisis de estructura de PDFs (RT1-RT4) |
| `aiobotocore` | `2.x` | Upload async a S3 (PDFs + companion `.txt` del OCR) **y** OCR via Amazon Textract |
| `hypothesis` | `6.x` | Property-Based Testing para pipeline Zero Trust |
| `moto` | `5.x` | Mock de S3 y Textract en tests (solo en `requirements-dev.txt`) |
| `pytest-httpx` | `0.30.x` | Mock de llamadas httpx en tests (Keycloak + SAB API) |

**Nota sobre Amazon Textract**: Se usa la operación `detect_document_text` (síncrona, para PDFs ≤ 5 páginas) o `start_document_text_detection` + polling (asíncrona, para PDFs de más páginas). `aiobotocore` ya cubre el cliente de Textract — **no se requiere ninguna librería adicional**.

```env
# Textract usa el mismo AWS_REGION y credenciales que S3
# No se necesitan variables adicionales para Textract en producción
# (usa el IAM Role del servidor que ya cubre S3 y Textract)
# Para desarrollo local:
AWS_ACCESS_KEY_ID=...          # SECRETO
AWS_SECRET_ACCESS_KEY=...      # SECRETO  
AWS_REGION=us-east-1           # Misma región que el bucket S3
```

**Ventaja clave**: Dado que el proyecto usa AWS, Textract comparte la misma capa de autenticación IAM que S3. No se necesita gestionar credenciales adicionales de un proveedor diferente (como GCP).

**Dependencias de U-01 reutilizadas sin cambios:**
- `sqlalchemy[asyncio]` — para conexión a BD del SAB (engine adicional)
- `asyncpg` — driver PostgreSQL async
- `pydantic-settings` — para las nuevas variables de entorno

---

## Resumen Ejecutivo de Decisiones Nuevas (U-02)

| Categoría | Tecnología | Versión |
|---|---|---|
| Cliente HTTP async | httpx | 0.27.x |
| Autenticación Keycloak | httpx + OAuth2 ROPC (implementación propia) | — |
| Parsing PDF | pypdf | 4.x |
| Detección homoglíficos | unicodedata (stdlib Python) | — |
| OCR para PDFs escaneados | **Amazon Textract** (via aiobotocore) | — |
| Almacenamiento OCR | S3 companion file `.txt` (no en BD) | — |
| Storage S3 | aiobotocore | 2.x |
| BD SAB (read-only) | SQLAlchemy async + asyncpg (reutilizado) | — |
| PBT Security Tests | hypothesis | 6.x |
| Mock S3/Textract en tests | moto | 5.x |
| Mock httpx en tests | pytest-httpx | 0.30.x |

---

## Dependencias Prohibidas (U-02)

| Librería | Razón |
|---|---|
| `requests` | Bloqueante; usar `httpx` async en su lugar |
| `PyPDF2` | Deprecado; usar `pypdf` (sucesor oficial) |
| `pdfminer.six` | Innecesario para U-02; `pypdf` cubre los casos requeridos |
| `subprocess` + `pdftotext` | Sin dependencias de sistema en el contenedor; solo libs Python puras |
| `boto3` (sync) | Bloqueante; usar `aiobotocore` async en su lugar |
| `pytesseract` / `tesseract` | Requiere binario de sistema; usar Amazon Textract en su lugar |
| `google-cloud-documentai` | El proyecto usa AWS; usar Amazon Textract (misma capa IAM que S3) |
