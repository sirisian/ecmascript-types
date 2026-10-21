# Class constructors as reflection type inputs

A class constructor denotes its associated instance type at a type-input reflection boundary. This includes a closed specialization such as `Box.<string>`. `Reflect.isAssignable` and `Reflect.matchType` must accept that denotation just as structural `Reflect.getReflection` and `Reflect.makeType` do.

```js
interface Has<T: type> { get(): T; }
class Box<T: type> implements Has.<T> {
  value: T;
  constructor(value: T) { this.value = value; }
  get(): T { return this.value; }
}

Reflect.isAssignable(Box.<string>, Has.<string>); // true
Reflect.isAssignable(Box.<string>, Has.<boolean>); // false
Reflect.matchType(Box.<string>, type Box.<string>); // a successful match
Reflect.makeType(Box.<string>); // the associated, interned instance Type Object
```

Resolve the actual constructor value's existing association. Preserve its evaluated declaration identity, source context, and ordered type/value arguments. Do not re-resolve its name, inspect its prototype, invoke a constructor or member, or use a value's runtime type as an implicit substitute. An ordinary function, class instance, bound function, or Proxy does not acquire the target class's association merely by resembling or wrapping that class.

A generic declaration constructor still denotes its declaration base; this lookup does not apply defaults. An already formed specialization denotes that specialization. Reflection does not instantiate a class or reopen its admission checks. Argument evaluation and specialization formation precede the call in their usual order; their effects and failures remain observable. The first invalid type input is rejected before the relation is tested, even when the other input is `any` or `never`.

`typeof Box.<string>` remains `"function"`. `Reflect.typeOf(Box.<string>)` asks for the constructor value's function type, while the type-input APIs ask what type that constructor denotes. The internal record representation and the reflection admission test must not redefine JavaScript's callability or construction behavior. This decision does not change `===` or the existing interning representation; `Reflect.makeType(C)` gives the canonical Type Object when a constructor and its associated object are represented separately. Match bindings likewise return canonical Type Objects.

## Decision

Required outcomes: every existing class denotation is accepted consistently; different evaluated declarations and different arguments stay distinct; invalid inputs remain errors; no user code runs to discover an association; no object-shape change is required; reference initialization, access and liveness rules remain independent.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Require an explicit `type C` or `Reflect.makeType(C)` at each relation call | Extra syntax and different acceptance across neighboring APIs; autocomplete's class value is an easy wrong guess that throws | Rust's `TypeId::of::<T>()` is explicit, but Rust does not make JavaScript class constructors first-class type denotations | Direct record fast path; no association lookup | Can preserve identity, but contradicts the proposal's existing reflection association rule | **Rejected.** Would deliberately retain the API inconsistency |
| Resolve the actual existing constructor association at all type-input reflection boundaries | `C` and `C.<T>` work where their type meaning is expected; invalid unrelated values still throw | Rust's opaque type identity supports preserving exact identity, though it supplies no class-constructor analogy; ECMAScript reflection already distinguishes the supplied value from how an API interprets it | A direct internal slot or identity-table lookup; no property access, prototype walk, shape transition or user-visible inline-cache invalidation | Preserves declaration and argument identity; no constructor execution, coercion, new liveness proof or admission bypass | **Recommended.** One rule for the existing four reflection consumers |
| Globally classify every associated constructor as an ordinary Type Object | Superficially uniform, but risks changing value-level behavior beyond reflection | Rust keeps type identity separate from callable values; ECMAScript fixes callable constructors' `typeof` behavior | Could use a slot, but global reclassification changes many fast paths; adding observable properties changes shapes | Requires a broader review of conversion, `typeof`, membership, construction and interning; association lookup alone does not justify these changes | **Rejected for this decision.** The API rule does not require a global semantic change |
| Infer a type from any callable, prototype, instance, Proxy target or coercion hook | Convenient guesses, but a wrong guess can silently select a function type or run arbitrary code | Rust's `TypeId` identifies a specified type, not a guessed callable result; ECMAScript Proxy behavior gives no general transparent identity promise | Prototype/property reads, traps and invalidation dependencies make caching harder | Conflates denotation with runtime type and can lose nominal identity; introduces effects and failure-order changes | **Rejected.** Fails identity and no-execution requirements |
| Repair only `isAssignable`, leaving `matchType` unchanged | Fixes the immediate example while the adjacent matching API still throws | Neither Rust identity nor ECMAScript callability explains an operation-specific exception | Essentially the same lookup cost as the shared rule | Preserves the local relation but leaves the stated reflection rule inconsistent | **Rejected.** Scope must follow type-input semantics, not the first reproducer |

