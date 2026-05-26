# Business Logic Model — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-25  
> **Componentes**: C-08 AutenticadorOAuth, C-10 APIRest, C-13 RepositorioLocal, C-14 GestorWebSocket

---

## Descripción General

U-01 Fundación provee la infraestructura de negocio sobre la que se construyen todas las demás unidades:

1. **AutenticadorOAuth (C-08)**: Gestiona el ciclo de vida de la identidad y autorización del usuario
2. **APIRest (C-10)**: Define el contrato de comunicación cliente-servidor con versionado y manejo de errores
3. **RepositorioLocal (C-13)**: Gestiona la persistencia del dominio y la trazabilidad vía AuditTrail
4. **GestorWebSocket (C-14)**: Provee notificaciones en tiempo real del pipeline de análisis

---

## Módulo 1: AutenticadorOAuth (C-08)

### 1.1 Flujo de Autenticación OAuth2 con Google

```
Usuario → Clic "Iniciar sesión"
    │
    ▼
[C-08] Genera state + code_verifier (PKCE)
    │
    ▼
Redirect → Google OAuth2 Consent Screen
    │
    ▼ (usuario aprueba)
Google → Callback /api/v1/auth/callback?code=...&state=...
    │
    ▼
[C-08] Valida state (anti-CSRF)
    │
    ▼
[C-08] Intercambia code por access_token + id_token (Google)
    │
    ▼
[C-08] Decodifica id_token → extrae email, nombre, iss
    │
    ├─ iss != accounts.google.com → HTTP 401 (BR-01)
    ├─ email dominio != ALLOWED_DOMAIN → HTTP 403 (BR-02)
    │
    ▼
[C-08] Busca Usuario en BD por email
    │
    ├─ No existe + email != ADMIN_BOOTSTRAP_EMAIL → HTTP 403 + AuditTrail(ACCESO_DENEGADO) (BR-03)
    ├─ No existe + email == ADMIN_BOOTSTRAP_EMAIL → Crear Usuario(rol=ADMIN) + AuditTrail(LOGIN_EXITOSO)
    ├─ Existe + activo=TRUE → AuditTrail(LOGIN_EXITOSO) + actualizar ultimo_login
    └─ Existe + activo=FALSE → HTTP 403 + AuditTrail(ACCESO_DENEGADO)
    │
    ▼
[C-08] Genera JWT de sesión (exp: 3600s) + Refresh Token
    │
    ▼
Redirect → Frontend con tokens en cookie segura (HttpOnly, Secure, SameSite=Strict)
```

### 1.2 Flujo de Refresh de Sesión

```
Request con JWT expirado
    │
    ▼
[C-08] Detecta 401 Unauthorized en middleware
    │
    ▼
[C-08] Intenta renovar con Refresh Token
    │
    ├─ Refresh válido → Emite nuevo JWT (3600s) + nuevo Refresh → continúa request
    └─ Refresh expirado/inválido → HTTP 401 → Frontend redirige a /login
```

### 1.3 Flujo de Autorización por Rol

```
Request autenticado → Endpoint protegido
    │
    ▼
[C-08] Extrae rol del JWT
    │
    ▼
[C-08] Evalúa permiso requerido por el endpoint (BR-05 matriz de permisos)
    │
    ├─ Permiso concedido → Continúa ejecución
    └─ Permiso denegado → HTTP 403 + AuditTrail(ACCESO_DENEGADO)
```

---

## Módulo 2: APIRest (C-10)

### 2.1 Estructura de Endpoints Base

```
/api/v1/
├── auth/
│   ├── GET  /login          → Inicia flujo OAuth2 (público)
│   └── GET  /callback       → Recibe callback OAuth2 (público)
│
├── portafolios/
│   ├── GET  /               → Lista portafolios (paginado) [ANALISTA]
│   ├── POST /               → Crea portafolio [ANALISTA]
│   ├── GET  /{id}           → Detalle portafolio [ANALISTA]
│   └── POST /{id}/analisis  → Inicia análisis [ANALISTA]
│
├── usuarios/
│   ├── GET  /               → Lista usuarios [ADMIN]
│   ├── POST /               → Crea usuario [ADMIN]
│   └── PATCH /{id}/rol      → Cambia rol [ADMIN]
│
├── seguridad/
│   ├── GET  /audit-trail    → Consulta AuditTrail [ADMIN, CISO]
│   └── GET  /alertas        → Lista alertas de seguridad [ADMIN, CISO]
│
└── /health                  → Healthcheck (público)
```

### 2.2 Pipeline de Procesamiento de Request

```
HTTP Request
    │
    ▼
[Middleware] Rate Limiting (BR-16): 100 req/min por usuario → 429 si excede
    │
    ▼
[Middleware] Autenticación: Valida Bearer Token → 401 si inválido
    │
    ▼
[Middleware] Autorización: Valida rol para el endpoint → 403 si no autorizado
    │
    ▼
[Handler] Valida body con schema Pydantic (BR-24) → 422/400 con RFC 7807 si inválido
    │
    ▼
[Service] Ejecuta lógica de negocio
    │
    ▼
[Repository] Persiste en BD
    │
    ▼
HTTP Response (200/201/204)
```

