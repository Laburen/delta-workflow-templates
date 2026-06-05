# {PROJECT_NAME} — AGENTS.md

> 🤖 **Purpose:** Canonical reference for AI coding agents working in this repository.
> 📅 **Last Updated:** {YYYY-MM-DD}
> 📚 **Companion docs:** see `Canonical Documents` section below.

---

## ⚠️⚠️⚠️ MANDATORY FIRST STEP FOR EVERY TASK ⚠️⚠️⚠️

**BEFORE DOING ANYTHING ELSE, YOU MUST:**

1. **IDENTIFY** the task type from the user's request:
   - Backlog task (T-XXX-NNN)? → `.claude/workflows/WORKFLOW_TAREA.md`
   - PR feedback / review comments? → `.claude/workflows/WORKFLOW_PR_FEEDBACK.md`
   - Bug in already-merged code? → `.claude/workflows/WORKFLOW_BUGFIX.md`
   - Architectural change / new ADR needed? → Stop. Ask the Delta first.

2. **IMMEDIATELY READ** the corresponding workflow file in full before doing anything else.

3. **FOLLOW THAT WORKFLOW EXACTLY** — do not skip steps, do not improvise, do not paraphrase the workflow into a shorter version "for efficiency."

**❌ NEVER START CODING WITHOUT READING THE WORKFLOW FIRST ❌**

**Trigger phrase examples:**

| User says                                  | Read this workflow                              |
| ------------------------------------------ | ----------------------------------------------- |
| "Iniciar T-XXX-NNN" / "Start T-XXX-NNN"    | `.claude/workflows/WORKFLOW_TAREA.md`           |
| "Desarrolla T-XXX-NNN"                     | `.claude/workflows/WORKFLOW_TAREA.md`           |
| "Tengo feedback del PR" / "PR feedback"    | `.claude/workflows/WORKFLOW_PR_FEEDBACK.md`     |
| "Review del PR de T-XXX-NNN"               | `.claude/workflows/WORKFLOW_PR_FEEDBACK.md`     |
| "Hay un bug en {file/feature}"             | `.claude/workflows/WORKFLOW_BUGFIX.md`          |
| "Hotfix" / "fix urgente en producción"     | `.claude/workflows/WORKFLOW_BUGFIX.md`          |

If the task does not match any trigger above, **ask the Delta** what kind of task it is before proceeding.

---

## 🔥🔥🔥 ABSOLUTE PROHIBITIONS — ZERO TOLERANCE 🔥🔥🔥

These rules apply **everywhere** in the codebase — production code, tests, scripts, configuration. There are **no exceptions**.

### ⛔ RULE 0: NEVER use `any`, `@ts-ignore`, or `eslint-disable`

#### ❌ FORBIDDEN

```typescript
// Forbidden — `any` type
const data: any = response;
function process(item: any) {}
const items: any[] = [];

// Forbidden — escape hatches
// @ts-ignore
// @ts-nocheck
// eslint-disable
// eslint-disable-next-line
/* eslint-disable */
```

#### ✅ REQUIRED ALTERNATIVES

```typescript
// Use specific types
interface ResponseData { id: number; name: string }
const data: ResponseData = response;

// Use `unknown` + Zod / type guards for external input
const data: unknown = await response.json();
const parsed = ResponseSchema.parse(data);

// Use generics
function process<T extends BaseType>(item: T): T { return item; }

// Tests: use precise mock types, never `any`
const mockRepo: Mocked<Partial<PedidoRepository>> = {
  create: vi.fn().mockResolvedValue(mockPedido),
};
```

#### 🚨 CONSEQUENCE OF VIOLATION

If you write `any`, `@ts-ignore`, or `eslint-disable` anywhere:
1. **STOP** immediately.
2. **REWRITE** with proper types.
3. **NEVER** commit code with these violations.

#### Pre-commit grep check (run before every commit)

```bash
# Each must return ZERO results
grep -rn ": any" src/ tests/
grep -rn "@ts-ignore" src/ tests/
grep -rn "eslint-disable" .
```

If results appear: rewrite before committing.

---

### ⛔ RULE 0.1: NEVER use `--no-verify` to skip git hooks

