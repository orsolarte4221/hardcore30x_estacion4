# User Stories — Assistent Buy AI Agent

> **Versión**: 1.0 (MVP)  
> **Formato**: "Como [persona], quiero [acción], para [beneficio]" + Criterios de Aceptación en Gherkin (Given/When/Then)  
> **Organización**: Por Journey de Usuario (Épicas)  
> **Idioma**: Español  
> **Total de stories**: 25 (22 Must Have · 3 Should Have)  
> **Referencia al PRD**: `assitant_buy_prd.md` v1.0 | Referencia de Personas: `personas.md`

---

## Índice de Épicas

| Épica | Descripción | Stories |
|---|---|---|
| **E-01** | Activación y Pre-análisis | US-01, US-02, US-03 |
| **E-02** | Análisis y Extracción | US-04, US-05, US-06, US-07, US-08 |
| **E-03** | Comunicación y Aclaración HITL | US-09, US-10, US-11, US-12, US-13, US-14 |
| **E-04** | Reportes y Decisión | US-15, US-16, US-17, US-18 |
| **E-05** | Seguridad y Administración | US-19, US-20, US-21, US-22, US-23 |
| **E-06** | Supervisión Avanzada (v1.1) | US-24, US-25 |

---

## Épica E-01 — Activación y Pre-análisis

> El Analista activa el análisis sobre un portafolio completo de SAB. El Sistema ingesta, valida y sanitiza todos los documentos.

---

### US-01 — Iniciar Análisis de Portafolio
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M1, M2  
**RF**: RF-01.1, RF-01.2, RF-01.3

> **Como** Analista de Compras,  
> **quiero** seleccionar un proceso de adquisición activo en SAB y activar Assistent Buy con un solo clic,  
> **para** que el sistema ingeste automáticamente las 100% de las cotizaciones y sus documentos adjuntos sin que yo tenga que subir archivos manualmente.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Inicio exitoso de análisis
  Given el Analista está autenticado en Assistent Buy via OAuth2
  And hay un proceso activo en SAB con cotizaciones y PDFs adjuntos
  When el Analista selecciona el proceso y presiona "Iniciar Análisis"
  Then el sistema se conecta a SAB en modo read-only y lee los datos estructurados de todas las cotizaciones
  And accede al filesystem del servidor SAB y recupera el 100% de los PDFs adjuntos
  And muestra un indicador de progreso en tiempo real con "Procesando X de Y cotizaciones"
  And la ingesta inicia en ≤ 5 segundos desde el clic

Escenario 2 (Límite de volumen): Portafolio dentro del límite MVP
  Given el portafolio tiene hasta 50 cotizaciones y hasta 200 PDFs
  When el Analista inicia el análisis
  Then el sistema acepta el portafolio completo sin error
  And el análisis completo termina en ≤ 60 segundos

Escenario 3 (Edge case): El proceso no tiene cotizaciones registradas
  Given el Analista selecciona un proceso sin cotizaciones en SAB
  When intenta iniciar el análisis
  Then el sistema muestra un mensaje: "Este proceso no tiene cotizaciones registradas en SAB"
  And no inicia ningún procesamiento
```

---

### US-02 — Ver Estado de Ingesta en Tiempo Real
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M1, M2  
**RF**: RF-01.4, RF-01.5

> **Como** Analista de Compras,  
> **quiero** ver en tiempo real cuántas cotizaciones y PDFs se están procesando y cuáles tienen problemas,  
> **para** saber si debo intervenir manualmente en documentos problemáticos sin esperar al final del análisis.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Progreso visible durante el análisis
  Given el sistema está procesando un portafolio
  When el Analista observa la pantalla de progreso
  Then ve el contador actualizado en tiempo real: "Procesando cotización 12 de 18"
  And ve el estado por cotización: ✅ Completa / ⚠️ Parcial / ❌ No procesable
  And ve el número de PDFs procesados vs. total por cada cotización

Escenario 2 (Edge case): PDF corrupto o no encontrado
  Given uno de los PDFs de una cotización no puede leerse
  When el sistema intenta ingerirlo
  Then marca la cotización como "Parcial" con indicador ⚠️
  And continúa procesando las demás cotizaciones sin interrupción
  And muestra el nombre del archivo problemático con el motivo del error

Escenario 3 (Edge case): Cotización completamente sin PDFs
  Given una cotización no tiene ningún documento adjunto
  When el sistema la procesa
  Then la marca como "No procesable" con indicador ❌ y motivo: "Sin anexos PDF"
  And notifica al Analista para acción manual
```

---

