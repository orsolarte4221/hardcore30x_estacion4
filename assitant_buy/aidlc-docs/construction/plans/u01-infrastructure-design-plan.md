# Plan de Infrastructure Design — U-01 Fundación

> **Unidad**: U-01 — Fundación (Auth + Datos + API + WebSocket)  
> **Fase**: CONSTRUCTION — Infrastructure Design  
> **Fecha**: 2026-05-26  
> **Componentes**: C-08 AutenticadorOAuth, C-10 APIRest, C-13 RepositorioLocal, C-14 GestorWebSocket  

> **Instrucciones**: Complete el campo `[Answer]:` de cada pregunta. Cuando termine todas, indíquelo y el workflow generará los 2 artefactos de Infrastructure Design de U-01.

---

## Checkboxes de Ejecución

### PART 1 — Planning
- [x] NFR Requirements aprobados (nfr-requirements + tech-stack-decisions)
- [x] Plan de infraestructura creado con preguntas
- [x] Respuestas recibidas y analizadas
- [ ] Plan aprobado (pendiente revisión)

### PART 2 — Generation
- [x] `infrastructure-design.md` generado
- [x] `deployment-architecture.md` generado
- [x] `shared-infrastructure.md` generado

---

## Sección 1 — Entorno de Despliegue

### Pregunta 1.1 — Proveedor Cloud

En NFR se definió cloud managed (AWS/GCP/Azure). ¿Cuál es el proveedor preferido para el piloto?

A) Google Cloud Platform (GCP) — coherente con el proveedor OAuth2 (Google Workspace); Cloud Run, Cloud SQL, Cloud Storage  
B) AWS — el más usado en el mercado; EC2, RDS, S3, ECS  
C) Azure — si la universidad ya tiene licencias Microsoft; App Service, Azure DB for PostgreSQL, Blob Storage  
X) Otro (describa después del tag [Answer]:)

[Answer]: B

---

### Pregunta 1.2 — Estrategia de ambientes

¿Cuántos ambientes se gestionan en el piloto?

A) 2 ambientes: `development` (local) + `production` — simple, los devs prueban en local  
B) 3 ambientes: `development` (local) + `staging` (cloud) + `production` (cloud) — estándar para validar antes de producción  
C) 1 ambiente: solo `production` en cloud — el piloto es pequeño y no justifica múltiples ambientes  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 2 — Cómputo

### Pregunta 2.1 — Servicio de cómputo para FastAPI

¿En qué servicio se ejecuta el contenedor FastAPI en producción?

A) Container as a Service (CaaS) — Google Cloud Run / AWS App Runner / Azure Container Apps: serverless, paga por uso, escala a 0  
B) VM con Docker — instancia EC2/Compute Engine con Docker instalado y Docker Compose: más control, costo fijo  
C) Kubernetes (GKE/EKS/AKS) — máxima flexibilidad, pero overhead de gestión no justificado para piloto  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 2.2 — Gestión del proceso en la VM (si aplica VM)

Si se usa VM con Docker (opción B de 2.1), ¿cómo se gestiona el reinicio automático del contenedor?

A) Docker Compose con `restart: always` — simple, el daemon Docker reinicia el contenedor si cae  
B) Systemd service — el SO gestiona el proceso y lo reinicia; más robusto que `restart: always`  
C) No aplica — se usa CaaS (el servicio cloud gestiona el ciclo de vida)  
X) Otro (describa después del tag [Answer]:)

[Answer]: 

---

## Sección 3 — Almacenamiento

### Pregunta 3.1 — Servicio de PostgreSQL en la nube

¿Qué servicio managed de PostgreSQL se usa?

A) Cloud SQL (GCP) / RDS PostgreSQL (AWS) / Azure Database for PostgreSQL — managed completo: backups automáticos, failover, parches  
B) PostgreSQL en Docker en la misma VM de la aplicación — más simple, sin costo adicional de managed DB, pero sin HA  
C) PlanetScale / Supabase / Neon — DBaaS con tier gratuito, bueno para piloto de bajo presupuesto  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 3.2 — Almacenamiento de archivos PDF

Los PDFs de cotizaciones deben almacenarse de forma persistente y accesible por el pipeline de IA. ¿Dónde?

