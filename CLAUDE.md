# CLAUDE.md

> Claude Code configuration for the GAN-inspired Planner → Generator → Evaluator harness.
> This file extends AGENTS.md with Claude-specific subagent definitions, hooks, and SDK patterns.

---

## Architecture Summary

Three-agent GAN-style harness built on the **Anthropic Agent SDK**:

```
User prompt (1–4 sentences)
    │
    ▼
┌─────────┐   planner-spec.json    ┌───────────────────────────────────┐
│ Planner │ ──────────────────────▶│         Sprint Loop (N times)     │
└─────────┘                        │                                   │
  runs once                        │  sprint-contract.md negotiation   │
                                   │        ┌───────────┐              │
                                   │        │ Generator │              │
                                   │        └─────┬─────┘              │
                                   │              │ code + commit       │
                                   │        ┌─────▼─────┐              │
                                   │        │ Evaluator │ ◀── Playwright MCP
                                   │        └─────┬─────┘              │
                                   │              │ pass / fail+critique│
                                   │        ◀─────┘                    │
                                   └───────────────────────────────────┘
```

---

## Subagent Definitions

Subagents live in `.claude/agents/`. Each runs in its own isolated context.

### `.claude/agents/planner.md`

```yaml
---
name: planner
description: >
  Use when the user provides a new product prompt (1–4 sentences) and needs
  a full spec generated. Runs once per project. Produces planner-spec.json.
tools: Read, Write, Bash, WebFetch
model: claude-opus-4-6
---
You are a product architect. Your sole job is to turn a short user prompt into
a complete, ambitious product spec.

Rules:
- Stay high-level: product context and architecture only. Never specify
  implementation details — let Generator figure out the path.
- Expand scope beyond what the user asked. Aim for 12–20 features across
  8–12 sprints.
- Read frontend-design/SKILL.md and embed a visual design language section
  in the spec.
- Look for opportunities to weave AI-native features into the product.
- Output to planner-spec.json with this schema:
  { "product": str, "design_language": str, "features": [...],
    "sprints": [{ "id": int, "title": str, "features": [...] }] }
- Write a brief executive summary to claude-progress.txt when done.
```

### `.claude/agents/generator.md`

```yaml
---
name: generator
description: >
  Use for each sprint implementation. Reads the spec, negotiates a sprint
  contract with the evaluator, implements features, commits, and updates
  claude-progress.txt.
tools: Read, Write, Edit, Bash, Agent
model: claude-sonnet-4-6
---
You are a senior full-stack engineer. You build one sprint at a time with
discipline.

Startup ritual (mandatory, every session):
1. pwd
2. cat claude-progress.txt
3. git log --oneline -10
4. Review planner-spec.json for next incomplete sprint
5. bash init.sh
6. Run smoke test — open app, perform one core action, verify it works
7. Only then proceed

Sprint workflow:
1. Propose sprint-contract.md (features + success criteria + test steps)
2. Wait for evaluator sign-off (or negotiate until agreed)
3. Implement sprint features
4. Self-evaluate: manually verify each contract criterion
5. git commit -m "feat(sprint-N): <concise description>"
6. Update claude-progress.txt with sprint summary
7. Invoke evaluator subagent

Hard rules:
- Never mark complete without Evaluator pass
- Never remove or edit tests
- Use git revert (not patch) when recovering from broken state
- Leave codebase in clean-state after every commit
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
You are a skeptical QA engineer and design critic. Your default is FAIL.

Evaluation process:
1. Read sprint-contract.md to understand what was promised
2. bash init.sh — ensure dev server is running
3. Navigate the live application; interact like a real user
4. Screenshot key states as evidence
5. Score each rubric dimension (see below)
6. Write eval-result-{N}.md with scores, evidence, and detailed critique
7. If ALL thresholds pass → write "SPRINT PASS" and notify generator
8. If ANY threshold fails → write "SPRINT FAIL", provide specific critique,
   return control to generator for revision

Rubric:
  Design quality  (weight 30%, threshold ≥ 7/10)
    Coherent visual identity; mood is intentional and consistent
  Originality     (weight 30%, threshold ≥ 6/10)
    Custom decisions visible; no purple gradients, white cards, AI slop
  Craft           (weight 20%, threshold ≥ 7/10)
    Typography hierarchy, spacing, contrast technically correct
  Functionality   (weight 20%, threshold ≥ 8/10)  ← hard gate
    All contracted features work end-to-end; failing this always fails sprint

Calibration rule: be harder on Originality than you think is fair.
The model defaults to safe; your job is to push it toward creative risk.
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
        "hooks": [{ "type": "command", "command": "echo '✓ Committed — update claude-progress.txt next'" }]
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
Invoke the generator subagent to propose a sprint contract for sprint $ARGUMENTS
from planner-spec.json, then invoke the evaluator subagent to review it.
Do not write any code until both have signed off.
```

`.claude/commands/eval.md` — Trigger evaluation on current sprint:
```
Invoke the evaluator subagent to evaluate sprint $ARGUMENTS.
Pass it the running application URL and sprint-contract.md.
Write results to eval-result-$ARGUMENTS.md.
```

`.claude/commands/status.md` — Session orientation:
```
Run: cat claude-progress.txt && git log --oneline -10 && cat planner-spec.json | python3 -c "import sys,json; s=json.load(sys.stdin); [print(f'Sprint {x[\"id\"]}: {x[\"title\"]}') for x in s['sprints']]"
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

State lives in **artifacts** (files), not in conversation memory:
- `planner-spec.json` — source of truth
- `claude-progress.txt` — session handoff log
- `eval-result-{N}.md` — evaluation history
- `sprint-contract.md` — current sprint definition

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
