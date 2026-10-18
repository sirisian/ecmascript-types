# Evaluated nongeneric class arguments

A nongeneric local class still has an evaluated identity. Making it an argument of a generic class must retain that identity. Repeated calls to a factory create distinct classes even when their source, captured values and visible members agree.

```js
class P<T: type> { constructor(value: string) {} }
function make() {
  class C {}
  return (value: any) => { const p: P.<C> = value; return p; };
}
const first = make(), second = make();
const holder = { p: first("x") };
function replace(raw: any) { raw.p = second("y"); }
replace(holder);
const p = holder.p; // TypeError: the retained view belongs to first's C
```

The ordinary backing write remains possible. The receiving view retains its independently established type. The same replacement through `first` is admitted. Autocomplete can retain the declared members of `P` without treating all evaluations of `C` as one argument.

## Requirements and alternatives

The solution must distinguish separate class evaluations, preserve repeated equal applications, agree across nominal comparison and interning, retain specialization reuse across forward completion, respect lexical shadowing and declaration-based types, and preserve reference liveness. Checking templates must remain independent of any one activation. Evaluating a target cannot inspect the candidate instance or rerun the producer.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Identify nongeneric arguments by source declaration alone | Same-looking types unexpectedly accept incompatible replacements; autocomplete conceals the distinction | Rust has nominal types, but its nested items are not JavaScript class evaluations and do not capture outer generic parameters | Small keys but unsound interning and IC assumptions | Merges distinct evaluated classes | Rejected |
| Add constructor-pointer inequality only at the receiving boundary | The immediate wrong replacement fails, while reflection and nested specializations can still disagree | Rust uses type identity consistently, not a separate receiving-boundary convention | Extra guard is cheap but leaves caches keyed inconsistently | Forward records lack a constructor; other paths still conflate identity | Rejected |
| Inspect the instance's public constructor or prototype to choose its argument | A wrong replacement can choose its own acceptable target and trigger observable reads | Rust generic arguments are not inferred from arbitrary public properties | Adds effects and unstable property guards | Circular admission; supplies no liveness permission | Rejected |
| Disable specialization reuse or allocate a fresh identity for every use | Repeating the same application changes reflection and may repeat static effects | Rust distinguishes nominal types while preserving repeated use of the same type | Loses code/type sharing and increases allocation | Does not define stable equality or forward completion | Rejected |
| Carry the established evaluated identity through every nominal record and argument key | Distinct activations reject predictably; aliases and equal applications retain members and identity | Rust supports nominal distinctions and scoped binders. ECMAScript class evaluation is the closer model for per-evaluation identity | Internal tokens and shared descriptions support stable family/argument guards without new user-object properties or shape transitions | Consistent comparison, interning, forward completion and cache reuse; no new reference permission | Recommended independently of implementation effort |

Choose the final direction. A generic class uses its family identity plus ordered arguments. A nongeneric class uses its evaluated identity with no arguments. This does not make constructor objects interchangeable with unresolved declaration templates.

Rust's [type categories](https://doc.rust-lang.org/reference/types.html) distinguish nominal types, and its [generic-parameter scope](https://doc.rust-lang.org/reference/items/generics.html) excludes outer generic parameters from nested item definitions. Those are limited analogies, not a model of this proposal's capturing factories. ECMAScript [ClassDefinitionEvaluation](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-runtime-semantics-classdefinitionevaluation) creates a class constructor on evaluation; the proposed type identity follows that evaluated class while preserving type-declaration lookup.

## Forward completion and caches

A forward class reference retains the identity of its particular lexical binding. Completing that binding supplies the constructor and class description for that identity. It must never select the latest constructor recorded for the source declaration, including after another activation has run. Equal argument keys remain equal before and after completion, so static effects do not repeat merely because an argument's constructor became available.

The internal identity can be shared by symbolic references closed over the same activation, aliases, checked contributions and runtime class records. The shared checking template stays symbolic. Neither a class's ordinary value-binding reassignment nor a same-spelled inner binding replaces an already completed identity.

## Composite argument correspondence

Making identity explicit also requires completing a class nested inside an argument description such as `P.<[C]>` or `P.<C | null>`. This is correspondence between checked type descriptions, not a claim about elements read from a runtime array. Keep each target's extent, tuple modes, metadata and other structure, then require exact argument equality. Union and intersection member order is immaterial; an ambiguous correspondence cannot select one of several evaluated classes.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Complete only direct nominal arguments | Wrapping the same class in a tuple or array unexpectedly rejects valid calls and loses reliable completion | Rust retains nominal arguments through composite types, though it does not have this runtime closure model | Cheap traversal with incomplete results | Rejects a valid retained dependency | Rejected |
| Pair every composite by position or choose the first matching member | Union spelling or normalization order can change the chosen activation | Rust has no direct union-set analogue; its tuples are ordered, which does not justify ordered union identity | Simple traversal but order-dependent guards | May select an arbitrary class when several candidates fit | Rejected |
| Complete corresponding argument structure and require exact equality, with unique set-member correspondence | Equivalent argument spellings work; wrong domains, modes and ambiguous identities remain errors | Rust's composite types preserve component identity; union matching follows this proposal's own set semantics | Memoized graph traversal; unordered candidate matching can be indexed by stable shape in a production engine | Retains the target's structure and independent evidence without reading values or extending liveness | Recommended independently of effort |

Choose the final direction. This bounded extension handles type descriptions inside nominal arguments; broader transport through arbitrary array/tuple result values remains a separate audit.

## Fresh callable initialization

An immutable binding initialized directly by a function expression or arrow must retain that fresh function's independently established captured result dependency. Two wrappers forwarding the same captured converter can then agree before either body executes. The implementation verifies that the created function's code belongs to the initializer syntax and completes the inferred result from lexical declarations and immutable forwarding sources. It does not use an arbitrary candidate callable's runtime type to rewrite a destination.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Leave every fresh wrapper's result symbolic | Introducing an immutable wrapper can make previously valid code fail | Rust retains a closure's inferred result, but its closure representation differs | Avoids completion but loses useful specialization guards | Does not instantiate an established captured dependency | Rejected |
| Adopt the runtime signature of every initializer value | A replacement can redefine the type it is supposed to satisfy | Rust's inference does not authorize rewriting a declared destination from an arbitrary replacement | Requires signature inspection on unrelated values | Circular admission and loss of authoritative targets | Rejected |
| Complete the result of a verified fresh callable initializer from its own declarations | Wrapping a captured callable preserves usable members and compatible results | Rust supports retaining inferred callable results; source/capture verification is specific to this runtime proposal | Creation-time internal context work can be shared with signature publication; no public shape changes or extra calls | Completes only unresolved inferred dependencies, preserving body effects, target obligations and reference rules | Recommended independently of effort |

Choose the final direction. Parentheses do not change initializer identity. Written annotations remain authoritative, and general replacement values or overload correspondence do not acquire this exception.

## Coverage and scope

The regression corpus exercises declared, anonymous and named expression classes, nested and structural argument forms, wrong and correct replacements, aliases, forward completion, recursive factories, reflection, heritage, cache reuse and erasure. This completes the queued nongeneric argument identity repair within the broader declaration-identity audit; it does not close the remaining convergence, checking-phase, budget, specification-only or full-regression gates.
