# Preguntas de Verificación de Requerimientos
## Proyecto: Assistent Buy AI Agent

> **Instrucciones**: Por favor responda cada pregunta llenando el campo `[Answer]:` con la letra de su opción (A, B, C, etc.) o con su respuesta personalizada si selecciona la opción X/E.

---

## Análisis de Intención (Contexto del PRD)

El PRD ha sido leído completamente. Se identificó:

- **Tipo de solicitud**: Nuevo Producto (Greenfield)
- **Alcance**: Sistema completo (múltiples módulos y componentes)
- **Complejidad**: Alta — Agente de IA con análisis de PDFs, integración SAB, pipeline de seguridad Zero Trust, HITL, y reportes
- **Profundidad de requerimientos**: Comprehensive

---

## Sección 1 — Contexto Técnico de SAB

### Pregunta 1.1 — Integración con SAB
¿Cómo se conectará Assistent Buy a SAB en el MVP?

A) SAB es un sistema propio con API REST interna disponible — se usará esa API  
B) SAB expone sus datos mediante acceso directo a base de datos (SQL/NoSQL)  
C) El acceso a SAB en el MVP se hará con archivos de exportación estáticos (CSV + PDFs) hasta que la API esté disponible  
D) SAB tiene un SDK o librería cliente que se puede importar directamente  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.2 — Almacenamiento de PDFs en SAB
¿Cómo están almacenados actualmente los PDFs (documentos adjuntos de cotizaciones) en SAB?

A) En un sistema de archivos local / servidor de archivos accesible por ruta  
B) En un bucket de almacenamiento cloud (S3, GCS, Azure Blob)  
C) En la base de datos de SAB (BLOB / campo binario)  
D) En un sistema de gestión documental externo (SharePoint, Google Drive, etc.)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 1.3 — Volumen esperado por portafolio
El PRD menciona el ejemplo de 18 cotizaciones con 64 documentos. ¿Cuál es el rango máximo esperado para el MVP?

A) Hasta 50 cotizaciones / hasta 200 PDFs por portafolio  
B) Hasta 100 cotizaciones / hasta 500 PDFs por portafolio  
C) Sin límite definido — el sistema debe escalar dinámicamente  
D) El ejemplo del PRD (18 cotizaciones / 64 PDFs) representa el caso promedio — no se esperan volúmenes mucho mayores en el MVP  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Stack Tecnológico

### Pregunta 2.1 — Lenguaje / Framework del backend
¿Cuál es la preferencia para el backend del agente?

A) Python (FastAPI / Django) — natural para IA/ML, amplio ecosistema LLM  
B) Node.js (Express / NestJS) — si SAB ya usa JavaScript  
C) Java / Spring Boot — si el equipo o SAB ya lo usa  
D) Sin preferencia — el agente decide según mejor opción técnica  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Proveedor de LLM
¿Cuál es la preferencia para el modelo de lenguaje grande (LLM) principal?

A) OpenAI (GPT-4o / GPT-4.1) — madurez y capacidad de contexto largo  
B) Anthropic Claude — capacidad de contexto muy largo, buen razonamiento  
C) Google Gemini — integración con ecosistema Google  
D) Modelo local / self-hosted (Ollama, LM Studio) — por privacidad de datos  
E) Evaluar ≥2 proveedores en paralelo (como lo indica el PRD en dependencias críticas)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 2.3 — Extracción de texto de PDFs
¿Cuál es el enfoque preferido para extraer texto de PDFs (incluyendo PDFs escaneados)?

A) Librería open-source (PyMuPDF, pdfplumber, pdfminer) — para PDFs nativos digitales  
B) OCR nativo + PDF parsing (Tesseract + PyMuPDF) — para PDFs escaneados  
C) Servicio cloud de Document AI (Google Document AI, AWS Textract, Azure Form Recognizer)  
D) Combinación: OCR local para contenido escaneado + parsing para digitales  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 2.4 — Infraestructura de despliegue
¿Dónde se desplegará el MVP?

A) Cloud pública (AWS / GCP / Azure) — mayor escalabilidad  
B) On-premise / servidor propio — por restricciones de datos o presupuesto  
C) Plataforma del cliente (universities run their own infra)  
D) Contenedores Docker (sin preferencia de cloud específico)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 3 — Interfaz de Usuario

### Pregunta 3.1 — Tipo de interfaz
¿Cómo se accederá a Assistent Buy?

A) Módulo web integrado dentro del UI de SAB (iframe o componente embebido)  
B) Aplicación web independiente (SPA o multi-page) que se abre desde SAB via link  
C) API pura (sin UI propia en el MVP) — SAB consumiría la API  
D) Combinación: API + interfaz web básica de administración  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 3.2 — Tecnología de Frontend (si aplica)
Si se construye una interfaz web, ¿cuál es la preferencia tecnológica?