A) Object Storage managed (Cloud Storage/S3/Blob Storage) — escalable, durabilidad 99.999999999%, acceso via URL firmada  
B) Volumen de disco persistente adjunto a la VM — más simple, los PDFs están en el filesystem del servidor  
C) Base de datos (BYTEA en PostgreSQL) — sin infraestructura adicional, pero no recomendado para archivos grandes  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 4 — Mensajería y Procesamiento Async

### Pregunta 4.1 — Cola de tareas para pipeline IA

El pipeline de IA (análisis de PDFs con Claude/Document AI) es un proceso largo que no puede bloquear el request HTTP. ¿Cómo se gestiona en U-01?

A) Async in-process con `asyncio` + `BackgroundTasks` de FastAPI — sin infraestructura de queue; el worker corre en el mismo proceso  
B) Cola de tareas con Celery + Redis — worker separado, más robusto pero requiere más infraestructura  
C) Cola de tareas nativa del cloud (Cloud Tasks/SQS/Azure Service Bus) — serverless, sin gestión de worker  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 5 — Networking

### Pregunta 5.1 — Dominio y certificado SSL

¿Cómo se gestiona el dominio y certificado SSL para la API?

A) Dominio propio con Let's Encrypt (certbot) — certificado gratuito, renovación automática cada 90 días  
B) Certificado gestionado por el proveedor cloud (Cloud Load Balancer / CloudFront / Azure Front Door) — sin gestión manual  
C) Sin dominio propio en piloto — se usa la URL autogenerada del servicio cloud (ej: `*.run.app`, `*.amazonaws.com`)  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 5.2 — API Gateway / Load Balancer

¿Se pone un API Gateway o Load Balancer frente a la aplicación FastAPI?

A) Sin API Gateway en piloto — la aplicación FastAPI es el único punto de entrada (detrás de nginx o el LB del cloud)  
B) API Gateway managed (Cloud Endpoints / AWS API Gateway / Azure APIM) — gestión de rate limiting, autenticación y caché a nivel de gateway  
C) Nginx como reverse proxy en la misma VM — gestiona SSL, compresión y sirve como proxy hacia FastAPI  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 6 — Monitoreo e Infraestructura de Observabilidad

### Pregunta 6.1 — Destino de logs en producción

Los logs de FastAPI van a stdout. ¿Dónde se recolectan y almacenan?

A) Servicio de logging del cloud (Cloud Logging/CloudWatch/Azure Monitor) — recolección automática de stdout del contenedor, sin configuración adicional  
B) Archivo de log en la VM + logrotate — simple, los logs van al disco local, rotación automática diaria  
C) Stack ELK/EFK externo (Elasticsearch + Kibana) — máxima capacidad de búsqueda, pero overhead de infraestructura  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 6.2 — Alertas de infraestructura

¿Qué alertas de infraestructura se configuran para el piloto?

A) Solo alerta de downtime — notificación si el healthcheck `/health` falla por > 3 intentos consecutivos  
B) Downtime + uso de CPU > 80% + uso de memoria > 85% — alertas básicas de recursos  
C) Sin alertas automatizadas en piloto — el equipo monitorea manualmente el dashboard del cloud  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

## Sección 7 — Infraestructura Compartida

### Pregunta 7.1 — ¿U-01 comparte infraestructura con otras unidades (U-02…U-07)?

Dado que el sistema es un monolito modular, las unidades comparten el mismo proceso FastAPI y la misma BD. ¿La infraestructura de U-01 se define como **infraestructura compartida** para todas las unidades?

A) Sí — U-01 define la infraestructura base (servidor, BD, storage, networking) que todas las unidades heredan; cada unidad agrega solo sus componentes específicos  
B) No — cada unidad define su propia infraestructura de forma independiente (más repetición, pero más claro por unidad)  
C) Híbrido — U-01 define el `shared-infrastructure.md` para los componentes comunes; las unidades solo documentan sus adiciones específicas  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

### Pregunta 7.2 — CI/CD Pipeline

¿Se configura un pipeline de CI/CD en el piloto?

A) GitHub Actions — pipeline automatizado: tests → build Docker → push registry → deploy al cloud en merge a `main`  
B) Despliegue manual — el dev corre `docker build` y `docker push` manualmente; sin pipeline automatizado en piloto  
C) Sin CD automatizado — CI corre tests automáticamente (GitHub Actions), pero el deploy a producción es manual y aprobado  
X) Otro (describa después del tag [Answer]:)

[Answer]: A

---

> **Próximo paso**: Una vez completadas todas las respuestas, el workflow generará los 2 artefactos de Infrastructure Design de U-01.
