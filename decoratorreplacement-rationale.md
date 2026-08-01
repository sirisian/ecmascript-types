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

## 16. Implementing the specification changes

Eight new clauses, two edits, six early errors — and the plan was wrong about one
edit in a way worth recording.

**The plan listed four edited clauses; there are three.** It called for changing
block reflection fields to `TokenStream` in `sec-decorators`, and **the
specification does not define reflection fields at all** — `BlockReflection` and
`ClassFieldReflection` appear nowhere in it. `sec-decorator-contexts` says so
outright: "This is the one context this specification defines; the rest are the
decorators extension's."

So the `Expression`  to  `TokenStream` change is a decorators.md change only. **The
plan assumed a division of labour between the design documents and the
specification without checking where the line actually falls**, which is the same
class of error as the ten-versus-two `initial` count: an estimate wearing the
clothes of a fact.

**`sec-expansion` was the load-bearing clause and it landed smaller than
expected.** Inserting a phase between scanning and parsing needed one algorithm
and four notes, because everything that would have made it complicated had
already been decided: the fixpoint terminates by a limit, the order is
outer-first, nothing is re-lexed because the input is tokens, and the name set is
fixed before the loop so no import can appear mid-expansion. **A clause is short
when its decisions were made elsewhere.**

**One convention check worth having done.** The new `sec-import-attributes`
reference reads as dangling, because it names an ECMA-262 clause rather than one
of this proposal's. Six such references already existed and use exactly the same
form — `<emu-xref href="#sec-numeric-types"></emu-xref>` — so the new one is
consistent rather than broken. **Comparing the dangling set before and after
separates "I introduced an error" from "this is how the document already cites
its base."**

## 17. Verifying the implementation — a probe on the wrong branch

The coverage pass against the plan found one genuine specification gap and one
error that had been running since the beginning.

**The gap:** `ApplyReplacementDecorator` never set [[Macro]] or [[Generation]] on
the tokens a decorator returns. The Source Reference Record table defined the
fields and nothing populated them — the same shape as `SourceRef` being used and
never defined, caught the same way, by checking a table's fields against their
assignments rather than reading the prose.

Closing it settled something the prose had implied and never said: a token a
decorator COPIED keeps the Span it arrived with, and only a token it CREATED is
attributed to the decorator. That is what makes a diagnostic inside generated
code point at the macro rather than at the splice.

**The error:** every measurement of the block decoration grammar in this project
wrote the decorator in the wrong place. **It goes on the BLOCK** —
`if (c) @g { … }`, not `@g if (c) { … }` — and the reflection context is chosen
by the block's position, which is exactly why `IfBlockReflection` carries a
`condition` the decorator did not write.

Measured on the right branch, eight forms already work: `Block`, `IfBlock`,
`ElseBlock`, `ElseIfBlock`, `WhileBlock`, `DoWhileBlock`, `ForBlock`,
`ForInBlock`, `ForOfBlock`. **"The other block forms are not in the grammar" was
never true**, and it had been carried in the design's state table, in the plan's
out-of-scope list, and in the rationale.

Two things hid it. The syntax was inferred from `@g { … }` working and never
checked against the reflections that name an enclosing statement — **the
`condition` field was the clue and it sat unread for the whole project**. And the
probes were run against the wrong branch: a chained
`git reset --hard origin/master || origin/main || origin/proposal-runtime-types`
succeeded at `origin/main` for the engine, which HAS that branch, so it never
reached the proposal branch and every measurement in that cycle described
upstream engine262.

**A fallback chain is not a fallback when an earlier alternative can succeed for
the wrong reason.** The first symptom was `@g { … }` failing — a form measured
working many times — and treating that as a regression rather than as an apparatus
failure cost a bisect before the branch was checked.

## 18. The specification did not build

Two ecmarkup errors survived every review pass of this work, and both were
invisible to the checks that were being run.

**A step ended in `, and`.** The `TokensOf` algorithm was written as a sentence
with a `where:` clause and two joined sub-items — correct English, and not an
algorithm. `algorithm-line-style` requires each freeform step to end with a
period. Rewriting it as a loop with a step per token fixed the lint and produced
a better operation: the recursion into a delimited run is now explicit rather
than implied by the phrase "the tokens of the run".

