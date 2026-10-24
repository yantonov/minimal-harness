---
name: minimal-harness
description: Use when a project lacks `AGENTS.md` or a single-command verification (`make check`), or when asked to bootstrap, set up, or add an agent harness. Creates both. Based on the Harness methodology.
---

# Minimal Harness

Creates the two files every agent harness starts from, for a project with a working codebase.

- **`AGENTS.md`**: the router agents read first. A router names where to look; deep detail lives in `docs/*.md` behind pointers.
- **`Makefile`**: `make check` runs lint + types + tests (+ e2e) and returns one green/red verdict.

Done when the cold start test (Step 4) passes.

---

## Step 1: Gather project information

Discover each item from the repo (manifests, configs, imports, existing scripts); ask the user only for what the repo does not show.

- Language and version, framework, database, cache/queue
- Package manager, test framework, type checker, linter
- Exact dev server command
- E2E command, if any
- Task tracker: whether `bl` (backlogger) is available to hold work as tickets
- Hard constraints: auth requirements, SQL rules, security rules, forbidden operations (the user is the only source)

Unknown items become `TODO` placeholders. Done when every item is either filled or listed as a `TODO`.

---

## Step 2: Create `AGENTS.md`

Write it in the repository root, ≤150 lines. It is a router: when it outgrows the limit, move detail to `docs/*.md` and point to it.

- Put critical rules at the top or bottom (recall is weakest mid-file).
- Give every rule its reason, so a later reader can tell when it expires.
- Leave style to the linter: say "run `make lint`" instead of listing style rules.

### Template

```markdown
# AGENTS.md

## Project
<One or two sentences: what this is and why it exists.>
Stack: <language + version>, <framework>, <database>, <cache/queue>.

## Starting a session
Work lives in `bl` tickets: one committed `backlog.md` derived from `.tickets/`, so the tracker stays in the repo and never drifts from the code.
- `bl available` — claimable tickets (open, deps satisfied)
- `bl started` — tickets in progress
- `bl show <id> --json` — one ticket in full
- `bl help` — the full command surface

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
- The ticket is closed with `bl done <id>`.

NOT done: "code is written", "looks right", "unit tests pass", "works on my machine".

## Hard constraints (MUST / MUST NOT)
<Only the project's real constraints, each with its reason. Examples:>
1. MUST: all endpoints use <auth mechanism>.
2. MUST: all SQL is parameterized (injection).
3. MUST NOT: change `migrations/` or `infra/` without explicit human approval.
4. MUST NOT: commit secrets; all values come from environment variables.

## Work rules
- **WIP=1.** Claim one ticket at a time: `bl start <id>`, and close it with `bl done <id>` before claiming another.
- Stay inside the claimed ticket. Record any other problem with `bl add "<title>"` and continue.
- Priority order: correctness → performance → style.

## Escalate to human (stop and ask)
- Changing database schema or migrations.
- New external dependency or service.
- Any decision affecting security or privacy.
- Changing the public API.
- Deleting anything that cannot be recovered.

## Details (read when needed)
- `docs/api-patterns.md`: before adding any endpoint
- `docs/db-rules.md`: before changing models or migrations
- `docs/testing.md`: when writing tests
- `docs/security.md`: when working with auth, payments, or personal data
```

Keep only `Details` entries whose file exists or that you create now. Unknown commands become `TODO: fill in <command>` lines, resolved with the user before finishing. If the project does not track work with `bl`, point `Starting a session` at the tracker it actually uses instead.

Done when every template section is filled or marked `TODO`, and every file it names exists.

---

## Step 3: Create `Makefile`

Write it in the repository root. Every target runs the real command; a target that only echoes is a lie.

### Grouping targets

Split targets into two tiers:

- **Public** — the targets a casual user or contributor needs day-to-day.
  Minimum: `check`. Typically also: `dev`, `build`, `test`, `run`, `install`.
- **Internal** — individual steps that a public target already calls (`lint`, `types`, `e2e`),
  plus maintenance tools (`outdated`, `setup`). They exist and are callable,
  but are hidden from the default help.

`make help` shows only public targets, plus a pointer: `Run 'make help-all' for the full list.`
`make help-all` shows both groups, clearly separated.

This keeps `make` friendly for first-time users while preserving full discoverability.

### Template

```makefile
.PHONY: help help-all setup dev test types lint check
<if E2E exists:.PHONY: e2e</if>

.DEFAULT_GOAL := help

help:
<TAB>@echo "Usage: make [target]"
<TAB>@echo ""
<TAB>@echo "  check   — run lint + types + test<if E2E exists:> + e2e</if>"
<TAB>@echo "  dev     — start dev server"
<TAB>@echo "  test    — run tests"
<TAB>@echo ""
<TAB>@echo "Run 'make help-all' for the full list, including internal targets."

help-all:
<TAB>@echo "Public targets:"
<TAB>@echo "  check   — run lint + types + test<if E2E exists:> + e2e</if>"
<TAB>@echo "  dev     — start dev server"
<TAB>@echo "  test    — run tests"
<TAB>@echo ""
<TAB>@echo "Internal targets (typically called via check):"
<TAB>@echo "  lint    — run linter"
<TAB>@echo "  types   — run type checker"<if E2E exists:>
<TAB>@echo "  e2e     — run end-to-end tests"</if>
<TAB>@echo "  setup   — <package manager> install"

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

Unknown command: `<TAB># TODO: fill in test command` plus `<TAB>@exit 1`, so `make check` stays red until it is filled.

Done when `make check` has run, and every red result is either fixed or traced to a `TODO`.

---

## Step 4: Cold start test

Reread only the two files and answer from them:

1. What is this system and why does it exist? (`Project`)
2. How do I run it? (`Commands`)
3. How do I verify it? (`make check`)
4. What rules must never be broken? (`Hard constraints`)
5. Where is the current work? (`Starting a session` → `bl started`)

Each unanswered question is a gap the agent will fill by guessing, and writing the answer once costs less. Done when all five are answered; report remaining `TODO`s to the user.
