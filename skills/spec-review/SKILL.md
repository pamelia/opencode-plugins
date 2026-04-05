---
name: spec-review
description: "Orchestrates a three-phase parallel spec review using an agent team. Phase 1: dynamically selected specialists each review the spec and self-critique findings. Phase 2: lead-mediated cross-review where specialists challenge each other's findings. Phase 3: deduplicated synthesis with priority-based output and binary approval verdict."
---

# Spec Review Agent Team

**YOU MUST SPAWN AN AGENT TEAM.** Do NOT review the spec yourself. You are the team lead — your job is orchestration, not review.

Your workflow:
1. Gather the spec
2. Classify risk lane and select relevant specialists
3. Create a team and spawn specialists
4. **Phase 1**: Collect specialist findings (each agent reviews + self-critique)
5. **Phase 2**: Mediate cross-agent challenges (L1/L2 only)
6. **Phase 3**: Synthesize into final review
7. Clean up

The target of the review is: $ARGUMENTS

If $ARGUMENTS is empty, ask the user what to review.

## Step 1: Gather the Spec

Obtain the spec content before spawning agents.

### Input Detection

- **`#N`** (GitHub number): GitHub uses a single number space — `#N` is either an issue or a PR. Run `gh issue view <N> 2>/dev/null || gh pr view <N>` to resolve which. Then:
  - **If issue**: The issue body is the primary spec. Pull comments via `gh api repos/{owner}/{repo}/issues/<N>/comments` for discussion context, clarifications, and decisions made in-thread. Check for linked PRs — if one exists, also pull its diff as implementation context.
  - **If PR**: Check if the PR description is the spec, or if the diff contains spec files (`.md`, `.txt`, design docs). Read both. Check for a linked issue — if one exists, pull it as additional spec context.
- **File path**: Read the file directly. Supports markdown, txt, or any text format.
- **URL**: Use `WebFetch` to retrieve the content. Works with Google Docs, Notion, Confluence, and other web-accessible docs.
- **"staged"**: `git diff --cached --name-only` to find staged files, then read spec-like files (`.md`, `.txt`, etc.).
- **No argument**: The spec should be in the conversation context. If not, ask the user.

Multiple arguments can be combined (e.g., `#42 path/to/spec.md`). When combining, the file or issue body is the primary spec; other sources provide supporting context.

### Gathering Context

Also gather:
- Related existing specs or design docs in the repo (use Glob/Grep to find them)
- The codebase area this spec targets (understand the current architecture)
- Linked issues or PRs referenced in the spec (follow `#N` references via `gh issue view` or `gh pr view`)

## Step 2: Assess, Classify, and Select Agents

### Spec Size

Note the spec length (sections, paragraphs, word count). Larger specs warrant more reviewers.

### Risk Lane

| Lane | Criteria | Self-Critique? | Cross-Review? |
|------|----------|----------------|---------------|
| **L0 — Minor** | Typo fixes, small clarifications, addenda to existing specs | No | No |
| **L1 — Significant** | New features, API additions, workflow changes, 3+ sections | Yes | Yes |
| **L2 — Strategic** | Architecture changes, new services, public API, security-sensitive, data model changes | Yes | Yes |

### Dynamic Agent Selection

Analyze the spec and select which specialists are relevant. Not every spec needs all 8 agents. Err toward including rather than excluding for L1/L2.

| # | Agent | Spawn guidance |
|---|-------|----------------|
| 1 | clarity-reviewer | Always. The "two engineers" test — would two engineers build the same thing from this spec? |
| 2 | completeness-reviewer | Always. Missing edge cases, error behavior, NFRs, state transitions. |
| 3 | product-reviewer | L1/L2 always. Goal alignment, user value, success criteria. Answers "are we building the right thing?" |
| 4 | feasibility-reviewer | New services, architecture changes, significant new features. Hidden complexity, alternative approaches. |
| 5 | api-reviewer | Spec defines or modifies an API. Backward compat, protobuf conventions, naming, idempotency. |
| 6 | operations-reviewer | Production-impacting changes. Failure modes, observability, rollback, SLO impact. |
| 7 | scope-reviewer | L1/L2 only. Multi-team, multi-phase, or large specs. Incremental delivery, dependency risks. |
| 8 | complexity-reviewer | L1/L2. Specs introducing new abstractions, multi-layer designs, configurable systems, or framework-like patterns. Catches premature abstraction, over-engineering, and accidental complexity. |

For **L0**: spawn only agents 1 and 2.

