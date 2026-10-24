# Annotations see adopted generic scope

A fresh function, arrow or object method that adopts a generic signature can name its own parameters in written annotations. The absence of a written generic list does not make those references closed expressions:

```js
type Adapter<C: type> = <U: type extends C>(value: U) => { captured: C, own: U };
function factory<T: type>(seed: T) {
  const adapter: Adapter.<T> = (value: U) => {
    const copy: U = value;
    return { captured: seed, own: copy };
  };
  return adapter;
}
const adapter = factory.<number>(0);
const result = adapter.<number>(0);
```

`U` belongs to the adopted signature; `T` belongs to the creating activation. Classify an annotation's dependencies using the selected effective own scope, including adopted declarations. An unresolved own parameter stays symbolic until its ordinary binding point. Revisiting the nested literal during enclosing specialization does not supply its own parameters from a same-named outer declaration. Functions, arrows and methods follow the same rule, including parameter, result and body annotations and nested closures.

This is not blanket deferral of a generic body. An independent missing name, a closed throwing type builder, or an established incompatible assignment retains its required check and phase. A fragment such as `U | Missing` still checks the independent `Missing` obligation. Selected parameter defaults retain ordinary per-call evaluation and admission; supplying an argument bypasses its default. Checking an open annotation does not execute a callback, initialize a capture, manufacture reference authority or extend a lifetime. Restore effective scopes after every completion.

## Phase decision

Required properties are declaration identity, consistent completion and reflection, symbolic own inputs, independent closed checks, phase/effect preservation and failed-check recovery. Each direction below is assessed independently of its implementation effort.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require a written generic list whenever any annotation names an own binder | Completion advertises a contextual generic signature that annotations cannot use; adding a redundant list becomes a surprising repair. | Rust context can infer closure inputs, but Rust does not directly model reified contextually adopted JavaScript generic lists. | Saves a metadata lookup at the cost of inconsistent language rules. | Contradicts adopted body scope and the effective callable contract. | Reject. |
| Treat every unrecognized name in an adopted body as pending | Easy examples work; typos survive until a later invocation or disappear in unused code. | Rust distinguishes inference variables from unresolved names. | Less early checking shifts failures into runtime paths and weakens specialization. | Suppresses independent closed errors and changes failure order; no liveness justification. | Reject. |
| Substitute an outer argument for every matching spelling | Completion looks concrete but silently describes the wrong declaration when names repeat. | Rust binder scopes distinguish declarations; nested Rust items do not inherit outer generics. | Name maps are cheap but produce invalid cache entries when declaration identity matters. | Rebinds nested own parameters and conflates activations. | Reject. |
| Insert adopted syntax into the shared parse tree | Tooling appears uniform until another context or trial adopts different records. | Rust syntax and inferred types are distinct compiler inputs; this is no precedent for mutating shared source. | Shared AST mutation needs invalidation and complicates speculative checking. | Risks cross-trial and cross-activation scope leakage. | Reject. |
| Use selected effective own declarations for dependency classification and nested specialization | Completion and written annotations agree; typos still fail at the specified phase. | Contextual Rust closure inference and lexical binder identity are useful analogies, with the differences above. | Compiler-owned scope records need no public properties, user-object shape changes or property IC keys. Reuse the selected record rather than recomputing a runtime contract. | Preserves own/captured identities, independent obligations, effect order and reference rules. | Choose. Performance is architectural reasoning, not a benchmark. |

