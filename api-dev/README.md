# api-dev

Claude Code plugin for the **core-web-api** team at Kodim. Acts as a **Senior Rails Developer**: takes a Notion task, generates a detailed `.md` implementation plan, and then can execute it by writing specs, code, and rubocop-clean output in TDD mode.

**Flows:**

### End-to-end (recommended for new tasks)
```
/api-dev:ship <notion-url>      → [Opus → Sonnet → Sonnet]
                                   plan → ⏸️ approve → execute → ⏸️ approve → commit → PR → signoff
```

Two human checkpoints (plan approval, commit approval). Everything else is automated.

### Individual skills (for targeted use cases)
```
/api-dev:plan <notion-url>                    → [Opus]   generates plan-<TICKET>-<slug>.md
/api-dev:execute plan-<TICKET>-<slug>.md      → [Sonnet] writes specs (RED) → implements → rubocop → undercover → report
/api-dev:pr-signoff [<notion-url>]            → [Sonnet] squash → bin/ci → undercover → update Notion task
```

A human reviews everything before code is committed. **Skills never commit, push, or open PRs on their own — only `api-dev:ship` does, and only after the human approves at checkpoint 2.**

### Model choice (enforced by the skills)

- **`api-dev:ship`** runs on **Opus** — orchestrator with reasoning for checkpoints.
- **`api-dev:plan`** runs on **Opus** — complex reasoning: classifying the module, designing specs with exact values, anchoring analogues with `path:line`, detecting subtle ENV vars.
- **`api-dev:execute`** runs on **Sonnet** — more mechanical: follow the plan, write RED specs, implement until GREEN, run rubocop.
- **`api-dev:pr-signoff`** runs on **Sonnet** — deterministic git/CI flow.

All skills declare `model:` in their frontmatter, so they run on the right model regardless of your session's active model. Internal skill invocations from `api-dev:ship` each respect their own `model:`.

---

## Automatic hooks

The plugin registers two PostToolUse hooks in `settings.json` that fire automatically while you work in core-web-api (requires `jq` installed):

1. **Migration reminder** — when you Edit/Write a file under `db/migrate/*.rb`, prints a reminder to run `annotaterb` and commit the annotated models.
2. **Doppler warning** — when you run `bundle exec rspec` or `bin/rails db:*`/`runner` without the `doppler run` prefix, prints a warning that ENV vars may be missing.

These are **non-blocking** (stderr warnings only). They exist to cut down on two common mistakes.

---

## What makes this plugin specific to core-web-api

This plugin understands the unique patterns of **core-web-api**:

- **Multi-tenancy by subdomain** via `acts_as_tenant`
- **Recorders** — the repo's CRUD+event pattern (one operation, one event)
- **Services** (Query / Action / LLM Tool) — with confirmation flow for LLM tools
- **3 API audiences** — `admin`, `customer`, `kathy` (no JWT for kathy)
- **Blueprinter + Ransack + Pagy** as the standard serialization/search/pagination stack
- **Discard** for soft deletes (never `destroy!`)
- **Doppler + Kamal** deployment — the plan statically verifies ENV vars against `config/deploy.yml` and `.kamal/secrets`
- **Spec-driven testing** with hard rules:
  - Only model specs touch the DB — everything else uses `instance_double`
  - `undercover --compare origin/main` is a blocking coverage gate

See `context/*.md` for the full rules embedded in the plugin.

---

## Requirements

