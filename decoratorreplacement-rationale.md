# Decorator Replacement — rationale

> How [decoratorreplacement.md](decoratorreplacement.md) reached its decisions,
> including four reversals. **Not normative.** It is kept because the reasoning
> that produced a decision is what tells a later reader whether a new
> consideration disturbs it, and because four of these decisions were reached by
> being wrong first.
>
> Section numbers refer to decoratorreplacement.md.

Every quotation was checked verbatim against decorators.md and README.md, and
every claim marked **(measured)** against the reference implementation. Three
things were wrong in the previous draft and are corrected above:

1. **The two quoted passages describe two DIFFERENT gaps** — a placeholder field
   on the block family, and a constants-only `initial` on fields and parameters.
   The draft treated them as one. Correcting it made Part A larger, since the
   second gap covers every field and parameter with a non-constant initializer.
2. **The block family is twelve reflection types, not the three shown**, and it
   includes `MatchArmBlock` — whose `pattern` field is the only way a decorator
   could ever see a pattern, since a pattern has no runtime form.
3. **`Reflect.MatchArmBlock` does not exist** in the implementation, though the
   other eleven do. The draft said the whole family was present.

The verification also found the §7.7 overlap, which was not visible from the
replacement table alone: it took enumerating which reflections carry
`Expression` to see that two of them are also value-replaceable.

## 1. The token-stream reversal

A later revision replaced Part B's string protocol with token streams, reversing
§7.1. The earlier recommendation was not wrong on its own terms — it was
answering a narrower question than the one that mattered:

- **The cost was over-priced.** "Avoid committing to a standardized structure"
  treated tokens as an AST. ECMAScript's lexical grammar is already normative and
  moves far more slowly than its syntactic grammar (§1).
- **The benefit was under-counted.** It weighed tokens on HYGIENE alone, where
  the Rust ecosystem's tolerance of `call_site` is real evidence, and never
  weighed them on SPAN COMPOSITION (§5.3) — where Rust has no complaint precisely
  because spans make the problem not exist. **Absence of a complaint about a
  problem someone solved is not evidence the problem is small.**

Three sections shrank or vanished as a result: the `MacroResult.map` protocol,
the composition chain, and the re-scan strategy. **A change that deletes more
than it adds is usually the right one**, and that this one did was the signal the
earlier reasoning had missed something.

Part A was affected after all, though not in the way that revision assumed: see
§9.3.

## 2. Consistency pass after the reversal

A full pass found seven places where the string-era text survived the change,
and two errors of a different kind:

- **Six sections still described strings**: §2's statement of what Part B
  returns, §4.4's phases (which still re-lexed a splice §7.3 says needs no
  re-lexing), §5.2's outside-in argument, §7.6's "it sees text", and §7.7's
  "text replacement" throughout. All reconciled.
- **The summary table CONTRADICTED ITSELF.** It carried both "Engine composes
  the map chain" and "Spans compose themselves; no map protocol" — the first a
  survivor of the protocol §5.3 deleted. Rebuilt and grouped by part.
- **Two quotations had been silently lowercased** to fit a sentence. Both are
  now exact and attributed. A quotation altered for flow is a small thing in
  prose and not a small thing in a document meant to be cited against its
  sources.

**The failure mode worth noting is the first one.** Each stale passage was
locally coherent — nothing read as wrong in isolation — and only contradicted a
section elsewhere. A change that deletes a mechanism leaves references to it
scattered where nothing points at them, so the check has to be a sweep for the
deleted concept rather than a reading of the changed sections.

## 3. Removing `source: string`

A draft kept a `source: string` field on the reflections alongside the tokens.
It should not have — the instruction had been to drop the string concept
entirely — and the two arguments for keeping it were both wrong:

- **Fidelity.** "Tokens lose comments and whitespace" is true of Rust's stream
  BY CONVENTION and not of token streams as such. With spans over a buffer, the
  gap between consecutive tokens IS the trivia. **An implementation's convention
  was imported as though it were a property of the design.**
- **Staging.** It let Part A land without the token vocabulary — at the price of
  a second way to say one thing. **This project's most frequent defect is two
  facilities describing one fact and disagreeing**, and adding a redundant field
  to buy a scheduling convenience is that defect on purpose.

What the field was actually wanted for — logging, an error message — is a
`toString()` on the stream, which is where Rust puts it too.

**The split survives, in a better shape**: Part A is READING and Part B is
WRITING, which divides the hard questions cleanly. Hygiene, CSP, expansion order
and the preprocessor phase all belong to writing; none of them arises for a
reflection that only hands out what is already there.

## 4. External review

