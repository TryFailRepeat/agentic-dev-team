---
name: compliance-scope
description: Guides the Architect through assessing existing codebase compliance when working on established projects. Defines how to classify compliance violations found during codebase survey, what triggers human escalation vs. what can be fixed in-cycle, and how to communicate scope decisions to the Tech Lead.
---

# Compliance Scope Assessment

Existing projects may not fully follow the team's mandatory architecture principles (DDD, Onion Architecture, Independent Definitions, etc.). Forcing full compliance during a feature cycle introduces regression risk and scope creep. The goal is **targeted improvement, not wholesale refactoring**.

This skill defines how to classify violations you find and what must be escalated to the Tech Lead before proceeding.

---

## When to Run

After surveying the existing codebase (before designing the new feature). Every time you work on an existing project — even if you believe the codebase is clean. What looks clean from a distance often has hidden coupling.

---

## Classification Rules

For every compliance violation you find in the codebase, classify it into one of three categories:

### Category 1 — Fix in cycle (no escalation needed)

All of the following must be true:
- The file containing the violation **must be modified anyway** to implement this feature
- The fix is small — contained within the lines already being changed (not a restructure of the file)
- Fixing it does not require modifying other files outside the feature boundary
- No regression risk to existing behavior (the change is additive or purely structural within the modified block)

These are low-risk improvements. Apply them as part of the normal implementation without asking.

### Category 2 — Escalate to Tech Lead → Human

Escalate when **any** of the following is true:
- Fixing the violation requires touching files **outside the feature boundary** (files that wouldn't otherwise need to change)
- The new feature **cannot be written to comply** because the surrounding existing code would require significant restructuring first
- The fix introduces a meaningful regression risk — it changes behavior in paths not covered by the current task
- The fix is a structural change (splitting a class, moving a module, renaming domain terms across files)
- You find yourself wanting to refactor a file that already works, just to make the new code cleaner

Do not proceed with the design until you receive a human decision on each escalated item. Surface them all at once before starting the spec.

### Category 3 — Defer to backlog (log it, do not act)

Use when:
- The violation is in code **completely unrelated** to this feature — no path exists from the new feature to this code
- Fixing it now would be pure technical debt cleanup with no connection to the current task

Log it in the spec's Compliance Scope section. The Retrospective will include it in the improvement backlog.

---

## Escalation Format

When you need a human decision, report to the Tech Lead using this format for each item that requires escalation. Report all items together before proceeding — do not surface them one by one.

```
## Compliance Scope Decision Required

**Feature being designed:** <feature name>

### Decision <N>: <short title>

**Where:** `path/to/file.ext` — <which lines / which class>
**Violation:** <which principle is violated and how>
**Why it matters for this feature:** <does the new code have to integrate with this? does it propagate the violation?>

**Option A — Fix in this cycle**
<What would need to change, what files, rough size of change>
Blast radius: <what existing behavior could be affected>

**Option B — Defer**
<What this means: the new code is written knowing this violation exists, possibly working around it>
Risk of deferring: <does the violation compound? does the new feature make it harder to fix later?>

**My recommendation:** Option A / Option B — <one sentence reasoning>
```

The Tech Lead will present this to the human and return their decision before you continue designing.

---

## Intentional Deviation Format

Sometimes the human approves **not** fixing a violation — and you must write new code that knowingly does not fully comply, because restructuring the surrounding code is out of scope. When this happens:

- Note it explicitly in the spec's Compliance Scope section (see below)
- Note the constraint in the affected module boundaries in the spec, so the Backend Engineer knows
- Do not pretend the new code fully complies when it does not — the Code Reviewer will check the spec

An approved intentional deviation is not a failure. It is a documented scope decision.

---

## Spec Section Format

Add this section to every spec where the codebase survey found compliance issues:

```markdown
## Compliance Scope

### In-Cycle Fixes
Violations being fixed as part of this cycle (Category 1 decisions).

| File | Violation | Fix Applied |
|---|---|---|
| `path/to/file.ext` | <principle violated> | <what was changed> |

### Approved Deviations
Human-approved decisions to not fix a violation in this cycle.

| File | Violation | Reason | Consequence |
|---|---|---|---|
| `path/to/file.ext` | <principle violated> | <why it's out of scope> | <what this means for future work> |

### Deferred to Backlog
Violations noted but not in scope for this cycle.

| File | Violation | Priority |
|---|---|---|
| `path/to/file.ext` | <principle violated> | High / Medium / Low |
```

If the codebase is clean or this is a greenfield feature with no existing violations, write: "No existing compliance violations found."

---

## The Code Reviewer's Contract

This classification matters for the Code Reviewer. They will read the Compliance Scope section of the spec. Their rules:

- **In-Cycle Fixes**: reviewed normally — the fix must be correct
- **Approved Deviations**: the Code Reviewer will not flag these as findings (they were approved)
- **Deferred items**: the Code Reviewer will not flag pre-existing violations in untouched code as Critical or High. Noting them as Low is allowed.

If you do not fill in the Compliance Scope section, the Code Reviewer has no way to distinguish a new violation from a pre-existing approved one. Fill it in.
