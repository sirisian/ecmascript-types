# Recursive alias formation and evaluation limits

The application below describes a finite recursive function contract. Its
parameter refers back to the object type; a function value does not contain
its parameter's storage inline.

```js
type N<T: type> = { f: (N.<T>) => void };
type X = N.<uint8>;
String(1); // "1"
```

Generic application does not change the alias recursion rule. A repeated
application with the same declaration context and completed arguments supplies
a reference to the type graph being formed. It does not require an infinite
sequence of expanded copies. The resulting alias remains structural and
transparent. Its formation context is not a new nominal identity.

Three outcomes must remain distinct:

- A productive graph whose cycles cross permitted reference positions denotes
  a type. Callable parameter and result positions are such positions. The
  original example and its non-generic spelling have the same recursive shape.
- A bare alias cycle denotes no type. An infinite inline layout is invalid.
  These are semantic errors, including when an application occurs in an
  uncalled function. A callable edge elsewhere does not excuse a separate
  inline cycle within the graph.
- A computation that cannot finish within the evaluation budget reports
  exhaustion. In particular, applications that keep changing their arguments
  need not form a finite graph. Native stack exhaustion is none of these
  outcomes.

## Required properties

Formation must preserve declaration identity, captured bindings, completed
type and value arguments, constraints and selected definitions. Identifier
spelling, source syntax alone, and diagnostic text are insufficient keys.
Different activations can supply different captures; equal completed
structural expansions can still share one Type Object.

Only a structural description edge may refer to its unfinished graph. It
supplies no completed type for an executable computation, argument admission,
reflection, member lookup or cache publication. Required defaults, predicates
and computed types retain their evaluation and failure rules. Forward type
resolution supplies no ordinary value-binding initialization and cannot make
an unavailable imported input available. Failed or suspended formation supplies
no successful result to another evaluation.

Complete the graph before its recursive validity judgment and public
interning. Preserve all independent obligations. A bare chain of forwarding
aliases cannot acquire an empty object type from its temporary graph nodes.
Temporary state must be removed on rejection, exhaustion and host interruption.
None of these compiler records are program references, and none extend
reference liveness or grant a storage permission.

## Directions considered

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reject recursive generic aliases, or require nominal wrappers | Adding an unused generic parameter changes acceptance. Autocomplete loses a contract the non-generic spelling can describe. | Rust requires a nominal participant in recursive types. That is a weaker analogy here: this proposal already admits transparent structural recursion. | Simple formation, but extra source wrappers can add nominal distinctions and specialization diversity. | Conflicts with the existing alias rule. | Reject. |
| Expand repeated applications until a depth limit stops them | A short finite contract fails depending on implementation capacity. The user cannot repair it by supplying the already explicit type. | Compiler recursion limits bound work, but are not a recursive type's meaning. Rust's layout rule does not require infinite unfolding. | Repeats compiler work and allocation proportional to the limit; gains no runtime shape or IC advantage. | Can report resource exhaustion, but does not implement finite recursive graph formation. | Reject as the semantic solution. |
| Close a cycle by name or declaration syntax alone | Reuse can silently select another specialization or captured binding; an editor displays the wrong members. | Neither Rust item identity nor TypeScript's recursive alias support establishes equality of different arguments or activations. | Cheap lookup with invalid cache hits; incompatible contracts can reach optimized code. | Conflates contexts and can waive constraints. | Reject. |
| Close a repeated structural application using its actual context and completed arguments; validate before publishing | Generic and non-generic spellings agree. Tools can describe the finite graph; invalid layouts get a semantic diagnostic. | TypeScript's delayed recursive alias resolution is a useful structural analogy. Rust supplies the finite-layout principle, while deliberately differing on nominal recursion. | Compiler graph nodes and dependency records avoid repeated expansion. No additional user-object fields, shape transitions or IC guards are required. | Preserves transparent identity and all obligations when unfinished nodes remain private and computations retain their own rules. | Recommend independently of implementation effort. |
| Make unfinished aliases observable lazy Type Objects | Reflection and construction can see different members before and after completion; a wrong guess may depend on evaluation order. | TypeScript has no runtime Type Objects. Rust does not expose a mutable, partially described type value. | Adds observable lazy states, repeated guards and more complicated publication. | Breaks completed Type Object identity and permits unfinished facts to escape. | Reject. |

## Containing resource exhaustion

