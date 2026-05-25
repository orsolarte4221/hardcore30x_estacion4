# Servicios — Assistent Buy AI Agent

> **Versión**: 1.0 (Application Design)  
> Los servicios son los orquestadores de alto nivel que coordinan múltiples componentes para ejecutar un caso de uso completo.

---

## Mapa de Servicios

| ID | Servicio | Rol | Casos de uso que orquesta |
|---|---|---|---|
| S-01 | `ServicioPipeline` | Orquestador central | US-01, US-02, US-04, US-05, US-06, US-07 |
| S-02 | `ServicioAclaraciones` | Sub-orquestador HITL | US-09, US-10, US-11, US-12, US-13, US-14 |
| S-03 | `ServicioReportes` | Orquestador de reportes | US-15, US-16, US-17, US-18 |
| S-04 | `ServicioAutenticacion` | Orquestador de auth | US-23 |
| S-05 | `ServicioAdmin` | Orquestador de administración | US-19, US-20, US-22 |

> **Nota**: `ServicioAclaraciones` corresponde al componente `GestorAclaraciones` (C-05) en su rol de servicio. `ServicioPipeline` corresponde al componente C-09.

---

## S-01 — ServicioPipeline

**Descripción**: Orquestador central del análisis. Es el único punto de entrada para ejecutar el análisis de un portafolio. Gestiona el estado en memoria y coordina todos los componentes del pipeline.

### Flujo de Orquestación Principal

```
Analista activa análisis
       │
       ▼
ServicioPipeline.ejecutar_analisis_portafolio(proceso_id)
       │
       ├─── 1. IngestaConector.iniciar_ingesta_portafolio()
       │         │
       │         └─── Para cada cotización:
       │               IngestaConector.obtener_cotizaciones_proceso()
       │               IngestaConector.recuperar_pdf() × N PDFs
       │                    │
       │                    └─── [automático] SanitizadorZeroTrust.sanitizar()
       │                              │
       │                              ├─── Si ok → PDFSanitizado
       │                              └─── Si NoSanitable → log + skip PDF
       │
       ├─── 2. Para cada PDF sanitizado:
       │         MotorExtraccion.extraer_variables(pdf_sanitizado)
       │              │
       │              └─── → ResultadoExtraccion
       │
       ├─── 3. Para cada cotización completa (todos sus PDFs):
       │         MotorCruce.cruzar_cotizacion(datos_sab, resultado_extraccion)
       │              │
       │              └─── → ResultadoCruce
       │
       ├─── 4. GestorAclaraciones.generar_borrador() [si hay discrepancias altas]
       │
       ├─── 5. GeneradorReportes.generar_sintesis_proveedor() [por proveedor]
       │
       └─── 6. GestorWebSocket.publicar_progreso() [en cada paso]
                    → Notificación final al Analista: "Análisis completo"
```

### Sub-flujo: Reingesta de Respuesta de Proveedor

```
GestorAclaraciones detecta email IMAP con adjuntos
       │
       ▼
ServicioPipeline.reingestar_respuesta_proveedor(cotizacion_id, pdfs_respuesta)
       │
       ├─── [automático] SanitizadorZeroTrust.sanitizar() × PDFs adjuntos
       ├─── MotorExtraccion.extraer_variables() [variables adicionales/corregidas]
       ├─── MotorCruce.cruzar_cotizacion() [actualización delta]
       ├─── GeneradorReportes.generar_sintesis_proveedor() [síntesis actualizada]
       └─── GestorWebSocket.publicar_notificacion() → "Síntesis actualizada"
```

### Gestión de Estado en Memoria

```python
# Estado en memoria del ServicioPipeline (por proceso activo)
estado_pipeline: dict[str, EstadoProceso] = {
    "proceso_id": EstadoProceso(
        total_cotizaciones=18,
        cotizaciones_completadas=12,
        cotizaciones_parciales=2,
        cotizaciones_error=1,
        estado_actual="extrayendo_variables",
        iniciado_en=datetime,
        conexiones_ws=[WebSocket, ...]
    )
}
# El estado se persiste en PostgreSQL en cada actualización para durabilidad.
# El estado en memoria se usa para publicaciones WebSocket de baja latencia.
```

