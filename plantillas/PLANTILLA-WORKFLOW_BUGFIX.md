# Workflow: Bugfix in Merged Code

> **When to use this workflow:** the user reports a bug in code that is
> already merged to `develop` or `main`. This is **not** for bugs found
> during the development of a task (those are handled within
> `WORKFLOW_TAREA.md` Step 7).
>
> **Trigger phrases:** "Hay un bug en {feature}", "Encontré un bug en
> producción", "Fix urgente", "Hotfix", "El cron está fallando", "{X} no
> está funcionando bien".

---

## ⚠️ Before you read further

If you haven't read [`../../AGENTS.md`](../../AGENTS.md) yet in this session, **stop and read it now**. Pay special attention to **RULE 0** (no escape hatches under any circumstance, even for "urgent" fixes) and **Rule 5** (never merge yourself).

---

## 🎯 Goal of this workflow

Take a reported bug in merged code, **classify** as hotfix vs regular bugfix, **diagnose** before fixing, **reproduce with a failing test** (TDD), **apply minimal fix**, **prevent recurrence** by documenting the lesson, and **stop at PR open** (the Delta merges).

> **🚨 Critical rule for bugs:** the fix is **NEVER** done in the branch of an unrelated current task. Bugs in merged code get their own branch, their own PR, their own merge.

---

## 📋 MANDATORY TODO LIST

```
1.  Classify: hotfix (production-impacting) vs regular bugfix
2.  Gather evidence: input, output, logs, environment, expected behavior
3.  Diagnose before proposing a fix (don't jump to code)
4.  Confirm root cause with the Delta
5.  Create the correct branch (hotfix/* or fix/*) from the correct base
6.  Write failing test that reproduces the bug (Red)
7.  Implement minimal fix (Green)
8.  Verify the fix and that no other tests broke
9.  Run all quality gates (format, lint, test, build, check)
10. Update "Recurring Bugs & Lessons Learned" in AGENTS.md (if pattern-relevant)
11. Commit with Conventional Commits (`fix(T-XXX-NNN):` or `fix:` if no task)
12. Push branch
13. Open PR targeting the correct branch (main for hotfix, develop for fix)
14. Notify the Delta — STOP. Do NOT merge.
```

---

## Step 1 — Classify: hotfix vs regular bugfix

This is the most important decision in this workflow. Get it wrong and you either delay a critical production fix, or you bypass `develop` integration unnecessarily.

### Hotfix (branch: `hotfix/...`, target: `main`)

All of these must be true:
- The bug affects **production** right now (or imminently).
- The impact is **severe**: data corruption, security exposure, customer-facing failure, money loss, SLA breach.
- **Cannot wait** for the normal `develop → main` release cycle.

Examples:
- WA template rejected, breaking the proactive contact flow today.
- Cron deleting wrong records in D1.
- Webhook endpoint leaking secrets in logs.

### Regular bugfix (branch: `fix/...`, target: `develop`)

Any of these is enough:
- The bug is in `develop` but **not yet released to production**.
- The bug is in production but its impact is **minor and bounded** (cosmetic, edge case with workaround, rare).
- The fix can wait until the next normal release.

Examples:
- Pagination off-by-one in an admin endpoint.
- Logged date is one day shifted in a non-critical audit field.
- A retry happens twice in a corner case where it doesn't cause damage.

### Ask the Delta if unclear

If you can't determine the classification with confidence, **ask explicitly**:

> "Antes de empezar — ¿clasificás esto como hotfix (afecta producción ahora,
> branch desde main) o como fix regular (puede ir por develop)? Mi
> percepción es {X} porque {reason}. ¿Confirmás?"

Wait for the Delta's response.

---

## Step 2 — Gather evidence

Before any code, collect:

- **Input** that triggers the bug (exact request body, message, cron time, file content).
- **Actual output** (what happened — error message, wrong data, missing side effect).
- **Expected output** (what should have happened).
- **Logs / stack traces** if available.
- **Environment** (production / staging / local).
- **When it started** (first observation, recent deploys, recent merges).
- **Reproducibility** (always, sometimes, once).

If any of this is missing, ask the Delta:

> "Para diagnosticar bien necesito: {missing pieces}. ¿Los podés pasar?"

Do not proceed without enough evidence. A fix without reproduction is a guess.

---

## Step 3 — Diagnose BEFORE proposing a fix

This is where most agents go wrong: they read the symptom and jump to "fix" code. **Don't.**

Read the relevant code paths:

```bash
# Conceptually
grep -rn "{related-keyword}" src/
```

Read referenced ADRs from `docs/DECISIONES-ARQUITECTONICAS-PENDIENTES.md` — sometimes "bugs" are actually the intended behavior per ADR.

