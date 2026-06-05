# Workflow: Backlog Task Development

> **When to use this workflow:** the user asks you to start, develop, or work on
> a backlog task identified as `T-XXX-NNN` (e.g., "Iniciar T-IF-007", "Start
> T-XXX-012", "Develop the next task").
>
> **Trigger phrases:** "Iniciar T-XXX-NNN", "Start T-XXX-NNN", "Develop
> T-XXX-NNN", "Empezar T-XXX-NNN", "Continuar con T-XXX-NNN".

---

## ⚠️ Before you read further

If you haven't read [`../../AGENTS.md`](../../AGENTS.md) yet in this session, **stop and read it now**. The rules in AGENTS.md apply to every step below.

---

## 🎯 Goal of this workflow

Take one atomic backlog task (T-XXX-NNN) from **Pending** to **PR Open** following TDD, quality gates, and Git Flow — without compacting context, without skipping steps, and without merging the PR (the Delta merges).

---

## 📋 MANDATORY TODO LIST

Create this TODO list at the start of the task. Check items off **immediately** as you complete them. Do not batch them at the end.

```
1.  Read the task block from docs/BACKLOG-{PROJECT}.md
2.  Read referenced ADRs and AGENTS.md sections
3.  Confirm understanding (or ask the Delta if anything is ambiguous)
4.  Verify atomicity (≤5 files, ≤4h human estimate); flag the Delta if not
5.  Create feature branch from up-to-date develop
6.  Write failing test (Red)
7.  Implement minimal code to pass test (Green)
8.  Refactor if needed (keep tests green)
9.  Run pnpm format
10. Run pnpm lint
11. Run pnpm test (all tests pass, coverage targets met)
12. Run pnpm build
13. Run pnpm check (final gate)
14. Notify the Delta for manual E2E test (if applicable)
15. Update docs/BACKLOG-{PROJECT}.md (task ✅ COMPLETADA, sub-tasks [x])
16. Commit with Conventional Commits
17. Push branch to origin
18. Open PR targeting develop with full description
19. STOP. Notify the Delta. Do NOT merge.
```

If any step fails, do **not** skip ahead. Stop, surface the problem to the Delta, and only continue once it's resolved.

---

## Step 1 — Read the task block

Read **only** the specific T-XXX-NNN block from `docs/BACKLOG-{PROJECT}.md`. Do not load the entire backlog.

Use grep or targeted file read:

```bash
# Conceptually — adapt to your tool
grep -A 30 "T-XXX-007" docs/BACKLOG-{PROJECT}.md
```

Extract from the block:

- **Title**
- **Priority** (CRITICA / ALTA / MEDIA / BAJA; flag if RUTA CRÍTICA)
- **Hybrid estimation** (informational only — does not change how you work)
- **Dependencies** (other T-XXX, external blockers)
- **Asana task** (sprint name)
- **HUI covered** (note the HU IDs — you may need to verify acceptance criteria)
- **Description** (1-2 paragraphs)
- **Specific sub-tasks** (the checklist you will execute)
- **Acceptance criteria** (what "done" means)
- **Technical notes** (schemas, commands, API links — read these)

---

## Step 2 — Read referenced ADRs and AGENTS.md sections

For each ADR mentioned in "Notas Técnicas" or implied by the domain, read the corresponding section from `docs/DECISIONES-ARQUITECTONICAS-PENDIENTES.md`. Skip closed ADRs that are not referenced; skip all pending ADRs unless explicitly relevant.

If the task touches an integration documented in `AGENTS.md` under "External Integrations", read that subsection.

If the task touches an area listed in `AGENTS.md` under "Recurring Bugs & Lessons Learned", **read that entry first**. Many recurring bugs are caused by agents "improving" code without knowing why it was written that way.

---

## Step 3 — Confirm understanding

Before writing any code, write a short summary back to the Delta:

> "Voy a hacer T-XXX-007: {1-sentence description}. Toca: {files/modules}.
> Tests que voy a escribir primero: {list of test cases}. ADRs relevantes:
> {list}. ¿Confirmás o algo está mal interpretado?"

Wait for confirmation. If the Delta says "go", proceed to Step 4.

If anything in the task block is ambiguous (a sub-task is vague, an acceptance criterion is unclear, the dependency state is unknown), **ask explicit questions**. Do not invent answers.

---

## Step 4 — Verify atomicity

Before creating the branch, sanity-check the task:

- **Files touched:** does the task plausibly touch ≤5 source files? Count: schemas, services, handlers, tests, docs.
- **Sub-tasks:** are there ≤10 specific sub-tasks?
- **Hybrid estimate:** is it ≤3h hybrid (which means ≤5h human base)?

If **any** of these fails, the task may have been split badly. Flag the Delta:

> "Esta tarea parece más grande de lo atómico. Propongo dividirla en:
> {your proposal — list 2-3 sub-tasks with their own T-XXX-NNN}. ¿Querés
> que actualicemos el backlog antes de arrancar?"

Wait for the Delta's response. Either:
- The Delta agrees → update `BACKLOG-{PROJECT}.md`, then start with the first sub-task.
- The Delta confirms the task is OK as-is → proceed.

---

## Step 5 — Create feature branch

```bash
git checkout develop
git pull origin develop
git checkout -b feature/T-XXX-NNN-short-slug
```

Slug rules:
- Short (≤6 words).
- Lowercase, kebab-case.
- Describes the **what**, not the how.
- Examples:
  - `feature/T-IF-007-creacion-pedido` ✅
  - `feature/T-IF-007-implementar-el-servicio-completo` ❌

Verify you're on the new branch:

```bash
git status   # Should show "On branch feature/T-XXX-NNN-..."
```

---

## Step 6 — Write the failing test (Red)

For **each** acceptance criterion, write a test **before** writing the implementation.

Test location: `tests/unit/{module}/{name}.test.ts` (or `tests/integration/...` for cross-service flows).

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

describe('createPedido', () => {
  // Setup mocks
  beforeEach(() => vi.clearAllMocks());

  it('rejects input with tarifa_modo=fija and missing tarifa_valor', async () => {
    // Arrange
    const input = { ...validInput, tarifa_modo: 'fija', tarifa_valor: undefined };
    // Act + Assert
    await expect(createPedido(input, mockEnv)).rejects.toThrow(/tarifa_valor required/);
  });

  // ... one test per acceptance criterion
});
```

Run the tests **before** the implementation exists:

```bash
pnpm test
```

The tests **must fail**. If a test passes without implementation, it's testing the wrong thing. Rewrite.

---

## Step 7 — Implement minimal code (Green)

Write the minimum code necessary to make the failing tests pass. **Resist the urge to gold-plate** — no premature abstractions, no "while I'm here" refactors.

Follow project conventions from `AGENTS.md`:
- TypeScript strict.
- No `any` — use `unknown` + Zod validation for external input.
- `async/await` only — no `.then()` chains.
- Typed errors with `code` and `statusCode`.

Run tests after each significant change:

```bash
pnpm test {path-to-test-file}
```

When all tests pass: Green achieved. Continue.

---

## Step 8 — Refactor (keep tests green)

Now you can clean up: extract duplicates, rename for clarity, simplify logic. After every change, re-run the test to confirm green.

**Do not** add functionality during refactor. Refactor = improving structure without changing behavior.

---

## Step 9 — `pnpm format`

```bash
pnpm format
```

Prettier auto-formats. Do not skip this step — CI will fail if format is inconsistent.

---

## Step 10 — `pnpm lint`

```bash
pnpm lint
```

If errors appear:
- **Read each error.** Do not blindly auto-fix everything.
- For real issues, fix them.
- For style issues only, use `pnpm lint:fix`.
- **Never** add `// eslint-disable-next-line` to silence a rule. See AGENTS.md RULE 0.

---

## Step 11 — `pnpm test` with coverage

```bash
pnpm test:coverage
```

Confirm:
- All tests pass.
- Coverage for business logic ≥ 80%.
- Coverage for integrations ≥ 60%.

If coverage is below target, add tests for the uncovered branches before continuing.

---

## Step 12 — `pnpm build`

```bash
pnpm build
```

Build must succeed. If it fails, fix the underlying issue. Common causes:
- Type error revealed by build that wasn't caught in dev.
- Missing dependency.
- Wrangler config issue.

---

## Step 13 — `pnpm check`

```bash
pnpm check
```

This is the same gate the pre-commit hook runs. If `pnpm check` is green, the commit will succeed. If it fails here, fix before committing.

---

## Step 14 — Notify the Delta for manual E2E test (if applicable)

If the task is testable end-to-end (HTTP endpoint, agent action, cron, integration), tell the Delta:

> "T-XXX-007 listo para test manual E2E. Para probar:
> 1. `pnpm dev`
> 2. {Specific steps the Delta should run — curl command, simulated WA message, cron trigger, etc.}
> 3. Expected side effects: {list — D1 records, outgoing messages, audit log entries}
> ¿Querés probarlo antes de que abra el PR?"

**Wait for the Delta's response.** Do not proceed to commit until the Delta either:
- Confirms manual E2E passed.
- Confirms E2E is not applicable for this task (pure schema, config-only, etc.).
- Reports a bug — in which case you go to "Bug found in E2E" below.

### Bug found in E2E

If the Delta reports a bug:

1. Ask for: input, output, logs, Delta's hypothesis.
2. **Do not propose a fix immediately.** Diagnose first:
   > "Antes de fixear: las 3 causas más probables son {A, B, C}. Para descartar
   > A, probamos {experiment}. Para descartar B, {experiment}. ¿Cuál descartamos
   > primero?"
3. Once the root cause is clear: TDD again. Test that reproduces the bug (Red) → fix (Green).
4. Re-run steps 9-13 (format, lint, test, build, check).
5. Re-notify the Delta for another E2E pass.
6. Repeat until E2E is clean.

---

## Step 15 — Update the backlog

Edit `docs/BACKLOG-{PROJECT}.md`:

1. Mark the task as completed:

```markdown
### T-XXX-NNN: {title}

**Prioridad:** ...
**Estimación:** ...
**Dependencias:** ...
**Estado:** ✅ COMPLETADA — {YYYY-MM-DD}
```

2. Mark every sub-task as `[x]`:

```markdown
#### Tareas específicas

- [x] Sub-task 1
- [x] Sub-task 2
- [x] Sub-task 3
```

3. Update the summary table at the end of the backlog (status column).

**Do not** skip this step. The backlog is the source of truth for sprint progress; if it's out of sync, the next session will be confused.

---

## Step 16 — Commit

Use Conventional Commits with the task ID in scope. Multiple commits per task are fine if they reflect logical units of work:

```bash
git add {files}
git commit -m "feat(T-XXX-007): add createPedido service with Zod validation"

git add {test-files}
git commit -m "test(T-XXX-007): add coverage for duplicate detection and id generation"

git add docs/BACKLOG-{PROJECT}.md
git commit -m "docs(T-XXX-007): mark task as completed in backlog"
```

The pre-commit hook will run `pnpm check`. If it fails, **fix the underlying issue** — never `--no-verify`.

---

## Step 17 — Push

```bash
git push -u origin feature/T-XXX-NNN-short-slug
```

---

## Step 18 — Open the PR

```bash
gh pr create \
  --base develop \
  --title "feat(T-XXX-NNN): {short title}" \
  --body "$(cat <<'EOF'
## What this resolves
{1-2 sentences referencing HUI-NNN and the business value.}

## How it was resolved
{Bullet list of the key implementation decisions.}

## ADRs applied
{List the ADRs this PR depends on or implements.}

## What was tested
- Unit tests: {count} new tests covering {scenarios}.
- Manual E2E: {what the Delta tested, or "N/A — schema only".}
- Coverage: {percentage achieved on the touched modules}.

## Notes for reviewer
{Any context the reviewer needs — known limitations, follow-up tasks, etc.}

Closes: {Asana task link if available}
EOF
)"
```

PR target: **`develop`**. Never `main` (except for hotfix branches — see `WORKFLOW_BUGFIX.md`).

---

## Step 19 — STOP

> **You DO NOT merge the PR. The Delta does.**

Notify the Delta:

> "PR abierto: {link}. CI corriendo. Avisame cuando reciba el feedback del
> reviewer y procesamos. NO mergeo — lo hacés vos cuando estés conforme."

Stay available for follow-up:
- When the AI reviewer (Copilot / CodeRabbit / Qodo / manual) leaves comments, the Delta will trigger `WORKFLOW_PR_FEEDBACK.md`.
- When the Delta confirms the merge happened, you may mark the Asana subtask as completed in your final response.

---

## ❌ DO NOT

- Skip any step in the TODO list above.
- Implement before writing the failing test.
- Add features not in the sub-task list.
- Refactor unrelated code "while you're there".
- Use `any`, `@ts-ignore`, or `eslint-disable`.
- Use `--no-verify` to bypass the pre-commit hook.
- Open the PR against `main`.
- Merge the PR yourself, ever.
- Continue to the next backlog task without explicit user request.

## ✅ DO

- Read the task block in full from `docs/BACKLOG-{PROJECT}.md`.
- Ask the Delta when something is ambiguous.
- Test first, implement second.
- Update the backlog before the final commit.
- Stop at PR open and let the Delta merge.

---

**End of WORKFLOW_TAREA** — return to `AGENTS.md` if you need to consult any other rule.
