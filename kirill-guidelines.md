# Programming Guidelines from Kirill Mokevnin

A synthesis of practical advice from a series of talks by Kirill Mokevnin (Hexlet). Written for humans. Examples are translated from Ruby to JavaScript / TypeScript.

The thread running through everything: write code that matches how the task is described in the real world, push complexity out of the inner loop, and prefer boring discipline over clever exceptions.

---

## 1. Mental modeling

### Match code to the mental model of the task
Names, functions, and structure should mirror the language the business or domain uses. A non-programmer who knows the domain should be able to follow the high-level flow.

```ts
// bad — implementation language
session.userId = user.id;

// good — domain language
authorize(session, user);
```

### Code is read more than written
A snippet pasted into chat with no context should be understandable. If a peer can't say what it does in business terms, the code is hiding intent.

```ts
// hides intent
const keys = [...new Set([...Object.keys(a), ...Object.keys(b)])];

// reveals intent
const keys = unionKeys(a, b);
```

### Use the vocabulary of the problem
If the operation has a real name (union, intersection, set, graph, projection), use that name. It forces correct framing and lets readers pattern-match instantly.

### Prefer declarative over imperative when describing structure
For HTML, configs, CLI specs, state machines, route tables — describe *what*, not *how*. Imperative wiring grows long and unreadable.

```ts
// imperative DOM
const div = document.createElement("div");
div.className = "card";
div.appendChild(h2);

// declarative
const card = <div className="card"><h2>{title}</h2></div>;
```

### Treat code as an executable spec
"Readable" isn't enough. After reading the code, you should be able to reconstruct the *task* it solves, not just describe what each line does.

---

## 2. Simple vs easy

### Choose simple, not familiar
"Easy" is subjective (familiar). "Simple" is objective (few entangled parts). Choose simple even when it feels harder. Constants are simpler than variables; pure functions are simpler than mutable objects; explicit branches are simpler than implicit ones.

```ts
// easy but complex (mutation, time-dependence)
let total = 0;
for (const n of nums) total += n;

// simple
const total = nums.reduce((a, n) => a + n, 0);
```

### Habits compound
Apply discipline everywhere, not only "where it matters." If you let yourself be sloppy in trivial spots, you'll be sloppy in critical ones too. Coding is motor memory.

---

## 3. Naming

### Names must stay truthful as code evolves
If the behavior changes, rename every callsite. Never invert internals while keeping the old name.

```ts
// lying name
isActive() { return !this.active; }

// rename instead
isInactive() { return !this.active; }
```

### Functions are verbs, not nouns
A function name should describe an action: `parseConfig`, `buildTree`, `chargeCard`. Noun names mislead readers about call semantics.

### Plural for collections, singular for one
Even with static types, a wrong-number name forces re-derivation.

```ts
// bad
const user = await db.users.findAll();

// good
const users = await db.users.findAll();
```

### Paste-test names
When a name feels off, paste only the name into chat and ask a peer what it means. If their guess diverges from your intent, rename it.

### Replace compound conditions with predicates
Any boolean expression that operates on one aggregate should become a named predicate on that aggregate.

```ts
// bad
if (user.company && user.subscription.active && user.balance > 0) { ... }

// good
class User {
  canPublish() {
    return this.company && this.subscription.active && this.balance > 0;
  }
}
if (user.canPublish()) { ... }
```

---

## 4. Comments

### Don't restate code
If a comment explains what the code does, the code needs a better name. Extract a function instead.

### Comment non-obvious context
Legacy quirks, system constraints, business reasons that can't be expressed in code — these belong in comments. They are the comments that age well.

```ts
// Legacy: pre-2019 accounts have role=null but are effectively admins.
if (user.role === "admin" || user.legacyAdmin) { ... }
```

### Use TODO and FIXME inline
Mark deferred work and known bugs in the code, even if you also have a ticket. Out-of-code markers (Jira links) get ignored. Reserve `FIXME` for real bugs.

---

