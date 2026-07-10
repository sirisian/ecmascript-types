# Ranges

Every language a JavaScript programmer is likely to reach for has a range: Rust's ```1..=6```, Swift's ```1...6```, Ruby's ```1..6```, Kotlin's ```1..6```, Julia's ```1:6```, Python's ```range(1, 7)```. JavaScript has ```for (let i = 0; i < n; ++i)``` and, if the ```Iterator.range``` proposal lands, a function call. It has no way to name an interval as a value, which is why ```Math.random(min, max)``` must encode inclusivity in a convention nobody can see, why ```slice``` and ```subarray``` take loose pairs of numbers, and why "is x between a and b" is written out by hand every time.

This document proposes range literals, gives them types, and then checks the result against every feature of this proposal it could touch. The last part is the point: a range is only worth adding if it makes the rest of the language smaller, and the test of that is whether existing features absorb it without special cases.

## The Lexical Problem

Any range operator built from ```.``` collides with the numeric literal grammar, because ```1.``` is already a complete NumericLiteral. Precisely:

```js
1..6           // SyntaxError today: NumericLiteral `1.` then NumericLiteral `.6`
0..10          // SyntaxError today
a..b           // SyntaxError today: `a.` is not a valid member access
1..=6          // SyntaxError today
1...6          // SyntaxError today

1..toString()  // Parses today. `1.` then `.toString()`. Evaluates to "1"
1..x           // Parses today. `(1.).x`, which is undefined
```

So ```a..b``` and ```1..6``` are free. The one thing standing in the way is an integer literal followed by ```..``` followed by an identifier.

**The rule.** A ```.``` is not consumed as the decimal point of a NumericLiteral when the next character is also a ```.```. That single change makes ```1..6```, ```1..=6```, and ```1..x``` all lex as three tokens.

**What breaks.** Exactly one production: ```IntegerLiteral .. IdentifierName```, the ```1..toString()``` idiom. It doesn't break silently. After the change, ```1..toString()``` is a range from ```1``` to the result of calling ```toString```, which throws where it doesn't simply fail to construct. Every other spelling of the same idiom keeps working:

```js
(1).toString(); // Fine
1 .toString(); // Fine
1.0.toString(); // Fine
1..toString(); // Now a range whose end is not a number
```

Rust made floats require ```1.0``` for precisely this reason, and got ```..``` in exchange. The trade here is smaller: JavaScript's ```1..toString()``` is a golfing curiosity, not a pattern in production code, and the failure is loud.

**The alternatives, and why not.** ```...``` is free lexically, since spread only ever appears in prefix position, but Ruby's ```..```/```...``` pairing, where one extra dot silently flips inclusivity, is a well-documented footgun and ```...``` reads as spread to every JavaScript programmer alive. ```..<``` (Swift, Kotlin) is unusable here because this proposal already gives ```.<``` to generic application, so ```1..<6``` would begin parsing a type argument list. Contextual keywords (```1 to 6```, Kotlin's ```until```) need no lexical change at all and are the fallback if the committee finds the breakage unacceptable, but they read poorly in index position (```arr[2 to 5]```) and interact badly with ASI.

## Syntax

```
RangeExpression :
    AssignmentExpression? .. AssignmentExpression?
    AssignmentExpression? ..= AssignmentExpression
```

Six forms, following Rust:

```js
a..b   // [a, b)  half-open
a..=b  // [a, b]  inclusive
a..    // [a, ∞)  from
..b    // (∞, b)  to
..=b   // (∞, b]  to inclusive
..     // full
```

Bare ```..``` is half-open because that is what JavaScript already means everywhere else: ```slice```, ```subarray```, ```splice```, array indices, and ```Iterator.range``` are all half-open, and ```0..arr.length``` should be the loop everyone writes without thinking.

```..``` binds tighter than assignment and looser than ```||``` and ```??```, and it is **non-associative**: ```a..b..c``` is a SyntaxError rather than a puzzle. Member access binds tighter than the range, so ```0..10.length``` is ```0..(10.length)``` and the whole range is parenthesized to reach its own members:

```js
(0..10).length; // 10
```

Two hazards worth writing into the grammar notes. A statement beginning with ```..``` continues the previous expression, joining ```(```, ```[```, and ```` ` ```` on the list of ASI hazards. And a range is an expression, so it appears in ```case``` labels, object literals, and argument lists without further ceremony.

## Types

A range is a value type class, so ```0..10``` allocates nothing and copies by value. The endpoint kinds are distinct types, because an omitted endpoint is a different shape, not a missing value:

```js
enum Interval: uint8 { Closed, ClosedOpen, OpenClosed, Open }; // Exposed as Range.Interval

interface RangeBounds<T> {
	contains(value: T): boolean;
}