Finite graph formation and stack containment are separate requirements. A
growing application can still visit arbitrarily many distinct nodes.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Catch a native stack exception and call it a type error | Diagnostics depend on host stack usage and can mask implementation defects. | Rust compiler limits are explicit compiler resources, not a program catch of a native crash. | Leaves work and cleanup uncontrolled; no shape benefit. | A native exception establishes neither invalidity nor semantic budget exhaustion. | Reject. |
| Increase the native stack | Moves the failure threshold without improving the contract or editor result. | The Rust layout analogy supplies no guarantee about an embedding process's stack. | Consumes more native memory and remains host dependent. | Does not ensure the configured limit can be reached. | Reject as the repair. |
| Only lower the default nesting limit | Some deep finite programs fail sooner; embedding-specific tuning remains necessary. | Explicit recursion limits are legitimate implementation resource controls. | Cheap containment for a measured host, without runtime shape changes. | Permitted when safe, but does not fix unbounded delegation or make larger configured limits safe. | Reject as the sole repair; retain resource limits. |
| Drive recursive type evaluation through an explicit continuation stack, retaining logical budget accounting | Diagnostics identify the language computation. Finite aliases are unaffected by host frame depth. | Compiler graph/worklist processing is compatible with Rust's finite-layout principle; Rust does not prescribe this interpreter mechanism. | Uses bounded compiler-owned work records and predictable logical limits. It changes no program object's representation or IC assumptions. | Preserves evaluation order, completions and cleanup while removing the recursive evaluator's native call chain. | Recommend together with graph formation and existing limits. |

These performance assessments describe implementation costs, not benchmark
results. The specification requires the outcomes and containment, not this
particular interpreter representation.

## Validation scope

The initial failure was reproduced at engine commit `587d83ea`: the default
depth overflowed the host stack, while depths 8, 32 and 64 reported the type
evaluation limit. The non-generic callable cycle succeeded. The old budget
test incorrectly grouped productive cycles, bare cycles and infinite inline
layouts under one expected failure.

The repair separates successful graph formation, semantic rejection and
budget exhaustion. Tests cover callable and container cycles, mutual forward
declarations, transparent identity, differing arguments and captures,
constraints, cleanup, and metered growing applications. Structural description
does not shortcut a recursive dependency used as an unfinished input to another
application or computation.

A separate probe, `type G<T: type> = { f: (G.<[].<T>>) => void }; type X =
G.<uint8>;`, exposes unmetered recursive obligation propagation in the static
checker, before builder evaluation begins. The same failure occurs with just
the generic declaration. Static propagation is now included in the accounting
boundary described below. The broader portable-limit and inference-completion
gates remain open.

The initial finite-graph repair's validation covered 466 tests across 27 files: 463 passed;
the two inspector snapshot failures and one generator diagnostic mismatch
also occur on the unchanged engine. The engine bundle and diagnostic checks
pass. The fallback TypeScript check and lint report the same baseline issues.
The specification's strict build reports the same existing unescaped
`Composite.<S>` markup error on both versions. The full extension suite was
not run.

