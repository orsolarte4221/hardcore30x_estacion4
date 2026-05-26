# Business Rules — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Functional Design  
> **Fecha**: 2026-05-25  

---

## Convenciones

- **BR-XX**: Identificador único de regla de negocio
- **Severidad**: `CRITICA` (fallo bloquea el flujo) | `ALTA` (impacta la integridad) | `MEDIA` (afecta UX) | `BAJA` (validación de datos)
- **Componente**: C-08 (Auth), C-10 (API REST), C-13 (Repositorio), C-14 (WebSocket)

---

## Sección 1 — Autenticación y Autorización (C-08)

### BR-01: Solo Google Workspace como proveedor OAuth2
- **Severidad**: CRITICA
- **Regla**: El sistema solo acepta tokens de identidad emitidos por Google Workspace. Cualquier token de otro proveedor es rechazado con HTTP 401.
- **Validación**: El `iss` (issuer) del JWT debe ser `https://accounts.google.com` o `accounts.google.com`. Cualquier otro valor resulta en rechazo.

### BR-02: Email del dominio institucional obligatorio
- **Severidad**: CRITICA
- **Regla**: Solo se permiten emails del dominio institucional configurado (vía variable de entorno `ALLOWED_DOMAIN`). Emails de dominios externos son rechazados.
- **Excepción**: Durante el piloto, si `ALLOWED_DOMAIN` no está configurado, se permite cualquier email de Google (`@gmail.com` incluido). En producción debe ser obligatorio.

### BR-03: El rol debe existir en tabla local
- **Severidad**: CRITICA
- **Regla**: Tras autenticar con Google, el sistema busca al usuario en la tabla `Usuario` por `email`. Si el usuario no existe en la tabla local (no ha sido registrado por un Admin), el acceso es denegado con HTTP 403 y se registra `ACCESO_DENEGADO` en `AuditTrail`.
- **Excepción**: El email bootstrap `ADMIN` definido en `.env` se auto-registra con rol `ADMIN` en el primer login.

### BR-04: Duración de sesión 1 hora con refresh automático
- **Severidad**: ALTA
- **Regla**: El JWT de sesión expira en 3600 segundos (1 hora). El refresh token puede renovar la sesión silenciosamente si el usuario está activo. Al expirar el refresh token, el usuario debe re-autenticarse con Google.
- **Acción de expiración**: La sesión expirada redirige al flujo OAuth2 sin mensaje de error al usuario (refresh silencioso).

### BR-05: Matriz de permisos por rol (inmutable en MVP)
- **Severidad**: CRITICA
- **Regla**: Los permisos están predefinidos por rol y no son configurables en MVP:

| Acción | ANALISTA | ADMIN | CISO | COMITE |
|--------|----------|-------|------|--------|
| Iniciar análisis | ✅ | ❌ | ❌ | ❌ |
| Ver resultados de análisis | ✅ | ❌ | ❌ | ❌ |
| Gestionar aclaraciones | ✅ | ❌ | ❌ | ❌ |
| Generar reportes | ✅ | ❌ | ❌ | ❌ |
| Ver reporte comparativo | ✅ | ❌ | ❌ | ✅ |
| Ver log de sanitización | ❌ | ❌ | ✅ | ❌ |
| Ver alertas de seguridad | ❌ | ✅ | ✅ | ❌ |
| Ver logs del sistema | ❌ | ✅ | ❌ | ❌ |
| Configurar sistema | ❌ | ✅ | ❌ | ❌ |
| Gestionar usuarios/roles | ❌ | ✅ | ❌ | ❌ |

- Cualquier acción no autorizada devuelve HTTP 403 y registra `ACCESO_DENEGADO` en `AuditTrail`.

### BR-06: Un usuario tiene exactamente un rol
- **Severidad**: ALTA
- **Regla**: No existen roles combinados. Un usuario es `ANALISTA` OR `ADMIN` OR `CISO` OR `COMITE`. Un `ADMIN` no puede también actuar como `ANALISTA`.
- **Cambio de rol**: Solo un `ADMIN` puede modificar el rol de otro usuario. El cambio se registra en `AuditTrail` con evento `ROL_ASIGNADO`.

---

## Sección 2 — Esquema de Base de Datos (C-13)

### BR-07: UUID v7 obligatorio para todos los IDs primarios
- **Severidad**: ALTA
- **Regla**: Todos los `id` de entidades son UUID v7 (RFC 9562). No se usan BIGSERIAL, UUID v4 ni otros esquemas de ID. Los UUID v7 se generan en la capa de aplicación (Python), no en la BD.

