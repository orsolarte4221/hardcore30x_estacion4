# Tech Stack Decisions — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — NFR Requirements  
> **Fecha**: 2026-05-26  

---

## Decisiones de Tech Stack

### TSD-01: Runtime y Lenguaje

| Atributo | Decisión |
|----------|----------|
| **Lenguaje** | Python 3.12 |
| **Gestor de dependencias** | `pip` + `requirements.txt` |
| **Entorno virtual** | `venv` (estándar de Python) |
| **Archivos de dependencias** | `requirements.txt` (producción) + `requirements-dev.txt` (dev/test) |

**Rationale:** `pip` + `requirements.txt` es el estándar más ampliamente conocido, sin overhead de herramientas adicionales. Para el equipo piloto, reduce la curva de aprendizaje frente a `poetry` o `uv`. El lock de versiones se garantiza con versiones exactas (`==`) en `requirements.txt`.

**Estructura de dependencias:**
```
backend/
├── requirements.txt          # Dependencias de producción (versiones exactas)
└── requirements-dev.txt      # Dev/test (incluye requirements.txt con -r)
```

---

### TSD-02: Framework Web

| Atributo | Decisión |
|----------|----------|
| **Framework** | FastAPI 0.115.x |
| **Servidor ASGI** | Uvicorn con múltiples workers (`--workers 4`) |
| **Validación de datos** | Pydantic v2 (incluido con FastAPI) |
| **Documentación API** | OpenAPI 3.1 (generado automáticamente por FastAPI) |

**Rationale:** FastAPI con Uvicorn multi-worker aprovecha todos los cores de la instancia cloud (2 vCPU), permitiendo manejar 5–15 usuarios concurrentes con holgura. Pydantic v2 provee validación de schemas en la capa API (BR-24).

**Configuración de Uvicorn:**
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4 --loop uvloop
```

---

### TSD-03: Base de Datos y ORM

| Atributo | Decisión |
|----------|----------|
| **Base de datos** | PostgreSQL 16 (Managed service en cloud) |
| **ORM** | SQLAlchemy 2.x con modo **async** (`asyncpg` driver) |
| **Migraciones** | Alembic |
| **Pool de conexiones** | SQLAlchemy async connection pool (`min_size=5`, `max_size=20`) |
| **UUID v7** | Generado en Python con librería `uuid-utils` o `uuid7` |

**Rationale:** SQLAlchemy 2.x async permite que las queries a BD no bloqueen el event loop de Uvicorn, maximizando la concurrencia de requests. Alembic provee migraciones versionadas y reproducibles. El managed PostgreSQL del cloud garantiza el 99% de uptime con backups automáticos diarios.

**Estructura de archivos BD:**
```
backend/app/core/database/
├── engine.py          # AsyncEngine, async session factory
├── base.py            # DeclarativeBase de SQLAlchemy
└── session.py         # Dependency injection: get_db_session()

backend/alembic/
├── env.py             # Configuración Alembic (apunta a models de app/core/models/)
├── versions/          # Archivos de migración versionados
└── alembic.ini        # Configuración general
```

**Trigger de integridad AuditTrail (implementación recomendada):**
```sql
-- Trigger PostgreSQL para proteger audit_trail de UPDATE/DELETE
CREATE OR REPLACE FUNCTION prevent_audit_trail_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'audit_trail is append-only: UPDATE and DELETE are not allowed';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_trail_immutable
BEFORE UPDATE OR DELETE ON audit_trail
FOR EACH ROW EXECUTE FUNCTION prevent_audit_trail_modification();
```

---

### TSD-04: Autenticación OAuth2

| Atributo | Decisión |
|----------|----------|
| **Proveedor** | Google Workspace (exclusivo en MVP) |
| **Librería OAuth2** | `authlib` (manejo del flujo PKCE + OIDC) |
| **JWT de sesión** | `python-jose` (generación y validación de JWT HS256/RS256) |
| **Duración JWT** | 3600 segundos (1 hora) |
| **Almacenamiento token cliente** | Cookie HttpOnly + Secure + SameSite=Strict |
| **PKCE** | Obligatorio (code_verifier + code_challenge) |

**Rationale:** `authlib` es la librería más completa y mantenida para OAuth2/OIDC en Python. `python-jose` provee validación robusta de JWT. Las cookies HttpOnly previenen acceso desde JavaScript (XSS protection).

---

### TSD-05: WebSocket

| Atributo | Decisión |
|----------|----------|
| **Soporte WebSocket** | FastAPI WebSocket nativo (basado en Starlette) |
| **Gestión de conexiones** | Dict en memoria: `Dict[UUID, List[WebSocket]]` |
| **Autenticación WS** | Token enviado en primer mensaje tras handshake (< 5s timeout) |
| **Serialización** | `json` estándar de Python (payloads simples) |

**Rationale:** FastAPI provee soporte WebSocket out-of-the-box sin dependencias adicionales. Para el volumen del piloto (5–15 conexiones simultáneas), un Dict en memoria es suficiente y evita la complejidad de Redis pub/sub.

**Limitación conocida:** Con múltiples workers Uvicorn, las conexiones WebSocket son locales al proceso. Para el piloto (usuario conectado a un solo portafolio a la vez), esto es aceptable. En producción con múltiples instancias, se requerirá Redis pub/sub o sticky sessions.

---

### TSD-06: Configuración y Secrets

| Atributo | Decisión |
|----------|----------|
| **Gestión de config** | `pydantic-settings` (`BaseSettings`) |
| **Archivo de configuración** | `.env` (excluido de git con `.gitignore`) |
| **Validación al inicio** | Al importar `Settings`, Pydantic valida todas las variables requeridas |
| **Ejemplo de configuración** | `.env.example` en el repositorio (sin valores reales) |

**Variables de entorno requeridas para U-01:**
```env
# Base de datos
DATABASE_URL=postgresql+asyncpg://user:password@host:5432/assistent_buy

