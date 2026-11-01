# Descriptor observation, accessor extraction and typed metadata

This is milestone 5 increment 17 of the [eval and stable names plan](eval-name-stability-plan.md), following [legacy prototype setters](legacy-prototype-effects.md). The [implementation record](descriptor-observation-effects-implementation.md) gives exact bases, tests and remaining limits. Milestones 5 and 6 remained open at this increment's delivery; the [integrated completion record](milestones-5-6-validation.md) gives their current status.

## Recommended direction

Descriptor observation is its own operation. For a positively established ordinary target it copies own descriptor fields without reading the described property. A returned getter or setter is an unbound callable value. Its later invocation retains its actual receiver, arguments, effects and receiving checks.

Recognize original reflection functions by callable identity. Separate observing the descriptor, reading a field of the returned descriptor, and invoking an extracted callable. Preserve operation positions and the actual built-in's realm. Treat the descriptor as a fresh mutable object containing shallow references to its field values, not as a live view of the target.

Preserve the declared Type Object in the descriptor's type field, as the design already promises. This requires a runtime repair as well as effect analysis: the engine previously recorded the type in storage metadata but omitted it from the descriptor returned by ordinary reflection. A copied typed descriptor must pass through ordinary typed definition and admission again. Reflection grants no authority over the source property's storage or reference lifetime.

## Cases that motivate the distinction

An earlier Script can initialize:

~~~js
const target = {
  get run() {
    globalThis.f = () => 1;
    return 0;
  }
};
~~~

The consumer can then perform:

~~~js
function f(): string { return "old"; }
const descriptor = Object.getOwnPropertyDescriptor(target, "run");
const before = f;
descriptor.get();
const n: number = f();
~~~

The observation does not call the getter. Its later invocation can change the Script-global function, so the final read cannot use the former string result. Replacing that initializer with before() preserves the earlier incompatible result. Treating the observation itself as an ambient effect loses that capture.

The descriptor's fields are not live aliases of the target's property slots:

~~~js
const target = { run() {} };
const descriptor = Object.getOwnPropertyDescriptor(target, "run");
target.run = replacement;
descriptor.value(); // the function obtained earlier
~~~

Conversely, a referenced object is not cloned:

~~~js
const target = { world: globalThis };
const descriptor = Object.getOwnPropertyDescriptor(target, "world");
descriptor.value.f = () => 1;
~~~

Assigning descriptor.value changes the descriptor object, not target.world. A Type Object in descriptor.type has the same distinction: replacing that field does not change the source property's declared storage type.

## Requirements for a valid solution

| Requirement | Consequence |
| --- | --- |
| Actual reflection implementation | A replaced method is not summarized by its name; a saved original remains recognizable |
| Own-descriptor observation | No inherited property traversal or getter invocation just to discover an own descriptor |
| Operation-specific effects | Key coercion, proxy traps, trap-result conversion and storage-backed observations retain their effects |
| Fresh result with shallow field values | Descriptor mutation is independent; referenced objects and accessors retain identity |
| Temporal evidence | Later target mutations do not retarget a previously observed field |
| Realm evidence | Fresh descriptor/table prototypes come from the actual built-in's realm |
| Declared type metadata | Preserve the property's established Type Object rather than inferring a type from its current value |
| Existing obligations | No check elision, conversion bypass, new reference permission or extended liveness follows from reflection |
| Conjunctive and bounded proof | An ordinary alternative does not cancel an unknown one; exhaustion loses evidence |

The following decisions use all five requested criteria. Implementation effort is not the selection criterion. The interpreter's conservative joins are recorded separately from the recommended language rules.

## Decision 1: classify by method name, by reflection generally, or by the actual operation?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Give the standard method spelling a no-effect summary | Completion looks convenient until the method is replaced or borrowed | JavaScript properties are mutable; Rust item identity is stronger than a matching name or signature | Cheap but invalid specialization | Misses replaced methods and cross-realm originals | Reject |
| Treat all descriptor inspection as opaque ambient execution | Harmless inspection unexpectedly destroys captures | JavaScript distinguishes descriptor retrieval from property access; Rust does not require arbitrary effects merely for referring to a function | Broad invalidation and missed optimizations | Safe fallback, not the intended complete rule | Reject as the language direction |
| Establish actual callable identity and target operation separately | Saved helpers and normal reflection behave consistently; unsupported targets have an explainable cost | Matches JavaScript internal operations; Rust contributes the identity/signature distinction only | Reuse target identity, shape and representation guards | Retains proxy/backing effects without inventing getter execution | **Recommend** |

The reflection function's identity alone is insufficient. The base engine's ordinary GetOwnProperty method also implements some proposal backing reads. Its method pointer is therefore not a universal no-user-code certificate. This delivery checks applicable ordinary representation metadata as well as internal method identity, and retains a conservative effect for unsupported targets.