### BR-08: Borrado físico (sin soft delete en MVP)
- **Severidad**: ALTA
- **Regla**: Las entidades `Portafolio`, `Cotizacion` y `Aclaracion` pueden ser eliminadas físicamente solo por un `ADMIN` mediante operación directa de BD (no expuesta en UI). No hay funcionalidad de borrado en la interfaz de usuario v1.
- **Trazabilidad**: El `AuditTrail` es append-only y provee el historial completo de eventos, independientemente del borrado físico de registros.

### BR-09: `AuditTrail` es inmutable (append-only)
- **Severidad**: CRITICA
- **Regla**: La tabla `AuditTrail` solo acepta operaciones `INSERT`. Las operaciones `UPDATE` y `DELETE` están prohibidas:
  - A nivel de aplicación: el repositorio solo expone un método `registrar_evento()`, sin métodos de actualización o borrado.
  - A nivel de BD (recomendado): trigger PostgreSQL que rechaza `UPDATE` y `DELETE` sobre la tabla con `RAISE EXCEPTION`.
- **`created_at` inmutable**: El campo `created_at` se establece con `DEFAULT NOW()` en BD y nunca es sobreescrito por la aplicación.

### BR-10: Catálogo completo de eventos de AuditTrail
- **Severidad**: ALTA
- **Regla**: Solo eventos del catálogo definido en `domain-entities.md` se registran en `AuditTrail`. Cualquier evento fuera del catálogo debe ser adicionado formalmente al catálogo antes de ser registrado.
- **Eventos de sistema**: `usuario_id = NULL` para eventos generados por procesos automáticos (pipeline IA, polling IMAP).

### BR-11: Integridad referencial estricta
- **Severidad**: CRITICA
- **Regla**: Todas las foreign keys deben tener restricciones `NOT NULL` donde aplique. Los cascade delete están definidos así:
  - `Portafolio` → `Cotizacion`: CASCADE DELETE (si el portafolio es eliminado, sus cotizaciones también)
  - `Cotizacion` → `DocumentoPDF`: CASCADE DELETE
  - `Cotizacion` → `VariableExtraida`: CASCADE DELETE
  - `Cotizacion` → `Discrepancia`: CASCADE DELETE
  - `AuditTrail` no tiene cascade (registros históricos son independientes)

---

## Sección 3 — API REST Base (C-10)

### BR-12: Formato de error RFC 7807 obligatorio
- **Severidad**: ALTA
- **Regla**: Todas las respuestas de error de la API deben seguir el estándar RFC 7807 Problem Details:
  ```json
  {
    "type": "https://assistent-buy.api/errors/{error-code}",
    "title": "Título legible del error",
    "status": 400,
    "detail": "Descripción detallada del error para el cliente",
    "instance": "/api/v1/portafolios/abc-123"
  }
  ```
- Los errores de validación de FastAPI/Pydantic también deben ser transformados a este formato mediante un handler global de excepciones.

### BR-13: Versionado de API en URL path
- **Severidad**: ALTA
- **Regla**: Todos los endpoints de la API deben incluir el prefijo `/api/v1/`. Ningún endpoint existe sin el prefijo de versión. Ejemplo: `GET /api/v1/portafolios`, `POST /api/v1/portafolios/{id}/analisis`.

### BR-14: Paginación offset/limit en todas las listas
- **Severidad**: MEDIA
- **Regla**: Todos los endpoints que devuelven listas deben aceptar parámetros `?page=1&page_size=20`. Valores por defecto: `page=1`, `page_size=20`. Máximo permitido: `page_size=100`.
- **Formato de respuesta de lista**:
  ```json
  {
    "data": [...],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 150,
      "total_pages": 8
    }
  }
  ```

### BR-15: Autenticación obligatoria en todos los endpoints
- **Severidad**: CRITICA
- **Regla**: No existe ningún endpoint público (sin autenticación) excepto:
  - `GET /health` — healthcheck del sistema
  - `GET /api/v1/auth/callback` — callback de OAuth2
  - `GET /api/v1/auth/login` — inicio del flujo OAuth2
- Todo lo demás requiere Bearer Token válido en header `Authorization`.

### BR-16: Rate limiting por usuario
- **Severidad**: MEDIA
- **Regla**: Máximo 100 requests por minuto por usuario autenticado. Al superarlo, la API devuelve HTTP 429 con el tiempo de espera en header `Retry-After`.