### Manejo de Errores del Pipeline

| Error | Comportamiento |
|---|---|
| PDF corrupto/no encontrado | Marca cotización como Parcial; continúa con las demás |
| Sanitización incompleta (NoSanitable) | Marca PDF como no procesado; alerta de seguridad; continúa |
| Timeout de Document AI | Reintentos × 2, luego marca variable con baja confianza |
| Timeout de LLM Claude | Reintentos × 2, luego escala al Analista con flag de revisión |
| Error de BD | Log + alerta al Admin; el análisis en memoria se intenta persistir |

---

## S-02 — ServicioAclaraciones (GestorAclaraciones en su rol de servicio)

**Descripción**: Gestiona el ciclo completo de comunicación HITL. Combina la gestión de borradores (flujo síncrono, iniciado por el Analista) con el polling IMAP (flujo asíncrono, iniciado por el background task).

### Flujo Síncrono: Gestión de Borradores

```
MotorCruce detecta discrepancias altas → GestorAclaraciones.generar_borrador()
                                                    │
                                                    └─── Borrador guardado en RepositorioLocal
                                                         Estado: Pendiente

Analista abre borrador → revisa → edita (opcional) → presiona "Aprobar"
       │
       ▼
APIRest POST /aclaraciones/{id}/aprobar
       │
       └─── GestorAclaraciones.aprobar_y_enviar()
               ├─── RepositorioLocal.registrar_aprobacion() [audit trail]
               ├─── SMTP.enviar(remitente=buzon_monitorizado, asunto=f"Aclaración ID-{borrador_id}")
               └─── Estado del borrador → "Enviado"
```

### Flujo Asíncrono: Polling IMAP

```
Al arrancar FastAPI:
asyncio.create_task(GestorAclaraciones.iniciar_polling_imap())
       │
       └─── Loop asyncio (cada 5 min, configurable):
               GestorAclaraciones.poll_imap_iteration()
                    │
                    ├─── IMAP.conectar(buzon_monitorizado)
                    ├─── IMAP.listar_emails_no_procesados()
                    │
                    └─── Para cada email:
                          ├─── correlacionar_respuesta(email) → Aclaracion | None
                          ├─── Si correlacionado:
                          │     ├─── Extraer PDFs adjuntos
                          │     ├─── ServicioPipeline.reingestar_respuesta_proveedor()
                          │     ├─── notificar_analista() vía GestorWebSocket
                          │     └─── RepositorioLocal.registrar_respuesta_imap() [audit trail]
                          └─── Si no correlacionado:
                                └─── ModuloAdmin.registrar_log(nivel=WARNING, detalle=email_metadata)
```

---

## S-03 — ServicioReportes

**Descripción**: Coordina la generación de los reportes de síntesis y comparativo. Lee datos consolidados del análisis y las aclaraciones para producir los documentos finales.

### Flujo: Síntesis Comparativa

```
Analista accede a la vista comparativa
       │
       ▼
APIRest GET /reportes/{proceso_id}/sintesis
       │
       └─── GeneradorReportes.generar_sintesis_proveedor(cotizacion_id)
               │
               ├─── RepositorioLocal.listar_variables(cotizacion_id)
               ├─── RepositorioLocal.listar_discrepancias(cotizacion_id)
               ├─── RepositorioLocal.listar_aclaraciones(cotizacion_id)
               └─── → SintesisProveedor (con 100% de datos con referencia de fuente)
```

### Flujo: Reporte para Comité + Export PDF

```
Analista presiona "Generar Reporte para Comité"
       │
       ▼
APIRest POST /reportes/{proceso_id}/generar
       │
       └─── GeneradorReportes.generar_reporte_comparativo(proceso_id)
               │
               ├─── Lee todas las síntesis de proveedores del proceso
               ├─── Genera ranking (criterio configurable)
               ├─── Genera matriz de cumplimiento
               ├─── Aplica restricciones de lenguaje (sin "fraude", "colusión", "culpable")
               ├─── Valida que 100% de los datos tienen referencia de fuente
               └─── → ReporteComparativo guardado en RepositorioLocal

Analista (o Comité) presiona "Exportar a PDF"
       │
       ▼
APIRest GET /reportes/{reporte_id}/pdf
       │
       └─── GeneradorReportes.exportar_a_pdf(reporte_id)
               └─── → bytes (descarga directa)
```

