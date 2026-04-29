# Mode: Fix

Smallest correct change to broken behavior.

## Workflow

1. **Reproduce the bug.** See it happen. If you can't reproduce it, you don't understand it yet — don't propose a fix.

2. **Diagnose the root cause.** Find the actual cause, not a symptom that correlates. The fix is in the cause; if you can't name the cause, you're not done diagnosing.

3. **Plan the smallest correct change.** The diff should be the minimum needed to address the cause. Don't add defensive code beyond what fixes *this* bug — "just in case" guards multiply.

4. **Make the change.** Match the existing code style. Don't take the bug as license to refactor adjacent code; surface what you saw, but don't fix it silently.

5. **Pin it with a regression test.** A test that fails before the fix and passes after. Follow the codebase's testing convention — don't introduce a framework as part of a fix.
