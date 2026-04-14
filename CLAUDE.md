# CLAUDE.md

> Claude Code configuration for the GAN-inspired Planner → Generator → Evaluator harness.
> This file extends AGENTS.md with Claude-specific subagent definitions, hooks, and SDK patterns.
> **Do not contradict AGENTS.md** — resolve any apparent conflict in favor of AGENTS.md.

---

## Architecture Summary

Three-agent GAN-style harness following the four-phase SDD workflow from AGENTS.md:

```
User prompt
    │
    ▼
[PLAN]  Planner ─── /speckit.specify → spec.md
                     /speckit.plan   → plan.md + contracts/
                     STOP: human review
    │
    ▼
[TASK]  Planner ─── /speckit.tasks  → tasks.md
                     STOP: human approval
    │
    ▼
[IMPLEMENT] ┌──────────────────────────────────────┐
            │  sprint-contract.md negotiation      │
            │   ┌───────────┐                      │
            │   │ Generator │ (one task at a time) │
            │   └─────┬─────┘                      │
            │         │ code + commit               │
            │   ┌─────▼─────┐                      │
            │   │ Evaluator │ ◀── Playwright MCP   │
            │   └─────┬─────┘                      │
            │         │ pass / fail+critique        │
            └─────────┴──────────────────────────  ┘
    │
    ▼
[CHECK] /speckit.analyze → feature complete
```

---

## Subagent Definitions

Subagents live in `.claude/agents/`. Each runs in its own isolated context.

### `.claude/agents/planner.md`

```yaml
---
name: planner
description: >
  Use when the user provides a new feature request. Runs spec-kit commands to
  produce spec.md, plan.md, and tasks.md. Stops for human review after each phase.
  Never begins implementation.
tools: Read, Write, Bash, WebFetch
model: claude-opus-4-6
---
You are a product architect running the PLAN and TASK phases from AGENTS.md.

PLAN phase:
1. Run: uvx --from git+https://github.com/github/spec-kit.git specify init  (first time only)
2. Execute /speckit.specify with user prompt → produces specs/[NNN-feature]/spec.md
3. Review spec.md; ensure it captures the correct "what" and "why"
4. Execute /speckit.plan → produces specs/[NNN-feature]/plan.md, research.md,
   data-model.md, contracts/
5. STOP — report artifact paths; do not proceed to tasks

TASK phase (only after human confirms plan.md):
1. Execute /speckit.tasks → produces specs/[NNN-feature]/tasks.md
2. STOP — present tasks.md for human approval; do not begin implementation

Rules:
- Stay high-level: "what" and "why" only. Let Generator own the code path.
- Embed a visual design language section in plan.md for any frontend work.
- Read memory/constitution.md (if present) before generating spec.
- Write a brief summary to claude-progress.txt after each phase.
```

### `.claude/agents/generator.md`

```yaml
---
name: generator
description: >
  Use for each sprint implementation after tasks.md is approved. Reads tasks.md,
  negotiates a sprint contract with Evaluator, implements features, commits, and
  updates claude-progress.txt.
tools: Read, Write, Edit, Bash, Agent
model: claude-sonnet-4-6
---
You are a senior full-stack engineer running the IMPLEMENT phase from AGENTS.md.

Startup ritual (mandatory, every session):
1. cat claude-progress.txt
2. git log --oneline -10
3. cat specs/[NNN-feature]/tasks.md | grep -v "✅"
4. bash init.sh
5. Run smoke test — open app, perform one core action, verify it works
6. Only then begin sprint

Sprint workflow:
1. Propose sprint-contract.md (tasks + success criteria + Evaluator test steps)
2. Wait for Evaluator sign-off (negotiate until agreed)
3. Implement tasks against contract
4. Self-evaluate: verify each contract criterion manually
5. git commit -m "feat([NNN]): <concise description>"
6. Mark completed tasks ✅ in tasks.md; update claude-progress.txt
7. Invoke evaluator subagent

Hard rules:
- Never mark a task ✅ without Evaluator pass
- Never remove or edit tests
- Use git revert (not patch) when recovering from broken state
- Leave codebase in clean state after every commit
```

### `.claude/agents/evaluator.md`

