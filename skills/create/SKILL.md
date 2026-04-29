---
name: create
description: >
  Use this skill when starting a new module, file, service, component, or
  library from scratch — greenfield code where the shape isn't already
  decided. Walks through identifying the domain vocabulary, designing the
  public API from the consumer's perspective, choosing the structure (pure
  core vs effectful edges, state machines for stateful processes,
  polymorphism over flag-forks, no dependency cycles, closed-by-default),
  implementing the simplest version that satisfies the API, and deferring
  irreversible decisions to the last responsible moment. Trigger when the
  user says "build a new," "let's start a," "I want a fresh," "from
  scratch," or describes a piece of software that doesn't yet exist.
---

# Create

Greenfield: a new module, file, service, or component. You're choosing the shape.

## Workflow

1. **Map the domain.** Identify the vocabulary the business uses — charge, ship, archive, union — and use those names in the code. A non-programmer who knows the domain should be able to follow the high-level flow.

2. **Design the API from the consumer's perspective.** Sketch how callers will use this module before writing internals. Once anything depends on an API it becomes legacy you support forever.

3. **Choose the structure.** Before implementation, decide:

    - **Pure core, effectful edges.** Push I/O to the outermost layer; keep transformations in the middle as pure functions.

      ```ts
      // bad — parser also reads from disk
      class Parser {
        constructor(path) { this.data = fs.readFileSync(path, "utf8"); }
        parse() { /* ... */ }
      }

      // good — pure parser, effects orchestrated outside
      const raw = await fs.readFile(path, "utf8"); // effect (edge)
      const ast = parse(raw);                       // pure (core)
      await fs.writeFile(out, render(ast));         // effect (edge)
      ```

    - **Stateful processes are state machines.** Anything with discrete states (orders, payments, users) gets named states and explicit transitions. Hidden state spread across booleans causes correlation bugs. Even two states today should be modeled as named states — when a third appears, a lazy developer will add another boolean instead of refactoring.

      ```ts
      type State = "pending" | "active" | "blocked" | "deleted";
      const transitions = {
        pending: { confirm: "active" },
        active:  { block: "blocked", remove: "deleted" },
        blocked: { unblock: "active" },
        deleted: {},
      };
      ```

    - **Polymorphism, not flag-forks.** N interchangeable algorithms producing the same result is one interface with N implementations, not a switch on a string field.

      ```ts
      interface PaymentProvider {
        charge(amount: number, token: string): Promise<Receipt>;
      }
      class StripeProvider implements PaymentProvider { /* ... */ }
      class PaypalProvider implements PaymentProvider { /* ... */ }
      ```

    - **No cycles in the dependency graph.** A module graph with cycles is a distributed monolith.

    - **Closed by default.** Whitelist what's allowed; deny what isn't. Fail-safe: a crash, dropped connection, or missing config should not unlock anything.

4. **Implement the simplest version that satisfies the API.** Choose simple, not familiar — simple is objective (few entangled parts), familiar is what you've seen before. Constants beat variables; pure functions beat mutable state. Prefer declarative over imperative for structure (HTML, configs, CLI specs, state machines). Return objects, not sentinels — keep call chains usable.

5. **Defer irreversible decisions to the last responsible moment.** Postpone choices that calcify before you have the information to make them well. When you reach for a pattern, classify the problem first (locking, polymorphism, delivery semantics, idempotency) — the pattern follows from the class.

The first instance of a pattern in a codebase becomes the template that gets copy-pasted. Invest extra time getting it right.