A) React + TypeScript — ecosistema amplio  
B) Vue.js — más simple para equipos pequeños  
C) HTML/JS vanilla — mínima complejidad, máxima portabilidad  
D) Angular — si el equipo o SAB ya lo usa  
E) No aplica — MVP es solo API (ver respuesta 3.1)  
X) Otro (describa después del tag [Answer]:)

[Answer]: D

---

## Sección 4 — Alcance del MVP

### Pregunta 4.1 — Módulo 6 (Administración y Monitoring) en MVP
El PRD define este módulo como "básico" en v1 (logs y alertas). ¿Qué debe incluir mínimamente?

A) Solo logs de procesamiento y alertas de error — nada más en MVP  
B) Logs + vista básica de estado de análisis por proceso  
C) Logs + vista de estado + configuración básica de conexión a SAB  
D) El dashboard completo es necesario desde v1 (contrario al PRD que lo difiere a S3)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.2 — Exportación de reportes
El PRD menciona "Export PDF" para los reportes comparativos (Módulo 5). ¿Es Must Have para MVP?

A) Sí — el reporte debe exportarse a PDF directamente desde la interfaz  
B) Sí — pero también a Word/DOCX para que el Comité pueda editarlo  
C) No — para el MVP basta con visualización en pantalla; el export se difiere a v1.1  
D) El analista puede imprimir desde el navegador (print-to-PDF del browser) — no se necesita export nativo  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 4.3 — Comunicación real con proveedores (HITL Módulo 4)
El PRD dice que el agente genera borradores y el analista los aprueba. ¿Cómo se envían realmente al proveedor en el MVP?

A) Por email (el sistema integra con un servidor SMTP para enviar directamente)  
B) El sistema genera el borrador aprobado y el analista lo copia-pega o envía manualmente desde su email  
C) Integración con SAB para que el envío se haga desde los canales de comunicación de SAB  
D) No hay envío real en MVP — solo generación y aprobación de borradores (comunicación fuera del sistema)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 5 — Seguridad y Datos

### Pregunta 5.1 — Autenticación de usuarios
¿Cómo se autenticarán los usuarios en Assistent Buy?

A) SSO integrado con SAB (Single Sign-On — usuarios de SAB acceden directamente)  
B) OAuth2 / OIDC con proveedor externo (Google, Microsoft Azure AD)  
C) Autenticación propia (JWT con usuario/contraseña)  
D) Sin autenticación en MVP (protegido por red interna)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 5.2 — Privacidad y residencia de datos
Los documentos de cotización pueden contener información confidencial. ¿Hay restricciones sobre dónde se procesan los datos?

A) Los datos NUNCA pueden salir de servidores colombianos / on-premise  
B) Datos pueden procesarse en cloud internacional si están cifrados y el proveedor tiene acuerdo de procesamiento  
C) Sin restricciones especiales para el MVP — se valida en piloto  
D) Depende del cliente: algunas universidades tendrán restricciones, otras no  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

## Sección 6 — Extensiones AI-DLC

### Pregunta 6.1 — Extensión: Security Baseline
¿Deben aplicarse las reglas de la extensión **Security Baseline** como restricciones bloqueantes en este proyecto?

Dado que el PRD tiene un fuerte énfasis en Zero Trust, IPI, EchoLeak y seguridad documental, esta extensión es **altamente recomendada**.

A) Sí — aplicar TODAS las reglas de Security Baseline como restricciones bloqueantes *(recomendado para este proyecto dado el perfil de seguridad del PRD)*  
B) No — omitir las reglas de Security Baseline (adecuado solo para PoCs / prototipos)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 6.2 — Extensión: Property-Based Testing
¿Deben aplicarse las reglas de **Property-Based Testing** en este proyecto?

Dada la complejidad del motor de extracción y cruce de datos, esta extensión agrega valor para verificar invariantes del sistema.

A) Sí — aplicar TODAS las reglas de PBT como restricciones bloqueantes *(recomendado para el motor de análisis y cruce de datos)*  
B) Parcial — aplicar PBT solo para funciones puras y transformaciones de datos  
C) No — omitir PBT (adecuado para proyectos simples CRUD o capas de integración finas)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

> **Próximo paso**: Una vez que complete todas las respuestas en este archivo, el workflow procederá a generar el **documento de requerimientos completo** (`aidlc-docs/inception/requirements/requirements.md`) y continuará con el resto de las etapas de INCEPTION.
