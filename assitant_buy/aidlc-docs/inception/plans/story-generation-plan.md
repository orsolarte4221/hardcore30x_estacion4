# Plan de Generación de User Stories — Assistent Buy AI Agent

> **Instrucciones**: Responda cada pregunta llenando el campo `[Answer]:` con la letra de su opción o respuesta personalizada.

---

## Checkboxes de Ejecución del Plan

### PART 1 — Planning
- [x] Step 1: Assessment validado — User Stories justificadas
- [x] Step 2: Plan creado
- [x] Step 3: Preguntas generadas (ver secciones abajo)
- [x] Step 4: Respuestas recibidas y analizadas
- [x] Step 5: Plan aprobado por usuario

### PART 2 — Generation
- [x] Step 6: Personas generadas (`personas.md`)
- [x] Step 7: User Stories generadas (`stories.md`)
- [x] Step 8: Criterios de aceptación documentados
- [ ] Step 9: Stories aprobadas por usuario

---

## Contexto del Proyecto

El PRD define 4 buyer personas:
1. **Analista de Compras** — usuario operativo principal
2. **Director de Finanzas / Cadena de Suministro** — supervisor y admin
3. **CISO / Oficial de Seguridad** — puede vetar la adopción
4. **Comité de Contratación** — consumidor de reportes (solo lectura)

Y el sistema agente en sí mismo tiene comportamientos que necesitan stories de sistema.

---

## Sección 1 — Personas y Roles

### Pregunta 1.1 — Cobertura de personas
El PRD define 4 personas. ¿Debemos crear user stories para todas?

A) Sí — crear stories para las 4 personas (Analista, Director/Admin, CISO, Comité)  
B) Solo las operativas — Analista y Director/Admin (el CISO y el Comité son secundarios para el MVP)  
C) Solo el Analista — es el usuario operativo principal; los demás son stakeholders, no usuarios directos del MVP  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.2 — Persona del sistema agente
El agente en sí tiene comportamientos autónomos (sanitizar, extraer, cruzar, escalar). ¿Incluimos "system stories" para documentar el comportamiento del agente?

A) Sí — incluir stories de sistema del tipo "Como sistema, debo sanitizar cada PDF antes de enviarlo al LLM"  
B) No — las stories son solo para usuarios humanos; el comportamiento del agente se documenta en los requerimientos funcionales  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Formato y Granularidad

### Pregunta 2.1 — Formato de las user stories
¿Qué formato usar para las user stories?

A) Estándar: "Como [persona], quiero [acción], para [beneficio]" + criterios de aceptación en formato Gherkin (Given/When/Then)  
B) Estándar: "Como [persona], quiero [acción], para [beneficio]" + criterios de aceptación en lista de bullets  
C) Formato Job Story: "Cuando [situación], quiero [motivación], para [resultado esperado]"  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Nivel de granularidad
¿Qué nivel de detalle deben tener las stories?

A) Épicas + Stories: primero agrupar en épicas por módulo, luego desglosar en stories individuales  
B) Solo Stories: stories individuales directamente, sin nivel de épica  
C) Stories + Sub-tasks: cada story se desglosa en tareas técnicas específicas  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.3 — Tamaño de las stories
¿Qué tan granulares deben ser las stories para cumplir el criterio INVEST (Small)?

A) Una story por feature principal del PRD (M1-M12) — 12 stories totales aproximadamente  
B) Una story por flujo de usuario completo (ej: "Iniciar análisis de portafolio", "Aprobar borrador de aclaración") — 20-30 stories  
C) Stories muy granulares: una por interacción de UI o decisión del agente — 50+ stories  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 3 — Enfoque de Organización

### Pregunta 3.1 — Criterio de agrupación
¿Cómo deben organizarse las stories?

A) Por persona (Analista / Director / Comité / CISO) — cada sección contiene las stories de esa persona  
B) Por módulo del sistema (Módulo 1 Ingesta, Módulo 2 Sanitización, etc.) — alineado con la arquitectura del PRD  
C) Por journey de usuario (Pre-análisis → Análisis → Aclaración → Reporte) — alineado con el flujo de trabajo  
D) Por épica de negocio (Cobertura total de cotizaciones / Seguridad documental / Decisión informada / Comunicación HITL)  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

## Sección 4 — Criterios de Aceptación

### Pregunta 4.1 — Nivel de detalle en criterios de aceptación
¿Qué tan detallados deben ser los criterios de aceptación?

A) Criterios de alto nivel: 3-5 bullets por story que describan el comportamiento esperado  
B) Criterios detallados con escenarios: Happy Path + al menos 1 edge case por story  
C) Criterios completos con métricas: incluir los KPIs del PRD (>96% precisión, <60s latencia, etc.) donde aplique  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 4.2 — Mapeo a los journeys del PRD
El PRD define 4 journeys (Happy Path Analista, Happy Path Director, PDF corrupto, Ambigüedad no resoluble). ¿Los incorporamos directamente en los criterios de aceptación?

A) Sí — cada journey del PRD se convierte en el escenario de aceptación de una story o conjunto de stories  
B) No — los journeys son referencia, pero los criterios de aceptación deben ser independientes y más granulares  
C) Parcial — el Journey 1 (Happy Path) como base, los otros como edge cases en stories específicas  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 5 — Alcance MVP

### Pregunta 5.1 — Priorización MoSCoW en stories
¿Las stories deben incluir la priorización MoSCoW del PRD?

A) Sí — cada story etiquetada con Must Have / Should Have / Could Have según el PRD  
B) Solo Must Have — generar stories únicamente para los features M1-M12 del MVP  
C) Must Have + Should Have — incluir también S1-S5 aunque estén diferidos a v1.1  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 5.2 — Stories de seguridad
Dado que la Security Baseline está habilitada y el CISO es un buyer persona clave, ¿incluimos stories explícitas de seguridad?

A) Sí — stories explícitas del tipo "Como CISO, quiero ver el estado de sanitización de cada PDF analizado" y "Como sistema, debo rechazar todo PDF que no pase el pipeline Zero Trust"  
B) No — la seguridad es un cross-cutting concern; se documenta en los criterios de aceptación de otras stories, no como stories independientes  
C) Mixto — stories de seguridad para las interacciones visibles del CISO, y criterios de aceptación de seguridad en las stories operativas  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 6 — Idioma y Estilo

### Pregunta 6.1 — Idioma de las stories
¿En qué idioma se redactan las user stories?

A) Español — consistente con el PRD y el equipo  
B) Inglés — estándar de la industria para documentación técnica  
C) Bilingüe — título y descripción en español, criterios de aceptación en inglés  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará `personas.md` y `stories.md` con las user stories completas.