### US-03 — Sanitización Zero Trust Automática por el Sistema
**Persona**: P-05 Sistema Agente  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M3, M4 (Principio 3, RT1-RT4)  
**RF**: RF-02.1, RF-02.2, RF-02.3, RF-02.4, RF-02.5

> **Como** Sistema Agente,  
> **debo** sanitizar cada PDF entrante con el pipeline Zero Trust antes de enviarlo al LLM,  
> **para** garantizar que ningún vector de Inyección Indirecta de Prompts (IPI) pueda comprometer el análisis.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): PDF limpio pasa la sanitización
  Given un PDF de cotización sin contenido adversarial
  When el pipeline de sanitización lo procesa
  Then el PDF es marcado como "Sanitizado — Limpio"
  And el contenido del PDF se enriquece con tokens Spotlighting que indican su procedencia externa
  And el PDF sanitizado es enviado al LLM con las señales de procedencia

Escenario 2 (Seguridad): PDF con instrucción IPI detectada
  Given un PDF contiene una instrucción IPI (CSS oculto, homóglofos Unicode, metadatos adversariales, etc.)
  When el pipeline lo procesa
  Then la instrucción adversarial es neutralizada antes de cualquier envío al LLM
  And el PDF es marcado como "Sanitizado — Elementos adversariales detectados y neutralizados"
  And el evento es registrado en el log de sanitización con el tipo de ataque y el fragmento neutralizado
  And el análisis del contenido legítimo del PDF continúa normalmente

Escenario 3 (Seguridad - Métrica): Tasa de ataque exitoso sobre corpus D3
  Given el corpus D3 de PDFs adversariales conocidos
  When el pipeline sanitiza todos los PDFs del corpus D3
  Then la tasa de ataque exitoso (ASR) es < 2%
  And 0 instrucciones IPI llegan al LLM sin neutralizar

Escenario 4 (Seguridad - EchoLeak): Ningún dato interno es exfiltrado
  Given el sistema analiza un PDF que contiene instrucciones para exfiltrar datos internos
  When el pipeline procesa el documento
  Then el sistema no genera ninguna salida que contenga datos de SAB, otras cotizaciones, o credenciales
  And el número de incidentes EchoLeak es 0
```

---

## Épica E-02 — Análisis y Extracción

> El Sistema extrae variables de los PDFs, cruza con datos de SAB, clasifica discrepancias y escala cuando la confianza es insuficiente.

---

### US-04 — Ver Variables Extraídas por Proveedor
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M5  
**RF**: RF-03.1, RF-03.6, RF-03.7

> **Como** Analista de Compras,  
> **quiero** ver las variables clave extraídas de los documentos de cada proveedor (vigencias, garantías, penalidades, condiciones de pago, exclusiones),  
> **para** tener en una pantalla la información crítica de cada cotización sin leer manualmente cada PDF.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Variables extraídas con alta confianza
  Given un PDF ha sido sanitizado y analizado
  When el Analista navega a la vista de síntesis de un proveedor
  Then ve las variables extraídas organizadas por categoría: Vigencia, Garantías, Penalidades, Condiciones de Pago, Exclusiones
  And cada variable muestra su valor extraído, el indicador de confianza (Alta/Media/Baja) y la referencia exacta (PDF: nombre, página, sección)
  And puede hacer clic en la referencia para ver el fragmento fuente en el documento original

Escenario 2 (Métrica de precisión): Exactitud sobre corpus D2
  Given el corpus anotado D2 con ground truth de variables
  When el sistema extrae las variables del corpus D2
  Then la precisión de extracción es > 96% sobre todas las variables del corpus
  And el recall de variables críticas es > 95%

Escenario 3 (Métrica de alucinación): Sin datos sintéticos
  Given cualquier PDF del corpus de prueba
  When el sistema extrae variables
  Then la tasa de alucinación (datos no presentes en el PDF fuente) es < 1%
  And toda afirmación en la vista de síntesis tiene una referencia verificable (nunca vacía)
```

---

### US-05 — Ver Discrepancias Detectadas entre SAB y PDF
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M6  
**RF**: RF-03.2, RF-03.5

> **Como** Analista de Compras,  
> **quiero** ver las discrepancias detectadas entre los datos registrados en SAB (ítems, precios) y lo que realmente dice el PDF del proveedor,  
> **para** identificar los riesgos ocultos en la "letra chica" antes de la reunión de Comité.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Discrepancias detectadas y clasificadas
  Given el sistema cruzó los datos de SAB con las variables extraídas del PDF
  When el Analista ve la lista de discrepancias de un proveedor
  Then cada discrepancia muestra: campo en SAB, valor en SAB, valor en PDF, y severidad (🔴 Alta / 🟡 Media / 🟢 Baja)
  And cada discrepancia tiene referencia a su fuente: campo de BD + (PDF: nombre, página, sección)
  And las discrepancias de severidad Alta aparecen al tope de la lista