Object and Reflect single-descriptor operations have different target-validation rules. Key coercion and abrupt completion retain their specified order. Bulk observation includes own-key enumeration, non-enumerable properties and symbols; it is not object spread or Object.assign.

## Decision 2: what does the returned descriptor represent?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use a live alias of the target property | An edited descriptor appears to update its source; a later replacement silently retargets a saved accessor | Unlike JavaScript descriptors; Rust borrows are explicit and are not the right analogy | Avoids a temporary allocation but changes semantics and ownership assumptions | Breaks snapshot behavior and reference authority | Reject |
| Read or invoke the described accessor to discover its result | Inspection causes surprising side effects and exceptions | JavaScript separates descriptor fields from property Get; Rust reflection offers no justification | Runs arbitrary code and may repeat work | Violates evaluation order and liveness constraints | Reject |
| Return a fresh descriptor carrying shallow field references | Editing fields is independent while referenced values keep identity; completion can expose members without promising purity | Fits JavaScript; Rust copying a function-item value illustrates identity preservation, not a JS heap model | Often scalar-replaceable when the descriptor does not escape; ordinary allocation otherwise | Preserves accessor identity, value capabilities and normal receiver rules | **Recommend** |

A returned getter is not automatically bound to the object whose property was inspected. Calling descriptor.get() uses the descriptor as its receiver; an explicit call adapter can supply another receiver. The checker may propagate either invocation's effects but cannot insert a call, bind a receiver implicitly, or use an accessor's signature as proof of purity.

## Decision 3: how should observations interact with mutation and uncertain descriptors?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Connect a result field to the target's timeless mutable value set | Later writes unexpectedly invalidate an earlier capture | Rust item identity supports keeping an obtained value; mutable JS property paths require sequencing | Easy graph, over-broad dependencies | Loses the distinction between a value and its producing path | Reject as the intended rule |
| Select the latest textual descriptor unconditionally | Attractive diagnostics, wrong for branches, repeated/deferred execution or aliases | Neither JavaScript execution nor Rust control flow is determined by a flat source index | Small state, unsound joins | Can discard a viable accessor or data alternative | Reject |
| Retain observations at their operation positions, with positive presence/kind proofs | Earlier captures stay usable; uncertainty belongs to the relevant operation | Matches JavaScript sequencing; Rust's fixed item identity is a limited comparison | Shape/property versions and dependency graphs separate observations from later writes | Requires all viable alternatives; unknown never becomes definite absence | **Recommend** |

The delivered graph records source/native data contributions and accessor definitions separately from their later use. It filters later writes from earlier descriptor observations, including supported immutable copies and invocation adapters. An argument-side namespace replacement does not retarget a reflection function already obtained, while changes to the target/key before the internal observation can matter.

Historical descriptor, deletion and branch alternatives remain conservative. Exact strong updates, definite table-key presence, complete per-invocation contexts and arbitrary heap mutation histories are not finished. Tests isolate capture ordering from the existing broad treatment of deferred source bodies. That separate limitation is not silently reclassified as solved.

## Decision 4: what should descriptor.type expose?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Remove the promised field or leave reflection asymmetric | A copied descriptor silently loses its receiving contract; completion disagrees with the proposal | Rust has no analogous mutable descriptor API, so its omission does not justify removing this promise | Minimal immediate work, breaks compositional reflection | Contradicts the existing design | Reject |
| Infer a type from the current value or return a type-name string | Confuses a snapshot value with declared storage and may lose nominal/generic identity | Rust likewise distinguishes type identity from an arbitrary name string; runtime JS values do not establish storage contracts | Extra inference/formatting and ambiguous reconstruction | Cannot preserve declared any, broader declared types or identity | Reject |
| Return the established Type Object as descriptor metadata | Matches the documented field and permits faithful typed descriptor round trips | Rust supplies an identity lesson, not a runtime reflection equivalent | Existing metadata lookup; canonical Type Object materialization where needed | Preserves the declared contract without granting source storage access | **Recommend** |

The engine repair preserves ordinary typed own-property metadata through its internal descriptor and FromPropertyDescriptor. Covered routes include typed literal properties, class fields, global var properties and typed Object/Reflect definitions. A declared any is still a declared type. An untyped property holding a typed value does not thereby gain a declared type.

The returned type field is ordinary writable, enumerable, configurable descriptor data. Replacing it changes only the returned object. Passing it to a later definition invokes existing type validation, conversion, defaulting and storage admission. The Type Object must remain reachable while its descriptor is live. Reflection does not release the source's obligations or copy a reference capability.

This does not define every backing/indexed-storage descriptor or add metadata to an inherited property that has no own descriptor. Those representations retain their own rules and require further coverage. The effect analysis does not construct a Type Object or execute type evaluation just to resolve a source expression's unknown metadata.