### 2.3 Transformación de Errores a RFC 7807

Todos los tipos de excepción del sistema se mapean al formato RFC 7807 (BR-12):

| Excepción Python/FastAPI | HTTP Status | `type` RFC 7807 |
|--------------------------|-------------|-----------------|
| `AuthenticationError` | 401 | `/errors/authentication-required` |
| `AuthorizationError` | 403 | `/errors/access-denied` |
| `EntityNotFoundError` | 404 | `/errors/resource-not-found` |
| `ValidationError` (Pydantic) | 422 | `/errors/validation-failed` |
| `BusinessRuleError` | 400 | `/errors/business-rule-violation` |
| `RateLimitError` | 429 | `/errors/rate-limit-exceeded` |
| `InternalError` | 500 | `/errors/internal-server-error` |

---

## Módulo 3: RepositorioLocal (C-13)

### 3.1 Modelo de Repositorios

Cada entidad tiene su repositorio con operaciones CRUD básicas y consultas de negocio:

```
RepositorioPortafolio
├── crear(portafolio: Portafolio) → Portafolio
├── buscar_por_id(id: UUID) → Portafolio | None
├── buscar_por_codigo_proceso(codigo: str) → Portafolio | None
├── listar(page: int, page_size: int, analista_id: UUID?) → PaginatedResult[Portafolio]
├── actualizar_estado(id: UUID, estado: EstadoPortafolio) → Portafolio
└── eliminar(id: UUID) → bool  ← Solo acceso interno de Admin, no expuesto en API

RepositorioCotizacion
├── crear(cotizacion: Cotizacion) → Cotizacion
├── buscar_por_id(id: UUID) → Cotizacion | None
├── listar_por_portafolio(portafolio_id: UUID, page, page_size) → PaginatedResult[Cotizacion]
└── actualizar_estado_analisis(id: UUID, estado: EstadoAnalisis) → Cotizacion

RepositorioVariableExtraida
├── crear(variable: VariableExtraida) → VariableExtraida
├── listar_por_cotizacion(cotizacion_id: UUID) → List[VariableExtraida]
└── registrar_revision_hitl(id: UUID, valor_revisado: str, revisado_por: UUID) → VariableExtraida

RepositorioUsuario
├── crear(usuario: Usuario) → Usuario
├── buscar_por_email(email: str) → Usuario | None
├── buscar_por_id(id: UUID) → Usuario | None
├── listar(page, page_size) → PaginatedResult[Usuario]
├── actualizar_rol(id: UUID, rol: Rol) → Usuario
└── actualizar_ultimo_login(id: UUID) → Usuario

RepositorioAuditTrail
└── registrar_evento(evento: AuditTrailEvento) → None  ← ÚNICO método; no hay update/delete
```

### 3.2 Flujo de Registro en AuditTrail

```
Acción de negocio (exitosa o fallida)
    │
    ▼
[Service] Construye AuditTrailEvento:
    ├── usuario_id (del JWT, o None si es sistema)
    ├── evento (del catálogo en BR-10)
    ├── entidad_tipo + entidad_id (si aplica)
    ├── descripcion (detalle legible)
    ├── metadata (JSONB: estado anterior/nuevo, datos adicionales)
    └── ip_origen (del request context)
    │
    ▼
[RepositorioAuditTrail] INSERT en tabla audit_trail
    │
    └─ Si INSERT falla → log de sistema (NO lanzar excepción que interrumpa el flujo principal)
```

### 3.3 Flujo de Creación de Portafolio

```
POST /api/v1/portafolios { codigo_proceso, nombre_proceso }
    │
    ▼
[API] Valida schema Pydantic
    │
    ▼
[Service] Verifica que codigo_proceso no exista → 400 + RFC 7807 si ya existe
    │
    ▼
[Service] Crea objeto Portafolio:
    ├── id = generar_uuid_v7()
    ├── estado = PENDIENTE
    ├── analista_id = usuario_actual.id
    ├── total_cotizaciones = 0
    └── cotizaciones_procesadas = 0
    │
    ▼
[RepositorioPortafolio] INSERT en BD
    │
    ▼
[RepositorioAuditTrail] Registrar PORTAFOLIO_CREADO
    │
    ▼
HTTP 201 Created { portafolio }
```

---

## Módulo 4: GestorWebSocket (C-14)

### 4.1 Flujo de Conexión WebSocket

