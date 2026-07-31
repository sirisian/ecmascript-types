# Pipeline Operator

An operator for reading a computation in the order it happens. `x |> f(%)` evaluates the left side, binds it to `%`, and evaluates the right side, so a chain of operations reads left to right whether or not the operations are methods.

```js
const size: uint8 = width
  |> % * devicePixelRatio
  |> Math.round(%)
  |> clamp(%, 0, 255);
```

Written without it, that is either nested — `clamp(Math.round(width * devicePixelRatio), 0, 255)` — which reads inside out, or a chain of temporaries whose names exist only to be read once. Method chaining reads correctly but is available only when the operations happen to be methods of the value, which is why `map` and `filter` compose and `Math.round` and `clamp` do not.

This document does three things. It records what a type system changes about the proposal's longest-running argument, because the answer is not the one an erased language has to give. It states the type of the topic and the handful of rules that follow from it. And it says what a pipeline does *not* do — the two refusals matter more than any of the rules, because both are things a reader will otherwise assume.

## The Expression

`|>` is left-associative and binds tighter than the comma and looser than everything else, so a pipeline is a sequence of steps rather than a nest. The left operand is evaluated once. The right operand is an ordinary expression, evaluated with ```%``` bound to that value, and its result is the pipeline's.

```js
value |> f(%)             // f(value)
value |> f(%, extra)      // the topic need not be first
value |> f(g(%))          // nor outermost
value |> % + 1            // nor an argument at all
value |> [%, %]           // it may appear more than once
```

The topic is a binding, not a textual substitution. It is evaluated once however many times it appears, so `expensive() |> [%, %]` calls `expensive` once, and it is **immutable** — `value |> (% = 1)` is an error, because a pipeline step that reassigns the thing being piped is a step whose input the next step cannot predict.

```%``` is meaningful only inside a pipeline's right operand. Outside one it is the remainder operator and nothing else, and a bare ```%``` where no pipeline encloses it is a syntax error rather than a reference to something.

A nested pipeline rebinds it. In `a |> f(b |> g(%))`, the ```%``` inside `g` is `b`, not `a`; the inner pipeline shadows the outer for the extent of its own right operand, which is the same rule any binding form follows and the reason a topic is described as a binding rather than a placeholder.

## Why the Topic Is Explicit

The proposal's longest argument is between this form, where a step names its input, and the tacit form — `x |> f`, calling `f` with the value and no placeholder — which reads better in the case where the step is exactly a unary call and cannot express any other case.

The case against tacit calls in JavaScript has always rested on the language not knowing what a function expects. `x |> f` where `f` takes two parameters is a call with one, silently, and the failure arrives somewhere else. `await` and `yield` have no tacit form at all, so a pipeline containing one needs a placeholder anyway. And making the tacit form useful in general needs partial application, which is a second proposal.

**A typed language answers the first of those.** A function here has a declared signature: `x |> f` could be checked, the topic tested against the first parameter, and a signature requiring more arguments than the pipe supplies rejected at the pipe rather than at the eventual call. That is worth stating because the argument against tacit calls is usually made as though it were about JavaScript rather than about erasure, and here it is about erasure. The other two objections survive: `await` is not a function, and no amount of type information makes `x |> await` mean anything.

So the explicit topic is the right form, and the reason is narrower than it is usually given. It is not that a tacit call cannot be checked — it can be, here. It is that a tacit call cannot express the steps that are not calls, and a pipeline whose steps are sometimes tacit and sometimes not is two features wearing one operator.

## The Token

```%``` follows the proposal. What it costs is worth writing down, because the cost is this design's rather than the base language's, and nobody arguing the token in committee has been arguing from a language like this one.

An expression language with operator overloading has fewer free tokens than one without. This design has spent:

| Token | Spent on |
|---|---|
| ```#``` | private fields |
| ```?``` | optional chaining, and optional parameters and members — ```a?: T``` |
| ```@``` | decorators, in about twenty positions rather than the base proposal's five |
| ```_``` | the [pattern matching](patternmatching.md) wildcard |
| ```%```, ```^``` | binary operators a class or a primitive may **declare** |

The last row is the one that matters. ```%``` parses unambiguously — a nullary ```%``` sits where an operand is expected and a binary one between two operands, and a declared `operator %` supplies a meaning for the binary form without ever supplying a new arity. What suffers is reading, in the one case where both appear together:

```js
total |> compute(% % divisor)   // topic, then a remainder that a type may have redefined
```

That is legal and it is not good. It is also rare enough to be a reason to revisit the token rather than a reason to reject it, and if the committee settles on ```^``` instead, this paragraph applies unchanged, because ```^``` is declarable here too.

**```_``` is the token this design would otherwise want, and cannot have.** Scala spells the topic that way and it is the obvious choice. But ```_``` is already the pattern wildcard, where it matches anything and binds nothing, and a topic is nearly its opposite: it names exactly one value. One glyph meaning "anything at all" in a `match` and "this specific thing" in a pipeline is worse than the remainder collision, and unlike the token question, this one this design can settle on its own.

## The Type of the Topic

