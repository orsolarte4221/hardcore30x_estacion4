# Component Methods — Assistent Buy AI Agent

> **Versión**: 1.0 (Application Design)  
> **Nota**: Las firmas de métodos son de alto nivel. Las reglas de negocio detalladas se definen en **Functional Design** (fase CONSTRUCTION).

---

## C-01 — IngestaConector

```python
class IngestaConector:

    async def iniciar_ingesta_portafolio(
        proceso_id: str,
        usuario_id: str
    ) -> EstadoIngesta:
        """
        Inicia la ingesta completa de un portafolio de cotizaciones desde SAB.
        Recupera datos estructurados + PDFs de todas las cotizaciones del proceso.
        Retorna estado de ingesta (total_cotizaciones, iniciado_en, ID de trabajo).
        """

    async def obtener_cotizaciones_proceso(
        proceso_id: str
    ) -> list[CotizacionSAB]:
        """
        Retorna la lista completa de cotizaciones de un proceso desde SAB (read-only).
        Cada CotizacionSAB incluye: ID, proveedor, items, precios, fechas, rutas_pdf.
        """

    async def recuperar_pdf(
        ruta_filesystem: str,
        cotizacion_id: str,
        pdf_nombre: str
    ) -> ResultadoRecuperacionPDF:
        """
        Recupera un PDF desde el filesystem de SAB y lo pasa al middleware de sanitización.
        Retorna: estado (ok/corrupto/no_encontrado), bytes_sanitizados si ok.
        """

    async def reportar_estado_cotizacion(
        cotizacion_id: str,
        estado: EstadoIngesta  # Completa | Parcial | NoProcesable
    ) -> None:
        """
        Actualiza el estado de ingesta de una cotización en el RepositorioLocal.
        """
```

---

## C-02 — SanitizadorZeroTrust

```python
class SanitizadorZeroTrust:

    async def sanitizar(
        pdf_bytes: bytes,
        metadata_origen: MetadataOrigen  # proveedor_id, cotizacion_id, nombre_archivo
    ) -> PDFSanitizado:
        """
        Pipeline Zero Trust completo: detección y neutralización de RT1-RT4 + Spotlighting.
        Retorna PDFSanitizado con: contenido_limpio, estado (Limpio|ElementosDetectados|NoSanitable),
        elementos_detectados[], referencia_origen.
        IMPORTANTE: Si estado == NoSanitable, el campo contenido_limpio es None.
        """

    async def aplicar_spotlighting(
        contenido: str,
        metadata_origen: MetadataOrigen
    ) -> str:
        """
        Inserta tokens de procedencia que distinguen instrucciones del sistema
        de contenido externo. Se aplica siempre, incluso en PDFs sin elementos adversariales.
        """

    def registrar_evento_seguridad(
        pdf_id: str,
        tipo_elemento: str,
        fragmento_neutralizado: str,
        timestamp: datetime
    ) -> None:
        """
        Registra el evento en el ModuloAdmin. Llamado internamente durante sanitizar().
        """
```

---

## C-03 — MotorExtraccion

```python
class MotorExtraccion:

    async def extraer_variables(
        pdf_sanitizado: PDFSanitizado,
        variables_objetivo: list[str],  # lista de variables a extraer según tipo de cotización
        umbral_confianza: float = 0.6   # default configurable
    ) -> ResultadoExtraccion:
        """
        Extrae variables de riesgo documental del PDF sanitizado.
        Proceso: Document AI (OCR/texto) → Claude (interpretación semántica) → post-procesamiento.
        Retorna ResultadoExtraccion con:
          - variables: list[VariableExtraida(nombre, valor|None, confianza, referencia_fuente, requiere_revision)]
          - estado: Exitoso | Parcial | Fallido
          - metadata: {documento, paginas_procesadas, duracion_ms, tokens_usados}
        Variables con confianza < umbral tienen valor=None y requiere_revision=True.
        """

    async def extraer_texto_document_ai(
        pdf_bytes: bytes,
        documento_id: str
    ) -> TextoExtraido:
        """
        Invoca IGatewayDocumentAI para OCR y extracción estructurada de texto.
        Retorna TextoExtraido con texto_plano, bloques[], tablas[], referencia_paginas[].
        """

    async def interpretar_con_llm(
        texto_extraido: TextoExtraido,
        variables_objetivo: list[str],
        metadata_origen: MetadataOrigen
    ) -> list[VariableLLM]:
        """
        Construye el prompt estructurado con tokens Spotlighting y envía a IGatewayLLM.
        El prompt instruce al LLM a extraer variables y retornar en formato estructurado JSON.
        Nunca incluye datos de otras cotizaciones en el prompt (anti-EchoLeak).
        """
```

