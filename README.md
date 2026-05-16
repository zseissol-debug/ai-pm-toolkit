# AI PM Toolkit

A collection of Claude Code skills for AI Product Managers. Built and battle-tested by a non-technical PM.

## Skills

### SDD (Spec Driven Development) - AI Coding Quality Control

A methodology for non-technical PMs to manage AI-generated code quality through structured specs and systematic verification.

Core insight: AI is ~80% correct per decision, but a feature involves 20+ decisions. 0.8^20 ≈ 1% — the probability of getting everything right in one shot is near zero. SDD solves this with a Specify → Verify → Fix loop.

| Skill | Description | Trigger |
|-------|-------------|---------|
| **pm-init** | Project constitution setup (mission, constraints, tech stack, roadmap) | "Initialize project", "new project setup" |
| **pm-specify** | PRD → Spec translator (7-element structure + Gherkin acceptance criteria) | "Write Spec", "PRD to Spec" |
| **pm-verify** | Verifier agent — checks code against Spec, outputs structured deviation report | "Verify", "check the code" |
| **pm-fix** | Dual-path fix guide — bug (fix code) vs. direction drift (fix Spec first) | "There's a bug", "how to fix" |

**Workflow**: pm-init → pm-specify → (AI writes code) → pm-verify → pm-fix → pm-verify → ... until 100% pass

### AI Product PRD Writer

| Skill | Description | Trigger |
|-------|-------------|---------|
| **ai-product-prd-writer** | Full-process PRD writing for AI products (Agent, Tool, Conversational, Platform types) | "Write PRD", "product requirements" |

Covers AI-specific PRD elements: Agent/Workflow design, Prompt design specs, evaluation frameworks, fallback strategies. Includes Coach Mode (guided) and Express Mode (fast generation).

## Installation

Add to your Claude Code project settings (`.claude/settings.json`):

```json
{
  "permissions": {
    "allow": []
  },
  "skills": [
    "github:zseissol-debug/ai-pm-toolkit/skills/pm-specify",
    "github:zseissol-debug/ai-pm-toolkit/skills/pm-verify",
    "github:zseissol-debug/ai-pm-toolkit/skills/pm-fix",
    "github:zseissol-debug/ai-pm-toolkit/skills/pm-init",
    "github:zseissol-debug/ai-pm-toolkit/skills/ai-product-prd-writer"
  ]
}
```

Or install individual skills:

```bash
# SDD series
claude skills add github:zseissol-debug/ai-pm-toolkit/skills/pm-init
claude skills add github:zseissol-debug/ai-pm-toolkit/skills/pm-specify
claude skills add github:zseissol-debug/ai-pm-toolkit/skills/pm-verify
claude skills add github:zseissol-debug/ai-pm-toolkit/skills/pm-fix

# PRD Writer
claude skills add github:zseissol-debug/ai-pm-toolkit/skills/ai-product-prd-writer
```

## Real-world Validation

The SDD methodology was validated on an English essay AI grading tool (Express + vanilla HTML):

- 5 Specs, 58 Gherkin acceptance criteria
- First verify pass rate: 58% (7/12) — found 5 implementation deviations
- After fix: 100% (12/12)
- Non-technical PM identified and located all issues without reading a single line of code

## License

MIT