Escenario 2 (Sin discrepancias): Proveedor sin inconsistencias
  Given el PDF de un proveedor es 100% consistente con SAB
  When el Analista ve la vista de discrepancias
  Then ve el mensaje: "✅ Sin discrepancias detectadas para este proveedor"

Escenario 3 (Edge case): Campo en SAB no presente en PDF
  Given un campo registrado en SAB no aparece en ningún lugar del PDF
  When el sistema cruza los datos
  Then registra la discrepancia como "Campo ausente en PDF" con severidad Alta
  And lo lista como discrepancia para revisión del Analista
```

---

### US-06 — Consultar Indicador de Confianza y Fuente de una Variable
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M7 (Principio 4)  
**RF**: RF-03.3

> **Como** Analista de Compras,  
> **quiero** ver el indicador de confianza de cada variable extraída y poder navegar al fragmento exacto del documento fuente,  
> **para** decidir si confío en el dato del sistema o si debo verificarlo manualmente en el PDF.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Confianza Alta con referencia verificable
  Given una variable fue extraída con confianza Alta
  When el Analista hace clic en la referencia de la variable
  Then el sistema navega o resalta el fragmento exacto del PDF fuente (página + sección)
  And el fragmento en el PDF coincide con el valor mostrado en la síntesis

Escenario 2: Confianza Media — variable que requiere atención
  Given una variable fue extraída con confianza Media
  When el Analista la ve en la síntesis
  Then aparece con indicador visual 🟡 y tooltip: "Verificar esta variable en el documento fuente"
  And el link a la fuente sigue disponible para verificación manual

Escenario 3: Sin fuente — ningún dato sin referencia
  Given cualquier variable en cualquier síntesis
  Then toda variable tiene exactamente una referencia de fuente no vacía (PDF: página/sección o campo de BD)
  And ninguna variable aparece con referencia vacía o "N/A sin justificación"
```

---

### US-07 — Escalamiento Automático por Baja Confianza
**Persona**: P-05 Sistema Agente  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M8 (Principio 2)  
**RF**: RF-03.4

> **Como** Sistema Agente,  
> **debo** escalar automáticamente al Analista cuando la confianza de una variable cae bajo el umbral configurado,  
> **para** nunca inventar valores y mantener la tasa de alucinación < 1%.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Escalamiento visible al Analista
  Given una variable tiene confianza por debajo del umbral (configurable, default: < 60%)
  When el sistema la detecta durante el análisis
  Then marca la variable como "Requiere revisión humana" con indicador 🔴
  And notifica al Analista con el contexto: variable, PDF fuente, fragmento ambiguo
  And no genera ningún valor sintético para la variable
  And la variable aparece vacía con el flag de escalamiento

Escenario 2 (Seguridad - Métrica): Cero alucinaciones por variables escaladas
  Given variables con confianza bajo umbral
  Then la tasa de valores inventados (no presentes en ningún documento fuente) es 0% para estas variables
  And el 100% de las variables con baja confianza tienen el flag de escalamiento visible
```

---

### US-08 — Ver Cotizaciones con Estado Incompleto
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Journey 3  
**RF**: RF-01.4, RF-01.5

> **Como** Analista de Compras,  
> **quiero** ver de un vistazo qué cotizaciones quedaron con ingesta parcial o no procesable,  
> **para** decidir si solicito los documentos faltantes al proveedor o si descarto la cotización del proceso.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Lista diferenciada de estados de ingesta
  Given el análisis de un portafolio termina con estados mixtos
  When el Analista ve el panel de resumen del portafolio
  Then ve tres secciones: ✅ Completas / ⚠️ Parciales / ❌ No procesables
  And cada cotización en las secciones Parcial y No procesable tiene el motivo listado (PDF corrupto, PDF no encontrado, Sin anexos, etc.)

Escenario 2 (Edge case): Todas las cotizaciones completas
  Given todas las cotizaciones se ingestan exitosamente
  Then solo aparece la sección ✅ Completas
  And el mensaje muestra: "Cobertura total: 100% de cotizaciones procesadas"
```

---

## Épica E-03 — Comunicación y Aclaración HITL

> El Analista gestiona el ciclo completo de aclaración: revisión de borradores, aprobación, envío SMTP, monitoreo IMAP y reingesta de respuestas.

---