---

## Sección 4 — WebSocket (C-14)

### BR-17: Autenticación requerida para conexión WebSocket
- **Severidad**: CRITICA
- **Regla**: La conexión WebSocket en `/ws/{portafolio_id}` requiere un token válido enviado en el primer mensaje tras la conexión (handshake de autenticación). Si el token no se envía en los primeros 5 segundos, la conexión se cierra con código `4001` (Unauthorized).

### BR-18: Eventos WebSocket — Catálogo completo MVP
- **Severidad**: ALTA
- **Regla**: El `GestorWebSocket` publica exactamente estos 6 tipos de eventos:

| Tipo de Evento | Disparador | Destinatarios |
|---------------|------------|---------------|
| `PROGRESO_ANALISIS` | Cada cotización procesada | `ANALISTA` del portafolio |
| `PDF_PROCESADO` | Cada PDF sanitizado | `ANALISTA` del portafolio |
| `RESPUESTA_PROVEEDOR` | Aclaración respondida | `ANALISTA` del portafolio |
| `ERROR_PROCESAMIENTO` | Fallo en pipeline IA | `ANALISTA` del portafolio |
| `ANALISIS_COMPLETADO` | Portafolio en estado `COMPLETADO` | `ANALISTA` del portafolio |
| `ALERTA_SEGURIDAD` | PDF con `estado_sanitizacion = ADVERSARIAL` | `ADMIN` + `CISO` |

### BR-19: Formato estándar de mensaje WebSocket
- **Severidad**: ALTA
- **Regla**: Todos los mensajes WebSocket deben seguir este esquema:
  ```json
  {
    "tipo": "NOMBRE_EVENTO",
    "payload": { ... },
    "timestamp": "2026-05-25T17:00:00Z"
  }
  ```
- El `timestamp` es UTC en formato ISO 8601. El campo `tipo` es inmutable (del catálogo BR-18).

### BR-20: Reconexión con backoff exponencial (lado cliente)
- **Severidad**: MEDIA
- **Regla**: El cliente Angular gestiona la reconexión automática con backoff exponencial:
  - Intento 1: esperar 1 segundo
  - Intento 2: esperar 2 segundos
  - Intento 3: esperar 4 segundos
  - Intento N: esperar `min(2^(N-1), 60)` segundos (máximo 60 segundos entre intentos)
- El servidor no retiene estado de eventos para clientes desconectados (stateless).
- Máximo de reintentos: 10. Tras agotar reintentos, mostrar mensaje al usuario pidiendo recargar la página.

### BR-21: Suscripción por portafolio
- **Severidad**: ALTA
- **Regla**: Un cliente WebSocket se suscribe a los eventos de un portafolio específico. El endpoint es `/ws/{portafolio_id}`. Un usuario puede tener múltiples conexiones WebSocket activas (para múltiples portafolios en tabs diferentes).
- El servidor valida que el `ANALISTA` conectado sea el `analista_id` del portafolio. Un `ANALISTA` no puede suscribirse al WebSocket de un portafolio de otro analista (HTTP 403 al intentar la conexión).

---

## Sección 5 — Reglas Transversales

### BR-22: Timestamps siempre en UTC
- **Severidad**: ALTA
- **Regla**: Todos los timestamps almacenados en BD y devueltos por la API son UTC. El formato es ISO 8601: `2026-05-25T17:00:00Z`. El frontend es responsable de convertir a la zona horaria del usuario.

### BR-23: Registro obligatorio en AuditTrail para acciones críticas
- **Severidad**: CRITICA
- **Regla**: Las siguientes acciones **siempre** generan un registro en `AuditTrail`, independientemente del resultado (éxito o fallo):
  - Login exitoso / fallido
  - Acceso denegado (403)
  - Inicio de análisis de portafolio
  - Cambio de estado de portafolio
  - Aprobación/revisión HITL de variables
  - Generación y descarga de reportes
  - Detección de PDF adversarial
  - Cambio de rol de usuario
  - Modificación de configuración del sistema

### BR-24: Validación de input en capa API (nunca en capa repositorio)
- **Severidad**: ALTA
- **Regla**: Toda validación de datos de entrada (tipos, longitudes, formatos, rangos) se realiza en los schemas Pydantic de la capa API. La capa de repositorio asume que los datos ya fueron validados y no duplica validaciones.

---