Sources: [Rust recursive types](https://doc.rust-lang.org/reference/types.html#recursive-types),
[Rust function pointers](https://doc.rust-lang.org/reference/types/function-pointer.html),
and [TypeScript 3.7 recursive aliases](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#more-recursive-type-aliases).
Rust requires a nominal participant and finite size; its allowance for pointer
indirection supports the layout analogy, not unrestricted structural aliases.
TypeScript supports additional recursive aliases through delayed resolution,
but does not provide this proposal's runtime enforcement, type evaluation or
reference-liveness rules.

## Retained specialization obligations

Expanding an alias can erase the parameters that carried its admission,
default and predicate obligations. Checking must retain those obligations and
substitute their determining bindings when the existing specialization rules
require it. A short declaration can generate an unbounded dependency sequence
without invoking a builder or allocating a Type Object. Moving this work into
an earlier checking phase cannot move it outside the resource boundary.

The required properties are:

- Meter logical propagation through the existing type-evaluation allowance.
  Nested dependencies share it; unwinding cannot clear exhaustion.
- Preserve declaration-bound inputs and captured contexts. An active repeat
  suppresses only the same obligation, not different specializations or the
  independent obligations of a recursive type.
- Reach the configured diagnostic without depending on native recursive
  delegation. Compiler-owned continuations retain the traversal order and
  cleanup; they add no program-object shape, IC guard or reference permission.
- Report exhaustion in the phase that performed this work. It proves neither
  invalidity nor successful admission and supplies no fallback type.
- Reject the containing source before its body starts, withdraw its pending
  checking publications, and leave later checks independent. Ordinary generic
  deferral rules still determine which expressions require evaluation.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Meter only builder/runtime expansion | A declaration can crash tooling before any result or diagnostic exists. | Rust's compile-time limits apply to compiler operations; absence of user-code execution is not a general exemption. | Leaves substitution and dependency growth unbounded; no shape advantage. | Misses required static specialization work. | Reject. |
| Stop on the declaration name and accept, or use an unknown/any fallback | Autocomplete can lose deeper constraints and display a contract that checking never established. Wrong assumptions survive until use. | Rust goal identity and TypeScript structural recursion do not equate different generic arguments. | Cheap but unsound reuse can corrupt specialized code assumptions. | Drops distinct obligations and can change a successful type with the limit. | Reject. |
| Declare every argument-changing recursion invalid | Rejects finite cycles that alternate arguments and computations that eventually terminate. | Rust resource limits distinguish capacity from a general proof of semantic invalidity. | Cheap syntax checks cannot establish all relevant outcomes. No runtime shape benefit. | Growth at one step does not prove an invalid type. | Reject. |
| Defer all recursive obligation checking to later type evaluation | Conceals closed errors inside uncalled code or an alias whose result erases its parameter contract. Editor feedback becomes phase-dependent. | Rust deferred obligations depend on which judgment requires them; deferral is not universal acceptance. | Can avoid duplicate compiler work, but requires preserving every demanded obligation. | Blanket deferral changes required checking phases. Existing explicit generic deferral remains valid. | Reject as a general policy; retain demand-driven evaluation where already specified. |
| Spend the return-inference allowance instead | Reports a return-inference failure even with explicit result annotations and no inferred result. | Rust has distinct controls for different compiler resources; it does not establish this proposal's diagnostic classification. | Reuses a counter but couples unrelated work. No IC benefit. | Specialization is already type computation; relabeling it obscures the resource and phase. | Reject. |
| Add only a native-recursion depth guard | Better diagnostics on one measured host, but larger configured limits retain the same failure mode. | Explicit compiler depth limits are legitimate controls, including Rust's. | Low implementation overhead; native frames still scale with the path. | A sole guard does not remove the recursive traversal's host dependence. | Reject as the sole repair; keep logical depth accounting. |
| Meter retained propagation and drive recursive dependencies through explicit continuations | Editors and hosts receive evaluation-limit exhaustion; a larger allowance can enable completion without inventing a different contract. | Rust's compiler limits and depth-sensitive solver caching support careful resource accounting; their exact semantics differ. | Compiler work records replace nested native frames. Repeated obligations retain exact input checks. Program shapes and ICs are unchanged. | Preserves obligations, phases, sticky failure and cleanup without interpreting native exceptions as language evidence. | Recommend independently of implementation effort. |

The engine uses its existing type-evaluation counters for this propagation,
including in the initial static pass. It charges a specialization request
before checking for an active repeat. The continuation stack prevents nested
alias-obligation traversal from retaining a native call chain. Exhaustion
stops further propagation and the containing checking passes; failed pending
inference and binding publications are withdrawn. It is not an inferred result
and cannot be dismissed by dynamic-function diagnostic recovery.

The implementation has not become a portable total fuel model. Other recursive
type operations still need their own capacity audits, and configured limits
must remain within the implementation's capacity. These performance statements
are cost analysis, not benchmark results.

[Rust's limits](https://doc.rust-lang.org/reference/attributes/limits.html)
bound compile-time recursion and type-substitution growth; the documented type
length limit currently requires a nightly enforcement flag.
[Rust's new trait solver](https://rustc-dev-guide.rust-lang.org/solve/caching.html)
accounts for remaining depth in cache reuse. Its overflow can produce
ambiguity, which differs from this proposal's required failure without a
fallback type. Neither analogy authorizes silently accepting unfinished work.

At engine base `d258023d`, the growing callable example, its unused declaration,
and nullable and mutual variants escaped with a native stack exception. The
repair reports `RT_TYPE_EVALUATION_LIMIT` in the static phase at default and
explicit depths 8, 32, 64 and 256. Step exhaustion, finite alternating inputs,
a finite chain that succeeds with more fuel, erased constraints, failed-module
recovery, dynamic Function checking and parsing without an active execution
context have focused coverage. Across 21 selected files, 667 distinct tests
pass. The engine build and diagnostic checks pass; TypeScript, lint and the
strict specification build retain their baseline failures. The full extension
suite was not run.
