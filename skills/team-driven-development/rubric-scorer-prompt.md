# Rubric Scorer Template

This is NOT a teammate prompt. The lead uses this template to collect scores
from all reviewer teammates, evaluate against thresholds, and decide whether
to loop or proceed.

## Collecting Scores

After all reviewers have reported, compile the rubric:

```
┌─────────────────┬───────┬───────────┬────────┐
│ Dimension       │ Score │ Threshold │ Status │
├─────────────────┼───────┼───────────┼────────┤
│ Spec Compliance │  X/10 │    ≥ T    │ ✅/❌  │
│ Code Quality    │  X/10 │    ≥ T    │ ✅/❌  │
│ Test Coverage   │  X/10 │    ≥ T    │ ✅/❌  │
│ Correctness     │  X/10 │    ≥ T    │ ✅/❌  │
│ [Custom 1]      │  X/10 │    ≥ T    │ ✅/❌  │
│ [Custom 2]      │  X/10 │    ≥ T    │ ✅/❌  │
└─────────────────┴───────┴───────────┴────────┘
```

## Decision Logic

**All dimensions ≥ threshold:**
→ Task passes. Shut down team. Mark task complete. Move to next task.

**Any dimension below threshold AND round < max:**
→ Message implementer with:
  1. Which dimensions failed
  2. Each failing reviewer's justification and specific issues
  3. Priority order (lowest relative score first)
→ Tell ONLY failing-dimension reviewers to re-score after fixes
→ Do NOT re-score passing dimensions

**Any dimension below threshold AND round = max:**
→ Escalate to human with:
  1. Full rubric table
  2. Each failing dimension's history across rounds
  3. Options: adjust threshold, provide guidance, skip, take over

## Score Conflict Resolution

If two reviewers report conflicting findings about the same code:
1. Present both findings to all reviewers
2. Let them debate (1-2 messages each)
3. If no resolution, present conflict to human for decision

## Round Tracking

Keep track of convergence progress:

```
Task N — Round 2/3
Previously failing: Spec Compliance (6→8 ✅), Test Coverage (5→6 ❌)
Still failing: Test Coverage (6/7)
Reviewer feedback: "Missing edge case for empty input, no error path tests"
```