Husky hooks exist for a reason. If `pnpm check` fails, **fix the errors**. Do not bypass.

```bash
# ❌ FORBIDDEN
git commit --no-verify -m "..."

# ✅ REQUIRED
# Read the error. Fix the underlying issue. Then commit normally.
```

---

### ⛔ RULE 0.2: NEVER use `.then()` chains — only async/await

```typescript
// ❌ FORBIDDEN
fetchData().then(d => process(d)).then(r => save(r));

// ✅ REQUIRED
const d = await fetchData();
const r = await process(d);
await save(r);
```

---

### ⛔ RULE 0.3: NEVER accept external input without Zod validation

Any data crossing a trust boundary (HTTP request bodies, external API responses, file contents, environment variables read at runtime) MUST be validated with Zod before use.

```typescript
// ❌ FORBIDDEN
const body = await request.json() as PedidoInput;
await createPedido(body);

// ✅ REQUIRED
const body: unknown = await request.json();
const input = PedidoCreateSchema.parse(body);  // throws ZodError on invalid input
await createPedido(input);
```

---

## 🚨 CRITICAL RULES FOR AI AGENTS

### Rule 1: NEVER develop beyond the requested task

Only develop the specific task the user requested. **NEVER** assume you should continue with the next task, even if:
- The user says "continue"
- You just finished a task and see the next one in the backlog
- The tasks are related or dependent

**Always ask explicitly** before starting a new task.

❌ **INCORRECT:**
```
User: "Develop T-XXX-007"
Agent: [Completes T-XXX-007]
Agent: [Automatically starts T-XXX-008 without asking]
```

✅ **CORRECT:**
```
User: "Develop T-XXX-007"
Agent: [Completes T-XXX-007]
Agent: "T-XXX-007 completed. Do you want me to continue with T-XXX-008?"
```

---

### Rule 2: PRs ALWAYS target `develop`, NEVER `main`

```bash
# ✅ CORRECT
gh pr create --base develop --title "..." --body "..."

# ❌ INCORRECT
gh pr create --base main --title "..." --body "..."
```

**Reason:** Git Flow — `develop` is the integration branch; `main` is for production releases only. The Delta merges to `main` separately as part of release flow.

**Exception:** `hotfix/...` branches PR to `main` (see `WORKFLOW_BUGFIX.md`).

---

### Rule 3: ALWAYS validate PR feedback before applying

When you receive feedback from a PR review (Copilot, CodeRabbit, Qodo, manual review), **NEVER apply changes automatically**.

Follow `.claude/workflows/WORKFLOW_PR_FEEDBACK.md` exactly. Summary:

1. **VALIDATE** which files the feedback mentions.
2. **VERIFY** those files are actually in the PR (`gh pr diff {PR_NUMBER} --name-only`).
3. **CATEGORIZE** each feedback item:
   - ✅ Valid: file in feedback AND in PR
   - ❌ Invalid: file in feedback but NOT in PR
   - ⚠️ Unclear: needs Delta decision
4. **REPORT** to the Delta with the structured format from the workflow.
5. **WAIT** for explicit confirmation before applying.

**IMPORTANT:** AI reviewers (Copilot, CodeRabbit, etc.) can hallucinate, reference wrong files, or contradict project conventions. Always verify first.

---

### Rule 4: MANDATORY quality gates with TODO list tracking

For EVERY task, create a TODO list at the start and check items off **immediately** as you complete them. Default checklist (adapt per workflow):

```
1. Read the task block from docs/BACKLOG-{PROJECT}.md
2. Read relevant ADRs from docs/DECISIONES-ARQUITECTONICAS-PENDIENTES.md
3. Confirm understanding with the Delta if anything is ambiguous
4. Create feature branch from develop
5. Write failing test (Red)
6. Implement minimal code to pass test (Green)
7. Refactor if needed (keep tests green)
8. Run pnpm format
9. Run pnpm lint
10. Run pnpm test (all tests pass)
11. Run pnpm build
12. Run pnpm check (typecheck + lint)
13. Manual E2E test if applicable
14. Update docs/BACKLOG-{PROJECT}.md (mark task ✅ COMPLETADA)
15. Commit with Conventional Commits format
16. Push branch
17. Open PR targeting develop
18. Notify the Delta — DO NOT merge yourself
```

