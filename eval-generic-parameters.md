# Generic parameter evidence in direct eval

A written annotation in direct eval can use the bound generic parameter from its lexical environment. Checking must retain the argument of that particular activation, including after a closure escapes and another activation binds the same declaration differently.

```js
function outer<T:type>(outer:T) {
  type G = <V:type>(value:V)=>{outer:T,own:V};
  return function() {
    return eval(`const f:G = function(value:V):{outer:T,own:V} {
      return {outer,own:value};
    }; f.<string>("ok");`);
  };
}
const n = outer.<number>(7);
const s = outer.<string>("outer");
n(); s(); n();
```

The preserved case previously failed `RT_ASSIGNABILITY` during eval's static check: `G` retained its captured argument but the separately parsed written `T` did not. This increment makes that case succeed, without requiring the previously successful outer-alias or omitted-return-annotation workarounds. Wrong bodies remain errors before eval's body runs.

## Decision

A valid direction must retain declaration and activation identity, already selected arguments, local/adopted-parameter shadowing, lexical availability, checking phases, resource accounting and reference liveness. A caller's unrelated generic frame, public type equality or a previous check of the same source cannot supply a missing binding. Checking must not execute environment methods, getters, initializers or bodies to recover evidence.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat an unresolved outer parameter as `any` | Autocomplete promises a parameter contract that an invalid body can evade. | Rust does not resolve an unavailable generic name by erasing its contract. | Cheap but destroys usable specialization facts. | Loses the selected argument and allows incorrect bodies. | Reject. |
| Recover from an equal Type Object or a name/source cache | An unrelated declaration or warm-up changes which annotation works. | Rust's generic scope is declaration based, not a reverse search over equal values. | Low-cost lookup with insufficient keys; broad caches can retain unrelated activations. | Confuses declarations, shadowing and captures. | Reject all variants. |
| Use the caller's active generic frame, or declaration identity without its binding instance | Calling an escaped closure from another specialization can change its meaning; recursion can reuse the same declaration with another argument. | Rust lexical scope does not mean dynamic caller scope. | Avoids retained environment evidence but needs unsound dynamic guards. | A declaration alone does not identify one activation's selected argument. | Reject both variants. |
| Require an outer alias or omit the written annotation | The user must rewrite an already in-scope parameter; a useful explicit return annotation causes an error. | Rust has no runtime eval counterpart that justifies this spelling exception. | Low implementation cost, redundant source. | Makes equivalent lexical references depend on how they are written. | Reject as the semantic rule; the controls remain useful. |
| Read values, invoke environment methods, or replay a producer during checking | Merely adding an annotation can execute user code or move a failure. | Neither Rust scope nor ECMAScript eval authorizes these extra evaluations. | Adds calls, effects and invalidation to a lookup; inhibits reuse. | Can move TDZ/effect failures and cannot establish reference authority. | Reject. |
| Retain the selected arguments with their actual lexical parameter environment and consult that evidence for newly parsed annotations | The written parameter means what the enclosing activation bound; completion and explicit annotations agree. | Rust supports declaration-based generic scope as a limited analogy. ECMAScript supplies the actual lexical/global distinction for eval and dynamic functions. | Private environment/specialization metadata; retain only the environment's own parameters. No user-object shape change or new namespace IC is required. Cache reuse must preserve the determining environment and arguments. | Preserves binding-instance identity, nearer declarations, failure phases and liveness without a value read. | **Recommend**, independently of effort. Implemented for the annotation coverage below. |

Rust's nested `fn` items do not inherit an enclosing function's generic parameters; this proposal's nested JavaScript functions and runtime eval have different rules. The useful Rust analogy is declaration identity and explicit generic scope, not identical nesting semantics. The performance assessments describe feasible engine implementations, not interpreter benchmarks.

## Binding and checking boundaries

The retained argument belongs to the parameter's environment, not to an interned Type Record, a shared syntax node, an ordinary value binding, or a function object reused by multiple calls. A function activation retains its established explicit/adopted arguments at entry and refreshes the evidence when inference has selected its arguments. A generic class specialization retains its own parameter evidence on its class environment. The argument-binding procedure exposes only the arguments already bound in its own frame as it proceeds left to right; an unavailable parameter does not borrow one from an older invocation.

Lookup follows the applicable lexical environment without invoking environment methods. A nearer ordinary binding blocks fallback even when uninitialized or unable to supply evidence. An opaque object environment supplies no proof through this lookup. An eval-local declaration or an effective own parameter of the function literal takes precedence over outer evidence. Syntax belonging to another source's declaration is checked in that declaration's context, not against the current eval environment.