### US-09 — Revisar Borrador de Solicitud de Aclaración
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M9  
**RF**: RF-04.1, RF-04.2

> **Como** Analista de Compras,  
> **quiero** revisar los borradores de solicitudes de aclaración generados por el agente para cada ambigüedad o discrepancia,  
> **para** verificar que el contenido sea profesional y apropiado antes de enviarlo al proveedor.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Borrador generado y visible
  Given el sistema detectó una ambigüedad en el PDF de un proveedor
  When el Analista accede a la sección de Aclaraciones
  Then ve la lista de borradores pendientes con: proveedor, tipo de ambigüedad, fecha de generación
  And puede expandir cada borrador para ver el texto completo en lenguaje formal de negocio
  And el borrador referencia la variable específica que motivó la aclaración con su fuente

Escenario 2 (Usabilidad): Tiempo de revisión razonable
  Given el Analista revisa un borrador
  Then el tiempo total para leer, evaluar y tomar una decisión es ≤ 2 minutos por borrador
```

---

### US-10 — Editar y Aprobar Borrador de Aclaración
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M10  
**RF**: RF-04.2, RF-04.4, RF-04.5

> **Como** Analista de Compras,  
> **quiero** editar el borrador si es necesario y aprobarlo con mi nombre antes del envío,  
> **para** tener control total sobre qué se comunica al proveedor y que quede registrado que yo lo autoricé.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Aprobación sin edición
  Given el Analista revisó el borrador y está conforme
  When presiona "Aprobar y Enviar"
  Then el sistema registra la aprobación con: nombre del Analista, timestamp ISO 8601, contenido exacto del borrador aprobado
  And el estado del borrador cambia a "Aprobado — Pendiente de envío"

Escenario 2: Aprobación con edición
  Given el Analista modifica el texto del borrador
  When guarda los cambios y presiona "Aprobar y Enviar"
  Then el sistema registra la versión original del agente Y la versión editada por el Analista en el audit trail
  And envía la versión editada al proveedor

Escenario 3: Descarte de borrador
  Given el Analista decide que la aclaración no es necesaria
  When presiona "Descartar"
  Then el borrador se archiva con estado "Descartado" y el motivo (si el Analista lo ingresa)
  And no se envía ninguna comunicación al proveedor

Escenario 4 (Seguridad): Ningún envío sin aprobación
  Given cualquier borrador en el sistema
  Then ningún email es enviado al proveedor sin que exista un registro de aprobación explícita del Analista
  And el sistema no tiene ningún mecanismo de envío automático sin intervención humana
```

---

### US-11 — Envío de Aclaración por SMTP
**Persona**: P-05 Sistema Agente  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M10  
**RF**: RF-04.3

> **Como** Sistema Agente,  
> **debo** enviar la solicitud de aclaración aprobada por SMTP usando el buzón monitorizado como remitente,  
> **para** garantizar que la respuesta del proveedor llegue al canal IMAP que el sistema monitoriza.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Envío exitoso
  Given el Analista aprobó un borrador
  When el sistema envía el email
  Then usa el buzón dedicado como remitente (ej: aclaraciones@dominio.com)
  And el email es enviado en ≤ 30 segundos desde la aprobación
  And el asunto del email incluye el ID de la solicitud de aclaración para correlación futura
  And el estado del borrador cambia a "Enviado" con timestamp

Escenario 2 (Edge case): Fallo de SMTP
  Given el servidor SMTP no está disponible
  When el sistema intenta enviar
  Then reintenta 3 veces con backoff exponencial
  And si falla definitivamente, cambia el estado a "Error de envío" y notifica al Analista
  And no pierde el borrador aprobado
```

---

### US-12 — Recibir Notificación de Respuesta de Proveedor
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Journey 1 (paso 10)  
**RF**: RF-04.7, RF-01.6

> **Como** Analista de Compras,  
> **quiero** recibir una notificación cuando un proveedor responde a una solicitud de aclaración,  
> **para** saber inmediatamente que hay nueva información disponible y que el sistema la está procesando.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Notificación de respuesta recibida
  Given el proveedor respondió al email de aclaración
  And el sistema IMAP detectó el email en el buzón monitorizado
  When el sistema correlaciona la respuesta con su solicitud original
  Then notifica al Analista en la interfaz web: "Proveedor [nombre] respondió a la aclaración [ID]"
  And la notificación indica si la respuesta incluye documentos PDF adjuntos
  And el estado de la solicitud de aclaración cambia a "Respondida"

Escenario 2 (Edge case): Respuesta sin adjuntos
  Given el proveedor respondió solo con texto, sin PDFs adjuntos
  When el sistema procesa la respuesta
  Then notifica al Analista que la respuesta es solo texto
  And el texto de la respuesta queda disponible en el historial de la aclaración
  And no inicia pipeline de reingesta de PDFs (no hay adjuntos)
```