class Range<T, I: Interval = Interval.ClosedOpen> implements RangeBounds.<T> {
	readonly start: T;
	readonly end: T;
	get isEmpty(): boolean;
	contains(value: T): boolean;

	// The interval is a value generic, not an argument. A runtime interval
	// would defeat the specialization the type exists to provide.
	static of<I: Interval>(start: T, end: T): Range.<T, I>;
}

class RangeFrom<T> implements RangeBounds.<T> { readonly start: T; }
class RangeTo<T, I: Interval = Interval.ClosedOpen> implements RangeBounds.<T> { readonly end: T; }
class RangeFull<T> implements RangeBounds.<T> {}
```

```..``` has no endpoint to infer from, so ```RangeFull.<T>``` takes its ```T``` from the context that consumes it, and is ```RangeFull.<any>``` where there is none.

```T``` needs only ```operator<```, so ranges work over the integer types, the float types, ```bigint```, ```decimal128```, ```rational```, dimensioned quantities, and ```Temporal.Instant``` - anything with an ordering.

The interval kind lives in the **type**, not in a field. ```1..=6``` is a ```Range.<uint8, Interval.Closed>```, so a function taking a range specializes on its inclusivity and no branch survives to runtime. The two literal forms produce ```Closed``` and ```ClosedOpen```; ```OpenClosed``` and ```Open``` come from ```Range.of```, matching every language surveyed, none of which give ```(a, b]``` a literal.

Element types come from literal propagation, as anywhere else:

```js
const a = 1..=6; // Range.<number, Interval.Closed>
const b: Range.<uint8, Interval.Closed> = 1..=6; // uint8 endpoints
0n..10n; // Range.<bigint>
Meter(0)..Meter(10); // Range.<float32.<{ m: 1 }>>
```

Descending ranges are **empty**, not reversed: ```10..0``` contains nothing, and ```(0..10).reverse()``` is how you count down. This is Rust's rule and it exists because the alternative silently produces a loop that runs backwards when a variable goes negative.

## Random

The [random extension](random.md) takes a range as its only bounds parameter, which is what motivated this document. ```Math.random.<uint8>(1..=6)``` is a die, ```Math.random.<float32>(0..1)``` is the unit interval, and ```Math.random.<uint8>(..)``` is the type's full range. The interval lives in the range's type, so the four cases specialize with no runtime branch, and there is no convention to remember or misdocument.

## Ranges as Types

Here is where the feature pays for itself. A range literal with constant endpoints is a compile-time-evaluable expression, and this proposal already says that such an expression is a valid type annotation. A range is also, exactly, what ```NumberBounds``` describes:

```js
type Die = uint8.<1..=6>; // uint8.<{ minimum: 1, maximum: 6 }>
type Unit = float32.<0..1>; // float32.<{ minimum: 0, exclusiveMaximum: 1 }>
type Percent = uint8.<0..=100>;
type Angle = float32.<0..6.283185307>;
```

The desugaring is mechanical: ```..``` sets ```exclusiveMaximum```, ```..=``` sets ```maximum```, and an omitted endpoint omits the corresponding constraint. One notation now spells an interval in all three places a program needs one - as a value, as a constraint, and as a random source - and they agree by construction:

```js
type Die = uint8.<1..=6>;
const roll: Die = Math.random.<Die>(); // No arguments: the type is the range
JSON.parse.<{ roll: Die }>(body); // Validated during the parse
function play(value: Die) {} // Checked at the boundary
```

```Math.random.<Die>()``` produces a value that satisfies ```Die```'s constraint by construction, so the validation a cast would normally run at the return boundary is elided. The generator cannot produce anything else.

## Iteration

```Range``` and ```RangeFrom``` over integer types define ```*operator...()```, so they iterate:

```js
for (const i of 0..array.length) {} // The loop everyone writes
for (const i of 0..=9) {} // 0 through 9
[...0..5]; // [0, 1, 2, 3, 4]. Spread is prefix, so no ambiguity
(0..100).step(2); // Every other value
(0..).take(10); // RangeFrom is infinite; iterator helpers apply
```

Float ranges are **not iterable**, because there is no canonical step, and iterating one is a TypeError rather than a silent choice of ```1```. ```RangeTo``` and ```RangeFull``` are not iterable either, having no start. Both restrictions match Rust, and both are the kind of thing that must be decided rather than discovered.

Because ```Range``` is a value type and ```*operator...()``` is an ordinary iteration operator, ```for (const i of 0..n)``` inlines to a counted loop with no iterator object allocated. It is the C-style loop, spelled once.

This subsumes the common case of ```Iterator.range```, which remains useful for a dynamic step and for ```bigint``` sequences.

## Indexing and Slicing

This proposal already has user-defined index operators. A range-taking overload is all that slicing needs, and it lands on the ```window``` method the arrays section recently gained:

```js
class Array<T> {
	get operator[](range: Range.<uint32>): [].<T>; // Aliases, doesn't copy
}