**❌ NEVER:**
- Skip `format` because "lint will fix it"
- Forget to update the backlog before the final commit
- Create commits without updating documentation
- Assume any step "is not important"
- Merge the PR yourself (Rule 5)

**✅ ALWAYS:**
- Follow the exact step order from the workflow
- Use a TODO list for tracking
- Verify ALL steps are ✅ before the final commit
- Update the backlog BEFORE the final commit

---

### Rule 5: NEVER merge the PR yourself

The merge to `develop` (or `main` for hotfixes) is **ALWAYS** done by the Delta, **NEVER** by the agent.

Even if:
- CI is green
- The Delta said "looks good"
- All review comments are resolved

You stop after pushing the branch and opening the PR. You notify the Delta. They merge.

---

### Rule 6: Conventional Commits with task ID in scope

```
feat(T-XXX-007): add createPedido service with Zod validation
test(T-XXX-007): add coverage for duplicate detection window
fix(T-XXX-007): correct date parsing when input format is dd/mm/yyyy
refactor(T-XXX-007): extract formatTarifa to utils
docs(T-XXX-007): document V001 D1 schema in ESQUEMA-DB-D1.md
chore: configure Cron Trigger in wrangler.toml
```

Types allowed: `feat`, `fix`, `test`, `refactor`, `docs`, `chore`, `style`, `perf`, `ci`, `build`.

The Husky `commit-msg` hook validates this with commitlint. If it rejects, rewrite the message — **never** use `--no-verify`.

---

### Rule 7: One task = one branch = one PR

- Branch name format: `feature/T-XXX-NNN-short-slug` (or `fix/...`, `hotfix/...`, `refactor/...`).
- Never mix two tasks in one branch.
- Never reuse a branch from a previous task.
- Always branch from up-to-date `develop`.

---

## 🧠 Context Discipline (anti auto-compact)

Auto-compacting is the symptom of having loaded too much context that you don't need yet. Follow these rules to keep context lean and avoid compaction mid-task:

### Read on demand, not preemptively

- Do NOT load `AGENTS.md` in full at every turn. Read it once at session start; consult specific sections via grep/targeted read when needed.
- Do NOT load the entire `BACKLOG-{PROJECT}.md`. Read only the specific T-XXX-NNN block for the current task.
- Do NOT load every ADR. Read only the ADRs referenced by the current task's "Notas Técnicas".
- Do NOT preload all workflow files. Read only the one the trigger phrase points to.

### Respect task atomicity

A task is atomic when it can be completed end-to-end without context compaction. Signs that a task is **too big**:
- The Delta gave you a task that touches more than 5 files.
- The task description has more than 10 sub-tasks.
- Estimated hours (human base) are more than 4.

**If you detect any of these BEFORE starting**, stop and tell the Delta:

> "This task looks too large to complete atomically. I suggest splitting it into {N} smaller tasks: {your proposal}. Should we update the backlog first?"

### Detect your own context degradation

If during a task you notice ANY of these:
- You repeated the same file read more than 3 times in the session.
- You start contradicting earlier decisions in the same conversation.
- You can't remember which sub-task you're on.
- You start re-importing things you already removed.

**Stop immediately**. Tell the Delta:

> "I'm losing context. I recommend pausing this task, committing what's done in a WIP branch, and continuing in a fresh session."

This is not a failure — it's the system working. Better to stop than to ship broken code.

### Do not paraphrase the workflow

When you read `WORKFLOW_*.md`, follow it **verbatim**. Do not produce a "summary" of the workflow in your output to save tokens. The workflow is the source of truth; your job is to execute it.

---

## 📚 Canonical Documents

Read these in order when onboarding to the project:

| Order | File                                                                 | Purpose                                                |
| ----- | -------------------------------------------------------------------- | ------------------------------------------------------ |
| 1     | `AGENTS.md` (this file)                                              | Behavior rules, stack, architecture                    |
| 2     | `docs/RESUMEN-RELEVAMIENTO-{PROJECT}.md`                             | What the agent does and why (business)                 |
| 3     | `docs/DECISIONES-ARQUITECTONICAS-PENDIENTES.md`                      | ADRs (closed and open)                                 |
| 4     | `docs/BACKLOG-{PROJECT}.md`                                          | Current sprint with HUs and technical tasks            |
| 5     | `.claude/INSTRUCTIONS.md`                                            | Short entry-point for AI agents                        |
| 6     | `.claude/workflows/WORKFLOW_TAREA.md`                                | How to develop a task (read when assigned a task)      |
| 7     | `.claude/workflows/WORKFLOW_PR_FEEDBACK.md`                          | How to process PR feedback                             |
| 8     | `.claude/workflows/WORKFLOW_BUGFIX.md`                               | How to handle bugs in merged code                      |

---

## 🏗️ Project: {PROJECT_NAME}

{1-3 sentence description of what this project does, who uses it, and the business value. Replace with project-specific content.}

**Sprints:**

- **Sprint 1 · {Name}:** {Brief description.}
- **Sprint 2 · {Name}:** {Brief description.}
- **Sprint 3 · {Name}:** {Brief description.}

---

## 🏛️ Architecture

```
{ASCII diagram of system architecture — components, integrations, data flow.
Keep it scannable. Replace with project-specific diagram.}
```

### Macro flow

```
{ASCII flow diagram showing the main business flow — actors, events,
decisions, side effects. Replace with project-specific flow.}
```

### Key architectural decisions (summary — details in `docs/DECISIONES-ARQUITECTONICAS-PENDIENTES.md`)

- **{ADR-1 title}:** {one-line summary of the decision and rationale}. (ADR-1)
- **{ADR-2 title}:** {one-line summary}. (ADR-2)
- **{ADR-N title}:** {one-line summary}. (ADR-N)

---

## 🛠️ Tech Stack

| Technology              | Used for                                                       |
| ----------------------- | -------------------------------------------------------------- |
| Node.js 20 LTS          | Runtime                                                        |
| TypeScript (strict)     | Language for agent logic and services                          |
| pnpm                    | Package manager                                                |
| Cloudflare Workers      | Serverless runtime (business logic)                            |
| Cloudflare D1           | Database (source of truth)                                     |
| Cloudflare KV           | Cache (OAuth tokens, retry queues, idempotency)                |
| Cloudflare R2           | Object storage (when needed)                                   |
| Zod                     | Schema validation (payloads, external responses)               |
| Vitest + miniflare      | Testing framework (simulates Cloudflare bindings)              |
| Wrangler                | Cloudflare CLI for dev/deploy                                  |
| {Other integrations}    | {Project-specific — WhatsApp, Graph API, OCR endpoint, etc.}   |

---

## 📂 Repository Structure

```
{project}/
├── AGENTS.md                       # this file
├── README.md
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json                   # strict + noUncheckedIndexedAccess
├── wrangler.toml                   # bindings + cron triggers
├── vitest.config.ts
├── commitlint.config.js
├── .nvmrc                          # 20
├── .prettierrc
├── eslint.config.js
├── .husky/
│   ├── pre-commit                  # pnpm check (read-only)
│   └── commit-msg                  # commitlint --edit
├── .claude/
│   ├── INSTRUCTIONS.md             # short entry for AI agents
│   └── workflows/
│       ├── WORKFLOW_TAREA.md
│       ├── WORKFLOW_PR_FEEDBACK.md
│       └── WORKFLOW_BUGFIX.md
├── .github/
│   ├── copilot-instructions.md     # mirror of .claude/INSTRUCTIONS.md (optional)
│   └── workflows/
│       └── ci.yml                  # typecheck + lint + test
├── docs/                           # project documentation
├── migrations/
│   └── V001__initial_schema.sql
├── src/
│   ├── index.ts                    # thin entrypoint (dispatch only)
│   ├── env.ts                      # Cloudflare bindings types
│   ├── types.ts                    # shared domain types
│   ├── agent/                      # agent prompt + tool definitions
│   ├── services/                   # business logic
│   ├── schemas/                    # Zod schemas
│   ├── utils/                      # helpers
│   ├── integrations/               # external API wrappers
│   └── cron/                       # cron handlers
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
```