---

## C-04 — MotorCruce

```python
class MotorCruce:

    async def cruzar_cotizacion(
        cotizacion_sab: CotizacionSAB,
        resultado_extraccion: ResultadoExtraccion
    ) -> ResultadoCruce:
        """
        Cruza campo a campo los datos de SAB vs. las variables extraídas del PDF.
        Retorna ResultadoCruce con:
          - discrepancias: list[Discrepancia(campo, valor_sab, valor_pdf, severidad, referencia_fuente, requiere_aclaracion)]
          - cobertura: float  (% de campos de SAB encontrados en el PDF)
          - variables_sin_fuente: list[str]  (campos SAB ausentes en el PDF)
        """

    def clasificar_discrepancia(
        campo: str,
        valor_sab: Any,
        valor_pdf: Any,
        contexto: dict
    ) -> Severidad:  # Alta | Media | Baja
        """
        Aplica las reglas de negocio de clasificación (detalladas en Functional Design).
        Determina la severidad de la discrepancia según el tipo de campo y la magnitud de la diferencia.
        """

    async def identificar_variables_para_aclaracion(
        resultado_cruce: ResultadoCruce
    ) -> list[SolicitudAclaracion]:
        """
        Identifica qué discrepancias de alta severidad y qué variables con baja confianza
        requieren solicitud de aclaración al proveedor.
        """
```

---

## C-05 — GestorAclaraciones

```python
class GestorAclaraciones:

    async def generar_borrador(
        solicitudes: list[SolicitudAclaracion],
        cotizacion: CotizacionSAB,
        analista_id: str
    ) -> Borrador:
        """
        Genera un borrador de email en lenguaje formal dirigido al proveedor.
        El borrador referencia específicamente las variables/discrepancias que motivaron la solicitud.
        Estado inicial: Pendiente.
        """

    async def aprobar_y_enviar(
        borrador_id: str,
        contenido_editado: str | None,  # None si no se editó
        analista_id: str
    ) -> ResultadoEnvio:
        """
        Registra la aprobación en el audit trail (analista_id, timestamp, contenido_original, contenido_enviado).
        Invoca el envío SMTP con el buzón monitorizado como remitente.
        El asunto del email incluye el ID del borrador para correlación IMAP.
        """

    async def descartar_borrador(
        borrador_id: str,
        motivo: str | None,
        analista_id: str
    ) -> None:
        """
        Registra el descarte en el audit trail. No envía ninguna comunicación.
        """

    async def iniciar_polling_imap(self) -> None:
        """
        Asyncio background task que se inicia al arrancar el servidor FastAPI.
        Ejecuta poll_imap_iteration() cada N minutos (default: 5, configurable).
        Corre en el event loop del proceso FastAPI — no requiere proceso externo.
        """

    async def poll_imap_iteration(self) -> None:
        """
        Una iteración del ciclo IMAP: conectar al buzón, leer emails no procesados,
        correlacionar con solicitudes activas (por thread ID o asunto con ID),
        extraer PDFs adjuntos y delegarlos al ServicioPipeline para reingesta.
        Registra en audit trail: timestamp, remitente, asunto, n_adjuntos, correlación.
        """

    async def correlacionar_respuesta(
        email: EmailRecibido
    ) -> Aclaracion | None:
        """
        Busca en el RepositorioLocal la aclaración cuyo ID de borrador aparece en el
        thread ID o asunto del email. Retorna None si no hay correlación.
        """

    async def notificar_analista(
        aclaracion_id: str,
        email_recibido: EmailRecibido
    ) -> None:
        """
        Publica una notificación al GestorWebSocket indicando que el proveedor respondió.
        Incluye: aclaracion_id, proveedor, tiene_adjuntos (bool).
        """
```

---

## C-06 — GeneradorReportes