Primary prior art: Rust [closure expressions](https://doc.rust-lang.org/reference/expressions/closure-expr.html), [E0401](https://doc.rust-lang.org/error_codes/E0401.html), [scopes](https://doc.rust-lang.org/reference/names/scopes.html) and [enumerations](https://doc.rust-lang.org/reference/items/enumerations.html). Rust's static closure types and ownership model do not determine this proposal's runtime checking or reference-liveness rules.

## Follow-up decisions exposed by the audit

The audit exposed two adjacent implementation questions. The lexical precedence repair is now described in [adopted generic shadowing](adopted-generic-shadowing.md); the uniquely compatible union-context repair is described in [contextual union selection](contextual-union-selection.md), with overlapping alternatives retained as follow-up. The decisions below retain the alternatives considered.

### Lexical alias precedence

An adopted own `U` must shadow an outer `type U = string`; a more local block alias must in turn shadow that own `U` only inside the block. The original audit failed both directions for functions, arrows and methods; the subsequent shadowing repair covers all six probes.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Always prefer a syntax-resolved alias | An outer alias unexpectedly overrides the signature advertised by completion. | Unlike Rust's lexical scopes; the absence of written syntax is an implementation detail here. | Existing lookup is cheap but chooses the wrong declaration. | Erases adopted own scope. | Reject. |
| Always prefer any active generic binder | A local alias appears valid in completion but is ignored at runtime. | Unlike ordinary lexical shadowing, including Rust. | A flat map is cheap but lacks scope ordering. | Misbinds local annotations; does not establish any reference permission. | Reject. |
| Resolve effective declarations by lexical scope and declaration identity | Completion and annotations agree at each source location; an inner alias has a bounded effect. | Rust lexical scopes are the close analogy; Rust nested-item restrictions still differ. | Compiler scope metadata and identity-based lookup need no user-object shape changes or new property IC polymorphism. | Preserves both shadowing directions, captures and independent reference rules. | Chosen and implemented in the subsequent shadowing repair. Written function/method controls confirmed that bypassing inner aliases was also a runtime issue without adoption. |

### Complete union contexts

For `{ tag: "bad", fn: (value: string) => string } | { tag: "good", fn: <U: type>(value: U) => U }`, the literal `{ tag: "good", fn: (value: U): U => value }` needs the complete compatible alternative. Reversing these arms currently changes admission for functions and arrows.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep the first member-local trial that reports no immediate error | Reordering equivalent union arms changes completion and whether the literal works; a deferred error is mistaken for success. | Rust enums are explicit nominal alternatives, not structural contextual unions, so they do not justify this order dependence. | Cheap early exit can select a contract that the enclosing literal cannot inhabit. | Combines incompatible alternative evidence and risks retaining failed-trial state. | Reject. |
| Merge generic lists and parameter contracts across arms | Completion invents a callable that no complete arm declares; wrong guesses become hard to diagnose. | Rust enum variant constructors retain variant-specific contracts rather than merging them. The analogy is limited by explicit construction. | Merged signatures increase union/inference work and weaken specialization. | Invents binder correspondences and can waive required admission. | Reject. |
| Require users to split or annotate every such literal manually | Predictable but imposes extra syntax for an otherwise established compatible arm. | Explicit Rust enum constructors are coherent in Rust; imposing them here changes this proposal's structural rules. | Simple implementation does not justify rejecting the specified compatible literal. | Avoids inference by dropping an existing contextual obligation. | Reject. |
| Check complete alternatives and commit only the selected compatible context | Completion describes a viable whole object and arm order does not change this uniquely compatible example. | Rust's variant-specific contracts support keeping alternatives coherent; its nominal enums do not specify structural-union selection. | Isolated compiler trials can share immutable records and prune using established discriminants; no runtime wrapper or object-shape change is required. | Preserves correlated members and binder identities; losing trials must restore obligations and state. Ordinary evaluation and reference liveness remain separate. | Recommend. Audit ambiguity and existing ranking rules before generalizing beyond these probes. |

## Bounded implementation result

The checker now includes selected adopted parameters when classifying annotation dependencies and when excluding a nested declaration's own names from enclosing specialization. Arrow checking enters and restores the same effective own scope used by functions and methods. The source AST and runtime value layouts are unchanged.

The independent Module audit improves from 23/88 to 80/88 requirements satisfied. The original six-probe queue is fully satisfied, including both explicit-annotation failures. Permanent coverage includes the 80 conforming cases, their 77 applicable Script versions and three same-realm failure/recovery sequences. The original eight remaining probes were explicit follow-up evidence: six lexical-alias failures and two union-order failures. The subsequent shadowing repair closes the six lexical-alias probes; the subsequent [complete-shape contextual selection repair](contextual-union-selection.md) addresses the two union-order probes; overlapping alternatives remain a separate audit. Passing a negative case means the required diagnostic, phase and body-entry boundary were observed. This bounded repair does not close the broader generic-context, convergence, phase, resource or specification-determinacy gates.
