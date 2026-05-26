# Plan de NFR Requirements — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — NFR Requirements  
> **Fecha**: 2026-05-26  
> **Componentes**: C-08 AutenticadorOAuth, C-10 APIRest, C-13 RepositorioLocal, C-14 GestorWebSocket  

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta. Cuando termine todas, indíquelo y el workflow generará los 2 artefactos NFR de U-01.

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] Functional Design aprobado (domain-entities, business-rules, business-logic-model)
- [x] Plan NFR creado con preguntas
- [x] Respuestas recibidas y analizadas
- [x] Plan aprobado

### PART 2 — Generation
- [x] `nfr-requirements.md` generado
- [x] `tech-stack-decisions.md` generado

---

## Sección 1 — Escalabilidad y Carga

### Pregunta 1.1 — Volumen esperado en el piloto

¿Cuántos portafolios y cotizaciones simultáneas se esperan durante el piloto?

A) Escala micro: 1–5 portafolios activos simultáneamente, 10–50 cotizaciones por portafolio (carga baja, prioridad en velocidad de desarrollo)  
B) Escala pequeña: 10–20 portafolios activos, hasta 100 cotizaciones cada uno (carga moderada, necesita pool de conexiones BD)  
C) Escala media: 50+ portafolios, portafolios con 200+ cotizaciones (requiere queue de procesamiento y worker separado)  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.2 — Usuarios concurrentes

¿Cuántos usuarios conectados simultáneamente se espera en el piloto y en producción?

A) Piloto: 2–5 usuarios; Producción: 10–20 usuarios — no se necesita infraestructura de escalado  
B) Piloto: 5–15 usuarios; Producción: 50–100 usuarios — FastAPI con uvicorn workers es suficiente  
C) Piloto: 20+ usuarios; Producción: 200+ usuarios — requiere load balancer y múltiples instancias  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 2 — Performance

### Pregunta 2.1 — Tiempo de respuesta API REST

¿Cuál es el tiempo de respuesta máximo aceptable para los endpoints de la API?

A) < 500ms para el 95% de las requests (excluyendo operaciones de IA que son asíncronas)  
B) < 1 segundo para el 95% — más permisivo, adecuado para piloto  
C) < 200ms para el 95% — alta exigencia, requiere caché y optimización desde el inicio  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 2.2 — Performance de autenticación (OAuth2)

El flujo OAuth2 involucra redirecciones externas a Google. ¿Qué SLA se define para el proceso completo de login?

A) < 3 segundos el flujo completo (incluyendo round-trip a Google) — aceptable para login ocasional  
B) < 5 segundos — el usuario entiende que hay una redirección externa  
C) No se define SLA para login — es un flujo externo fuera de nuestro control  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 2.3 — Performance de WebSocket

¿Cuál es la latencia máxima aceptable entre que un evento ocurre (ej: PDF procesado) y que el cliente Angular lo recibe?

A) < 500ms — el usuario percibe el progreso en tiempo casi real  
B) < 2 segundos — aceptable para un piloto, el contexto de uso no es crítico en tiempo  
C) No se define SLA — el WebSocket es "best-effort" en MVP  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 3 — Disponibilidad y Confiabilidad

### Pregunta 3.1 — Uptime requerido

¿Cuál es el uptime mínimo requerido durante el piloto?

A) 95% mensual (~36 horas de downtime/mes aceptables) — ambiente de desarrollo, mantenimiento flexible  
B) 99% mensual (~7.2 horas de downtime/mes) — piloto con usuarios reales, ventanas de mantenimiento planificadas  
C) 99.9% (~43 minutos/mes) — producción crítica, requiere HA y failover automático  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 3.2 — Estrategia de backup de PostgreSQL

¿Con qué frecuencia se respalda la base de datos local?

A) Backup diario automático — restore posible en < 24h de datos  
B) Backup cada 6 horas — máxima pérdida de datos: 6 horas  
C) Sin backup automatizado en piloto — la BD se puede reconstruir del sistema SAB  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 3.3 — Manejo de fallos del pipeline IA

Si el pipeline de IA (Claude/Document AI) falla durante el análisis, ¿qué comportamiento se espera del sistema?

A) Retry automático 3 veces con backoff exponencial, luego marcar cotización con estado `ERROR` y notificar vía WebSocket  
B) Fallo inmediato — la cotización queda en `ERROR`, el analista puede reintentar manualmente desde la UI  
C) Circuit breaker — si > 3 fallos consecutivos en 5 minutos, se pausa todo el pipeline y se alerta al Admin  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 4 — Seguridad

### Pregunta 4.1 — Cifrado de datos en reposo