## 5. Functions and abstractions

### A function is an abstraction, not a deduplication tool
Extract to *name* an operation and raise the level of abstraction. A single-call-site extraction is a good extraction. Once an operation has a name and a boundary, the inside no longer matters and can be rewritten freely.

### One level of abstraction per function
Don't mix high-level domain steps with low-level I/O or string manipulation in the same function. Push lower levels into helpers.

```ts
// bad — mixed levels
async function publishPost(input) {
  const html = marked.parse(input.markdown);
  const slug = input.title.toLowerCase().replace(/\s+/g, "-");
  await db.query("INSERT INTO posts ...", [slug, html]);
  await fetch("https://hooks.slack.com/...", { method: "POST", body: ... });
}

// good — same level throughout
async function publishPost(input) {
  const post = renderPost(input);
  await postsRepo.save(post);
  await notifier.announce(post);
}
```

### Eliminate states early with guard clauses
At the top of a function, exit on edge cases. Each line below should handle fewer possible states than the line above.

```ts
function diff(a, b, key) {
  if (a[key] === undefined) return { type: "added", key, value: b[key] };
  if (b[key] === undefined) return { type: "removed", key, value: a[key] };
  if (isObj(a[key]) && isObj(b[key])) return { type: "nested", children: diff(a[key], b[key]) };
  if (a[key] === b[key]) return { type: "unchanged", key, value: a[key] };
  return { type: "changed", key, before: a[key], after: b[key] };
}
```

### Don't nest deeply — name the intermediates
If an expression chains many calls or nests more than two levels, break it up with named locals. They double as inline documentation.

```ts
// bad
sendEmail(formatBody(translate(loadTemplate(user.lang), pickKey(user))), user);

// good
const template = loadTemplate(user.lang);
const key = pickKey(user);
const body = formatBody(translate(template, key));
sendEmail(body, user);
```

### Extract on the second occurrence, not the third
The classic rule of three is too lenient on a real team. If you already recognize a pattern from past work, extract it on the second occurrence, or write it as a helper the first time. On a large team you won't see the third copy in time.

### Write the first occurrence right
The first instance of a pattern becomes the template that gets copy-pasted. Invest extra time on it. Sloppiness propagates; so does quality.

---

## 6. State, mutation, and effects

### Push side effects to the edges; keep the core pure
Most code is conceptually pure (input → output). Mixing I/O into transformation code prevents reuse and breaks the moment requirements shift (e.g. "now read from network instead of disk").

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

### Command-Query Separation
A function either changes state (command) or returns a value (query). Never both. A predicate like `isValid` must not write to the database; a `save` must not return business data the caller needs to decode.

```ts
// bad — query with side effects
class User {
  isValid() {
    this.avatar = null; // hidden mutation
    return this.email.includes("@");
  }
}

// good
class User {
  isValid() { return this.email.includes("@"); }
  normalize() { this.avatar = null; }
}
```

This applies to HTTP too: GET must not mutate. The classic disaster is a search-engine crawler following GET-style "delete" links and wiping the database.

### `const` does not freeze the object
Mutating a shared object via `push`/`pop`/`splice` causes spooky action at a distance. Don't assume immutability just because the binding is `const`.

### Match operation semantics to the data structure
Don't use `pop` on something you treat as a list — `pop` mutates and signals stack semantics. Use `arr.at(-1)` if you just want the last element.

### Treat shared mutable state as a deliberate design event
Concurrency, races, and most "hard bugs" only exist where there is shared mutable state. Adding it should be a conscious decision, not an accident.

---

## 7. Conditions and states

### Make `else` explicit
Don't rely on "everything not X" branches. Spell out the alternative even when there are only two cases today.

```ts
// bad — implicit "non-ru" means "en"
if (lang === "ru") return ru();
return en();

// good
if (lang === "ru") return ru();
if (lang === "en") return en();
throw new Error(`Unsupported lang: ${lang}`);
```