{Adapt the structure section to the actual project layout. Document conventions for how endpoints, services, and handlers are organized, if the project has multiple request origins or specific patterns.}

---

## 📐 Code Style Guidelines

### TypeScript — mandatory strict configuration

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

### Code rules

- **No `any`** — use `unknown` or specific types. See RULE 0.
- **Always `async/await`** — no `.then()` chains. See RULE 0.2.
- **Parallelize independent operations** with `Promise.all` or `Promise.allSettled`.
- **Early returns** — avoid deep nesting.
- **Validate external input with Zod** — see RULE 0.3.
- **Typed errors** — use classes with `code` and `statusCode` fields.
- **Import order:** 1) Node built-ins, 2) External deps, 3) Internal `@/`, 4) Relative, 5) Types.
- **English everywhere** — names, comments, JSDoc, error messages.

### Naming

| Kind                  | Convention            | Example                    |
| --------------------- | --------------------- | -------------------------- |
| Files                 | kebab-case            | `format-tarifa.ts`         |
| Classes               | PascalCase            | `SharepointGraphClient`    |
| Functions / variables | camelCase             | `findCandidatesForPedido`  |
| Constants             | UPPER_SNAKE           | `MAX_RETRIES`              |
| Types / Interfaces    | PascalCase            | `PedidoCreateInput`        |
| Zod schemas           | PascalCase + `Schema` | `PedidoCreateSchema`       |

### Prettier

```json
{ "semi": true, "singleQuote": true, "tabWidth": 2, "trailingComma": "es5", "printWidth": 100 }
```

### ESLint

```js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: { parserOptions: { project: true } },
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/prefer-nullish-coalescing': 'error',
    },
  }
);
```

---

## 🧪 Testing — Vitest + miniflare

This project uses **Test-Driven Development**. For every feature:

1. **Red** — write the failing test first.
2. **Green** — implement the minimal code to pass the test.
3. **Refactor** — improve while keeping tests green.

### TDD rules for this project

- Each backlog task (T-XXX-NNN) is implemented test-first.
- Unit tests for all business logic (`services/`, `schemas/`, `utils/`).
- Integration tests for cross-service paths.
- **Mock all external dependencies**: HTTP APIs, KV, D1, R2. Never hit real APIs in tests.
- Coverage targets: **80%+ in business logic**, **60%+ in integrations**.

### Test file convention

```
tests/
├── unit/
│   ├── services/
│   │   └── pedidos.test.ts
│   ├── schemas/
│   │   └── pedido.test.ts
│   └── utils/
│       └── format-tarifa.test.ts
├── integration/
│   └── flows/
│       └── crear-pedido-end-to-end.test.ts
└── fixtures/
    ├── pedidos.json
    └── usuarios.json
```

### Example unit test

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createPedido } from '@/services/pedidos';

describe('createPedido', () => {
  const mockD1 = { prepare: vi.fn(), batch: vi.fn() };
  const mockEnv = { DB: mockD1 } as unknown as Env;

  beforeEach(() => vi.clearAllMocks());

  it('rejects input with tarifa_modo=fija and missing tarifa_valor', async () => {
    const input = { ...validInput, tarifa_modo: 'fija', tarifa_valor: undefined };
    await expect(createPedido(input, mockEnv)).rejects.toThrow(/tarifa_valor required/);
  });

  it('generates id with PED-YYYYMMDD-NNN format', async () => {
    mockD1.prepare.mockReturnValue({
      bind: vi.fn().mockReturnThis(),
      first: vi.fn().mockResolvedValue({ next_seq: 7 }),
      run: vi.fn().mockResolvedValue({ success: true }),
    });
    const result = await createPedido(validInput, mockEnv);
    expect(result.id).toMatch(/^PED-\d{8}-\d{3}$/);
  });
});
```

---

## 🌳 Git Workflow

> **GOLDEN RULE — no exceptions:**
> **Every backlog task (T-XXX-NNN) is developed in its own feature branch.**
> Never commit task work directly to `develop` or `main`.

### Branch naming

| Prefix      | When to use                       | Example                              |
| ----------- | --------------------------------- | ------------------------------------ |
| `feature/`  | Any T-XXX-NNN backlog task        | `feature/T-XXX-007-creacion-pedido`  |
| `fix/`      | Bug that is NOT a production hotfix | `fix/timeout-graph-token`          |
| `hotfix/`   | Urgent production fix on `main`   | `hotfix/template-wa-rejected`        |
| `refactor/` | Refactoring with no functional change | `refactor/extract-format-tarifa`  |

### Branch tree

```
main                              # Production — merges from develop only
└── develop                       # Integration — base for all features
    ├── feature/T-XXX-001-tooling
    ├── feature/T-XXX-007-creacion-pedido
    ├── fix/timeout-graph-token
    └── hotfix/template-wa-rejected
