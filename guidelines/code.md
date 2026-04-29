# Writing rules

Local rules that fire whenever you write or modify a line of code. They apply to the code *you are writing* — they don't reach out to reshape surrounding code, so they don't conflict with the defaults.

## Naming

**Names stay truthful as the code evolves.** When a method's behavior changes, rename it everywhere it's called. Never invert internals while keeping the old name.

```ts
// lying name — internals were inverted
isActive() { return !this.active; }

// rename instead
isInactive() { return !this.active; }
```

**Replace compound boolean conditions with named predicates.** Any boolean expression on one aggregate should become a named method on that aggregate.

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

## Comments

**Comment non-obvious context.** Legacy quirks, system constraints, business reasons that can't be expressed in code — these belong in comments and they age well. If a comment explains *what* a line does, the line needs a better name; extract or rename instead.

```ts
// Legacy: pre-2019 accounts have role=null but are effectively admins.
if (user.role === "admin" || user.legacyAdmin) { ... }
```

## Function shape

**One level of abstraction per function.** Don't mix high-level domain steps with low-level I/O or string manipulation in the same function.

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

**Guard clauses; eliminate states early.** Exit on edge cases at the top. Each line below should handle fewer possible states than the line above.

```ts
function diff(a, b, key) {
  if (a[key] === undefined) return { type: "added", key, value: b[key] };
  if (b[key] === undefined) return { type: "removed", key, value: a[key] };
  if (isObj(a[key]) && isObj(b[key])) return { type: "nested", children: diff(a[key], b[key]) };
  if (a[key] === b[key]) return { type: "unchanged", key, value: a[key] };
  return { type: "changed", key, before: a[key], after: b[key] };
}
```

**Don't nest deeply — name intermediates.** If an expression chains many calls or nests more than two levels, break it up with named locals. They double as inline documentation.

```ts
// bad
sendEmail(formatBody(translate(loadTemplate(user.lang), pickKey(user))), user);

// good
const template = loadTemplate(user.lang);
const key = pickKey(user);
const body = formatBody(translate(template, key));
sendEmail(body, user);
```

**Use the operation that matches the role of the data.** If you're treating a value as a stack, use stack operations (`push` / `pop`). If you're treating it as a list of things, don't reach for mutating stack methods just to read the end — choose operations whose semantics match what you mean.

```ts
// bad — items is a list, but pop mutates it and signals stack semantics
const last = items.pop();

// good — read without mutating, when you just want the last item
const last = items.at(-1);

// also good — push/pop are correct when the role really is a stack
const undoStack: Action[] = [];
undoStack.push(action);
const lastAction = undoStack.pop();
```

## Conditions and states

**Make `else` explicit.** Spell out the alternative even when there are only two cases today.

```ts
// bad — implicit "non-ru" means "en"
if (lang === "ru") return ru();
return en();

// good
if (lang === "ru") return ru();
if (lang === "en") return en();
throw new Error(`Unsupported lang: ${lang}`);
```

**Don't encode meaning in truthiness.** Don't decide "is leaf" by absence of `children`, "is logged in" by absence of `null`, "is empty" by `!collection`. Make the structural fact explicit.

**A collection variable is never null.** If a value can be a list, give it `[]` when there are no items. Otherwise every callsite needs `if (xs && xs.length)`.

**Test the cause, not a correlated proxy.** A condition should test the actual reason, not a value that happens to correlate today.

```ts
// bad — confirms password by absence of a confirmation token
if (!user.confirmationToken) validatePassword(user);

// good — confirms by the real cause
if (user.signupMethod === "email") validatePassword(user);
```

## Mutation and effects

**Command-Query Separation.** A function either changes state (command) or returns a value (query). Never both. A predicate like `isValid` must not write to the database.

```ts
// bad — query with hidden mutation
class User {
  isValid() {
    this.avatar = null;
    return this.email.includes("@");
  }
}

// good
class User {
  isValid() { return this.email.includes("@"); }
  normalize() { this.avatar = null; }
}
```