```python
class GeneradorReportes:

    async def generar_sintesis_proveedor(
        cotizacion_id: str
    ) -> SintesisProveedor:
        """
        Consolida para un proveedor: variables extraídas con confianza y referencias,
        discrepancias detectadas con severidad y fuente. Incluye aclaraciones respondidas.
        """

    async def generar_reporte_comparativo(
        proceso_id: str
    ) -> ReporteComparativo:
        """
        Genera el reporte consolidado del proceso completo:
        - Ranking de proveedores (configurable por criterio)
        - Matriz de cumplimiento por variable/criterio
        - Riesgos identificados ordenados por severidad
        - Trazabilidad completa: cada dato con cita de fuente
        Aplica restricciones de lenguaje: sin "fraude", "colusión", "culpable".
        """

    async def exportar_a_pdf(
        reporte_id: str
    ) -> bytes:
        """
        Genera el archivo PDF del reporte comparativo con formato oficial.
        Incluye encabezado con: proceso, fecha, analista que generó.
        """
```

---

## C-07 — ModuloAdmin

```python
class ModuloAdmin:

    def registrar_log(
        componente: str,
        operacion: str,
        nivel: LogLevel,  # INFO | WARNING | ERROR | SECURITY
        contexto: dict,
        timestamp: datetime
    ) -> None:
        """
        Registra un evento en el log estructurado (append-only). Síncrono para no bloquear.
        """

    async def registrar_evento_seguridad(
        tipo_evento: str,
        componente_origen: str,
        detalle: dict,  # tipo_ataque, fragmento, pdf_id, etc.
        timestamp: datetime
    ) -> None:
        """
        Registra un evento de seguridad en la tabla de AuditTrail con nivel SECURITY.
        Genera una alerta visible al CISO si el nivel es CRÍTICO.
        """

    async def listar_logs(
        filtros: FiltrosLog,  # rango_fechas, componente, nivel, proceso_id
        rol_solicitante: Rol
    ) -> list[LogEvento]:
        """
        Retorna logs filtrados. El CISO solo ve logs de nivel SECURITY.
        El Admin ve todos los niveles. El Analista no tiene acceso.
        """

    async def listar_alertas_activas(
        rol_solicitante: Rol
    ) -> list[Alerta]:
        """
        Retorna alertas no marcadas como revisadas para el rol solicitante.
        """
```

---

## C-08 — AutenticadorOAuth

```python
class AutenticadorOAuth:

    async def iniciar_flujo_oauth(
        proveedor: Proveedor  # Google | Microsoft
    ) -> str:
        """
        Genera y retorna la URL de autorización del proveedor OAuth2 con PKCE.
        """

    async def completar_flujo_oauth(
        codigo_autorizacion: str,
        state: str,
        code_verifier: str
    ) -> Sesion:
        """
        Intercambia el código por tokens. Valida el token ID. Asigna el rol según configuración.
        Crea la sesión en el RepositorioLocal y retorna los tokens de sesión.
        """

    async def validar_token(
        token: str
    ) -> ContextoAutenticacion | None:
        """
        Valida el token de sesión en el servidor. Retorna el contexto (usuario_id, rol, exp)
        o None si el token es inválido/expirado. Invocado en cada request por el middleware.
        """

    async def cerrar_sesion(
        token: str
    ) -> None:
        """
        Invalida el token en el servidor (no solo en el cliente). Registro en audit trail.
        """

    def requerir_rol(
        roles_permitidos: list[Rol]
    ) -> Callable:
        """
        Dependency de FastAPI que verifica que el usuario autenticado tenga el rol requerido.
        Retorna HTTP 403 si el rol no está en la lista de permitidos.
        """
```

---

## C-09 — ServicioPipeline

```python
class ServicioPipeline:

    async def ejecutar_analisis_portafolio(
        proceso_id: str,
        usuario_id: str
    ) -> str:  # job_id
        """
        Punto de entrada del pipeline. Orquesta:
          1. IngestaConector.iniciar_ingesta_portafolio()
          2. Para cada cotización + PDF:
             a. [SanitizadorZeroTrust middleware automático]
             b. MotorExtraccion.extraer_variables()
             c. MotorCruce.cruzar_cotizacion()
             d. GestorAclaraciones.generar_borrador() (si hay discrepancias)
          3. GeneradorReportes.generar_sintesis_proveedor() (por proveedor)
          4. Notificar finalización via GestorWebSocket
        El pipeline corre como asyncio task para no bloquear la API.
        Publica actualizaciones de progreso al GestorWebSocket en cada paso.
        """

    async def reingestar_respuesta_proveedor(
        cotizacion_id: str,
        pdfs_respuesta: list[bytes],
        aclaracion_id: str
    ) -> None:
        """
        Sub-pipeline para PDFs recibidos vía IMAP.
        Ejecuta: [SanitizadorZeroTrust] → MotorExtraccion → MotorCruce (actualización delta) → síntesis actualizada.
        Actualiza el estado de la aclaración a "Respondida".
        """

    def publicar_progreso(
        proceso_id: str,
        cotizacion_actual: int,
        total_cotizaciones: int,
        estado_cotizacion: EstadoIngesta,
        cotizacion_nombre: str
    ) -> None:
        """
        Publica el progreso actual al GestorWebSocket para transmisión al frontend.
        Formato: {tipo: 'progreso', proceso_id, cotizacion_actual, total, estado, nombre}
        """
```

