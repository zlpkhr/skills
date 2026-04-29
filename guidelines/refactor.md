# Mode: Refactor

Deliberately reshape working code. The user explicitly asked for a quality change.

## Workflow

1. **Confirm what's in scope.** The diff should match the request — don't expand into adjacent code that "could also use cleanup."

2. **Decide what's worth changing.** Bad code well isolated behind a clean API is cheap — leave it. Spend the budget on coupling: code that leaks into other modules, repeated patterns showing up in multiple places (extract on the second occurrence, not the third), shapes that block the change you actually want to make.

3. **Ensure tests cover the existing behavior.** A refactor preserves observable behavior; tests verify that. If coverage is missing where you're touching, add it before the refactor, not after.

4. **Make incremental changes.** One shape change at a time. Big-bang rewrites almost always lose. Run the tests after each step.

5. **Don't change behavior.** If a test breaks but the externally-observable behavior didn't change, the test was wrong — it was peeking at internals or over-mocking. Such tests are evidence to delete or rewrite, not to undo the refactor.

Design rules from `create.md` are available where the refactor naturally lands — but only there. Don't introduce FSMs, layers, or polymorphism in code the refactor wasn't supposed to touch.