A review of §7 by a reader without the rest of the proposal in view found three
things worth adopting and one detail overstated.

**Adopted.**

- **§7.2's scope-based resolution was the weakest claim in the document**, and
  is replaced by the Strict Lexical Rule. The reviewer's reason was that
  pre-parsers do not build scope chains; **the stronger reason, which the review
  did not give, is that resolution during expansion is CIRCULAR** — a macro may
  introduce shadowing declarations, so the scope to resolve against is not final
  until the thing being decided has finished. No pre-parser capability fixes
  that; only making the rule syntactic does.
- **§7.1's `def_site` default had a missing mechanism.** A userland tag function
  has no way to mark a token as the macro's own, because `quote!`'s equivalent
  in Rust is itself privileged and a JavaScript tag is not. **A policy with no
  way to invoke it is not a policy.** Replaced by `TokenStream.gensym()`, which
  also avoids making identifier equality context-sensitive in the engine's
  hottest path.
- **§7.4 governed what code runs and not what it can reach.** Expansion is now
  given a purity constraint — a runtime sandbox at the time, compile-time
  evaluability after §9.13. **The strongest argument is one neither the review
  nor the draft made: determinism.** A preprocessor that can `fetch` makes
  expansion unrepeatable, so the output cannot be cached — forfeiting code
  caching in a design whose phase split exists to protect compilation
  performance.

**Overstated.** "Pre-parsers do not build scope chains" is too strong — V8's
pre-parser does track declarations, which is how it decides context allocation
for lazy functions. The conclusion survives, but on the circularity argument
rather than this one, and the distinction matters: an implementation-capability
claim invites "then make the pre-parser smarter", where a circularity does not.

**The reusable finding is §7.1's.** The earlier analysis treated protocol and
hygiene as one question — strings-with-gensym versus tokens-with-contexts — and
they are orthogonal. Tokens are right for span composition whatever the hygiene
answer; gensym is right for hygiene whatever the protocol. **Two questions
bundled into one alternative will be answered together and at least one of them
wrongly.**

## 5. Locking §7.2 and §7.4

Marking the two decisions settled forced two changes in the body that neither
section had mentioned, and one of them was an outright incompatibility:

- **§4.1's registration form was incompatible with §7.2.** It used a bare
  side-effect import with the module registering to global scope. A bare import
  **introduces no identifier**, so the Strict Lexical Rule — spell the decorator
  with the exact name the import introduced — has nothing to key on, and the
  ambiguity §7.2 exists to remove comes straight back. Registration now requires
  a named import clause.
- **§4.4's sandbox reaches EVALUATION, not just expansion.** A preprocessor
  module that could `fetch` while evaluating would bake network data into the
  closures its decorators are, and §7.4's determinism argument would fail
  identically. So the module cannot use top-level await either — the loader
  blocks on it the way it blocks for TLA, but there is nothing for the module
  itself to await.

**§5.4 also stopped being a decision.** It offered "error" versus "re-resolve"
and recommended the first; now §7.2 fixes the name set before expansion so the
error follows without a rule, and §7.4 removes the network so re-resolving is
not merely undesirable but impossible. **A section that was an open question is
now a consequence of two others** — which is the shape to look for when checking
whether a decision has really been absorbed, rather than only recorded.

## 6. Final pass

Every quotation re-checked verbatim (6/6), every **(measured)** claim re-run
against the implementation (9/9), all cross-references resolving, and the
twenty-two `Expression` fields recounted.

**One contradiction, from locking §7.1.** §4.2's `Span` interface still declared
`context: HygieneContext` — **the mechanism §7.1 had ruled out** — and §2, §3 and
§3.2 all described Part B as carrying hygiene contexts. Removing them simplified
Part A: there is no reading-only subset of the span to carve out, because the
span a reflection hands out is now the whole span.

**That is the same failure §9.2 recorded**, one revision later and in the
opposite direction: a decision was recorded in its own section while the
mechanism it displaced survived in the interface that declares it. **A settled
question leaves debris in the sections that used to depend on the other answer**,
and the check has to be a sweep for the displaced mechanism, not a reading of the
section that settled it.

**Two gaps filled ahead of a decision on §7.6 and §7.7**, since both bear on it:

- **§7.6's cost was overstated by its own framing.** "A macro cannot see types"
  suggests more loss than there is: a macro can emit code that NAMES a type and
  let the checker resolve it afterwards, which is how `serde` works. What is
  genuinely lost is narrower — a macro that must BRANCH on a resolved type.
- **§7.7 did not say whether a replacement may be EMPTY.** Conditional
  compilation needs it, and "must parse in the position it replaces" does not
  settle it: an empty stream parses as a statement and not as a parameter in the
  middle of a list. Left open, and named, rather than decided in passing.