---

### US-13 — Monitoreo IMAP y Reingesta de Documentos de Respuesta
**Persona**: P-05 Sistema Agente  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Journey 1 (paso 10), M1  
**RF**: RF-01.6, RF-01.7, RF-01.8, RF-04.8

> **Como** Sistema Agente,  
> **debo** monitorizar el buzón IMAP cada 5 minutos, correlacionar respuestas con solicitudes originales y reingestar los PDF adjuntos al pipeline de análisis,  
> **para** cerrar el ciclo de aclaración sin intervención manual del Analista en la parte técnica.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Ciclo IMAP → Sanitización → Análisis completo
  Given el buzón IMAP tiene un email de respuesta con PDFs adjuntos
  When el sistema ejecuta el poll IMAP (default: cada 5 minutos)
  Then extrae los PDFs adjuntos del email
  And correlaciona el email con la solicitud de aclaración original (por thread ID o asunto con ID)
  And pasa los PDFs por el pipeline de sanitización Zero Trust
  And pasa los PDFs sanitizados al motor de análisis
  And actualiza la síntesis del proveedor con la información de los nuevos documentos
  And registra en el audit trail: email recibido, timestamp, número de adjuntos, correlación con solicitud ID

Escenario 2 (Configuración): Intervalo de polling configurable
  Given un Admin configura el intervalo de polling a 10 minutos
  When el sistema arranca
  Then usa el intervalo configurado en lugar del default (5 minutos)
  And registra el intervalo activo en los logs de sistema

Escenario 3 (Edge case): Email de respuesta no correlacionable
  Given un email llega al buzón IMAP que no corresponde a ninguna solicitud activa
  When el sistema intenta correlacionarlo
  Then lo marca como "Email no correlacionado" en los logs
  And notifica al Admin con el asunto y remitente para revisión manual
  And no inicia ningún pipeline de análisis

Escenario 4 (Seguridad): Audit trail completo
  Given cualquier email procesado por IMAP
  Then el audit trail registra: timestamp ISO 8601, remitente, asunto, número de adjuntos, solicitud correlacionada (o "no correlacionado"), resultado del pipeline
```

---

### US-14 — Ver Historial de Aclaraciones
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: §9 Trazabilidad  
**RF**: RF-04.5

> **Como** Analista de Compras,  
> **quiero** ver el historial completo de aclaraciones con su estado y la identidad de quién aprobó cada una,  
> **para** tener trazabilidad completa del proceso de comunicación ante una auditoría.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Historial completo visible
  Given hay aclaraciones en varios estados para un portafolio
  When el Analista accede a la sección de Historial de Aclaraciones
  Then ve todas las aclaraciones con: estado (Pendiente/Aprobado/Enviado/Respondida/Descartado), proveedor, fecha, Analista que aprobó/descartó
  And puede filtrar por estado o por proveedor
  And puede ver el contenido exacto del borrador enviado y la respuesta recibida (si existe)

Escenario 2 (Auditoría): Inmutabilidad del historial
  Given un borrador fue aprobado y enviado
  Then el registro de ese borrador no puede ser editado ni eliminado desde la interfaz
  And el audit trail en base de datos es append-only para estas entradas
```

---

## Épica E-04 — Reportes y Decisión

> El Analista genera el reporte comparativo. El Comité lo lee para tomar la decisión de adjudicación.

---

### US-15 — Ver Síntesis Comparativa de Proveedores
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M11  
**RF**: RF-05.1, RF-05.4, RF-05.5

> **Como** Analista de Compras,  
> **quiero** ver una síntesis comparativa de todos los proveedores del portafolio en una sola pantalla,  
> **para** identificar de un vistazo cuáles tienen más discrepancias o riesgos antes de preparar el reporte para el Comité.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Síntesis comparativa completa
  Given el análisis de un portafolio está completo
  When el Analista accede a la vista comparativa
  Then ve todos los proveedores en una tabla con: nombre, número de discrepancias (Alta/Media/Baja), variables extraídas, estado de documentación (completa/parcial)
  And puede ordenar la tabla por severidad de discrepancias
  And cada celda de la tabla tiene referencia a la fuente verificable (ningún dato flotante)

