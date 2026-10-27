# Alias contexts after preparation and in eval

A contextual signature belongs to the declaration and binding that supplied it. Its bound parameter names cannot be recovered from the canonical Type Object: alpha-equivalent signatures share that object. This applies after a computed alias is prepared and when direct eval parses a new source without import declarations of its own.

```js
// dep
export type F = <U: type>(value: U) => U;
export type G = <V: type>(value: V) => V;

// main
import * as ns from "dep";
eval(`
  const f: ns.F = function(value: U): U { return value; };
  const g: ns.G = function(value: V): V { return value; };
  if (f.<number>(7) !== 7 || g.<string>("ok") !== "ok") throw new Error("context");
`);
```

The named-import and class-field equivalents have the same requirement. An eval selecting `G` must adopt `V` even if `F` was interned first. Autocomplete based on the selected signature should not depend on intern order. Runtime reflection still reports equal types.

## Decision: recover the selected binding, then check its consumers

A valid solution must preserve alpha-equivalence, binding/activation identity, lexical shadowing, the eval source's own declaration environment, module availability and ordinary reads. It must not execute properties to discover provenance, turn adopted binders into free names, skip independently closed checks, change failure precedence, or grant reference permissions/liveness.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require repeated generic declarations in eval and computed-alias consumers | Moving valid code into eval or changing an alias initializer breaks adoption; completion still shows the intended signature. | Rust preserves declaration scopes but has no matching runtime eval. ECMAScript direct eval retains lexical scope. | Smaller checker but duplicated source and missed context reuse. | Makes contextual-signature rules depend on source packaging. | Reject. |
| Recover names from the interned Type Object, or a realm-wide name table | Warm-up order or unrelated declarations determine whether the user's spelling works. | Rust declaration resolution does not select an equal type's spelling; ECMAScript resolves names through environments. | Cheap lookup but unsound cache keys; a name table also retains unrelated activations. | Loses declaration identity, shadowing and captured context. | Reject both variants. |
| Execute ordinary reads, getters or builders again to discover context | Checking can trigger effects before the source runs; completion no longer predicts context selection. | Neither Rust static paths nor ECMAScript eval authorizes these extra reads. | Effectful calls impede checking reuse and need invalidation; replay duplicates work. | Moves effects, TDZ failures and dependency execution; cannot establish reference liveness. | Reject. |
| Use actual alias publication and stable namespace-binding evidence in the applicable environment; refresh dependent obligations after preparation | Named/qualified paths and separately parsed code adopt the selected signature. Invalid bodies fail before their source runs. | Rust supports declaration-based scopes, with no direct eval analogue. ECMAScript specifies the lexical/global split for direct/indirect eval. | Private environment/linked-export metadata; no user-object property, wrapper or shape change. Namespace ICs can use existing binding guards. A dependency worklist can refresh affected judgments. | Preserves context independently of equality, ordinary reads, unavailable bindings, failure precedence and liveness. | **Recommend**, independently of implementation effort. Implemented for the scope below. |

These are feasible implementation costs, not measurements. Rust's [generic scopes](https://doc.rust-lang.org/reference/items/generics.html) and [paths](https://doc.rust-lang.org/reference/paths.html) motivate declaration identity only; Rust does not specify runtime Type Objects or this proposal's contextual binder adoption. ECMAScript's [PerformEval](https://tc39.es/ecma262/multipage/global-object.html#sec-performeval) supplies the actual direct/indirect environment distinction.

An external initialized immutable slot containing a genuine module namespace can identify its module without a property read. An ordinary object, Proxy, mutable slot, reference slot, uninitialized binding or opaque object environment supplies no such evidence. A nearer binding blocks lookup through it; it does not authorize trying a same-spelled outer alias. A non-alias value containing an equal Type Object does not acquire another declaration's provenance.

The initial eval check has its applicable outer environment. After declaration instantiation and type preparation, rechecking uses eval's own declaration environment over that outer environment. This matters for aliases prepared inside eval and aliases from enclosing activations. Indirect eval keeps its global environment.

Computed alias results may become available after the initial syntax-based traversal. Adopt their effective binders before consuming dependent annotation obligations from that traversal. Otherwise valid `function(value: U): U` syntax can spuriously evaluate `U` as a closed, unbound type. Independent closed annotations and invalid bodies still require checking. No ordinary value initializer is executed to obtain missing inputs.

## Coverage

The portable engine corpus `prepared-alias-contexts.json` covers named, renamed, namespace and nested/re-exported routes in direct eval; immutable aliases; bindings, fields, arrows, arguments, returns, assignments, objects and tuples; reverse/warm order and alpha-equivalent signatures; local and outer shadows; private dependencies; repeated captured contexts; nested eval and locally prepared computed aliases. Computed aliases with one signature spelling work through named and namespace imports before binding or field initializer checking. Invalid bodies diagnose before the consumer body; a failed eval leaves its own body marker unset.

Controls cover indirect eval, ordinary stored Type Objects, mutable namespace slots, Proxies, lookalike objects, getter/trap counters, TDZ cycles, failed producers and consumers, same-realm recovery and the feature-off path. These preserve existing refusals; they do not define a new policy for arbitrary runtime property-based type computation.

An adjacent old fixture expected budget exhaustion from `type R<T:type> = R.<T>`. The pulled engine already rejects this nonproductive alias because it denotes no type. The fixture now expects that type-evaluation diagnostic and retains its subsequent independent successful evaluation. Finite structural recursion and a growing computation that actually exhausts its budget remain distinct cases.

## Remaining: computed result context before alias publication