The engine retains only each published environment's own declaration-bound arguments, using weak environment keys. It uses the resulting record when resolving an otherwise external unqualified type reference in the newly checked source. Existing alias preparation, type computation, ordinary reads, argument admission and reference checks still run at their established boundaries. An ordinary stored Type Object is not generic-parameter evidence; other existing rules for such values still apply.

## Coverage

The portable `eval-generic-parameters.json` corpus covers explicit, inferred and defaulted arguments; immediate eval and escaped functions, arrows and methods; binding, arrow, field, object-method, return, argument and assignment consumers; explicit own binders; generic class methods; named and namespace module routes; nested eval; an escaped function created by eval; interleaved activations and recursion through the same declaration; unrelated caller parameters; local aliases, own/adopted parameters and built-in-name shadowing; parameter defaults; constraints; invalid bodies and unused bodies; an unavailable local shadow; and recovery after a failed eval whose body marker stays unset.

An additional engine test checks ordinary lexical eval with runtime types disabled. The baseline/current probes also record the stronger static evidence for `eval("const x:T=false;")` under `T = number`: its `RT_ASSIGNABILITY` rejection moves from pre-evaluation to the static check, still before the eval body runs.

An assignment fixture initializes its typed function binding before assigning again: a bare `let f:G;` correctly requires a default value that this function type does not supply. A generic parameter named `string` also shadows that built-in in explicit call arguments; the successful control uses `boolean` for the function's own argument instead of accidentally requesting the enclosing parameter's numeric type.

This closes the preserved written-outer-parameter annotation case. It does not certify every qualified, higher-kinded, value-pack or generic suspension family. The runtime lookup issues below and the imported-constructor guard/effect audit remain open, along with the broader identity, inference, phase/recovery, resource and conformance gates.

## Newly preserved runtime lookup discrepancies

The baseline and this increment both accept these programs, even though the called source has a global environment and should not acquire the caller's generic parameter:

```js
function f<T:type>() { return (0,eval)("const x:T=7;x;"); }
f.<number>();

function g<T:type>() { return Function("const x:T=7;return x;")(); }
g.<number>();
```

The baseline and this increment both reject the following with a runtime `TypeError`, although the nearer ordinary binding should select `string`:

```js
function f<T:type>() {
  { const T = type string; return eval('const x:T="ok";x;'); }
}
f.<number>();
```

The new checker lookup supplies no outer-parameter evidence in these cases. The remaining discrepancy is the runtime's name-based generic-frame fallback, which is also used by ordinary value reads and type computations. These are required-outcome reproducers, not passing conformance expectations or a new policy allowing caller-scope access.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Bless dynamic caller lookup | Results depend on who calls the code; autocomplete cannot describe one lexical meaning. | Neither Rust generic scope nor ECMAScript's global environment for indirect eval/`Function` supports this. | Simple stack search, unreliable specialization reuse. | Violates lexical scope and nearer-binding precedence. | Reject. |
| Special-case only indirect eval and `Function` by clearing frames | Repairs two examples but leaves ordinary shadows and other call forms inconsistent. | ECMAScript defines environments, not a list of spellings to erase. Rust supplies no justification for a partial scope rule. | Small local change; restoration and suspension add state management. | Does not establish which lexical parameter a runtime read denotes; can disrupt needed captured bindings. | Reject as the complete solution. |
| Guess from Type Object equality or execute an extra read to decide | Equal values or getters can change which declaration wins. | Rust resolves declarations; ECMAScript performs each required read in its specified order. | Extra lookup/effect guards or an inadequate equality cache. | Cannot establish the actual binding and can duplicate effects. | Reject. |
| Carry actual lexical parameter-resolution evidence into runtime reads and type denotation | Direct eval, ordinary shadows and globally scoped code obey one scope rule. | Rust motivates declaration ownership only; ECMAScript provides the source environments and ordinary read order. | Private reference/environment evidence with lexical guards; no public object-layout change is required. | Can preserve activation identity, availability, ordinary reads and liveness together. Audit specialization captures, metadata parameters, defaults and suspension as well as the three examples. | **Recommend** for the next runtime-scope repair, independently of effort; not implemented by this annotation increment. |

The prior [constructor guard/effect reproducers](prepared-alias-contexts.md#remaining-stable-imported-constructor-guards) remain unchanged. Do not equate every imported `instanceof` with the current local answer before checking hook effects and invalidation.

Sources: [Rust generic scopes](https://doc.rust-lang.org/reference/items/generics.html), [ECMAScript PerformEval](https://tc39.es/ecma262/multipage/global-object.html#sec-performeval), and [CreateDynamicFunction](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-createdynamicfunction).