Escenario 2 (Lenguaje): Sin términos prohibidos
  Given la síntesis comparativa de cualquier portafolio
  Then ninguna celda, título, tooltip o mensaje usa las palabras: "fraude", "colusión", "culpable", "sospechoso"
  And los términos usados son exclusivamente: "discrepancia", "riesgo documental", "anomalía", "inconsistencia"
```

---

### US-16 — Generar Reporte Comparativo para el Comité
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M12  
**RF**: RF-05.2, RF-05.4

> **Como** Analista de Compras,  
> **quiero** generar el reporte comparativo consolidado para el Comité de Contratación con un clic,  
> **para** entregar al Comité un documento con ranking de proveedores, riesgos identificados y trazabilidad completa de cada dato.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Reporte generado con todos los elementos
  Given el análisis y las aclaraciones de un portafolio están completos
  When el Analista presiona "Generar Reporte para Comité"
  Then el sistema genera un reporte que incluye: ranking comparativo de proveedores, matriz de cumplimiento por criterio, riesgos identificados por severidad, trazabilidad de cada dato a su fuente
  And el reporte está listo en ≤ 30 segundos

Escenario 2 (Trazabilidad): 100% de datos con fuente
  Given el reporte generado
  Then cada dato, valor, y afirmación en el reporte tiene una cita de fuente verificable
  And el reporte no contiene ningún dato sin referencia
```

---

### US-17 — Acceder al Reporte en Modo Lectura (Comité)
**Persona**: P-04 Comité de Contratación  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: CU4  
**RF**: RF-07.3

> **Como** miembro del Comité de Contratación,  
> **quiero** acceder al reporte comparativo en modo de solo lectura desde un link compartido,  
> **para** revisar la información completa antes y durante la reunión de adjudicación sin necesidad de una cuenta de analista.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Acceso controlado al reporte
  Given el Analista compartió el link del reporte con el Comité
  When un miembro del Comité abre el link
  Then se autentica con su cuenta Google/Microsoft institucional (OAuth2)
  And ve el reporte en modo solo lectura (sin botones de análisis, aclaración o edición)
  And puede navegar por todas las secciones del reporte

Escenario 2 (Control de acceso): Miembro sin permisos no puede iniciar análisis
  Given un usuario con rol "Comité" autenticado en la plataforma
  When intenta acceder a la sección de "Iniciar Análisis" o "Gestionar Aclaraciones"
  Then el sistema le deniega el acceso con mensaje: "No tienes permisos para esta acción"
  And el intento queda registrado en el audit trail
```

---

### US-18 — Exportar Reporte a PDF
**Persona**: P-01 María (Analista)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: M12  
**RF**: RF-05.3

> **Como** Analista de Compras,  
> **quiero** exportar el reporte comparativo a PDF directamente desde la interfaz,  
> **para** compartirlo con el Comité como documento oficial del proceso sin depender del botón de imprimir del navegador.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Exportación exitosa a PDF
  Given el reporte comparativo está generado
  When el Analista presiona "Exportar a PDF"
  Then el sistema genera un archivo PDF fiel al contenido del reporte en ≤ 15 segundos
  And el PDF incluye: encabezado con nombre del proceso, fecha, y nombre del Analista que generó el reporte
  And el PDF se descarga directamente al equipo del Analista

Escenario 2: PDF listo para acta de reunión
  Given el PDF exportado
  Then tiene un formato legible para impresión (márgenes, fuente legible, tablas bien formateadas)
  And conserva todas las referencias de fuente del reporte original
```

---

## Épica E-05 — Seguridad y Administración

> El CISO audita el sistema. El Admin supervisa logs y alertas. El Analista se autentica con OAuth2.

---

### US-19 — Ver Log de Sanitización de PDFs (CISO)
**Persona**: P-03 Alejandro (CISO)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Módulo 6  
**RF**: RF-06.3, RNF-02 (SECURITY-03, SECURITY-14)

> **Como** CISO,  
> **quiero** ver el estado de sanitización de cada PDF analizado en cualquier portafolio,  
> **para** verificar que el pipeline Zero Trust funcionó correctamente y que ningún documento malicioso comprometió el análisis.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Log de sanitización completo
  Given el CISO accede al panel de seguridad
  When selecciona un portafolio procesado
  Then ve la lista de todos los PDFs del portafolio con su estado de sanitización: ✅ Limpio / ⚠️ Elementos detectados y neutralizados
  And para los PDFs con elementos adversariales, ve: tipo de elemento detectado, fragmento neutralizado, timestamp del evento

Escenario 2 (Auditoría): Logs inmutables
  Given el log de sanitización de cualquier portafolio
  Then el CISO puede ver pero NO puede editar o borrar ninguna entrada del log
  And el log tiene timestamps ISO 8601 para cada evento
  And los logs se retienen por mínimo 90 días (SECURITY-14)
