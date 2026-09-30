# Reglas universales para agentes

Este repositorio es un framework de análisis arquitectónico reutilizable para cualquier sistema.

## Orden obligatorio
1. requirements-analyst
2. solution-leader (alcance de decisiones)
3. architect
4. database-specialist
5. security-reviewer
6. devops-architect
7. architecture-reviewer
8. solution-leader (consolidación final)

## Regla principal
Hasta aprobar la recomendación final y el ADR:
- NO generar código de la aplicación.
- NO instalar frameworks.
- NO crear migraciones.
- NO crear infraestructura real.
- NO alterar RF/RNF silenciosamente.

## Fuente de verdad
- `proyecto/00_contexto/`
- `proyecto/01_requisitos/`
- `proyecto/02_configuracion/`

## Reutilización
Para usar este framework con otro sistema:
1. copiar `plantillas/` a `proyecto/`;
2. reemplazar el ejemplo de tienda;
3. conservar `.agents/skills/`, `.opencode/agents/` y `.claude/agents/`.

## Trazabilidad
Toda decisión debe relacionarse con un RF, RNF, regla de negocio, restricción o medición.


## Política de skills
No crear skills de base de datos si ya existe uno instalado que cubra la tarea.
Los skills de BD deben provenir de repositorios registrados en
`SKILLS_SOURCES.md`.

Los skills instalados se encuentran en:
`.agents/skills/`

## Flujo de BD
Cuando se solicite diseño/implementación de base de datos:
1. leer `SKILLS_SOURCES.md`;
2. leer `.agents/workflows/02_database_workflow.md`;
3. leer `.agents/state/database-workflow.json`;
4. usar los skills externos indicados para la etapa;
5. trabajar en orden;
6. actualizar el checkpoint.

## Reglas
- No saltar al SQL antes de completar el diseño.
- No elegir DBMS por preferencia.
- Mantener trazabilidad RF/RNF → modelo → constraint/índice/decisión.
- No guardar credenciales en archivos versionados.
- No desplegar antes de `STATUS: APPROVED`.
- No ejecutar operaciones destructivas sin autorización explícita.







## Desarrollo de aplicación — API, backend y frontend

Después del workflow de base de datos usar, en orden:

1. `.agents/workflows/03_api_pilot_workflow.md`
2. `.agents/workflows/04_backend_workflow.md`
3. `.agents/workflows/05_frontend_workflow.md`
4. `.agents/workflows/06_integration_workflow.md`

States:
- `.agents/state/api-pilot-workflow.json`
- `.agents/state/backend-workflow.json`
- `.agents/state/frontend-workflow.json`
- `.agents/state/integration-workflow.json`

### Regla obligatoria de checkpoint

- ejecutar UN checkpoint por interacción;
- probarlo;
- actualizar el state;
- marcar `HUMAN_STATUS: PENDING`;
- detenerse;
- esperar otro mensaje.

### Código

Todo código nuevo se crea en:

`proyecto/06_codigo/`

### Decisiones vigentes

- `proyecto/04_decisiones/decisiones_api.md`
- `proyecto/04_decisiones/decisiones_backend.md`
- `proyecto/04_decisiones/decisiones_frontend.md`
- `proyecto/04_decisiones/decisiones_desarrollo_incremental.md`

### Stack vigente

- PHP puro, sin framework.
- MVC.
- MySQL 8.x / InnoDB.
- PDO.
- API REST JSON.
- Frontend PHP + HTML + CSS + JavaScript puro.
- Sin React/Vue/Angular en esta fase.

### Lectura progresiva

Al recibir `continúa`, leer primero state + workflow actuales.
No releer todo el repositorio salvo que exista contradicción, cambio de decisión o el checkpoint lo requiera.

## Alcance didáctico reducido 2026-09-29

La fase de desarrollo está congelada en:
- `auth_usuario`: login/logout/me;
- `cat_categoria`: API piloto;
- `cat_producto`: segundo módulo.

Pantallas: `/login`, `/dashboard`, `/categorias`, `/productos`.

No implementar otros módulos aunque existan tablas en la BD.
La tabla piloto ya fue decidida: `cat_categoria`.
Categorías y productos usan inactivación lógica.
Un checkpoint por interacción y validación humana obligatoria.
