# Rest and iterator effects

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

## Problem and chosen direction

```js
// First Script
const box = {
  [Symbol.iterator]() {
    return {
      next() { return { done: false, value: 0 }; },
      return() { globalThis.f = () => 1; return {}; }
    };
  }
};
// Second Script
function f(): string { return "old"; }
const [] = box;
const n: number = f();
```

An empty array pattern still acquires and closes an iterator. The preceding implementation could reject the Number receiver using the old string result because no indexed element projection occurred. Object rest, spread arguments and `for…of` had related gaps. Thirteen initial probes reproduced false Early Errors; several other probes already passed through conservative rules and remain useful controls.

**Recommendation:** represent implicit protocol operations in the effect judgment, while preserving the distinct identities of their inputs, produced values and fresh result containers. Use a positive intrinsic proof for the supported dense-array case and an opaque effect otherwise. Preserve actual evaluation positions. At runtime, retain the iterator method already obtained when deciding whether to fall back to a tuple conversion.

The latter is a concrete runtime repair discovered during the audit: the old tuple-conversion helper read `Symbol.iterator` to check for an iterator, then ordinary GetIterator read it again. This repeated getters and could choose a different method. The checker must not create that problem, and the evaluator must not already have it either.

## Requirements for a valid solution

1. Include iterator-method acquisition, `next` lookup/calls, result reads and applicable closing even when a pattern has no bound elements. Async iteration and delegation can additionally re-enter user code through suspension and their protocols.
2. Include object-rest enumeration and reads of selected own enumerable properties. Excluded keys, inherited properties and non-enumerable properties are different from selected properties.
3. Preserve fresh container identity. Copying a global alias into `rest.world` does not make `rest` itself the global object.
4. Preserve RHS-before-pattern, prefix-default-before-rest and callee-before-spread ordering. Required receiving boundaries, independent contracts and reference liveness still apply.
5. Checking executes no getters, iterator steps, conversion bodies or proxy traps. Runtime fallback preserves captured operands, abrupt completions and the selected method.
6. Keep summaries finite and private. Required error behavior must be specified independently of an implementation's analysis budget; an empty candidate set is not an absence-of-effects proof.

## Decision 1: how should rest results be represented?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Ignore rest bindings | Destructuring refactors lose evidence and can reject valid replacement code | Rust rest patterns do not supply JavaScript copy semantics; ECMAScript does | Cheapest scan | Leaves the recorded alias and getter failures | **Reject** |
| Identify the rest result with the original operand or every contained capability | Writes to `rest.f` appear to modify global `f`; autocomplete promises the wrong object relationship | Neither ECMAScript copies nor Rust places have this identity rule | Compact labels, broad invalidation and optimization loss | Confuses fresh container identity with contained values | **Reject** |
| Execute or exactly snapshot every runtime copy during checking | Appears precise until a proxy/getter runs early; later mutations invalidate snapshots | Rust static field patterns do not authorize executing arbitrary JavaScript getters | Unbounded or effectful analysis; inappropriate for required checking | Violates phase and single-evaluation rules | **Reject** |
| Fresh abstract aggregate, property-sensitive copying, explicit copy effects | Preserves common data-copy expectations and keeps uncertain reads conditional | ECMAScript own-enumerable copying is the direct model; Rust motivates identity distinctions only | Finite graph edges, shared slots for aliases, no extra fields on user objects | Follows capabilities without conflating their carriers; exclusions retain useful precision | **Recommend** |

Known excluded string keys are omitted from the rest summary. Unknown computed keys remain conservative. Earlier ordinary own descriptors can supply enumerable data without invoking user code; accessor or exotic copying supplies an effect at the copy operation. Source aggregates connect their tracked slots to fresh rest slots. Array rest renumbers supported known suffix positions; unknown yields contribute to a wildcard in a fresh array. Rest parameters similarly collect argument values into fresh arrays. Spread arguments contribute possible values beyond a single fixed argument position.

This is a may-flow model, not a claim to reconstruct exact runtime keys or arity. It retains the prior context-merging limits. Object spread's older wildcard summaries and wider descriptor mutation routes still need the broader audit.

## Decision 2: how much iterator behavior may be inferred from an array view?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Model every pattern as numeric property reads | Looks simple, but an empty pattern and a custom iterable behave incorrectly | Rust array/slice patterns are structurally different; Rust `for` invokes iterator traits. Neither establishes JavaScript indexed semantics | Cheap; misses real calls | Ignores acquisition, result getters and closing | **Reject** |
| Trust an array type or syntactic array literal unconditionally | Keeps precise errors until a prototype, indexed hole or iterator method changes | Rust arrays have stronger structural guarantees; ECMAScript iteration methods are replaceable | Enables unsafe specialization | Type/shape alone does not establish the protocol implementation | **Reject** |
| Treat every iteration as an arbitrary effect with no positive cases | Handles opaque iterables but unnecessarily loses ordinary dense-array evidence | Rust still distinguishes known built-in operations from arbitrary trait implementations | Cheap fallback; discards useful optimization and diagnostics | Conservative but fails the desired precision controls | **Reject** as the finished direction |
| Positive dense-array summary plus opaque protocol effects | Ordinary proven arrays retain useful diagnostics; unknown iterables keep runtime checks | ECMAScript algorithms establish the dependencies. Rust supports the limited distinction between structural patterns and method-driven loops | Private summaries and existing compiler dependencies; no new object shapes or IC states | Requires actual intrinsic identities, safe close behavior and source stability; includes empty patterns | **Recommend** |

