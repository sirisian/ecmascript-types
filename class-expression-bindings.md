# Immutable class-expression dependencies

Changing `class C {}` into `const C = class {};` must not erase an established nominal argument. Each successful class-expression evaluation creates its own class. An immutable binding and its stable aliases retain that particular evaluation when used in a type.

```js
class P<T: type> { constructor(value: string) {} }
function make() {
  const C = class {};
  return (value: any) => { const p: P.<C> = value; return p; };
}
const first = make(), second = make();
const holder = { p: first("x") };
function replace(raw: any) { raw.p = second("y"); }
replace(holder);
const p = holder.p; // TypeError: this view retains first's class argument
```

The backing write succeeds through its ordinary dynamic path. The receiving view has an independently established target. Calling `first` again supplies a compatible replacement. Named class expressions, parentheses and stable immutable aliases preserve the same distinction. Completion and reflection retain the declared members rather than silently weakening the target to `any`.

## Requirements and alternatives

A valid solution must preserve separate activations, same-activation reuse, lexical shadowing, generic captures, nested argument dependencies, self references, authoritative written targets, and failure ordering. Shared checking templates cannot contain one execution's constructor. Retaining a dependency must not rerun class initialization, read a candidate instance's public properties, or grant reference liveness. An uninitialized outer binding must not expose an expression that failed to initialize it.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Reject class-expression bindings in type positions | A routine syntax change rejects otherwise useful code; autocomplete loses a known class | Rust has nominal types but no directly corresponding runtime class-expression binding; ECMAScript permits this constructor binding | Simple rejection loses available specialization information | Avoids conflation by unnecessarily excluding established types | Rejected |
| Erase the expression argument to `any` | Wrong replacements pass despite the visible `P.<C>` contract | Rust preserves nominal arguments; erasure is not an analogue of its inference | Cheap, but discards guards and optimization information | Loses the declared dependency and receiving obligation | Rejected |
| Select the latest constructor stored for the source expression | Another factory call changes which class an earlier closure accepts | Rust nominality does not merge evaluated JavaScript classes; ECMAScript creates a fresh constructor per evaluation | Small map with an unsound cache/IC key | Conflates activations and recursive evaluations | Rejected |
| Infer the target from the tested instance or public `constructor` | A replacement can choose its own admissible target or run a getter | Rust generic arguments do not come from arbitrary instance properties | Observable reads and unstable property guards | Circular admission; no liveness evidence | Rejected |
| Publish the expression's outer binding dependency when class construction starts | A failed initializer can leave an apparently usable binding dependency | ECMAScript lexical initialization has a distinct successful commit; Rust offers no runtime class-initializer analogy | Early publication needs rollback and exposes partially established state | Confuses inner class self-context with the outer binding's initialization | Rejected |
| Retain the stable source template and close it from its successful immutable binding initialization or its own inner class context | Equivalent forms keep precise members, correct replacements work, and wrong activations fail | Rust supports nominal distinctions and scoped binders; ECMAScript evaluation and lexical initialization supply the closer activation model | Internal environment associations and canonical class identity; no added user-object properties, shape transitions or instance reads. Engines can capture the resolved identity in existing closure metadata and specialize guards on it | Preserves independent targets, initialization order, aliases, recursive contexts and all existing reference obligations | Recommended independently of implementation effort |

Choose the final direction. The [Rust Reference on types](https://doc.rust-lang.org/reference/types.html) identifies nominal types; [generic parameter scope](https://doc.rust-lang.org/reference/items/generics.html) excludes enclosing function parameters from nested items. These support only limited analogies: this proposal deliberately supports capturing class factories. ECMAScript [ClassDefinitionEvaluation](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-runtime-semantics-classdefinitionevaluation) supplies each class evaluation's fresh constructor and inner lexical context.

## Binding and completion rules

The checker follows the existing stable-origin rules for an ordinary immutable binding and its immutable aliases. A direct class-expression initializer supplies a symbolic class declaration, not a globally selected constructor. Parentheses preserve the origin. Existing restrictions on annotations, mutable or reference bindings, direct eval and potentially replacing decorators remain authoritative; this change does not prove arbitrary initializer expressions stable.

After successful initialization, associate the binding's own lexical environment with the class expression it actually created. Verify both the ordinary immutable binding and the created class's source correspondence. A failed evaluation or failed admission publishes no outer binding dependency. An alias preserves the original identity instead of allocating another one. Within class evaluation, the fresh inner context supports established self-reference rules without making its private expression name visible outside.

Complete only unresolved checking dependencies from that retained context. This reaches checked generic arguments, nested type descriptions, inferred results and callable result descriptions. Preserve already completed identities, argument structure, own generic binders and written targets. A closure from an earlier activation never switches to a later class from the same source. Specializations retain their original family rather than replacing the primary expression association.

The implementation no longer uses the latest-constructor-by-source fallback for expression-template admission. No available matching lexical context means that fallback cannot establish membership. This does not add a forward-initialization exception for a `const` binding or waive compile-time evaluability.

## Coverage and limits

The paired Script corpus covers anonymous, named and parenthesized expressions; type and value aliases; nested argument shapes; different and equal activations; recursion; loop captures; generic class expressions and captured parameters; self references; reflection; wrapper results; erasure; static-effect counts; private-name visibility; mutable-binding rejection; hostile public constructor access; and a failed initializer followed by a separate Script. Module cases cover exported and re-exported factories, asynchronous entry, and same-family reuse across exported converters.

This completes the queued immutable class-expression binding repair. It does not establish general overload correspondence, arbitrary composite result transport, mutable-binding inference, or the remaining convergence, phase, budget and final-verification gates. See [evaluated nongeneric class arguments](nongeneric-argument-identity.md) for the underlying identity and structural argument rules.
