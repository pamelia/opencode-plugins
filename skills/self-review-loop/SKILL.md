---
name: self-review-loop
description: "Iterative self-review loop for PRs. Launches a fresh, context-free sub-agent each turn to run a code review skill against the PR, then evaluates and applies feedback. Loops until only minor/nit feedback remains or 5 turns complete. Auto-discovers the available code review skill."
---

# Self-Review Loop

You are an orchestrator that iteratively improves a PR by running fresh, unbiased code reviews and applying feedback. Each review is performed by a sub-agent with NO prior context — it sees only the PR diff, ensuring unbiased feedback with no anchoring to previous decisions.

The target PR is: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which PR to work on.

---

## Step 1: Parse the PR Reference and Set Up

### 1a. Parse the PR number

`$ARGUMENTS` should be a PR number. Accepted formats:

- **`#N`** or **`N`**: e.g., `#42` or `42`
- **GitHub PR URL**: e.g., `https://github.com/owner/repo/pull/42` — extract the PR number from the URL path

Any other format is not supported. If the argument doesn't match these patterns, ask the user to provide a PR number.

### 1b. Pre-flight: discover the code review skill

Before doing anything else, determine which code review skill is available. Check for skill files on disk rather than spawning agents — this is lightweight and avoids the cost of executing the skill just to test availability.

Check the known skill installation locations in order:

1. **Project-local**: `Glob(pattern: ".opencode/skills/code-review/SKILL.md")`
2. **Global config**: `Glob(path: "~/.config/opencode", pattern: "skills/code-review/SKILL.md")`

If a match is found in either location, set `review_skill` to `code-review` and proceed to Step 1c. Stop searching after the first match.

**Not available — stop:**

If no code review skill was found in any location, stop immediately and inform the user:

```
/self-review-loop requires a code review skill, but none was found.

Install a code-review skill to use this workflow.
```

### 1c. Fetch PR details

```
gh pr view <N>
```

Confirm the PR is open. If merged or closed, inform the user and stop.

### 1d. Check out the PR branch

```
gh pr checkout <N>
but pull --json --status-after
```

If `but` is not available, fall back to `git pull`.

### 1e. Initialize the loop state

Set up tracking for the loop:

- `turn`: 1
- `max_turns`: 5
- `changelog`: empty list — accumulates ALL changes across ALL turns
- `all_skipped`: empty list — accumulates ALL skipped feedback across ALL turns
- `stop_reason`: null — will be set when the loop terminates
- `files_changed_per_turn`: empty map of turn → set of files changed — used for oscillation detection

---

## Step 2: The Review Loop

Repeat the following for each turn until a stop condition is met.

### 2a. Launch a fresh review sub-agent

**CRITICAL**: The sub-agent must have NO context from previous turns. This prevents bias — the reviewer should evaluate the code as-is, not relative to what it used to be. Each turn gets a completely fresh agent that knows nothing about prior feedback or changes.

Spawn the sub-agent using the `task` tool, using the `review_skill` discovered in Step 1b:

```
task(
  description: "Code review turn N",
  prompt: "Run /<review_skill> against PR #<N>. Do not add any additional context or commentary — just run the skill and report back the full review output exactly as produced.",
  background: true
)
```

The sub-agent will invoke the code review skill, which may in turn spawn its own agent team (via `team_create`). The code review skill handles its own team cleanup, so when the sub-agent returns, any review team should already be torn down along with the sub-agent itself.

> **Trust assumption**: The sub-agent runs in the background to avoid blocking the orchestrator. This is safe because code review skills only read code (Glob, Grep, Read, git diff) and send messages between their own agents. They do not edit files, push code, or make external calls. The orchestrator itself (this skill) is the one that edits files and pushes.

### 2b. Capture the review output

When the sub-agent returns, capture its full output. This contains the structured review with findings organized by priority tier (Critical, High, Medium, Low) and a verdict (APPROVE or REQUEST CHANGES).

### 2c. Evaluate the stop condition

Parse the review output and check if the loop should stop:

**Stop condition — only non-actionable feedback remains:**
A review qualifies as "non-actionable" if ALL of the following are true:

- The verdict is **APPROVE**
- There are **zero** findings in the Critical or High tiers
- All remaining findings are classified as `suggestion`, `nitpick`, `thought`, `risk`, or `question` (none of these require code changes — `risk` is an acknowledged trade-off and `question` is a request for clarification, not a code fix)

**Stop condition — max turns reached:**

- `turn` equals `max_turns` (5)

If either stop condition is met, set `stop_reason` to `"clean_review"` or `"max_turns"` respectively and proceed to Step 3.

**Stop condition — oscillation detected:**
After turn 2, check `files_changed_per_turn` for thrashing. If the set of files changed in the current turn's triage (Step 2d) overlaps significantly (>50%) with the files changed two turns ago, the loop is oscillating — reviewers are undoing each other's changes. Set `stop_reason` to `"oscillation"` and proceed to Step 3. When reporting, note which files were thrashing and the conflicting feedback.

If no stop condition is met, continue to 2d.

### 2d. Triage the feedback

For each finding in the review output, decide whether to **address** or **skip** it.

**Address** the finding if:

- It is in the Critical or High tier
- It identifies a real bug, logic error, or correctness issue
- It requests reasonable error handling, validation, or edge case coverage
- It points out a style/convention violation consistent with the codebase
- It is a `blocker` or `risk` finding with a concrete harm scenario

**Skip** the finding if:

- It is a `nitpick` or `thought` with no impact on correctness
- It is a subjective style preference with no codebase convention backing it
- Addressing it would require architectural changes beyond the PR's scope
- The finding is based on a misunderstanding of the code
- The suggested change would introduce a regression or break existing behavior
- It conflicts with feedback from a previous turn that was already applied

For each finding, record:

- **Action**: `addressed` or `skipped`
- **Summary**: 1-2 sentence description of what you did or why you skipped
- **Files changed**: list of files modified (if addressed)
- **Turn**: current turn number

### 2e. Apply changes

For each finding you are addressing:

1. Read the referenced file
2. Make the code change using `Edit`
3. Verify the change makes sense in context

### 2f. Verify changes

After applying all changes for this turn, run the project's test suite and/or linter if one exists. Detect the test runner by checking for common patterns:

- `package.json` with a `test` script → `npm test` or equivalent
- `Makefile` with a `test` target → `make test`
- `pytest.ini`, `pyproject.toml`, or `setup.cfg` with pytest config → `pytest`
- `go.mod` → `go test ./...`
- `Cargo.toml` → `cargo test`
- `Gemfile` with rspec → `bundle exec rspec`
- `*.csproj` or `*.sln` → `dotnet test`
- `pom.xml` → `mvn test`
- `build.gradle` or `build.gradle.kts` → `gradle test`
- `.github/workflows/` CI config → inspect for the test command used in CI

If a test runner is found, run it. If tests fail:

1. Examine the failure and determine if it was caused by changes made in this turn
2. If yes, fix the issue before proceeding — this counts as part of the same turn's changes
3. If the failure is pre-existing (also fails on the PR's base branch), note it in the changelog but proceed

If no test runner is detected, skip this step and note "no test suite detected" in the turn summary.

### 2g. Commit and push

If any files were changed, commit and push. Use `but` (GitButler CLI) if available, otherwise fall back to `git`:

**With GitButler (`but`):**

```
but status --json
but commit <branch> -m "Address code review feedback (turn N)

- <summary of change 1>
- <summary of change 2>
..." --changes <id1>,<id2> --json --status-after
but push
```

**Without GitButler (fallback):**

```
git add <file1> <file2> ...
git commit -m "Address code review feedback (turn N)

- <summary of change 1>
- <summary of change 2>
..."
git push
```

If no files were changed (all findings skipped or minor-only), skip the commit.

### 2h. Update the changelog

Append this turn's results to the changelog and skipped lists. Record:

- Turn number
- Number of findings in each tier
- What was addressed (with file references)
- What was skipped (with reasons)
- Commit SHA (if a commit was made)
- Update `files_changed_per_turn[turn]` with the set of files modified this turn

### 2i. Increment and continue

Increment `turn` by 1 and go back to Step 2a.

---

## Step 3: Final Summary

After the loop terminates, present a comprehensive summary to the user.

```
## Self-Review Complete: PR #<N>

**Turns completed**: <turn count>
**Stop reason**: <"Clean review — only non-actionable feedback remaining" or "Maximum turns (5) reached" or "Oscillation detected — turns were undoing each other's changes">

### Turn-by-Turn Summary

#### Turn 1
- **Verdict**: <APPROVE/REQUEST CHANGES>
- **Findings**: <X Critical, Y High, Z Medium, W Low>
- **Addressed**: <count>
- **Skipped**: <count>
- **Commit**: <SHA> (or "no changes")

#### Turn 2
...

### All Changes Applied

- [file:line] — <what was changed> (turn N)
- ...

### All Feedback Skipped

- <finding summary> — <why it was skipped> (turn N)
- ...

### Final Review State

<Paste the verdict section from the last review>
```

---

## Guidelines

### Why fresh agents each turn

The core design principle: each review must be unbiased. If the reviewer knows what feedback was given last turn, it anchors on those findings and may miss new issues introduced by the fixes, or fail to notice that a previous concern was only partially addressed. A fresh agent sees the code with no history and evaluates it purely on its current state.

### Conflict resolution across turns

If a later turn's review gives feedback that contradicts a change made in an earlier turn:

- Do NOT revert the earlier change automatically
- Evaluate both positions and pick the one that is objectively better for the codebase
- Record the conflict and your reasoning in the skipped list
- If this happens repeatedly on the same files, the oscillation detector (Step 2c) will catch it and break the loop

### Safety

- Never force-push
- Never modify files outside the scope of the PR's changes unless a reviewer explicitly requests it and the change is safe
- If a requested change seems risky (could break tests, change public API behavior), skip it and record why
- If applying a fix introduces a new issue you notice, fix that too in the same turn

### Commit hygiene

- One commit per turn (not per finding)
- Commit messages reference the turn number for traceability
- Each commit should be independently meaningful

---

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The first review was clean enough, let's stop" | Check the stop condition rigorously. "Clean enough" is not the same as "zero Critical/High findings with an APPROVE verdict." If the review returned REQUEST CHANGES, another turn is needed. |
| "The reviewer is being too strict, I'll skip most findings" | A fresh reviewer has no context bias — if it flags something, evaluate it objectively. Skipping most findings across multiple turns means either the review skill is miscalibrated (unlikely) or you're avoiding real issues. |
| "Tests pass, so the changes are safe" | Tests verify the changes don't break existing behavior. They don't verify the changes are correct, well-structured, or address the review findings properly. The next review turn will catch what tests can't. |
| "Five turns is too many, I'll stop at 2" | The max is 5 turns, not the target. Most PRs converge in 2-3 turns. If you're at turn 4, something is wrong — but stopping early hides it. Let the stop conditions decide, not impatience. |
| "I already know what the reviewer will say, I'll pre-apply the fixes" | The point of fresh agents is unbiased evaluation. Pre-applying imagined fixes introduces your bias and may miss real issues the fresh reviewer would catch. Let the review run, then respond. |
| "This finding conflicts with a previous turn, so I'll skip it" | Conflicts between turns are signal. Evaluate both positions on their merits. If the later review is right, the earlier change was wrong — fix it. The oscillation detector handles genuine thrashing. |

## Red Flags

- Same findings appearing across multiple turns without being addressed
- All findings skipped in every turn (the loop becomes a no-op)
- Loop runs all 5 turns with no improvement in verdict or finding counts
- Changes made without running the test suite (Step 2f skipped)
- Files changing back and forth across turns without the oscillation detector triggering
- Sub-agent spawned with context from previous turns (violates the fresh-agent principle)
- Commit pushed with zero changes listed in the changelog
- Test failures caused by review changes not fixed before the next turn

## Verification

After the loop completes:

- [ ] The stop reason is documented and accurate (clean review, max turns, or oscillation)
- [ ] Every turn has a commit or an explicit note that no changes were made
- [ ] The changelog accounts for ALL changes across ALL turns
- [ ] All skipped findings have documented reasons
- [ ] Tests were run after each turn's changes (or "no test suite" was noted)
- [ ] The final review output is included in the summary
- [ ] No uncommitted changes remain on the branch
- [ ] If oscillation was the stop reason, the thrashing files and conflicting feedback are documented