---

## S-04 — ServicioAutenticacion

**Descripción**: Orquesta el flujo completo OAuth2/OIDC. Es el punto de entrada para todas las operaciones de autenticación y autorización.

### Flujo: Login OAuth2 (PKCE)

```
Usuario accede a Assistent Buy
       │
       ▼
APIRest GET /auth/login/{proveedor}
       │
       └─── AutenticadorOAuth.iniciar_flujo_oauth(proveedor)
               └─── → URL de autorización (con PKCE code_verifier almacenado en sesión temporal)

Proveedor OAuth redirige a /auth/callback
       │
       ▼
APIRest GET /auth/callback?code=...&state=...
       │
       └─── AutenticadorOAuth.completar_flujo_oauth(codigo, state, code_verifier)
               ├─── Intercambia código por tokens con el proveedor
               ├─── Valida token ID (firma, audiencia, expiración)
               ├─── Asigna rol según email/grupo del usuario
               ├─── RepositorioLocal.crear_sesion()
               └─── → Redirige al frontend con token de sesión (cookie HttpOnly)
```

### Middleware de Autenticación (aplicado en cada request)

```
Cada request a la APIRest (excepto /auth/*)
       │
       └─── FastAPI Dependency: AutenticadorOAuth.validar_token(token)
               ├─── Si válido → ContextoAutenticacion (usuario_id, rol) inyectado en el endpoint
               └─── Si inválido → HTTP 401

       └─── FastAPI Dependency: AutenticadorOAuth.requerir_rol([roles_permitidos])
               ├─── Si rol permitido → continúa
               └─── Si rol no permitido → HTTP 403
```

---

## S-05 — ServicioAdmin

**Descripción**: Expone las capacidades de observabilidad y seguridad al Admin (Director) y CISO.

### Flujo: Panel de Logs (Admin)

```
Admin accede a /admin/logs
       │
       └─── ModuloAdmin.listar_logs(filtros, rol=Admin)
               └─── → Todos los niveles (INFO, WARNING, ERROR, SECURITY)
```

### Flujo: Panel de Sanitización (CISO)

```
CISO accede a /admin/sanitizacion/{proceso_id}
       │
       └─── ModuloAdmin.listar_logs(filtros={nivel=SECURITY, proceso_id}, rol=CISO)
               └─── → Solo eventos de sanitización del proceso
```

### Flujo: Gestión de Alertas

```
Alerta generada por SanitizadorZeroTrust o ServicioPipeline
       │
       └─── ModuloAdmin.registrar_evento_seguridad() → Alerta creada en RepositorioLocal

Admin/CISO ve alerta activa → marca como "Revisada"
       │
       └─── APIRest POST /admin/alertas/{id}/revisar
               └─── ModuloAdmin.marcar_alerta_revisada(alerta_id, usuario_id) [audit trail]
```

---

## Principios Transversales de la Capa de Servicios

### 1. Zero Trust Pipeline (todos los servicios)
- Ningún servicio acepta contenido de PDF sin que haya pasado por `SanitizadorZeroTrust`
- El middleware es automático: los servicios downstream solo reciben `PDFSanitizado`

### 2. Inmutabilidad del Audit Trail
- Todo evento de negocio significativo (aprobación, envío, recepción, descarte) se registra en `RepositorioLocal.insertar_evento_audit()` como append-only
- Ningún servicio puede modificar o eliminar entradas del audit trail

### 3. Principio HITL (Human-in-the-Loop)
- `ServicioAclaraciones` no puede enviar ningún email sin que exista un registro de aprobación explícita en el audit trail
- El `ServicioPipeline` no puede descartar cotizaciones automáticamente

### 4. Fail Closed para seguridad
- Si `SanitizadorZeroTrust` retorna `NoSanitable`, el `ServicioPipeline` descarta el PDF y registra el evento — nunca pasa contenido no sanitizado al `MotorExtraccion`

### 5. Trazabilidad obligatoria
- Cualquier dato que aparezca en un reporte generado por `ServicioReportes` debe tener una referencia de fuente verificable en `RepositorioLocal`
