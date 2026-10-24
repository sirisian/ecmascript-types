# Named object-spread construction

A known named destination supplies the context for a fresh spread object's data initializer. The actual spread protocol decides whether that initializer contributes an argument. For example:

```js
type Item = { fn: (value: string) => string }
          | { fn: <U: type>(value: U) => U };
function run(a: Item) { const fn: any = a.fn; fn.<number>(7); }
run(...{ a: { fn(value: U): U { return value; } } });
```

The inner method adopts its own generic declaration. Named extraction preserves the selected complete alternative until ordinary parameter admission; it must not silently specialize the method to the overlapping nongeneric arm. The receiving storage contract remains `Item`.

## Context before protocol selection

An established fixed by-value name supplies its parameter type. A homogeneous Array rest supplies its element type. Parentheses preserve a fresh object source. Resolve the callable view, captured arguments and explicit specialization first. Do not guess an unresolved overload, reference destination or count-dependent tuple-rest position. Stored values receive no retrospective construction context.

This context determines how the fresh expression is created, not whether it will be consumed by name. A decorator, initializer effect or prototype change can install an iterator. Ordinary `GetMethod` still runs once on the evaluated value; a callable iterator wins, a nullish hook selects names, and a non-callable or throwing hook fails. An iterator receives the already constructed values and ordinary admission rules; it does not receive property construction selections. Autocomplete should show the declared named field context without promising the object's runtime spread protocol.

Repeated fixed names retain last-value-wins binding. Earlier expressions still evaluate, and their function bodies are independently checked under their construction contexts, even though those values are not admitted as the surviving argument. The same applies to duplicate data keys in the fresh source. Dropping an overwritten argument from binding must not turn its adopted `U` into an unbound name.

| Available direction | Ergonomics, completion and wrong guesses | Prior art and limits | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Withhold every spread field context until its protocol is known | A plainly named fresh initializer loses declared generic scope; completion cannot explain `U` | Rust gives declarations and struct fields construction contexts, but has no JS named spread | Avoids metadata, but prevents useful specialization | Discards established contextual information; learning the protocol after construction is too late | Reject as the general rule |
| Infer named protocol from object-literal syntax | Completion looks certain but a custom iterator makes the guess wrong | Neither ECMAScript's value-based iteration nor Rust's explicit calls imply this | Cheap syntax test; incorrect guards under mutable prototypes | Reorders or bypasses required protocol effects; changes existing named-spread routing | Reject |
| Probe or replay protocol effects before construction | Apparent precision costs duplicate getter/iterator effects or changed exceptions | Rust traits are static; their resolution cannot justify running JS effects | Speculation needs deoptimization and cannot undo arbitrary user code | Changes evaluation order and failure precedence; no reference authority follows | Reject |
| Supply conditional named construction context, then use actual protocol | Declared names support completion; custom iterators consume the resulting values and can still fail ordinary checks | Rust field/coercion sites support contextual creation; ECMAScript supports once-only actual operand/iterator evaluation. Neither specifies these receipts | Compile-time projection plus operation-local metadata; no extra value fields, wrapper, shape transition or IC key | Keeps effective declaration identity, source effects and live references; context is not admission or protocol proof | Recommend independently of implementation effort |

## Capturing a live contribution

Named spreading captures its own key list once, checks each current own String descriptor, and reads an enumerable value once in key order. Symbols are ignored. At that actual read, a pending child selection is eligible only if the source result still carries the original data descriptor/value and every inner contribution is live. Getter results, stored sources and equal values from independent operations gain no selection.

Once the argument is captured, its outer source property has been consumed. Later replacement of that source property does not change the captured value. Retire exactly that outer edge; keep every nested descriptor edge inside the captured value. Thus replacing `source.a` after reading it preserves a selection for the captured argument, but replacing `captured.child` can invalidate a selection belonging to that child. A same-value descriptor replacement before capture invalidates the source association. Reentrant checks remain independent.

| Available direction | Ergonomics, completion and wrong guesses | Prior art and limits | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain the consumed source edge until parameter admission | Later unrelated source-slot replacement unexpectedly changes an already captured argument's contract | Ordinary JS argument capture retains its value; Rust moves/copies are an analogy, not a JS descriptor rule | Unnecessary long-lived guards and retention | Confuses the source slot with the captured occurrence; no longer an intervening edge | Reject |
| Drop every contribution edge at capture | Nested replacement appears harmless but can preserve an obsolete selection | Rust nominal enum variants are value identity; this structural union has no permanent variant tag | Fewer guards but unsound reuse | Loses nested contribution liveness and same-value replacement semantics | Reject |
| Recover selections globally by value identity | An ordinary getter, alias or custom iterator can unexpectedly regain a special context | Rust enums are explicitly tagged; that analogy does not justify tags here | Global lifetime tables or fields can harm shapes and ICs | Leaks across occurrences and reentrant operations; risks stale declaration contexts | Reject |
| Admit eagerly while extracting each named property | Later arguments see new getter effects or never execute | ECMAScript retains argument evaluation before invocation; Rust does not specify JS accessor timing | Can duplicate checks/materialization | Changes failure precedence and reference-liveness check timing | Reject |
| Validate the live path at capture, retire only the consumed outer edge | Captured arguments retain their context; actual inner mutations still matter | ECMAScript argument capture supports the snapshot principle; Rust ownership helps distinguish values from places but lacks mutable JS descriptor paths | Index selections once by property; inspect only relevant paths. Direct known constructions can eliminate metadata without object-shape changes | Preserves occurrence identity, nested liveness, ordinary admission order and independent reentry | Recommend independently of implementation effort |

The interpreter indexes the producing completion's direct property contributions once and passes eligible selections with argument occurrences. It never calls an accessor to validate a receipt. A real engine can use compiler facts and write guards instead of this interpreter's descriptor objects. Pending selections require neither wrappers nor public properties. Selected-member failure has no fallback; ordinary receiving checks retain their existing timing and failed-check effects.

## Coverage and remaining scope

Tests cover function expressions, arrows and methods in both union orders; names, duplicate names/keys, nested objects/tuples, homogeneous named rests, callable views, specialization, constructors, captures and imports. Controls cover identity, source effects, getters, replacement before/after capture, nested replacement, actual custom/nullish/invalid iterator hooks, key filtering, stored sources, discarded-source checks, reentry and failure recovery. Module, Script, eval, Function and feature-off surfaces are included.

This closes the preserved six named object-spread cases. Iterable call-spread construction and assignment-write transport remain separate required-success follow-ups. Full competing-body/default applicability, unresolved alternatives, conflicting numeric preferences and rollback/cache reuse remain broader gates. This does not make an arbitrary property read or iterator yield a fresh construction result.

Primary prior art: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), [Rust struct expressions](https://doc.rust-lang.org/reference/expressions/struct-expr.html), and [ECMAScript argument-list evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-argument-lists-runtime-semantics-argumentlistevaluation). Rust has no equivalent named object-spread protocol; its analogy is limited to contextual creation and the distinction between values and places.