```

---

### US-20 — Recibir Alertas de Seguridad por Elemento Adversarial
**Persona**: P-03 Alejandro (CISO)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Módulo 3 / Principio 3  
**RF**: RF-02.4, RNF-02 (SECURITY-14)

> **Como** CISO,  
> **quiero** recibir alertas inmediatas cuando el pipeline detecta elementos adversariales en un PDF,  
> **para** poder escalar el incidente de seguridad y notificar al proveedor si corresponde.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Alerta en tiempo real
  Given el pipeline detectó una instrucción IPI o elemento adversarial en un PDF
  When el evento es registrado
  Then el CISO recibe una alerta en la interfaz en ≤ 2 minutos desde la detección
  And la alerta contiene: nombre del proveedor, nombre del PDF, tipo de elemento detectado, severidad
  And la alerta queda en el panel de seguridad hasta que el CISO la marque como "Revisada"

Escenario 2: Alerta de intento de EchoLeak
  Given el sistema detecta un intento de exfiltración de datos (EchoLeak)
  Then la alerta tiene severidad CRÍTICA y aparece como la primera en el panel de seguridad
  And el sistema registra el incidente con 0 datos exfiltrados (EchoLeak = 0 en producción)
```

---

### US-21 — Rechazo Automático de PDFs que No Pasan Sanitización
**Persona**: P-05 Sistema Agente  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Principio 3  
**RF**: RF-02.1, RF-02.5

> **Como** Sistema Agente,  
> **debo** rechazar todo PDF que no pueda ser sanitizado correctamente y nunca enviarlo al LLM,  
> **para** garantizar que la arquitectura Zero Trust no sea vulnerada por ningún documento.

#### Criterios de Aceptación

```gherkin
Escenario 1: PDF con elemento no sanitizable
  Given el pipeline encuentra un PDF con un elemento que no puede neutralizar definitivamente
  When intenta la sanitización
  Then marca el PDF como "No enviable al LLM — sanitización incompleta"
  And no envía el PDF al LLM bajo ninguna circunstancia
  And notifica al Analista que ese PDF requiere revisión manual
  And registra el evento en el log de seguridad

Escenario 2 (Métrica): ASR < 2% sobre corpus adversarial
  Given el corpus D3 de PDFs adversariales
  When el sistema procesa todos los PDFs del corpus
  Then la tasa de ataque exitoso (ASR) es < 2%
  And 0 PDFs marcados como no sanitizables llegan al LLM
```

---

### US-22 — Ver Logs de Procesamiento y Alertas (Admin)
**Persona**: P-02 Carlos (Director / Admin)  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: Módulo 6  
**RF**: RF-06.1, RF-06.2

> **Como** Admin (Director),  
> **quiero** ver los logs de procesamiento y las alertas de error del sistema,  
> **para** supervisar la operación y diagnosticar problemas sin tener que consultar al equipo técnico.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Panel de logs disponible
  Given el Admin accede al panel de Administración
  When ve la sección de Logs
  Then ve logs estructurados de cada operación: timestamp, ID de proceso, cotización, resultado (éxito/error), duración
  And puede filtrar por: rango de fechas, proveedor, tipo de evento (ingesta, análisis, envío, IMAP)

Escenario 2: Alertas de error visibles
  Given el sistema tuvo un error de procesamiento (fallo SMTP, PDF no encontrado, timeout de LLM)
  Then la alerta aparece en el panel de Admin con: tipo de error, contexto (proveedor, portafolio), timestamp, y acción recomendada
  And la alerta persiste hasta que el Admin la marque como "Revisada"
```

---

### US-23 — Autenticación con Cuenta Institucional (OAuth2)
**Persona**: P-01 María, P-02 Carlos, P-03 Alejandro, P-04 Comité  
**Prioridad**: 🔴 Must Have  
**Módulo PRD**: §9 Roles  
**RF**: RF-07.2, RNF-02 (SECURITY-12)

> **Como** usuario de Assistent Buy (Analista, Admin, CISO, o Comité),  
> **quiero** autenticarme con mi cuenta Google o Microsoft institucional,  
> **para** acceder al sistema sin crear una contraseña separada y con el mismo nivel de seguridad que mi cuenta corporativa.

#### Criterios de Aceptación

```gherkin
Escenario 1 (Happy Path): Login exitoso con Google / Microsoft
  Given el usuario abre Assistent Buy desde el link en SAB
  When hace clic en "Iniciar sesión con Google" o "Iniciar sesión con Microsoft"
  Then es redirigido al flujo OAuth2/OIDC del proveedor
  And después de autenticarse, es redirigido de vuelta a Assistent Buy con su sesión activa
  And el sistema asigna automáticamente su rol (Analista/Admin/CISO/Comité) según la configuración