---

## C-10 — APIRest (Endpoints FastAPI)

```python
# ── Autenticación ──────────────────────────────────────────────────────────
GET  /auth/login/{proveedor}         → URL de OAuth2
GET  /auth/callback                  → Sesion (completar flujo OAuth2)
POST /auth/logout                    → 204 (invalidar sesión)

# ── Portafolios y Pipeline ─────────────────────────────────────────────────
GET  /procesos                       → list[ProcesoSAB]  (procesos activos en SAB)
POST /procesos/{proceso_id}/analisis → JobAnalisis  (inicia el pipeline)
GET  /procesos/{proceso_id}/estado   → EstadoPortafolio  (progreso general)
WS   /procesos/{proceso_id}/ws       → WebSocket stream de progreso

# ── Análisis: Cotizaciones ─────────────────────────────────────────────────
GET  /cotizaciones/{cotizacion_id}/variables    → list[VariableExtraida]
GET  /cotizaciones/{cotizacion_id}/discrepancias → list[Discrepancia]
GET  /cotizaciones/{cotizacion_id}/estado       → EstadoCotizacion

# ── Aclaraciones ───────────────────────────────────────────────────────────
GET  /aclaraciones                              → list[Borrador] (filtrado por rol)
GET  /aclaraciones/{borrador_id}                → Borrador
POST /aclaraciones/{borrador_id}/aprobar        → ResultadoEnvio
POST /aclaraciones/{borrador_id}/descartar      → 204
GET  /aclaraciones/historial                    → list[AclaracionHistorico]

# ── Reportes ───────────────────────────────────────────────────────────────
GET  /reportes/{proceso_id}/sintesis            → SintesisComparativa
POST /reportes/{proceso_id}/generar             → ReporteComparativo
GET  /reportes/{reporte_id}/pdf                 → bytes (descarga PDF)

# ── Administración ─────────────────────────────────────────────────────────
GET  /admin/logs                                → list[LogEvento]  (Admin/CISO)
GET  /admin/alertas                             → list[Alerta]
POST /admin/alertas/{alerta_id}/revisar         → 204
GET  /admin/sanitizacion/{proceso_id}           → list[EventoSanitizacion]  (CISO)

# ── Visor de PDF (RF-07.5) ────────────────────────────────────────────────
GET  /pdfs/{pdf_id}/fragmento?pagina=N&seccion=X  → FragmentoPDF  (imagen o HTML renderizado de la página)
GET  /pdfs/{pdf_id}/metadata                     → MetadataPDF  (nombre, páginas, tamaño, proveedor)
```

---

## C-11 — GatewayLLM

```python
class IGatewayLLM(Protocol):

    async def completar(
        prompt_sistema: str,
        prompt_usuario: str,
        schema_respuesta: dict  # JSON Schema del output esperado
    ) -> RespuestaLLM:
        """
        Envía el prompt al LLM y retorna la respuesta estructurada.
        Retorna RespuestaLLM con: contenido (JSON), tokens_usados, duracion_ms.
        Lanza LLMError si timeout, rate limit, o error de API.
        """


class AnthropicClaudeGateway(IGatewayLLM):

    async def completar(
        prompt_sistema: str,
        prompt_usuario: str,
        schema_respuesta: dict
    ) -> RespuestaLLM: ...
    # Implementación concreta usando anthropic SDK
    # Gestiona chunking si el contenido excede el contexto del modelo
    # Registra métricas en ModuloAdmin
```

---

## C-12 — RepositorioSAB