```

### Full task flow

```bash
git checkout develop && git pull origin develop
git checkout -b feature/T-XXX-NNN-short-name       # 1. branch from develop
# ... TDD: red → green → refactor, commit with Conventional Commits ...
git push -u origin feature/T-XXX-NNN-short-name    # 2. push
# 3. open PR → develop, request AI review, wait for Delta to merge
```

### Pre-commit hook

`pnpm check` (tsc + eslint, read-only) runs before every `git commit` via Husky. If it fails, **fix all errors before committing** — never `--no-verify`. See RULE 0.1.

### Merge

- **Squash and merge** to keep history clean.
- PRs **< 400 lines ideal, < 1000 max**. If your PR exceeds, the task was probably mis-split.
- CI must be green before merging.
- **The Delta merges. Not the agent.** See RULE 5.

---

## 🎯 Project-specific rules

{Fill this section with rules unique to the project. Examples below — replace with project-specific content.}

### Example: IDs format

```typescript
// Pedido ID: PED-{YYYYMMDD}-{NNN}
// Generated by utils/id.ts → generatePedidoId(date, seq)
```

### Example: External validation policy

The agent **does NOT certify authenticity of documents**. Always frame as "document received and read"; final validation is the customer's responsibility (see ADR-9).

### Example: Data source of truth

D1 is the source of truth. Never read from {alternative source} to make decisions. {Alternative} is a synchronized view, not a primary store.

---

## 🐛 Recurring Bugs & Lessons Learned

> 🔴 **READ THIS SECTION BEFORE touching code in any area mentioned below.**
> These are bugs that have happened before. The fixes are documented so they
> don't repeat. If you "improve" code in these areas without understanding
> why it's written this way, you WILL recreate the bug.

<!-- Add bugs and lessons as they are discovered during development.

Template for new entries:

### {YYYY-MM-DD} — {Short title of the bug or trap}

**Where:** {file/module/area}
**Symptom:** {how the bug manifested}
**Root cause:** {what was actually wrong}
**Fix:** {the correct approach, with code example if useful}
**❌ Do NOT:** {what you might be tempted to do that recreates the bug}
**✅ DO:** {the correct pattern}

-->

---

## 🤖 Automated Workflows

### 🔴 WORKFLOW READING IS MANDATORY — NOT OPTIONAL 🔴

**EVERY TIME** you receive a task, you MUST:

1. **STOP** before doing anything.
2. **READ** the appropriate workflow file in full.
3. **FOLLOW** every step in that workflow.
4. **DO NOT** skip ahead or improvise.

### Workflow files

| Workflow                | Path                                              | When to use                                |
| ----------------------- | ------------------------------------------------- | ------------------------------------------ |
| **Task development**    | `.claude/workflows/WORKFLOW_TAREA.md`             | Any T-XXX-NNN backlog task                 |
| **PR feedback**         | `.claude/workflows/WORKFLOW_PR_FEEDBACK.md`       | Processing review comments                 |
| **Bugfix**              | `.claude/workflows/WORKFLOW_BUGFIX.md`            | Bug in merged code (hotfix or fix branch)  |

### ❌ What NOT to do

- Start coding without reading the workflow.
- Create a TODO list without following the workflow's format.
- Jump to implementation without reading context.
- Assume you know what to do based on the task name.
- Skip the workflow because "you've done similar tasks before."
- Paraphrase the workflow into a shorter version "for efficiency."

### ✅ What to do

1. Identify the task type from the user's request.
2. Read the corresponding workflow file COMPLETELY.
3. Follow the workflow steps IN ORDER.
4. Use the workflow's TODO list format.
5. Complete all quality gates mentioned in the workflow.
6. Stop after pushing the PR — let the Delta merge.

**IF YOU FIND YOURSELF CODING WITHOUT HAVING READ THE WORKFLOW, YOU ARE DOING IT WRONG. STOP AND READ IT.**

---

## 🔧 Build & Test Commands

```bash
# Development
pnpm dev                    # Start Worker locally (wrangler dev)
pnpm build                  # Production build