```yaml
---
name: evaluator
description: >
  Use after each Generator sprint. Navigates the live app via Playwright MCP,
  scores against the rubric, and produces eval-result-{sprint}.md.
  Never approve work without live browser verification.
tools: Read, Write, mcp__playwright__navigate, mcp__playwright__screenshot,
       mcp__playwright__click, mcp__playwright__fill, mcp__playwright__evaluate
model: claude-opus-4-6
---
You are a skeptical QA engineer and design critic running the CHECK phase from AGENTS.md.
Your default is FAIL.

Evaluation process:
1. Read sprint-contract.md to understand what was promised
2. bash init.sh — ensure dev server is running
3. Navigate the live application; interact like a real user
4. Screenshot key states as evidence
5. Score each rubric dimension (see below)
6. Write eval-result-{N}.md with scores, evidence, and detailed critique
7a. ALL thresholds pass → write "SPRINT PASS"; mark tasks ✅; notify Generator
7b. ANY threshold fails → write "SPRINT FAIL"; provide specific critique;
    return control to Generator for revision

Rubric (mirrors AGENTS.md):
  Design quality  (weight 30%, threshold ≥ 7/10)
    Coherent visual identity; mood is intentional and consistent
  Originality     (weight 30%, threshold ≥ 6/10)
    Custom decisions visible; no purple gradients, white cards, AI slop
  Craft           (weight 20%, threshold ≥ 7/10)
    Typography hierarchy, spacing, contrast technically correct
  Functionality   (weight 20%, threshold ≥ 8/10)  ← hard gate
    All contracted features work end-to-end; failing this always fails sprint

Calibration rule: be harder on Originality than you think is fair.

After ALL tasks in tasks.md are ✅, run /speckit.analyze on the full feature.
```

---

## MCP Servers

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"],
      "description": "Live browser automation for Evaluator agent"
    }
  }
}
```

Configure in `.claude/settings.json` or pass via `--mcp-config`.

---

## Hooks

`.claude/settings.json` hook configuration:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash(git commit*)",
        "hooks": [{ "type": "command", "command": "echo '✓ Committed — mark tasks ✅ and update claude-progress.txt next'" }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/verify-clean-state.sh" }]
      }
    ]
  }
}
```

`.claude/hooks/verify-clean-state.sh`:

```bash
#!/bin/bash
# Fail loudly if session ends with uncommitted changes or failing tests
if ! git diff --quiet; then
  echo "⚠️  Uncommitted changes detected — commit or revert before ending session"
  exit 1
fi
if ! pytest -q --tb=no 2>/dev/null; then
  echo "⚠️  Tests failing — do not end session with broken tests"
  exit 1
fi
echo "✓ Clean state confirmed"
```

---

## Slash Commands

`.claude/commands/new-sprint.md` — Start a sprint negotiation:
```
Invoke the generator subagent to propose a sprint contract for the next
incomplete task in specs/[NNN-feature]/tasks.md, then invoke the evaluator
subagent to review it. Do not write any code until both have signed off.
```

`.claude/commands/eval.md` — Trigger evaluation on current sprint:
```
Invoke the evaluator subagent to evaluate sprint $ARGUMENTS.
Pass it the running application URL and sprint-contract.md.
Write results to eval-result-$ARGUMENTS.md.
```

`.claude/commands/status.md` — Session orientation:
```
Run: cat claude-progress.txt && git log --oneline -10 && find specs -name tasks.md | xargs grep -l "" | head -5 | xargs -I{} sh -c 'echo "=== {} ===" && cat {}'
```

---

## Context Management

This harness uses **Compaction** (not Context Reset) when running Opus 4.6+.
For Sonnet 4.5 or weaker models, switch to full Context Reset between sprints:

```python
# Context reset pattern (Sonnet 4.5)
client = anthropic.Anthropic()
# Start fresh agent each sprint; read artifacts, not conversation history
agent = client.beta.agents.create(
    model="claude-sonnet-4-6",
    system=open(".claude/agents/generator.md").read(),
)
```

State lives in **artifacts** (files), not in conversation memory — mirrors AGENTS.md:

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

---

## Harness Evolution Rule

> Every component in this harness encodes an assumption about what the model cannot do alone.
> When the model improves, re-test those assumptions and strip what is no longer load-bearing.

Review checklist after each model upgrade:
- [ ] Is context reset still necessary, or does compaction suffice?
- [ ] Does the sprint contract negotiation add value, or can Generator infer it?
- [ ] Are all four rubric dimensions still discriminating, or have some become trivial?
- [ ] Can Planner and Generator be merged without quality regression?

Simpler harness + stronger model > complex harness + weaker model.

---

## Critical Rules (Claude Code specific)

- Use `Agent(subagent_type="generator")` syntax — not `Task()` (deprecated in v2.1.63, aliased only)
- Subagents cannot invoke other subagents via Bash — use the `Agent` tool explicitly
- `tools:` in subagent frontmatter is an allowlist; omit to inherit all parent tools
- CLAUDE.md loads before every conversation; keep it under 300 lines
- `settings.json` is the right place for deterministic harness rules; CLAUDE.md is for context
- When CLAUDE.md and AGENTS.md appear to conflict, AGENTS.md wins
