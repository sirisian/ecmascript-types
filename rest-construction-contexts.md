# Construction contexts for rest parameters

A rest annotation describes the collected value. A fresh argument therefore needs the type of its destination **element**, while admission still checks the collected value at the ordinary rest-binding boundary. The same rule applies when a constructor or callable view supplies the contract.

```js
type Item = { fn: (value: string) => string }
          | { fn: <U: type>(value: U) => U };
function consume(...items: [].<Item>) {
  const fn: any = items[0].fn;
  fn.<number>(7);
  fn.<string>("ok");
}
consume({ fn(value: U): U { return value; } });
```

The destination selects the generic alternative before the method is created. Carry that selected construction result with this argument occurrence into the fresh rest array, then use it during the array's ordinary admission. Selecting again by union order at that boundary can erase the generic contract and make the first specialization fail.

## Determining the destination

Use the selected declaration and source argument structure, without evaluating expressions, invoking iterators, reading properties, or inspecting argument values. Pin explicit names using ordinary named binding, including a named rest's segment and any suffix returned to required following parameters. Previously overwritten fixed named expressions retain their own fixed construction context even though their values will not be supplied. Complete a positional prefix and sole trailing rest where their correspondence is determined independently of values. Value-dependent partitions, including optional-prefix choices during named distribution, provide no speculative destination.

A homogeneous array rest supplies its element type. A tuple rest supplies the effective position type for the **count assigned to that rest**, after generic substitution. Written tuple rest entries are not individual positions: nested tuples, compact finite runs, required suffixes, and trailing defaults follow the tuple's positional domain. Count omitted defaults as omitted arguments; do not evaluate them to discover a construction context. A reference destination supplies no fresh value-construction context.

Static checking and runtime creation use the same effective positional domain. This does not replace the normal arity, reference, numeric-admission, or distribution checks. Context is an input to constructing a fresh expression, not proof that its value is admitted.

## Keeping the occurrence through collection

Rest collection consumes the invocation's argument receipts in ordinary binding order. An unchanged value contributes its pending selection through the rest array's newly created data descriptor. Reference decay reads a location and contributes no fresh construction selection. Preserve intervening aggregate descriptor identity; replacing a contribution with the identical value still invalidates its old association.

At the ordinary collected-type boundary, pass eligible selections into recursive checks and conversions. A selected-member failure does not fall back to another union arm. Retain the original declared union for storage. Later argument effects precede this admission; abrupt argument completion prevents it. Collection and parameter-default order remain unchanged, including the existing position of a non-final rest's admission. A failure preserves ordinary side effects and publishes no successful rest binding.

Ordinary reads of escaped values, reentrant calls from getters, reference viability, and unrelated boundaries do not consult this invocation's pending selection. End collection bookkeeping on every normal or abrupt exit. Do not attach a permanent selection tag or wrapper to the collected array or its children.

## Directions considered

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust and analogy limits | Real-engine performance, including shapes/ICs | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Withhold all context whenever a rest is present | Completion loses a known element contract; a valid written generic binder fails or a later specialization throws. | Rust call arguments are coercion sites; TypeScript relates tuple rests to parameter lists. Neither suggests discarding a determined destination. | Avoids metadata but loses direct-call specialization opportunities. | Static adoption and runtime creation disagree. | Reject as the final design. Unknown correspondence alone still defers. |
| Give every argument the collected array type, first parameter type, or union of all tuple positions | Completion suggests members of the wrong position; a wrong guess can select an incompatible callable. | Rust formal parameters and TypeScript tuple elements are positional, not interchangeable. Neither has this proposal's runtime structural-union admission. | Cheap projection, but incorrect conversions and repeated failures are not useful savings. | Loses finite-run/suffix correspondence and declaration identity. | Reject. |
| Evaluate or partition argument values before contextual construction | A binder must already exist when its expression executes; replay can duplicate arbitrary effects. | ECMAScript argument evaluation and rest collection are ordered; Rust's explicit coercion sites do not justify replaying JavaScript expressions. | Extra iteration, property reads and allocation; unstable observations inhibit optimization. | Changes effects, failure precedence and potentially reference liveness. | Reject even if it makes the interpreter simpler. |
| Supply correct context but discard the selected result while collecting | Autocomplete looks correct but the first generic use can fail after union-order admission. | TypeScript's erased types do not model this runtime boundary; Rust nominal variants retain identity, unlike these structural alternatives. | Saves transient receipts but cannot implement the selected contract. | Re-selects an overlapping arm and loses the construction obligation. | Reject. |
| Store choices on values or admit each argument immediately | Escaped/reentrant reads can inherit stale choices; early admission changes which later effects run. | Rust's nominal representation is not a model for adding structural-union tags; ECMAScript evaluates arguments before binding. | Permanent fields affect object shapes and ICs; global tables add lifetime cost; early checks can duplicate work. | Conflates occurrences, ignores same-value replacement, or moves liveness-sensitive checks. | Reject. |
| Use declaration-proved element contexts and invocation-owned receipts until ordinary collection admission | Completion reflects the actual element contract; errors stay at the established boundary. | Rust supports destination-based coercion, TypeScript supports tuple/rest correspondence, and ECMAScript supplies collection order. Receipt transport is this proposal's own runtime rule. | Compile known correspondence and compact tuple runs; carry optional call metadata without public fields, wrappers, shape changes or new IC keys. Elide receipts where no selected child exists. | Preserves captured declarations, source indices, descriptor liveness, ordinary reads, numeric modes and reference decay. | **Recommend independently of implementation effort.** |

Primary references: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), [TypeScript variadic tuple types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html#variadic-tuple-types), and [ECMAScript parameter iterator binding](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-function-definitions-runtime-semantics-iteratorbindinginitialization). These analogies support destination and ordering choices; they do not establish runtime structural adoption or its implementation. The performance assessment is a real-engine design inference, not a benchmark of this interpreter.

## Coverage and remaining questions

The implementation covers direct positional and explicitly named arguments with declaration-proved correspondence, including homogeneous, fixed, defaulted, finite, nested and suffix tuple rests, captured aliases, imports, callable views, explicit specialization and constructors. The six original final-rest failures are required-success regressions.

Fresh iterable and object call-spread operands remain separate required-success work. Their context and contribution transport must follow the actual protocol, source occurrence and live descriptor; guessing from expression syntax or value equality is insufficient. Assignment writes, unresolved alternative applicability, heterogeneous multiple-rest partitions and independently stable imported guards remain outside this increment. Ordinary spread reads do not gain pending construction choices here.
