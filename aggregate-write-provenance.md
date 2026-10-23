# Aggregate contributions after writes

An unchanged binding can name a changing object. Return inference must distinguish its private initializer description from a contract admitted at a typed boundary.

```js
function make(seed: number) {
  const box = { x: "s" };
  box.x = 1;
  return box.x;
}
const n: number = make(0);
```

The returned contribution is Number. The original String initializer cannot reject this program or describe the later return. `box` acquires no storage annotation. An independently declared or inferred binding contract remains authoritative, including when an erased alias replaces the backing property.

Track established aliases, including nested aggregate extraction, by value identity. Transfer a simple assignment's known value and evidence to an established own data property, using the receiver selected before the right-hand side. Keep unaffected properties and distinct objects. A scalar copied before a write keeps its earlier contribution; an aggregate copy remains an alias. Overwriting a member removes the overwritten value's evidence. A typed stored value can anchor its subsequent contribution, while a plain literal cannot.

An uncertain destination or possible user-code effect withdraws private descriptions. Calls, getters, unknown keys, deletion, compound updates and suspension cannot preserve a snapshot merely because the binding is `const`. Apply the existing branch joins, loop widening and exceptional-entry widening. A missing fact on a possible predecessor means unknown. Mutable array writes conservatively withdraw initializer descriptions; an array's storage contract and iteration contract remain separate.

Patterns model key and selected-default effects in extraction order. In `const {a = (box.x = 1), x} = box`, a missing own `a` may reach an inherited getter. Its possible effects invalidate the private description before the next extraction. With an established own `a: undefined`, the selected default can transfer its known write. A supplied non-undefined `a` skips the default. A scalar extracted before either effect remains unchanged. None of this executes source code during checking.

## Requirements and decision

A valid solution preserves evaluation order, declared contracts, alias identity, scalar snapshots, contribution provenance and reference liveness. It must support both useful pre-effect information and sound post-effect results, without inventing storage contracts or inspecting runtime values during analysis.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep initializer members after every write | Editors and diagnostics describe old values; a valid return can be rejected before execution. | Rust assignments operate on typed places. Its fixed field types do not justify freezing untyped JavaScript contents. | Cheap stale metadata; no shape or inline-cache benefit. | Confuses binding identity with mutable contents. | Reject. |
| Give every literal aggregate a permanent structural contract | A plain local object silently gains restrictions and new runtime boundaries. | Rust is statically typed throughout; its ordinary inference is not this proposal's gradual opt-in policy. | Adds admission work and contracts to ordinary code without a required layout benefit. | Changes the existing annotation-seeded design. | Reject. |
| Erase every aggregate initializer description | Safe pre-write information, selected members and autocomplete disappear unnecessarily. | Rust preserves established component types; its borrow rules do not require this erasure. | Less analysis at the cost of precision; no user-object layout benefit. | Avoids stale answers but loses existing transparent-contribution behaviour. | Reject as the general rule. |
| Join every syntactic assigned value with the initializer | Even a definite replacement leaves an impossible old alternative; unrelated aliases and unreachable writes distort results. | Rust checks each assignment against its place type, rather than unioning every value ever written. | Requires a write inventory but still misses dynamic effects. | Ignores source positions, aliases and effectful reads. | Reject. |
| Execute writes, getters or defaults during checking | Autocomplete and rejected code can have effects. | Neither Rust's assignment analysis nor JavaScript's runtime evaluation order authorizes this. | Unbounded effects obstruct predictable compilation and caching; ICs cannot fix it. | Violates checking phases and can inspect invalid references. | Reject. |
| Track proved transfers and invalidate private descriptions across uncertain effects | Known writes update editor information; uncertain effects retain the gradual receiving boundary. Earlier scalar copies and independent contracts remain useful. | Rust distinguishes places from values and preserves typed assignments. Its ownership, borrowing and pervasive static types limit the analogy to mutable JavaScript objects. | Compiler-owned identity and flow metadata can share the existing dependency graph. No wrappers, public fields, user-object shape changes or new inline-cache keys are required. | Preserves order, aliases, declaration evidence, receiving checks and independent reference permissions/liveness. | Choose independently of implementation effort. |

This performance assessment is design reasoning, not a benchmark. [Rust assignment expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html#assignment-expressions) and [borrow operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators) explain the limited place/value analogy. [ECMAScript assignment operators](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-assignment-operators) determine receiver and right-hand-side evaluation order; the proposal separately defines contribution inference and its gradual contracts.

## Audit boundary

The engine change tracks private aggregate origins in inference flow, transfers established own data writes, preserves nested aliases, and snapshots returned descriptions. It conservatively invalidates unsupported transfers. It does not add a general heap analysis, infer nonescaping call effects, or infer a new array storage contract. The operand-sequencing follow-up is described in [Contributions follow expression evaluation order](expression-effect-sequencing.md). General precision, remaining expression forms, convergence and specification determinacy remain separate audits.