State which agents you're spawning and why before proceeding.

## Step 3: Create Team and Spawn Agents

### 3a. Create the team

Use the `team_create` tool:
```
team_create(name: "spec-review-<short-identifier>")
```

### 3b. Create tasks for each selected agent

Use the `team_task` tool with operation "create" for each agent:
```
team_task(operation: "create", team_id: "<team_id>", subject: "clarity-review", owner: "clarity-reviewer")
```

### 3c. Spawn all agents in a SINGLE message

Each specialist has a custom agent definition (in `agents/`) with its review protocol, specialist instructions, and persistent memory. You do NOT need to assemble prompts — the agent's `.md` file provides its system prompt automatically.

Spawn using `subagent_type` matching the agent name, with `background: true` and the `team_id`. The Task prompt contains only the dynamic content:

```
task(
  subagent_type: "clarity-reviewer",
  description: "Clarity review",
  background: true,
  team_id: "<team_id>",
  prompt: "RISK LANE: L1\n\nSELF-CRITIQUE REQUIREMENT:\nThis is an L1/L2 review. After your specialist review, stress-test your findings through self-critique before sending them. Follow your Self-Critique protocol and prune or downgrade findings that don't survive scrutiny.\n\nSPEC CONTEXT:\n<Title, purpose, and any relevant background>\n\nRELATED CODEBASE CONTEXT:\n<Brief description of existing system>\n\nSPEC CONTENT:\n<the full spec text>\n\nYour task has been created. Use team_task to update it to in_progress when you start, and mark it completed when done sending findings."
)
```

Repeat for every selected agent — all `task` calls in ONE message.

**CRITICAL**: Each agent's prompt MUST contain the full spec text. Agents cannot see the spec unless you include it in their prompt.

## Step 4: Phase 1 — Collect Specialist Findings

Agents work in parallel:
1. Each agent conducts its specialist review of the spec
2. Each agent self-critiques findings to harden them (L1/L2 only)
3. Each agent sends hardened findings to you via `send_message`
4. Each agent then goes idle, waiting for Phase 2

Wait for **all** agents to report. Messages are delivered automatically — you do not need to poll.

**Error recovery**: If an agent fails or crashes, re-spawn it with the same prompt and reassign its task.

## Step 5: Phase 2 — Lead-Mediated Cross-Review (L1/L2 only)

**Skip for L0.**

**Short-circuit rule**: If ALL Phase 1 findings are `suggestion`, `nitpick`, `thought`, or informational (zero `blocker`, `risk`, or `question` findings across all agents), skip cross-review. State in the synthesis: "Phase 2 skipped: no blocker/risk/question findings to challenge."

After collecting all Phase 1 findings:

1. **Identify cross-review targets** using your judgment AND the routing table:

   | Finding Source | Route To |
   |----------------|----------|
   | clarity-reviewer | completeness-reviewer |
   | completeness-reviewer | product-reviewer |
   | product-reviewer | scope-reviewer |
   | feasibility-reviewer | complexity-reviewer |
   | api-reviewer | operations-reviewer |
   | operations-reviewer | feasibility-reviewer |
   | scope-reviewer | product-reviewer |
   | complexity-reviewer | feasibility-reviewer |

   Prioritize routing:
   - **Contradictions**: two agents disagree
   - **Domain overlap**: a finding where another specialist has relevant expertise
   - **High-severity findings** (blockers and P0/P1 risks) that deserve a second opinion

2. **Route challenges** via `send_message` to the best-positioned agent. Include the original finding, its source agent, and what you want challenged.

3. **Collect responses**: the challenged agent evaluates and responds. Max 1 challenge round per finding.

4. **Arbitrate**: if agents cannot align, you decide. You are the final arbiter.

5. **Tag findings**: Confirmed / Modified / Withdrawn / Disputed

## Step 6: Phase 3 — Synthesize the Final Review

### Comment Taxonomy

| Label | Meaning | Blocking? |
|-------|---------|-----------|
| `blocker` | Must resolve before implementation begins. Cite concrete harm. | Yes |
| `risk` | Gap or failure mode to consciously accept. | Discuss |
| `question` | Seeking clarification from the spec author. | No |
| `suggestion` | Concrete alternative wording or addition with rationale. | No |
| `nitpick` | Minor wording or formatting preference. | No |
| `thought` | Observation or future consideration, not a request. | No |

### Comment Framing

