---
name: code-review
description: "Reviews a GitHub PR using a parallel agent team. Spawns specialist reviewers (correctness, style, architecture) that examine the diff, self-critique findings, and report back. Produces a structured review with findings organized by priority tier and a binary verdict."
---

# Code Review

**YOU MUST SPAWN AN AGENT TEAM.** Do NOT review the PR yourself. You are the team lead — your job is orchestration, not review.

Your workflow:

1. Gather the PR diff and context
2. Assess scope and select specialists
3. Create a team and spawn specialists
4. Collect findings
5. Deduplicate and synthesize into final review
6. Clean up

The target of the review is: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which PR to review.

## Step 1: Gather the PR

### 1a. Parse the PR reference

`$ARGUMENTS` should be a PR number. Accepted formats:

- **`#N`** or **`N`**: e.g., `#42` or `42`
- **GitHub PR URL**: e.g., `https://github.com/owner/repo/pull/42` — extract the PR number from the URL path

Any other format is not supported. If the argument doesn't match these patterns, ask the user to provide a PR number.

### 1b. Fetch PR details

```
gh pr view <N>
gh pr diff <N>
```

Capture the PR title, description, base branch, and the full diff.

### 1c. Understand the diff scope

Analyze the diff to determine:

- **Files changed**: list all modified/added/deleted files
- **Languages**: what languages are involved
- **Scope**: is this a small fix, a feature addition, a refactor, or an architecture change?
- **Lines changed**: rough count of additions/deletions

### 1d. Read key files for context

For each file in the diff, read the full file (not just the diff hunks) so reviewers understand the surrounding context. If there are more than 15 files, prioritize:

1. Files with the most changes
2. Core logic files over config/test files
3. New files over modified files

Gather the contents into a context block that will be passed to specialists.

## Step 2: Assess Scope and Select Specialists

### Scope Classification

| Scope      | Criteria                                                    | Reviewers                                                      |
| ---------- | ----------------------------------------------------------- | -------------------------------------------------------------- |
| **Small**  | 1-3 files, <100 lines changed, bug fix or config change     | 2 reviewers: correctness + style                               |
| **Medium** | 4-15 files, 100-500 lines, feature or refactor              | 3 reviewers: correctness + style + completeness                |
| **Large**  | 15+ files, 500+ lines, architecture change or new subsystem | 4 reviewers: correctness + style + completeness + architecture |

### Specialist Descriptions

| #   | Agent                     | Focus                                                                                                            |
| --- | ------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1   | **completeness-reviewer** | Missing edge cases, error handling, validation, untested paths, incomplete migrations                            |
| 2   | **complexity-reviewer**   | Unnecessary complexity, over-engineering, premature abstractions, readability, codebase convention violations     |
| 3   | **feasibility-reviewer**  | Architectural fit, technical correctness, hidden complexity, resource management, security risks                  |
| 4   | **operations-reviewer**   | Production readiness, backward compatibility, performance implications, failure modes, observability gaps         |

State which reviewers you're spawning and why before proceeding.

## Step 3: Create Team and Spawn Reviewers

### 3a. Create the team

```
team_create(name: "code-review-pr-<N>")
```

### 3b. Create tasks for each reviewer

```
team_task(operation: "create", team_id: "<team_id>", subject: "<reviewer>-review", owner: "<reviewer>")
```

### 3c. Spawn all reviewers in a SINGLE message

Each reviewer gets the full diff, file contexts, and PR description in their prompt. All `task` calls must be in ONE message for parallel execution.

```
task(
  subagent_type: "<reviewer-type>",
  description: "<Reviewer> review of PR #<N>",
  background: true,
  team_id: "<team_id>",
  prompt: "<see prompt template below>"
)
```

### Reviewer Prompt Template

Each reviewer receives this prompt (fill in the specifics):

```
REVIEW TYPE: Code Review
PR: #<N> — <title>
SCOPE: <Small/Medium/Large>

SELF-CRITIQUE REQUIREMENT:
After completing your review, self-critique each finding:
- Is this a real problem or am I being overly cautious?
- Would a senior engineer on this team flag this?
- Does the codebase already have this pattern elsewhere (making it intentional)?
Prune or downgrade findings that don't survive scrutiny.

PR DESCRIPTION:
<PR description/body>

DIFF:
<full diff output>

FILE CONTEXTS:
<for each changed file, include the full file content with path>

REVIEW INSTRUCTIONS:
Review this PR from your specialist perspective. For each finding, provide:
1. **File and line reference** — e.g., `src/foo.ts:42`
2. **Category** — one of: `blocker`, `risk`, `question`, `suggestion`, `nitpick`, `thought`
3. **Priority** — P0 (critical), P1 (high), P2 (medium), P3 (low)
4. **Description** — what the issue is and why it matters
5. **Suggested fix** — concrete recommendation (code snippet if applicable)

Category definitions:
- `blocker`: Must fix before merge. Causes bugs, data loss, security issues.
- `risk`: Likely problem that should be consciously accepted or mitigated.
- `question`: Seeking clarification — something looks wrong but you may be missing context.
- `suggestion`: Concrete improvement with rationale. Not blocking.
- `nitpick`: Minor style/formatting preference.
- `thought`: Observation or future consideration. Not requesting changes.

After self-critique, send your final findings to the lead via send_message. Format as a numbered list. If you have no findings, send "No findings — code looks good from my perspective."

Your task has been created. Use team_task to update it to in_progress when you start, and mark it completed when done.
```

