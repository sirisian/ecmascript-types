# Build Reporter

A build-metrics reporter: environment configuration read into typed settings, a timing feed fetched and validated, and a summary formatted. This is the scenario the [pipeline operator](../pipelineoperator.md) exists for - a program that is almost entirely *transformations of one value*, where every step is a call whose result feeds the next and half the steps are free functions that were never going to be methods.

It is built from the shapes the proposal and its issue tracker keep returning to: flattening a nest that reads inside out, mixing methods with free functions in one chain, steps that are not calls at all, `await` in the middle of a chain, and the ternary-with-processing-on-the-head that [issue 315](https://github.com/tc39/proposal-pipeline-operator/issues/315) reports as a real-world pattern. What the types add is that each of those keeps its meaning: the topic carries a type through the chain, a literal in a step is the type the step wants, and a test on the topic narrows it.

Features exercised:

- The topic carrying a type from step to step, so a chain needs no annotations and a mistyped step is an error at that step rather than at the end.
- Contextual typing reaching through the pipe - ```clamp(%, 1, 32)``` in a ```uint8``` position types its bounds as ```uint8```, not as Numbers that happen to fit.
- The topic narrowing: ```% is Directory ? %.childCount : 1``` in a single step.
- A pipeline into an exhaustive ```match```, which is [issue 315](https://github.com/tc39/proposal-pipeline-operator/issues/315)'s ternary written as what it wanted to be.
- ```await``` as a step, the case no amount of type information lets a tacit form express.
- A ```do``` step for the one transformation that needs a statement, with the topic in scope inside the block.
- Steps that are not calls - ```% * 1000```, a template literal, a member access - which is the argument for an explicit topic rather than a tacit call.
- Overload resolution and generic inference driven by the topic's type.
- The two refusals, shown as errors: ```ref %``` and a step with no topic.

## Settings from the Environment

The shape [issue 314](https://github.com/tc39/proposal-pipeline-operator/issues/314) reports for factory APIs: `Object.entries` in, transformations in the middle, `Object.fromEntries` out. Written as a nest it reads from the inside; written as a chain it reads in order, and the two free functions sit in the chain beside the two methods without ceremony.

```js
interface Settings {
  workers: uint8;
  timeout: uint32;
  verbose: boolean;
}

const settings: Settings = env
  |> Object.entries(%)
  |> %.filter(([key]) => key.startsWith('BUILD_'))
  |> %.map(([key, value]) => [key.slice(6).toLowerCase(), value])
  |> Object.fromEntries(%)
  |> coerceSettings(%);
```

The topic's type changes at every step and no annotation says so: `env` is an object with an index signature of `string` to `string`, `Object.entries(%)` makes it `[].<[string, string]>`, the `map` keeps that shape, `Object.fromEntries` returns an object, and `coerceSettings` is the one function that has to know what the settings *are*. A step that returned the wrong thing would be an error at that step, where the mistake is, rather than at the assignment four lines later.

## Numbers That Know Their Type

Contextual typing is what makes a numeric chain worth writing. The annotation on the binding reaches every step, so the bounds are the type the step expects:

```js
const workers: uint8 = cpuCount
  |> Math.min(%, 16)
  |> clamp(%, 1, 32);
```

`1`, `16`, and `32` are `uint8` here, not Numbers that happen to fit. Without the propagation each would need writing out, and the chain that was meant to be shorter than the nest would be longer.

The same reaches a range, where the interval is a value rather than a pair of loose arguments:

```js
const percent: uint8 = ratio
  |> % * 100
  |> Math.round(%)
  |> clamp(%, 0..=100);
```

`% * 100` is the step that makes the argument for an explicit topic. It is not a call, so no tacit form expresses it, and writing it as one would mean a `multiply` function existing for the sake of the pipe.

## Narrowing in a Step

A test on the topic narrows it, so a step may ask what it received and act on the answer without leaving the chain:

```js
const size: uint32 = entry
  |> (% is Directory ? %.childCount : 1);
```

`%.childCount` typechecks because the test governs it, exactly as it would for a named binding. This is why the topic is a binding rather than a textual substitution - a substitution has no identity for a test to narrow.

## The Ternary That Wanted to Be a Match

[Issue 315](https://github.com/tc39/proposal-pipeline-operator/issues/315) reports "ternaries with extra processing on the ternary head" as a pattern people write today. It is a `match` with no `match` available:

```js
const label: string = await fetchTimings(url)
  |> match (%) {
    when { status: 200, let body }: body.label;
    when { status: 404 }: 'no build recorded';
    when { status: 500..600, let status }: `server error ${status}`;
    when { let status }: `unexpected ${status}`;
  };
```

The subject is the topic, each arm narrows it, and the `match` is exhaustive or it is a compile-time error. Written as a ternary chain this is neither checked nor readable; written as a `match` whose subject is a nested call, the subject is where the reader has to start.

## Awaiting Mid-Chain

`await` is the case that no type information rescues for a tacit form: it is not a function, so there is no signature to check against and nothing to call. As a step over an explicit topic it is ordinary:

```js
const report: Report = endpoint
  |> new URL(%, base)
  |> fetch(%, { headers })
  |> await %
  |> %.json()
  |> await %
  |> parseReport(%);
```

Each `await %` awaits what the previous step produced. `new URL(%, base)` is a constructor in the chain, which is the other half of what [issue 314](https://github.com/tc39/proposal-pipeline-operator/issues/314) asks for.

## A Step That Needs Statements

One transformation in this program needs a temporary. A `do` expression is a step like any other, and the topic is in scope inside it:

```js
const key: CacheKey = request
  |> do {
    const normalized = %.path.toLowerCase();
    Composite.<CacheKey>({ path: normalized, method: %.method })
  };
```

The block's completion value is the step's value. Without it this step becomes a named function called once, or a temporary hoisted above the chain that the chain existed to avoid.

## Resolution and Inference

The topic is an argument like any other, which matters because a pipeline *looks* like substitution. An overloaded function resolves on the topic's type:

```js
function format(v: uint32): string;
function format(v: Duration): string;

const a: string = elapsedMs |> format(%);        // the uint32 signature
const b: string = elapsed   |> format(%);        // the Duration signature
```

And a generic infers through it, with the explicit form available where inference is not enough:

```js
const first: Reading = readings |> firstOr(%, fallback);
const empty: [].<Reading> = 0 |> Array.from.<Reading>({ length: % });
```

## What the Chain Will Not Do

Two errors worth seeing, because both are things a reader expects to work.

A pipe does not reach through to the place its value came from. The topic holds a *value*, so a reference through it would bind a temporary:

```js
counters.failures |> increment(ref %);   // TypeError: the topic is not a reference
increment(ref counters.failures);        // what was meant, and shorter
```

And every step must mention the topic, which is what stops a tacit call arriving by accident:

```js
timings |> summarize();     // SyntaxError: this step discards what was piped
timings |> summarize(%);    // the call that was meant
```

The second error also refuses a step whose only topic belongs to a *nested* pipe, so `a |> f(b |> g(%))` is an error and `a |> f(%, b |> g(%))` is how it is written.

## The Whole Thing

```js
interface Env { [key: string]: string; }

async function report(env: Env, url: string): string {
  const settings: Settings = env
    |> Object.entries(%)
    |> %.filter(([key]) => key.startsWith('BUILD_'))
    |> Object.fromEntries(%)
    |> coerceSettings(%);

  const workers: uint8 = cpuCount |> Math.min(%, 16) |> clamp(%, 1, 32);

  return url
    |> new URL(%, base)
    |> fetch(%, { signal: timeoutAfter(settings.timeout) })
    |> await %
    |> %.json()
    |> await %
    |> match (%) {
      when { status: 200, let timings }: timings
        |> %.reduce((a, t) => a + t.ms, 0)
        |> % / 1000
        |> `${workers} workers, ${%.toFixed(2)}s`;
      when { status: 500..600, let status }: `build service unavailable (${status})`;
      when { let status }: `unexpected response ${status}`;
    };
}
```

The last arm's body is itself a pipeline, and its topic is the arm's binding rather than the outer one - an inner pipe shadows the outer for the extent of its own step, which is the rule any binding form follows.
