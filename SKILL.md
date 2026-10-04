---
name: minimal-harness
description: Bootstraps a minimal agent harness for a project — `AGENTS.md` (the router/entry point for AI agents) and `Makefile` (single-command verification). Based on the Harness methodology.
---

# Minimal Harness

Bootstraps a minimal agent harness for a project: `AGENTS.md` (the router/entry point for AI agents) and `Makefile` (single-command verification). Based on the [Harness methodology](https://github.com/justxor/Harness_ru).

## What this skill does

Given a project with a working codebase, this skill creates two files that form the foundation of an agent harness:

1. **`AGENTS.md`** — the standard entry point that tells any AI agent: what this project is, how to run it, how to verify it, what rules must never be broken, and where to find deeper docs.
2. **`Makefile`** — a single `make check` command that runs all verification (lint + types + tests + e2e) and returns a green/red verdict.

After running this skill, the agent should pass the **cold start test**: a fresh session with only the repository contents can answer all five questions (what the system is, how to run it, how to verify it, what rules can't be broken, and where the current work is).

---

## Prerequisites

- An existing project directory with a working codebase under `src/` or similar.
- Known tech stack (language, framework, database, cache/queue).
- Known package manager and test framework.
- A `docs/` directory is optional but recommended.

---

## Step 1: Gather project information

Before creating any files, ask the user for or discover from the codebase:

| Question | How to find it |
|---|---|
| **Language & version** | `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, etc. |
| **Framework** | imports in `src/`, config files |
| **Database** | connection strings, ORM config |
| **Cache / queue** (optional) | Redis, RabbitMQ, etc. |
| **Package manager** | `uv`, `poetry`, `npm`, `pnpm`, `yarn`, `go`, `cargo` |
| **Test framework** | `pytest`, `jest`, `vitest`, `go test`, etc. |
| **Type checker** | `mypy`, `tsc`, `pyright`, or "none" |
| **Linter** | `ruff`, `eslint`, `biome`, `golangci-lint`, etc. |
| **Dev server command** | the exact shell command to start the app locally |
| **E2E test command** (if any) | command for integration/end-to-end tests |
| **Hard constraints** | does the project have auth requirements, SQL rules, security rules, forbidden operations? |

If the user doesn't know an answer, leave a `TODO` placeholder in the generated file and note it.

---

## Step 2: Create `AGENTS.md`

Create `AGENTS.md` in the repository root. It must follow these rules:

- **50–150 lines maximum.** If you find yourself writing more, move details to `docs/*.md` files and link to them from `AGENTS.md`.
- **Critical rules go at the top or bottom, never in the middle** (models have worse recall from the middle of long texts).
- **Every rule has a source and an expiration condition** (or at minimum: why it exists).
- **The file is a router, not an encyclopedia.** It tells the agent *where to look*, not everything there is to know.

### Template

```markdown
# AGENTS.md

## Project
<One or two sentences: what this is and why it exists.>
Stack: <language + version>, <framework>, <database>, <cache/queue>.

## Commands
- Setup: `<package manager install command>`
- Dev server: `<dev server command>`
- Tests: `<test command>`
- Types: `<type check command>`
- Lint: `<lint command>`
<if E2E exists:>- E2E: `<e2e command>`</if>
- **Full verification: `make check`**

## Definition of Done
A feature is done only when:
- `make check` is green (including tests that existed before this session).
- The feature's specific verification passes.
- `PROGRESS.md` is updated.

NOT done: "code is written", "looks right", "unit tests pass", "works on my machine".

## Hard constraints (MUST / MUST NOT)
<Fill in based on the project. Examples:>
1. MUST: all endpoints use <auth mechanism>.
2. MUST: all SQL is parameterized. Never concatenate strings into queries.
3. MUST: external data is validated at the domain boundary.
4. MUST NOT: `eval`, `exec`, or deserialize untrusted data.
5. MUST NOT: change `migrations/` or `infra/` without explicit human approval.
6. MUST NOT: commit secrets; all values come from environment variables.

## Work rules
- **WIP=1.** Exactly one feature in `active` state at a time.
- No refactoring, renaming, or reorganizing outside the active feature.
  If you notice a problem, write it to `BACKLOG.md` and continue.
- Priority order: correctness → performance → style. Do not reverse.

## Escalate to human (stop and ask)
- Changing database schema or migrations.
- New external dependency or service.
- Any decision affecting security or privacy.
- Changing the public API.
- Deleting anything that cannot be recovered.

## Details (read when needed)
- `docs/api-patterns.md` — required before adding any endpoint
- `docs/db-rules.md` — required before changing models or migrations
- `docs/testing.md` — when writing tests
- `docs/security.md` — when working with auth, payments, or personal data
```

### Placeholder handling

If a section has no known answer, use:
```markdown
## Commands
- Setup: `TODO: fill in package manager install command`
- Dev server: `TODO: fill in dev server command`
- Tests: `TODO: fill in test command`
```

Replace `TODO` items by asking the user before finishing.

---

## Step 3: Create `Makefile`

Create a `Makefile` in the repository root. The goal is **one command, one green/red answer** for full verification.

### Template

```makefile
.PHONY: setup dev test types lint check
<if E2E exists:.PHONY: e2e</if>

setup:
<TAB><package manager install command>

dev:
<TAB><dev server command>

test:
<TAB><test command>

types:
<TAB><type check command>

lint:
<TAB><lint command>
<if E2E exists:>
e2e:
<TAB><e2e test command>
</if>
check: lint types test <if E2E exists:e2e</if>
<TAB>@echo "ALL CHECKS PASSED"
```

### Fill in based on stack

| Stack | setup | test | types | lint |
|---|---|---|---|---|
| Python (uv) | `uv sync` | `uv run pytest -x -q` | `uv run mypy src --strict` | `uv run ruff check src` |
| Python (poetry) | `poetry install` | `poetry run pytest -x -q` | `poetry run mypy src --strict` | `poetry run ruff check src` |
| Node (npm) | `npm ci` | `npm test` | `npx tsc --noEmit` | `npx eslint src` |
| Node (pnpm) | `pnpm install --frozen-lockfile` | `pnpm test` | `pnpm tsc --noEmit` | `pnpm lint` |
| Go | `go mod download` | `go test ./...` | `go vet ./...` | `golangci-lint run` |
| Rust | `cargo build` | `cargo test` | `cargo check` | `cargo clippy` |

### If a command is unknown

Leave a comment placeholder and ask the user:
```makefile
test:
<TAB># TODO: fill in test command
<TAB>@echo "TODO: add test command"
```

---

## Step 4: Verify with cold start test

After creating both files, run this self-check mentally:

1. **What is this system and why does it exist?** — `AGENTS.md` > Project section answers this.
2. **How do I run it?** — `AGENTS.md` > Commands section answers this.
3. **How do I verify it?** — `make check` exists and is listed.
4. **What rules must never be broken?** — `AGENTS.md` > Hard constraints section.
5. **Where is the current work?** — `AGENTS.md` > Starting a session points to `PROGRESS.md`.

Every unanswered question is a gap the agent will guess. Guessing is always more expensive than writing the answer once.

---

## Anti-patterns to avoid

1. **Instruction bloat.** `AGENTS.md` over 150 lines. If it grows, split into `docs/*.md` and keep `AGENTS.md` as a router.
2. **Vague commands.** `make test` that doesn't actually run tests. Every command must actually do what it says.
3. **Style rules in AGENTS.md.** Style is enforced by the linter. `AGENTS.md` should say "run `make lint`", not list 20 style rules.
4. **Missing condition for escalation.** Without explicit "stop and ask" triggers, the agent will guess on dangerous changes.
5. **No Definition of Done.** "Code is written" is not done. Define what "done" means in verifiable terms.
