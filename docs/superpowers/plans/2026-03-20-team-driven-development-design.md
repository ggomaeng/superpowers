# Team-Driven Development — Design Document

## Overview

A new execution option for superpowers that uses Claude Code's native Agent Teams feature to spawn specialist teammates per task, with adversarial debate and a scoring rubric with convergence loops.

**Status:** Approved design, pending implementation
**Target:** PR to https://github.com/obra/superpowers

---

## Problem

After planning, superpowers offers two execution paths:

1. **Subagent-Driven Development (SDD)** — fresh subagent per task, sequential two-stage review (spec compliance → code quality)
2. **Inline Execution** — execute in current session with batch checkpoints

Both use subagents, which only report back to the caller. They cannot communicate with each other, debate findings, or challenge each other's conclusions. Review is sequential and binary (pass/fail).

## Solution

**Team-Driven Development** — a 3rd execution option that uses Agent Teams (multiple Claude Code instances with inter-agent communication) to provide:

- Parallel multi-angle review with adversarial debate
- Numeric scoring rubric with configurable thresholds
- Convergence loops until quality criteria are met
- User-configurable specialist dimensions

## How It Differs from SDD

| | SDD | Team-Driven Development |
|---|---|---|
| **Primitive** | Subagents (report back only) | Agent Teams (inter-agent communication) |
| **Review** | Sequential: spec → quality | Parallel: all specialists simultaneously |
| **Quality gate** | Pass/fail per reviewer | Numeric scoring rubric (1-10) with thresholds |
| **Iteration** | Fix loop per reviewer | Convergence loop until rubric met |
| **Communication** | Implementer ↔ controller only | Teammates debate and challenge each other |
| **Configurable** | Fixed 2 review stages | 4 core dimensions + user-defined extras |
| **Token cost** | Lower | Higher (tradeoff for quality) |

**Prerequisite:** Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` enabled.

---

## Core Flow

### Execution Handoff (in `writing-plans`)

After saving the plan, present three options:

```
"Plan complete and saved. Three execution options:

1. Subagent-Driven (recommended for most tasks)
   - Fresh subagent per task, two-stage review, lowest token cost

2. Inline Execution
   - Execute in this session, batch with checkpoints

3. Team-Driven (highest quality, highest token cost)
   - Agent teams with specialist teammates per task
   - Scoring rubric with convergence loops
   - Adversarial debate between reviewers
   - Requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS

Which approach?"
```

Only if user picks option 3, prompt for rubric configuration.

### Rubric Configuration (only for Team-Driven)

```
"Team-Driven selected. Let's configure your quality rubric.

4 core dimensions are always included:
- Spec Compliance (≥ 8)
- Code Quality (≥ 7)
- Test Coverage (≥ 7)
- Correctness (≥ 8)

What additional quality dimensions matter for this project?
(e.g., security, performance, accessibility, i18n)

Or 'none' for core only."
```

Thresholds confirmed and written into plan header.

### Plan Header Extension

```markdown
**Quality Rubric:**

| Dimension | Threshold | Teammate Role | Notes |
|-----------|-----------|---------------|-------|
| Spec Compliance | ≥ 8 | Spec Reviewer | Core (always present) |
| Code Quality | ≥ 7 | Code Quality Reviewer | Core (always present) |
| Test Coverage | ≥ 7 | Test Analyst | Core (always present) |
| Correctness | ≥ 8 | Correctness Reviewer | Core (always present) |

**Loop Config:**
- Max rounds per task: 3
- Max rounds project gate: 2
- Escalate to human on: threshold not met after max rounds
```

### Per-Task Flow

```
Lead reads plan, extracts all tasks, creates shared task list
       ↓
Per Task:
  Lead spawns agent team (4+ teammates):
    ├── Implementer       — writes code, tests, commits
    ├── Spec Reviewer     — verifies against plan spec (scores 1-10)
    ├── Code Quality      — reviews code quality (scores 1-10)
    ├── Test Analyst      — validates test coverage & edge cases (scores 1-10)
    ├── Correctness       — runs code, checks integration (scores 1-10)
    └── [Custom]          — user-defined specialists from rubric config
           ↓
  Phase 1: Implementation
    Implementer works. Other teammates can message implementer
    with early concerns during implementation.
           ↓
  Phase 2: Parallel Review + Debate
    All reviewers work simultaneously on committed code.
    They message each other directly with findings.
    Adversarial debate surfaces disagreements.
           ↓
  Phase 3: Scoring
    Each reviewer scores their rubric dimension (1-10).
    Lead collects scores, checks against thresholds.
           ↓
  Phase 4: Convergence Loop (if any dimension below threshold)
    - Implementer gets specific feedback on failing dimensions
    - Fixes issues
    - Reviewers re-score ONLY failing dimensions
    - Repeat until all dimensions ≥ threshold
    - Max 3 rounds before escalating to human
           ↓
  Task complete → shut down team → next task
       ↓