The topic has the type of the left operand, and it is a binding in every sense the type system cares about — scoped to the right operand, immutable, shadowed by an inner pipeline.

```js
const n: uint32 = xs.length |> % * 2;   // % is uint32; the pipeline is uint32
```

Everything else follows from that, and the rules below are short because they are the ordinary rules applied to one more binding.

### A contextual type reaches through it

The type a pipeline is written into flows into its right operand, so a literal in a step takes the expected type rather than defaulting:

```js
const level: uint8 = base |> clamp(%, 0, 10);   // 0 and 10 are uint8, not number
```

Without this the feature would be typed and unusable, which is the same failure a [`do` expression](doexpressions.md) would have had, and the same propagation answers both.

### The topic narrows

A test on the topic narrows it, in the positions the test governs, exactly as a test on any other reference does:

```js
shape |> (% is Circle ? %.radius : 0)
value |> (typeof % === 'string' ? %.length : 0)
```

This is the rule a pipeline cannot do without, and it is the reason the topic has to be a binding rather than a substitution: a substitution has no identity to narrow.

### `await` and `yield` are the enclosing function's

A pipeline is an expression, not a function body. `await` inside a step is the enclosing async function's, and `yield` inside a step is the enclosing generator's:

```js
async function load(id: uint32): Config {
  return id |> fetchConfig(%) |> await % |> validate(%);
}
```

## What a Pipeline Does Not Do

Two refusals, both of which a reader will otherwise assume, and the second of which is the more dangerous.

### `|>` cannot be overloaded

A class may declare operators, so the question is fair. The answer is no, and the reason is structural rather than a matter of taste: [operator resolution](operatoroverloading.md) searches the *left operand's type* for a signature, and a pipeline has nothing to search for. `|>` does not compute a value from two operands; it binds one and evaluates the other. There is no operation to give a meaning to.

### The topic is a value, not a place

This is the sharp edge. A [reference parameter](references.md) binds to the caller's location, so `f(ref o.a)` writes back through `o.a`. A topic holds the *value* the left operand produced, so a reference through it would bind a temporary and the write would go nowhere:

```js
o.a |> f(ref %)   // Error: the topic is not a reference
```

**```ref %``` is a type error.** It is refused rather than allowed-and-useless because that expression reads as though the pipe were transparent to the place, and it is not. A program that wants the write writes `f(ref o.a)`, which is shorter anyway.

## Resolution, Inference, and the Rest

Three things that are ordinary once said and confusing if left unsaid, because a pipeline looks like substitution and is not.

**Overloads resolve on the topic's type.** `x |> f(%)` chooses among `f`'s signatures by what the topic is, the same as any argument. **Generic parameters infer through it**, so `x |> identity(%)` takes its parameter from the topic and `x |> f.<uint8>(%)` supplies it explicitly. And the topic goes wherever an expression goes, including a spread or a [rest position](README.md): `xs |> f(...%)` spreads it, and `x |> f(a, %, ...rest)` places it among the arguments the rest assignment then distributes.

Two smaller confirmations. A step that diverges makes the pipeline `never` — a call returning `never`, or a `throw` in a step once throw expressions exist — by the same divergence analysis a `switch` and a `match` use. And a pipeline is compile-time evaluable exactly when its operands are, so it may appear inside a [type builder](typeprogramming.md) on the same terms as any other expression; it introduces a binding and substitutes, and reads nothing the surrounding code could not.

## What It Composes With

**Pattern matching**, most of all. A pipeline that ends in a `match` is the shape this operator exists for, and the one that is hardest to write without it:

```js
const label: string = response
  |> parse(%)
  |> match (%) {
    when { status: 200, let body }: body.title;
    when { status: 404 }: 'not found';
    when { let status }: `error ${status}`;
  };
```

The subject is the topic, the arms narrow it, and the `match` is exhaustive or it is an error. Written without the pipe this is a `match` whose subject is a nested call, or three temporaries; written with a ternary, which is how it is usually written today, it is neither exhaustive nor readable.

**`do` expressions**, for a step that needs statements. `x |> do { … }` is a step whose value is the block's completion value, so a step that needs a temporary or a `try` does not have to become a function.

**Errors.** A pipeline over a value that is either a success or a failure is the pattern the pipe proposal calls railway-oriented, and its own issue tracker records as far-future because JavaScript lacks the pieces. Some of those pieces are here: [typed `catch`](errorhandling.md) filters by type, `match` discriminates a result without a library, and `do` gives a step statements. What is not here is a pipe that *knows* about the failure track — `value |maybe> f(%)` — and this document does not propose one. It is worth knowing how much of the distance is already covered.

**Ranges and the numeric types.** `x |> clamp(%, 0..255)` and `n |> Math.mod(%, len)` are the small everyday case, and they are where the contextual type matters most, since the literals in a step should be the type the step expects rather than Numbers that happen to fit.

A worked program using all of the above is in [examples/pipeline.md](examples/pipeline.md): environment configuration read into typed settings, a timing feed fetched and validated, and a summary formatted, built from the shapes the proposal's own issue tracker keeps returning to.