### Don't use raw strings as enums
Strings are unchecked: a typo silently fails. Use a constants object, an enum, or a typed union.

```ts
// bad
if (order.status === "shiped") { ... } // typo, never matches

// good
const OrderStatus = { Pending: "pending", Shipped: "shipped", Cancelled: "cancelled" } as const;
type OrderStatus = typeof OrderStatus[keyof typeof OrderStatus];
if (order.status === OrderStatus.Shipped) { ... }
```

### Don't encode meaning in truthiness
Don't decide "is leaf" by absence of `children`, "is logged in" by absence of `null`, "is empty" by `!collection`. Make the structural fact explicit.

```ts
// bad
const isLeaf = (n: any) => !n.children;

// good
type Node = { kind: "leaf"; value: V } | { kind: "branch"; children: Node[] };
const isLeaf = (n: Node) => n.kind === "leaf";
```

### A collection is never null
If a value can be a list, give it `[]` when there are no items. Otherwise every callsite needs `if (xs && xs.length)`.

### Test the cause, not a correlated proxy
A condition should test the actual reason, not a value that happens to correlate today. Correlated proxies fail the moment the underlying mechanism changes.

```ts
// bad — confirms password by absence of confirmation token
if (!user.confirmationToken) validatePassword(user);

// good — confirms by the real cause
if (user.signupMethod === "email") validatePassword(user);
```

### Model stateful processes as finite state machines
Anything with discrete states (orders, articles, payments, users) should be an FSM with declared states and transitions. Hidden state spread across booleans causes correlation bugs and is impossible to reason about.

```ts
type State = "pending" | "active" | "blocked" | "deleted";
const transitions = {
  pending: { confirm: "active" },
  active:  { block: "blocked", remove: "deleted" },
  blocked: { unblock: "active" },
  deleted: {},
};
```

Even with two states today, model them as named states. When the third state appears, a lazy developer adds another boolean instead of refactoring.

Use a library (XState in JS) — it verifies reachability and generates predicates for free.

### Replace null branches with the Null Object pattern
Code littered with "if logged in / else guest" should use one boundary that produces a real or guest object implementing the same interface. Polymorphism, not branching.

```ts
interface User { name(): string; canEdit(id: string): boolean; }
class RealUser implements User { /* ... */ }
class GuestUser implements User {
  name() { return "Guest"; }
  canEdit() { return false; }
}

const currentUser: User = session ? new RealUser(session) : new GuestUser();
```

---

## 8. Errors and defaults

### Whitelist, not blacklist
Deny by default; explicitly allow. Blacklists silently miss new cases; mixed lists force readers to invert their mental model line by line.

```ts
// bad
const blocked = ["/admin/secret"];
if (!blocked.includes(path)) allow();

// good
const allowed = ["/", "/login", "/products"];
if (allowed.includes(path)) allow();
```

### Fail-safe by default
A train brakes when it loses pressure. Software components should behave the same: a crash, dropped connection, or missing config should not unlock anything. Require active signal to enter the unsafe state.

### Wire frontend and backend error collectors
Sentry / Bugsnag / equivalent on every surface, piped to a dedicated channel. Production breaks in ways tests can't reach.

### Make critical signals impossible to ignore
CI/build/test status on a wall monitor, not in email. Email gets filtered; a red wall is forcing.

### Fix the system, not the person
When something breaks, change the system so the same mistake can't recur. "Who failed to check?" produces fear and zero learning.

---

## 9. Polymorphism, patterns, and SOLID

### Polymorphism is the core OOP concept
In class-based languages, polymorphism is what does the work. Encapsulation and inheritance are secondary. Most GoF patterns — Strategy, Adapter, Decorator, Null Object — are the same trick.

```ts
interface PaymentProvider {
  charge(amount: number, token: string): Promise<Receipt>;
}
class StripeProvider implements PaymentProvider { /* ... */ }
class PaypalProvider implements PaymentProvider { /* ... */ }

async function pay(provider: PaymentProvider, amount, token) {
  return provider.charge(amount, token);
}
```

