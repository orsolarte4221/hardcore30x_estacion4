# Personas — Assistent Buy AI Agent

> **Versión**: 1.0 (MVP)  
> **Metodología**: Basado en el PRD §3 y las decisiones del Story Generation Plan  
> **Idioma**: Español  

---

## Resumen de Personas

| ID | Persona | Tipo | Rol en el sistema |
|---|---|---|---|
| P-01 | María — Analista de Compras | Usuario operativo primario | Inicia análisis, revisa síntesis, gestiona aclaraciones |
| P-02 | Carlos — Director / Admin | Supervisor y administrador | Supervisa métricas, configura alertas, revisa logs |
| P-03 | Alejandro — CISO / Oficial de Seguridad | Usuario de control | Audita seguridad, revisa sanitización, gestiona incidentes |
| P-04 | Comité de Contratación | Consumidor de reportes | Lee reportes comparativos, toma decisiones de adjudicación |
| P-05 | Sistema Agente | Actor autónomo interno | Ejecuta análisis, sanitiza, extrae, cruza, escala, monitoriza IMAP |

---

## P-01 — María: Analista de Compras

### Perfil
| Atributo | Detalle |
|---|---|
| **Rol** | Analista de Adquisiciones / Jefa de Suministros |
| **Organización** | Universidad colombiana mediana (3.000–15.000 estudiantes) |
| **Experiencia tecnológica** | Media — usa SAB a diario, maneja Excel avanzado, no es técnica |
| **Portafolio típico** | 15-50 cotizaciones por proceso, 50-200 PDFs adjuntos |
| **Ciclo actual** | 15 días de revisión manual para cubrir un subconjunto de cotizaciones |

### Dolores Centrales
- **Fatiga cognitiva**: Lee cientos de páginas de contratos en formatos heterogéneos
- **Miedo a la "letra chica"**: Temor a firmar autorizaciones con cláusulas ocultas perjudiciales
- **Cobertura parcial**: Solo puede revisar 5-6 cotizaciones a profundidad; las demás las descarta sin leer
- **Presión de tiempo**: Los comités tienen fechas fijas; el tiempo de análisis es siempre insuficiente

### Motivaciones
- Cobertura total sin riesgo personal — que ninguna cotización quede sin revisar
- Comprimir el ciclo de 15 días a < 3 días hábiles
- Tener evidencia documentada de cada decisión (protección ante auditorías)

### Comportamiento con Assistent Buy
- Accede desde SAB vía link, se autentica con su cuenta Google institucional
- Inicia el análisis de un proceso completo con un clic
- Monitorea el progreso en tiempo real
- Revisa las síntesis y discrepancias detectadas
- Aprueba o descarta borradores de aclaración antes del envío
- Genera el reporte para el Comité

### Citas del PRD
> *"Fatiga cognitiva por leer miles de páginas en anexos y temor a firmar autorizaciones letales por detalles ocultos en la 'letra chica'."*

---

## P-02 — Carlos: Director de Finanzas / Cadena de Suministro (Admin)

### Perfil
| Atributo | Detalle |
|---|---|
| **Rol** | Director de Finanzas, Director de Cadena de Suministro, o Jefe de Compras |
| **Organización** | Mismo campus que María — su supervisor directo |
| **Poder de decisión** | Puede vetar la adopción si percibe riesgo de datos falsos |
| **Preocupación central** | Que la IA alucine y contamine reportes financieros |

### Dolores Centrales
- **Riesgo reputacional**: Una adjudicación basada en datos falsos de la IA lo expone ante Rectoría
- **Opacidad del proceso**: No tiene visibilidad de qué cotizaciones analizó el equipo y cuáles quedaron sin revisar
- **Concentración de proveedores**: Sospecha de patrones de adjudicación pero no tiene datos para sustentarlo

### Motivaciones
- Demostrar ante Rectoría que el proceso es transparente y auditable
- Detectar concentraciones de adjudicación antes de que se conviertan en un problema legal
- Reducir el tiempo de ciclo sin sacrificar calidad de análisis

### Comportamiento con Assistent Buy
- Accede como Admin — tiene todo lo que tiene María más funciones de supervisión
- Revisa logs de procesamiento y alertas de error
- Ve alertas del sistema de administración
- En v1.1: revisa dashboard de métricas y configura reglas de negocio

### Citas del PRD
> *"Miedo agudo a la 'alucinación sintética' de la IA que contamine reportes financieros."*  
> *"Puede vetar la adopción si percibe riesgo de datos falsos."*

---

## P-03 — Alejandro: CISO / Oficial de Seguridad

### Perfil
| Atributo | Detalle |
|---|---|
| **Rol** | CISO, Director de Tecnología, o Compliance Officer |
| **Poder** | Puede bloquear la integración completamente |
| **Conocimiento técnico** | Alto — entiende vectores de ataque, OWASP, políticas de datos |
| **Preocupación central** | IPI a través de PDFs maliciosos y exfiltración de datos internos |