## Decision 5: which realm owns the fresh descriptor?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Always use the checking/caller realm | Borrowed originals acquire the wrong inherited behavior | JavaScript realms have no direct Rust function-item counterpart | Convenient single prototype, invalid dependency | Misses getters on the actual descriptor prototype | Reject |
| Treat all cross-realm reflection as permanently opaque | Poor helper interoperability and capture precision | Rust item identity does not become unknown merely because an alias is elsewhere, but the realm analogy remains limited | Broad invalidation | Discards available actual-callee metadata | Reject as the complete rule |
| Use the established original built-in's realm | Borrowed originals retain ordinary semantics; result prototype behavior is explainable | Matches JavaScript built-in execution; Rust only supports the identity distinction | Internal realm pointer and ordinary prototype dependencies | Keeps descriptor fields and bulk tables on their actual initial chains | **Recommend** |

This is the prototype of the descriptor result, not a request to relocate field values or replace their Type Object identities. The cross-realm test installs a getter on the foreign descriptor prototype and checks that a later read retains its effect. No getter is probed while establishing that dependency.

## Decision 6: what may extraction prove about a later call?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat an extracted accessor as pure with an authoritative result signature | Completion hides receiver requirements and dynamic result checks | Rust distinguishes item identity from compatible function-pointer signatures; neither grants JS purity | Aggressive but invalid check removal | Confuses identity, effects, public contracts and liveness | Reject |
| Forget every extracted identity immediately | Inspection-based helpers lose useful evidence | JavaScript field values and Rust function items can retain identity through copies | Simple fallback, repeated broad invalidation | Misses available private provenance | Reject as the complete rule |
| Preserve private callable/value flow while checking invocation and reception | Useful identity without a promise of purity; wrong values fail at the receiving boundary | Uses the limited Rust identity analogy without importing ownership or borrow rules | Supports guarded specialization while retaining required slow paths | Preserves counts, abrupt completion, receivers and existing obligations | **Recommend** |

An ordinary descriptor read does not prove that a later getter invocation is harmless. A proxy can manufacture a descriptor; trap-result conversion can itself invoke descriptor-field getters. Unknown alternatives retain those effects. Static evidence never bypasses a receiving check, converts an unavailable result into a wildcard, or revives dead storage.

## How close the other-language comparisons are

| Prior art | Useful comparison | Limit of the analogy |
| --- | --- | --- |
| Rust function items and TypeId | Implementation identity and type identity are different facts; an opaque type identifier is stronger than a displayed name | Neither a type identifier nor a function-item identity specifies JavaScript descriptor mutation, realms, accessor receivers or reference liveness |
| .NET PropertyInfo.PropertyType | Reflection can expose a property's declared type independently of its current value | A .NET property metadata object is not JavaScript's fresh mutable descriptor, and CLR type/visibility rules are not this proposal's storage rules |
| Python inspect.getattr_static | Inspection can deliberately avoid dynamic attribute lookup and descriptor execution | Python's static lookup can return descriptor objects and differs from normal lookup; it is not the JavaScript own-descriptor algorithm or a blanket purity guarantee for proxies |

These comparisons support separating inspection, invocation and declared metadata. They do not justify importing another language's ownership rules or treating all reflective operations as harmless. See the primary documentation for [TypeId](https://doc.rust-lang.org/std/any/struct.TypeId.html), [PropertyType](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.propertyinfo.propertytype) and [static attribute inspection](https://docs.python.org/3/library/inspect.html#inspect.getattr_static).

## Production implications and remaining work

Production engines already store descriptor kind, attributes and accessor/value identities in property metadata. Ordinary inspection can reuse those facts and allocate the specified result; non-escaping results may admit scalar replacement. A typed metadata field can use the existing declared-type entry. No public hidden marker, callable wrapper, or visible target mutation is required.

The proof needs distinct dependencies for reflection callee, target representation, property identity/kind, observation position and result prototype. Shape/IC invalidation and runtime deoptimization preserve already evaluated operands, but cannot retroactively justify an Early Error. A borrowed built-in can use another realm even when the caller's namespace has the same spelling.

The interpreter remains a bounded inclusion analysis. Symbol and unknown keys can share wildcard candidates, source metadata can be unavailable, and historical joins can lose precision. These are implementation limits, not permission to vary observable acceptance/error phases with arbitrary budgets. Native callbacks, broader backing/indexed protocols, exact descriptor transitions and invocation contexts remain under milestone 5. Milestone 6 retains integrated conformance and recorded baseline blockers.

Primary references: [Object.getOwnPropertyDescriptor](https://tc39.es/ecma262/#sec-object.getownpropertydescriptor), [bulk descriptor observation](https://tc39.es/ecma262/#sec-object.getownpropertydescriptors), [Reflect.getOwnPropertyDescriptor](https://tc39.es/ecma262/#sec-reflect.getownpropertydescriptor), [FromPropertyDescriptor](https://tc39.es/ecma262/#sec-frompropertydescriptor), and [Rust function-item types](https://doc.rust-lang.org/reference/types/function-item.html). The IC strategy is a design inference, not a claim about an existing engine implementing this proposal.