La BD PostgreSQL almacena datos sensibles de cotizaciones. ¿Se requiere cifrado a nivel de columna o disco?

A) Cifrado a nivel de disco (filesystem encryption del servidor) — suficiente para el piloto  
B) Cifrado a nivel de columna para campos sensibles (ej: `email_proveedor`, datos financieros) usando `pgcrypto`  
C) Sin cifrado adicional en piloto — los datos no son clasificados como PII crítica en esta etapa  
X) Otro (describa después del tag [Answer]:)

[Answer]: C

---

### Pregunta 4.2 — Logs y datos sensibles

¿Se pueden almacenar datos de negocio (nombres de proveedores, valores de cotizaciones) en los logs del sistema?

A) Sí, datos completos en logs — facilita el debugging en piloto  
B) No — los logs solo contienen IDs (UUIDs) y metadatos, nunca valores de negocio  
C) Logs con datos enmascarados — los valores se truncan o se hashean en los logs  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 4.3 — Política de retención del AuditTrail

El `AuditTrail` es append-only y crecerá indefinidamente. ¿Cuánto tiempo se retienen los registros?

A) Retención ilimitada — los registros nunca se archivan ni eliminan (más simple para MVP)  
B) Retención 1 año activo + archivo en tabla histórica — después de 1 año se mueven a tabla `audit_trail_archive`  
C) Retención 90 días — período suficiente para auditorías del piloto  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

## Sección 5 — Tech Stack y Entorno

### Pregunta 5.1 — Entorno de despliegue del piloto

¿Dónde se desplegará el sistema durante el piloto?

A) Servidor local / on-premise de la universidad — sin nube, acceso en red interna  
B) Cloud managed — AWS/GCP/Azure con instancias básicas (VM o container)  
C) Docker Compose en un servidor dedicado — portable, sin necesitar orquestación Kubernetes  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 5.2 — Versión de Python y gestor de dependencias

¿Qué versión de Python y gestor de dependencias se usa en el backend?

A) Python 3.12 + `uv` (moderno, ultra-rápido) — el estándar emergente en 2025-2026  
B) Python 3.12 + `pip` + `requirements.txt` — el más simple y ampliamente conocido  
C) Python 3.12 + `poetry` — gestión de dependencias con lock file y entornos virtuales integrado  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 5.3 — ORM y migraciones de base de datos

Para la capa de persistencia con PostgreSQL, ¿qué ORM y herramienta de migración se usa?

A) SQLAlchemy 2.x (async) + Alembic — el stack estándar de FastAPI con soporte async completo  
B) SQLAlchemy 2.x (sync) + Alembic — más simple, sin complejidad de async en la capa de BD  
C) Tortoise ORM + Aerich — nativo async, diseñado para FastAPI, pero menos maduro  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 5.4 — Servidor ASGI y configuración de workers

¿Cómo se ejecuta FastAPI en el servidor?

A) Uvicorn con múltiples workers (`--workers 4`) — aprovecha múltiples cores para requests concurrentes  
B) Uvicorn single worker + Gunicorn como process manager — más robusto para producción  
C) Uvicorn single worker — suficiente para piloto con pocos usuarios concurrentes  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 6 — Observabilidad y Mantenibilidad

### Pregunta 6.1 — Estrategia de logging

¿Qué nivel y formato de logging se usa?

A) JSON estructurado con correlation ID en todos los logs — facilita búsquedas y integración con herramientas (ELK, CloudWatch)  
B) Logs de texto plano con nivel INFO/ERROR — más legible para desarrollo, suficiente para piloto  
C) JSON estructurado solo para errores; texto plano para INFO/DEBUG — balance entre observabilidad y simplicidad  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 6.2 — Healthcheck y monitoreo

¿Qué nivel de monitoreo se necesita en el piloto?

A) Solo endpoint `/health` con estado de BD — suficiente para detectar caídas del servicio  
B) `/health` + métricas básicas: latencia, requests activos, conexiones BD — para dashboard simple  
C) Observabilidad completa: `/health`, `/metrics` (Prometheus), alertas automáticas — stack de monitoreo dedicado  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 6.3 — Cobertura mínima de tests

¿Qué porcentaje de cobertura de tests se requiere para U-01 antes de considerar la unidad "lista para build"?

A) 70% cobertura mínima — cubre los caminos principales sin sobreingeniería de tests  
B) 80% cobertura — estándar de la industria para sistemas con flujos críticos de seguridad  
C) 90%+ cobertura — alta exigencia, apropiada dado que Auth y BD son la base de todo el sistema  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 2 artefactos de NFR Requirements de U-01.
