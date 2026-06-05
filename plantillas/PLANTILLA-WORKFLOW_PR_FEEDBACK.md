# Workflow: PR Feedback Processing

> **When to use this workflow:** the user pastes PR review comments (from
> GitHub Copilot, CodeRabbit, Qodo, Cursor BugBot, or any other reviewer)
> and asks you to process them.
>
> **Trigger phrases:** "Tengo feedback del PR", "PR feedback de T-XXX-NNN",
> "Review del PR", "Procesá este feedback", "Acá están los comentarios del
> review".

---

## ⚠️ Before you read further

If you haven't read [`../../AGENTS.md`](../../AGENTS.md) yet in this session, **stop and read it now**. Pay special attention to **Rule 3** (validate PR feedback before applying) and **Rule 5** (never merge PRs yourself).

---

## 🎯 Goal of this workflow

Take raw PR feedback from an AI reviewer, **validate** every item against the actual PR diff, **categorize** valid vs invalid vs unclear, **report** to the Delta with a structured summary, **wait** for explicit confirmation, **apply** only approved changes, and **stop** before merging (the Delta merges).

> **🚨 Critical rule:** AI reviewers (Copilot, CodeRabbit, Qodo, etc.) hallucinate. They reference files not in the PR, suggest patterns that contradict project conventions, and produce false positives. **NEVER** apply feedback automatically.

---

## 📋 MANDATORY TODO LIST

```
1.  Identify the PR (number, branch, task T-XXX-NNN)
2.  Fetch the PR diff and confirm which files are touched
3.  Parse the feedback into discrete items
4.  Validate each item against the PR diff (file in PR? yes/no)
5.  Categorize: ✅ Valid / ❌ Invalid / ⚠️ Unclear
6.  Cross-check valid items against AGENTS.md conventions
7.  Report to the Delta with the structured format below
8.  Wait for explicit confirmation (do NOT apply yet)
9.  Apply approved items in separate commits, one logical group per commit
10. Re-run quality gates (format, lint, test, build, check)
11. Push the new commits
12. Notify the Delta that CI is re-running
13. STOP. Do NOT merge. The Delta merges.
```

---

## Step 1 — Identify the PR

Extract from the user's input or from `gh pr list` / current branch:

- **PR number** (e.g., `#42`)
- **Branch name** (e.g., `feature/T-XXX-007-creacion-pedido`)
- **Task ID** (e.g., `T-XXX-007`)
- **Target branch** (should be `develop`; if it's `main`, only valid for hotfixes — see `WORKFLOW_BUGFIX.md`)

Verify you're on the correct branch:

```bash
git branch --show-current
# Should match the PR's branch
```

If not, switch:

```bash
git checkout {branch-name}
git pull origin {branch-name}
```

---

## Step 2 — Fetch the PR diff

Get the **authoritative list of files in the PR**:

```bash
gh pr diff {PR_NUMBER} --name-only > /tmp/pr-files.txt
cat /tmp/pr-files.txt
```

This is the **only** source of truth for which files are in the PR. Do not trust the reviewer's file references without checking this list.

---

## Step 3 — Parse the feedback

The Delta pastes the feedback as text. Parse it into discrete items. Each item has:

- **Item ID** (assign sequentially: F1, F2, F3, ...)
- **File mentioned** (verbatim from the feedback)
- **Line mentioned** (if any)
- **Severity** (if the reviewer assigned one: critical / major / minor / suggestion)
- **Description** (what the reviewer says is wrong)
- **Suggested fix** (if the reviewer proposed code)

If the feedback is in a structured format (e.g., CodeRabbit comments), preserve the structure. If it's unstructured prose, normalize it into the fields above.

---

## Step 4 — Validate each item against the PR diff

For each item:

1. **Is the mentioned file in `/tmp/pr-files.txt`?**
   - Yes → potentially valid (continue to Step 5).
   - No → **Invalid** (the reviewer is referencing a file not in this PR).

2. **Is the mentioned line within the diff hunks of that file?**

   ```bash
   gh pr diff {PR_NUMBER} -- {file_path}
   ```

   - Line is in a diff hunk → potentially valid.
   - Line is outside the diff (in untouched code) → **Unclear** (the reviewer may be commenting on context, not on the change). Flag for Delta decision.

---

## Step 5 — Categorize

Assign each item one category:

### ✅ Valid

The reviewer's comment references a real file/line in the PR, the issue is reproducible, and the suggestion aligns with `AGENTS.md` conventions.

### ❌ Invalid

Any of:
- File is not in the PR.
- The "issue" is actually a project convention (e.g., reviewer suggests using `any` "for flexibility" — explicitly forbidden by RULE 0).
- The suggestion contradicts a closed ADR.
- The "bug" doesn't reproduce (the reviewer hallucinated).
- The suggestion would introduce a regression for a documented test case.

### ⚠️ Unclear

- File is in PR but the line referenced is in untouched code (reviewer commenting on context).
- The suggestion is a stylistic preference not covered by `AGENTS.md` or Prettier/ESLint config.
- The issue is real but the suggested fix is questionable — needs Delta judgment.
- You can't tell whether the suggestion improves or worsens the code without more context.

---

## Step 6 — Cross-check against AGENTS.md

For every item in ✅ Valid, do a second pass:

- Does the suggested fix violate **RULE 0** (no `any`, no `@ts-ignore`, no `eslint-disable`)? → reclassify as ❌ Invalid.
- Does it violate **RULE 0.2** (`.then()` chains forbidden)? → ❌ Invalid.
- Does it violate **RULE 0.3** (Zod at boundaries)? → ❌ Invalid.
- Does it contradict a closed ADR? → ❌ Invalid.
- Does it conflict with a "Recurring Bugs & Lessons Learned" entry? → ❌ Invalid (this is exactly the pattern those entries exist to prevent).

This is non-negotiable. An AI reviewer that suggests using `any` is wrong, even if it provides an elaborate justification.

---

## Step 7 — Report to the Delta

Use **exactly** this format. Do not abbreviate, do not skip categories even if empty:

```
📋 ANÁLISIS DE FEEDBACK DEL PR #{N} — T-XXX-NNN

Total de items: {N}
- ✅ Válidos: {X}
- ❌ Inválidos: {Y}
- ⚠️ Dudosos: {Z}

==========================================
✅ FEEDBACK VÁLIDO ({X} items)
==========================================

F1 — {file}:{line}
Severidad: {severity}
Issue: {description}
Acción propuesta: {your concrete proposal for the fix}

F2 — ...

==========================================
❌ FEEDBACK INVÁLIDO ({Y} items)
==========================================

F3 — {file}:{line}
Razón: {why this is invalid — e.g., "archivo no está en el PR", "viola RULE 0
(any)", "contradice ADR-3 sobre modelo relacional"}
Recomendación: descartar.

==========================================
⚠️ FEEDBACK DUDOSO ({Z} items)
==========================================

F4 — {file}:{line}
Issue: {description}
Por qué es dudoso: {your reasoning — e.g., "la sugerencia mejora legibilidad
pero el patrón actual está justificado por X"}
Necesito tu decisión: {specific question for the Delta}

==========================================

¿Procedo a aplicar los {X} items válidos? Para los dudosos, decime cuáles aplico
y cuáles descarto. NO voy a aplicar nada hasta tu confirmación explícita.
```

**Do not proceed until the Delta responds.** This is the most important wait point in the workflow — applying feedback automatically is the #1 way agents introduce bugs in PR review cycles.

---

## Step 8 — Wait for explicit confirmation

The Delta will respond with one of:

- **"Aplicá los válidos"** → apply all ✅ items, skip ❌ and ⚠️.
- **"Aplicá F1, F2, F4 y F5; descartá F3"** → apply only the listed items.
- **"Cambialo así: ..."** → the Delta gives custom instructions; follow them.
- **"Releé F3 que sí aplica porque ..."** → re-evaluate F3 with the new context.

If the Delta's response is ambiguous, **ask** before applying. Never assume.

---

## Step 9 — Apply approved items

Apply each approved item. Group related fixes into **separate commits** — do not make one mega-commit "Apply review feedback".

Recommended grouping:
- One commit per **logical category** (e.g., "fix: address type safety issues from review", "refactor: extract duplicated logic per review").
- One commit per **file** if changes are unrelated.
- Never mix unrelated fixes in a single commit.

Conventional Commits format with the task ID:

```bash
git commit -m "fix(T-XXX-007): address Zod validation gap on tarifa_valor (review F1)"
git commit -m "refactor(T-XXX-007): extract format-tarifa helper (review F2)"
```

After each commit (or group), run quality gates:

```bash
pnpm format
pnpm lint
pnpm test
pnpm build
pnpm check
```

If any gate fails, fix before continuing. **Never** `--no-verify`.

---

## Step 10 — Re-run all quality gates

After all approved feedback is applied, run the full suite once more:

```bash
pnpm format
pnpm lint
pnpm test:coverage
pnpm build
pnpm check
```

All must be green. Coverage must still meet targets (80% business / 60% integrations).

---

## Step 11 — Push

```bash
git push origin {branch-name}
```

CI will re-run on GitHub. Wait for it to complete.

---

## Step 12 — Notify the Delta

> "Feedback aplicado en commits {hashes}. CI corriendo. Cuando CI esté verde,
> el PR está listo para que vos mergeés. Si querés un segundo pase del reviewer
> con los cambios, pediselo antes del merge. NO mergeo — vos decidís."

If the Delta wants a second review pass:
- For Copilot: re-request review from `@copilot`.
- For CodeRabbit: comment `@coderabbitai review` on the PR.
- For Qodo: re-trigger per Qodo docs.
- For manual review with Opus: the Delta pastes the new diff to Opus.

---

## Step 13 — STOP

> **You do not merge. The Delta merges.**

Stay available in case:
- A second review pass produces new feedback (loop back to Step 1).
- CI fails on the new commits (debug, fix, push, notify).
- The Delta has follow-up questions.

When the Delta confirms the PR was merged, you may:
- Mark the Asana subtask as completed in your final response.
- Acknowledge the task is done.
- **Wait** for the next explicit task request. Do not auto-start the next backlog item.

---

## ❌ DO NOT

- Apply any feedback automatically without the Delta's confirmation.
- Trust the reviewer's file references without checking `gh pr diff --name-only`.
- Apply a suggestion that violates RULE 0 (any, @ts-ignore, eslint-disable), even if the reviewer is insistent.
- Apply a suggestion that contradicts a closed ADR.
- Mix unrelated fixes in a single commit.
- Skip quality gates because "they passed before".
- Merge the PR — ever.

## ✅ DO

- Fetch and verify the diff before validating feedback.
- Categorize every item: ✅ / ❌ / ⚠️.
- Report with the exact format from Step 7.
- Wait for explicit confirmation.
- Apply in separate, logically-grouped commits.
- Re-run all quality gates after applying.
- Stop at "ready to merge" and let the Delta merge.

---

**End of WORKFLOW_PR_FEEDBACK** — return to `AGENTS.md` if you need any other rule.