The prototype's positive case checks the array iterator and iterator-next intrinsic identities, absence of a `return` hook along the ordinary iterator prototype path, and the existing source-effect screen. Sparse literals do not qualify: an indexed hole can reach an inherited getter. Async iteration and delegated yield do not use this array shortcut. The existing positive proof for a current-source second resume of fresh local generators is retained separately; it establishes the applicable resumed-input boundary, not general purity of generator execution. A blanket opaque-delegation experiment discarded that boundary and failed an existing regression, so that direction was rejected. Captured intrinsic identities do not justify assuming future stability after an affecting source operation.

The opaque summary covers possible effects of the protocol, rather than executing a speculative sequence. For `for…of`, the binding receives possible yielded values; the same summary accounts for acquisition and closing around the loop. For async iteration and delegation it includes possible re-entry conservatively, without claiming an exact schedule or a precise summary of each awaited value.

Autocomplete can retain an independently declared element or callable contract, but a replaceable implementation cannot promise the old global result merely because the loop or pattern looks harmless. A wrong runtime value still meets its explicit receiving boundary. No new public `opaque` type or borrowing rule is proposed.

## Decision 3: how should operation positions and tuple fallback work?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use the text position of the pattern as the position of all effects | A later rest getter appears to run before the initializer or an earlier default | ECMAScript evaluation order is authoritative; Rust's place evaluation also is not mere textual-offset ordering | Cheap scan | Loses already captured evidence and misorders effects | **Reject** |
| Re-read the iterator method after checking whether conversion is needed | Getter counts and even selected implementations change unexpectedly | Rust `IntoIterator` desugaring is a useful single-selection analogy, not a tuple-conversion rule | Extra lookup and potentially arbitrary repeated work | Reproduced double-get bug; abrupt completion and identity can change | **Reject** |
| Try conversion whenever iterator acquisition fails | Conceals a throwing getter or invalid method, then runs another user operation | Neither ECMAScript GetMethod nor Rust iterator conversion licenses this recovery | Extra user work and complex recovery | Swallows or changes an existing failure boundary | **Reject** |
| Attach effects to semantic positions; retain the first method and convert only when absent | Preserves ordinary capture, getter counts and predictable errors | ECMAScript supplies GetMethod/call/close behavior; Rust is only an analogy for explicit protocol selection | Removes a redundant lookup; no new user allocation, wrapper or replay | Preserves tuple-conversion priority, abrupt completion, composite iteration and receiving/liveness rules | **Recommend** |

The RHS initializer executes before any destructuring operation. Object rest executes after preceding property defaults. Call spread executes after the callee has been obtained. Assignment-pattern syntax is not itself an expression spread: analyzing a rest target as an iterable expression would invent an operation that never happens.

The runtime now shares one acquisition helper between binding and assignment patterns. A present iterator method is called using the captured value. A throwing lookup, non-callable result or failed method call propagates its failure. Only an absent method permits the declared tuple conversion; the converted result then undergoes normal iterator acquisition. Composite tuples retain kind-based iteration. Primitive handling remains with the ordinary iterator operation.

This repair makes no new reference live, reacquires no reference after failure and does not suppress IteratorClose. Counts and order are tested alongside completion and error phase, including a primitive exception thrown by closing.

## Scope and remaining work

This increment covers the recorded rest/iterator cases and the duplicate lookup repair. It does not complete milestone 5. The [ninth increment](descriptor-access-effects.md) adds descriptor conversion and installed-accessor summaries. Remaining work includes wider descriptor/prototype mutation paths, builtin callback/reflection combinations, and other consumers of callable or ancestry evidence. Within-pattern joins, unknown keys and call contexts can still be conservative. The normative precision rules must eventually be complete; private graph limits cannot decide observable language behavior arbitrarily.

The separate deep `operandParticipates` overflow and the recorded test/build failures are unchanged. The integrated milestone 6 gate remains open. No production-engine benchmark or complete extension/test262 result is claimed.

## Primary references

[ECMAScript iterator operations](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-getiterator) and [CopyDataProperties](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-copydataproperties) supply the direct protocol and copying constraints. [Rust iterator loops](https://doc.rust-lang.org/reference/expressions/loop-expr.html#iterator-loops) and [rest patterns](https://doc.rust-lang.org/reference/patterns.html#rest-patterns) are useful comparisons, but Rust structural matching and borrowing guarantees do not transfer to JavaScript getters, proxies or mutable prototypes. The shape/IC discussion above is an engineering inference about keeping analysis private, not a benchmark of this interpreter.
