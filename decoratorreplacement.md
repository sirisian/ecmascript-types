# Decorator Replacement

> **Status: draft, for iteration.** Part A is ready to specify. Part B's open
> questions are analysed in §7, each with a decision or a recommendation and the
> reasons the alternatives are ruled out. **§7.1 through §7.4 are settled and §7.5
> does not apply**, and §7.6 and §7.7 are settled. **Every open question in §7 is
> now decided**, including §7.8's invocation cluster. What remains is
> specification work, listed at the end of §8. Part B exchanges TOKEN
> STREAMS WITH SPANS — the rationale records the two reversals that got there, since the
> reasoning is more useful than the conclusions.
>
> Claims marked **(measured)** were checked against the reference implementation.
> §6 is recalled rather than measured and should be confirmed against each
> language's own documentation before it is cited anywhere normative.

## 1. What this replaces

[decorators.md](decorators.md) gives every block reflection a field it does not
define:

```js
namespace Reflect {
	type BlockReflection = {
		label?: string;
		block: Expression;
	};
}
```

and says of it:

> That `Expression` is not defined here. Macro AST is out of scope. The
> Expression is a placeholder.

and, of the parameter case:

> Similar to `Reflect.ClassField` the parameter decorators only capture constant
> values. This is a limitation. Ideally one could capture the `Expression`, but
> I'm trying not to directly implement AST into this yet. Just leaving it as an
> extension.

**These are two different gaps and this document closes both**, but they are not
the same shape and an earlier draft conflated them:

- **The block family has a PLACEHOLDER FIELD.** Twelve reflection types carry
  `Expression` across twenty-two fields; the field exists and its type does not.
  Here a token stream REPLACES something.
- **`initial` on fields and parameters has a LIMITATION.** There is no
  `Expression` field; there is `initial: T | undefined`, which captures
  constants only. **(measured)** `@g a: uint8 = f()` reports `initial` as
  *undefined*, and so does a parameter default `x: uint32 = f()`. Here a token
  stream ADDS something beside `initial`.

The second is the larger win in practice. A decorator over a field whose
initializer is `f()` currently learns nothing about it at all; with its syntax it
learns what was written, which is what a derive or a dependency-injection
decorator wants.

**`Expression` becomes a TOKEN STREAM.** An earlier draft made it a string, on
the argument that anything structured would have to be standardized and
versioned. That argument conflated a token stream with an AST:

- ECMAScript's **lexical** grammar is already normative, with defined token
  kinds — `IdentifierName`, `Punctuator`, `NumericLiteral`, `StringLiteral`,
  `Template`, `RegularExpressionLiteral`. Exposing tokens exposes a vocabulary
  the specification already maintains; exposing an AST would invent one.
- That vocabulary changes far more slowly than the syntactic grammar. A new
  statement form adds no tokens. A new operator adds one punctuator. **The
  perpetual-breakage surface that makes an AST unwise is mostly absent for
  tokens.**

**There is no `source: string` field**, and the rationale records why a draft
kept one and why that was wrong: source text is DERIVABLE from a token's span, so a separate
field would be a second way to say one thing — two derivations that must agree,
which is the failure this project has met more often than any other.

**(measured)** In the reference implementation, source text survives verbatim
and includes type annotations:

```js
function f(a: uint8): uint8 { return a; }
f.toString();   // "function f(a: uint8): uint8 { return a; }"
```

## 2. Two parts, landed separately

**Part A — reading.** A decorated construct's syntax can be READ, as a token
stream. This needs the token vocabulary and spans, and nothing else: no
preprocessor phase, no execution during parsing, no hygiene, no CSP question.

**Part B — replacement.** A decorator can RETURN a token stream, which is spliced
before compilation. This needs the preprocessor phase, a phase-resolution rule, a
hygiene primitive, and an answer on CSP.

**They are separated because reading is sound without writing.** A reflection
that hands out tokens is useful on its own — a derive that inspects, a linter, a
documentation extractor — and none of Part B's hard questions arise until
something is spliced. If Part B is never specified, Part A stands.

The reverse is not true: a replacement protocol that has not settled what a
decorator can SEE has settled nothing.

## 3. Part A — reading syntax

Every reflection that decorators.md gives an `Expression` field takes a
`TokenStream` instead. The name changes because `Expression` implies a parsed
shape and a token stream is deliberately less than that.

```js
namespace Reflect {
	type BlockReflection = {
		label?: string;
		block: TokenStream;
	};

	type IfBlockReflection = {
		label?: string;
		block: TokenStream;
		condition: TokenStream;
	};

	type ForBlockReflection = {
		label?: string;
		block: TokenStream;
		initializer?: TokenStream;
		condition?: TokenStream;
		update?: TokenStream;
	};
}
```

The stream is defined in §4.2, and Part A needs all of it — kinds, spans,
grouping. There is no reading-only subset to carve out, because §7.1 put hygiene
in a minting primitive rather than in the span, so the span a reflection hands
out is the whole span.

**The family is twelve reflection types, not the three above**: `Block`,
`IfBlock`, `ElseIfBlock`, `ElseBlock`, `WhileBlock`, `DoWhileBlock`, `ForBlock`,
`ForInBlock`, `ForOfBlock`, `DoBlock`, `DoGeneratorBlock` and `MatchArmBlock`.
Two are worth calling out:

- **`MatchArmBlockReflection` carries `subject`, `pattern?` and `guard?`** as
  well as `block`. A `pattern` as tokens is the only way a decorator could ever
  see one, since a pattern is not a value and has no runtime form.
- **`DoBlockReflection` and `DoGeneratorBlockReflection` are the two block
  positions that are ALSO value-replaceable**, so they appear in both tables of
  §7.7. §7.2 is what keeps that unambiguous: a runtime decorator's return
  replaces the `do` expression's VALUE, and a replacement decorator's return
  replaces its SYNTAX. Without a phase rule the same position has two kinds of
  return.

### 3.1 Source text is DERIVED, not a second field

A draft of this document carried a `source: string` field beside the tokens, on
two arguments. Both fail.

**"Tokens lose comments and whitespace."** They do in Rust, which is a
CONVENTION of that stream and not a property of token streams. Here a span
carries a buffer and a character range, so **everything between two consecutive
tokens is exactly the whitespace and comments between them** — recoverable, not
discarded. **(measured)** the buffer holds it:

```js
function f() { /* keep me */ return 1; }   // toString() keeps the comment
function g(  a,   b  ) { … }               // keeps the exact spacing
```

So the fidelity argument was about Rust's convention, imported as though it were
inherent.

**"It lets Part A land without the token vocabulary."** True, and not worth what
it costs. **A `source` field beside a token stream is two ways to say one
thing**, and they must agree forever — the shape this project has been bitten by
more than any other, most recently where a member's `type` was reported by the
decorator context and not the read path. One of the two is redundant, and the
redundant one is the string, because the tokens carry strictly more.

`TokenStream.prototype.toString()` covers what a `source` field was actually
wanted for — logging, snapshotting, an error message — as a method on the stream
rather than a parallel field on every reflection. Rust's `TokenStream` implements
`Display` for the same reason.

### 3.2 What Part A does NOT need

- No preprocessor phase. The tokens are available when the rest of the context
  is.
- No CSP question. Nothing is compiled that was not already written.
- No hygiene. Nothing is spliced, and §7.1's `gensym()` is a Part B facility.
- No AST. The stream is deliberately below one.

### 3.3 State of the block family

**(measured)** in the reference implementation:

| form | today |
| --- | --- |
| `@g { … }` | fires, context is `{ kind, label }` |
| `lbl: @g { … }` | parses, and **`label` is `undefined`** |
| `@g if (…) { … }` | SyntaxError |
| `@g while (…) { … }` | SyntaxError |
| `@g for (…) { … }` | SyntaxError |

**(measured)** eleven of the twelve reflection types exist;
**`Reflect.MatchArmBlock` does not** — an earlier draft said the whole family was
present. So Part A is four pieces of work, only the last of which is this
document's:

1. **`label` is never populated.** A defect independent of this proposal — the
   label is in the AST at the decoration site and nothing reads it. **A property
   that always answers `undefined` is worse than an absent one.**
2. **The other block forms are not in the grammar.** Ordinary grammar work.
3. **`Reflect.MatchArmBlock` is absent** while the other eleven exist. Since the
   `match` expression itself is implemented, this is a missing reflection rather
   than a missing feature.
4. **The `Expression` fields themselves** — this document.