## 7. Locking §7.6 and §7.7

As with §7.2 and §7.4, the value was in what the decisions forced elsewhere
rather than in the table cells.

**§7.6 settled what a replacement decorator RECEIVES, which no section had
said.** Expansion precedes checking, so a replacement decorator cannot be handed
a context object: every field of one — `type`, `metadata`, `access`,
`addInitializer` — needs either a resolved type or a runtime, and expansion has
neither. It receives tokens and nothing else. **The two decorator kinds now
differ as sharply in their input as in their output**, which is a cleaner account
of why they are two things than "they run at different times".

It also fixed checking's position in §4.4: parse, check, compile, with the
checker never seeing an unexpanded decoration.

**§7.7 required actually writing the table**, and writing it justified the
decision better than the argument for it did. The rows that CANNOT be
value-replaced — parameter, return, enum, tuple, record, `let`, `const`, and ten
of the twelve block forms — are the majority, and they carry a parameter
attribute, a derive over an enum, and conditional compilation. **Restricting
syntax replacement to the value table's positions would have excluded most of
what macros are for**, which was asserted in §7.7(b) and is only obvious once the
rows are on the page.

**One question stays open inside a settled section**: whether a replacement may
be EMPTY, and in which positions. It is named in §7.7 and in §8, and it is the
kind of thing that should be answered when the table is enumerated rather than
discovered by whoever first writes a `@cfg`.

## 8. Closing the last three

**The sandbox** became simpler by being stated as a rule rather than a list. The
rule adopted here was "ECMA-262 and nothing host-defined" — **superseded in
§9.11**, which keeps the rule-over-enumeration argument and fixes the boundary it
drew.

**Empty replacements needed no rule at all.** "Must parse in the position it
replaces" already answers it — empty parses as a statement or a class member and
not as a parameter — so `@cfg` works where it should and fails where it should.
**A grammar constraint answers "may it be empty" for free; a type constraint
would not have**, which is a second argument for §7.7's shape that only appeared
when this question was pushed at it.

**§7.6's residue turned out to be smaller than its own framing.** "Cannot see
types" is wrong: a macro sees every annotation AS WRITTEN and fails only to
resolve a NAME — so `temp: float64` is readable and `temp: Celsius` is not.

Asked how a macro could learn the resolved type, the two obvious answers both
fail, and **the first fails for exactly §7.2's reason**: resolving a type name
mid-expansion needs a type environment that expansion is still building, since a
macro can emit `type` declarations. **The same circularity has now ruled out
scope-based phase resolution and type-aware macros** — worth noticing as a
pattern, because anything that asks expansion to consult a namespace expansion
can modify will meet it.

The answer is deferral, and **(measured)** the targets exist: overload resolution
sees through an alias, so a macro emits `ser(this.temp)` and never branches; and
`Reflect.typeOf` sees through it at runtime for anything static dispatch cannot
carry. **This proposal has two deferral targets where Rust has one**, which is
why a limitation inherited from Rust's framing is milder here than there.

What genuinely does not work is a macro whose generated STRUCTURE depends on a
resolved type. No accommodation, because every escape reintroduces the
circularity.

## 9. A framing error in §7.6

§7.6 listed three "ways a macro could learn the resolved type" and made the
third "do not resolve — defer", which contradicts the heading it sat under.
**Deferral is not a way of learning; it discards the need to learn**, and
presenting it as a sibling of the other two made re-expansion and deferral look
like the same idea because both involve something happening after checking.

They are categorically different. **Re-expansion moves the TIMING of the macro's
decision; deferral moves the DECISION ITSELF out of the macro.** Under
re-expansion the macro still branches, so the compiler needs a loop over
expansion and checking. Under deferral the macro branches on nothing, so no loop
exists to need.

The test that separates them: **does the macro ever need to know the answer?**

This is a presentation error rather than a reasoning one — the conclusion was
right and the argument sound — but it is the kind that costs a reader more than a
wrong conclusion would, because the reader has to reconstruct the distinction
before they can disagree with anything.

## 10. Locking deferral

Deferral was described as what macros CAN do; locking it makes it what the
proposal SAYS to do, and that changed its status in three ways the descriptive
version did not have to carry.

- **It is now a dependency.** Deferral works because overload resolution and
  `Reflect.typeOf` see through an alias. Those were features of the proposal;
  they are now load-bearing for macros, and a change to either weakens the answer
  to §7.6.
