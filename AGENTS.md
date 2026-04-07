# AGENTS.md

> GAN-inspired three-agent harness, integrated with spec-kit Spec-Driven Development (SDD).
> Supported by: Claude Code, Cursor, GitHub Copilot, Codex, Aider, Windsurf, Zed, RooCode.

---

## Workflow Overview

Every feature follows an unbreakable four-phase sequence modeled on spec-kit SDD:

```
PLAN ──▶ TASK ──▶ IMPLEMENT ──▶ CHECK
  │         │          │           │
spec.md  tasks.md   git commit  eval-result.md
plan.md            progress.txt
```

**The gate rule**: no phase may begin until the previous phase's artifact exists and has been reviewed. Skipping phases is prohibited.

---

## Phase 1 — PLAN (`/speckit.specify` + `/speckit.plan`)

**Owner**: Planner Agent  
**Triggered by**: any new feature request or user prompt  
**Runs**: once per feature branch

### Steps

1. Run `uvx --from git+https://github.com/github/spec-kit.git specify init` (first time only)
2. Execute `/speckit.specify` with user prompt → produces `specs/[NNN-feature]/spec.md`
3. Review `spec.md`; do not proceed until it reflects the correct "what" and "why"
4. Execute `/speckit.plan` → produces `specs/[NNN-feature]/plan.md`, `research.md`, `data-model.md`, `contracts/`
5. **STOP**: report branch name, artifact paths. Do not begin tasks yet.
6. Write high-level summary to `claude-progress.txt`

### Output artifacts

| File | Content |
|------|---------|
| `specs/[NNN-feature]/spec.md` | User journeys, acceptance criteria, success definition |
| `specs/[NNN-feature]/plan.md` | Technical implementation plan, architecture decisions |
| `specs/[NNN-feature]/research.md` | Context gathered during planning |
| `specs/[NNN-feature]/data-model.md` | Entity definitions and relationships |
| `specs/[NNN-feature]/contracts/` | API contracts and interface definitions |

### Constraints

- Focus on **what** and **why** — never specify implementation detail in the spec
- Stay high-level in plan.md; let Generator figure out the code path
- If spec is unclear, ask clarifying questions before writing plan.md
- Embed a visual design language section in plan.md for any frontend work
- Read `memory/constitution.md` (if present) before generating spec

---

## Phase 2 — TASK (`/speckit.tasks`)

**Owner**: Planner Agent (handoff from Plan phase)  
**Triggered by**: plan.md confirmed and reviewed  
**Runs**: once per feature, after PLAN phase is complete

### Steps

1. Execute `/speckit.tasks` → reads `plan.md`, `contracts/`, `data-model.md`
2. Produces `specs/[NNN-feature]/tasks.md` with parallelization annotations
3. **STOP**: present `tasks.md` for human review before implementation begins
4. Each task in `tasks.md` must have:
   - Clear acceptance criterion
   - Estimated scope (S/M/L)
   - `[P]` marker if parallelizable
   - Explicit dependency list

### Constraints

- Tasks derive from contracts and entities — not from implementation intuition
- Mark independent tasks `[P]` so Generator can parallelize safely
- Task granularity: each task should be completable in one focused sprint
- Do not begin implementation until tasks.md is human-approved

---

## Phase 3 — IMPLEMENT

**Owner**: Generator Agent  
**Triggered by**: `tasks.md` approved, sprint contract negotiated with Evaluator  
**Runs**: once per task/sprint

### Session startup ritual (mandatory every session)

```
1. cat claude-progress.txt
2. git log --oneline -10
3. cat specs/[NNN-feature]/tasks.md | grep -v "✅"
4. bash init.sh
5. Run smoke test (one core user action, verify response)
6. Only then begin sprint
```

### Sprint workflow

```
Generator proposes sprint-contract.md
    │
    ▼
Evaluator reviews contract ──▶ (negotiate until agreed)
    │
    ▼
Generator implements tasks against contract
    │
    ▼
Generator self-evaluates all contract criteria
    │
    ▼
Generator commits: git commit -m "feat([NNN]): <description>"
    │
    ▼
Generator updates tasks.md (mark completed ✅) and claude-progress.txt
    │
    ▼
Hand off to CHECK phase
```

### Sprint contract format (`sprint-contract.md`)

```markdown
## Sprint: <N> — <title>
### Tasks from tasks.md
- [ ] Task ID: <id>, Criterion: <acceptance criterion>

### Success criteria (testable)
- <specific, browser-verifiable behavior>

### Evaluator test steps
1. Navigate to <URL>
2. Perform <action>
3. Verify <expected state>
```

### Constraints