Escenario 2 (Seguridad): Sesión invalidada al cerrar sesión
  Given el usuario tiene una sesión activa
  When presiona "Cerrar sesión"
  Then la sesión es invalidada en el servidor (no solo en el browser)
  And si el usuario intenta usar el token de sesión anterior, el sistema lo rechaza (SECURITY-12)

Escenario 3 (Seguridad): MFA para cuentas Admin y CISO
  Given un usuario con rol Admin o CISO intenta autenticarse
  Then el flujo OAuth2 requiere MFA (si el proveedor del usuario lo tiene habilitado)
  And Assistent Buy documenta que MFA es obligatorio para estos roles en la política de seguridad (SECURITY-12)
```

---

## Épica E-06 — Supervisión Avanzada (v1.1 — Should Have)

> Funcionalidades diferidas al ciclo S2/S3 del PRD. Documentadas como Should Have para v1.1.

---

### US-24 — Dashboard de Métricas de Desempeño (Admin)
**Persona**: P-02 Carlos (Director / Admin)  
**Prioridad**: 🟡 Should Have (S3 — post-piloto)  
**Módulo PRD**: S3

> **Como** Admin (Director),  
> **quiero** ver un dashboard con métricas de cobertura, precisión y ciclo de evaluación,  
> **para** supervisar el desempeño del sistema después del piloto y antes del despliegue a todas las universidades.

#### Criterios de Aceptación

```gherkin
Escenario 1: Dashboard con métricas clave
  Given datos de análisis acumulados del piloto
  When el Admin accede al dashboard
  Then ve: % de cobertura (cotizaciones procesadas vs. total), precisión de extracción estimada, tiempo promedio de ciclo por portafolio, número de escalamientos por baja confianza
```

---

### US-25 — Alertas de Concentración de Adjudicación (Admin)
**Persona**: P-02 Carlos (Director / Admin)  
**Prioridad**: 🟡 Should Have (S2/S5 — post-datos de S1)  
**Módulo PRD**: S2, S5

> **Como** Admin (Director),  
> **quiero** recibir alertas cuando la concentración de adjudicaciones históricas supera un umbral definido por la institución,  
> **para** detectar potenciales patrones de concentración en proveedores antes de que se conviertan en un riesgo institucional.

#### Criterios de Aceptación

```gherkin
Escenario 1: Alerta por umbral de concentración
  Given datos históricos de adjudicación del ciclo S1
  And un umbral configurado (ej: un proveedor recibe > 40% del valor total del proceso)
  When el sistema detecta que el umbral se supera
  Then notifica al Admin con: proveedor, porcentaje de concentración actual, umbral configurado
  And sugiere revisar la diversificación del portafolio
```

---

## Resumen de Cobertura INVEST

| Story | Independiente | Negociable | Valiosa | Estimable | Pequeña | Testeable |
|---|---|---|---|---|---|---|
| US-01 a US-23 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| US-24, US-25 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## Mapa de Trazabilidad Stories → RF

| Story | RF cubiertos |
|---|---|
| US-01 | RF-01.1, RF-01.2, RF-01.3 |
| US-02 | RF-01.4, RF-01.5 |
| US-03 | RF-02.1, RF-02.2, RF-02.3, RF-02.4, RF-02.5, RF-02.6 |
| US-04 | RF-03.1, RF-03.6, RF-03.7 |
| US-05 | RF-03.2, RF-03.5 |
| US-06 | RF-03.3 |
| US-07 | RF-03.4 |
| US-08 | RF-01.4, RF-01.5 |
| US-09 | RF-04.1, RF-04.2 |
| US-10 | RF-04.2, RF-04.4, RF-04.5 |
| US-11 | RF-04.3 |
| US-12 | RF-04.7, RF-01.6 |
| US-13 | RF-01.6, RF-01.7, RF-01.8, RF-04.8 |
| US-14 | RF-04.5 |
| US-15 | RF-05.1, RF-05.4, RF-05.5 |
| US-16 | RF-05.2, RF-05.4 |
| US-17 | RF-07.3 |
| US-18 | RF-05.3 |
| US-19 | RF-06.3 |
| US-20 | RF-02.4 |
| US-21 | RF-02.1, RF-02.5 |
| US-22 | RF-06.1, RF-06.2 |
| US-23 | RF-07.2 |
