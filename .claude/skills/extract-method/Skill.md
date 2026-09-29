---
name: extract-method
description: Refactoring guidance for decomposing long methods into smaller, focused ones using Extract Method (Fowler) and Composed Method (Beck) patterns, grounded in SRP and Clean Architecture layer hygiene.
---

## Purpose

Use this skill when a method is too long, mixes levels of abstraction, or does more than one thing. It codifies the **Extract Method** refactoring (Martin Fowler, *Refactoring*) and the **Composed Method** pattern (Kent Beck, *Smalltalk Best Practice Patterns*), applied within SOLID and Clean Architecture constraints.

Load this skill when:
- A method exceeds ~20 lines
- A method mixes high-level intent with low-level implementation detail
- A code block inside a method can be named meaningfully
- A method has multiple `try/catch` blocks, nested loops, or multi-step logic
- You are reviewing code for SRP violations

---

## Core Concepts

### Extract Method (Fowler)

Replace a code block with a call to a named method that expresses **what** the block does, not **how**.

**When to extract:**
- The block can be given a name that describes its intent clearly
- The block is used in more than one place (eliminate duplication)
- The block is at a different level of abstraction than the surrounding code
- The block needs a comment to explain it — the comment is a signal that it deserves a name

**How to name the extracted method:**
- Use the intent, not the mechanism: `resolveWatermark()` not `getDateFromRepo()`
- The name should make the call-site read like prose
- If you struggle to name it, the extraction boundary may be wrong

---

### Composed Method (Beck)

> *"Divide your program into methods that perform one identifiable task. Keep all of the operations in a method at the same level of abstraction."*
> — Kent Beck, *Smalltalk Best Practice Patterns*

A well-composed method:
- Has one level of abstraction throughout — either high-level calls or low-level operations, never both mixed
- Is short enough to fit on one screen
- Makes the algorithm readable without reading its sub-methods

**Level of abstraction test:** If a method contains both a low-level loop and a call to a named operation, it is mixing levels. Extract the loop.

**Signs a method violates Composed Method:**
- You need to scroll to understand it
- Comments separate logical sections — each section is a candidate for extraction
- Some lines read as intent and others as mechanism

---

## SOLID Connection

### Single Responsibility Principle

A method has one responsibility when it has one reason to change. Ask: *"If the business rule for X changes, which methods need to change?"* Each should be exactly one.

Long methods accumulate responsibilities silently. Extraction makes responsibilities explicit and individually testable.

### Open/Closed at the method level

Extracted methods are easier to override in subclasses or replace with strategy objects if behaviour needs to vary — without touching the orchestrating method.

---

## Clean Architecture Connection

Extraction enforces **layer purity**. A use case method that mixes application orchestration with infrastructure detail (e.g. pagination logic, raw DB calls) is a layer violation in disguise. Extracting infrastructure detail into infrastructure-layer methods and keeping the use case calling only named operations preserves the boundary.

**Rule:** If an extracted method name sounds like infrastructure (`oDataPages`, `upsertChunk`), it belongs in the infrastructure layer. If it sounds like application logic (`syncSource`, `resolveWatermark`), it belongs in the use case.

---

## Checklist Before Finishing

- [ ] Every method fits on one screen (~20 lines max)
- [ ] Every method name expresses intent, not mechanism
- [ ] All lines in a method operate at the same level of abstraction
- [ ] No section of a method is explained only by a comment — the comment became a method name
- [ ] Extracted methods are individually testable in isolation
- [ ] No extracted method reaches into a layer it should not know about