- [Claude Code](https://claude.ai/code) installed
- Notion MCP configured (the plugin uses it to read tasks)
- `doppler` CLI installed and authenticated (every test command runs with `doppler run --config test --`)
- Optional but recommended: `undercover` gem installed in core-web-api for diff coverage checks

---

## Installation

### 1. Register the marketplace

From inside Claude Code, run:

```
/plugin marketplace add KodimTech/claude-code-plugins
```

This registers the `kodim-api` marketplace in your local config. Only needed once.

### 2. Install the plugin

```
claude plugin install api-dev@kodim-api
```

### 3. Enable it

```
claude plugin enable api-dev@kodim-api
```

### 4. Reload

```
/reload-plugins
```

You should see `api-dev` listed in the plugins count.

---

## Usage

### Generate a plan

```
/api-dev:plan https://www.notion.so/workspace/task-title-abc123
```

Or with the raw page ID:

```
/api-dev:plan abc123...
```

The planner:
1. Fetches task details from Notion (title, description, acceptance criteria, ticket, ENV vars mentioned)
2. Loads the 7 context files (architecture, recorders, services, controllers, testing, database, rubocop)
3. Explores the actual core-web-api codebase to anchor the plan in real analogues
4. Shows a visible analysis before writing
5. Statically verifies ENV vars against `config/deploy.yml` and `.kamal/secrets`
6. Writes `plan-<TICKET>-<slug>.md` in your current directory
7. Stops and waits — no branch, no files, no commits

### Execute a plan

```
/api-dev:execute plan-KODIM-456-add-item-variants.md
```

The executor:
1. Reads the plan (if any section is vague, it stops and asks)
2. Loads the 7 context files
3. Reads every analogue and modified file listed in the plan
4. Shows an execution plan before touching anything
5. Creates the branch from `main`
6. Runs the migration if applicable (with `annotaterb`)
7. Writes every spec from the Spec Skeleton first — **RED phase** (all must fail)
8. Implements the plan step by step, running specs after each step
9. Max 3 attempts per failing spec, then stops and asks
10. Runs full test suite + `undercover --compare origin/main` (blocking gate)
11. Runs `rubocop -A` then manual fixes until clean
12. Verifies new ENV vars in `config/deploy.yml` + `.kamal/secrets`
13. Produces a final report with exact commit/push/PR commands for the human

---

## Update

```
claude plugin update api-dev@kodim-api --scope local
```

If it errors with scope issues:

```
claude plugin uninstall api-dev@kodim-api
claude plugin install api-dev@kodim-api
```

---

## Structure

```
api-dev/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── senior-rails-dev.md    # Senior Rails dev persona + rules (Opus)
├── context/
│   ├── architecture.md        # Multi-tenancy, modules, audiences, routes
│   ├── recorders.md           # Recorder pattern (1 op + 1 event)
│   ├── services.md            # Query / Action / LLM Tool patterns
│   ├── controllers.md         # Thin controllers, Blueprinter, Ransack, Pagy
│   ├── testing.md             # RSpec + shared contexts + undercover
│   ├── database.md            # acts_as_tenant, discard, strong_migrations
│   └── rubocop.md             # Rails Omakase
├── skills/
│   ├── ship/
│   │   └── SKILL.md           # End-to-end orchestrator (plan → execute → commit → signoff)
│   ├── plan/
│   │   └── SKILL.md           # Planner workflow
│   ├── execute/
│   │   └── SKILL.md           # Executor workflow (TDD)
│   └── pr-signoff/
│       └── SKILL.md           # bin/ci + undercover + Notion status update
├── settings.json              # Activates the senior-rails-dev agent + hooks (annotaterb, doppler)
└── README.md
```

---

## Hard rules the plugin enforces

- **Spec-driven**: specs written FIRST, must fail (RED), then implementation until GREEN.
- **Only model specs touch the DB** — recorders, services, controllers, jobs, mailers, LLM tools use `instance_double`.
- **Diff coverage with undercover** — blocking. No success declared with uncovered changed lines.
- **Multi-tenant isolation** test is mandatory on components touching scoped models.
- **ENV vars verified statically** against Doppler deploy config — a silent deploy bug we won't ship.
- **3-failure rule** on specs — stop and ask the human, never loop blindly.
- **Never commits, pushes, or opens PRs** — the human owns the Git surface.