### Dolores Centrales
- **IPI (Inyección Indirecta de Prompts)**: Un proveedor malintencionado podría insertar instrucciones en su PDF para manipular el análisis del agente
- **EchoLeak**: Que el agente filtre datos internos (de otras cotizaciones o de SAB) hacia el proveedor
- **Superficie de ataque desconocida**: No tiene visibilidad de qué hacen los LLMs externos con los datos de la institución

### Motivaciones
- Pruebas demostrables de aislamiento hermético, sanitización matemática y Zero Trust
- Logs de auditoría de cada documento procesado
- Capacidad de bloquear el sistema si se detecta un incidente de seguridad

### Comportamiento con Assistent Buy
- Revisa el log de sanitización de cada portafolio analizado
- Recibe alertas cuando el pipeline detecta elementos adversariales
- Audita el cumplimiento de las reglas de Security Baseline
- Puede revocar accesos desde el panel de Admin

### Citas del PRD
> *"Riesgo inminente de exposición de datos (EchoLeak) e Inyección Indirecta de Prompts (IPI) vehiculizada a través de PDFs maliciosos."*  
> *"Puede bloquear la integración."*

---

## P-04 — Comité de Contratación

### Perfil
| Atributo | Detalle |
|---|---|
| **Composición** | 3-7 miembros: Vicerrector Administrativo, Jefe Jurídico, Director Financiero, Jefe de área solicitante |
| **Rol en el sistema** | Solo lectura — consumen el reporte comparativo final |
| **Frecuencia de uso** | Reunión de comité una vez por proceso de adquisición |
| **Conocimiento técnico** | Bajo — no interactúan con el sistema directamente durante el análisis |

### Dolores Centrales
- **Información incompleta**: Históricamente reciben análisis de 5-6 cotizaciones de las 20 recibidas
- **Tiempo de reunión**: Deben tomar decisiones en una reunión con información preparada el mismo día
- **Responsabilidad compartida**: Si la adjudicación falla, todos los firmantes son responsables

### Motivaciones
- Recibir un reporte que cubra el 100% de las cotizaciones con trazabilidad completa
- Decidir con evidencia documentada, no con intuición del analista
- Protección ante auditorías: que cada decisión tenga una fuente verificable

### Comportamiento con Assistent Buy
- Recibe el link del reporte generado por el Analista (solo lectura)
- Lee el reporte comparativo con ranking, riesgos y trazabilidad
- Puede exportar o imprimir el PDF del reporte para el acta de la reunión
- No puede iniciar análisis ni aprobar aclaraciones

### Citas del PRD
> *"Decisión colegiada con información completa y auditable."*

---

## P-05 — Sistema Agente (Actor Autónomo)

### Descripción
El Sistema Agente es el componente de IA de Assistent Buy que ejecuta el análisis de forma autónoma. Se incluye como persona para documentar su comportamiento en las user stories de sistema ("Como Sistema, debo...").

### Capacidades
- Conectarse a SAB (BD + filesystem) en modo read-only
- Sanitizar PDFs con el pipeline Zero Trust + Spotlighting
- Extraer variables de documentos via Document AI cloud
- Cruzar datos estructurados vs. no estructurados
- Clasificar discrepancias por severidad
- Generar indicadores de confianza con referencia a fuente
- Escalar al humano cuando la confianza cae bajo umbral
- Generar borradores de solicitudes de aclaración
- Enviar emails aprobados via SMTP
- Monitorizar el buzón IMAP cada 5 minutos
- Correlacionar respuestas con solicitudes originales
- Reingestar documentos de respuesta al pipeline

### Restricciones OBLIGATORIAS (Never Have)
- ❌ NUNCA enviar comunicaciones sin aprobación humana explícita
- ❌ NUNCA modificar registros en SAB
- ❌ NUNCA descartar cotizaciones automáticamente
- ❌ NUNCA inventar valores — si la confianza es baja, escala
- ❌ NUNCA usar términos "fraude", "colusión" o "culpable"
- ❌ NUNCA pasar contenido sin sanitizar al LLM

---

## Mapa de Personas × Stories

| Épica | P-01 Analista | P-02 Director/Admin | P-03 CISO | P-04 Comité | P-05 Sistema |
|---|---|---|---|---|---|
| **Activación** | ✅ Inicia análisis | | | | ✅ Sanitiza, extrae |
| **Análisis** | ✅ Revisa variables y discrepancias | | | | ✅ Extrae, cruza, clasifica |
| **Aclaración HITL** | ✅ Aprueba borradores | | | | ✅ Envía, monitoriza IMAP |
| **Reportes** | ✅ Genera reporte | | | ✅ Lee reporte | |
| **Seguridad** | | ✅ Ve logs | ✅ Audita sanitización | | ✅ Aplica Zero Trust |
