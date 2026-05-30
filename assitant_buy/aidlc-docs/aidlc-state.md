# AI-DLC State Tracking — Assistent Buy AI Agent

## Project Information
- **Project Name**: Assistent Buy AI Agent
- **Project Type**: Greenfield
- **Start Date**: 2026-05-23T21:43:00Z
- **Current Stage**: CONSTRUCTION PHASE — U-03 DISEÑOS COMPLETOS (Functional + NFR + Infra aprobados). Siguiente: U-04 Functional Design (Code Generation diferido — Opción C)

## Modo de Ejecución (Decisión 2026-05-27)
El usuario decidió **diferir Code Generation de todas las unidades** y completar primero los diseños (Functional Design + NFR Requirements + Infrastructure Design) de cada unidad. Una vez completados los diseños de U-01 a U-07, se ejecutará Code Generation por unidad en una segunda pasada. Esto invierte el orden estándar del Per-Unit Loop del AI-DLC para esta ejecución.

## Workspace State
- **Existing Code**: No
- **Reverse Engineering Needed**: No (Greenfield)
- **Workspace Root**: `c:\Users\Orlando\Google Drive\ProjectLife\Estudio\Hardcore30X\hardcore30x_estacion4\assitant_buy`
- **PRD Available**: Yes — `assitant_buy_prd.md`

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Extension Configuration
| Extension | Enabled | Decidido en |
|---|---|---|
| Security Baseline | ✅ Sí — todas las reglas bloqueantes | Requirements Analysis |
| Property-Based Testing | ✅ Parcial — PBT-02, PBT-03, PBT-07, PBT-08, PBT-09 bloqueantes | Requirements Analysis |

## Stage Progress
- [x] Workspace Detection — COMPLETED (2026-05-23T21:43:00Z)
- [x] Reverse Engineering — SKIPPED (Greenfield)
- [x] Requirements Analysis — COMPLETED (2026-05-23T22:11:00Z)
- [x] User Stories — COMPLETED (2026-05-23T23:07:00Z)
- [x] Workflow Planning — COMPLETED (2026-05-23T23:29:00Z)
- [x] Application Design — COMPLETED (2026-05-24T12:45:00Z)
- [x] Units Generation — COMPLETADO (2026-05-25T21:36:00Z)
- [/] Construction Phase — IN PROGRESS
  - [x] U-01 Functional Design — COMPLETADO (2026-05-26T01:14:00Z)
  - [x] U-01 NFR Requirements — COMPLETADO (2026-05-26T01:38:00Z)
  - [x] U-01 Infrastructure Design — APROBADO (2026-05-26T16:22:00Z)
  - [x] U-02 Functional Design — APROBADO (2026-05-26T21:33:00Z)
  - [x] U-02 NFR Requirements — APROBADO (2026-05-26T22:06:00Z)
  - [x] U-02 Infrastructure Design — APROBADO (2026-05-26T22:39:00Z)
  - [ ] U-02 Code Generation — DIFERIDO (decisión 2026-05-27 — se ejecuta tras terminar diseños U-03..U-07)
  - [x] U-03 Functional Design — APROBADO (2026-05-29; integración con schema SAB real, scope MVP=BIENES)
  - [x] U-03 NFR Requirements — APROBADO 2026-05-29 (nfr-requirements.md + tech-stack-decisions.md. Nota: Q5.3 generó delta al FD U-03 — cache catálogo eliminado, BR-U03-19 revisada)
  - [x] U-03 Infrastructure Design — APROBADO 2026-05-29 (infrastructure-design.md + deployment-architecture.md. Hereda infra U-01/U-02; agrega egress Anthropic + secret + prefijo S3 llm_calls). ✅ U-03 con todos sus diseños completos
  - [ ] U-03 Code Generation — DIFERIDO
  - [ ] U-04 Functional Design — PENDIENTE
  - [ ] U-04 NFR Requirements — PENDIENTE
  - [ ] U-04 Infrastructure Design — PENDIENTE
  - [ ] U-04 Code Generation — DIFERIDO
  - [ ] U-05 Functional Design — PENDIENTE
  - [ ] U-05 NFR Requirements — PENDIENTE
  - [ ] U-05 Infrastructure Design — PENDIENTE
  - [ ] U-05 Code Generation — DIFERIDO
  - [ ] U-06 Functional Design — PENDIENTE
  - [ ] U-06 NFR Requirements — PENDIENTE
  - [ ] U-06 Infrastructure Design — PENDIENTE
  - [ ] U-06 Code Generation — DIFERIDO
  - [ ] U-07 Functional Design — PENDIENTE
  - [ ] U-07 NFR Requirements — PENDIENTE
  - [ ] U-07 Infrastructure Design — PENDIENTE
  - [ ] U-07 Code Generation — DIFERIDO

## Cross-Unit Deltas Pendientes (estrategia Opción C — aplicar al iniciar Code Generation)

Estos deltas emergieron durante el diseño de unidades posteriores. **Se aplican al Functional Design de la unidad afectada como paso 1 del plan de Code Generation de esa unidad** (no antes). Durante el diseño de otras unidades, se asumen como ya aplicados.

| Unidad afectada | Archivo de deltas | Sección | Aplicar antes de | Estado |
|---|---|---|---|---|
| U-01 | `construction/u03-extraccion/functional-design/cross-unit-deltas.md` | §1 | Code Generation U-01 | PENDIENTE |
| U-02 | `construction/u03-extraccion/functional-design/cross-unit-deltas.md` | §2 | Code Generation U-02 | PENDIENTE |
| U-06 | `construction/u03-extraccion/functional-design/cross-unit-deltas.md` | §3 | Code Generation U-06 | PENDIENTE |

**Mecánica de aplicación** (cuando llegue el Code Generation de la unidad):
1. Leer el archivo de deltas correspondiente
2. Aplicar los cambios al Functional Design de la unidad (entidades, componentes, flujos)
3. Re-validar con el usuario el Functional Design actualizado
4. Generar el código

**Acumulación de nuevos deltas**: durante el diseño de U-04..U-07 pueden emerger más deltas. Se agregan al mismo `cross-unit-deltas.md` (o se crean archivos análogos en las carpetas de las unidades originadoras) y se aplican igualmente al Code Generation correspondiente.

**Banners de aviso colocados en**:
- `construction/u01-fundacion/functional-design/domain-entities.md` (banner al inicio)
- `construction/u02-ingesta/functional-design/domain-entities.md` (banner al inicio)
- `construction/u02-ingesta/functional-design/business-logic-model.md` (banner al inicio)