- Questions over statements: "What happens when X?" NOT "This is wrong"
- Personal perspective: "I find this ambiguous because..." NOT "This is unclear"
- Focus on the spec, not the author: "This section omits X" NOT "You forgot X"
- No diminishing language: never "simply," "just," "obviously," "clearly"
- No surprise late blockers: if the approach is wrong, say so immediately

### Deduplication

Consolidate findings flagged by multiple agents into the single most impactful framing. Note which agents agreed. When deduplicating, use the highest priority (lowest P-number) assigned by any agent.

### Priority Mapping (internal classification -> output tier)

| Output Tier | Maps From |
|---|---|
| Critical | Any `blocker` finding (regardless of P-level) |
| High | P0/P1 non-blocker findings |
| Medium | P2 findings |
| Low | P3 findings, nitpicks, thoughts |

Empty tiers are omitted. Questions get folded into the appropriate tier based on their priority.

### Output Structure

```
## Summary
- **Spec**: [Title or filename]
- **Risk Lane**: L0/L1/L2
- **Reviewers**: [List of specialists who reviewed]
- **One-line assessment**: [Overall take]

## Critical
[Items that must be resolved before implementation]

**Section — Title**
Blurb describing the gap, ambiguity, or issue. Include concrete harm scenario and suggested fix or rewrite.
- **Specialist**: [clarity/completeness/etc.] — [must fix / can defer] — [1-sentence rationale]
- **Cross-review**: [Confirmed / Modified / Disputed] — [1-sentence note] (L1/L2 only)

## High Priority
(same per-item format)

## Medium Priority
(same per-item format)

## Low Priority
(same per-item format)

## Verdict: APPROVED / REVISIONS NEEDED
[1-2 sentence rationale. If REVISIONS NEEDED, list the Critical items that must be resolved.]
```

### Verdict (REQUIRED — must be the LAST section)

Binary. No "approved with suggestions" — either the spec is ready for implementation or it isn't.

## Step 7: Clean Up

After delivering the review, use `team_delete` to disband the team. Agents persist learnings via their agent_memory — they do not need to stay alive for context retention.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This spec is straightforward, L0 is fine" | L0 skips self-critique and cross-review. If the spec introduces a new feature, API change, or touches multiple components, it's L1 at minimum. Don't under-classify to save time. |
| "Cross-review will just slow things down" | Cross-review catches contradictions between specialists and prevents false confidence. It's the difference between "3 agents independently agreed" and "3 agents challenged each other and what survived is solid." |
| "The spec is already approved by stakeholders, we shouldn't question it" | Stakeholder approval is about business intent, not technical completeness. A spec can be strategically correct and technically ambiguous. Review the technical quality, not the business decision. |
| "The reviewers keep disagreeing, I'll just pick the safer verdict" | Disagreement is signal, not noise. Arbitrate based on evidence from the spec, not by defaulting to REVISIONS NEEDED. If findings are contradictory, that's Phase 2's job to resolve. |
| "All agents returned suggestions only, so the spec is clearly fine" | Probably true, but verify the agents actually reviewed deeply. Check that findings reference specific sections and that the self-critique process ran. "No findings" from a reviewer that didn't read the spec is not approval. |
| "I'll spawn all 8 agents to be thorough" | More agents = more noise. The dynamic selection criteria exist to match reviewers to the spec's content. Spawning an API reviewer for a spec with no API changes wastes tokens and dilutes signal. |

## Red Flags

- Lead reviews the spec itself instead of spawning agents
- All agents return zero findings on an L2 spec
- Phase 2 cross-review skipped on a spec with blocker or risk findings
- Lead overrides agent findings without stating reasoning
- Only clarity + completeness spawned for a security-sensitive or API-changing spec
- Risk lane classified as L0 for a spec that introduces new services or data model changes
- Review output doesn't reference specific sections of the spec
- Verdict rendered before all agents have reported

## Calibration Principles

1. **The spec is not the implementation.** Don't demand implementation-level detail in a design document. The spec should constrain, not dictate.
2. **The author is competent.** Assume good intent and context you may lack. Ask before assuming wrong.
3. **Proportional rigor.** A 2-page addendum doesn't need the same scrutiny as a new service design.
4. **Every comment has a cost.** 3 high-signal findings > 30 mixed-signal findings.
5. **Be explicit about severity.** The taxonomy distinguishes "this will cause data loss" from "this could be worded better."
6. **Speed matters.** A timely review > a perfect review delivered after implementation started.
7. **Shared understanding is the product.** Success = the implementing team knows exactly what to build.