**`sec-import-attributes` does not exist.** An earlier pass compared the dangling
reference set before and after, found six pre-existing base-spec references in
the same form, and concluded the new one was "consistent rather than broken".
**That reasoning was wrong.** The six resolve through `@tc39/ecma262-biblio`;
the seventh did not, because the biblio names those clauses
`importattribute-record` and `sec-hostgetsupportedimportattributes`.

**A convention check says a reference is well-FORMED, not that its target
exists.** The two questions look alike and only one of them was being asked.

Both were found by a build that had never been run — `npm run build` in the
specification repository runs ecmarkup with `--strict` and `--lint-spec`, and it
had been sitting in `package.json` throughout. **Balanced tags, resolving
internal cross-references and a coherent reading are three proxies for a build,
and the build is not expensive.**

Fixing the second improved the text: the clause now names the ImportAttribute
Record's [[Key]] and [[Value]] rather than gesturing at "the import attribute",
and the Static Semantics operation matches it — which is the specification's own
vocabulary, arrived at because the wrong reference forced a look at the right
one.

## 19. Two verifications that verified nothing

Closing out this work, two checks turned out to be circular, and both had already
reported success.

**A `git stash -u && git reset --hard` chain broke at the stash**, so HEAD never
moved. Every check afterwards read the working tree, which held the very edits
being checked: "upstream is byte-identical to mine" compared a file to itself,
and "the patch reverse-applies" tested a patch against the tree it was generated
from. Both said yes. Both would have said yes if the work had never left this
machine.

**The conclusion happened to be right** - the work had landed - which is the
uncomfortable part. A circular check that agrees with reality teaches nothing and
leaves no trace. Reading `git show origin/master:decorators.md` instead of the
file on disk is what finally distinguished them, and it is the only form of the
question that could have.

**Then a write helper truncated a file before failing.** Appending to this
document meant encoding UTF-8 text into its cp1252 bytes, and a right-arrow
character has no cp1252 encoding. The exception fired mid-write, after the file
had been opened and emptied - 479 lines gone. Computing the payload fully and
only then opening the file for write is the rule this session recorded on its
first day, and it was not followed here.

**Both failures share a shape**: a step that can fail silently sits before a step
that reports success. A stash that returns non-zero, an encode that throws after
a truncate. **The check to run is not "did the last step succeed" but "did the
step I am relying on actually happen".**

## 20. The specification had the defect too

Implementing expansion found two things wrong in the ENGINE and, on checking,
one of them was wrong in the specification as well.

**The engine applied every expansion site in one pass.** For `@a @b class C {}`
the outer decoration's range CONTAINS the inner one's, so two edits overlapped
and corrupted each other. **The specification was already right** - `sec-expansion`
says "let _d_ be the outermost such decoration in source order", singular, one per
iteration - so this was an implementation diverging from a clear rule rather than
a rule that needed writing. Taking the algorithm literally fixed it.

**But `sec-applyreplacementdecorator` had the other defect.** It said
"Let _target_ be TokensOf(the declaration _d_ decorates)" while `sec-expansion`
replaces "_d_ itself" TOGETHER WITH what it decorates. Those two do not agree: a
decoration written between them - `@a @r class C {}` - is inside the range being
replaced and outside what the macro was handed, so it was dropped silently.

That contradicts §7.8(iii), which settles that a replacement encloses the runtime
decorations and may rewrite or remove them, and it is the whole reason the
outermost rule exists. _target_ is now everything the decoration encloses.

**The pattern worth keeping**: the two clauses were each defensible alone and
disagreed about a range. **A specification can be internally consistent
clause-by-clause and still describe an operation that loses information**, and
what found it was implementing the thing and printing what came out - not reading
either clause again.

`[[LineTerminatorBefore]]` also landed on the Token Record, which the engine had
carried since stage G and the specification had not. Newlines are semantically
significant through ASI, so a printer emitting a space where the source had a
newline changes what a program means.
