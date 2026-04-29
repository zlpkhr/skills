# Mode: Extend

Add to existing code: a new feature, endpoint, field, screen, or branch in an existing flow. The shape and conventions of the surrounding code are already decided.

## Workflow

1. **Find the nearest similar piece of existing code.** The closest existing example — an endpoint, a model, a component — is the template. Read it before writing anything new.

2. **Copy its shape.** Same naming, same layering, same error handling, same tests. The cost of one inconsistent style across the codebase is higher than the cost of any single style being suboptimal.

3. **Adapt it for the new feature.** Change only what's needed for the new behavior. Don't introduce architectural patterns the codebase doesn't have — FSMs, layers, dependency injection, new build tooling — that's `create` mode or an explicit refactor, not extension.

4. **Don't pull in new dependencies unless necessary.** If the codebase already does the thing another way, use that way. New dependencies fragment the codebase.

5. **Surface adjacent issues separately.** If you spot a real problem nearby, mention it in the change description or open a separate ticket. Don't expand the diff.