# Google OAuth2
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://domain/api/v1/auth/callback
ALLOWED_DOMAIN=universidad.edu.co

# JWT
JWT_SECRET_KEY=...  # mínimo 32 caracteres aleatorios
JWT_ALGORITHM=HS256

# Bootstrap Admin
ADMIN_BOOTSTRAP_EMAIL=admin@universidad.edu.co

# App
APP_ENV=production  # development | production
DEBUG=false
```

---

### TSD-07: Testing

| Atributo | Decisión |
|----------|----------|
| **Framework de tests** | `pytest` + `pytest-asyncio` |
| **Coverage** | `pytest-cov` (objetivo: ≥ 80% global U-01) |
| **Base de datos en tests** | `pytest-postgresql` (instancia real PostgreSQL en tests, no mock) |
| **Mock de APIs externas** | `respx` (mock de `httpx` para Google OAuth2) |
| **HTTP test client** | `httpx` con `AsyncClient` (compatible con FastAPI async) |
| **Fixtures** | `conftest.py` con fixtures de BD, usuario autenticado por rol, tokens |

**Estructura de tests:**
```
backend/tests/
├── conftest.py                 # Fixtures globales (DB, client, tokens por rol)
├── unit/
│   ├── test_auth/              # Tests de AutenticadorOAuth (C-08)
│   │   ├── test_oauth_flow.py
│   │   ├── test_jwt.py
│   │   └── test_authorization.py
│   ├── test_repositories/      # Tests de RepositorioLocal (C-13)
│   │   ├── test_portafolio_repo.py
│   │   ├── test_usuario_repo.py
│   │   └── test_audit_trail_repo.py
│   └── test_websocket/         # Tests de GestorWebSocket (C-14)
│       └── test_ws_manager.py
└── integration/
    ├── test_api_auth.py        # Tests e2e de endpoints /auth/*
    ├── test_api_portafolios.py # Tests e2e de endpoints /portafolios/*
    └── test_api_usuarios.py    # Tests e2e de endpoints /usuarios/*
```

---

### TSD-08: Contenedorización

| Atributo | Decisión |
|----------|----------|
| **Contenedor** | Docker con imagen `python:3.12-slim` |
| **Local dev** | Docker Compose (backend + PostgreSQL + pgAdmin) |
| **Producción** | Docker en instancia cloud (sin orquestación Kubernetes en piloto) |
| **Variables de entorno** | Inyectadas por el proveedor cloud (no en imagen) |

**`docker-compose.yml` para desarrollo:**
```yaml
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [db]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: assistent_buy
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports: ["5432:5432"]

  pgadmin:
    image: dpage/pgadmin4
    ports: ["5050:80"]
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@local.com
      PGADMIN_DEFAULT_PASSWORD: dev

volumes:
  pgdata:
```

---

## Resumen Ejecutivo de Decisiones

| Categoría | Tecnología Seleccionada | Versión |
|-----------|------------------------|---------|
| Lenguaje | Python | 3.12 |
| Gestor deps | pip + requirements.txt | — |
| Framework web | FastAPI | 0.115.x |
| Servidor ASGI | Uvicorn (4 workers) | 0.32.x |
| Validación | Pydantic v2 | 2.x |
| Base de datos | PostgreSQL Managed | 16 |
| ORM | SQLAlchemy async | 2.x |
| Driver BD | asyncpg | 0.29.x |
| Migraciones | Alembic | 1.13.x |
| OAuth2 | authlib | 1.3.x |
| JWT | python-jose | 3.3.x |
| Config | pydantic-settings | 2.x |
| WebSocket | FastAPI nativo | — |
| Testing | pytest + pytest-asyncio | 8.x |
| Contenedor | Docker python:3.12-slim | — |
| Dev orquestación | Docker Compose | 2.x |

---

## Dependencias Prohibidas

Las siguientes librerías **no** se usarán en U-01:

| Librería | Razón |
|----------|-------|
| `flask`, `django` | FastAPI es el framework definido |
| `celery` | No se requiere queue en U-01; pipeline IA es async en-proceso |
| `redis` | No se requiere para piloto; escalado horizontal post-piloto |
| `tortoise-orm` | SQLAlchemy async es el ORM seleccionado |
| `alembic` alternativo | Alembic es la única herramienta de migraciones permitida |