- **The helper's scope became the macro author's problem, and then the user's.**
  An emitted `ser(this.temp)` needs `ser` visible at the splice, and a
  member-position macro emits into a class body where an `import` cannot go. So
  the user imports it, as Rust users `use serde::Serialize`. Obvious in
  retrospect and absent from every draft.
- **It located §7.1's residue.** The referential-transparency gap `gensym()`
  does not close had been abstract — "a macro emitting `Date.now()` meets a
  shadowing `Date`". **(measured)**, deferral is where it actually bites: the
  emitted helper call is a reference to something the macro did not create, so
  gensym cannot protect it, and a local `ser` wins.

**A pattern being endorsed is not the same as a pattern being available**, and
the difference is exactly these obligations. The descriptive version could stay
silent on all three because it was only claiming the pattern exists.

The negative decision — **no `resolveType()` on the token stream** — is recorded
for the same reason it will be proposed: it is the reasonable-sounding form of
the option §7.6 ruled out, and without the note someone re-derives it and meets
the circularity the long way round.

## 11. The sandbox rule was wrong

"ECMA-262 and nothing host-defined" was adopted in §9.8 on the argument that a
RULE beats an ENUMERATION, because a list of exclusions has to be maintained and
will be wrong once. The argument was right and **the rule was still wrong**:
`Date`, `Math.random`, `WeakRef` and every `toLocale*` method are all 262, so the
rule admitted the clock, randomness, observable GC and the host's locale — and
§7.4's determinism argument, which is the whole reason for the sandbox, does not
survive any of them.

**The error was drawing the boundary at a SPECIFICATION rather than at the
PROPERTY.** "262 versus host-defined" is a real line, but it is not the line
purity falls on. Restating the rule as *expansion is a pure function of its token
input* derives both clauses — exclude host capabilities, exclude the impure parts
of 262 — and keeps the advantage the original was chosen for, since it needs no
maintenance as either the host or the standard grows.

It also became testable, which the enumeration would not have been: **expand
twice, compare.** A conformance suite can check the property directly instead of
checking that some list of globals is absent.

**And it forced a distinction worth having.** A macro cannot READ the clock; it
can EMIT code that reads it. The emitted call runs later, in the user's realm.
Without saying so, the exclusion reads as "macros cannot generate time-using
code", which would be a much larger restriction than the one intended.

The finding came from outside the document — from noticing that the `Date.now()`
example in §7.1 pointed at a second problem in §7.4 that neither section was
looking for. **An example written to illustrate one rule turned out to be a
counterexample to another.**

## 12. The invocation cluster

§7.8 was raised as four open questions and closed as four, but only one of them
was the shape it first appeared to be.

**The stacking "conflict" was not one.** It looked like two rules giving opposite
orders on identical syntax, with one having to give. **decorators.md's own
justification resolved it**: rule 1 says the reverse-source order is "Python's
`a(b(c(x)))`" — composition over a VALUE, where the inner must produce before the
outer can consume. Replacement rewrites SYNTAX, where the outer must rewrite
before the inner is consumed. **Both say `a` is outside `b`; the execution order
differs because the media do.**

That is the third time in this document a conflict dissolved once the REASON
behind a rule was read rather than the rule itself — after §7.7's value-versus-
syntax tables and §7.6's cannot-see-types. **A rule stated without its reason
will eventually be applied where its reason does not hold**, and both the
original decorators.md rules and this document's own §5.2 carry their reasons,
which is why the collision was recoverable.

**Two carve-outs from decorators.md were needed**, and both are written as
carve-outs rather than left implicit: decorator arguments are not evaluated for
replacement decorators, and replacement decorator names are not overloaded. Each
was a rule whose mechanism — evaluation, type-directed selection — does not exist
at parse time.

**One genuinely new rule**: replacement decorators are written OUTERMOST. It came
from rejecting "the relative order carries no meaning", which would have been a
silent trap, and the choice between innermost and outermost turned on capability
rather than taste — an innermost replacement cannot see the runtime decorations
outside it, so a `@cfg` could not remove a member together with them.

**And one thing that needed no rule at all**: whether a replacement may rewrite a
runtime decoration it encloses. It follows from what "replace" means, exactly as
the empty-replacement question followed from "must parse in the position it
replaces". **Twice now, a question about permission turned out to be answered by
a definition already written** — which is worth checking for before adding a
rule.

## 13. Reading typeprogramming.md

Neither came from re-reading this document. Both came from checking it against a
part of the proposal it had never consulted.