## 4. Part B — replacement

### 4.1 Registration

A module that provides replacements is imported with an attribute, so the engine
knows before parsing the host module that it must run first — **and with NAMED
bindings**, because §7.2's rule is that a replacement decorator is spelled with
the exact identifier its import introduced:

```javascript
import { derive, logged } from './expand.js' with { preprocessor: true };

@derive class Point { x: float64 = 0; y: float64 = 0; }
```

**A bare side-effect import will not do.** An earlier draft wrote
`import './expand.js' with { preprocessor: true }` and had the module register
its decorators to the global scope — which introduces no identifier, so there is
nothing for the rule to key on and the ambiguity §7.2 rules out comes straight
back. The names must arrive through the import clause.

Two properties follow, and they are why the rule is worth its strictness:

- **The set of replacement names is a syntactic scan of the module's import
  clauses**, decided before expansion begins.
- **Nothing expansion does can change it** — see §5.4.

### 4.2 Protocol

This is the stream Part A reads and Part B exchanges. A replacement decorator
receives one and returns one; each token carries a **span** saying where it came
from. The same span shape serves both parts — §7.1 settled hygiene on a minting
primitive rather than on span contexts, so there is nothing extra for Part B to
carry.

**A replacement decorator receives TOKENS AND A `{ kind }` CONTEXT** —
`(tokens, context, args)`. The type-dependent half of a runtime context is
absent and §7.6 is why: `type`, `metadata`, `access` and `addInitializer` each
need a resolved type or a running program, and expansion has neither.

**The SYNTACTIC half is absent for a different reason, and §3.1 already gave
it.** A `source` field beside a token stream was rejected there as "two ways to
say one thing", and the same argument disposes of `name`, `static`, `private`, a
`for`'s binding and a match arm's pattern: a replacement decorator receives the
TOKENS OF WHAT IT DECORATES, so all of them are in those tokens already. A
runtime decorator needs them in its context because it is handed no tokens.

What is left is the one thing the tokens cannot say — WHICH POSITION they came
from — and that is `kind`, whose values are decorators.md's reflection names.
The object is frozen: a context is a report, not a channel.