```
Cliente → GET /ws/{portafolio_id} (Upgrade: websocket)
    │
    ▼
[GestorWebSocket] Acepta conexión
    │
    ▼
[GestorWebSocket] Espera mensaje de autenticación (máx 5 segundos)
    │
    ├─ Timeout → Cierra conexión con código 4001
    ├─ Token inválido → Cierra conexión con código 4001
    └─ Token válido →
        │
        ▼
        [GestorWebSocket] Valida que usuario tenga acceso al portafolio_id:
        ├─ ANALISTA: solo su propio portafolio (analista_id == usuario.id)
        ├─ ADMIN/CISO: acceso a alertas de seguridad de cualquier portafolio
        │
        ├─ Sin acceso → Cierra con código 4003
        └─ Con acceso → Registra conexión en mapa portafolio→[conexiones]
```

### 4.2 Flujo de Publicación de Eventos

```
Pipeline IA / Servicio de negocio → Evento ocurrido
    │
    ▼
[GestorWebSocket] publi_evento(portafolio_id, tipo_evento, payload)
    │
    ▼
[GestorWebSocket] Construye mensaje:
    {
      "tipo": "NOMBRE_EVENTO",
      "payload": { ... },
      "timestamp": "UTC ISO8601"
    }
    │
    ▼
[GestorWebSocket] Busca conexiones activas para portafolio_id
    │
    ├─ Filtra por destinatarios según BR-18 (ej: ALERTA_SEGURIDAD → solo ADMIN y CISO)
    │
    ▼
[GestorWebSocket] Envía mensaje a cada conexión activa
    │
    └─ Si envío falla → Remueve conexión del mapa (cliente desconectado)
```

### 4.3 Flujo de Evento ALERTA_SEGURIDAD

```
[SanitizadorPDF] detecta elemento adversarial en PDF
    │
    ▼
[SanitizadorPDF] actualiza DocumentoPDF.estado_sanitizacion = ADVERSARIAL
    │
    ▼
[AuditTrail] registra PDF_ADVERSARIAL_DETECTADO
    │
    ▼
[GestorWebSocket] publica ALERTA_SEGURIDAD:
    {
      "tipo": "ALERTA_SEGURIDAD",
      "payload": {
        "portafolio_id": "...",
        "cotizacion_id": "...",
        "pdf_nombre": "...",
        "tipo_amenaza": "descripcion del elemento adversarial"
      },
      "timestamp": "..."
    }
    │
    ▼
[GestorWebSocket] Envía SOLO a usuarios con rol ADMIN y CISO conectados
```

### 4.4 Flujo de Evento ANALISIS_COMPLETADO

```
[Pipeline IA] finaliza análisis del portafolio (todas las cotizaciones procesadas)
    │
    ▼
[ServicioPortafolio] actualiza estado → COMPLETADO o REVISION_HITL
    │
    ▼
[AuditTrail] registra PORTAFOLIO_ESTADO_CAMBIADO
    │
    ▼
[GestorWebSocket] publica ANALISIS_COMPLETADO:
    {
      "tipo": "ANALISIS_COMPLETADO",
      "payload": {
        "portafolio_id": "...",
        "total_cotizaciones": 18,
        "cotizaciones_completadas": 18,
        "cotizaciones_con_error": 0,
        "requiere_revision_hitl": true,
        "estado_final": "REVISION_HITL"
      },
      "timestamp": "..."
    }
    │
    ▼
[GestorWebSocket] Envía al ANALISTA dueño del portafolio
```

### 4.5 Gestión de Reconexión (Lado Cliente — Angular)

```
Desconexión WebSocket detectada por cliente
    │
    ▼
Angular WebSocket Service:
    intento = 1
    │
    ▼
    loop:
        ├─ esperar min(2^(intento-1), 60) segundos
        ├─ intentar reconexión
        │
        ├─ Éxito → Enviar token de autenticación → Restablecer suscripción
        │
        ├─ Fallo + intento < 10 → intento++ → continuar loop
        └─ Fallo + intento >= 10 → mostrar banner "Conexión perdida. Recarga la página."
```

---

## Resumen de Interacciones entre Módulos

```
                    ┌─────────────┐
                    │   Frontend  │
                    │   Angular   │
                    └──────┬──────┘
                           │ HTTP /api/v1/*
                           │ WS /ws/{portafolio_id}
                           ▼
         ┌─────────────────────────────────────┐
         │            C-10 APIRest             │
         │  (rate limit, auth, routing, RFC7807)│
         └──────────────────┬──────────────────┘
                            │
              ┌─────────────┼─────────────────┐
              ▼             ▼                 ▼
         ┌─────────┐  ┌──────────┐  ┌────────────────┐
         │  C-08   │  │  C-13    │  │    C-14         │
         │  Auth   │  │  Repo    │  │  WebSocket      │
         │  OAuth  │  │  Local   │  │  GestorWS       │
         └─────────┘  └──────────┘  └────────────────┘
              │             │                 ▲
              │         PostgreSQL            │
              │         AuditTrail            │ publi_evento()
              │                               │
              └─────────── Servicios de negocio (U-02, U-03, U-04...) ──┘
```

---