**§7.4's mechanism was re-invented, and the proposal's version is better on the
proposal's own stated grounds.** This document specified a runtime sandbox;
[typeprogramming.md](typeprogramming.md) already has COMPILE-TIME EVALUABILITY
for the same purpose — "the proposal's answer to 'running user JS in the
compiler'" — with the same exclusions named explicitly (`Date.now`,
`Math.random`, I/O) and the same determinism justification. It also says why it
is preferable: **"the 'sandbox' is a property of what the code can name rather
than a wall around what it does"**, and "stricter and cheaper than a jail".

The property §7.4 identified was right; the mechanism was a second answer to a
solved problem. Switching gains a STATIC failure — a macro naming `Date` is
rejected where it is written rather than when it is called — and collapses two
disciplines into one, since type builders and replacement decorators are both
user JavaScript the compiler runs.

**§7.6's residue shrank again, and this time to one sentence.**
**(measured)** a runtime decorator's `context.type` is a type object, and
`Reflect.getReflection.<Reflect.Type>` cracks it open — `primitive`, `union:2`,
`object:2`, **and it resolves through an alias**. So "branch on a resolved type",
which §7.6 called the genuine loss, is available in full to a RUNTIME decorator.

A macro that needs it defers to one: it emits `@addSum`, and `@addSum` reflects
the type at definition time. **The residue is not "structure that depends on a
resolved type" — it is STATICALLY VISIBLE structure**, because a class
decorator's return is constrained to `T` and the checker keeps the declared type.

**The generalisable point.** §7.6 and §7.4 were each settled twice and verified
line by line, and both were still incomplete — not because the reasoning was
wrong but because neither had been checked against a sibling document. **A
document can be internally consistent, externally verified against the two
sources it cites, and still be re-deriving something the proposal solved
elsewhere.** The check that finds it is reading the proposal, not the document.

## 14. What the plan found

Writing the implementation plan surfaced four things §§1–8 had not settled. Three
were omissions; one was an interaction with decorators.md that nothing had looked
for.

- **A macro had no way to REJECT its input.** `@derive` on a non-class must be
  able to say so, and no section said how. Settled as throwing — §7.8(vi) — which
  is what a JavaScript function does to reject arguments, and which spans already
  locate.
- **`SourceRef` was used and never defined.** It appeared in `Span` from the
  first token draft and no revision noticed, because every consistency sweep
  looked for CONTRADICTIONS and this was an absence.
- **§5.4 had no home in the plan.** Settled in the design as a consequence of
  §7.2, which is exactly why it was easy to miss: a decision that follows from
  another still needs its own diagnostic.
- **decorators.md rule 5 — block decorators run per ENTRY — is a third
  carve-out.** A replacement block decorator runs once, at parse time.

**The last one is the find, and it points the other way from the other
carve-outs.** Rule 5 warns that a block decorator "in a hot loop costs a context
and a call per iteration, and that cost is not visible at the declaration site".
**That warning does not apply to the replacement form**, which rewrites once and
can emit the instrumentation inline — so the per-iteration work happens and the
per-iteration decorator does not.

That is a motivation for this feature the design had never stated: not merely
"macros can do things decorators cannot", but **a replacement decorator is the
answer to the one performance warning decorators.md issues about itself.**

**The general point:** a plan is a different reader of a design than a reviewer
is. A reviewer checks whether what is written is right; a plan asks what must be
built, and everything the design left unsaid becomes a clause with nothing to
put in it.

## 15. Implementing the proposal changes

The document split and all six decorators.md changes landed together
(`design-decorator-replacement.patch`). One estimate in the plan was wrong and
one mechanical step needed care.

**`initial` is on TEN reflections, not two.** The plan named `ClassField` and
`ClassMethodParameter` from memory. Enumerating found `ClassAccessor`,
`ClassSetterParameter`, `ClassOperatorParameter`, `FunctionParameter`, `Let`,
`Const`, `ObjectSetterParameter` and `ObjectMethodParameter` as well — every one
carrying the same constants-only limitation, and every one now carrying
`initializer`. **A count taken from memory in a plan is an estimate whatever it
looks like**, and this one was out by a factor of five.

**A hand edit and a scripted edit collided.** `ClassMethodParameterReflection`
was given `initializer` manually while working through the example, then again by
the pass over all ten — two identical adjacent lines, which reads as correct at a
glance. Caught by counting fields against reflections rather than by reading the
diff: 11 fields for 10 reflections. **When a mechanical pass follows a manual
one, the check is a count and not a reading.**

The three carve-outs went where their rules live rather than only in the new
document — argument evaluation beside "evaluated in document order", overload
selection beside the `@f`/`@f(0)`/`@f('a')` rule, and block frequency beside
rule 5 — which was the point of §2.5: a rule with its exception recorded
elsewhere is a rule that will be applied without it.
