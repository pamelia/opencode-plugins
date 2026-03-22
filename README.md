# opencode-plugins

Agent teams and skills for [opencode](https://github.com/anomalyco/opencode). Requires the teams/agents feature (team_create, team_task, send_message, agent_memory tools).

## Installation

Copy the agents and skills into your project's `.opencode/` directory:

```bash
# From your project root
cp -r path/to/opencode-plugins/agents .opencode/agents
cp -r path/to/opencode-plugins/skills .opencode/skills
```

Or symlink them:

```bash
ln -s path/to/opencode-plugins/agents .opencode/agents
ln -s path/to/opencode-plugins/skills .opencode/skills
```

## What's Included

### Agents (8 specialist reviewers)

All agents are designed to work as part of a coordinated review team, orchestrated by the `spec-review` skill.

| Agent | Focus |
|-------|-------|
| `clarity-reviewer` | Ambiguity, undefined terms, "two engineers" test |
| `completeness-reviewer` | Missing edge cases, error paths, state transitions |
| `product-reviewer` | Goal alignment, user value, success criteria |
| `feasibility-reviewer` | Technical feasibility, hidden complexity, alternatives |
| `api-reviewer` | API design, backward compat, naming, idempotency |
| `operations-reviewer` | Failure modes, observability, rollback, SLO impact |
| `scope-reviewer` | Incremental delivery, dependency risks, phasing |
| `complexity-reviewer` | Premature abstraction, over-engineering, accidental complexity |

### Skills (4 workflows)

| Skill | Description |
|-------|-------------|
| `spec-review` | Multi-agent parallel spec review with cross-review phase |
| `issue-to-spec` | GitHub issue → investigation → interview → hardened spec |
| `handle-pr-feedback` | Process and address PR review feedback systematically |
| `self-review-loop` | Iterative self-review loop: fresh agent reviews PR each turn |

## Usage

After installing, invoke skills via the skill tool:

```
/spec-review path/to/spec.md
/spec-review #42
/issue-to-spec #15
/self-review-loop #73
```

## Requirements

- opencode with teams/agents feature (team_create, team_delete, team_task, send_message, agent_memory tools)
- GitHub CLI (`gh`) for issue/PR workflows
