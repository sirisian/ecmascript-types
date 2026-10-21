# Runtime interface application contexts

An interface application retains the interface declaration's environment. Written arguments use the application's environment; omitted defaults, constraints and value-parameter domains use the declaration's environment and the bindings established earlier in that application.

```js
// other
export function make(): type { return string; }

// dep
import { make } from "other";
export interface Has<T: type = make()> { value: T; }

// main
import { Has } from "dep";
function make(): type { return number; }
type Applied = Has;
Applied === Has.<string>;       // true: the default uses dep's make
Has.<> === Has.<string>;        // true: expression applications bind defaults too
Has.<make()> === Has.<number>;  // true: written arguments use main's make
```

The same rule holds through renamed imports, re-exports, namespace access and a forwarded Type Object. Interfaces declared in separate function activations retain their own captured bindings. A consumer's same-spelled helper or generic parameter cannot replace a capture. Named arguments and spreads distribute before binding defaults, and an explicit argument bypasses its default computation. A bare interface value still denotes its generic declaration; obtaining that value does not apply its defaults. A completed application named through an alias is not implicitly reapplied.

## Requirements and directions

A valid direction must preserve actual declaration identity and captures, resolve written operands in their own scope, bind defaults and constraints consistently across application forms, retain existing checking/initialization boundaries, bound recursive required evaluation, restore the caller after failure, and grant no reference-liveness permission.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Keep caller-scoped defaults and raw argument attachment in expressions | Identical-looking applications disagree; local helper names silently change imported types; completion cannot present a stable default | Unlike declaration-scoped generic parameters in Rust or declaration-owned defaults in TypeScript | Cheap application by omitting required work | Drops defaults, names or constraints and conflates scopes | Reject |
| Switch only the lexical environment for the existing default loop | Repairs the simple import but leaves caller generic frames and separate expression/namespace paths; wrong guesses still produce different types | Copies only part of lexical scoping; Rust's binder distinction remains missing | Small patch but duplicated binders and repeated fixes | Does not preserve captured parameter identities, consistent argument distribution or all failure restoration | Reject |
| Eagerly evaluate every interface default when its declaration is formed | Completion can show a value, but even explicitly supplied arguments trigger unused failing defaults | Rust and TypeScript can describe defaults statically; neither is precedent for eagerly invoking this proposal's runtime helpers | Pays for bypassed defaults and may force dependency work | Violates default demand, earlier-argument dependencies and initialization boundaries | Reject |
| Cache defaults by source declaration or parameter name | Apparently stable until a second activation supplies different captures; completion can describe the wrong application | Rust's item identity is not a runtime activation identity; a TypeScript type parameter name is not a cache key | Fast hits at the cost of sharing across different environments | Conflates independent bindings and can reuse failed/incomplete evidence | Reject |
| Retain the context on the actual interface binding and use the shared argument binder | All supported application paths expose the same bound type and constraints; explicit operands retain the user's local meaning | Preserves the useful lexical-binding/default analogies, with the limits below | One weak context association per declaration binding; reuse existing argument distribution and interning. No instance shape or property inline-cache change is required | Preserves identity, captures, demand, metering and exception-safe restoration; no new liveness proof | Recommend independently of implementation effort |

Choose the final direction. The context association is engine-owned metadata, not a new public Type Record identity field. Captures may be retained by an implementation's normal closure/context machinery; an optimized engine need retain only determining dependencies. Any later completed-result cache must include those dependencies and publish only a completed result. This repair adds no new default-result cache.

Rust's [generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes) are attached to the declaring item, and inner items do not capture an outer item's generic parameters. That is useful evidence for binder identity, but not a direct model for this proposal's interfaces created in function activations. TypeScript's [generic defaults](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-parameter-defaults) make parameters optional, constrain their defaults and support defaults depending on other parameters. This is a useful surface/API analogy; it does not specify this proposal's runtime Type Objects, helper execution or initialization timing.

## Implementation and failure boundaries

The runtime stores the lexical, private, realm/module and generic context against the interface's existing binding identity. Both type-position and expression-position applications enter that context only after resolving written arguments. The shared binder handles names, packs, preceding arguments, defaults, constraints and literal-domain admission. Namespace-qualified references and bound higher-kinded interface applications use the same interface binding operation. Existing partial-member merging remains in place.

A required recursive application joins the containing type-evaluation budget and restores the caller on failure. Ordinary lookup of an unapplied interface and explicitly bypassed defaults perform no default computation. The repair does not execute dependency bodies during checking, change when a dynamic `any` application becomes known, or make a retained binding initialized.

BindTypeArguments requires a TypeError when a value argument cannot be admitted to its declared domain. The shared record binder now reports that binding failure as TypeError rather than leaking the internal conversion's RangeError; ordinary conversions outside generic binding keep their existing behavior. This applies to scalar and variadic value arguments, including the alias, callable and class paths using that binder.

Self-referential interface bounds and a direct higher-kinded interface argument exposed separate upstream checking failures during this audit. They are recorded as follow-ups rather than treated as passing application-context cases. The broader identity, convergence, failure-precedence, resource-accounting and specification-completeness gates remain open.