Read the "Recurring Bugs & Lessons Learned" section of `AGENTS.md` — the bug may be a recurrence of something documented.

Then propose a diagnosis (not a fix) to the Delta:

> "Diagnóstico — las 3 causas más probables, ordenadas por probabilidad:
>
> **A. {Hypothesis A}** ({rationale}).
>   Cómo descartar: {experiment}.
>
> **B. {Hypothesis B}** ({rationale}).
>   Cómo descartar: {experiment}.
>
> **C. {Hypothesis C}** ({rationale}).
>   Cómo descartar: {experiment}.
>
> ¿Cuál descartamos primero?"

Iterate with the Delta until **one** root cause is confirmed. This step is non-skippable.

---

## Step 4 — Confirm root cause with the Delta

Before writing any code, write back to the Delta:

> "Root cause confirmado: {1-2 sentence description}. El fix va a tocar:
> {files}. Test que voy a escribir primero para reproducir el bug:
> {description of failing test}. ¿Confirmás antes de arrancar?"

Wait for confirmation. Then continue.

If the Delta rejects the diagnosis ("eso no es", "el problema es otro"): go back to Step 3 with the new info.

---

## Step 5 — Create the correct branch

### For hotfix

```bash
git checkout main
git pull origin main
git checkout -b hotfix/short-description-of-bug
```

Slug rules: short, kebab-case, describes the bug (not the fix).
- `hotfix/wa-template-consulta-rejected` ✅
- `hotfix/fix-template-issue` ❌ (vague)

### For regular fix

```bash
git checkout develop
git pull origin develop
git checkout -b fix/short-description-of-bug
```

If the bug is associated with a known T-XXX-NNN, include it in the branch slug:
- `fix/T-XXX-007-duplicate-detection-edge-case` ✅

---

## Step 6 — Write the failing test (Red)

Write a test that **reproduces the bug** — i.e., that **fails** on current code.

```typescript
import { describe, it, expect } from 'vitest';
// ... imports

describe('createPedido — bug fix: tarifa_modo edge case', () => {
  it('rejects input when tarifa_modo=por_acuerdo and tarifa_str is empty', async () => {
    // Arrange — input that currently passes but shouldn't
    const input = { ...validInput, tarifa_modo: 'por_acuerdo', tarifa_str: '' };
    // Act + Assert
    await expect(createPedido(input, mockEnv)).rejects.toThrow(/tarifa_str required/);
  });
});
```

Run it:

```bash
pnpm test {path}
```

The test **must fail**. If it passes on current code, you're not reproducing the bug — re-investigate.

---

## Step 7 — Implement minimal fix (Green)

Write the smallest possible change that makes the failing test pass. **Resist the urge to "improve" surrounding code** — that's scope creep, and creates a worse PR for review.

If you find that the fix requires a larger refactor:
- Stop.
- Tell the Delta:
  > "El fix mínimo arregla el bug, pero el patrón está mal diseñado y va a
  > recaer. Propongo: fix mínimo en este PR + tarea de refactor T-XXX-NNN
  > en el backlog. ¿Confirmás?"
- Apply only the minimal fix in this branch; let the Delta decide on the refactor task separately.

---

## Step 8 — Verify

Run the full test suite:

```bash
pnpm test
```

- The previously failing test now passes (Green).
- **No other test broke** as a side effect of the fix. If any test newly fails, you've introduced a regression. Investigate before continuing.

---

## Step 9 — Quality gates

```bash
pnpm format
pnpm lint
pnpm test:coverage
pnpm build
pnpm check
```

All green. Coverage targets met. If anything fails, fix it. **Never** `--no-verify` — even for hotfixes. Skipping checks introduces follow-up bugs.

---

## Step 10 — Update "Recurring Bugs & Lessons Learned"

Most bugs that escape to merged code share a pattern. Document the bug in `AGENTS.md` under the "Recurring Bugs & Lessons Learned" section, so future agents (and humans) don't repeat it.

Template entry:

```markdown
### {YYYY-MM-DD} — {Short bug title}

**Where:** {file/module/area}
**Symptom:** {how the bug manifested in production / dev — be concrete}
**Root cause:** {what was actually wrong, in 1-2 sentences}
**Fix:** {the correct approach, with a small code example if helpful}
**❌ Do NOT:** {the tempting wrong approach that recreates the bug}
**✅ DO:** {the correct pattern, with code if useful}
```

Example:

```markdown
### 2026-05-18 — tarifa_str empty when tarifa_modo=por_acuerdo

**Where:** `src/schemas/pedido.ts` (PedidoCreateSchema)
**Symptom:** Pedidos with empty tarifa string in WA template, breaking the
   {{5}} variable rendering on Meta side. Manifested as templates being
   rejected by Meta with "empty parameter" errors.
**Root cause:** Zod schema validated `tarifa_str` as `z.string()` instead of
   `z.string().min(1)`. Empty strings passed validation.
**Fix:** Tighten the Zod schema and add `refine()` for the cross-field rule:
   when `tarifa_modo === 'por_acuerdo'`, `tarifa_str.length >= 1`.
**❌ Do NOT:** validate `tarifa_str` with `z.string()` alone.
**✅ DO:** `z.string().min(1, 'tarifa_str must be non-empty')` combined with
   a `refine` for the conditional case.
```

**Skip this step only if** the bug is not pattern-relevant (one-off config typo, dependency upgrade artifact, etc.). When in doubt, document — costs nothing, prevents future recurrences.

---

## Step 11 — Commit

Conventional Commits format with `fix:` type. Include the task ID if the bug is tied to a known task; otherwise just `fix:`:

```bash
git commit -m "fix(T-XXX-007): reject empty tarifa_str when tarifa_modo=por_acuerdo"
git commit -m "test(T-XXX-007): add regression test for empty tarifa_str case"
git commit -m "docs: document tarifa_str validation lesson in AGENTS.md"
```

Pre-commit hook (`pnpm check`) must pass. If it fails — even on a hotfix — **fix the underlying issue**. A hotfix that breaks types is worse than no hotfix.

---

## Step 12 — Push

```bash
git push -u origin hotfix/{slug}    # for hotfix
# OR
git push -u origin fix/{slug}        # for regular fix
```

---

## Step 13 — Open the PR with the correct target

### Hotfix → target `main`

```bash
gh pr create \
  --base main \
  --title "fix: {short description of bug}" \
  --body "$(cat <<'EOF'
## 🚨 Hotfix

**Production impact:** {what was broken — be concrete}
**Severity:** {critical / high}
**Root cause:** {1-2 sentence summary}

## Fix
{What was changed, in 1-2 bullets.}

## Test
- New regression test in {path} reproducing the bug (was Red, now Green).
- Full test suite green.
- Coverage maintained.

## Follow-up
- After merge to main, the Delta back-merges main → develop to keep both
  branches in sync.
- Documented in AGENTS.md "Recurring Bugs & Lessons Learned" for future
  agents/humans.
EOF
)"
```

### Regular fix → target `develop`

```bash
gh pr create \
  --base develop \
  --title "fix(T-XXX-NNN): {short description}" \
  --body "$(cat <<'EOF'
## What this resolves
{Description of the bug, where it manifested, who reported it.}

## Root cause
{1-2 sentences.}

## Fix
{What was changed, in 1-2 bullets.}

## Test
- New regression test in {path} reproducing the bug.
- Full test suite green.
- Coverage maintained.

## Documented in
- AGENTS.md "Recurring Bugs & Lessons Learned" (if pattern-relevant).
EOF
)"
```

---

## Step 14 — STOP

Notify the Delta:

> "PR de {hotfix|fix} abierto: {link}. CI corriendo. Cuando esté verde,
> mergeás vos. Para hotfix recordá que después hay que back-mergear main →
> develop para que develop tenga el fix. NO mergeo."

If the PR receives review feedback → switch to `WORKFLOW_PR_FEEDBACK.md`.

After the Delta confirms the merge:

- For hotfix, the Delta also back-merges `main` → `develop`. You can offer to do the back-merge PR if asked, but you do not initiate it.
- For regular fix, no extra step.

You **do not** auto-continue to the next task. Wait for the next explicit request.

---

## ❌ DO NOT

- Skip the diagnosis step. Jumping straight to a fix is how you create new bugs.
- Bypass tests "because it's a hotfix" — TDD applies regardless of urgency.
- Use `--no-verify`, `any`, `@ts-ignore`, or `eslint-disable` "just this once".
- Fix the bug in the branch of an unrelated current task.
- Mix the bugfix with refactor work in the same PR.
- Target `develop` for a hotfix, or `main` for a regular fix.
- Skip documenting the lesson if the bug is pattern-relevant.
- Merge the PR yourself.

## ✅ DO

- Classify hotfix vs regular bugfix before doing anything.
- Gather evidence before diagnosing.
- Diagnose before fixing — propose root causes, descartá con experimentos.
- Test first (Red) — the failing test is the proof you understood the bug.
- Apply the minimal fix.
- Document the lesson in `AGENTS.md`.
- Stop at PR open and let the Delta merge.

---

**End of WORKFLOW_BUGFIX** — return to `AGENTS.md` if you need any other rule.
