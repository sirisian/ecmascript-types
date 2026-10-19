# Declaration identity in callable replacement contracts

Instantiating an owner's contract substitutes its bound type positions. It does not introduce new nominal declarations, change an existing class evaluation's identity, or bind a nested generic function's own parameters. Use the resulting parameter list to distinguish a replacement from an additive overload, and require a replacement's result to be a subtype of the instantiated owner result.

```js
function make() {
  class C {}
  function f<T: type>(value: T): C { return new C(); }
  function f<uint8>(value: uint8): C { return new C(); }
  return f;
}
const f = make();
const selected = f.<uint8>;
const value = selected(1);
```

Both annotations name the same declaration in this activation. Substitution cannot turn the owner's `C` into a different class and then diagnose `C` as incompatible with itself. A class expression bound to an immutable local follows the same rule. Separate calls of `make` still introduce separate class identities.

The converse matters too:

```js
class C {}
function f<T: type>(value: T, context: C): string { return "owner"; }
function f<uint8>(value: uint8, context: C): number { return 1; }
```

This is an incompatible replacement and fails before the Script or Module body evaluates. Copying the owner's `C` identity would make the parameter lists look different, incorrectly classify the case as additive, and miss the declaration error. A genuinely different parameter type still makes an additive overload.

## Requirements and choice

A valid direction must preserve nominal declaration and evaluation identity, generic argument identity, nested binder scope, replacement classification, result covariance, early error timing, and the existing reference permissions and lifetime. It must support captured types without executing the owner or attached callable bodies or ordinary parameter defaults during checking. It must not let a tested value define its own receiving contract.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Clone every object reachable from an owner's type, replacing matching names | Equivalent annotations can fail; incompatible replacements can escape checking. Autocomplete and diagnostics disagree with source declarations | Rust preserves item identity and lexical generic scope; copying compiler metadata is not a new source declaration | May copy syntax, parent graphs and unrelated metadata; allocation cost is unrelated to type size | Changes declaration identity and binds same-named nested parameters | Rejected: both false rejection and false admission |
| Compare class names or structural members after substitution | Same-shaped unrelated classes become interchangeable; wrong guesses can survive until invocation | Rust function-item identities remain distinct even for matching signatures; its nominal items are not identified by spelling alone | Cheap comparisons lose identity guards; additional fallback guards cannot restore discarded distinctions | Violates nominal types and cannot distinguish different evaluations of a local class | Rejected |
| Forbid local nominal types or nested generic callable types in replacement contracts | Ordinary refactoring into a factory fails; useful member and callable information disappears | Rust generic scope is explicit, but does not justify forbidding this proposal's capturing functions | Simple restriction reduces supported programs | Avoids some failures by rejecting contracts already supported elsewhere | Rejected |
| Defer all replacement compatibility to invocation | Incorrect declarations survive checking and fail only on a selected call | Rust checks generic declarations within their declared bounds; it does not defer all static compatibility until execution | Adds invocation work and delays optimization assumptions | Loses required declaration failure timing; cannot repair incorrectly classified additive cases | Rejected |
| Substitute only type-bearing positions, keyed to owner binders, preserving declaration identities and nested scopes | Written and inferred views agree; valid refactoring works; wrong replacements fail at their declarations | Consistent with Rust's distinction between item identity and type arguments, and its lexical parameter scopes, within the limits below | Visits the type graph with sharing/memoization and reuses identity tokens. No public object fields, user-object shape changes, or new inline-cache invalidation rule | Retains nominal and activation distinctions, covariance, binding authority, modes and liveness | Recommended independently of implementation effort |

Choose the last direction. Apply it to both parameter comparison and return comparison. Composite types retain the identity of every nominal component. A nested callable's own generic parameters remain bound by that callable, even if they use an owner parameter's spelling; free occurrences of the owner parameter in that nested callable still substitute. This does not change the type relation or weaken admission to accommodate an implementation artifact.

Rust's [function item types](https://doc.rust-lang.org/reference/types/function-item.html) distinguish the defining item and its type arguments; coercion to a function pointer is a separate operation. Its [generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes) distinguish inner item binders from outer binders. These are useful identity and scoping analogies, not a specification of this proposal's overloads: Rust inner items cannot capture outer generic parameters, whereas this proposal permits captured dependencies and has runtime class evaluations. See also Rust's [generic parameters](https://doc.rust-lang.org/reference/items/generics.html).

## Implementation and audit scope

The engine checker previously used a general object-copying substitution in its final callable-group analysis. It copied nominal declaration nodes, and its name lookup ignored nested generic binders. Both contract comparisons now use the existing declaration-aware free-parameter substitution. This repairs the declaration comparison; [stored specialization completion](stored-specialization-bindings.md) still supplies the operation-owned contract at runtime. Neither operation grants reference authority or extends lifetime.

The focused audit covers captured declarations and class expressions, repeated selections and separate factory activations, nominal components in records/arrays/tuples/unions/callables, generic nominal arguments, inheritance direction, methods, nested generic binders, and failure before body evaluation. It is not closure of the overall inference audit.

At the time of this declaration-comparison repair, two factory-publication probes remained open. The follow-up [generic callable publication repair](generic-callable-publication.md) addresses both probes and the selected capture binding that they expose. The following records the alternatives considered before that repair. Returning an unapplied generic function whose inferred result is `P<U>`, or a captured-width replacement whose result is `P<N>`, produces a receiving contract stronger than the factory's published nested callable contract. The trace shows that the latter has no inferred result at the point it supplies completion evidence. This is a producer-publication problem, separate from declaration substitution.

| Direction for the remaining publication question | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase the receiving callable's inferred result | Loses expected result members and specialization information | Does not follow from Rust's identity-bearing function items | Cheap but gives up useful guards | Hides the disagreement instead of preserving established evidence | Rejected |
| Require explicit result annotations whenever a factory returns a generic callable | Users must repeat inferable contracts; autocomplete depends on extra annotations and factory refactoring becomes fragile | Rust normally writes item return types, but that is not this proposal's established participating-inference rule | Reduces dependency inference work by restricting the language | Avoids an implementation gap by withdrawing existing inference commitments | Rejected |
| Fill the receiver target from the arbitrary returned value | A wrong replacement can change the obligation it must satisfy | No comparable Rust rule makes a destination depend on the candidate it checks | Runtime reflection and unstable contract assumptions | Circular admission; loses independent contract authority | Rejected |
| Complete the producer's participating callable dependencies before publishing its result, preserving the owner's public contract rules | Refactoring through a factory preserves usable types; invalid replacements remain errors | Consistent with declaration-scoped generic checking, though Rust lacks this runtime publication boundary | Dependency work can be cached by complete query inputs; no user shape change is needed | Can preserve independent evidence, declaration scope and liveness, provided convergence and public owner/case projection agree | Chosen for the subsequent generic callable publication repair |

The follow-up repair establishes when the contribution-local task can read a generic class's declaration information so the factory can publish its completed callable dependency. The captured-width example additionally requires agreement between the checker's owner-plus-case view and the runtime owner's public generic value. Do not infer that fixing publication alone closes both probes.
