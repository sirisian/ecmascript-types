# do Expressions

A block that produces a value. `do { … }` evaluates its statements and yields the value of the last one that ran, so a computation needing a temporary, a branch, or a `try` can stay an expression instead of being hoisted into a statement above the thing that wanted it.

```js
const size: uint8 = do {
  const scale = devicePixelRatio;
  clamp(width * scale, 255)
};
```

This document does four things. It states the expression and the Early Errors that keep its value predictable. It gives the *type* of a `do` expression, which is the piece a typed language has to add and which the Early Errors are what make computable. It defines `do *`, the generator form, which shares a keyword with `do` and almost none of its semantics. And it retires a rule this proposal was carrying in the meantime: a [pattern matching](patternmatching.md) arm's block was given its own narrow value rule pending this feature, and that rule is this one now.

## The Expression

`do` followed by a block, in expression position. The value is the block's completion value - the value of the last statement that completed normally.

```js
const x = do { let tmp = f(); tmp * tmp + 1 };

const label = do {
  if (loggedIn) 'sign out';
  else if (pending) 'one moment';
  else 'sign in';
};
```

`do` expressions are not allowed where a statement is legal, because `do {` there begins a `do`-`while`. In statement position write a plain block, or parenthesize. The keyword costs the grammar nothing else: `do` is already reserved, so unlike a contextual keyword there is no program whose meaning changes.

An empty `do {}` is `undefined`, the value `void 0` - not the `void` type, which is the absence of a value and is a return type. `const x = do {};` is a `undefined`-typed binding and not an error.

### What a do expression may not end with

Three shapes are errors, and each because its completion value would not be what a reader predicts:

- **A declaration.** `do { 'before'; let x = 'after'; }` would be `'before'`, because a declaration's completion is empty and the value falls back to the statement before it. That is indefensible, so it is refused.
- **An `if` without an `else`.** `do { if (foo) bar }` is `bar` or `undefined` depending on a condition the reader has to trace, and the missing branch is invisible.
- **A loop.** `do { while (c) { … } }` is the last iteration's value, or `undefined` for zero iterations, which is a value nobody writes on purpose.

The rule is on the *completion*, not on the syntax, so it reaches through nesting: an `if`/`else` one branch of which ends in a loop is refused too, since that branch's value would be the loop's. A labelled block whose `break` leaves it holding a declaration is refused for the same reason.

```js
do { let x = 1; };                       // Error: ends in a declaration
do { if (foo) { bar } };                 // Error: no else
do { while (c) { work() } };             // Error: ends in a loop
do { if (c) { while (i) { f() } } else { 42 } };  // Error: a branch ends in a loop
```

A `var` is allowed anywhere but last, and hoists to the enclosing function as it always does. It is an error inside a `do` in a parameter expression, where there is no function body for it to hoist into yet.

## The Type

The type of a `do` expression is the type of its completion, read off the statements that can be the last one to run:

| Final statement | Contributes |
|---|---|
| an expression statement | the expression's type |
| a block, a labelled statement | the type of its own statement list |
| ```if```/```else``` | the union of both branches |
| ```try```/```catch``` | the union of the ```try``` block and the ```catch``` block; a ```finally``` contributes nothing, its completion being discarded unless abrupt |
| ```switch``` | the union of the case bodies, plus ```undefined``` unless the ```switch``` is exhaustive |
| ```throw```, ```return```, ```break```, ```continue``` | nothing: the path diverges |
| an empty block | ```undefined``` |

Nothing there is new. Divergence is the analysis the README's ```switch``` chapter already defines - syntactic, never reasoning about values - and the Early Errors above have already removed the shapes that would have made a completion type hard to state. What is left is a union over the tails.

The `switch` row is the design's own rule rather than the obvious one. An exhaustive `switch` - every enumerator, every direct subclass, or a `default` - takes no path where nothing ran, so it contributes no `undefined`. This is the same conclusion the README draws for a `switch` that maps a value to a type: because exhaustiveness is checked, such a `switch` yields a type object rather than `undefined`. A `switch` that is *not* exhaustive does contribute `undefined`, which is what makes the second line below an error:

```js
const t: type = do { switch (kind) { case 'int': int32; case 'float': float64; } };  // not exhaustive: type | undefined
const u: uint8 = do { switch (s) { case 'a': 1; } };                                 // Error: uint8 | undefined
```

**A `do` all of whose paths diverge is `never`.** That is the empty union, and it is assignable to everything, so a `do` that only throws satisfies any annotation:

```js
const port: uint16 = do { throw new ConfigError('no port'); };  // never
```

This is the first place ordinary code produces `never` rather than a type computation producing it, and it is the behaviour a reader wants: the binding is unreachable, so its declared type constrains nothing.

