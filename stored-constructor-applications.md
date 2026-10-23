# Stored constructor applications

Selecting a constructor fixes the arguments used by subsequent construction, instance members, static members and heritage. Ordinary immutable aliases and module exports retain that selection.

```js
class Box<T: type> {
  value: T;
  constructor(value: T) { this.value = value; }
  get() { return this.value; }
}
const Selected = Box.<string>;
const value: boolean = new Selected("ok").get(); // static type error
```

The checker describes the application where it is produced. It binds positional, named and spread arguments and completes defaults under the existing rules. A consumer uses those bindings; its contextual type and constructor arguments do not infer them again. This applies through local copies, named/default exports, re-exports and resolved namespace members. A selected value alias introduces no new type declaration or class identity.

## Requirements and alternatives

The solution must preserve selection, source and activation identities, default/checking timing, declared erasure and mutation boundaries, and reference permission and liveness. It must provide the same member information after factoring selection out of `new`, without evaluating a body, getter, constructor or initializer to discover a contract. Incomplete checking dependencies must remain incomplete.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Erase every stored application | Autocomplete loses members; a known wrong destination fails only at runtime | Rust function item values retain their arguments; TypeScript instantiation expressions retain selected constructor signatures | Little metadata, but removes useful guards and specialization information | Discards a known contract without a reference-liveness benefit | Rejected |
| Retain the declaration and infer arguments again at `new` | Moving an expression changes its selected type; an incorrect contextual guess can silently retarget it | Neither Rust's explicit item arguments nor TypeScript instantiation expressions behave this way | Repeats binding and complicates specialization-cache keys | Contradicts completed selection and default timing | Rejected |
| Replay the argument syntax in the consumer scope | A same-spelled import, alias or inner generic changes meaning | Both Rust and JavaScript use lexical name resolution | Cheap syntax retention, repeated lookup and unstable dependency keys | Confuses declarations and may replay a default; no liveness advantage | Rejected |
| Reject applications capturing outer parameters | Valid direct constructions stop working when factored into a stored selection | Rust nested items cannot capture outer generics, but that restriction is not this proposal's closure model | Simple restriction with no engine necessity | Removes existing capturing behaviour | Rejected |
| Attach capture context only to the constructed instance | Completion works for the instance but changes after a member result is extracted | Rust's argument retention applies throughout the type; its nominal model does not justify dropping a constituent dependency | Small root descriptor, but every extraction would need special repair | Loses binder identity in instance/static member results and inherited contracts; no liveness benefit | Rejected |
| Retain bound arguments, with each unresolved free parameter tied to its producing declaration | Member completion and error locations survive storage and import; shadowing is unsurprising | Rust item arguments and TypeScript selected constructor signatures support retention; runtime type capture and class activation identity are proposal-specific | Share compiler/application descriptors and existing specialization caches. Internal metadata need not change user-object shapes or inline caches | Preserves selection, pending obligations, nominal identity and existing reference checks | Recommended independently of implementation effort |

Choose the last direction. A parameter dependency must survive below the instance record: extracting a member result, specializing a static method contract, or inheriting a member must not discard it. Recording context only on the constructed instance was considered and rejected because these operations return constituent types. Name-only substitution was rejected because unrelated binders can have the same name. Declaration-bound symbolic parameters implement the existing substitution rule throughout those constituent types. No new runtime type identity or public property is introduced.

## Captured parameters and completion

```js
function make<T: type>(value: T) {
  class Box<V: type> {
    value: V;
    constructor(value: V) { this.value = value; }
    get() { return this.value; }
  }
  const Selected = Box.<T>;
  function read<T: type>(ignored: T) {
    return new Selected(value).get();
  }
  return read;
}
const text: string = make.<string>("ok").<uint8>(1);
```

The selected argument is the outer `T`. Applying `read` binds its own `T` and does not replace that capture. The same principle applies to value parameters, dependent defaults, composite results and inherited/static members. A retained selection may be described symbolically before an enclosing application supplies the concrete bindings. An unevaluated required default remains a dependency, not an inferred `any`, an omitted argument, or permission to choose a different default. Evaluation and failure retain their existing checking phase.

Completed inferred callable results close source class dependencies in the callable's declaring activation. Different calls of `make` still create different local class families. A replacement through `any` cannot redefine the previously inferred extraction contract. Publishing a selection never establishes an exact origin for a mutable, erased or decorator-replaceable source.

## A new explicit selection

Construction from a stored selection and a fresh written application are separate operations. The existing class-family operation permits `const Other = Selected.<boolean>` to select the same evaluated generic family with a new argument list. The first selection remains unchanged. The new list binds from scratch, including its own omitted defaults; it does not inherit unspecified arguments from the old selection.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reject all new explicit lists on a selected class constructor | Simple nongeneric-value model, but removes the existing class-family operation | Close to Rust applied types and TypeScript instantiated constructor signatures; neither models this proposal's retained runtime family capability | A small capability guard | Consistent only after deliberately removing the existing family operation | Rejected; the analogy alone does not justify that removal |
| Merge new arguments with the previous selection, or implicitly reopen it during construction | Omissions and contextual guesses can silently reuse or change old choices | No close Rust or TypeScript precedent | Extra merging state and cache-key ambiguity | Conflicts with normal argument binding and fixed stored selection | Rejected |
| A fresh explicit list selects the retained family anew; ordinary construction uses the stored arguments | The explicit operation identifies when selection changes; old aliases remain stable | Rust requires a new application of the generic declaration; this proposal additionally retains that declaration capability on class constructors | Reuse family identity, argument binding and specialization cache; no new instance shape | Preserves current class-family behaviour, default order, activation identity and reference rules | Recommended independently of implementation effort |

The recommendation concerns class constructors with an established generic family. It does not grant a new generic capability to an arbitrary value or change generic-function selection.

## Prior art and verification scope

Rust [function item types](https://doc.rust-lang.org/reference/types/function-item.html) carry item identity and type arguments. Rust [generic scopes](https://doc.rust-lang.org/reference/names/scopes.html) disallow capturing outer generic parameters from nested items, so they are not a direct model of the captured example. TypeScript [instantiation expressions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#instantiation-expressions) specialize callable and constructor signatures; its erased runtime does not supply this proposal's admission, activation or reference rules.

The proof of concept covers stored selections through established constructor origins, producer-owned module publication, captured type/value parameters, member/heritage consumers, defaults and existing explicit family reselection. The corresponding bounded tests include successful controls, static/pre-evaluation refusal, runtime erasure and same-realm failure recovery. Getter receiver inference and broader convergence, resource and full-regression gates remain separate work.
