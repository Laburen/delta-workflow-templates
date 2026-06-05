# delta-workflow-templates

Plantillas reutilizables para configurar **agentes de IA de codificación** (Claude Code, Cursor, Copilot, etc.) en cualquier repositorio, siguiendo el sistema de trabajo del **Delta**.

Estas plantillas estandarizan cómo un agente:

- entiende las reglas del proyecto (`AGENTS.md`),
- desarrolla una tarea de backlog (`WORKFLOW_TAREA`),
- procesa feedback de un PR (`WORKFLOW_PR_FEEDBACK`),
- arregla un bug en código ya mergeado (`WORKFLOW_BUGFIX`).

## Contenido

| Plantilla | Destino sugerido en tu repo | Propósito |
| --------- | --------------------------- | --------- |
| [`plantillas/PLANTILLA-AGENTS.md`](plantillas/PLANTILLA-AGENTS.md) | `AGENTS.md` (raíz) | Reglas de comportamiento, stack, arquitectura y prohibiciones para el agente. |
| [`plantillas/PLANTILLA-WORKFLOW_TAREA.md`](plantillas/PLANTILLA-WORKFLOW_TAREA.md) | `.claude/workflows/WORKFLOW_TAREA.md` | Cómo llevar una tarea `T-XXX-NNN` de Pendiente a PR abierto con TDD y quality gates. |
| [`plantillas/PLANTILLA-WORKFLOW_PR_FEEDBACK.md`](plantillas/PLANTILLA-WORKFLOW_PR_FEEDBACK.md) | `.claude/workflows/WORKFLOW_PR_FEEDBACK.md` | Cómo validar y aplicar feedback de reviews (Copilot, CodeRabbit, etc.). |
| [`plantillas/PLANTILLA-WORKFLOW_BUGFIX.md`](plantillas/PLANTILLA-WORKFLOW_BUGFIX.md) | `.claude/workflows/WORKFLOW_BUGFIX.md` | Cómo manejar bugs en código ya mergeado (`fix/` o `hotfix/`). |

## Cómo usar

1. Copiá las plantillas a tu repositorio en las rutas sugeridas (quitando el prefijo `PLANTILLA-`):

   ```bash
   # desde la raíz de tu proyecto
   curl -sL https://raw.githubusercontent.com/Laburen/delta-workflow-templates/main/plantillas/PLANTILLA-AGENTS.md -o AGENTS.md
   mkdir -p .claude/workflows
   for w in TAREA PR_FEEDBACK BUGFIX; do
     curl -sL "https://raw.githubusercontent.com/Laburen/delta-workflow-templates/main/plantillas/PLANTILLA-WORKFLOW_$w.md" \
       -o ".claude/workflows/WORKFLOW_$w.md"
   done
   ```

2. Reemplazá los marcadores `{PLACEHOLDER}` (p. ej. `{PROJECT_NAME}`, `{YYYY-MM-DD}`, `T-XXX-NNN`, el stack y la arquitectura) por los datos reales de tu proyecto.

3. Revisá que las rutas internas (`docs/BACKLOG-{PROJECT}.md`, ADRs, etc.) coincidan con la estructura de tu repo.

## Convenciones que asumen las plantillas

- **Git Flow**: PRs apuntan a `develop`; `main` es solo para releases (excepto `hotfix/`).
- **Una tarea = una rama = un PR**: `feature/T-XXX-NNN-slug`.
- **TDD obligatorio**: Red → Green → Refactor.
- **Sin escape hatches**: nada de `any`, `@ts-ignore`, `eslint-disable` ni `--no-verify`.
- **El Delta mergea, no el agente.**

> Las plantillas usan un stack de referencia (TypeScript + Cloudflare Workers/D1 + Vitest), pero las reglas de proceso son agnósticas del stack: adaptá las secciones de tooling a tu proyecto.
