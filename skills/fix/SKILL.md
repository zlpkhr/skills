---
name: fix
description: >
  Use this skill when fixing a bug, addressing broken behavior, or
  investigating why something doesn't work as expected. Walks through
  reproducing the failure, diagnosing the root cause, planning the smallest
  correct change, applying it in the codebase's existing style, and pinning
  it with a regression test. Trigger this skill when the user reports an
  error, a regression, incorrect output, a flaky test, or asks to "fix,"
  "debug," "make this work," or "figure out why X is happening" — even if
  they don't name a specific bug or use the word "fix."
---

# Fix

Smallest correct change to broken behavior.

## Workflow

1. **Reproduce the bug.** See it happen. If you can't reproduce it, you don't understand it yet — don't propose a fix.

2. **Diagnose the root cause.** Find the actual cause, not a symptom that correlates. The fix is in the cause; if you can't name the cause, you're not done diagnosing.

3. **Plan the smallest correct change.** The diff should be the minimum needed to address the cause. Don't add defensive code beyond what fixes *this* bug — "just in case" guards multiply.

4. **Make the change.** Match the existing code style. Don't take the bug as license to refactor adjacent code; surface what you saw, but don't fix it silently.

5. **Pin it with a regression test.** A test that fails before the fix and passes after. Follow the codebase's testing convention — don't introduce a framework as part of a fix.