# Testing
pnpm test                   # Run all tests
pnpm test:watch             # Watch mode
pnpm test:coverage          # Coverage report

# Quality gates
pnpm format                 # Prettier auto-format
pnpm lint                   # ESLint (read-only)
pnpm lint:fix               # ESLint with autofix
pnpm check                  # tsc + eslint (used by pre-commit hook)

# Database (D1)
pnpm db:migrate:dev         # Apply migrations locally
pnpm db:migrate:prod        # Apply migrations to production

# Deploy
pnpm deploy                 # wrangler deploy
```

{Adapt to actual project commands.}

---

## 🗄️ Data Model

{Document the database schema, key entities, relationships. Use SQL DDL or
a clear text description. Replace with project-specific content.}

```sql
-- Example structure — replace with actual schema
CREATE TABLE example (
  id           TEXT PRIMARY KEY,
  created_at   TEXT NOT NULL
);
```

---

## 🔌 External Integrations

{One subsection per integration. Document: base URL, auth method, rate limits,
known limitations, secrets needed. Replace with project-specific content.}

### Example: WhatsApp Cloud API (Meta)

- **Base URL:** `https://graph.facebook.com/v19.0/{phone_number_id}/messages`
- **Auth:** Bearer token (`WA_TOKEN` secret)
- **Webhook:** integrated via {builder/handler}
- **Proactive templates:** require Meta approval (24-48h — always plan as critical path)
- **Reactive messages:** within the 24h post-user-message window, free text allowed
- **Rate limit (Tier 1):** 1,000 msgs/day

---

## 🔐 Environment Variables

```toml
# wrangler.toml [vars] — non-secret config
[vars]
EXAMPLE_VAR = "value"

# Secrets (wrangler secret put)
# SECRET_NAME_1
# SECRET_NAME_2
```

{Document each variable: purpose, who provides it, when it must be set.}

---

## 📋 Current Backlog

See [`docs/BACKLOG-{PROJECT}.md`](docs/BACKLOG-{PROJECT}.md) for the full sprint backlog.

**Sprint {N}:** {N HUs · N technical tasks · estimated {N-M}h hybrid}.

{Optionally include a summary table — but the canonical reference is the
BACKLOG.md file. Keep this section as a pointer, not a duplicate.}

---

## 🚫 Non-negotiable rules (summary)

- Branch per task: `feature/T-XXX-NNN-slug` from `develop`. Squash and merge.
- TDD: test first, then implementation. No exceptions.
- No `any`, no `@ts-ignore`, no `eslint-disable`. See RULE 0.
- Zod at boundaries: validate all external input.
- English everywhere: code, comments, error messages.
- Conventional Commits with `T-XXX-NNN` in scope.
- `pnpm check` before every commit. Never `--no-verify`.
- The agent does NOT merge PRs. The Delta does.
- {Add project-specific non-negotiables here.}

---

## 🔍 When in Doubt

1. Re-read the relevant section of this `AGENTS.md`.
2. Read the canonical docs in `docs/` for business and architectural context.
3. Read the current workflow file in `.claude/workflows/`.
4. Check existing code patterns in similar modules.
5. Run `pnpm test` and `pnpm check` to verify before committing.
6. **Ask the Delta** — better one clarifying question than one bad commit.

---

**End of AGENTS.md** — follow these rules for consistent, high-quality, agent-friendly code.
