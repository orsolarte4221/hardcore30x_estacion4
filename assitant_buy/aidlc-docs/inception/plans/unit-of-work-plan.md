# Plan de Units of Work — Assistent Buy AI Agent

> **Instrucciones**: Responda cada pregunta llenando el campo `[Answer]:` con la letra de su opción o respuesta personalizada.

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Contexto cargado (requirements + stories + application-design)
- [x] Plan creado con preguntas
- [x] Respuestas recibidas y analizadas
- [x] Plan aprobado por usuario

### PART 2 — Generation
- [x] `unit-of-work.md` generado
- [x] `unit-of-work-dependency.md` generado
- [x] `unit-of-work-story-map.md` generado
- [x] Code organization strategy documentada
- [ ] Units aprobadas por usuario

---

## Contexto de Descomposición

El sistema tiene **16 componentes** organizados en **4 capas** con **25 user stories** en **6 épicas**. Para un proyecto Greenfield de este tamaño, se necesita decidir cómo agrupar los componentes en unidades de trabajo que faciliten el desarrollo incremental.

### Componentes a agrupar (del Application Design):

| Capa | Componentes |
|---|---|
| **Core Pipeline** | C-01 IngestaConector, C-02 SanitizadorZeroTrust, C-03 MotorExtraccion, C-04 MotorCruce, C-09 ServicioPipeline |
| **HITL** | C-05 GestorAclaraciones |
| **Reportes** | C-06 GeneradorReportes |
| **Administración** | C-07 ModuloAdmin |
| **Seguridad** | C-08 AutenticadorOAuth |
| **API + Notificación** | C-10 APIRest, C-14 GestorWebSocket |
| **Adaptadores** | C-11 GatewayLLM, C-12 RepositorioSAB, C-16 GatewayDocumentAI |
| **Persistencia** | C-13 RepositorioLocal |
| **Frontend** | C-15 SPAAngular |

### Propuesta de Unidades de Trabajo (candidata):

Dada la decisión de comunicación **in-process** (todo corre en un proceso FastAPI), este es un **monolito modular**, no microservicios. Las unidades de trabajo son **módulos lógicos dentro de un monolito**, cada uno desarrollable de forma incremental.

| Unit | Nombre | Componentes | Stories |
|---|---|---|---|
| **U-01** | Fundación (Auth + Datos + API) | C-08, C-10, C-13, C-14 | US-23 |
| **U-02** | Ingesta y Sanitización | C-01, C-02, C-12, C-16 | US-01, US-02, US-03 |
| **U-03** | Extracción y Análisis | C-03, C-04, C-09, C-11 | US-04, US-05, US-06, US-07, US-08 |
| **U-04** | Comunicación HITL | C-05 | US-09, US-10, US-11, US-12, US-13, US-14 |
| **U-05** | Reportes y Decisión | C-06 | US-15, US-16, US-17, US-18 |
| **U-06** | Administración y Seguridad | C-07 | US-19, US-20, US-21, US-22 |
| **U-07** | Frontend SPA | C-15 | Todas las US (capa UI) |

---

## Sección 1 — Story Grouping

### Pregunta 1.1 — Validación de la propuesta de unidades
Revise la propuesta de 7 unidades de trabajo arriba. ¿Está de acuerdo con la agrupación o prefiere ajustarla?

A) Aprobada como está — las 7 unidades reflejan las capas funcionales del sistema  
B) Fusionar U-05 y U-06 — Reportes y Admin son ambos "salidas del sistema" y pueden ser un solo módulo  
C) Separar U-02 en dos — Ingesta (C-01, C-12) y Sanitización (C-02, C-16) deben ser independientes para desarrollo separado  
D) Separar U-03 en dos — Extracción (C-03, C-11, C-16) y Cruce (C-04) son equipos/preocupaciones distintas  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.2 — Frontend como unidad separada o integrado
El frontend Angular (U-07) es un proyecto completamente separado del backend Python. ¿Cómo se desarrolla?

A) Unidad separada (U-07) — el frontend se desarrolla como un proyecto Angular independiente con su propia estructura, y consume la API REST ya deployada  
B) Integrado por unidad — cada unidad backend (U-01 a U-06) incluye sus vistas Angular correspondientes, se desarrolla full-stack por funcionalidad  
C) Frontend paralelo — el frontend se desarrolla en paralelo con un API mock/stub, y se integra después de que el backend esté completo  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Dependencias y Orden de Desarrollo

### Pregunta 2.1 — Secuencia de desarrollo
Las unidades tienen dependencias naturales (U-01 Fundación debe ir primero, U-03 necesita U-02, etc.). ¿Se desarrollan estrictamente en secuencia o en paralelo donde sea posible?

A) Secuencia estricta — U-01 → U-02 → U-03 → U-04 → U-05 → U-06 → U-07 (menor riesgo, más lento)  
B) Paralelo controlado — U-01 primero (fundación), luego U-02 y U-06 en paralelo, luego U-03, U-04, U-05, finalmente U-07  
C) Máximo paralelismo — U-01 primero, todo lo demás en paralelo usando interfaces/mocks para las dependencias  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 3 — Consideraciones Técnicas

### Pregunta 3.1 — Organización del código (Greenfield)
¿Cómo se organiza el código fuente del monolito FastAPI?

A) Por capa técnica — `app/models/`, `app/services/`, `app/api/`, `app/repositories/`  
B) Por módulo de negocio — `app/ingesta/`, `app/extraccion/`, `app/aclaraciones/`, `app/reportes/`, `app/admin/`  
C) Híbrido — módulos de negocio para los componentes core, capas técnicas para los transversales: `app/core/` (auth, config, logging), `app/modules/ingesta/`, `app/modules/extraccion/`, etc.  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 3.2 — Modelo de dominio compartido
Los modelos de datos (Cotizacion, VariableExtraida, Discrepancia, etc.) son usados por múltiples unidades. ¿Dónde se definen?

A) En un paquete compartido `app/shared/models/` — todos los modelos en un solo lugar  
B) Cada módulo define sus propios modelos internos + hay DTOs compartidos en `app/shared/schemas/` para la comunicación entre módulos  
C) Todos los modelos en `app/models/` (single directory) — aprovechando que es un monolito y no hay riesgo de acoplamiento entre servicios  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 3.3 — Configuración y secrets
¿Cómo se gestionan las variables de entorno y secrets (API keys de Claude, credenciales de BD SAB, credenciales SMTP/IMAP)?

A) Archivo `.env` con pydantic-settings — las variables se cargan desde `.env` y se validan con Pydantic `BaseSettings`  
B) Archivo `config.py` centralizado — una clase de configuración que lee de variables de entorno o secrets manager según el ambiente  
C) Secrets manager cloud (ej: AWS Secrets Manager, GCP Secret Manager) desde el inicio — sin archivos `.env` en ningún ambiente  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 4 — Dominio de Negocio

### Pregunta 4.1 — Bounded contexts
¿Las unidades de trabajo deben tratarse como bounded contexts (DDD) con modelos de dominio propios, o comparten un modelo de dominio unificado?

A) Bounded contexts — cada unidad tiene su propio modelo de dominio y se comunican via DTOs (más desacoplado, pero más complejidad de mapeo)  
B) Modelo de dominio unificado — todas las unidades comparten los mismos modelos (más simple para un monolito, adecuado para MVP)  
C) Modelo de dominio unificado con eventos de dominio — modelos compartidos pero las unidades se notifican entre sí via eventos internos (prepare for future decomposition)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 3 artefactos de Units of Work.