```python
class IRepositorioSAB(Protocol):

    async def listar_cotizaciones(proceso_id: str) -> list[CotizacionSAB]: ...
    async def obtener_items_cotizacion(cotizacion_id: str) -> list[ItemSAB]: ...
    async def obtener_rutas_pdf(cotizacion_id: str) -> list[RutaPDF]: ...
    async def obtener_datos_proveedor(proveedor_id: str) -> ProveedorSAB: ...
    # IMPORTANTE: ningún método tiene firma write (INSERT/UPDATE/DELETE)


class RepositorioSABSQL(IRepositorioSAB):
    # Implementación con SQLAlchemy, conexión read-only enforced a nivel de usuario de BD
    ...
```

---

## C-13 — RepositorioLocal

```python
class RepositorioLocal:

    # ── Portafolios y Cotizaciones ─────────────────────────────────────────
    async def crear_portafolio(portafolio: NuevoPortafolio) -> Portafolio: ...
    async def actualizar_estado_cotizacion(cotizacion_id: str, estado: EstadoIngesta) -> None: ...

    # ── Variables y Discrepancias ──────────────────────────────────────────
    async def guardar_resultado_extraccion(resultado: ResultadoExtraccion) -> None: ...
    async def guardar_resultado_cruce(resultado: ResultadoCruce) -> None: ...

    # ── Aclaraciones ───────────────────────────────────────────────────────
    async def crear_borrador(borrador: NuevoBorrador) -> Borrador: ...
    async def registrar_aprobacion(borrador_id: str, evento: EventoAprobacion) -> None: ...  # append-only
    async def registrar_respuesta_imap(aclaracion_id: str, evento: EventoRespuesta) -> None: ...  # append-only

    # ── Audit Trail ────────────────────────────────────────────────────────
    async def insertar_evento_audit(evento: EventoAudit) -> None: ...  # solo INSERT, nunca UPDATE/DELETE

    # ── Logs y Alertas ─────────────────────────────────────────────────────
    async def insertar_log(log: NuevoLog) -> None: ...
    async def crear_alerta(alerta: NuevaAlerta) -> None: ...
    async def marcar_alerta_revisada(alerta_id: str, usuario_id: str) -> None: ...

    # ── Sesiones ───────────────────────────────────────────────────────────
    async def crear_sesion(sesion: NuevaSesion) -> Sesion: ...
    async def invalidar_sesion(token: str) -> None: ...
```

---

## C-14 — GestorWebSocket

```python
class GestorWebSocket:

    async def conectar(
        websocket: WebSocket,
        proceso_id: str,
        usuario_id: str
    ) -> None:
        """
        Registra la conexión WebSocket activa para el proceso/usuario dado.
        Valida el token de autenticación durante el handshake.
        """

    async def desconectar(
        websocket: WebSocket,
        proceso_id: str
    ) -> None: ...

    async def publicar_progreso(
        proceso_id: str,
        mensaje: MensajeProgreso
    ) -> None:
        """
        Publica un mensaje de progreso a todos los clientes conectados al proceso_id dado.
        Si no hay clientes conectados, descarta silenciosamente (no hay pérdida de datos
        porque el estado persiste en PostgreSQL vía RepositorioLocal).
        """

    async def publicar_notificacion(
        usuario_id: str,
        notificacion: MensajeNotificacion
    ) -> None:
        """
        Publica una notificación dirigida a un usuario específico (ej: respuesta IMAP recibida).
        """
```

---

## C-16 — GatewayDocumentAI

```python
class IGatewayDocumentAI(Protocol):

    async def extraer_texto(
        pdf_bytes: bytes,
        documento_id: str
    ) -> TextoExtraido:
        """
        Envía el PDF al servicio cloud de Document AI y retorna texto estructurado.
        Retorna TextoExtraido con:
          - texto_plano: str
          - bloques: list[BloqueTexto(contenido, pagina, bbox)]
          - tablas: list[TablaExtraida(filas, columnas, pagina)]
          - referencia_paginas: list[ReferenciaPagina(numero, texto)]
        Lanza DocumentAIError si timeout, cuota excedida, o error de API.
        """


class GoogleDocumentAIGateway(IGatewayDocumentAI):

    async def extraer_texto(
        pdf_bytes: bytes,
        documento_id: str
    ) -> TextoExtraido: ...
    # Implementación concreta usando google-cloud-documentai SDK
    # Gestiona paginación si el PDF tiene muchas páginas
    # Registra métricas (páginas procesadas, latencia, costo) en ModuloAdmin
```