The following reconstructed pair remains unresolved; preparation alone is insufficient:

```js
// dep
function makeF(): type { type R = <U:type>(value:U)=>U; return R; }
function makeG(): type { type R = <V:type>(value:V)=>V; return R; }
export type F = makeF();
export type G = makeG();

// main (also test named imports and binding initializers)
import * as ns from "dep";
class C {
  f: ns.F = function(value:U):U { return value; };
  g: ns.G = function(value:V):V { return value; };
}
const c = new C();
if (c.f.<number>(7) !== 7 || c.g.<string>("ok") !== "ok") throw new Error("value");
```

Before the repair, both routes fail while evaluating the first literal's `U` before main body entry. After preparation is repaired, `F` works but `G` fails assignability before main body entry: an ordinary builder return has already reduced its result to the shared Type Object with `U`'s canonical signature. Publishing that result under another alias cannot reconstruct `V`.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require explicit own binders for computed results | Consumers must know how an alias was produced; completion can suggest unusable names. | Rust has no matching type-valued JavaScript builder and does not justify a packaging exception. | Low implementation cost. | Weakens uniform contextual-signature behavior. | Reject as the semantic solution; explicit binders remain a workaround. |
| Use the canonical representative, or split identity by binder spelling | First use chooses accepted names, or structurally equal types become unequal. | Rust item identity does not make these structural signatures nominal. | Canonical-only is cheap but lossy; split identity increases interning and IC diversity. | Breaks contextual identity or alpha-equivalence. | Reject both variants. |
| Inspect builder syntax or search equal results for a likely declaration | Branches, helpers and repeated activations can select another declaration; wrong guesses cause distant errors. | Rust declaration resolution is not reverse matching from a runtime value. | Whole-body searches or broad caches with inadequate keys. | Cannot prove the selected context; may borrow stale captures. | Reject. |
| Retain selected declaration context alongside type-evaluation results, separately from public value identity | Selected results supply their binders independently of warm-up; completion and construction agree. | Rust reinforces declaration identity only; ECMAScript public value equality must remain unchanged. | Private compiler/type-evaluation metadata and result-flow tracking; no required public shape or namespace IC change. Optimizations must retain determining context. | Can preserve alpha-equivalence and selected captures without replay, provided success/failure and liveness rules remain intact. | **Recommend**. Implementation and full propagation rules remain open, regardless of effort. |

The next repair must define context transport through selected returns and helpers, not attach one mutable provenance field to an interned Type Object. Ordinary value copies must not manufacture alias provenance. Include reverse/warm order, multiple activations, failed evaluations, branch selection and alias/value-copy boundaries. Successful required outcomes remain on the queue; relaxed expectations or passing neighboring tests do not close them.

## Remaining: an outer generic parameter written in eval

```js
function outer<T:type>(outer:T) {
  type G = <V:type>(value:V)=>{outer:T,own:V};
  return function() {
    return eval(`const f:G = function(value:V):{outer:T,own:V} {
      return {outer,own:value};
    }; f.<string>("ok");`);
  };
}
outer.<number>(7)();
```

The written `T` in the newly parsed return annotation is not recovered by the checker and produces `RT_ASSIGNABILITY` in eval's static check. Omitting the return annotation, or naming an initialized outer alias `type Captured=T` in it, succeeds, including interleaved number/string activations. These are controls, not replacements for the required successful case.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat an unresolved `T` as `any` | Completion promises a contract that wrong values can evade. | Rust does not erase unresolved generic parameters to a dynamic top type. | Cheap but loses useful specialization facts. | Discards the actual binding's contract. | Reject. |
| Match any equal Type Object value or cached name | Other activations can silently change acceptance. | Rust lexical scopes do not authorize value/name matching; ECMAScript eval selects actual bindings. | Inadequate cache guards; potential cross-activation retention. | Manufactures provenance and bypasses shadowing. | Reject. |
| Require an outer alias workaround | The user must rewrite an already in-scope parameter; completion cannot explain the exception. | Rust has no eval but preserves generic scope; ECMAScript direct eval has lexical access. | Low cost, extra declarations. | Makes equivalent lexical access depend on spelling. | Reject as the semantic solution. |
| Retain explicit generic-parameter binding-instance evidence | Written annotations refer to the parameter the user sees. | Rust motivates declaration scope only; ECMAScript supplies the eval environment. | Private binding metadata, no public shape changes; cache per determining activation. | Preserves selected arguments, shadowing and reference-liveness obligations. | **Recommend**; not implemented here. |

## Remaining: stable imported constructor guards

```js
// local control: RT_IMPOSSIBLE_TEST, static, no body entry
class C {}
function f(c:C) { if (c instanceof C) {} }

// dep
export class C {}
// main: currently accepted
import {C} from "dep";
function f(c:C) { if (c instanceof C) {} }
```

This is an import/local discrepancy, not proof that the local answer is valid for every surrounding effect. Adding `Object.defineProperty(C, Symbol.hasInstance, {value:()=>false})` before the local function declaration still produces the local diagnostic, while the corresponding imported case is accepted. Preserve these controls when auditing how function checking retains protocol facts.

Apply the alternatives and five-criterion recommendation in [imported nominal guards after effects](tuple-construction-contexts.md#regression-fixture-imported-nominal-guards-after-effects): reject nominal-annotation-only reasoning, speculative hook execution and blanket deferral; retain actual constructor/protocol evidence with effect invalidation. The next repair must establish the stable import proof and audit the effect controls together, not force every imported `instanceof` into the local result. This repair and the broader identity, inference, phase/recovery, resource and conformance gates remain open.