## Step 4: Collect Findings

Wait for all reviewers to report via `send_message`. Messages arrive automatically.

**Error recovery**: If a reviewer fails or crashes, note it in the output. Do not re-spawn — the review can proceed with fewer reviewers.

## Step 5: Synthesize the Final Review

### Deduplication

When multiple reviewers flag the same issue:

- Consolidate into one finding using the most impactful framing
- Note which reviewers agreed
- Use the highest severity assigned by any reviewer

### Comment Taxonomy

| Label        | Meaning                                    | Blocking? |
| ------------ | ------------------------------------------ | --------- |
| `blocker`    | Must fix before merge. Cite concrete harm. | Yes       |
| `risk`       | Gap or failure mode to consciously accept. | Discuss   |
| `question`   | Seeking clarification from the author.     | No        |
| `suggestion` | Concrete improvement with rationale.       | No        |
| `nitpick`    | Minor style/formatting preference.         | No        |
| `thought`    | Observation or future consideration.       | No        |

### Priority Mapping

| Output Tier | Maps From                       |
| ----------- | ------------------------------- |
| Critical    | Any `blocker` finding           |
| High        | P0/P1 non-blocker findings      |
| Medium      | P2 findings                     |
| Low         | P3 findings, nitpicks, thoughts |

Empty tiers are omitted.

### Output Format

```
## Code Review: PR #<N>

**PR**: <title>
**Scope**: <Small/Medium/Large> (<N> files, ~<N> lines changed)
**Reviewers**: <list of specialists>

## Critical
[Findings that must be fixed before merge]

**<file:line> — <title>**
<description of the issue and concrete harm>
- **Category**: blocker
- **Reviewer(s)**: <who found it>
- **Suggested fix**: <concrete recommendation>

## High Priority
[Same per-item format]

## Medium Priority
[Same per-item format]

## Low Priority
[Same per-item format]

## Verdict: APPROVE / REQUEST CHANGES
<1-2 sentence rationale. If REQUEST CHANGES, list the Critical/High items that must be addressed.>
```

### Verdict Rules

- **APPROVE**: Zero `blocker` findings AND zero `risk` findings in Critical/High tiers. Remaining findings are all suggestions, nitpicks, thoughts, or questions.
- **REQUEST CHANGES**: Any `blocker` finding exists, OR any `risk` finding in Critical/High tier that hasn't been explicitly accepted.

The verdict must be the LAST section.

## Step 6: Clean Up

After delivering the review, use `team_delete` to disband the team.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This is a small change, we don't need multiple reviewers" | Small changes can introduce subtle bugs. The scope classification exists for a reason — trust it. Two reviewers catch things one misses. |
| "The tests pass, so the code is correct" | Tests are necessary but not sufficient. They don't catch architecture problems, security issues, readability concerns, or missing edge cases the tests themselves don't cover. |
| "I already know this code, I can review it myself" | You are the orchestrator, not a reviewer. Self-review has blind spots. Spawn the agents — that's the entire point of this skill. |
| "The reviewers are too nitpicky, I'll consolidate everything as Low" | Don't downgrade findings to produce a cleaner report. If a reviewer flagged something as a blocker, preserve that severity. You arbitrate disagreements, not silence them. |
| "The PR is too large to review properly, but we'll do our best" | Mention in the review output that the PR should be split. Large PRs hide bugs and make reviews superficial. Flag it, don't ignore it. |
| "The reviewer failed, I'll just deliver the review with fewer specialists" | Acceptable for recovery, but note the gap. A missing correctness reviewer means correctness was not independently verified — say so in the output. |

## Red Flags

- Lead reviews the PR itself instead of spawning agents
- All reviewers return zero findings on a non-trivial PR (>100 lines)
- Lead systematically downgrades all findings to Low priority
- Large PR reviewed without mentioning that it should be split
- No file context provided to reviewers (they need full files, not just diff hunks)
- Reviewers spawned sequentially instead of in a single parallel message
- Verdict is APPROVE despite unresolved blocker findings
- Review output omits the file:line references, making findings unactionable

## Calibration Principles

1. **The diff is the scope.** Don't review code that wasn't changed unless the change creates a new issue in surrounding code.
2. **The author is competent.** Assume good intent. If something looks wrong, ask before asserting.
3. **Proportional rigor.** A 5-line bug fix doesn't need architecture review.
4. **Every comment has a cost.** 5 high-signal findings > 50 mixed-signal findings.
5. **Be specific.** File paths, line numbers, code snippets. Vague feedback is useless.
6. **Respect existing patterns.** If the codebase does X everywhere, don't flag this PR for doing X.
7. **Focus on correctness over style.** Bugs matter more than formatting.
