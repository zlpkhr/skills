---
name: extend
description: >
  Use this skill when adding a new feature, endpoint, field, screen, flag,
  or branch to existing code. Walks through finding the nearest similar
  existing piece as a template, copying its shape, adapting it for the new
  behavior, avoiding new dependencies or architectural patterns the codebase
  doesn't already use, and surfacing adjacent issues separately rather than
  expanding the diff. Trigger when the user asks to "add," "extend," "support
  also," "include," or otherwise grow existing functionality without
  redesigning it — even if they don't explicitly say which area of the
  codebase to touch.
---

# Extend

Add to existing code: a new feature, endpoint, field, screen, or branch in an existing flow. The shape and conventions of the surrounding code are already decided.

## Workflow

1. **Find the nearest similar piece of existing code.** The closest existing example — an endpoint, a model, a component — is the template. Read it before writing anything new.

2. **Copy its shape.** Same naming, same layering, same error handling, same tests. The cost of one inconsistent style across the codebase is higher than the cost of any single style being suboptimal.

3. **Adapt it for the new feature.** Change only what's needed for the new behavior. Don't introduce architectural patterns the codebase doesn't have — FSMs, layers, dependency injection, new build tooling — that's the `create` skill or an explicit refactor, not extension.

4. **Don't pull in new dependencies unless necessary.** If the codebase already does the thing another way, use that way. New dependencies fragment the codebase.

5. **Surface adjacent issues separately.** If you spot a real problem nearby, mention it in the change description or open a separate ticket. Don't expand the diff.