The recommendation is independent of implementation effort. An engine may use a slot, side table, or another equivalent representation. The engine262 repair shares the existing association resolution among the four reflection consumers and the two type-membership checks, and leaves assignability and matching relations unchanged. `Reflect.typeOf` is a value-input operation and is deliberately governed by its own runtime-type rule. Declaration-reflection contexts retain their existing dispatch.

## Typed reflection helpers

A specialized constructor must also pass through a parameter, result or binding annotated `type` or `Reflect.TypeObject`. Otherwise a typed wrapper around `isAssignable` fails before reaching the operation even though the equivalent wrapper receives `Reflect.makeType(C)` successfully. Membership recognizes the same association and retains the original value: a constructor remains constructible and compares identical to the argument supplied. This includes a generic declaration base whose associated Type Object was already admitted; kind and arity checks on actual generic applications remain separate.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Accept constructors only inside reflection intrinsics | Direct calls work, but the natural `type` annotation on a helper throws; autocomplete suggests a contract the argument cannot cross | Rust's explicit `TypeId` API is coherent because it has no constructor-as-type promise; it does not justify this proposal's mismatch | Cheap intrinsic lookup, but no benefit over checking the same association at the typed boundary | The normalized Type Object passes while its established denotation fails | **Rejected.** Leaves ordinary typed composition broken |
| Admit the associated constructor by membership and retain its value | Typed bindings, parameters and results compose; identity, static properties and `new` remain available | Rust separates identity from callability; here admission can likewise preserve the value's existing callable behavior without pretending Rust has this API | An internal identity lookup; no allocation, shape change or property read | Matches the already admitted associated Type Object without changing generic kind rules, runtime type or reference liveness | **Recommended.** Applied to both `type` and `Reflect.TypeObject` |
| Convert the constructor to its associated Type Object at every typed boundary | Reflection works, but adding an annotation can change `===`, property access and construction behavior | Rust's explicit `TypeId::of` is a separate operation, not an implicit value replacement at a parameter annotation | May reuse the interned object, but changes receiver identities and can disrupt call-site specialization | Violates value-preserving admission; may substitute a conversion callable for a class constructor | **Rejected.** Wrong semantic cost even if implementation is easy |
| Broaden the global Type Object classifier | Reduces special cases internally but changes operations that ask what a value is, rather than what it denotes | Neither Rust identity nor ECMAScript reflection requires global callable reclassification | More fast paths and caches require review; no benefit necessary for these membership checks | Risks runtime-type, conversion and construction behavior beyond admission | **Rejected.** Use a separate denotation resolver |

Structural `makeType` inputs retain their existing node-property reads when the input is not already a type denotation. The association lookup itself reads no program-visible property and adds no Proxy transparency.

## Prior-art sources and limits

- [Rust `TypeId`](https://doc.rust-lang.org/std/any/struct.TypeId.html): opaque type identity and explicit `TypeId::of::<T>()`; its documentation also warns that identity-based abstractions must account for subtyping/variance. It does not define JavaScript-like constructor reflection or a general assignability query.
- [ECMAScript `typeof`](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-typeof-operator): callable Objects report `"function"`. Accepting a constructor as a reflection type input must preserve that behavior.

The requirements on performance, reflection admission and Proxy identity above are this proposal's design conclusions, not claims that Rust supplies the same API.
