# Local aliases and multiple inferred result dependencies

A value-preserving alias does not change the result contract of its callee. Multiple return sites do not lose a dependency merely because they contribute separately. The following should preserve the class captured by `convert` through both returns:

```javascript
function make<T: type>() {
  class P { constructor(v: T) {} }
  return (v: any) => { const p: P = v; return p; };
}
const convert = make.<string>();
function use(flag: boolean) {
  const local = convert;
  if (flag) return local("first");
  return convert("second");
}
```

The previous implementation retained the converter's class but lost it at the inferred boundary of `use`. Both paths then threw ReferenceError for `P` after constructing the result. A repair must preserve the dependency independently of the returned value.

## Requirements and decision

Each contribution must retain its declaration, generic environment and applicable evaluated class identity. Local aliases must resolve at their own initializer's lexical position. Equal spelling or a shared class source declaration does not establish equal runtime identity. Reflection must not execute a producer, choose a branch, initialize a local binding or read through a reference. Completion must leave ordinary TDZ, conversion, nominal membership, replacement and reference-liveness rules intact. Autocomplete should preserve the published result members without implying that two distinct activations are interchangeable.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase the result or reject all local/multiple forwarding | Refactoring a direct call changes its contract; members disappear or valid code needs annotations | Rust's lexical bindings and captures do not erase a value's type on aliasing; explicit structural interfaces solve a different problem | Cheap implementation but loses useful specialization | Discards an existing dependency or rejects valid publication | Rejected: implementation limits do not justify changed semantics |
| Run initializers, producers or branch tests to discover a constructor | Reflection acquires surprising effects and wrong guesses can execute user code | Rust type checking does not execute closure bodies to inspect their result contracts | Unbounded work and effects; cannot safely cache by function alone | Violates phase separation, TDZ and reference obligations | Rejected: discovery is not an extra evaluation of the program |
| Pick one return, use equal names/source sites, or let the returned value choose | Behavior depends on traversal or call order; completion promises the wrong nominal identity | Rust captures preserve their lexical source; JavaScript dynamic class activations have no direct Rust analogue | Small apparent cache, but invalid across activations/realms; unsafe inline-cache assumptions | Conflates identities or makes admission circular | Rejected: neither spelling nor one observed value is sufficient evidence |
| Resolve everything only from the active invocation environment | Handles initialized local producers and avoids premature branch guesses, but leaves reflection incomplete even when a captured dependency is provable | Runtime lexical environments fit JavaScript; Rust resolves most type relationships statically | Repeated environment walks can be compiled to slots, but unnecessary work remains at each boundary | Useful for genuinely invocation-dependent obligations; cannot alone provide completed captured contracts | Insufficient alone; retained as part of the general solution |
| Retain dependencies per contribution; normalize proven aliases and close only on exact constructor consensus | Ordinary aliasing and same-result branches preserve members; distinct activations stay distinguishable | Closest to lexical binding/capture preservation in Rust, subject to the limits below | Weak syntax cache plus activation-local dependency guards; compiled slots can avoid walks without changing user object shapes or IC keys | Preserves declaration and activation identity; no producer execution or liveness waiver; unresolved alternatives remain obligations | Recommended independently of implementation effort |

Rust's [let statements](https://doc.rust-lang.org/reference/statements.html#let-statements) bind patterns to values, and its [closure types](https://doc.rust-lang.org/reference/types/closure.html) retain captured places. These support following a lexical value dependency. The analogy is limited: Rust moves and borrows have different aliasing rules, and Rust does not create a fresh nominal class identity each time a JavaScript-style class declaration executes. This proposal must preserve its own dynamic class and reference-liveness semantics.

The recommendation includes invocation-dependent dependencies even where they require more implementation work. Static constructor consensus is one sound completion case, not a substitute for the remaining representation.

## Bounded implementation

For a root unresolved nongeneric ClassDeclaration result, inspect the function's own written returned expressions, excluding nested functions and classes. Each must be a direct call through a captured immutable binding, possibly through parenthesized identifiers and plain body-local `const` aliases. Resolve every alias by declaration at its initializer. Typed or reference aliases, destructuring, mutable bindings, calls and other computed initializers are outside this proof. The scanner conservatively includes all written returns; it is not a new flow-reachability analysis.

For each captured source, read only an initialized ordinary value binding and obtain its existing published result. Follow these dependencies iteratively, with a cycle guard. All evidence must name the required nominal declaration, and every available concrete constructor must be the same ObjectValue. A shared declaration with different constructors does not pass this check. Require at least one concrete constructor. Forwarding evidence takes priority over an ambient same-spelled class; the latter cannot substitute for a disagreeing contribution. A contract already carrying a constructor is preserved.

The weak cache stores syntax and binding-declaration references, never activation-specific constructors. The traversal is bounded by the reachable dependency graph for this query. A production implementation can lower dependencies to captured slots and reuse guarded completions; any such reuse must retain activation identity and account for unavailable evidence. This work does not close the broader portable-budget or cache-accounting gate.

Reading a captured source for contract completion does not execute or initialize the body-local alias. Calling before that alias is initialized still throws normally. Reflection does not run a getter, Proxy trap, producer or reference read. Direct indirect Module bindings remain outside this proof, while an ordinary captured `const` retaining an imported value can supply evidence. Generic captures, independent factory activations, failure recovery, `finally` replacement, Function/eval, Modules and foreign Realms are covered by the focused tests.

## Remaining invocation-dependent cases

`const local = make.<string>()` inside `use` requires the callee created by that invocation. It cannot be normalized to a preexisting captured value without executing the initializer. Likewise, two captured converters created by separate calls to `make` can publish the same source declaration but different constructors. A function returning through either converter must preserve each contribution's dependency; it cannot close the entire result to the first constructor encountered.

These direct returned-call cases are now repaired by [invocation-specific result dependencies](invocation-result-dependencies.md). The callee's checked target travels with its internal completion, and the receiving boundary instantiates the matching unresolved dependency before checking again. Reflection retains its symbolic publication. This does not infer a target from the object being checked. Conditional operands and returned local value bindings still lose this evidence; generic class applications, nested nominal publications and complete imported-binding reflection also remain open.
