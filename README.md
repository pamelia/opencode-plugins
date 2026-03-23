# opencode-plugins

Agent teams and skills for [opencode](https://github.com/anomalyco/opencode). Requires the teams/agents feature (team_create, team_task, send_message, agent_memory tools).

## Installation

Install globally so the agents and skills are available across all your projects:

```bash
# Clone the repo
git clone git@github.com:pamelia/opencode-plugins.git

# Copy into global opencode config
cp -r opencode-plugins/agents ~/.config/opencode/agents
cp -r opencode-plugins/skills ~/.config/opencode/skills
```

Or symlink them for easier updates:

```bash
ln -s "$(pwd)/opencode-plugins/agents" ~/.config/opencode/agents
ln -s "$(pwd)/opencode-plugins/skills" ~/.config/opencode/skills
```

To update later, pull the latest changes — symlinks will pick them up automatically.

## What's Included

### Agents (8 specialist reviewers)

Agents work as part of coordinated review teams, orchestrated by the `spec-review` and `code-review` skills.

| Agent | Focus |
|-------|-------|
| `clarity-reviewer` | Ambiguity, undefined terms, "two engineers" test |
| `completeness-reviewer` | Missing edge cases, error paths, state transitions |
| `product-reviewer` | Goal alignment, user value, success criteria |
| `feasibility-reviewer` | Technical feasibility, hidden complexity, alternatives |
| `api-reviewer` | API design, backward compat, naming, idempotency |
| `operations-reviewer` | Failure modes, observability, rollback, SLO impact |
| `scope-reviewer` | Incremental delivery, dependency risks, phasing |
| `complexity-reviewer` | Premature abstraction, over-engineering, accidental complexity (specs and code) |

### Skills (5 workflows)

| Skill | Description |
|-------|-------------|
| `code-review` | Multi-agent parallel PR review with scope-based reviewer selection |
| `spec-review` | Multi-agent parallel spec review with cross-review phase |
| `issue-to-spec` | GitHub issue → investigation → interview → hardened spec |
| `handle-pr-feedback` | Process and address PR review feedback systematically |
| `self-review-loop` | Iterative self-review loop using `code-review` — fresh agent reviews PR each turn |

## Usage

After installing, invoke skills via slash commands:

```
/code-review #42
/spec-review path/to/spec.md
/spec-review #42
/issue-to-spec #15
/self-review-loop #73
```

## Requirements

- opencode with teams/agents feature (team_create, team_delete, team_task, send_message, agent_memory tools)
- GitHub CLI (`gh`) for issue/PR workflows