- One sprint = one coherent feature block from tasks.md
- Never skip sprint contract negotiation
- Never mark a task ✅ without Evaluator sign-off
- Never remove or edit existing tests
- Use `git revert` (not patches) to recover from broken state
- Leave codebase in clean state after every commit: tests pass, no uncommitted changes

---

## Phase 4 — CHECK

**Owner**: Evaluator Agent  
**Triggered by**: Generator sprint complete  
**Runs**: after every sprint; also runs `/speckit.analyze` for post-implementation review

### Evaluation process

```
1. Read sprint-contract.md
2. bash init.sh — ensure server is running
3. Navigate live application as a real user (Playwright MCP)
4. Screenshot key states as evidence
5. Score each rubric dimension
6. Write eval-result-{N}.md
7a. ALL thresholds pass → "SPRINT PASS"; mark tasks ✅; notify Generator
7b. ANY threshold fails → "SPRINT FAIL"; provide critique; return to Generator
```

### Evaluation rubric

| Dimension | Weight | Threshold | Passes when |
|-----------|--------|-----------|-------------|
| **Design quality** | 30% | ≥ 7/10 | Coherent visual identity; intentional mood |
| **Originality** | 30% | ≥ 6/10 | Custom decisions; no AI slop (purple gradients, white cards) |
| **Craft** | 20% | ≥ 7/10 | Typography, spacing, contrast technically correct |
| **Functionality** | 20% | ≥ 8/10 | All contract criteria verified via live browser |

**Functionality < 8 always fails the sprint regardless of other scores.**

### Post-implementation gate (`/speckit.analyze`)

After all tasks in `tasks.md` are ✅, run `/speckit.analyze` for the full feature:
- Fixes small issues in-place (scout rule)
- Creates new tasks in tasks.md for medium issues
- Generates analysis artifacts for large issues requiring re-planning

---

## Persistent Artifacts (spec-kit + harness)

| File | Owner | Purpose |
|------|-------|---------|
| `memory/constitution.md` | Team | Non-negotiable project principles |
| `specs/[NNN]/spec.md` | Planner | What and why — source of truth |
| `specs/[NNN]/plan.md` | Planner | How — technical decisions |
| `specs/[NNN]/tasks.md` | Planner | Task list with ✅ completion state |
| `specs/[NNN]/contracts/` | Planner | API contracts and interface specs |
| `sprint-contract.md` | Generator+Evaluator | Current sprint definition of done |
| `claude-progress.txt` | Generator | Cross-session handoff log |
| `eval-result-{N}.md` | Evaluator | Per-sprint critique and scores |
| `init.sh` | Planner | Reproducible dev server startup |
| `git history` | Generator | State recovery and audit trail |

---

## Phase Gate Summary

```
User prompt
    │
    ▼
[PLAN] /speckit.specify → spec.md
    │   /speckit.plan   → plan.md + contracts/
    │   STOP: human review
    ▼
[TASK] /speckit.tasks   → tasks.md
    │   STOP: human approval
    ▼
[IMPLEMENT] sprint-contract negotiation
    │         Generator builds + commits
    │         ↓ (one task at a time)
    ▼
[CHECK] Evaluator via Playwright MCP
    │   pass → next task
    │   fail → Generator revises
    ▼
All tasks ✅ → /speckit.analyze → feature complete
```

---

## Hard Rules

- **Never skip a phase gate** — each phase artifact must exist before the next phase starts
- **Never implement before tasks.md is approved**
- **Never mark a task complete without Evaluator live-browser verification**
- **Never remove or modify existing tests**
- **Never commit with failing tests or uncommitted changes**
- **Never praise your own work** — self-evaluation is always biased; that is the Evaluator's job

---

## Tech Stack

```
Frontend  : React + Vite (TypeScript)
Backend   : FastAPI + SQLite (dev) / PostgreSQL (prod)
Testing   : Playwright MCP (E2E), pytest (unit)
Spec tool : spec-kit (uvx, no npm runtime required)
VCS       : Git — one clean commit per sprint
```

---

## Build & Test Commands

```bash
# Bootstrap spec-kit (first time)
uvx --from git+https://github.com/github/spec-kit.git specify init

# Dev server
bash init.sh

# Tests
pytest -q
npx playwright test

# Session orientation
cat claude-progress.txt && git log --oneline -10
```

---

## Prohibitions

- Never commit secrets or credentials
- Never use inline styles in React components
- Never bypass sprint contract negotiation
- Never auto-approve an Evaluator result containing unresolved failures
- Never use `--force` push
- Never edit `spec.md` or `plan.md` during implementation without re-running the affected phase