| | runtime decorator | replacement decorator |
| --- | --- | --- |
| receives | a context object — `type`, `metadata`, `access`, … | a token stream and `{ kind }` |
| returns | a value (decorators.md's table) | a token stream (§7.7) |
| runs | at definition time, after checking | before parsing completes |

**Typing the context parameter declares WHERE a decorator applies** —
`(tokens: TokenStream, context: Reflect.Region)` is refused on a class, at the
decoration rather than inside the macro. It is optional, and has to be: the
specification lets one decorator serve several positions without being told
which it is in, and a required parameter would force every macro to enumerate
positions including the ones that work anywhere. **(measured)** the existing
checker enforces it with nothing added, the context being assignable to the
`Reflect.*` context types already.

**This is why Part A matters to both.** A runtime decorator gets its context AND,
with Part A, the tokens of what it decorates — types and syntax together, because
by then both exist.

```typescript
interface Token {
	kind: 'identifier' | 'punctuator' | 'numeric' | 'string' | 'template' | 'regexp' | 'group';
	value: string;
	span: Span;
	tokens?: Token[];   // `group` only: a delimited run, with its delimiter in `value`
}

interface Span {
	source: SourceRef;   // which buffer — the module, or a macro's output
	start: number;       // inclusive character index in that buffer
	end: number;         // exclusive
}

interface SourceRef {
	url?: string;        // the module's URL, for a token from written source
	macro?: string;      // the replacement decorator's name, for a generated one
	generation: number;  // 0 for written source; n for the nth expansion that produced it
}
```

**Delimited runs are grouped rather than flat.** A macro that receives
`{ a; b; }` receives one `group` token whose `value` is `{` and whose `tokens`
are its contents. This is what makes boundary detection reusable — the engine
already had to match delimiters to find the decoration boundary at all — and it
means a macro cannot emit unbalanced output by construction.

**A span is not a source map.** It says where a token came from; the engine
composes whatever chain of origins arises, and §5.3 explains why that matters
more than it sounds.

#### Constructing tokens

Building a stream by hand is tedious in any language — Rust needed the `quote!`
macro for it — but JavaScript already has template literals, so a userland
helper is a tag function:

```javascript
export function derive(input, ...args) {
	const name = input.find(t => t.kind === 'identifier');
	return js`class ${name} { serialize() { return "…"; } }`;
}
```

**The trailing arguments are the decorator's own**, and they are tokens like
everything else — `@derive(Serialize)` passes the identifier token `Serialize`,
never an evaluated value. §7.8(i) settles that and explains why nothing else was
available at parse time.

The tag lexes its literal parts and splices the interpolated tokens, preserving
their spans. **That the helper is userland is deliberate**: it is the `syn`
position — the engine standardizes the stream, and parsing it into something
higher-level is a library's business.

Which is exactly why hygiene cannot ride on a privileged construction step. A
userland tag has no way to mark a token as the macro's own, so the engine
exposes minting directly:

```javascript
const tmp = TokenStream.gensym('start');   // an identifier that cannot collide
return js`const ${tmp} = Date.now(); ${input} console.log(Date.now() - ${tmp});`;
```

See §7.1 — this is the mechanism, and its absence is what ruled out `def_site`
contexts.

### 4.3 The `/` problem, which Rust does not have

JavaScript tokenization is **not context-free**. `a / b / c` and `a = /b/c`
differ by parse context, and the specification resolves it with separate goal
symbols — `InputElementDiv`, `InputElementRegExp`, `InputElementTemplateTail`.

So a JavaScript token stream is **parse-informed**, not purely lexical, in a way
Rust's is not. Two consequences:

- **Incoming** tokens are unambiguous *where the decorated region is
  ECMAScript*, because the engine resolved the goal symbol when it produced
  them. That qualifier is load-bearing and was missing here; see below.
- **Outgoing** tokens are the problem. A macro emitting `/` must say which it
  means, or the engine must re-derive it — and re-deriving means re-parsing the
  output, which is the cost tokens were meant to avoid.

**The token kind carries it**: `regexp` and `punctuator` are distinct kinds, so a
macro that emits a regular expression emits a `regexp` token and one that emits
division emits a `punctuator`. The ambiguity is resolved by construction rather
than by re-scanning.

Rust has the same shape of problem in miniature — `>>` as a shift versus two
closing generics — and solves it the same way, with `Spacing::Joint`/`Alone` on
`Punct`. **This is the one place the Rust analogy needs a JavaScript-specific
answer**, and it is a token-kind distinction rather than a new mechanism.

#### The incoming direction is a second problem, and it is not the `/`

The bullet above originally read that incoming tokens are simply unambiguous.
That is true of a region the engine can lex, and every region it can lex is
ECMAScript — which is precisely what a syntax replacement is often wanted for
NOT being. A macro over a DSL cannot receive tokens the engine cannot produce.

JSX is the case that shows it, and the reason is not the `/` this section is
about. Measured in an implementation:

| source | result |
| --- | --- |
| `const v = < 2;` | Unexpected token |
| `const v = <div/>;` | Unexpected token — the **same** failure |
| `const a = 1; const v = a </div>/;` | **parses**, as `a < /div>/` |

`<` cannot begin an expression, so the parse stops there and never reaches the
closing tag. The third row is this section's ambiguity behaving exactly as
described, and it is downstream: it would matter only if the first two rows
succeeded.

So there are two problems of different kinds. **`<` beginning an expression is a
GRAMMAR question; `/` after it is a GOAL-SYMBOL question.** This section answers
the second, in the outgoing direction, by construction. The first has no answer
in a token-kind distinction, because there are no tokens yet to distinguish.

**The answer is to let the decorator CAPTURE its region and read it itself.** A
preprocessor decoration followed by `{` takes that brace and its match as a
region; where the decorator declares `capture`, the region is found by delimiter
and its text reaches the macro as tokens of the ordinary lexical grammar, for the
macro to read however its own syntax requires:

```js
import { jsx } from "./jsx.js" with { preprocessor: "true" };
const el = @jsx do { <div class="a">{name}</div> };
```

```js
// in jsx.js
export const jsx = Object.assign(expand, { capture: true });
```

The engine provides no grammars, so a decorator is not choosing among them. What
a macro cannot do for itself is decide whether `/` begins a regular expression or
a division - undecidable lexically, since after `}` it depends on whether the
brace closed a block or an object literal - so it hands those ranges back with
```stream.parse(start, end, goal)``` and gets tokens threaded from that parse. A
JSX VARIANT with different syntax decisions is therefore a different macro rather
than a different engine, which naming a grammar could never have given.

A capture governs INGESTION only — what a macro returns is ECMAScript however its
region was scanned — and the grammar outside a region is untouched, so `a < b`
and `.<T>` read as they always did. Keeping JSX a macro's business rather than
the language's is the whole point: TypeScript declines to disambiguate `<` in the
grammar at all and switches on the file extension instead.

Two consequences worth stating, because they are easy to assume away in turn:

- **Delimiter matching cannot be naive.** A region's end is found without lexing
  its contents, so string and comment forms must be recognised even though
  nothing else is. Otherwise `<a title="}">` ends the region at the wrong brace.
- **A regular expression is deliberately NOT recognised while scanning a
  region.** Inside one the contents are not ECMAScript, so `/` is whatever the
  mode says — in JSX it opens a closing tag — and a `{` within it is a real
  delimiter. Outside a region the opposite holds, which is the third row above.
  No single scanner serves both readings, which is why a region declares its
  mode rather than being guessed at.

### 4.4 Phases

1. **Preprocessor resolution.** Imports tagged `preprocessor: true` are fetched
   and evaluated before the host module is parsed. The LOADER blocks on this the
   way it blocks for top-level await — but the module itself cannot await, since
   §7.4 gives evaluation no asynchronous capability. Its import clause fixes the
   replacement names for the whole module (§4.1).
2. **Expansion.** The host module is pre-parsed far enough to find decoration
   boundaries; each replacement decorator is called with the tokens it decorates;
   the returned stream is spliced. **Nothing is re-lexed** — the splice is
   already tokens — so the engine walks the returned stream for nested
   decorators and repeats. See §5.
3. **Compilation.** The expanded stream is parsed, **then checked, then
   compiled**. §7.6 fixes that order normatively: the checker sees only expanded
   code, and never an unexpanded decoration. A type error inside generated code
   is reported through the spans its tokens carry.

**Phases 1 and 2 are subject to COMPILE-TIME EVALUABILITY** (§7.4): a
preprocessor module and every decorator it exports must be evaluable, so neither
can NAME the clock, randomness, I/O or ambient state. The constraint covers
evaluation as well as expansion, because a module that could fetch while
evaluating would bake network data into the closures its decorators are, and
determinism would be lost just the same.

## 5. Expansion is a fixpoint

**Macro output MAY contain macros.** This is settled, and it has four
consequences that the protocol has to carry.

### 5.1 A recursion limit is required

`@f x` expanding to `@f x` does not terminate. Expansion needs a depth limit
with an error naming the macro and the depth, not a hang. Rust's default is 128
and configurable per crate; the limit here should be specified rather than left
to implementations, or the same program will compile in one engine and not
another.

### 5.2 Order has to be stated: OUTSIDE-IN

An outer macro sees its inner macros **unexpanded**. That is the useful order —
it is what lets an outer macro rewrite or delete an inner one — but it means a
macro may receive tokens it does not understand, and must pass them through
rather than choke on them.

**Tokens make that survivable in a way strings would not.** An unexpanded inner
macro is still well-formed tokens with balanced groups, so an outer macro can
copy or move it without understanding it. A string containing an unexpanded
macro is just characters, and any outer macro that wanted to relocate it would
have to parse it first.

The alternative, inside-out, makes every macro's input fully expanded but makes
an outer macro unable to see what it contains. **Outside-in is settled**,
matching attribute macros in Rust, with the consequence written down rather than
discovered.

### 5.3 Spans compose themselves — and this is why tokens won

Nesting is where a string protocol becomes expensive, and where tokens stop
being a preference and start being the answer.

**With strings**, one expansion needs a map relating generated text to original
text; nesting needs a CHAIN — final text ? B's output ? A's output ? source —
and a position has to be walked back through every layer. Source-map composition
is where map correctness usually fails, and it fails SILENTLY: a stack trace
points somewhere plausible and wrong.

**With tokens, there is no chain to compose.** A token carries its origin span
wherever it is moved, so a token that macro A copied from the user's source and
macro B then relocated still says where the user wrote it. The engine folds
nothing; nothing was ever unfolded.

**This deleted a protocol and its hardest failure mode**, and it is the argument
that §7.1's earlier draft under-counted. That draft weighed tokens against
strings on HYGIENE alone and found the Rust ecosystem mostly tolerates
`call_site`. It never weighed them on span composition — where Rust macro
authors have no equivalent complaint precisely because spans make the problem
not exist.

### 5.4 A new macro name cannot appear during expansion

**This is no longer a choice; it follows from two settled decisions.**

§7.2 fixes the replacement names by a syntactic scan of the import clauses,
before expansion starts. So an expansion that introduces `@g` for a `g` no
`preprocessor: true` import brought in names something that is not a replacement
decorator — and it is an error, without a rule of its own.

The alternative — re-running resolution to fetch what an expansion introduced —
is not merely undesirable now, it is **impossible**: §7.4 makes expansion
synchronous and IO-free, so there is nothing to fetch with.

What this buys, and why both decisions are worth their cost: **the preprocessor
graph stays statically knowable**, which is what lets a bundler or an HTTP/2
server push preprocessors ahead of the module that needs them, and what lets a
build tool perform expansion ahead of time and get the answer the engine would.
It also matches Rust, where a macro must be in scope at its expansion site.

## 6. Prior art

> **Recalled, not measured.** Confirm before citing normatively; the Rust
> Rust `TokenStream` distinction below is the load-bearing one and the easiest
> to check, since §7.1 now rests on it.

- **Rust proc macros** receive a `TokenStream`, not a string, and `syn` parses
  that. `TokenStream::to_string()` exists and is documented as lossy. **This
  proposal now takes the same position** — §7.1 — having first argued against it
  and been wrong about the cost.
  Three places where JavaScript differs: the `/` ambiguity has no Rust analogue
  (§4.3); a region the engine cannot LEX has none either, since Rust's lexer is
  context-free and can tokenize any delimited run, which is why a mode is needed
  here and not there (§4.3); and `quote!` is a crate where a tag function is
  native (§4.2).
- **Sweet.js** is the JavaScript macro system that got furthest, and made
  hygiene its central feature rather than an extension.
- **Scheme `syntax-rules`** is hygienic by construction and is the origin of the
  term.
- **C preprocessor** is the counterexample: textual, unhygienic, and its capture
  bugs are the standard argument against string-based macros.

## 7. Decisions, and the alternatives rejected

Each decision below gives the alternatives, what each solves, what each costs,
and why the others are rejected. The criteria are the ones this proposal uses
elsewhere: **what a developer would expect**, **performance**,
**implementability**, and **whether it fits what the proposal already decided**.

---

### 7.1 Hygiene — tokens for spans, gensym for hygiene — SETTLED

A string has no hygiene. Splicing one means every identifier in it binds to
whatever is in scope at the splice site.

```js
@logExecution
function foo() { const start = getStart(); return start; }
```

If `logExecution` emits `const start = Date.now()`, it captures the user's
`start`, silently. Two distinct failures hide under one word:

- **Capture of a macro's OWN temporaries** — it introduces `start` and collides.
  Common, mechanical, and the macro author causes it.
- **Loss of referential transparency** — it emits `Date.now()` and the user has a
  local `Date`. Rarer, and the USER causes it, so no amount of care by the macro
  author avoids it.

#### (a) Accept capture, document it

**Solves:** nothing, at zero cost. The C preprocessor's answer.

**Costs:** the failure is a silent wrong binding, not an error. A macro that
works in every test captures on the one call site whose local happens to
collide — rare, invisible, and far from the code that caused it. **The worst
ergonomic profile available.**

#### (b) A gensym primitive

The engine mints identifiers that cannot collide; macros use them for anything
they introduce.

**Solves:** the first failure. Free at runtime, and composes with a string
protocol without changing it.

**Costs:** discipline — forget once and you capture. Nothing for referential
transparency.

#### (c) Token streams with spans

Spans carry hygiene context, so an identifier a macro CREATES is distinguishable
from one it RECEIVED, and the two cannot collide.

**Solves:** both failures, properly — and, per §5.3, the span-composition
problem as well.

**Costs:** the protocol is no longer opaque strings. A token vocabulary is
standardized — but see §1: ECMAScript's lexical grammar is already normative and
changes far more slowly than its syntactic grammar, so this is far less than an
AST and less than the earlier draft assumed.

#### (d) Declared introductions

The macro enumerates the identifiers it introduces; the engine alpha-renames.

**Costs:** the engine must rename inside something it does not parse, so it must
find identifier occurrences — and get them wrong inside string literals,
comments and property names — unless it lexes the output. **That is (c)'s cost
for a fraction of (c)'s benefit.**

#### These are TWO axes, not one — and an earlier draft treated them as one

The protocol question and the hygiene question are independent, and conflating
them is what made the earlier analysis wobble:

- **Which protocol** — strings or tokens. Decided by §5.3: tokens, because spans
  make nested expansion need no map chain at all. **Hygiene is not the reason,
  and does not have to be.**
- **Which hygiene mechanism** — gensym, or contexts on spans. Answerable either
  way, ON TOP of tokens.

#### Decided: **(c) for the protocol, (b) for the hygiene**

**Tokens**, for §5.3's reason, which stands whatever hygiene mechanism is
chosen. **And a `TokenStream.gensym()` primitive** for hygiene, rather than
`def_site` contexts, for two reasons:

1. **`def_site` needs the engine's binding resolution to change.** Contexts on
   spans mean identifier EQUALITY becomes context-sensitive: every binding
   carries a tag and every lookup compares it. For undecorated code the tags are
   all equal and the behaviour degenerates to today's — but the comparison is in
   the hottest path in the engine, and adding a field to every binding is a
   pervasive change to pay for a feature most modules never use.
2. **The minting mechanism was MISSING.** §4.2's `js` tag function is ordinary
   userland code, and nothing gave it a way to mark a token it created as
   `def_site`. Rust does not have this problem because `quote!` is itself a proc
   macro with privileged access to `Span::def_site()`; a JavaScript tag function
   has no equivalent. **A mechanism with no way to invoke it is not a
   mechanism**, and a draft specified the policy without noticing the gap.

`gensym()` has neither problem: it mints an identifier that is unique by
construction, so binding resolution is untouched and any userland helper can
call it.

**What is given up** is referential transparency — a macro emitting `Date.now()`
still meets a user's shadowing `Date`. But `def_site` does not solve that either
(it governs identifiers a macro CREATES, not ones it references), and neither
does Rust's `call_site`, which is what its ecosystem overwhelmingly uses. **The
coverage is the same; the implementation cost is not.**

**Ruled out:** (a), because a silent wrong binding is the worst outcome any
option produces. (d), because safe renaming requires lexing the output. And
`def_site`-by-default, on the two grounds above — with the note that **spans
still make capture DETECTABLE even when it is not prevented**: a token's span
says whether an identifier came from the user or the macro, so a linter or a
warning can find what gensym discipline missed. That is a middle path strings
could not have offered.

### 7.2 Distinguishing runtime from replacement decorators — SETTLED

The decorators already specified are RUNTIME decorators. **(measured)** a field
decorator runs before the declaring class's layout is computed, and
`addInitializer` fires at class-definition time. A replacement decorator runs
before the module executes. `@foo` would mean both.

#### (a) A global registry, matched by name

The original sketch: preprocessors register to global scope, and expansion
name-matches.

**Costs:** the host module has not run, so there is no scope — `@foo` cannot be
RESOLVED, only name-matched. A runtime decorator named `validate` and a
replacement named `validate` collide, and **the collision is decided before
scoping exists**, so nothing the developer writes can disambiguate it.

#### (b) A distinct sigil

A separate mark for replacement decorators.

**Solves:** the ambiguity, lexically and completely, at the earliest possible
point.

**Costs:** syntax budget, which is scarce, and a second thing to learn for a
distinction that the import statement already records. It also splits the
decorator surface in two for users who will reasonably think of both as
"decorators."

#### (c) Resolution through the static import binding

`import { expand } from './m.js' with { preprocessor: true }` makes `expand`
statically a replacement decorator; every other `@foo` is a runtime decorator.

**Solves:** the ambiguity, using information the program already states. The
import says what the module is FOR; the decorator does not have to repeat it.

**(measured)** the grammar already accommodates it — `@ns.f`, `@ns.sub.f` and
`@f()` all parse today, so `@expand.derive` needs no grammar change.

**Fit:** this is how the proposal already treats modules. README: "Typed
function and class exports carry their full signatures in the module record",
and `with { type: 'json' }` already conditions a binding's meaning on an import
attribute. Conditioning a decorator's PHASE on one is the same move.

**Costs, and this was understated in an earlier draft.** Deciding `@foo` by
scope means resolving a binding, and **resolution during expansion is
CIRCULAR**: a macro may introduce declarations, including shadowing ones, so the
scope to resolve against is not final until expansion finishes — which is the
thing being decided. A pre-parser tracking declarations does not break the
cycle; nothing does, while the rule is a scope query.

#### (c') The STRICT LEXICAL RULE

Take (c)'s idea — the import decides the phase — and make it **syntactic rather
than scoped**: a replacement decorator must be spelled with the exact identifier
the `preprocessor: true` import introduced. No aliasing, no local rebinding.
**A local declaration that shadows the name is a SyntaxError at expansion**, not
a silent reversion to runtime.

**Solves:** the circularity, completely. The set of replacement names is read
off the import statements at the top of the module — a syntactic scan, decided
before expansion begins and unaffected by anything expansion does.

**Costs:** ergonomics, narrowly. `import { derive } … ; const d = derive;` then
`@d` is refused, and so is any wrapper. That is a real restriction, and it is
the price of the rule being decidable.

**Fit:** it keeps §7.3's claim honest. Scope tracking would have pushed the
pre-parser toward the analysis §7.3 says it does not do; a syntactic scan does
not.

#### Decided: **(c')**, the Strict Lexical Rule

It needs no new syntax, no grammar change **(measured)**, and it reads the way a
developer would expect: the import that says `preprocessor: true` is what makes
its decorators preprocessors. The strictness is what makes it decidable.

**Ruled out:** (a), because the ambiguity is decided where no scope exists. (b),
the sigil, because it spends syntax budget to state what the import already
states — **but it remains the fallback**, and it is strictly cheaper for
implementers, so if the no-aliasing restriction proves too sharp in practice the
sigil is where to go. (c) unqualified, because scope-based resolution is
circular during expansion and no amount of pre-parser capability fixes that.

---

### 7.3 Boundary detection — SETTLED

An earlier draft claimed the engine "only lexes" during expansion. That is not
achievable: `/` is division or regex-start depending on the parse, and ASI needs
parse context. **The defensible claim is that the engine does not build its AST
during expansion**, which is true and is all the performance argument needs.

What it does is **pre-parse** — the pass engines already run for lazy
compilation, which skips function bodies while matching delimiters. Boundary
detection rides on it, and so does §4.2's grouping: the delimiter matching that
finds where `@g { … }` ends is the same matching that produces the `group`
token.

**Adopting tokens shrinks the rest of this question to nothing.** With strings,
each splice invalidated the region's lexing and the re-scan strategy mattered —
full re-scan being quadratic in the number of expansions. With tokens, spliced
output IS tokens; there is nothing to re-lex. The engine walks the returned
stream for nested decorators, which is a traversal of a structure it was handed,
not a re-derivation of one.

**Decided:** state the claim accurately — no AST during expansion, a pre-parse to
find boundaries — and note that re-scan cost disappears with §4.2 rather than
needing a strategy.

### 7.4 Content Security Policy and capability — SETTLED

Phase-locking constrains WHEN code appears, not WHAT it is. A preprocessor is
arbitrary JavaScript at load time and can compute its output from anything,
including a `fetch`.

**Tokens do not change this, and it is worth saying so.** A reader may assume a
structured stream is safer than a string; it is not. Tokens constrain how a
program is EXPRESSED, not which programs are expressible, so a preprocessor can
emit anything it could have emitted as text. The security question is identical
under either protocol.

#### (a) Argue it is not `eval`

The original sketch's position: the output inherits the host module's origin and
strict-mode context, and cannot execute arbitrary runtime strings.

**The first half is true and the second is a non-answer.** It cannot execute
arbitrary RUNTIME strings; it executes arbitrary PARSE-TIME ones. An argument
that will not survive review is worse than an acknowledged cost, and this one
will not: the whole point of `unsafe-eval` is that code whose text is computed
is governed separately from code whose text is fetched.

#### (b) Rely on existing `script-src`

A preprocessor is a module, so it is already fetched under `script-src`.

**Solves:** the origin question for the preprocessor itself — genuinely, and
this is worth stating, because it means a preprocessor cannot come from an
unapproved origin.

**Does not solve:** its OUTPUT, which is unbounded and never fetched.

#### (c) Require `unsafe-eval`

Treat expansion as what it resembles.

**Costs:** `unsafe-eval` is a large grant — it re-enables runtime `eval` and
`Function` for the whole document. Requiring it for a build-time-shaped feature
pushes deployments toward a weaker policy overall, which makes the security
posture WORSE than not having the feature.

#### (d) A distinct directive

Expansion is governed by its own directive, so it can be granted without
granting runtime `eval`.

**Solves:** the actual risk, at the actual granularity. Composes with (b): the
preprocessor's origin is governed by `script-src`, its expansion by the new
directive.

**Costs:** a new directive is a specification and an implementation in every
browser, and it must be defined to fail closed.

#### (e) A purity constraint on the expansion context

The directives above govern WHAT CODE RUNS. None of them governs **what that
code can reach**, and that is the larger hole: if a preprocessor runs on the
main thread with the ambient globals, a malicious dependency can read the
module's source — which it is handed — and exfiltrate it over the network during
parsing. A CSP directive that permits expansion permits that too.

**Expansion must be a PURE FUNCTION OF ITS TOKEN INPUT.** That is the property.
**The MECHANISM is compile-time evaluability, which the proposal already has** —
not a runtime sandbox, which an earlier draft specified before checking whether
the proposal had an answer.

[typeprogramming.md](typeprogramming.md), on the discipline `where` clauses,
metadata annotations and type builders already use:

> a function is evaluable when its body reads only its parameters, constants, and
> other evaluable functions. This is a static purity discipline rather than a
> runtime sandbox — an evaluable function *cannot name* ambient mutable state,
> I/O, or nondeterminism, so there is nothing to escape from. It is the
> proposal's answer to "running user JS in the compiler," and it is stricter and
> cheaper than a jail.

and, on builders specifically:

> Evaluability rules out `Date.now`, `Math.random`, I/O, and ambient reads by
> construction, so builds are deterministic and the "sandbox" is a property of
> what the code can name rather than a wall around what it does.

**That is this section's requirement, already specified, with the same
exclusions and the same justification.** A replacement decorator is required to
be compile-time evaluable, and everything below follows without new machinery:

- **The failure is STATIC.** A macro that names `Date` is rejected where it is
  written, not when it is called. An earlier draft's sandbox would have failed at
  expansion, far from the mistake.
- **Local mutation stays legal** — typeprogramming.md is explicit that a `Set` of
  seen keys or an accumulator is fine and only *shared module-level* mutable
  state is not — so a macro can still compute.
- **One concept, not two.** Type builders and replacement decorators are both
  user JavaScript the compiler runs, and they get one discipline.

What follows is the capability boundary that discipline produces, retained
because it is worth stating what falls outside it:

- **Nothing host-defined.** No DOM, no `fetch`, no timers, no file access, no
  module loading. These observe or affect state outside the input.
- **ECMA-262 MINUS the parts that observe state the input does not contain.**
  An earlier draft said "ECMA-262 and nothing host-defined" and stopped there.
  **That rule is insufficient**, and the gap is not small: `Date` and `Math` are
  262.

**(measured)** the nondeterministic parts of the standard library that
evaluability excludes — an evaluable function cannot name any of them:

| excluded | why |
| --- | --- |
| `Date`, `Temporal.Now` | the wall clock. `Date.now()` makes the same source expand differently on two runs |
| `Math.random` | the obvious one |
| `WeakRef`, `FinalizationRegistry` | make garbage collection observable |
| `SharedArrayBuffer`, `Atomics` | cross-agent state |
| `Intl`, and every `toLocale*` method | the host's locale data, so the same source expands differently on two MACHINES |

**A macro cannot READ the clock; it can EMIT code that reads it.** Nothing here
stops a macro generating `Date.now()` — that code runs later, in the user's
realm, with the user's `Date`. What is forbidden is the macro consulting the
clock while deciding what to generate.

What remains is enough to compute with: `Object`, `Array`, `Map`, `Set`,
`RegExp`, `JSON`, `String`, `Number`, `BigInt`, `Math`'s deterministic members,
and the `TokenStream` facilities of §4.2.

**And it is checkable statically, which a sandbox would not have been.**
Determinism can still be tested by expanding twice and comparing, but the point
of evaluability is that the check happens before anything runs — the property is
enforced by what a macro can NAME rather than by what a realm withholds.

**And the strongest argument for it is not security, it is DETERMINISM.** If a
preprocessor can `fetch`, the same source can expand differently on two loads,
so **the expanded output cannot be cached** — which forfeits code caching for
every module that uses a macro, in a design whose §4.4 phase split exists to
protect compilation performance. A sandbox makes expansion a pure function of
its input, so the expansion can be cached with the compiled code, and a build
tool can perform it ahead of time and get the same answer the engine would.

It also simplifies two things elsewhere: §4.4 needs no blocking for the
expansion step itself, only for the preprocessor module's own load, and §5.4's
worry about expansion depth becoming a network-round-trip count cannot arise.

#### Decided: **(b) + (d) + (e)**

The preprocessor module is governed by `script-src` like any module; expansion
needs its own grant because its output is computed rather than fetched; and
and expansion is constrained by compile-time evaluability, which makes it
synchronous and IO-free by what it can name.

**Ruled out:** (a), because the argument is wrong and its wrongness is the kind
reviewers find first. (c), because granting `unsafe-eval` to get expansion
leaves the document strictly less safe than before, which inverts the goal.
**(b)+(d) without (e)**, because policing the origin of code that is handed the
module's source and given a network is not a policy at all.

---

### 7.5 Erasability — the question does not apply

An earlier draft listed this as open, on the reasoning that type syntax is meant
to be erasable and a preprocessor seeing annotations breaks that.

**That was imported from TypeScript and is wrong for this proposal.** README's
rationale:

> The explicit goal of this proposal is not just to give developers static type
> checking. It's to offer information to engines to use native types and
> optimize callstacks and memory usage.

Types here are load-bearing at runtime by design — layout, native representation,
`Reflect.typeOf`, `is` tests, the whole reflection surface. **Erasability was
never a property to lose.** The word "erasure" appears once in README, describing
what OTHER languages do to `protected` and this one does not.

**The residual question is smaller and real:** a preprocessor runs before
checking, so it sees annotations AS WRITTEN — `uint8`, an unresolved alias, a
generic parameter — not as resolved types. That is a genuine ergonomic limit and
belongs in §7.6.

---

### 7.6 Ordering against the type checker — SETTLED

#### (a) Check, then expand

**Solves:** a macro could see resolved types.

**Ruled out as unsound:** generated code would never be checked. A macro could
emit anything.

#### (b) Check, expand, re-check

**Costs:** double the checking, and worse — **the first check runs on syntax the
macro was going to fix**, so a program that expands to something valid can fail
before expansion. That makes macros unable to introduce syntax the checker does
not yet accept, which is much of what macros are for.

#### (c) Expand, then check

Generated code is checked exactly as written code is. Errors inside generated
code resolve through the spans its tokens carry (§5.3).

**Costs:** a preprocessor cannot see types. It sees tokens, and must parse
annotations itself if it wants them — `uint8` arrives as an identifier token, not
as a resolved type.

**This matches Rust exactly and is worth stating as precedent rather than
apology**: proc macros do not see types either — they run on tokens, and type
information exists only later in the compiler. `serde`'s derive works entirely
on written syntax. The limit is real, understood, and has not prevented the most
successful macro ecosystem in current use.

#### What the loss actually is — SETTLED, and it is not what it sounded like

"A macro cannot see types" is the wrong statement. **A macro sees every
annotation exactly as written; what it cannot do is RESOLVE A NAME.** Written
`temp: float64`, a macro reads the token `float64` and can branch on it. The
limit bites only through indirection — an alias, a generic parameter, an imported
type.

The case that motivates the worry:

```js
type Celsius = float64;
type Name = string;

@derive class Reading {
	label: Name = "";
	temp: Celsius = 0;
}
```

A serializer wants numbers unquoted and strings quoted. It sees `Name` and
`Celsius` — two identifiers, indistinguishable.

**How could a macro learn the resolved type?** Two ways, and both fail:

- **Ask the checker mid-expansion.** Resolving `Celsius` needs the module's type
  environment, which is not complete until expansion finishes — a macro can emit
  `type` declarations. **This is §7.2's circularity again, in the type
  namespace**, and it fails for the same reason: the thing being asked depends
  on the thing asking.
- **Expand, check, re-expand with types.** The macro runs TWICE: it emits
  something provisional, the checker resolves `Celsius`, and the macro runs again
  knowing `temp` is a `float64`. A fixpoint over the type system — the second
  expansion can declare types that change what the first check computed, so it is
  not clear the loop terminates or that its answer is unique. §7.6(b) ruled this
  out and Rust rejected it for the same reason.

**Deferral is not a third answer to that question — it discards the question.**
The macro never learns the type, because it never branches on one. It emits code
whose meaning a LATER stage resolves, and runs exactly once.

|  | re-expansion | deferral |
| --- | --- | --- |
| macro runs | twice, or until a fixpoint | once |
| macro branches on the type | yes, later | **no, never** |
| what resolves it | the macro, on its second pass | overload resolution, or a runtime test |
| new machinery | a loop over expansion and checking | **none — both mechanisms exist** |

The test for telling them apart: **does the macro ever need to know the answer?**
Under re-expansion it does, which is why the loop is needed. Under deferral it
does not, which is why nothing is.

#### Deferral in full, and its targets already exist

Concretely, for the `Reading` class above: the macro emits `ser(this.label)` and
`ser(this.temp)` — **the same code for both fields** — and the difference is
resolved after expansion. **(measured)** overload resolution sees through an
alias:

```js
function ser(x: float64) { return "num:" + x; }
function ser(x: string)  { return "str:" + x; }

type Celsius = float64;
class R { temp: Celsius = 20; }

ser(new R().temp);        // "num:20" — the alias resolved, the overload picked
```

So the macro emits `ser(this.temp)` for every field and **does not branch at
all**. Overload resolution runs after expansion, when `Celsius` resolves, and
picks what the macro could not. Where static dispatch is not enough,
**(measured)** `Reflect.typeOf(this.temp) === (type float64)` is *true* through
the same alias, so the emitted code can branch at runtime.

**This proposal is better placed than Rust here**, which is worth saying because
the limitation was inherited from Rust's framing: Rust has trait resolution and
nothing else, while this has overloading AND runtime type reflection. Two
deferral targets, not one.

#### Decided: DEFERRAL is the proposal's answer

A macro that would branch on a type emits code that names it and lets a later
stage dispatch. **This is the endorsed pattern, not merely an available one**,
and it is what makes §7.6(c) affordable — expand-then-check costs little
precisely because the branch a macro cannot make is one it does not have to.

Three things follow, and the second and third are the price.

**It creates a DEPENDENCY on the rest of the proposal.** Deferral works because
overload resolution and `Reflect.typeOf` both see through an alias. If either
changed, this answer would weaken — so they are load-bearing for macros in a way
they were not before.

**The helper must be in scope at the splice site, and the macro cannot put it
there.** An emitted `ser(this.temp)` needs `ser` visible where the splice lands,
and a macro decorating a class member emits into a class body — where an
`import` cannot go. **So the USER imports the helper**, exactly as Rust requires
`use serde::Serialize`. That is a real ergonomic requirement and it should be
documented per macro, because the failure otherwise is an unresolved name in
code the developer did not write.

**And it is exposed to the one hygiene gap `gensym()` does not close.**
**(measured)**:

```js
function ser(x: float64) { … }   function ser(x: string) { … }

function scope() {
	const ser = () => "LOCAL";
	return ser(1.5);              // "LOCAL" — the local wins
}
```

A macro emitting `ser(…)` into a scope where the user has their own `ser` gets
the user's. `gensym()` mints identifiers a macro CREATES; it cannot protect a
reference to one it did not. §7.1 named this residue as the thing `def_site`
would not have fixed either — **deferral is where it actually bites**, and
naming the instance is better than leaving the residue abstract.

#### No `resolveType()` on the token stream

Stated as a negative decision because it will be proposed: **the token stream
exposes no way to ask what a type name resolves to.** Any such API is the
"ask the checker mid-expansion" option under another name, and carries the same
circularity — a macro can emit `type` declarations, so the environment it would
query is one it is still constructing.

#### A THIRD deferral target: `Reflect.Type` in a runtime decorator

[typeprogramming.md](typeprogramming.md) records that
`Reflect.getReflection.<Reflect.Type>(t)` cracks a type object into a node
discriminated by `kind`. **A runtime decorator's context carries a type object,
so it has this in full** — **(measured)**:

```js
type C = float64;
function g(c) {
	c.type === (type float64);                                    // true — the alias resolved
	Reflect.getReflection.<Reflect.Type>(c.type).kind;            // "primitive"
}
class A { @g a: C = 1; }
```

Union arms and object properties read back the same way — `union:2`,
`object:2`, measured. **This is precisely the "branch on a resolved type"
capability §7.6 was said to lose**, and it exists, fully, including through
aliases.

**So the deferral target for a structural decision is a runtime decorator.** A
macro that wants to branch on a type emits `@addSum`, and `addSum` reflects the
class's type at definition time and does the work. The macro defers not to a
dispatch but to *another decorator in a later phase*.

#### What genuinely does not work — narrower than stated

Not "generated structure that depends on a resolved type" — that works, by the
route above. **What fails is STATICALLY VISIBLE structure that depends on a
resolved type.**

A runtime decorator can add a `sum()` method to the object; it cannot make the
CHECKER know about it, because decorators.md constrains a class decorator's
return to `T` — the class or a subclass — so the declared type is what the
checker sees. So `p.sum()` runs and does not type-check.

**Still no accommodation**, and the reason is unchanged: closing that gap needs
the macro to resolve a type name during expansion, which is circular. But the
residue is now one sentence rather than a category, and everything outside it has
a route.

#### Decided: **(c)**, normatively

Expansion precedes checking, and the ordering is specified rather than left to
implementations, since a program's validity depends on it. **Ruled out:** (a) as
unsound, (b) because it forbids the macros worth writing.

Two consequences are now fixed rather than incidental: **the checker never sees
an unexpanded decoration** (§4.4), and **a replacement decorator receives tokens
and no context object** (§4.2), since every field of a context needs either a
resolved type or a runtime.

---

### 7.7 What can be replaced — two tables, not one — SETTLED

decorators.md already has a replacement table, and it excludes blocks with a
stated reason:

> a block produces nothing, so there was nothing to replace. A `do` block
> produces a value and a `do *` block produces a generator, which is why those
> two rows exist and the others do not.

**That reasoning is entirely about VALUE replacement, and it does not transfer.**
Every row of the existing table replaces a runtime value — a class, a field's
initial value, a getter, a `do` expression's result — and is typed by the
original's type. **Syntax replacement is a different axis.** A block produces no
value, but it has syntax.

#### (a) Extend the existing table with block rows

**Ruled out:** it conflates the two axes. A reader would then have to work out
which rows mean "returns a value at runtime" and which mean "returns tokens at
parse time", and the `Return type` column means nothing for the latter.

#### (b) Restrict syntax replacement to positions that can value-replace

**Ruled out as needlessly restrictive:** parameters, returns, enums, tuples,
records, `let` and `const` all have syntax. Excluding them removes most of what a
macro would want — a derive over an enum, a parameter attribute — for a reason
that applies only to values.

#### (c) A second table, for syntax replacement

Every decorable position can be syntax-replaced, including all the ones the value
table excludes.

**The constraint is different in kind**, and this is the point: value
replacement is constrained by TYPE — decorators.md: "The return type must be
compatible with the original." Syntax replacement is constrained by GRAMMAR — the returned
tokens must parse in the position they replace. A class-position macro must return
something that is a class declaration there; a parameter-position macro must
return something that is a parameter.

**That constraint is checkable and should be specified**, because the failure
otherwise surfaces as a syntax error at a location the developer did not write.

#### The two tables OVERLAP in exactly two rows

`DoBlock` and `DoGeneratorBlock` are value-replaceable AND carry an `Expression`
field, so they appear in both tables. **That is only unambiguous because of
§7.2**: a runtime decorator's return replaces the `do` expression's VALUE, and a
replacement decorator's return — a token stream — replaces its SYNTAX. The phase
rule is what tells them apart. Tokens make the two returns different TYPES as
well as different meanings, which helps, but does not decide by itself: a runtime
decorator may legitimately return an array of objects.

**This is the strongest argument for §7.2's rule** — the two decorator kinds do
not merely run at different times, they give a bare `return` at one position two
incompatible meanings, and the import name is what disambiguates.

#### May a replacement be EMPTY? — SETTLED: wherever it parses

Conditional compilation is a core macro use — Rust's `#[cfg]` removes what it
decorates — and it needs a replacement that returns nothing.

**No separate permission is needed.** The rule is already "must parse in the
position it replaces", and an empty stream parses in some positions and not
others: as a statement or a class member, yes; as a parameter in the middle of a
list, or as an operand, no. So `@cfg` works on a block, a method or a field, and
returning nothing for a parameter is the same error as returning a class
declaration for one.

This is the constraint doing its job rather than an exception to it, which is why
it needed no rule: **a grammar constraint answers "may it be empty" for free,
where a type constraint would not have.**

#### Decided: **(c)**

A separate table, constrained by grammar rather than by type. **Ruled out:** (a),
which conflates two different constraints under one heading; (b), which imports a
restriction whose reason does not apply.

#### The syntax replacement table


**The table lives in [decorators.md](decorators.md), and only there.** A copy
here is a second thing that must agree with it forever, which this document
names as the shape the project is bitten by most — and it had already drifted:
the copy that stood here lacked the `kind` column and the `Reflect.Region` row
that decorators.md gained, so a reader consulting whichever they found first got
different answers. What matters HERE is the argument for a second table at all,
which the rows below make and the enumeration does not:

| Position | Replacement must parse as | Also value-replaceable |
|---|---|---|
| `Reflect.ClassMethodParameter` | a formal parameter | **no** |
| `Reflect.ClassMethodReturn` | a type annotation | **no** |
| `Reflect.Block` and the eleven other block forms | the statement form it decorates | only `DoBlock`, `DoGeneratorBlock` |
| `Reflect.Enum` / `Tuple` / `Record` | the corresponding declaration | **no** |
| `Reflect.Let` / `Const` | a lexical declaration | **no** |

**The "no" rows are the point of having a second table.** Every one of them can
be syntax-replaced and none can be value-replaced, so restricting to the value
table's positions would have removed a parameter attribute, a derive over an
enum, and conditional compilation of a block — most of what macros are for.

**Two rows are in both tables**, and §7.2's phase rule is what keeps a return at
those positions unambiguous.

**The failure is a parse error attributed to the macro.** Returning a class
declaration for a parameter position fails at the splice, and the span says which
macro produced it — which is the whole reason the constraint is worth stating
rather than leaving to whatever the parser happens to do.

### 7.8 How a replacement decorator is INVOKED — SETTLED

Every question above concerns what a replacement decorator RECEIVES and RETURNS.
None concerned how it is CALLED, and decorators.md has detailed rules there,
written for runtime decorators. Four sub-questions, analysed in the order they
depend on each other.

---

#### (i) What are a decorator's ARGUMENTS?

decorators.md: "**Decorator expressions are evaluated in document order** — left
to right, top to bottom … Whatever `@f(BASE + '/x')` computes, it computes at the
position where it is written."

**A replacement decorator runs before the module is parsed, so `BASE` does not
exist.** And `@derive(Serialize)` is the canonical macro form.

**(a) Literals only, evaluated by the engine.** Simple, and **ruled out on the
canonical case**: `Serialize` is an identifier, not a literal, so the form the
whole feature exists to serve would be illegal.

**(b) Evaluated in the PREPROCESSOR's scope.** `@derive(Serialize)` resolves
`Serialize` where the macro lives. **Ruled out as inconsistent with §7.2**: the
Strict Lexical Rule insists the decorator NAME comes from the host's import
clause, precisely so the host can see what it is naming. Taking the arguments
from a scope the host cannot see reverses that in the same expression.

**(c) TOKENS, unevaluated.** `@derive(Serialize)` passes the identifier token
`Serialize`; the macro interprets it. **Recommended**, on three grounds:

- **Uniform.** A macro receives tokens for what it decorates; receiving tokens
  for its arguments is the same rule, not a second one.
- **It needs no evaluation, so it raises none of §7.6's problems.** There is no
  scope to resolve in, and nothing to be circular about.
- **It is strictly more expressive than (a).** A macro that wants the number `0`
  from `@f(0)` reads it off a numeric token; a macro that wants `Serialize`
  cannot get it any other way.

**This requires an explicit carve-out from decorators.md**, and it should be
written as one: *decorator expressions are evaluated in document order — except
for replacement decorators, whose arguments are tokens and are never evaluated.*
The alternative is a rule that silently means two things.

---

#### (ii) STACKED order — the conflict dissolves

The apparent problem: **(measured)** runtime decorators apply in reverse source
order — `@a @b @d m()` runs `d`, `b`, `a` — while §5.2 settled outside-in, which
for `@a @b class C` gives `a` first. Opposite orders on identical syntax.

**decorators.md gives its own reason, and the reason is the resolution.** Rule 1
says the order is "Python's `a(b(c(x)))`" — **function composition over a
value**. `c` must produce a value before `b` can consume it, so inner-first is
forced by the medium.

Replacement is not composition over a value; it is rewriting over syntax. **`a`
must rewrite before `b` is consumed**, because once `b` has run, the syntax `a`
was going to rewrite is gone. Outer-first is forced by ITS medium.

**Both rules say the same thing about NESTING — `a` is outside `b`.** They differ
in execution order because a value must exist before it is wrapped, and syntax
must be rewritten before it is consumed. That is one principle in two media, not
two principles.

The alternatives confirm it:

- **Replacement inner-first** would mean an outer macro cannot delete an inner
  one, because the inner already ran. **That kills `@cfg`** — the outermost
  decorator removing a construct including everything decorating it — which
  §7.7 settled as a use the design supports.
- **Runtime outer-first** would give each decorator the undecorated original and
  keep only the last return. It breaks composition outright.

**Recommended: keep both rules, and state the principle they share** rather than
leaving a reader to find two orders and no explanation. **(recalled)** Rust
reaches the same arrangement — an attribute macro receives the inner attributes
unexpanded and may remove them.

---

#### (iii) MIXED stacks

`@expand.derive @runtime.log class C {}` — the replacement runs at parse time and
the runtime one at definition time, so the replacement runs first **whatever the
source order says**.

**Whether a replacement may rewrite a runtime decoration is not a new question.**
A replacement returns tokens that replace the construct; if the returned tokens
do not contain `@runtime.log`, there is no `@runtime.log`. Requiring the engine
to re-insert decorations the macro omitted would contradict what "replace" means.
**So it follows, as the empty-replacement answer did, and needs no rule.**

The real question is placement, and here doing nothing is the trap:

**(a) Allow any order, and say the relative position of the two kinds carries no
meaning.** **Ruled out.** `@runtime.log @expand.derive class C` reads as `log`
wrapping `derive` and means nothing of the sort. A source order that looks
significant and is not is exactly the silent failure this document has rejected
throughout.

**(b) Require replacement decorators INNERMOST**, closest to the declaration.
**Ruled out on capability**: a replacement's input is what it encloses, so
innermost replacements cannot see the runtime decorations outside them — and a
`@cfg` that removes a member could not remove that member's runtime decorations,
leaving them attached to nothing.

**(c) Require replacement decorators OUTERMOST.** **Recommended.** The
replacement encloses the runtime decorations, so it can see them, remove them
with the construct, or copy them onto generated members. Source order then agrees
with execution order between the kinds, and the arrangement is checkable
syntactically — §7.2's name set is known before expansion, so a misplaced
replacement decorator is a SyntaxError rather than a surprise.

**One documented hazard remains, and it is not a language rule**: a macro that
rebuilds a construct from scratch will drop any decoration it did not copy
forward. That is a lint and a documentation matter, in the same way a macro that
forgets `gensym()` is.

---

#### (iv) OVERLOAD selection

decorators.md resolves `@f(0)` against `@f('a')` by overload resolution, and
§7.6 says types do not exist during expansion.

**(a) Select by arity.** **Ruled out as machinery for a problem that does not
exist.** A macro receives its arguments as tokens and can count them itself.

**(b) Select by token kind.** **Ruled out for the same reason, and it is worse**:
a crude structural match that looks like overload resolution and is not would
mislead by resemblance.

**(c) No overloading ON ARGUMENTS — one function per name per POSITION.**
**Recommended.** With arguments as tokens (i), every argument has the same type,
so type-directed selection has nothing to select on. The macro branches on its
own arguments, which is where a token-level decision belongs.

**This rules out overloading on ARGUMENTS and not on the CONTEXT**, and the
distinction matters. A decoration's position is decided by lookahead at the
decoration site — before any checking, without reading the region — so selecting
on the context type is decidable exactly when the position is, which is always:

```js
export function jsx(tokens: TokenStream, context: Reflect.Region): TokenStream;
export function jsx(tokens: TokenStream, context: Reflect.ClassMethod): TokenStream;
```

Two functions chosen by position, neither branching. A union of contexts and a
`switch` on `kind` remains available and is what decorators.md already describes
for a decorator "whose parameter is a union of contexts" — the same choice a
runtime decorator author has.

This is a second carve-out from decorators.md, and a smaller one: its overload
rule and its ambiguity TypeError are runtime-decorator rules, and §7.2 already
separates the two populations by name.

---

---

#### (v) BLOCK decorators run per ENTRY — a third carve-out

decorators.md rule 5:

> A BLOCK decorator runs on every ENTRY to the block, not once at the
> declaration. A block inside a loop is evaluated each iteration, so its
> decorator runs each iteration … It is also the ONE per-evaluation position in
> this extension … a block decorator in a hot loop costs a context and a call per
> iteration.

**A replacement block decorator runs ONCE, at parse time.** It rewrites syntax;
there is no per-entry anything to run. So this is a third carve-out from
decorators.md, alongside argument evaluation and overload selection.

**And it is the one carve-out that is a gain rather than a cost.** Rule 5's
warning — a context and a call per iteration, invisible at the declaration site —
does not apply to the replacement form at all, and the capability is not lost:
a replacement block decorator can EMIT the instrumentation inline, so the work
happens per entry while the decorator does not.

**That is a motivation for the feature this document had not stated.** A user who
wants per-iteration tracing without per-iteration overhead writes a replacement
decorator, and rule 5's cost disappears rather than being tolerated.

---

#### (vi) A macro that REJECTS its input

`@derive` applied to something that is not a class must be able to say so, and
nothing above says how.

**(a) Return a diagnostic record** alongside or instead of tokens. **Ruled out**:
it gives the protocol a second return shape for a case JavaScript already has a
mechanism for, and a macro that forgets to check the record's presence silently
succeeds.

**(b) THROW.** The exception is caught at the splice and becomes an early error,
carrying the macro's message and the span of what it was given.
**Recommended** — it is what a JavaScript function does to reject its arguments,
it cannot be ignored, and §4.2's spans already supply the location.

**Decided: (b).** An abrupt completion from a replacement decorator is a
SyntaxError attributed to the DECORATION SITE, with the macro's message. Rust
reaches the same place by a different route: `compile_error!` expands to a
diagnostic because a proc macro cannot throw, where a JavaScript macro can.

**One consequence worth stating**: this is why §7.4's evaluability matters for
error quality as well as determinism. A macro that fails because it named `Date`
fails STATICALLY, at its own definition — not as a thrown error at some
decoration site far from the mistake.

#### Summary of §7.8

| | |
| --- | --- |
| Arguments are TOKENS, never evaluated | carve-out from decorators.md |
| Stacking: outer-first for replacement, inner-first for runtime | one principle, two media |
| Replacement decorators must be written OUTERMOST | new rule, syntactically checkable |
| A replacement may rewrite runtime decorations it encloses | follows from "replace" |
| No overloading for replacement decorator names | carve-out from decorators.md |
| Block decorators are per-ENTRY at runtime, per-PARSE for replacement | carve-out from decorators.md |
| A macro rejects its input by THROWING | early error at the decoration site |

## 8. Summary

**Part A — reflection**

| | |
| --- | --- |
| `Expression` becomes a `TokenStream` on the reflections | **settled** — §1, §3 |
| The block family is twelve types; `initial` is a separate gap | **settled** — §1, §3 |
| **No `source: string` field — source text is span-derived** | **settled** — §3.1 |
| `TokenStream.toString()` covers what a `source` field was for | **settled** — §3.1 |
| Reading is sound without writing; lands independently | **settled** — §2 |

**Part B — protocol**

| | |
| --- | --- |
| **Exchanges TOKEN STREAMS, not strings** | **settled** — §1, §7.1 |
| Delimited runs are grouped, not flat | **settled** — §4.2 |
| Hygiene contexts belong to Part B only | **settled** — §3.2 |
| `regexp` and `punctuator` are distinct kinds, for the `/` ambiguity | **settled** — §4.3 |
| A region the engine cannot lex reaches a macro via a declared lexical MODE | **settled** — §4.3 |
| Hygiene from `TokenStream.gensym()`, not `def_site` contexts | **settled** — §7.1 |
| Protocol and hygiene are independent axes | **settled** — §7.1 |
| Phase decided by the import name, STRICTLY — no aliasing | **settled** — §7.2 |
| Preprocessor imports use NAMED bindings, not bare | **settled** — §4.1 |

**Expansion**

| | |
| --- | --- |
| Macro output may contain macros | **settled** — §5 |
| Order is outside-in | **settled** — §5.2 |
| Spans compose themselves; there is no map protocol | **settled** — §5.3 |
| New macro names cannot appear mid-expansion — now a CONSEQUENCE | **settled** — §5.4 |
| Pre-parse for boundaries; nothing is re-lexed | **settled** — §7.3 |

**Remaining answers**

| | |
| --- | --- |
| CSP: `script-src`, a distinct directive, AND a sandbox | **settled** — §7.4 |
| Purity via COMPILE-TIME EVALUABILITY, the proposal's own discipline | **settled** — §7.4 |
| Excludes the clock, randomness, GC observation, locale — by NAMING | **settled** — §7.4 |
| A macro may EMIT clock-reading code; it may not READ the clock | **settled** — §7.4 |
| Expansion is synchronous, IO-free, and therefore cacheable | **settled** — §7.4 |
| Evaluability binds preprocessor EVALUATION too, not just expansion | **settled** — §4.4 |
| Erasability | **does not apply** — §7.5 |
| Expand, then check | **settled** — §7.6 |
| The checker never sees an unexpanded decoration | **settled** — §4.4 |
| A replacement decorator receives tokens and NO context | **settled** — §4.2 |
| A second replacement table, constrained by grammar | **settled** — §7.7 |
| An empty replacement is legal wherever empty parses | **settled** — §7.7 |
| **Deferral is the endorsed pattern for type-dependent macros** | **settled** — §7.6 |
| Deferral depends on overloading and `typeOf` seeing through aliases | **settled** — §7.6 |
| A third deferral target: `Reflect.Type` in a runtime decorator | **settled** — §7.6 |
| The residue is STATICALLY VISIBLE structure only | **settled** — §7.6 |
| The USER imports a macro's helper; the macro cannot | **settled** — §7.6 |
| No `resolveType()` on the token stream | **settled** — §7.6 |
| No accommodation for branching on a resolved type | **settled** — §7.6 |
| Decorator ARGUMENTS are tokens, never evaluated | **settled** — §7.8 |
| Replacement stacks outer-first; runtime stacks inner-first | **settled** — §7.8 |
| Replacement decorators are written OUTERMOST | **settled** — §7.8 |
| A replacement may rewrite runtime decorations it encloses | **settled** — §7.8 |
| No overloading of replacement decorator names | **settled** — §7.8 |
| Block replacement runs once at parse, not per entry | **settled** — §7.8 |
| A macro rejects its input by throwing | **settled** — §7.8 |
| `SourceRef` names the buffer, macro and generation | **settled** — §4.2 |
| `DoBlock`/`DoGeneratorBlock` appear in both tables | **settled** — §7.7 |

**Every question in §7 is settled.** The two marked blocking three drafts ago —
hygiene and the runtime/replacement distinction — are among them: hygiene by a
minting primitive rather than span contexts, and the phase by the import name
under a strict lexical rule.

**What remains is specification work**: the recursion limit's value
(§5.1), the new CSP directive's NAME and failure mode — the decision to have one
is settled, its spelling is not (§7.4) — the per-position grammar constraints for
the replacement table (§7.7), and the token vocabulary's exact membership
(§4.2). Those are specification work rather than design questions.

**The three design questions of the previous draft are all closed**: expansion is
a pure function of its input, which derives the sandbox's capability list and
excludes the impure parts of ECMA-262 as well as everything host-defined
(§7.4); an empty replacement is permitted
wherever an empty stream parses, which needed no new rule (§7.7); and the narrow
case §7.6 loses gets no accommodation, because the deferral targets it would need
already exist (§7.6).
