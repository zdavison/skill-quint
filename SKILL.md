---
name: quint
description: >
  Quint formal specification language reference — constraints, code generation, verification, testing, debugging, and planning.
  TRIGGER when: working with .qnt files, Quint specs, formal specifications, model-based testing, invariants, witnesses, or the Choreo framework.
---

# Quint Specification Language Guide

You are working with **Quint**, a formal specification language for distributed protocols. This skill provides guidelines for writing, modifying, verifying, and debugging Quint specifications.

## When to load which reference

Read the relevant file(s) based on what you're doing:

| Task | Reference |
|------|-----------|
| Writing or reading Quint code | [quint-constraints.md](quint-constraints.md) — **always read first**, covers hard language limitations |
| Adding/modifying types, vars, actions, defs | [implementation.md](implementation.md) — insertion points, code generation, modification patterns |
| Handling parse or type errors during edits | [iteration.md](iteration.md) — error protocols, fix strategies, escalation |
| Evaluating or creating a refactor plan | [planning.md](planning.md) — quality matrices, checklists, approval criteria |
| Running witnesses, invariants, or tests | [verification.md](verification.md) — execution protocols, result interpretation |
| Debugging failed `run` definitions | [test-debugging.md](test-debugging.md) — error location ≠ failure point, frame counting |
| Writing tests for Choreo framework specs | [choreo-test-patterns.md](choreo-test-patterns.md) — `.perform()`, `.with_cue()`, `.step_with()` syntax |

## Critical rules (always apply)

1. **No string manipulation** — Quint strings are opaque; use sum types or structured data instead
2. **No nested pattern matching** — match one level at a time with sequential `match`
3. **No destructuring** — use explicit field access (`.field`, `._1`, `._2`)
4. **No mutable variables** — use state variables with `'` suffix for transitions
5. **No loops** — use set comprehensions, `.map()`, `.fold()`, recursion
6. **Witnesses: VIOLATED = good** (liveness confirmed); **Invariants: SATISFIED = good** (safety holds)
7. **Error location ≠ failure point** in `run` tests — always use `--verbosity=3` to find actual failure
8. **Always use `--match`** with `quint test` — never run without specifying which tests
9. **Typecheck after every change** — `quint typecheck <file>` must pass before proceeding
10. **Max 3 iteration attempts** on any error before escalating to the user
