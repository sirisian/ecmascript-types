# Lexical scope for runtime generic reads

Generic parameters follow lexical scope in both type and value positions. A caller's specialization cannot make an otherwise missing name visible to an unrelated function, indirect eval, or a dynamically constructed function. A nearer ordinary binding wins. An escaped closure keeps the argument of the activation that created it, even during another application of the same generic declaration.

The three reproducers preserved in [the direct-eval annotation increment](eval-generic-parameters.md#newly-preserved-runtime-lookup-discrepancies) exposed one runtime problem: resolving a parameter discarded its lexical owner, and reading the resulting unresolved reference searched active frames by name. The same fallback appeared separately in `typeof`. Thus a globally defined `function read() { return typeof T; }` could see the `T` of a generic caller.

```js
function outer<T:type>() { return () => eval("T"); }
const numberReader = outer.<number>();
const stringReader = outer.<string>();
function invoke<T:type>(read) { return read(); }
invoke.<boolean>(numberReader); // the number Type Object
stringReader();                // the string Type Object
numberReader();                // still the number Type Object
```

## Decision and alternatives

A valid solution must preserve the resolved declaration and binding instance, nearer ordinary bindings and their TDZ, the type/value/pack/metadata views, left-to-right availability, captured and suspended activations, ordinary effects and checking boundaries. Obtaining scope evidence does not create storage or reference authority. Public Type Object equality cannot establish a binding.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep dynamic lookup | A callback's meaning and completion candidates depend on an unrelated caller. A wrong guess can silently select the wrong type. | Rust parameters belong to their declaration; ECMAScript's lexical environments do not follow the calling stack. | A stack search is simple but gives unstable specialization and inline-cache assumptions. | Violates lexical scope, shadowing and activation identity. | Reject. |
| Clear frames only around indirect eval and function constructors | Fixes two spellings but leaves ordinary functions, closures and `typeof` inconsistent. | ECMAScript specifies the source environment, not a list of call spellings. Rust gives no basis for this exception list. | Requires frame restoration across calls and suspension; does not resolve ordinary shadows. | Can erase legitimate captures and still admit caller leakage elsewhere. | Reject as the semantic solution. |
| Use source spelling, declaration identity alone, or equal Type Objects | Same-named binders and recursive applications remain surprising; equal values can accidentally appear to grant access. | Rust's declaration scope motivates identity, but a runtime JavaScript activation needs an instance as well. | Cheap keys omit information required for cache reuse; equality searches add work without proof. | Separately parsed eval has no enclosing syntax edge; a declaration has multiple activations. | Reject these as sufficient evidence. |
| Re-read values, invoke getters, or replay a producer to recover the argument | An annotation or lookup can add observable work or change a failure. | ECMAScript preserves required reads and their order; Rust provides no analogy authorizing replay. | Adds calls, guards and invalidation, including effects on object inline caches. | Can duplicate effects, bypass the TDZ or invent reference authority. | Reject. |
| Resolve the parameter lexically and retain its actual argument environment through the read | Direct eval, closures, ordinary shadows and globally scoped code have one predictable meaning. | ECMAScript supplies the lexical/global distinction. Rust supports declaration ownership and const-parameter value access as limited analogies. | Private environment or specialization slots; a compiled implementation can resolve a slot or captured argument directly. No user-object shape change is required. | Keeps declaration and activation identity, availability and the proper value view without a caller search. | **Recommend**, independently of implementation effort. |

Rust's nested function items do not inherit enclosing generic parameters, and Rust has no corresponding runtime eval facility. Its type parameters also are not this proposal's first-class Type Objects. Those differences limit the analogy; they do not support dynamic caller scope.

For representation, ordinary mutable environment slots would unnecessarily introduce storage and writes. A private field on an internal reference or a read-only environment view can both retain the required owner without user-visible shape changes. The engine uses an environment view so ordinary reference operations, including `typeof` and calls through a type value, share the same resolved base. This is an implementation choice, not a requirement to allocate an extra environment in a production engine.

Making a parameter resolve must also settle whether `ref T` acquires a location. The invariant is that generic arguments provide values, not mutable storage; resolution alone cannot establish a borrow's permissions or lifetime.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Give each parameter ordinary writable storage | Assignment looks possible but would leave annotations and specialization identity disagreeing about the argument. | Rust generic arguments cannot be reassigned; JavaScript mutable locals are a different category. | Adds slots, stores and invalidation to specialization assumptions. | Violates the fixed argument and creates reference authority absent from application. | Reject. |
| Borrow a temporary or shared read-only argument slot | A plausible `ref` spelling hides which lifetime and identity it borrows; autocomplete cannot explain a missing owner. | Rust can promote eligible constant expressions, but that is a separate promotion rule, not evidence for borrowing every generic name here. | Requires promotion, lifetime bookkeeping or pinned argument storage; shared specialization slots complicate identity. | This proposal's storage and liveness rules supply no such promotion or lifetime. | Reject; any future promotion needs its own design. |
| Keep a read-only parameter view with no borrowable location | Reading is stable; assignment and borrowing fail at their use rather than changing a later application. | Rust's fixed generic arguments are a limited analogy; the no-location judgment follows this proposal's reference model. | A compiled read can use a constant or captured slot, with no address-taking obligation or public shape change. | Preserves fixed arguments and all existing reference permissions and liveness checks. | **Recommend**. |

## Runtime and checking boundaries

Resolution first performs the ordinary lexical walk, including any required object-environment operations. At a generic declaration's scope, the reference retains that particular parameter environment. Reading it produces the already bound argument's prescribed view: a Type Object for a type parameter, a value for a value parameter, and the existing pack or metadata view for those parameters. An unavailable argument does not fall through to an older invocation. An unresolved reference remains unresolved; `typeof` returns `"undefined"` by its ordinary rule.

The engine's private view creates no ordinary storage binding. Writes are refused, and taking a reference to a generic parameter cannot manufacture a borrowable location. Existing reference permissions and liveness checks remain necessary for ordinary locations.

A separately parsed annotation with no source-local declaration obtains generic evidence only from its actual lexical environment. Indirect eval and dynamic functions therefore cannot obtain a caller's argument. Source-owned symbolic type-formation frames still serve the existing type algorithms; an otherwise missing source name does not gain access to those frames. Generated family-pattern holes carry private syntax evidence, so internal placeholders remain usable without opening a name-based source fallback.

Some algorithms evaluate declaration work before a function activation exists. Alias expansion, argument binding, inference trials and callable selection need explicit environments for their own parameters. Those scopes retain the selected argument frame, including arguments filled left to right and temporary trial candidates. Closures created by a computation capture its scope normally. The environments are restored on abrupt completion. Specialization captures retain their declaration identities when selected and copied. Metadata hooks and primitive operators associate their header parameters with their own invocation. A primitive operator's computed signature retains its declared capture list during both pre-evaluation and runtime dispatch, including bodyless operators that have no function activation.

## Verification and remaining work

The portable `runtime-generic-scope.json` corpus covers the original three reproducers, missing value reads and `typeof` through indirect eval, dynamic functions and unrelated helpers, ordinary global and block shadows, the TDZ, escaped closures and repeated activations, recursive calls, eval-created closures, type/value/pack views, defaults, alias computations, class methods, imports, suspension and failure recovery. Adjacent suites exercise contextual generic annotations, selection, inference and metadata behavior. The delivery's verification record gives exact counts, baseline comparisons and any bounded runs that did not complete.

One separate implementation gap is preserved: an implicit-cast body reading its header capture, such as `primitive float32<const X: Dim> { operator float32.<X>() { if (X.m !== 1) throw new Error(); return this; } }`, throws `ReferenceError` on both the pulled baseline and the patched engine. Cast target resolution is covered here; implicit-cast dispatch still needs to supply the body's capture bindings. The delivery includes the paired reproducer rather than a test asserting that failure as correct.

The imported-constructor `instanceof` and hook-effect audit remains open. This change does not certify every identity, inference, resource, phase/recovery or conformance gate, and does not settle the unverified recursive-builder memo/fixpoint families.

Sources: [Rust generic parameters](https://doc.rust-lang.org/reference/items/generics.html), [Rust constant promotion](https://doc.rust-lang.org/reference/destructors.html#constant-promotion), [ECMAScript PerformEval](https://tc39.es/ecma262/multipage/global-object.html#sec-performeval), and [CreateDynamicFunction](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-createdynamicfunction).