After all tasks:
  Project-Level Gate:
    Spawn integration team to score entire codebase:
    - Cross-task consistency
    - Integration correctness
    - Architectural coherence
    Loop until project-level rubric passes (max 2 rounds)
       ↓
  Use superpowers:finishing-a-development-branch
```

---

## File Structure

### New Skill: `skills/team-driven-development/`

| File | Purpose |
|---|---|
| `SKILL.md` | Main skill documentation |
| `implementer-prompt.md` | Implementer teammate — builds code, receives real-time feedback from reviewers |
| `spec-reviewer-prompt.md` | Spec compliance reviewer — scores 1-10, debates with other reviewers |
| `code-quality-reviewer-prompt.md` | Code quality reviewer — scores 1-10, debates with other reviewers |
| `test-analyst-prompt.md` | **New role** — evaluates test coverage, edge cases, test design quality |
| `correctness-reviewer-prompt.md` | **New role** — runs code, checks integration, verifies it works |
| `rubric-scorer-prompt.md` | Template for lead to collect and evaluate scores |
| `custom-reviewer-prompt.md` | Parameterized template for user-defined dimensions |

### Modified Skill: `skills/writing-plans/SKILL.md`

- Add option 3 (Team-Driven) to execution handoff
- Add rubric configuration prompt (only when Team-Driven chosen)
- Update plan header to include optional Quality Rubric section

---

## Teammate Roles

### Implementer
Same core role as SDD implementer, but:
- Receives real-time messages from reviewers during implementation
- Can respond to early concerns before committing
- Reports same statuses: DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, BLOCKED

### Spec Reviewer
Same focus as SDD spec reviewer, but:
- Outputs numeric score (1-10) with justification
- Messages other teammates with findings for debate
- Can challenge other reviewers' assessments

### Code Quality Reviewer
Same focus as SDD code quality reviewer, but:
- Outputs numeric score (1-10) with justification
- Messages other teammates with findings for debate

### Test Analyst (New)
- Evaluates test coverage and quality
- Checks for missing edge cases
- Validates test design (not just "tests exist" but "tests are meaningful")
- Scores 1-10

### Correctness Reviewer (New)
- Actually runs the code / tests
- Checks integration with rest of codebase
- Verifies the implementation works end-to-end
- Scores 1-10

### Custom Reviewer (Parameterized)
- Generic template parameterized with dimension name, description, and threshold
- Examples: Security, Performance, Accessibility, i18n

---

## Convergence Loop Design

The loop is **event-driven**, not interval-based:

```
Round 1:
  - Spawn team, implementation, parallel review, collect scores
  - Any dimension below threshold? → Round 2

Round 2:
  - Message implementer with specific failing dimensions + reviewer feedback
  - Implementer fixes
  - Re-score ONLY failing dimensions (don't re-review passing ones)
  - Still below threshold? → Round 3

Round 3 (max):
  - Same as Round 2
  - Still failing? → Escalate to human:
    "Task N stuck after 3 rounds.
     Failing: [dimension] ([score]/[threshold]).
     Reviewer feedback: [...]
     Options: adjust threshold, provide guidance, or skip"
```

Project-level gate follows the same pattern with max 2 rounds.

---

## Benchmark Plan

Run the same representative task through both SDD and Team-Driven to produce comparison data for the PR:

| Metric | How to measure |
|---|---|
| Token usage | Total tokens across all agents/teammates |
| Rubric scores | Final scores on all 4 core dimensions |
| Rounds to converge | Loop iterations per task |
| Issues caught | Count of spec gaps, quality issues, test gaps |
| Issues caught early | Issues found during implementation via teammate messages |
| Wall clock time | Total elapsed time |
| Review cycles | Number of fix → re-review loops |

Results included in PR body as a comparison table.

---

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** — Set up isolated workspace before starting
- **superpowers:writing-plans** — Creates the plan (modified to offer 3rd option)
- **superpowers:finishing-a-development-branch** — Complete development after all tasks

**Teammates should use:**
- **superpowers:test-driven-development** — Follow TDD for implementation

**Alternative workflows:**
- **superpowers:subagent-driven-development** — Lower cost, sequential review
- **superpowers:executing-plans** — Inline execution with checkpoints

---

## Design Decisions

1. **Agent Teams over subagents** — Inter-teammate communication enables adversarial debate, which produces higher quality than sequential isolated review
2. **4 fixed core + configurable extras** — Core dimensions cover the essentials; extras let users tailor to project needs without overwhelming default case
3. **Numeric scoring (1-10) with thresholds** — More granular than pass/fail; thresholds make quality gates objective and configurable
4. **Event-driven convergence (not interval-based)** — Score results trigger next round; intervals waste time or fire prematurely
5. **Re-score only failing dimensions** — Efficient; don't re-review what already passed
6. **Max 3 rounds then escalate** — Prevents infinite loops; human judgment for stuck cases
7. **Rubric in plan header** — Configuration travels with the plan; set once during planning, used during execution
8. **Project-level gate after all tasks** — Catches integration issues that per-task review cannot see