array[2..5]; // Elements 2, 3, 4 as a view
array[2..=5]; // Elements 2 through 5
array[2..]; // The tail
array[..3]; // The head
array[..]; // The whole thing
```

```array[a..b]``` and ```array.window(a, b)``` are the same operation, and the range form should be the one people write. Two consequences fall out for free. When the endpoints are compile-time constants the length is too, so the result is a fixed-extent view whose type says so:

```js
const head: [3].<uint8> = bytes[0..3]; // The length is known
```

And the SIMD pass in the [entity component system](examples/ecs.md) example loses its last piece of arithmetic:

```js
const lanes = [].<float32x4>(vx[0..whole]);
```

A runtime start with a compile-time length still wants ```window.<N>(start)```, since ```start..start + Words``` requires symbolic arithmetic to see that the length is constant. That is an honest limit, not a gap: the two forms coexist.

## Containment, switch, and `is`

```(1..=6).contains(x)``` is the containment test. It is deliberately **not** spelled ```x in 1..=6```, Kotlin's way, because ```in``` already means "has this property" and ```3 in someRange``` is currently legal and false; overloading it would change the meaning of running code rather than of a syntax error.

Instead, containment reaches the type system, which is the more useful direction:

```js
if (x is uint8.<1..=6>) {} // Structural test against the bounded type
if ((1..=6).contains(x)) {} // The value-level test
```

Range case labels extend ```switch``` the way sealed classes did. When a case label is a range, the clause matches if the range contains the discriminant. It is the same generalization already made for type-object labels, and it costs no new concept:

```js
switch (statusCode) {
	case 200..300: return Result.Ok;
	case 400..500: return Result.ClientError;
	case 500..600: return Result.ServerError;
	default: return Result.Unknown;
}
```

Exhaustiveness is not claimed for range labels. Deciding whether a set of intervals covers a type is a different analysis from listing an enum's members, and this proposal's stated position is that exhaustiveness belongs to enums and sealed classes. Ranges don't change that.

## The Rest of the Language

- **Dimensioned quantities.** ```Meter(0)..Meter(10)``` is a ```Range.<Meter>``` because ```float32.<D>``` defines ```operator<```. Mixing dimensions is the same TypeError it is anywhere: ```Meter(0)..Second(10)``` fails at the range's construction.
- **Temporal.** ```start..end``` over ```Temporal.Instant``` works, and it subsumes the ad-hoc ```Interval``` record in [temporal.md](temporal.md). It isn't iterable without a step, and ```(start..end).step(Temporal.Duration.from('PT1H'))``` reads better than the loop it replaces. A non-empty requirement stays a ```where``` clause, since an empty range is a legal value, not an error.
- **decimal and rational.** Ordered, so ranged. ```Math.random.<decimal128.<{ scale: 2 }>>(0..=100)``` quantizes at the return boundary like any other decimal.
- **Dependent record types.** ```where (0..=150).contains(this.age)``` reads as the constraint it is, and the bounded-type spelling ```age: uint8.<0..=150>``` is better still, since it validates at every boundary rather than at record construction.
- **Value type references.** ```for (const ref p of particles[0..n])``` composes: the slice is a view, and reference iteration over a view is reference iteration.
- **SoA.** ```mesh.fields.position[0..count]``` is the upload slice.
- **Destructuring.** ```const { start, end } = 0..10``` works, because those are the fields.
- **Spread.** ```f(...0..3)``` passes ```0, 1, 2```. Legal, since spread is prefix and the range is its operand.

## What This Costs

One lexical rule, one broken idiom, one new expression form, four small classes, and one operator overload on arrays. In exchange: the C-style loop, ```slice```'s argument pairs, the inclusivity convention in ```Math.random```, the ```minimum```/```maximum``` metadata spelling, the ```window``` method, ```Iterator.range```'s common case, and ```temporal.md```'s ```Interval``` record all become one notation.

The completeness test this document set for itself was whether existing features absorb ranges without special cases. They do, with two exceptions worth naming rather than hiding: containment cannot use ```in``` without breaking legal code, and exhaustiveness over ranges is not offered. Both are refusals, not gaps.

## Open Questions

- Should ```String``` gain the range index operator (```str[1..3]```), or is ```slice``` enough? Strings are not typed arrays here, so this is a separate decision.
- ```Range.of.<Interval.Open>(a, b)``` is the only way to name the two open-start intervals, and it needs its interval as a compile-time constant. Choosing an interval at runtime therefore produces a union of range types, which a ```switch``` handles and which is rare enough that no sugar is proposed.
- Coordination with ```Iterator.range```: ranges subsume its two-argument form. The proposals should agree on whether ```Iterator.range(0, 10)``` and ```0..10``` produce the same iterator.
- Precedence against the pipeline operator, if that proposal advances.
