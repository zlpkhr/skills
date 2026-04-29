---
name: refactor
description: >
  Use this skill when deliberately reshaping working code to improve its
  quality without changing observable behavior. Walks through confirming
  scope, deciding what's worth changing (coupling and repeated patterns,
  not isolated cosmetic ugliness), ensuring tests cover existing behavior,
  making incremental changes one shape change at a time, and verifying
  behavior didn't drift. Trigger when the user says "refactor," "clean up,"
  "improve this code," "make this better," "extract," "consolidate," or
  asks for structural changes to working code.
---

# Refactor

Deliberately reshape working code. The user explicitly asked for a quality change.

## Workflow

1. **Confirm what's in scope.** The diff should match the request — don't expand into adjacent code that "could also use cleanup."

2. **Decide what's worth changing.** Bad code well isolated behind a clean API is cheap — leave it. Spend the budget on coupling: code that leaks into other modules, repeated patterns showing up in multiple places (extract on the second occurrence, not the third), shapes that block the change you actually want to make.

3. **Ensure tests cover the existing behavior.** A refactor preserves observable behavior; tests verify that. If coverage is missing where you're touching, add it before the refactor, not after.

4. **Make incremental changes.** One shape change at a time. Big-bang rewrites almost always lose. Run the tests after each step.

5. **Don't change behavior.** If a test breaks but the externally-observable behavior didn't change, the test was wrong — it was peeking at internals or over-mocking. Such tests are evidence to delete or rewrite, not to undo the refactor.

Some refactors introduce new structure where there wasn't any. In those parts, the design discipline from greenfield work applies — but only there. Don't introduce FSMs, layers, or polymorphism in code the refactor wasn't supposed to touch.