### LSP is about types, not classes
Liskov Substitution: a subtype cannot strengthen preconditions or weaken postconditions of its supertype. It is a discipline rule (type systems can't enforce it). It has nothing to do with parent/child classes.

```ts
// violates LSP: subtype rejects negatives the supertype accepted,
// so anyone holding Counter breaks when handed PositiveCounter.
class Counter { add(n: number) {} }
class PositiveCounter extends Counter {
  add(n: number) { if (n < 0) throw new Error("no negatives"); }
}
```

### Don't override standard behavior
Never change the semantics of built-in methods, operators, or framework hooks. Add a new, differently named method instead.

```ts
// bad — Array.prototype.push now does something custom
Array.prototype.push = function (x) { /* magic */ };

// good — different name for different behavior
class UniqueList { addUnique(x) { /* ... */ } }
```

### SOLID is overhyped
Single Responsibility, single level of abstraction, separation of concerns, and the Law of Demeter come up daily. SOLID makes a tidy acronym but several of its letters rarely matter. Don't quote rules you haven't actually applied.

### Classify the problem before reaching for patterns
The productive question is "which problem is this?" — locking, polymorphism, delivery semantics, idempotency, resource scheduling. The pattern follows from the class.

### Use Strategy instead of multiplying services
N interchangeable algorithms producing the same result is one process with one interface and N implementations. Not N microservices.

```ts
interface Decryptor {
  canHandle(file: Buffer): boolean;
  decrypt(file: Buffer): Buffer;
}
const decryptors: Decryptor[] = [aes, rsa, gpg];

function decrypt(file: Buffer): Buffer {
  const d = decryptors.find(d => d.canHandle(file));
  if (!d) throw new Error("no decryptor for this file");
  return d.decrypt(file);
}
```

---

## 10. API design

### Design the API first
Design the public interface from the consumer's perspective before writing internals. Once anything depends on an API it becomes legacy you support forever. TDD helps because it forces you to use the API before building it.

### Validate at the form / use case, not on the entity
Don't put context-dependent validation hooks inside the model. Validate at the boundary that knows the context.

### Return objects, not sentinels — keep call chains usable
Write code that lets the next person extend the result without rewriting earlier lines.

```ts
// bad — adding another assertion forces a rewrite
expect(findUser(id).name).toBe("Kirill");

// good — value is addressable
const user = findUser(id);
expect(user.name).toBe("Kirill");
expect(user.body).toBe("...");
```

### Don't postpone needed dependencies to "save weight"
If you're already reaching for a small piece of a library and you'll want more soon, pull it in now. The migration window closes silently and you carry duplicated, half-correct logic forever.

---

## 11. Testing

### Fewer tests, more behavior
Don't pursue unit-test-everything. Prefer integration tests that exercise real behavior; reach for unit tests only when needed. The goal is working software, not test count.

```ts
// bad — tests internals; breaks on refactor
expect(checkout._calculateTaxLine(item)).toBe(1.2);

// good — tests observable behavior
const result = await checkout.process(cart);
expect(result.total).toBe(13.2);
```

### A good test survives a refactor
If externally-observable behavior didn't change but the test broke, the test was wrong — it was peeking at internals or over-mocking.

### Know mocks vs stubs
A stub returns canned data to silence a collaborator. A mock asserts that an interaction happened. Most developers conflate them and over-mock. Use mocks only when the interaction itself is what you're testing (e.g. "we sent the welcome email").

```ts
// stub
const userRepo = { find: async () => ({ id: 1, name: "Ann" }) };

// mock
const mailer = { send: jest.fn() };
await signup.run(input, { userRepo, mailer });
expect(mailer.send).toHaveBeenCalledWith("welcome", input.email);
```

### Don't argue from authority you haven't read
Quoting Fowler / Martin / Beck is meaningless if you've only absorbed the slogans. Original authors often walk back their positions years later. Reason from first principles instead of cargo-culting decades-old advice.

---

## 12. Refactoring and decisions

### Defer decisions to the last responsible moment
Postpone irreversible architectural choices until you have the information to make them well. "Too early" reliably becomes "too late." Use the wait to gather usage data and let tooling mature.

### Don't pre-optimize
Worry about performance only after measurement. If the abstraction is right, an inefficient implementation can be swapped later without architectural cost. Real tech debt is the kind that grows non-linearly.

### Don't rewrite the whole thing
Big-bang rewrites of working systems almost always lose. Refactor incrementally. Netscape paused for 2–3 years to rewrite their browser and lost the entire market while the lights were off.

### Bad code well isolated is cheap
On code review, don't fight ugly internals if the function is pure and isolated behind a clean API. Spend the review budget on code that leaks into the rest of the system. Technical debt is paid through coupling.

### Don't borrow elite-team practices blindly
"We don't write tests" / "we trunk-commit" works for a champion team and breaks a normal one. A bodybuilder's training program will injure an amateur. Factor in your context.

### Don't let quality slip
Broken Windows: hold the baseline. Once standards visibly drop in one place, contributors match the lower bar. Soon nobody dares to write tests "because the project isn't tested anyway."

---

## 13. Architecture and modularity

### Layers and barriers, not "thin controller / fat model"
Real modularity comes from designing what each layer is allowed to know about the others, not from naming directories. "Service layer" is just moving the pile.

### Modular = no cycles in the dependency graph
"Independent modules" is too vague. The concrete, testable property is acyclic dependencies. A microservice mesh with cycles is a distributed monolith.

### Microservices won't fix bad code
Splitting a messy monolith makes it worse. The domain complexity stays and you add network failure, distributed transactions, queues, distributed logging, and async coupling. Microservices solve scale and team independence — different problems entirely.

Real example: a team migrated to microservices because tree queries were slow. The fix was a different tree-storage strategy (materialized path), not a new architecture.

```ts
// materialized path: each node stores its ancestor chain
type Node = { id: number; parentId: number | null; path: string };

async function subtreeOf(rootPath: string) {
  return db.query("SELECT * FROM nodes WHERE path LIKE $1", [`${rootPath}%`]);
}
```

---

## 14. Concurrency and distributed systems

### Concurrent edits → think locking
When multiple actors share write access, classify it as a locking problem and pick optimistic vs pessimistic explicitly. Mature ORMs already implement both — use the built-in support.

```ts
async function updateArticle(id: string, patch: Patch, version: number) {
  const r = await db.update("articles")
    .set({ ...patch, version: version + 1 })
    .where({ id, version });
  if (r.rowCount === 0) throw new ConflictError();
}
```

### At-most-once or at-least-once — pick one
You cannot have exactly-once message delivery between independent systems. Choose at-most-once or at-least-once and design the receiver accordingly (typically: at-least-once + idempotent handler).

---

## 15. HTTP and web

### Read the HTTP spec
Read the RFCs. They are more readable than most JS specs and answer questions about idempotency, caching, and method semantics correctly. Most "REST" code in the wild violates HTTP because the team learned from third-hand summaries.

### DELETE is idempotent — return 204 on a missing resource
A second `DELETE` of an already-removed resource should still return 2xx. Don't model "not found on delete" as 404.

```ts
app.delete("/articles/:id", async (req, res) => {
  await db.articles.deleteWhere({ id: req.params.id }); // no-op if absent
  res.status(204).end();
});
```

### Don't build URLs by string interpolation
Use the framework's named-route helpers. Helpers fail loudly when a route changes; hand-built URLs silently 404 in production until a user complains.

```ts
// bad
const href = `/users/${id}/posts`;

// good
const href = routes.userPosts({ userId: id });
```

---

## 16. Database and migrations

### Migrations only go forward
No rollbacks. Never drop or rename columns in the same release that introduces their replacement. Add new structure, dual-write during the transition, retire the old later. Zero-downtime deploys require old and new app versions to coexist against the same DB.

### Avoid DB-level defaults for business-meaningful columns
NOT NULL is fine. Defaults that change behavior are not — during a deploy, old and new code see different defaults and one of them breaks. Encode the value in application code instead.

### Store potentially-multi-state booleans as enums from day one
"Has machine" → `yes` / `no` becomes `yes` / `no` / `pending` faster than you think. A SQL boolean forces a painful migration; a string column doesn't.

---

## 17. Process and delivery

### Deploy continuously, in small pieces
Deploy multiple times per day in tiny increments. Big batches make it impossible to attribute outcomes to changes and hide course-correction signals. Time-to-market is the dominant business metric.

### Trunk-based with non-blocking review
Merge small changes straight to `main`. Review should not block the merge. Long-lived feature branches accumulate divergence and only surface mistakes after a week of work.

### Treat production bugs as a budget
Allocate a monthly loss budget for incidents. Spending the budget is normal cost of moving fast. Spending less means you were too conservative. You will never reach zero bugs.

### Invest in mobility, not absence of failure
Spend energy on monitoring, feature flags, canary releases, and easy revert — not on preventing every error. Cheap rollback turns a bad deploy into a five-minute issue.

```ts
if (await flags.isEnabled("new-checkout", user)) {
  return runNewCheckout(user);
}
return runLegacyCheckout(user);
```

### Isolate destructive operations from production access
Never make it possible to run "recreate database" or similar from a normal developer console against prod. GitLab famously wiped its database live because nothing prevented it.

### A separate QA department isn't required
With strong monitoring, feature flags, canary releases, and developer-owned tests, dedicated testers and staging add latency without proportional safety. Booking has thousands of engineers and zero testers.

### Async standups, not synchronous theater
A bot asks each morning: "yesterday / today / blocked?" People respond on their own schedule and discuss in-thread. Synchronous standups become reporting theater.

### Coarse, honest estimates
"Roughly a week" / "roughly a month." Don't run blame retros on missed point estimates. Detailed estimates don't make code ship faster — they create stress and turn into manipulation tools. Substitute visibility (frequent commits, async updates) for estimation theater.

---

## 18. Code-quality habits

### Pair programming and review keep the codebase consistent
Use pairing, mentorship pairs, and consistent review so the entire codebase reads as if one qualified developer wrote it. Style consistency is what eliminates context-switching cost; it also propagates quality faster than documents do.

### Code is good when little context is needed
On review, judge a chunk by how much surrounding code, documentation, or chat history a reader must load. The less context required, the better the code. Locality of understanding is the real readability metric.

### Use a Makefile as the project task runner
Every repo gets a `Makefile` with at least `setup`, `start`, `test`. It is universal across languages, doubles as runnable documentation, and survives the npm/rake/gradle zoo. Real projects are heterogeneous (Docker + npm + db tools); npm scripts collapse for multi-step shell pipelines.

```make
setup:
	npm install
	docker compose up -d db
	npm run migrate

start:
	npm run dev

test:
	npm test
```

### Use named workflows (e.g. git-flow) for onboarding
Adopt a documented branching/release model and a tool that implements it, so newcomers learn one named workflow instead of inventing a mental model from raw git primitives. Senior devs can drop the tool later — it pays off most for fast onboarding.

---

## 19. Career

### Work alongside people stronger than you
Choose teams where you are not the strongest engineer. Becoming the de-facto senior on a team that has no real seniors stalls your growth — the gap quietly widens.

### Learn languages from different paradigms
Pick one from each cluster: a Haskell-style pure FP, a Lisp (Clojure or Racket), a JVM-style typed OOP language (Kotlin), a C-family systems language, and an Erlang-family actor language. You cannot see the limits of your current language until you have worked in a fundamentally different one.

### Modern editors are converging via LSP
You no longer need a heavyweight IDE per language. Pick any editor with LSP support and add the language server you need. The "must use IntelliJ for Java" rule is mostly outdated.