### A contextual type reaches every tail

The type a `do` is written into flows into every position that can be its value, so a literal there takes the expected type by ordinary [literal propagation](README.md) rather than defaulting to `number`:

```js
const level: uint8 = do {
  if (verbose) 3;      // uint8, not number
  else 0;              // uint8
};
```

Without this the feature would be typed but not usable: every literal in a `do` would need the annotation the surrounding declaration already carries. It is the same propagation a `match` gives its arms, reaching the same kind of position - the one that carries the value out.

### Narrowing

A test inside a `do` narrows inside it, and nothing escapes. What leaves is the type, and the union-of-tails rule is what carries the narrowing outward:

```js
const area = do { if (shape is Circle) PI * shape.radius ** 2; else 0; };  // float64
```

Which branch ran is not visible outside the expression, so no fact about `shape` survives it - the same rule a `match` arm follows for the same reason.

## Control Flow Leaves the Expression

`return`, `break`, and `continue` inside a `do` mean what they mean in the surrounding code, because a `do` block is a block and not a function body. This is what makes the shape below work, and it is why a `do` is not sugar for an immediately-invoked arrow:

```js
function userId(blob: string): uint32 | null {
  const parsed = do {
    try { JSON.parse.<Config>(blob) }
    catch (e: SyntaxError | TypeError) { return null; }   // returns from userId
  };
  return parsed.userId;
}
```

An arrow could not do that: its `return` lands in the arrow, its `await` demands an async arrow whose value is a promise, and `break` cannot reach out of it at all. `await` and `yield` inside a `do` are the enclosing function's, so a `do` in an async function may `await` and a `do` in a generator may `yield`.

Two restrictions follow the same reasoning that gives the Early Errors. An unlabelled `break` or `continue` may not appear in the head of a loop, where which loop it targets is a puzzle. And a `return` is allowed in a parameter default - `function f(x = do { return null; })` returns from `f` - but not in a computed property name in a class body, which is not inside any function yet. Where a `return` inside a `do` in a parameter default has an operand, it is checked against **the enclosing function's declared return type**, not against the parameter's: the check follows the run time, and at run time that `return` returns from the function.

## do * Generators

`do * { … }` is the generator form. Its value is a generator object, and the body is a generator body, so `yield` inside it belongs to the do-generator rather than to the enclosing function.

It exists to delete an IIFE that is otherwise unavoidable:

```js
// the shape without it
await Promise.all(function* () {
  for (const channel of this.channels) {
    if (!channel.enabled) continue;
    const next = newValues[channel.tag];
    if (channel.value !== next) yield channel.update({ value: next });
  }
}.call(this));

// with it
await Promise.all(do * {
  for (const channel of this.channels) {
    if (!channel.enabled) continue;
    const next = newValues[channel.tag];
    if (channel.value !== next) yield channel.update({ value: next });
  }
});
```

and it composes generators without naming them:

```js
for (const v of do * { yield* head(); yield* tail(); }) { use(v); }
```

### It is a function boundary, and do is not

The `.call(this)` above is what the syntax exists to remove, so `do *` binds `this` lexically, and `arguments` and `new.target` are the enclosing function's - it is to `function*` what an arrow is to `function`. But its body is a generator body, and that is not a preference: a construct that yields has to be one. So four keywords part company between two forms that differ by a single token:

| | ```do { … }``` | ```do * { … }``` |
|---|---|---|
| value | the completion value | a generator object |
| ```return``` | returns from the **enclosing function** | sets the **generator's** return value |
| ```yield``` | the enclosing function's | the do-generator's own |
| ```break```/```continue``` | may target an enclosing loop | cannot leave the body |

The `return` row is the one to watch, and it deserves saying plainly: **adding a `*` changes what `return` does.** In a `do` it leaves the function, as `userId` above depends on; in a `do *` it completes the generator. Nothing else in the language turns on one token that way.

### The Early Errors do not apply

A `do *` has no completion value - a generator body's completion is discarded, as any generator body's is - so the three restrictions have nothing to restrict. A `do *` may end in a loop, which the first example above does, and in a declaration:

```js
const updates = do * { for (const c of channels) yield c.update(); };  // fine: no completion value
const values = do { for (const c of channels) c.update(); };           // Error: ends in a loop
```

Sharing a keyword is what makes this worth stating. The restrictions are about a value the `*` form does not have.

### Types

A `do *` is a `Generator.<Y, R, N>`, and since there is no annotation site the three are found rather than declared:

- **`Y`**, the yield type, is the union of what the body yields: each `yield` operand's type, and for a `yield*`, the yield type of its operand.
- **`R`**, the return type, is the union of the `return` operands, or `void` where there are none.
- **`N`**, the next type, is `void` unless a contextual type supplies it. Nothing in a body determines what a caller will send to `next`, so a body that reads a `yield` expression's value without a contextual type is reading a `void` - the same diagnosis an unannotated `function*` gets, with the same fix.

```js
const merged = do * { yield* head(); yield* tail(); };  // Generator.<H | T, void, void>

const echo: Generator.<uint8, void, string> = do * {
  const sent = yield 1;    // 1 takes uint8; sent is string
  yield 2;
};
```

A contextual type flows into the `yield` operands, which is the same mechanism that reaches a plain `do`'s tails: in each form it finds the position that carries the value out.

That is what the typed version buys over the IIFE. `Promise.all` takes an iterable of promises, so `Y` is checked against what it will await - the `channel.update()` above has to yield a promise, and the checker says so at the yield rather than at the `await`. Through `function* () {}.call(this)` the same relation exists but has to be read through a call whose `this` is supplied separately.

`async do * { … }` is the async generator form, an `AsyncGenerator.<Y, R, N>`, with `await` and `yield` both belonging to it. The three forms cover the three function bodies the language has.

## Pattern Matching Arms

A [pattern matching](patternmatching.md) arm's block **is** a `do` expression's block. Its value is its completion value by the rule above, and it carries the Early Errors above:

```js
const size = match (request) {
  when { let width, let height }: {
    const scale = devicePixelRatio;
    clamp(width * scale, height * scale)
  }
  default: DEFAULT_SIZE;
};
```

Pattern matching shipped with a narrower rule while this feature was outstanding: an arm's block took the value of its final statement only where that statement was an expression statement, and was `void` otherwise. That rule was deliberately a subset of this one, so nothing written under it changes meaning, and two things get better. An arm may now end in an `if`/`else`, a `try`/`catch`, or a `switch`, which the narrow rule made `void` and therefore unusable. And an arm ending in a declaration is now an error naming the declaration, where before it was a silent `void` and a confusing complaint at whatever read the `match`'s value.

An arm's block is the plain form, never `do *`: a `match` arm produces the arm's value, and a generator there would produce a generator.

## Decorators

A `do` block and a `do *` block are blocks, so each takes a [block decorator](decorators.md) with no new syntax, and each has its own reflection context - `Reflect.DoBlock` and `Reflect.DoGeneratorBlock`. They are two contexts rather than one with a flag for the reason `ForBlock` and `ForOfBlock` are two: what the block *is* differs, a block in one case and a generator body in the other.

These are the first block contexts that admit a **return replacement**, and they are the first blocks that could: every other block produces nothing, so there has never been anything for a block decorator to replace. The exclusion of blocks from replacement was never about their being structural positions; it was about their having no value. A `do` has one.

```js
const config = @memo do { expensiveParse(readFile(path)) };

for (const v of @take(3) do * { yield* head(); yield* tail(); }) { use(v); }
```

A replacement is typed as what it replaces - `T` for a `do` of type `T`, and `Generator.<Y, R, N>` for a `do *` - which is what keeps the substitution checkable. Combined with the rule that a block decorator runs on **every entry** rather than once at the declaration, that is memoization, caching, tracing that substitutes, and stubbing, at an expression as small as one wants. The generator case is the more useful of the two: what a `do *` decorator wraps is an iterator, and filtering, limiting, and buffering a sequence is what a decorator on one is for.

Nothing else is needed. There is no new context family, no `addInitializer` - a `do` declares nothing to initialize - and no metadata of its own.

## What It Composes With

**Typed `catch`** is the pairing that makes the `userId` shape above typecheck: a `do` whose final statement is a `try`/`catch` unions the two blocks, and an annotated clause narrows what the handler sees, so the union is over exactly the paths the [error handling](errorhandling.md) rules admit. The two errors named there are the two a typed parse can raise - a `SyntaxError` for a document that is not JSON, a `TypeError` for one that is but does not fit the type - and a clause that named only one would leave the other to propagate, which is the filtering typed clauses are for.

**`const` and value types.** A composite key or a value-class instance often wants a temporary or two to build, and hoisting them above the `const` puts them in a scope that outlives the thing that needed them. A `do` keeps them where they belong:

```js
const key = do {
  const normalized = path.toLowerCase();
  Composite.<CacheKey>({ path: normalized, page: 0 })
};
```

**`match`, both ways.** A `match` is an expression, so it may be a `do`'s final statement and contribute its type; and a `match` arm's block is a `do` block, per above.

**Ranges and iteration.** `do *` is an iterable expression, so it goes wherever one does - a `for`-`of` head, a spread, `Promise.all` - which is the shape the composition example uses.
